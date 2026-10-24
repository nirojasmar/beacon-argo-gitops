# Beacon Argo CD manifests

This repository contains a Kustomize base and OCI overlay for the Beacon workload. Sync the Argo CD Application in `argocd-application.yaml` after configuring the repository URL, GitHub owner/image, ingress host, Kubernetes API CIDR, and out-of-band Secret.

## Before deploying

1. In `overlays/oci/kustomization.yaml`, replace `REPLACE_WITH_GITHUB_OWNER` with the GitHub owner that publishes the Beacon image. The `latest` tag is produced by the Beacon repository's `main` workflow; use a `git-<short-sha>` tag for immutable deployments.
2. Edit `overlays/oci/config.env`:
   - Set `INGRESS_HOST` to `beacon.<PUBLIC_IP>.nip.io` (for example, `beacon.203-0-113-10.nip.io`).
   - Set `K3S_API_CIDR` to the control-plane node IP/CIDR used by scheduler pods to reach Kubernetes on TCP 6443. The checked-in `0.0.0.0/0` is only a placeholder for the port-restricted policy and must be narrowed before production.
3. Change the GitOps repository URL in `argocd-application.yaml`.
4. Create the `beacon-secrets` Secret outside Git. Generate independent values and apply from a protected workstation/secret manager:

   ```sh
   kubectl create namespace beacon
   kubectl -n beacon create secret generic beacon-secrets \
     --from-literal=NATS_PASSWORD="$(openssl rand -hex 32)" \
     --from-literal=BEACON_TOKEN="$(openssl rand -hex 32)" \
     --from-literal=WEBHOOK_SECRET="$(openssl rand -hex 32)"
   ```

   The `base/secret-example.yaml` file is a documentation template and is deliberately not included in any Kustomize resource list. Do not commit generated Secret YAML or replace the placeholders with real credentials.
5. Ensure the cluster has the `local-path` StorageClass and Traefik IngressClass. The Traefik controller is expected in `kube-system` or `traefik-system`; adapt the ingress NetworkPolicy namespace selectors if it runs elsewhere.
6. The scheduler egress policy permits only TCP 6443, but currently has a broad placeholder CIDR. Set the actual control-plane node CIDR before sync.

## Layout and behavior

- `base/` creates the `beacon` namespace, config, dedicated service accounts, scheduler Lease RBAC, resource quota/limits, default-deny NetworkPolicies, NATS JetStream StatefulSet and PVC, API, scheduler, checker, notifier and digest CronJob.
- Argo CD sync waves are 0 for namespace/RBAC/config/quota/NATS, 1 for long-lived Beacon services and ingress, and 2 for the digest CronJob.
- Only the scheduler ServiceAccount mounts a Kubernetes API token. NATS and all other workloads use dedicated accounts with token automount disabled.
- All containers run as UID/GID 65532 with read-only root filesystems, dropped capabilities, no privilege escalation, and the RuntimeDefault seccomp profile. API shutdown uses the binary's `sleep` subcommand for its five-second preStop delay; no shell is needed in the distroless image.
- API serves the public status page and SSE through Traefik; its 15-second SSE heartbeat and `X-Accel-Buffering: no` response header keep long-lived connections active.
- Checker, notifier, and digest egress is allowed for targets/webhooks. Scheduler API egress is restricted to TCP 6443. NATS accepts client traffic only from the Beacon namespace.

Build and inspect the generated manifests locally with:

```sh
kustomize build overlays/oci
```
