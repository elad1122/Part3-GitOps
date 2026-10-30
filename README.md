# Part3-GitOps

The infra (GitOps) repo for the sample Node.js app. **What's in this repo is what runs on the cluster:** Argo CD watches it and keeps the cluster matching it.

| Repo | Holds |
|---|---|
| [Part1-Infrastucture](https://github.com/elad1122/Part1-Infrastucture) | The RKE2 cluster, Argo CD, MongoDB |
| [Part2-Application](https://github.com/elad1122/Part2-Application) | App code, Helm chart, CI/CD pipeline |
| **Part3-GitOps** (this repo) | Which version is deployed, and with what settings |

App URL: `http://app.34.56.31.129.nip.io/my-app`

## Why a separate repo instead of the app repo

The assignment allows either. I chose a separate repo because:

| Reason | What it means here |
|---|---|
| **Separation of concerns** | The app repo describes what the app is; this repo describes what's running |
| **Clear deployment history** | `git log` here is the deployment log, with no code commits mixed in |
| **Simple rollback** | `git revert` returns the cluster to the previous version, with no rebuild and no change to app code |
| **No CI loops** | Deployment commits land outside the app repo, so they can't retrigger its pipeline |
| **Least privilege** | CI can write only to this repo through a deploy key, and Argo CD only reads |
| **Room to grow** | More apps and environments can be added as more files here |

**Trade-offs:** one more repo to maintain, and CI needs a credential to write to it.

## How a release reaches the cluster

```
merge to main (Part2-Application)
  → security gates → image pushed and signed → release vX.Y.Z
  → deploy job commits "targetRevision: vX.Y.Z" here
  → Argo CD detects the commit (polls about every 3 minutes)
  → rolling update on the cluster
```

- **The pipeline has no cluster credentials.** It only commits to git, and Argo CD applies the change from inside the cluster.
- **A blocked build can't deploy.** The deploy job runs only after every gate, the image push and the release have passed.
- **One line moves everything:** the chart at tag `vX.Y.Z` has `appVersion: X.Y.Z`, which is the default image tag, so the chart and the image can't get out of step.

## Layout

```
.
├── bootstrap/root-app.yaml        # applied once by hand; manages everything under argocd/
├── argocd/
│   ├── projects/sample-nodejs.yaml  # AppProject: allowed repos, namespace and resource kinds
│   └── apps/sample-nodejs.yaml      # Application: which chart version to deploy
└── values/sample-nodejs/values.yaml # cluster settings for the chart
```

Values sit outside `argocd/` because the root app applies every file under that folder as a Kubernetes manifest.

## Declarative setup

- **App of apps:** the root Application watches `argocd/`. A change to the Application manifest in git is applied automatically, including a new version.
- **Multi-source Application:** the chart comes from Part2-Application at the release tag, and the values come from this repo.
- **Automated sync:** `prune` removes what's deleted from git, `selfHeal` reverts manual changes on the cluster, and failed syncs retry 3 times.
- **Restricted AppProject:** only these two repos, only the `default` namespace (the assignment's requirement), only the resource kinds the chart creates, and no cluster-scoped resources.
- **Ingress health rule:** Traefik here binds hostPorts and publishes no address, so Argo CD's default check would never finish. The rule is in [Part 1's Argo CD values](https://github.com/elad1122/Part1-Infrastucture/blob/main/argocd/values.yaml).

## Steps

**1. Bootstrap (once):**
```bash
kubectl apply -f bootstrap/root-app.yaml
kubectl -n argocd get applications
```

**2. Give CI write access (once):**
```bash
ssh-keygen -t ed25519 -C gitops-deploy -f gitops_deploy
```
- Add `gitops_deploy.pub` under this repo's Settings → Deploy keys, with **Allow write access**.
- Add the contents of `gitops_deploy` to Part2-Application as the Actions secret `GITOPS_DEPLOY_KEY`.
- Delete both local files.

**3. Deploy:** merge a `feat:` or `fix:` commit to `main` in Part2-Application. Nothing else is needed.

**4. Roll back:**
```bash
git revert <deploy commit>
git push origin main
```
Argo CD returns the cluster to the previous version. Don't use `helm rollback` or `kubectl edit`, because `selfHeal` reverts anything that isn't in git.

**5. Change settings:** edit `values/sample-nodejs/values.yaml` and push.

## Verification

First deployment of v1.0.0 through Argo CD:
```
$ kubectl -n argocd get applications
NAME            SYNC STATUS   HEALTH STATUS
mongodb         Synced        Healthy
root            Synced        Healthy
sample-nodejs   Synced        Healthy

$ kubectl -n default get pods -l app.kubernetes.io/instance=sample-nodejs
sample-nodejs-55d89fc5db-7qf9s   1/1   Running   0
sample-nodejs-55d89fc5db-kgn8q   1/1   Running   0
```

Through the Ingress:
```
/my-app -> 200 "Hello, World!"     /about -> 200
/classified -> 404   /metrics -> 404   /ready -> 404   /live -> 404
```

- **Headers:** `X-Powered-By` is removed; `X-Frame-Options`, `X-Content-Type-Options` and `Referrer-Policy` are added.
- **Pods:** image `ghcr.io/elad1122/sample-nodejs:1.0.0`, uid 65532, read-only filesystem, about 23 MiB of memory each.

## Limitations

- **Sync delay:** Argo CD polls, because GitHub webhooks can't reach it through the firewall. Deployments appear within about 3 minutes.
- **Access:** the app URL is reachable only from IPs allowed by the GCP firewall.
- **One environment:** a staging environment would be another values file and Application here.
