# Rancher Fleet Hands-On Tutorial

A practical guide to managing Kubernetes clusters with Rancher Fleet.

## Prerequisites

- Kubernetes cluster (v1.21+)
- `kubectl` and `helm` CLI installed
- Git repository access

## Step 1: Install Fleet

```bash
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
helm repo update
helm install fleet-crd fleet/fleet-crd -n cattle-fleet-system --create-namespace
helm install fleet fleet/fleet -n cattle-fleet-system
kubectl get pods -n cattle-fleet-system
```

## Step 2: Create Fleet GitRepo

Create and apply `gitrepo.yaml`:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: fleet-tutorial
  namespace: fleet-local
spec:
  repo: https://github.com/mamoutou-diarra/rancher-fleet-tuto
  branch: main
  paths:
  - manifests
```

```bash
kubectl apply -f gitrepo.yaml
kubectl get gitrepo -n fleet-local
kubectl get bundles -n fleet-local
```

## Step 3: Repository Structure

The `manifests/` directory contains Fleet bundles:

- **nginx/** - Nginx deployment (1 replica) with `correctDrift` enabled
- **nginx-service/** - ClusterIP service for Nginx
- **nginx-ingress/** - Ingress for nginx
- **cert-manager/** - Helm chart deployment (cert-manager v1.13.3)

Each directory has a `fleet.yaml` defining deployment configuration.

**Note:** The nginx bundle has `correctDrift: enabled: true` in its `fleet.yaml`, enabling automatic reconciliation when local changes are detected.

## Step 4: Watch Fleet Sync

Fleet automatically syncs changes from Git:

```bash
kubectl get bundles -n fleet-local -w
kubectl get deployments -n nginx
kubectl get svc -n nginx
```

## Step 5: Update Manifests

Modify any manifest (e.g., change nginx replicas to 2), commit and push:

```bash
git add manifests/
git commit -m "Update nginx replicas"
git push
```

Fleet will automatically detect and apply changes.

## Step 6: Test Drift Correction

The nginx bundle has drift correction enabled. Test it by modifying the deployment locally:

```bash
kubectl scale deployment nginx -n nginx --replicas=5
kubectl get deployment nginx -n nginx
```

Fleet will detect the drift and automatically restore replicas back to 1 (the desired state in Git).

Alternatively, delete the resource:

```bash
kubectl delete deployment nginx -n nginx
kubectl get deployments -n nginx -w
```

Fleet recreates the resource from Git.

## Useful Commands

```bash
# View GitRepo status
kubectl describe gitrepo fleet-tutorial -n fleet-local

# Check bundle details
kubectl get bundles -n fleet-local
kubectl describe bundle <bundle-name> -n fleet-local

# View Fleet logs
kubectl logs -n cattle-fleet-system -l app=fleet-controller

# Force sync
kubectl annotate gitrepo fleet-tutorial -n fleet-local force-sync="$(date +%s)" --overwrite
```

## Resources

- [Fleet Documentation](https://fleet.rancher.io/)
- [Fleet GitHub Repository](https://github.com/rancher/fleet)
- [GitOps Best Practices](https://opengitops.dev/)
