# Parallel You Kubernetes deployment

Kustomize manifests for deploying [Parallel You](https://github.com/aetheric-forge/parallel-you) to the Aetheric Forge Kubernetes cluster.

The deployment uses the immutable image:

```text
ghcr.io/aetheric-forge/parallel-you:1.0.0
```

## Layout

```text
base/          Deployment, Service, and Data Protection storage
overlays/dev/  Public development ingress and TLS configuration
```

The base targets the existing `aetheric-forge` namespace. That namespace is already covered by the cluster-wide Aetheric Forge Velero schedule, so this repository does not create a competing backup schedule.

## Prerequisites

- The `aetheric-forge` namespace exists.
- NGINX Ingress, cert-manager, and the `le-prod` ClusterIssuer are available.
- MongoDB and Redis are reachable from the namespace.
- The Keycloak client permits the callback URI:

  ```text
  https://parallel-you-dev.aethericforge.ca/signin-oidc
  ```

## Application configuration secret

Create a local file named `.env.parallel-you`. Do not commit it.

```dotenv
Keycloak__Authority=https://keycloak.example/realms/example
Keycloak__Realm=example
Keycloak__ClientId=parallel-you
Keycloak__ClientSecret=replace-me
MongoDb__Host=mongodb.example
MongoDb__Port=27017
MongoDb__Username=parallel-you
MongoDb__Password=replace-me
MongoDb__DatabaseName=parallel-you
MongoDb__AuthenticationDatabase=parallel-you
MongoDb__DirectConnection=true
Redis__Host=redis.example
Redis__Port=6379
Redis__Password=replace-me
Redis__Ssl=false
Redis__Database=0
```

Create or update the Kubernetes Secret:

```bash
kubectl create secret generic parallel-you-config \
  --namespace aetheric-forge \
  --from-env-file=.env.parallel-you \
  --dry-run=client \
  --output=yaml \
  | kubectl apply -f -
```

After changing this Secret on an existing deployment, replace the pod so the process receives the new environment:

```bash
kubectl rollout restart deployment/parallel-you \
  --namespace aetheric-forge
kubectl rollout status deployment/parallel-you \
  --namespace aetheric-forge \
  --timeout=180s
```

## GHCR pull credentials

The Deployment references `ghcr-pull-secret`. If the package is private, create it from the Docker configuration already authenticated to GHCR:

```bash
kubectl create secret generic ghcr-pull-secret \
  --namespace aetheric-forge \
  --type=kubernetes.io/dockerconfigjson \
  --from-file=.dockerconfigjson="$HOME/.docker/config.json" \
  --dry-run=client \
  --output=yaml \
  | kubectl apply -f -
```

If the package is public, remove `imagePullSecrets` from `base/deployment.yaml` or retain an existing cluster credential.

## Validate and deploy

Render the complete development overlay before applying it:

```bash
kubectl kustomize overlays/dev
```

Apply and observe the rollout:

```bash
kubectl apply -k overlays/dev
kubectl rollout status deployment/parallel-you \
  --namespace aetheric-forge \
  --timeout=180s
```

Verify the platform endpoints:

```bash
kubectl port-forward \
  --namespace aetheric-forge \
  service/parallel-you 8080:80

curl --fail http://127.0.0.1:8080/health/live
curl --fail http://127.0.0.1:8080/health/ready
```

Then test the public workflow: log in, create a capture, restart the pod, and confirm that the capture remains available.

## Updating the image

Change the tag in `overlays/dev/kustomization.yaml`, render the overlay, and commit the new immutable version. Avoid deploying `latest`.

```yaml
images:
  - name: ghcr.io/aetheric-forge/parallel-you
    newTag: 1.0.1
```

## Operational behavior

- Liveness uses `/health/live`; readiness verifies MongoDB and Redis through `/health/ready`.
- The application runs as the image's non-root UID and uses a read-only root filesystem.
- An `emptyDir` supplies writable temporary storage.
- ASP.NET Core Data Protection keys are stored on a persistent volume, preserving authentication cookies across pod replacement.
- The initial deployment has one replica. NGINX cookie affinity is configured for Blazor Server circuits before any later horizontal scaling.
- The rolling strategy never starts a surge pod, which avoids concurrent attachment problems with the single-writer Data Protection volume.
- Redis is recoverable cache state. MongoDB is durable application state and must be backed up independently if it is external to the namespace backup scope.
