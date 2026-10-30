# Part3-GitOps

The infra repo for the sample Node.js app. It holds the Argo CD configuration that deploys the app.

| File | Purpose |
|---|---|
| [`appproject.yaml`](appproject.yaml) | `AppProject`: what this app may deploy, and where |
| [`application.yaml`](application.yaml) | `Application`: what to deploy (chart from the app repo) and how to sync it |

| Repo | Holds |
|---|---|
| [Part1-Infrastucture](https://github.com/elad1122/Part1-Infrastucture) | The RKE2 cluster, Argo CD, MongoDB |
| [Part2-Application](https://github.com/elad1122/Part2-Application) | App code, Helm chart with its `values.yaml`, CI/CD pipeline |
| **Part3-GitOps** (this repo) | The Argo CD `AppProject` and `Application` |

App URL: `http://app.34.56.31.129.nip.io/my-app`

## Why a separate repo instead of the app repo

The assignment allows either. I chose a separate repo because:

| Reason | What it means here |
|---|---|
| **Separation of concerns** | The app repo describes the app; this repo describes how the cluster deploys it |
| **Access control** | Who can change deployment rules (allowed repos, namespace, sync policy) can be managed apart from who can change code |
| **No CI noise** | Changes here don't run the app's pipeline, and app commits don't touch the deployment rules |
| **Room to grow** | More apps or environments can be added here as more manifests |

**Trade-off:** one more repo to maintain, and these two files are applied by hand (below). The structure is deliberately simple: no app-of-apps, so a change to the files here takes effect only after `kubectl apply`.

## How a release reaches the cluster

```
merge to main (Part2-Application)
  → security gates → image pushed and signed → release job creates tag vX.Y.Z
  → Argo CD finds the new tag (checks about every 3 minutes)
  → rolling update on the cluster
```

- **Follows release tags:** `targetRevision: ">=1.0.0"` resolves to the newest `vX.Y.Z` tag of the app repo.
- **A blocked build can't deploy:** the tag is created only after every gate and the image push have passed. Commits to `main` alone don't deploy anything.
- **One tag moves everything:** the chart at tag `vX.Y.Z` has `appVersion: X.Y.Z`, which is the default image tag, so the chart and image can't get out of step.
- **No credentials in CI:** the pipeline holds no cluster credentials and no key for this repo. Argo CD pulls from the public repo.
- **One values file:** settings live only in `helm/sample-nodejs/values.yaml` in the app repo, so they're versioned with each release.

## Sync policy and project

- **Automated sync:** `prune` removes what's deleted from the chart, `selfHeal` reverts manual changes on the cluster, and failed syncs retry 3 times.
- **Restricted AppProject:** only the app repo, only the `default` namespace (the assignment's requirement), only the resource kinds the chart creates, and no cluster-scoped resources.
- **Ingress health rule:** Traefik here binds hostPorts and publishes no address, so Argo CD's default check would never finish. The rule is in [Part 1's Argo CD values](https://github.com/elad1122/Part1-Infrastucture/blob/main/argocd/values.yaml).

## Steps

**1. Install (once, and again after changing a file here):**
```bash
kubectl apply -f appproject.yaml -f application.yaml
kubectl -n argocd get applications
```

**2. Deploy:** merge a `feat:` or `fix:` commit to `main` in Part2-Application. Nothing else is needed.

**3. Change settings:** edit `helm/sample-nodejs/values.yaml` in Part2-Application and release. Settings take effect with the release, because Argo CD reads the chart at the tag.

**4. Roll back:** pin an exact tag in `application.yaml` and apply it:
```yaml
targetRevision: v1.0.0
```
```bash
kubectl apply -f application.yaml
```
To return to automatic deployments, restore `">=1.0.0"` and apply again. While pinned, new releases are not deployed. Don't use `helm rollback` or `kubectl edit`, because `selfHeal` reverts anything that doesn't match the chart.

## Verification

First deployment of v1.0.0 through Argo CD:
```
$ kubectl -n argocd get applications
NAME            SYNC STATUS   HEALTH STATUS
mongodb         Synced        Healthy
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
- **Manual apply:** changes to the two files here need `kubectl apply`. App-of-apps would automate that, but it isn't needed for one app.
- **One environment:** a staging environment would be another Application here, with its own values file in the chart.
