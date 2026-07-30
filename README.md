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
                        Renovate, Tuppr, cert-manager,
                        etcd backups, External Secrets,
                        VictoriaLogs + Alloy (logs), Gatus (uptime),
                        CloudNativePG, Authelia (SSO)
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
Tailscale-only (`gatus.<tailnet>.ts.net`) —
a dashboard listing every app and its health isn't LAN-public information.

## A note on VictoriaLogs + Alloy (logging)
Not Promtail — it hit end-of-life on March 2, 2026 (no security patches,
no bug fixes). **Grafana Alloy** (via the `k8s-monitoring` chart) is the
current replacement, running as a DaemonSet on all 3 nodes — including
the tainted control plane, deliberately, since that node's logs matter
given it's your single point of failure. It discovers every pod through
the Kubernetes API automatically, so "logs from all applications" needs
no per-app configuration.

This originally ran on Loki, switched to **VictoriaLogs** after a real
incident: Loki's chart shipped a memcached cache sub-component with a
~9.8Gi default memory request — bigger than an entire 8GB node — which
permanently blocked scheduling and cascaded into blocking Grafana and
Prometheus too, since they share the same NFS provisioner dependency
chain. VictoriaLogs is a single binary with no equivalent sub-components
to carry a similar surprise default, and per its own docs auto-tunes
resource usage to what's actually available rather than shipping
hardcoded production-scale defaults. Alloy ships logs to it using Loki's
own wire format (`/insert/loki/api/v1/push`) — VictoriaLogs accepts this
natively, so the collector side barely changed, only the destination URL
did. Grafana needs one extra piece Loki didn't: the
`victoriametrics-logs-datasource` plugin, since VictoriaLogs isn't a
built-in Grafana datasource type the way Loki is.

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

`infrastructure/garage/` was removed entirely (it was built specifically
as Velero's backup target, then became unused after Velero was removed).
Leaving it deployed but unused turned out to be an active problem, not
just clutter: its Deployment required a manually-created secret that
`DEPLOYMENT.md` told people to skip, which made its own health check
fail — and because the `infrastructure` Kustomization health-checks every
resource in it, that one failure blocked everything else behind it too,
apps included. If you want local S3 storage again later (for Velero or
anything else), it's straightforward to re-add — just make sure whatever
you add actually gets its required secrets set up, not left "optional."

## A note on Radar (Kubernetes UI / GitOps visibility)
Originally had Capacitor here as a minimal, read-only Flux dashboard —
removed after Radar was added, since Radar's own GitOps view covers the
same ground with more depth (field-level drift detection, stuck-reconcile
diagnosis, one-click remediation) plus everything else Capacitor didn't
do at all (resource browsing, topology, RBAC visibility, cluster audit).
Keeping both meant maintaining a second thing that duplicated a subset of
what the first already did better — not worth the surface area.

Deployed via Helm (`skyhook/radar`), not a raw OCI artifact — every field
in `infrastructure/radar/release.yaml` is traced directly to the chart's
real `values.yaml` and its own auth docs, not guessed. Sits behind
Authelia on the LAN path (`radar.home.dakin.im`) same as Gatus, ungated
on Tailscale same reasoning as everywhere else — tailnet membership is
already a real access boundary.

One piece here is genuinely load-bearing, not just config hygiene: the
`strip-identity-headers` Traefik Middleware in
`infrastructure/radar/middleware-strip-headers.yaml`. Radar's proxy-auth
mode trusts whatever `Remote-User`/`Remote-Groups` headers arrive at it —
without stripping those from external requests first, anyone on the LAN
could set them directly and impersonate any user, bypassing Authelia
entirely. Confirmed straight from Radar's own security docs, not an
assumption.

## One-time manual steps
- `infrastructure/external-secrets/README.md` — **do this first**:
  Infisical project + Machine Identity (the one remaining manual secret)
- `infrastructure/tailscale/README.md` — OAuth client → `/tailscale/*`
- `infrastructure/renovate/README.md` — GitHub token → `/renovate/*`
- `infrastructure/tuppr/README.md` — talosconfig → `/tuppr/*`
- `infrastructure/monitoring/README.md` — Pushover credentials → `/pushover/*`
- `infrastructure/cert-manager/README.md` — Cloudflare API token → `/cloudflare/*`
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
`radar.home.dakin.im`, both `two_factor` policy) — Jellyfin and
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
