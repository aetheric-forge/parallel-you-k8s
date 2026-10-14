# Parallel You Kubernetes deployment

Deploys the standalone [Parallel You host](https://github.com/aetheric-forge/parallel-you) at **https://py.int.aethericforge.ca** in the `aetheric-forge` namespace on context `blackcircuit-ca-platform-prod`.

## Layout

- `base/`: standalone host, ClusterIP Service, service account, and persistent authentication-key storage.
- `overlays/prod/`: private production ingress and pinned image.
- `argocd/application.yaml`: automated GitOps deployment of `overlays/prod` from main.
- `overlays/dev/`: historical dev ingress, retained for reference. Its old cluster is not the deployment target; a separate dev deployment requires its own consumer artifact and binding overrides.

The production ingress uses `nginx-private`, the `step-ca-int-ca` ClusterIssuer, and internal external-dns. Access requires the private network and trust in the internal CA. No public ingress is created. The namespace is already covered by the platform Velero backup schedule.

## Published image

The Linux amd64 image comes from merged Parallel You source commit `9c83e03affe700c07f526e0b5908d655ccea07e0` (PR #16), with its pinned runtime and primitives dependencies. The image is `ghcr.io/aetheric-forge/parallel-you:sha-9c83e03@sha256:b8f81991ec0adb278320fe9365aa934a152ae5e69415732c3164eb251636c755`. The overlay pins this immutable digest. See `overlays/prod/kustomization.yaml` for the exact tag and digest.

## Consumer configuration

The host mounts only the scoped `parallel-you-consumer` Secret, under key `configuration.json`. It uses the existing provisioned institution:

- Deployment ID: `f7448b244efacc5bf6b2decc4a7b0879eeb971dbf87e79657eb6735e9445318e`.
- Environment: `production`.
- Plan ID: `41926c2300f105b25f5d66e77c33a920ae58ed0fa177485ddb5a867f170ffe2a`.
- Library database: `parallel-you`, collection `knowledge`.
- Parent Workbench: the existing scoped Redis grant.

The private host artifact is a projection of `aethericforge-parallel-you-consumer`: the Library, Workbench, institution identity and plan are preserved; only the browser client differs. It is delivered separately so the public host keeps its own OIDC settings. The Secret annotations record the source artifact, deployment, plan and host origin.

The dedicated browser client is `parallel-you-private-web-f7448b244efacc5bf6b2decc`, in realm `int.aethericforge.ca`, using authority `https://sso.aethericforge.ca`. It uses confidential authorization-code flow and S256 PKCE, with service accounts and direct access grants disabled. Its provisioning-owner marker is the institution deployment ID, and its host-origin marker is `https://py.int.aethericforge.ca`.

Exact browser settings:

- Redirect URI: `https://py.int.aethericforge.ca/signin-oidc`.
- Logout callback: `https://py.int.aethericforge.ca/signout-callback-oidc`.
- Web origin: `https://py.int.aethericforge.ca`.

Deliver the version-1 artifact through provisioning/operator tooling. Do not commit credential values. For an existing private artifact file:

```bash
kubectl --context blackcircuit-ca-platform-prod create secret generic parallel-you-consumer \
  --namespace aetheric-forge --from-file=configuration.json=/private/path/configuration.json \
  --dry-run=client --output=yaml \
  | kubectl --context blackcircuit-ca-platform-prod apply --server-side \
      --field-manager=parallel-you-host-delivery -f -
```

Before replacing an existing artifact, verify its deployment, environment, plan and private browser client. Refresh this host projection after provisioning rotates upstream Library/Workbench credentials; update the manifest plan pin when applying a newly reviewed plan, then restart the Deployment. The public artifact must retain its public browser client.

The host does not mount the admin root credential store or encryption key. It does not recreate the Library or migrate existing personal records. Its service-account token is disabled.

## Persistent storage and trust

`parallel-you-data-protection` supplies a 1Gi persistent volume mounted at `/data/keys`. Preserve it across pod replacement. `ParallelYou:ProtectionKeyDirectory` points there. The non-root container runs with a read-only root filesystem; `/tmp` uses an `emptyDir`.

The host mounts `step-ca-root-ca` and sets `SSL_CERT_FILE` for platform trust. Rolling updates have no surge pod, avoiding concurrent attachment of the single-writer authentication-key volume. Blazor Server uses ingress cookie affinity.

## Validate and deploy

The namespace, consumer Secret, CA ConfigMap, private ingress controller and internal certificate issuer must exist before activating the Application. The image package is public; no pull credential is required.

```bash
kubectl kustomize overlays/prod
kubectl --context blackcircuit-ca-platform-prod apply --dry-run=server -k overlays/prod
kubectl --context blackcircuit-ca-platform-prod apply -f argocd/application.yaml
kubectl --context blackcircuit-ca-platform-prod --namespace aetheric-forge \
  rollout status deployment/parallel-you --timeout=180s
```

The GitOps Application retains automatic sync, pruning and self-healing. Changes must be committed to main before applying the Application. It manages this host's manifests, not its externally delivered consumer Secret.

## Verify

From the private network:

```bash
curl --fail https://py.int.aethericforge.ca/health/live
curl --fail https://py.int.aethericforge.ca/health/ready
```

`/` redirects to `/parallel-you`. The landing page allows anonymous access; all seven working pages require sign-in. Readiness checks the actual Mongo database and parent Redis connection. Verify that login reaches the configured Keycloak realm without callback errors. A signed-in user can then verify that existing personal records are available.

## Update or roll back

Update the image tag and digest in the production overlay and base, render and review the manifests, and merge the change. Do not use `latest`. Roll back by reverting the Git image pin to a compatible standalone-host revision. The old v1.0.0 container uses a different configuration contract and is not a compatible image-only rollback.

After a consumer configuration update:

```bash
kubectl --context blackcircuit-ca-platform-prod --namespace aetheric-forge \
  rollout restart deployment/parallel-you
```
