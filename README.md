# Homelab: 3-node Talos + Cilium + Flux

**New deployment? Start with [`DEPLOYMENT.md`](DEPLOYMENT.md)** — a
sequenced checklist from bare metal through every app being verified
reachable. Everything below is reference detail that checklist links out
to, not a replacement for it.

3x Lenovo ThinkCentre M710 (8GB RAM each), Talos Linux.

- PC1: control plane only (dedicated — no workloads scheduled here)
- PC2, PC3: workers (Jellyfin, Immich, Traefik, Prometheus/Grafana)
- Cilium replaces kube-proxy + Flannel (eBPF networking, L2-announced LoadBalancer IPs)
- Traefik: ingress, exposed via a Cilium-announced LoadBalancer IP
- Flux CD: GitOps, reconciles everything in `infrastructure/` and `apps/`
- Storage: NFS back to your NAS via `nfs-subdir-external-provisioner`

## Diagrams
- [`docs/traffic-flow.html`](docs/traffic-flow.html) — how a request reaches
  an app: Tailscale vs LAN/Traefik, and which apps sit behind Authelia
- [`docs/homelab-layout.html`](docs/homelab-layout.html) — the physical
  hardware and network layout (UniFi Cloud Gateway, UniFi Express, the 3
  nodes, the NAS)

Both are self-contained HTML files — open directly in a browser, no server
needed.

## Bootstrap order (chicken-and-egg: cluster needs a CNI before Flux can run)

1. **Get the worker image** (PC2 + PC3 only) from Talos Image Factory —
   bakes in both GPU support (Quick Sync, for Jellyfin/Immich transcoding)
   and NFS client kernel modules (required for any PVC mount to work at
   all, given Talos's minimal image doesn't include this by default):
   ```
   curl -X POST --data-binary @talos/extensions/worker-extensions.yaml \
     https://factory.talos.dev/schematics
   ```
   This returns a schematic ID — use it as the install image for worker nodes,
   e.g. `factory.talos.dev/metal-installer/<schematic-id>:v1.9.x`.
   PC1 (control plane) can use the stock `metal-installer` image — no GPU needed there.

2. **Generate configs**, applying the shared Cilium patch to all 3 nodes,
   plus the role-specific patch:
   ```
   talosctl gen config homelab https://<PC1-IP>:6443 \
     --config-patch @talos/patches/cluster-network.yaml \
     --config-patch-control-plane @talos/patches/controlplane.yaml \
     --config-patch-worker @talos/patches/worker.yaml
   ```

3. **Apply configs** to each node (`talosctl apply-config --insecure -n <ip> -f controlplane.yaml` / `worker.yaml`), then **bootstrap** on PC1 only:
   ```
   talosctl bootstrap -n <PC1-IP>
   ```
   Nodes will sit `NotReady` — expected, there's no CNI yet.

4. **Manually install Cilium once** (Flux can't run without networking):
   ```
   helm repo add cilium https://helm.cilium.io/
   helm repo update
   helm install cilium cilium/cilium -n kube-system -f infrastructure/cilium/values.yaml
   ```
   Nodes should flip to `Ready` within a minute or two.

5. **Bootstrap Flux** pointing at this repo:
   ```
   flux bootstrap github --owner=<you> --repository=<repo> --path=clusters/homelab
   ```
   From here, Flux takes over reconciling everything below — including re-managing
   the Cilium release you just seeded by hand, so future changes go through Git.

## Layout
```
clusters/homelab/     Flux Kustomizations (entry points)
infrastructure/        Cilium, Traefik, NFS storage, monitoring, Tailscale,
                        Renovate, Harness Delegate, Tuppr, cert-manager,
                        Capacitor, Garage, etcd backups, External Secrets,
                        Loki + Alloy (logs), Gatus (uptime), CloudNativePG,
                        Authelia (SSO)
apps/base/              One folder per app
apps/overlays/production/  Toggles which apps are enabled + per-app patches
```

## A note on Gatus (uptime monitoring)
Deliberately not redundant with the Prometheus alerts already in
`infrastructure/monitoring/rules/` — those check Kubernetes' *internal*
view of health (is the pod Ready). Gatus checks from the *outside*,
actually requesting `https://jellyfin.home.dakin.im` the way a person
would, catching failures Kubernetes' own view can't see: a broken Traefik
route, DNS not resolving, or — usefully, given the cert-manager wildcard
— a certificate that's actually expired as experienced by a real client,
cross-checking `CertificateExpiringSoon`'s view from a completely
different angle. Alerts reuse the exact same Pushover credentials already
set up for Alertmanager, no new secret needed. Status page is
Tailscale-only (`gatus.<tailnet>.ts.net`) — same reasoning as Capacitor,
a dashboard listing every app and its health isn't LAN-public information.

## A note on Loki + Alloy (logging)
Not Promtail — it hit end-of-life on March 2, 2026 (no security patches,
no bug fixes). **Grafana Alloy** (via the `k8s-monitoring` chart) is the
current replacement, running as a DaemonSet on all 3 nodes — including
the tainted control plane, deliberately, since that node's logs matter
given it's your single point of failure. It discovers every pod through
the Kubernetes API automatically, so "logs from all applications" needs
no per-app configuration. Loki runs in monolithic mode (appropriate at
homelab scale) storing to the same NFS-backed storage everything else
uses, 14-day retention to match the pattern already used for Prometheus.
Logs show up in the same Grafana you already have, as a new datasource —
no second dashboard to check.

Also worth knowing: the official Loki Helm chart moved to a new
community-maintained repository on March 16, 2026 — `helm-repositories.yaml`
points at the new location, not the old `grafana.github.io` one.

## Secrets: Infisical + External Secrets Operator
Every secret below used to be a separate manual `kubectl create secret`
step. As of this setup, there's exactly **one** manual secret left — the
Infisical Machine Identity credentials in
`infrastructure/external-secrets/README.md`. Everything else is an
`ExternalSecret` that pulls from a path in your Infisical project, synced
in automatically by ESO. **Start with
`infrastructure/external-secrets/README.md` first** — nothing else below
will sync until that ClusterSecretStore shows `Valid`.

## Disaster recovery
See **`RUNBOOK.md`** for actual recovery procedures. Short version: etcd
snapshots (`infrastructure/talos-backup/`) back up all cluster state
including Secrets. **PVC data (Immich library, Jellyfin config,
Grafana/Prometheus) currently has no backup at all** — Velero was removed
by request, and nothing has replaced it. Read the "Known gap" section of
`RUNBOOK.md` before assuming your photos are safe.

`infrastructure/garage/` is now unused — it was built specifically as
Velero's backup target and nothing else in this repo talks to it. Left in
place in case you want to reintroduce Velero (or something else that
wants local S3) later, but it's dead weight as-is. Say the word if you'd
rather I remove it too.

## A note on Capacitor (Flux dashboard)
Flux itself has no GUI by design — it's CLI/API-first. Capacitor is the
Flux project's own official dashboard recommendation (fluxcd.io has
blogged about it directly), read-only, and installed as an OCI artifact
reconciled straight by a top-level Flux Kustomization
(`clusters/homelab/capacitor.yaml`) rather than folded into the usual
`infrastructure/kustomization.yaml` build — that's the documented install
path, not a deviation from how everything else here is structured. Access
is Tailscale-only (`capacitor.<tailnet>.ts.net`), same reasoning as
Jellyfin/Immich — its RBAC includes reading Secrets, which shouldn't sit
on the LAN-facing Traefik path even behind auth.

This is a genuinely different installation shape from Flux Operator
(`FluxInstance`-based, manages Flux's own controllers/upgrades) — this
repo still bootstraps Flux the classic way, Capacitor is just a viewer on
top of it.

## One-time manual steps
- `infrastructure/external-secrets/README.md` — **do this first**:
  Infisical project + Machine Identity (the one remaining manual secret)
- `infrastructure/tailscale/README.md` — OAuth client → `/tailscale/*`
- `infrastructure/renovate/README.md` — GitHub token → `/renovate/*`
- `infrastructure/harness/README.md` — Harness account signup, delegate token → `/harness/*`
- `infrastructure/tuppr/README.md` — talosconfig → `/tuppr/*`
- `infrastructure/monitoring/README.md` — Pushover credentials → `/pushover/*`
- `infrastructure/cert-manager/README.md` — Cloudflare API token → `/cloudflare/*`
- `infrastructure/garage/README.md` — currently unused, see note above
- `infrastructure/gatus/README.md` — DNS for the gatus.home.dakin.im LAN route
- `infrastructure/authelia/README.md` — core secrets, Postgres password, user database

## A note on CNPG + Authelia
CNPG runs one workload so far: a single-instance Postgres backing
Authelia's sessions and config (`instances: 1` — same reasoning as
Garage's `replication_factor: 1`, real HA is overhead you don't need
here). Immich deliberately stays on its bundled chart Postgres, not
CNPG — see the conversation that led here: Immich's official Postgres
image doesn't work with CNPG at all, and the community workaround images
are version-locked to specific vector-extension requirements that drift
as Immich itself updates. CNPG is the right foundation for *future*
Postgres-requiring workloads (Gitea, Paperless-ngx, etc.), just not that
one.

Authelia protects the Traefik/LAN path specifically — it has no effect on
the Tailscale path, which already has its own access control (tailnet
membership). Two apps are behind it (`gatus.home.dakin.im`,
`capacitor.home.dakin.im`, both `two_factor` policy) — Jellyfin and
Immich's LAN routes (`jellyfin.home.dakin.im`, `immich.home.dakin.im`)
deliberately are not, since both already have their own login and
stacking Authelia on top would just mean logging in twice for no real
security gain. To protect another app later: add a `domain:` rule in
`authelia/release.yaml`'s `access_control.rules`, then add the same
`router.middlewares` annotation used on `gatus/ingress-lan.yaml` to that
app's Ingress.

## A note on Tuppr + Renovate
Tuppr executes Talos/Kubernetes upgrades but does not enforce safe upgrade
paths — Renovate does that instead, via the `separateMajorMinor` /
`separateMinorPatch` rule in `renovate.json`, which forces one version-step
PR at a time. **Merge these PRs in order, never batch them** — skipping a
minor version is an unsupported upgrade path Tuppr won't stop you from
attempting.

## A note on cert-manager
Traefik's own TLS was never actually wired up to a real ACME flow — this
replaces that gap entirely. cert-manager issues one wildcard cert via
Cloudflare DNS-01 (the only way to get a wildcard; HTTP-01 can't), and
Traefik uses it as its default certificate for everything routed through
it — no per-app annotations needed. Your domain's *registration* stays at
Namecheap; only the DNS hosting moves to Cloudflare, which is what makes
the DNS-01 automation practical (Namecheap's own API requires manual IP
whitelisting that breaks on a residential dynamic IP).

## Alerting coverage
`infrastructure/monitoring/rules/` adds Jellyfin/Immich down-detection,
NFS volume space warnings, Renovate job health, and certificate expiry —
on top of kube-prometheus-stack's own defaults (node down, disk pressure,
generic pod crash-looping, which already cover most workloads
cluster-wide without any extra config).

**Known gaps, not covered yet** — each would need an exporter that isn't
in this stack:
- Postgres/Redis internals for Immich (would need postgres_exporter /
  redis_exporter sidecars — outright crashes still get caught by the
  generic pod-crash rules, just not things like slow queries or connection
  pool exhaustion)
- NFS mount-level failures specifically, as opposed to volume space
  (would need node_exporter's NFS mountstats collector enabled)

## A note on Harness
The old free self-hosted "Harness CD Community Edition" was retired in
Dec 2023. What's here instead is a lightweight in-cluster **Delegate**
(single pod, ~512Mi-1Gi) that connects outbound to Harness's SaaS Manager
(free tier available) — the pipelines/dashboard live in Harness's cloud,
not on your hardware. Much lighter than Devtron would have been, but it
does mean your CD tooling now depends on an external service being up,
unlike everything else in this repo which is fully self-contained.
