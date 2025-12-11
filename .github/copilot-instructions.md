# Copilot Instructions for home-operations

This is a **Talos Linux + Flux GitOps Kubernetes cluster** for home deployment with declarative infrastructure-as-code.

## Architecture Overview

**Cluster Stack:**
- **OS:** Talos Linux (immutable, minimal) - configured via `talos/talconfig.yaml` + patches in `talos/patches/`
- **CNI:** Cilium (custom, installed via Helmfile before Flux)
- **GitOps Engine:** Flux v2 + Kustomize (multi-tiered reconciliation)
- **Secret Management:** SOPS encryption with age keys
- **Ingress:** Envoy Gateway + HTTPRoute (not Ingress)
- **DNS/Tunneling:** Cloudflare Tunnel, external-dns, cloudflared

**Critical Flow:**
1. Talos nodes bootstrap from talconfig (via talhelper generation)
2. Bootstrap helmfile deploys CNI + cert-manager + Flux (no GitOps yet)
3. Flux takes over and reconciles `kubernetes/apps/**` from Git
4. Apps use HelmReleases + Kustomize, managed per-namespace

## Key Workflows

### Bootstrap a New Cluster
```bash
task bootstrap:talos   # Generate secrets, apply machine config, bootstrap etcd
task bootstrap:apps    # Apply namespaces, install SOPS secrets, let Flux sync
```

**What happens:** `talhelper` generates machine configs from `talconfig.yaml`, applies them via `talosctl`, then `bootstrap-apps.sh` creates namespaces and installs cluster secrets so Flux can decrypt everything.

### Deploy Changes
1. Modify Kubernetes manifests in `kubernetes/apps/` (structured by namespace)
2. Commit to Git (SOPS encrypts before push)
3. Run: `task reconcile` (or wait max 1h for auto-sync)

**Secret workflow:** Encrypted with SOPS age key (`age.key`). Files matching `kubernetes/**/*.sops.yaml` and `talos/**/*.sops.yaml` are auto-encrypted on commit by `.sops.yaml` rules.

### Upgrade Talos/Kubernetes
```bash
task talos:upgrade-node IP=192.168.50.51    # Per-node Talos upgrade
task talos:upgrade-k8s                      # Kubernetes version bump
```

Modify versions in `talos/talenv.yaml` first. `talhelper` regenerates configs with new versions.

## Code Organization Patterns

### Kubernetes Structure
```
kubernetes/
├── apps/                          # All workloads, grouped by namespace
│   ├── default/
│   │   └── freshrss/
│   │       ├── ks.yaml            # Flux Kustomization (reconciles ./app)
│   │       └── app/
│   │           ├── kustomization.yaml
│   │           ├── helmrelease.yaml
│   │           ├── ocirepository.yaml
│   │           └── httproute.yaml
│   ├── cert-manager/
│   ├── observability/
│   └── [namespace]/
├── components/                    # Reusable kustomize patches
│   ├── sops/
│   └── alerts/
└── flux/
    └── cluster/ks.yaml            # Root Kustomization (patches all HelmReleases)
```

**Every app follows:** `namespace/app-name/ks.yaml` → `app/kustomization.yaml` → HelmRelease + HTTPRoute (no plain Deployments).

### Kustomization Pattern
- **Root:** `kubernetes/flux/cluster/ks.yaml` patches ALL HelmReleases globally (CRD creation, retry strategy, remediation)
- **Per-app:** `ks.yaml` is a Flux Kustomization CR (not a kustomize resource), points to `./app` directory
- **App resources:** Use `resources:` to include helmrelease.yaml, ocirepository.yaml, httproute.yaml
- **Secrets:** Components automatically apply `../../components/sops` (imports cluster-secrets)

### HelmRelease Conventions
All HelmReleases inherit patches from root Kustomization:
```yaml
install:
  crds: CreateReplace
  strategy:
    name: RetryOnFailure
upgrade:
  crds: CreateReplace
  strategy:
    name: RemediateOnFailure
```
Don't repeat these in individual HelmReleases.

## Secret Management (SOPS + age)

**Files that are encrypted:**
- `kubernetes/components/sops/cluster-secrets.sops.yaml` (key=value secrets for templating)
- `talos/talsecret.sops.yaml` (Talos machine secrets)
- Any `*.sops.yaml` in kubernetes or talos directories

**How to add a secret:**
1. Create/edit `*.sops.yaml` in plaintext
2. Run `sops --encrypt` or use `sops -i` to edit in place
3. Commit encrypted file; `.sops.yaml` rules auto-detect and encrypt

**Decryption:** Requires `age.key` file and `$SOPS_AGE_KEY_FILE` env var (set by mise in `.mise.toml`).

## Tool Dependencies & Environment

Managed by **mise** (`.mise.toml`). Key tools:
- `talos` / `talhelper` - Talos config generation and node management
- `flux` / `helm` / `helmfile` - Kubernetes deployment
- `kustomize` - YAML templating
- `sops` / `age` - Secret encryption
- `kubectl` / `cilium-cli` - Cluster inspection
- `task` - Workflow automation (Taskfile.yaml)

**Setup:**
```bash
mise trust && mise install    # Installs all pinned versions
```

## Common Patterns & Gotchas

**HTTPRoute over Ingress:** All ingress uses `envoygateway.io/v1a1 HTTPRoute`, not K8s Ingress. Example:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: freshrss
spec:
  parentRefs:
  - name: external  # Envoy Gateway instance
  hostnames:
  - "freshrss.example.com"
```

**Namespace secrets:** Apps reference `cluster-secrets` Secret for templating via `postBuild.substituteFrom`. This Secret is created by `bootstrap-apps.sh` from encrypted `.sops.yaml`.

**No plain manifests:** Every workload must be wrapped in a HelmRelease. Use Helm community charts or OCI repos (e.g., `oci://ghcr.io/...`).

**Renovate patching:** `.renovaterc.json5` uses custom managers for Helmfile and Kubernetes manifests. Renovate auto-updates chart versions, digests, and pins them. Merge strategy is automatic via CI.

**Talos patches are composable:** Global patches + role-specific patches (controller/worker) applied in order. Modify `talos/patches/` to customize nodes.

## Storage & Backup (Longhorn + Volsync)

**Status:** Work in progress with Longhorn v1 + Volsync Kopia fork

**Architecture:**
- **Longhorn v2 Data Engine:** Distributed block storage with CSI provisioning (default StorageClass)
- **Volsync Controller:** Runs backup/restore jobs on a schedule (uses perfectra1n fork for Kopia support)
- **Kopia Backend:** Repository-based snapshots (encrypted, deduplicated)
- **Snapshot Management:** VolumeSnapshot resources, with cleanup jobs for old snapshots

**Key Configuration Files:**
- `kubernetes/apps/longhorn/longhorn/app/` - Longhorn HelmRelease (v2 data engine enabled)
  - `volumeSnapshotClass.yaml` - VolumeSnapshot resource for Kopia backups
  - `snapshot-cleanup.yaml`, `snapshot-delete.yaml` - Cleanup CronJobs
- `kubernetes/apps/system/volsync/app/` - Volsync HelmRelease
  - Uses `ghcr.io/perfectra1n/volsync` image for Kopia support
  - `secret.sops.yaml` - Kopia repository encryption/credentials
- `kubernetes/apps/system/kopia/app/` - Kopia server (repository UI + API)

**Backup Workflow:**
1. **Add PVC to backup:** Create `ReplicationSource` CRD pointing to PVC
2. **Define schedule:** Specify cron schedule in ReplicationSource spec
3. **Configure backend:** Point to Kopia Secret with repository credentials
4. **Volsync runs job:** On schedule, creates VolumeSnapshot and syncs to Kopia repo
5. **Restore:** Use `ReplicationDestination` CRD to restore from Kopia snapshots

**Example ReplicationSource:**
```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: app-backup
  namespace: default
spec:
  sourcePVC: app-data
  trigger:
    schedule: "0 2 * * *"  # Daily at 2 AM
  kopia:
    repositorySecret: kopia-secret
    path: /backups/app-data
```

**Important Notes:**
- Kopia Secret must contain `.sops.yaml` encrypted credentials (password, S3 config, etc.)
- VolumeSnapshot cleanup is automated; adjust retention in snapshot-cleanup CronJob if needed
- Longhorn overprovisioning is set very high (100000%) to accommodate Volsync temp space
- Test restores in dev namespace before relying on backups for critical workloads

## Testing & Validation

```bash
task bootstrap:kubeconform    # Validates all Kubernetes manifests
flux reconcile ks flux-system # Manual Flux sync
```

Use **flux-local** for dry-run diffs before merging (see README).

## Files to Know

- `talos/talconfig.yaml` - Node IP, disks, VIP, patches (source of truth for cluster)
- `talos/talenv.yaml` - Talos & Kubernetes versions
- `bootstrap/helmfile.d/01-apps.yaml` - Bootstrap chart versions (CNI, cert-manager, Flux)
- `kubernetes/flux/cluster/ks.yaml` - Root HelmRelease patches
- `.sops.yaml` - SOPS encryption rules (age key paths)
- `.mise.toml` - Tool versions
- `Taskfile.yaml` + `.taskfiles/` - Workflow tasks (bootstrap, upgrades, reconcile)
- `kubernetes/apps/longhorn/` - Longhorn v1 (storage) + snapshot cleanup/deletion jobs
- `kubernetes/apps/system/volsync/` - Volsync controller (uses perfectra1n fork for Kopia)
- `kubernetes/apps/system/kopia/` - Kopia repository UI and metadata server
