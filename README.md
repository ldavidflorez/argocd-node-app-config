# ArgoCD Node App Config

This repository contains the ArgoCD configuration for deploying a Node.js application on Kubernetes, using a private Oracle Cloud Infrastructure (OCI) registry.

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

### 1. Configure the Secret in the Manifest

The secret is created by ArgoCD when it applies the `secret.yaml` manifest. Edit the `stringData` section in `node-app/secret.yaml` with your OCI credentials.

1. **Get your OCI credentials:**
   - **Server:** `phx.ocir.io` (adjust based on your region, e.g., `ord.ocir.io` for Chicago).
   - **Username:** `<tenancy-namespace>/<oci-username>`.
     - `tenancy-namespace`: Your tenancy namespace (e.g., `ansh81vru1zp`).
     - `oci-username`: Your OCI username.
   - **Password:** Your OCI Auth Token (same as used in the CI pipeline).

2. **Generate the auth string:**
   Run in your terminal:
   ```bash
   echo -n '<tenancy-namespace>/<oci-username>:<password>' | base64 -w 0
   ```
   Copy the output (this is the `auth` value).

3. **Update `node-app/secret.yaml`:**
   Edit the `stringData` section with your server, auth, and email:
   ```yaml
   stringData:
     .dockerconfigjson: |
       {
         "auths": {
           "phx.ocir.io": {
             "auth": "<your-base64-auth>",
             "email": "your-email@example.com"
           }
         }
       }
   ```

   **Note:** Replace placeholders with your actual values. The JSON is in plain text for easier editing. ArgoCD will create the secret in the cluster when syncing.

#### Alternative: Create Secret Directly with kubectl

If you prefer not to store the secret in the repository, you can create it directly in the cluster using `kubectl`. This is useful for environments where you want to avoid committing sensitive data.

1. Set your OCI Auth Token as an environment variable:
   ```bash
   export OCIR_PASSWORD='your-oci-auth-token'
   ```

2. Create the secret in the `node-app` namespace:
   ```bash
   kubectl create secret docker-registry ocir-secret \
     --docker-server=phx.ocir.io \
     --docker-username='ansh81vru1zp/your-oci-username' \
     --docker-password=$OCIR_PASSWORD \
     --docker-email='your-email@example.com' \
     -n node-app
   ```

   **Note:** If you use this method, remove or comment out `secret.yaml` from the repository, as ArgoCD won't need to apply it.

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