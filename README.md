# Part3-GitOps

The infra repo for the cluster's applications. It holds the Argo CD configuration that deploys the sample Node.js app and MongoDB, using the **app of apps** pattern.

| File | Purpose |
|---|---|
| [`root-app.yaml`](root-app.yaml) | Root `Application`: applied by hand once, then manages everything else in this repo |
| [`sample-nodejs/appproject.yaml`](sample-nodejs/appproject.yaml) | `AppProject`: what the app may deploy, and where |
| [`sample-nodejs/application.yaml`](sample-nodejs/application.yaml) | `Application`: the chart from Part2-Application, following release tags |
| [`mongodb/appproject.yaml`](mongodb/appproject.yaml) | `AppProject`: what MongoDB may deploy, and where |
| [`mongodb/application.yaml`](mongodb/application.yaml) | `Application`: the Bitnami chart, with values from Part1-Infrastucture |

| Repo | Holds |
|---|---|
| [Part1-Infrastucture](https://github.com/elad1122/Part1-Infrastucture) | The RKE2 cluster, Argo CD, MongoDB's values and chart-repo Secret |
| [Part2-Application](https://github.com/elad1122/Part2-Application) | App code, Helm chart with its `values.yaml`, CI/CD pipeline |
| **Part3-GitOps** (this repo) | The Argo CD root app, `AppProject`s and `Application`s |

App URL: `http://app.34.56.31.129.nip.io/my-app`

## Why a separate repo instead of the app repo

The assignment allows either. I chose a separate repo because:

| Reason | What it means here |
|---|---|
| **Separation of concerns** | The app repo describes the app; this repo describes how the cluster deploys it |
| **Access control** | Who can change deployment rules (allowed repos, namespace, sync policy) can be managed apart from who can change code |
| **No CI noise** | Changes here don't run the app's pipeline, and app commits don't touch the deployment rules |
| **Room to grow** | More apps or environments can be added here as more folders |

**Trade-off:** one more repo to maintain.

## Why app of apps

Argo CD watches the sources an Application points to, not the file that defines the Application. Without a parent, a change to a file here would do nothing until someone ran `kubectl apply`.

The root app watches this repo, so **a commit here is all it takes**: changing a manifest, adding an app, or removing one. After the one bootstrap command, the cluster's Argo CD configuration comes entirely from Git, and rebuilding means applying the root app again.

## Secrets: why no repo credentials are needed here

I didn't need to create a Secret to connect Argo CD to these repositories:

- **Part2-Application is public**, so Argo CD can read the sample app's chart without credentials. The same is true of this repo and Part1-Infrastucture, which provides MongoDB's values.
- **MongoDB's chart repo uses the Secret in Part1-Infrastucture**: [`argocd/repositories/bitnami-oci.yaml`](https://github.com/elad1122/Part1-Infrastucture/blob/main/argocd/repositories/bitnami-oci.yaml). It registers Bitnami's OCI registry with Argo CD (`enableOCI: "true"`). The registry is public, so this Secret holds no password, which is why it's safe in Git.

Only one real credential exists: `mongodb-auth`, MongoDB's own root password and replica-set key. It's created directly on the cluster and is never in Git.

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
- **One values file per app:** the app's settings live only in `helm/sample-nodejs/values.yaml` in Part2-Application, and MongoDB's only in `mongodb/values.yaml` in Part1-Infrastucture.

## Sync policy and projects

- **Automated sync:** `prune` removes what's deleted from Git, and `selfHeal` reverts manual changes to anything Git defines. Fields that Git doesn't define are ignored. The root app applies this to the Applications themselves.
- **Restricted AppProjects:** each app can use only its own sources, only the `default` namespace (the assignment's requirement), and only the resource kinds its chart creates, with no cluster-scoped resources. Sync waves create each project before its Application.
- **Ingress health rule:** Traefik here binds hostPorts and publishes no address, so Argo CD's default check would never finish. The rule is in [Part 1's Argo CD values](https://github.com/elad1122/Part1-Infrastucture/blob/main/argocd/values.yaml).

## Steps

**1. Install (once).** Argo CD, the Bitnami repo Secret and `mongodb-auth` must exist first; see [Part 1's README](https://github.com/elad1122/Part1-Infrastucture#mongodb). Then:
```bash
kubectl apply -f root-app.yaml
kubectl -n argocd get applications
```

**2. Deploy the app:** merge a `feat:` or `fix:` commit to `main` in Part2-Application. Nothing else is needed.

**3. Change settings:**
- **App:** edit `helm/sample-nodejs/values.yaml` in Part2-Application and release. Settings take effect with the release, because Argo CD reads the chart at the tag.
- **MongoDB:** edit `mongodb/values.yaml` in Part1-Infrastucture and push to `main`. Note that MongoDB has no persistence, so a change that restarts its pods can lose data.

**4. Roll back the app:** pin an exact tag in `sample-nodejs/application.yaml`, then commit and push:
```yaml
targetRevision: v1.0.0
```

