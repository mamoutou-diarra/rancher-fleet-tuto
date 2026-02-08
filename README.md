# Rancher Fleet Hands-On Tutorial

A Quick tutorial on how to manage a Kubernetes cluster with Rancher Fleet.

## Overview

This hands-on session covers:
- Installing Fleet via Helm
- Creating GitRepo objects to watch and apply manifests
- Setting up Fleet bundles and fleet.yaml configurations
- Managing deployments through Git
- Observing Fleet's reconciliation behavior

## Prerequisites

- A Kubernetes cluster (v1.21+)
- `kubectl` configured and connected to your cluster
- `helm` CLI installed
- Git repository access

## Part 1: Install Fleet

Install Fleet using the Helm chart:

```bash
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
helm repo update
helm install fleet-crd fleet/fleet-crd -n cattle-fleet-system --create-namespace
helm install fleet fleet/fleet -n cattle-fleet-system
```

Verify installation:

```bash
kubectl get pods -n cattle-fleet-system
```

## Part 2: Create Fleet GitRepo

Create a GitRepo resource to watch this repository:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: fleet-tutorial
  namespace: fleet-local
spec:
  repo: https://github.com/<your-username>/rancher-fleet-tuto
  branch: main
  paths:
  - manifests
```

Apply it:

```bash
kubectl apply -f gitrepo.yaml
```

Check status:

```bash
kubectl get gitrepo -n fleet-local
kubectl get bundles -n fleet-local
```

## Part 3: Repository Structure

Organize your manifests in the following structure:

```
manifests/
├── nginx-deployment/
│   ├── deployment.yaml
│   └── fleet.yaml
├── nginx-service/
│   ├── service.yaml
│   └── fleet.yaml
├── nginx-ingress/
│   ├── ingress.yaml
│   └── fleet.yaml
└── cert-manager/
    └── fleet.yaml
```

Each `fleet.yaml` defines how the bundle should be created and huw it should run (precedence tree):


## Part 4: Deploy and Update Workloads

### Example 1: Nginx with Service and Ingress

Create `manifests/nginx-deployment/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

Create `manifests/nginx-deployment/fleet.yaml`:

```yaml
namespace: default
```

Create `manifests/nginx-service/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Create `manifests/nginx-service/fleet.yaml`:

```yaml
namespace: default
```

Create `manifests/nginx-ingress/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Create `manifests/nginx-ingress/fleet.yaml`:

```yaml
namespace: default
```

### Example 2: Deploying Helm Charts with Fleet

Fleet can deploy Helm charts directly. Create `manifests/cert-manager/fleet.yaml`:

```yaml
namespace: cert-manager
helm:
  chart: cert-manager
  repo: https://charts.jetstack.io
  version: v1.13.3
  releaseName: cert-manager
  values:
    installCRDs: true
```

This deploys the cert-manager Helm chart without needing to store any Kubernetes manifests in your repository.

Commit and push:

```bash
git add manifests/
git commit -m "Add nginx and cert-manager deployments"
git push
```

Watch Fleet sync the changes:

```bash
kubectl get bundles -n fleet-local -w
kubectl get deployments -n default
```

### Update the Deployment

Modify `manifests/nginx/deployment.yaml` (e.g., change replicas to 3), commit, and push. Observe Fleet automatically applying the changes.

For the Helm example, update the `fleet.yaml` values:

```yaml
namespace: cert-manager
helm:
  chart: cert-manager
  repo: https://charts.jetstack.io
  version: v1.13.3
  releaseName: cert-manager
  values:
    installCRDs: true
    replicaCount: 2  # Add custom values
```

## Part 5: Test Reconciliation

Delete a resource manually:

```bash
kubectl delete deployment nginx -n default
```

Watch Fleet recreate it:

```bash
kubectl get deployments -n default -w
```

Fleet will detect the drift and reconcile back to the desired state from Git.

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
