# Techtaurant GitOps

GitOps manifests for the Techtaurant k3s cluster.

## Structure

```text
apps/
  argocd.yaml            # Argo CD Application manifests
  be-dev.yaml
  redis-dev.yaml
  rustfs-dev.yaml
  sealed-secrets.yaml
argocd/
  kustomization.yaml     # Resources managed by apps/argocd.yaml
  argocd-server-nodeport.yaml
be-dev/
be-prod/
postgres-dev/
postgres-prod/
redis-dev/
redis-prod/
rustfs-dev/
rustfs-prod/
```

`apps` contains Argo CD `Application` resources. Each file tells Argo CD what directory to manage.

The other top-level directories contain the actual Kubernetes manifests managed by those Applications.

For example, `apps/argocd.yaml` points to `argocd`.
`redis-dev` uses a SealedSecret generated from `redis-dev/secret.local.yaml`.
`rustfs-dev` is prepared for dev object storage and exposes the public RustFS S3 endpoint through NodePort `30900` and console debug access through NodePort `30901`.
`be-dev` deploys the backend image from GHCR and exposes a debug NodePort on `30880`.

## Argo CD

Initial Argo CD installation is done manually on `tt-dev`.

After installation, create an Argo CD Application from the UI:

- Repository URL: `https://github.com/Techtaurant/gitops.git`
- Revision: `HEAD`
- Path: `apps`
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `argocd`
- Sync policy: automated with prune and self-heal

Argo CD is exposed through `argocd.techtaurant.com`. The cluster-side NodePort service is defined in `argocd/argocd-server-nodeport.yaml`.

## Sealed Secrets

`apps/sealed-secrets.yaml` installs the Sealed Secrets controller first.

After it is synced and healthy, generate sealed secrets with `kubeseal` and then register workloads such as `redis-dev` under `apps`.

## RustFS dev

`rustfs-dev` contains the RustFS StatefulSet, internal S3-compatible service on port `9000`, public S3 NodePort on `30900`, and console debug NodePort on `30901`.

To update the sealed runtime secret:

```sh
cp rustfs-dev/secret.example.yaml rustfs-dev/secret.local.yaml
# Fill RUSTFS_ACCESS_KEY, RUSTFS_SECRET_KEY, and allowed origins.
kubeseal --format yaml < rustfs-dev/secret.local.yaml > rustfs-dev/secret.yaml
```

## Backend dev

`be-dev` expects the backend image to be published as `ghcr.io/techtaurant/be-k8s-poc`.
The dev image tag is controlled by `be-dev/kustomization.yaml`, and CI updates only that file.

Before syncing `apps/be-dev.yaml`, create the sealed runtime secret:

```sh
cp be-dev/secret.example.yaml be-dev/secret.local.yaml
# Fill database, OAuth, JWT, RustFS, and monitoring values.
kubeseal --format yaml < be-dev/secret.local.yaml > be-dev/secret.yaml
```

Then add `secret.yaml` to `be-dev/kustomization.yaml` resources before committing.

## init

```sh
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f https://raw.githubusercontent.com/Techtaurant/gitops/main/argocd/argocd-server-nodeport.yaml
```
