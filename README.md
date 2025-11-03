# ArgoCD Node App Config

This repository contains the ArgoCD configuration for deploying a Node.js application on Kubernetes, using a private Oracle Cloud Infrastructure OCI registry.

## Repository Structure

- `application.yaml`: ArgoCD application manifest.
- `node-app/`: Directory with Kubernetes manifests.
  - `deployment.yaml`: Application deployment.
  - `service.yaml`: LoadBalancer service.
  - `secret.yaml`: Secret for authentication to the private OCI registry.

## Prerequisites

- A Kubernetes cluster with ArgoCD installed.
- Access to a private OCI registry (OCIR).
- OCI credentials: Auth Token, Tenancy OCID, User OCID, etc.
- Docker installed locally to generate authentication config.

## Manifest Configuration

### 1. Create the Namespace

Create the `node-app` namespace manually:

kubectl create namespace node-app

**Note:** The application.yaml has `CreateNamespace=false`, so ArgoCD will not create the namespace automatically.

### 2. Create the Secret for OCI Registry

The secret is created manually in the cluster using `kubectl`. This avoids storing sensitive data in the repository.

Run the following command in your terminal (replace placeholders with your actual OCI credentials):

export OCI_AUTH_TOKEN='your_auth_token_here'

kubectl create secret docker-registry ocir-secret --docker-server=ord.ocir.io --docker-username=axtid53peedq/luis.florez@blackpool.la --docker-password=$OCI_AUTH_TOKEN --docker-email=luis.florez@blackpool.la -n node-app

**Note:** This creates the secret in the `node-app` namespace. Ensure ArgoCD has permissions to access it. The deployment.yaml references this secret via `imagePullSecrets`.

### 2. Verify the Deployment

The `deployment.yaml` already includes `imagePullSecrets` to use the configured secret. The image is automatically updated by the CI pipeline (see `.github/workflows/ci.yml` in the app repository).

### 3. Service

The `service.yaml` is configured as LoadBalancer to expose the app on port 3000.

## Deployment with ArgoCD

1. Ensure ArgoCD is installed on your cluster.
2. Create the application in ArgoCD using `application.yaml`:
   - The source points to this repo.
   - The destination is your cluster and `node-app` namespace.
3. ArgoCD will create the namespace automatically (`CreateNamespace=true`).
4. Monitor the deployment in the ArgoCD UI.

## CI/CD Pipeline

The pipeline in the app repository (not this one) builds the image, pushes it to OCIR, and triggers an event for ArgoCD to update the deployment with the new image tag.


## Troubleshooting

- **"image can't be pulled" error**: Verify the secret is correctly configured and OCI credentials are valid.
- **Namespace does not exist**: ArgoCD should create it, but check permissions.
- **Image not updating**: Ensure the CI pipeline is working and sending the correct event.

For more details, refer to [ArgoCD documentation](https://argo-cd.readthedocs.io/) and [OCI Registry documentation](https://docs.oracle.com/en-us/iaas/Content/Registry/home.htm).