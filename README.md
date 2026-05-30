# Techtaurant GitOps

GitOps manifests for the Techtaurant k3s cluster.

## Structure

```text
apps/
  argocd.yaml            # Argo CD Application manifests
  redis-dev.yaml
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

## init

```sh
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f https://raw.githubusercontent.com/Techtaurant/gitops/main/argocd/argocd-server-nodeport.yaml
```
