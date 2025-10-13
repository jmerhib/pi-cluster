# Pi-Cluster (Homelab GitOps)

A reproducible Kubernetes homelab, managed end-to-end with GitOps. My cluster is 2x HP Elitedesk 800's and 2x Raspberry Pi 4's all running on Talos Linux. This repo holds cluster config, platform controllers, and app manifests, so you can bootstrap, upgrade, and recover the entire stack from source.

> **Highlights**
> - **GitOps with Flux** — declarative everything, automatic drift remediation
> - **Layered Kustomize** — clear separation of infra controllers vs. apps
> - **Observability** — Prometheus + Grafana
> - **Home services** — privacy-friendly apps and utilities
> - **Secure secrets** — SOPS + age-encrypted credentials in-repo, also everything runs through cloudflare tunnels, so nothing on my home network is exposed.
> - **Automated updates** — Renovate PRs for Helm charts and images

---

## Repository layout

```
.
├─ apps/                     # Application HelmReleases / Kustomizations
├─ clusters/
│  └─ staging/               # Cluster entrypoint(s): Flux Kustomizations
├─ infrastructure/
│  └─ controllers/           # Platform controllers (ingress, certs, DNS, storage)
├─ monitoring/               # Monitoring stack (dashboards, rules, alerts)
└─ renovate.json             # Renovate config for automated dependency updates
```

Each folder is intentionally small and composable:
- **infrastructure**: (openebs for automated pv creation, renovate for updates)
- **apps**: end-user services (e.g., Pi-hole, linkding, audiobookshelf).
- **monitoring**: (grafana)
- **clusters/**: per-cluster overlays (e.g., `staging/`) wiring the pieces together via Flux `Kustomization`s.

---

## Architecture (at a glance)

```
[ Users ] ──> [ Ingress Controller ] ─> [ Apps (ClusterIP/HTTP) ]
                    │
                    ├── Cert-Manager (ACME/Local CA)
                    ├── External-DNS (optional)
                    └── Grafana/Prometheus/Alertmanager/Loki

[ Nodes: 2x EliteDesk 800's, 2x Raspberry Pi 4 ]
- K8s distro: Talos-managed kubelet (ARM/x86 mix OK)
- Storage: local-path using OpenEBS)
```

---

## Getting started

### Prerequisites
- A Kubernetes cluster (ARM Pis and/or x86 mini PCs)
- `kubectl`, `helm`, `flux`, `age`, `sops`, and `kustomize`
- A GitHub Personal Access Token (if using `flux bootstrap github`)


### 1) Generate an age key and configure SOPS
```bash
age-keygen -o age.key
mkdir -p .sops && mv age.key .sops/
AGE_PUBLIC_KEY=$(grep public .sops/age.key | awk '{print $3}')
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*(secrets|secret|credentials)\.ya?ml$
    age: ["$AGE_PUBLIC_KEY"]
EOF
```

### 2) Encrypt any secret manifests
```bash
# Example
sops --encrypt --in-place apps/<app>/secret.enc.yaml
```

### 3) Bootstrap Flux
```bash
# Option A: GitHub repo bootstrap
flux bootstrap github \
  --owner=jmerhib \
  --repository=pi-cluster \
  --branch=main \
  --path=clusters/staging \
  --personal

```

### 4) Verify reconciliation
```bash
flux get kustomizations
kubectl get pods -A
```

---

## Operations

### Secrets
- Store encrypted files alongside manifests (SOPS + age).
- Rotate keys by re-encrypting and committing. Flux can use `SOPS_AGE_KEY_FILE` or a Kubernetes Secret containing your private key.

### Upgrades
- ​Renovate opens PRs bumping Helm chart versions and container tags.
- Merge → Flux reconciles → rollout happens automatically with health checks.

### Storage
- OpenEBS (`local-path`). For this I just used a USB on node01. Anything that needs storage gets assigned to that node. PVC's are defined in the storage.yaml's and OpenEBS automatically provisions the PVs

---

## What I learned

- This was my first time building a homelab, so I've learned a lot. At first I used k3s, but eventually migrated to using Talos Linux as I feel it just ran smoother.
- Flux is really nice to keep everything distributed and ensure there aren't any snowflakes in the cluster that get lost if a pod restarts or something.
- Had never used a provisioner before for automatically creating the PVs, was a bit of a headache to get it working, but now that I did, it's really nice.
- 

---

## Local dev & troubleshooting

```bash
# See what Flux will apply (dry-run a directory)
kustomize build clusters/staging | kubectl diff -f -

# Inspect a Kustomization
flux get kustomization cluster-staging -o yaml

# Check app rollout
kubectl -n <ns> get deploy,sts,po,svc,ing
kubectl -n <ns> describe deploy/<name>
kubectl -n <ns> logs deploy/<name> -f
```

---

## Roadmap / Ideas

- Want to move to using certmanager to manage certificates
- Start using vault to rotate secrets
- Build it using terraform on AWS


