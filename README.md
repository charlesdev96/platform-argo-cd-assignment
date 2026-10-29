# Finda GitOps — Argo CD + Kubernetes configuration

This repository is the second part of the GitOps assignment. It contains Kubernetes manifests managed with Kustomize and Argo CD Applications for staging and production. The backend application stays in the separate `gitops-demo-api` repository.

## Architecture
- GitOps branch: `main`
- Argo CD watches this GitOps repository, not the backend repository.
- `overlays/staging` deploys to namespace `staging` with automatic sync, prune, and self-heal.
- `overlays/production` deploys to namespace `production` with manual sync.
- Both Argo CD Applications use the same GitOps `main` branch but different paths and independently controlled image tags.
- Argo CD itself runs in namespace `argocd`.

## Before applying
1. Replace `YOUR_GITHUB_USERNAME` in both files under `argocd/` and in the image references in `base/deployment.yaml` and both overlays.
2. Replace `newTag: latest` in each overlay with a tag that exists in GHCR. Your backend workflow publishes a commit SHA tag and `latest`; using a specific SHA is more reproducible.
3. Make the GHCR image public for the simplest local demo, if you are comfortable doing so. If private, configure image-pull credentials in both app namespaces.

## Files
```text
finda-gitops/
├── argocd/
│   ├── namespace.yaml
│   ├── staging-app.yaml
│   └── production-app.yaml
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── staging/kustomization.yaml
│   └── production/kustomization.yaml
└── README.md
```

## Step 1 — Check the cluster
In PowerShell:
```powershell
kubectl config current-context
kubectl get nodes
```
The context should be `docker-desktop` and nodes should be `Ready`.

## Step 2 — Create the Argo CD namespace from the file
From the root of this repository:
```powershell
kubectl apply -f argocd/namespace.yaml
```
This creates only the namespace, not the Argo CD installation.

## Step 3 — Install Argo CD itself
Argo CD's controllers and UI are maintained in the official upstream installation manifest. After creating the namespace, apply the official manifest:
```powershell
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Wait for pods:
```powershell
kubectl get pods -n argocd -w
```
Stop watching with Ctrl+C when the pods are ready. The upstream installation manifest installs Argo CD itself; the YAML files in this repository configure Argo CD to manage your app.

## Step 4 — Push this GitOps project to GitHub
Create a separate GitHub repository, for example `finda-gitops`, and push this folder to its `main` branch after replacing placeholders. Argo CD must be able to read the repo. Public is simplest for practice; private repos require repository credentials configured in Argo CD.

## Step 5 — Validate Kustomize output
```powershell
kubectl kustomize overlays/staging
kubectl kustomize overlays/production
```
Check that the image references and tags are correct.

## Step 6 — Register both Argo CD Applications
```powershell
kubectl apply -f argocd/staging-app.yaml
kubectl apply -f argocd/production-app.yaml
kubectl get applications -n argocd
```
Staging has automated sync, prune, and self-heal. Production has no automated sync, so you manually click Sync in Argo CD when promoting a version.

## Step 7 — Open the Argo CD dashboard
In a second PowerShell terminal:
```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open `https://localhost:8080`. A local certificate warning can be expected.

Get the initial password in PowerShell:
```powershell
$secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($secret))
```
Username: `admin`.

## Step 8 — Image promotion and assignment demonstrations
1. CI builds and pushes a backend image to GHCR.
2. CI updates the staging image tag in this GitOps repository.
3. Argo CD automatically deploys staging.
4. After testing, promote the same tested image by changing the tag in `overlays/production/kustomization.yaml`, then commit and push.
5. Argo CD detects the production configuration change but does not deploy it automatically. Manually sync the production Application.

Demonstrations:
- **Auto staging:** update and push the staging image tag; verify Argo CD syncs it.
- **Manual production:** update production's tag, push, and manually sync the production Application.
- **Self-heal:** manually alter a managed resource in namespace `staging`; Argo CD should restore the desired state.
- **Rollback:** revert the Git commit that changed the image tag and push the revert.
- Capture screenshots of the Argo CD dashboard showing both Applications.

## Notes
- This uses local Docker Desktop Kubernetes; AWS/EKS is not required.
- Never commit plaintext credentials. For runtime secrets, use Kubernetes Secrets or an appropriate secret-manager integration.
- The service is `ClusterIP`. Use `kubectl port-forward` to test the API from your host.
