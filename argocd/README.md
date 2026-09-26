# Argo CD bootstrap

This guide installs Argo CD outside GitOps. Argo CD must exist before it can manage the `Application` manifests in this repository.

## Prerequisites

- `kubectl` configured for the target RKE2 cluster
- Helm 3 installed
- A GitLab token with read-only repository access (`read_repository` is sufficient)
- DNS for `argo.fluffy.lst` configured when the Argo CD Ingress is enabled

The initial installation does not enable the Ingress because the current configuration depends on Traefik and the `step-ca-issuer`. Enable it after those core services are available.

## Install Argo CD

Add the official Helm repository and create the namespace:

```powershell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
```

Install Argo CD with the repository values, but keep its Ingress disabled during bootstrap:

```powershell
helm upgrade --install argocd argo/argo-cd `
  --namespace argocd `
  --version 9.4.4 `
  --values .\argocd\values.yaml `
  --set server.ingress.enabled=false `
  --wait
```

Check that the control plane is ready:

```powershell
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Retrieve the initial admin password SAVE IT FOR EMERGENCY CASES:

```powershell
$adminPassword = kubectl -n argocd get secret argocd-initial-admin-secret `
  -o jsonpath='{.data.password}' | ForEach-Object { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
$adminPassword
```

Access the UI temporarily through port-forwarding:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Open `http://localhost:8080` and log in with username `admin` and the password retrieved above. Change the admin password after the first login.

## Configure the GitLab repository credential

Do not apply `token.yml` or commit a personal access token to Git. Create the repository Secret directly in the cluster and enter the token interactively:

```powershell
$gitlabTokenSecure = Read-Host 'GitLab token' -AsSecureString
$gitlabToken = [System.Net.NetworkCredential]::new('', $gitlabTokenSecure).Password

kubectl -n argocd create secret generic gitlab-repo-secret `
  --from-literal=name='xxx' `
  --from-literal=type=git `
  --from-literal=url='xxx' `
  --from-literal=username='xxx' `
  --from-literal=password=$gitlabToken `
  --dry-run=client -o yaml | kubectl apply -f -

Remove-Variable gitlabToken, gitlabTokenSecure
```

Verify that Argo CD recognizes the Secret without printing its contents:

```powershell
kubectl -n argocd get secret gitlab-repo-secret `
  -o jsonpath='{.metadata.labels.argocd\.argoproj\.io/secret-type}'
```

The command should return `repository`.

For long-term GitOps management of this credential, use a secret-management solution such as External Secrets with Vault. Do not store the token in this repository.

## Enable the Argo CD Ingress

After Traefik, cert-manager, and the `step-ca-issuer` are working, enable the existing Ingress configuration:

```powershell
helm upgrade argocd argo/argo-cd `
  --namespace argocd `
  --version 9.4.4 `
  --values .\argocd\values.yaml `
  --wait
```

Confirm the Ingress and certificate:

```powershell
kubectl get ingress -n argocd
kubectl get certificate -n argocd
```

Argo CD itself does not require Traefik. Without Traefik, use port-forwarding or another supported exposure method instead of the Ingress configuration in `values.yaml`.

## Deploy the repository Applications

Once Argo CD and the repository credential are ready, apply the `Application` manifests manually to start GitOps management:

```powershell
kubectl apply -f *-app.yaml
```

Review each Application before applying it in a new cluster. Several applications depend on earlier core services, especially Traefik, cert-manager, Step CA, and Authentik.
