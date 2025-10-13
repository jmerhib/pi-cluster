# Pi-Cluster (Homelab GitOps)

A reproducible Kubernetes homelab, managed end-to-end with GitOps. My cluster is 2x HP Elitedesk 800's and 2x Raspberry Pi 4's all running on Talos Linux. This repo holds cluster config, platform controllers, and app manifests, so you can bootstrap, upgrade, and recover the entire stack from source.

> **Highlights**
> - **GitOps with Flux** — declarative everything, automatic drift remediation
> - **Layered Kustomize** — clear separation of infra controllers vs. apps
> - **Observability** — Prometheus + Grafana (dashboards, alerts, logs)
> - **Home services** — privacy-friendly apps and utilities
> - **Secure secrets** — SOPS + age-encrypted credentials in-repo
> - **Automated updates** — Renovate PRs for Helm charts and images

---

## 📦 Repository layout

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
- **infrastructure/controllers**: ingress controller, cert-manager, external-dns, storage CSI/LVS, etc.
- **apps**: end-user services (e.g., Pi-hole DNS sinkhole, bookmarks, media, dashboards).
- **monitoring**: kube-prometheus-stack, logging pipeline, SLO / alert rules.
- **clusters/**: per-cluster overlays (e.g., `staging/`) wiring the pieces together via Flux `Kustomization`s.

> Tip: keep cluster-specific values under `clusters/<name>/` and re-use the same app/controller definitions across environments with `values/<env>.yaml`.

---

## 🏗️ Architecture (at a glance)

```
[ Users ] ──> [ Ingress Controller ] ─> [ Apps (ClusterIP/HTTP) ]
                    │
                    ├── Cert-Manager (ACME/Local CA)
                    ├── External-DNS (optional)
                    └── Grafana/Prometheus/Alertmanager/Loki

[ Nodes: Raspberry Pi 4 + optional x86 minis ]
- K8s distro: k3s or Talos-managed kubelet (ARM/x86 mix OK)
- Storage: local-path, NFS, or lightweight SDS (e.g., OpenEBS)
```

---

## 🚀 Getting started

### Prerequisites
- A Kubernetes cluster (ARM Pis and/or x86 mini PCs)
- `kubectl`, `helm`, `flux`, `age`, `sops`, and `kustomize`
- A GitHub Personal Access Token (if using `flux bootstrap github`)
- (Optional) DNS credentials for `external-dns`
- (Optional) object or NFS storage for persistent apps

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

# Option B: Manual install, then point Flux at this repo path
flux install
flux create source git pi-cluster \
  --url=https://github.com/jmerhib/pi-cluster \
  --branch=main
flux create kustomization cluster-staging \
  --source=GitRepository/pi-cluster \
  --path="./clusters/staging" \
  --prune=true --interval=1m
```

### 4) Verify reconciliation
```bash
flux get kustomizations
kubectl get pods -A
kubectl get httproutes,ingresses -A  # depending on your ingress stack
```

---

## 🔧 Operations

### Secrets
- Store encrypted files alongside manifests (SOPS + age).
- Rotate keys by re-encrypting and committing; Flux can use `SOPS_AGE_KEY_FILE` or a Kubernetes Secret containing your private key.

### Upgrades
- Dependabot/​Renovate opens PRs bumping Helm chart versions and container tags.
- Merge → Flux reconciles → rollout happens automatically with health checks.

### Storage
- Start simple (`local-path`) or NFS for shared data.
- For node-pinned disks, document PVC ↔ node affinity and your reclaim policy.

### Ingress & TLS
- Use a single ingress controller and DNS zone.
- Cert-Manager issues certificates (ACME HTTP-01 / DNS-01). Keep a staging issuer for dry runs.

---

## 📚 Apps & platform components (examples)

_This repo is structured to host:_
- **Ingress** (e.g., ingress-nginx or Traefik) with HTTP→HTTPS, HSTS
- **Cert-Manager** (ACME issuers, ClusterIssuer)
- **External-DNS** (optional, updates DNS records)
- **Monitoring** (kube-prometheus-stack, Grafana dashboards)
- **Logging** (e.g., Loki + Promtail)
- **Home services** like:
  - Pi-hole (DNS filtering)
  - Bookmarking (Linkding)
  - Audiobookshelf / media utilities
  - Homepage / portal
  - Cloudflared tunnel (remote access without port-forwarding)

> Enable/disable components per environment in `clusters/<env>/kustomization.yaml`.

---

## 🧠 What I learned

- **GitOps discipline beats manual fiddling**: keeping every change declarative made rollbacks and rebuilds painless.
- **ARM/x86 heterogeneity**: mixed architectures work well if images/charts are multi-arch; pin images only when necessary.
- **Networking matters**: ingress + DNS + TLS is where most usability wins come from; bake these in early.
- **Storage trade-offs**: local SSDs are fast but cluster-local; shared backends (NFS/S3-like) simplify migrations at the cost of latency.
- **Secrets ergonomics**: SOPS + age is low-friction; keep the private key out of the repo and document recovery steps.
- **Automated updates**: Renovate’s small, frequent PRs reduce risk and keep you honest on patching.

---

## 🧪 Local dev & troubleshooting

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

## 🔐 Backups & recovery (quick note)

- Back up persistent volumes (NFS snapshots or rsync), and export Grafana dashboards/Alertmanager config in-repo.
- Keep `age` private key safe; store a sealed copy outside the cluster.
- Document a **cold start** path: boot K8s → install Flux → point to `clusters/<env>`.

---

## 🗺️ Roadmap / Ideas

- Harden network policies (default-deny + app-specific allows)
- SLOs + alert tuning for homelab noise reduction
- GitHub Actions for pre-merge `kustomize`/`helm` validation
- Add `production/` overlay and secrets separation
- Optional CSI migration for dynamic PVs

---

## 🤝 Contributing

PRs and issues welcome! For larger changes, open a discussion/issue first so we can agree on direction.

---

## 📄 License

MIT (see `LICENSE`).
