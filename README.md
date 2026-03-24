# MongoDB & Mongo Express on Kubernetes

A Kubernetes deployment that sets up a **MongoDB** database with **Mongo Express** — a web-based admin UI for managing MongoDB.

## Architecture

```
┌──────────────┐       ┌─────────────────────┐
│ mongo-secret │──────▶│  mongodb-deployment  │
│ (credentials)│──┐    │  image: mongo        │
└──────────────┘  │    │  port: 27017         │
                  │    └──────────┬───────────┘
                  │               │
                  │    ┌──────────▼───────────┐
                  │    │   mongodb-service     │
                  │    │   ClusterIP: 27017    │
                  │    └──────────┬───────────┘
                  │               │ (via ConfigMap)
                  │    ┌──────────▼───────────┐
                  └───▶│ mongo-express-deploy  │
                       │ image: mongo-express  │
                       │ port: 8081            │
                       └──────────┬───────────┘
                                  │
                       ┌──────────▼───────────┐
                       │ mongo-express-service │
                       │ NodePort: 30100       │
                       └──────────────────────┘
```

## Components

| File | Resource(s) | Description |
|------|-------------|-------------|
| `mongo-secret.yml` | Secret | Stores MongoDB root credentials (base64-encoded) |
| `mongo-configmap.yml` | ConfigMap | Stores the MongoDB internal service URL |
| `mongo-deployment.yml` | Deployment + Service | MongoDB database (single replica, ClusterIP) |
| `mongo-express.yml` | Deployment + Service | Mongo Express web UI (single replica, NodePort) |

## Prerequisites

- A running Kubernetes cluster (e.g., [Minikube](https://minikube.sigs.k8s.io/docs/start/), [kind](https://kind.sigs.k8s.io/), or a cloud provider)
- `kubectl` installed and configured to connect to your cluster

## Setup Instructions

### 1. Create the Secret

Before deploying, you need to create `mongo-secret.yml` with your own credentials. Base64-encode your desired username and password:

```bash
echo -n 'your-username' | base64
echo -n 'your-password' | base64
```

Then create `mongo-secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongodb-root-username: <base64-encoded-username>
  mongodb-root-password: <base64-encoded-password>
```

### 2. Deploy Resources (in order)

Resources must be applied in a specific order so that dependencies (Secret, ConfigMap) exist before the Deployments reference them.

```bash
# Step 1: Create the secret (credentials)
kubectl apply -f mongo-secret.yml

# Step 2: Create the configmap (database URL)
kubectl apply -f mongo-configmap.yml

# Step 3: Deploy MongoDB
kubectl apply -f mongo-deployment.yml

# Step 4: Deploy Mongo Express
kubectl apply -f mongo-express.yml
```

### 3. Verify Everything Is Running

```bash
# Check all resources
kubectl get all

# Check pods are in Running state
kubectl get pods

# Check services
kubectl get svc
```

### 4. Access Mongo Express

**Using Minikube:**

```bash
minikube service mongo-express-service
```

**Using any other cluster:**

Open your browser and navigate to:

```
http://<node-ip>:30100
```

To find your node IP:

```bash
kubectl get nodes -o wide
```

## Useful Commands

```bash
# View pod logs
kubectl logs <pod-name>

# Describe a pod for debugging
kubectl describe pod <pod-name>

# Delete all resources
kubectl delete -f mongo-express.yml
kubectl delete -f mongo-deployment.yml
kubectl delete -f mongo-configmap.yml
kubectl delete -f mongo-secret.yml
```

## Security Notes

- **Never commit secrets** with real credentials to version control. Use a secrets manager (e.g., HashiCorp Vault, Sealed Secrets) for production.
- The `mongo-secret.yml` file in this repo uses **placeholder values** — replace them with your own base64-encoded credentials before deploying.
- Consider adding `mongo-secret.yml` to `.gitignore` to prevent accidental credential leaks.
