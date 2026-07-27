# Deployment Checklist: bare metal → reachable apps

This sequences every step already documented across this repo into the
actual order to run them in. Each step links to the file with the real
detail — this is the ordering layer on top, not a replacement for those
files. Check things off as you go; skipping ahead out of order is the
most likely way to hit a confusing failure, since several steps are
genuinely blocked on earlier ones (noted explicitly below).

---

## Phase 0 — Before you touch any hardware

- [ ] All 3 ThinkCentre M710s wired to the UniFi Cloud Gateway (not WiFi —
      cluster nodes should be wired)
- [ ] NAS reachable at `10.70.5.185`, with an NFS export created for this
      cluster (matches `infrastructure/storage/nfs-provisioner.yaml`'s
      `nfs.path`)
- [ ] `dakin.im` confirmed pointed at Cloudflare nameservers (already done
      per earlier setup)
- [ ] This repo pushed to a GitHub repository you control
- [ ] `talosctl`, `kubectl`, `flux`, and `helm` CLIs installed on your
      admin machine (not the cluster nodes themselves)

---

## Phase 1 — Install Talos on all 3 PCs

Full detail: top of `README.md`.

- [ ] Boot all 3 PCs from a Talos installer ISO/USB into maintenance mode
- [ ] Submit `talos/extensions/worker-extensions.yaml` to Image Factory,
      get back a schematic ID — **this is the image for PC2 and PC3 only**
      (bakes in GPU + NFS client support). PC1 uses the stock installer.
- [ ] `talosctl gen config homelab https://<PC1-IP>:6443` with the three
      `--config-patch` flags from `README.md` step 2
- [ ] `talosctl apply-config` to each node with its matching config
      (controlplane.yaml → PC1, worker.yaml → PC2 and PC3, unique
      hostname each — see `talos/patches/worker.yaml`'s comment)
- [ ] `talosctl bootstrap -n <PC1-IP>` — **PC1 only, once**

At this point all 3 nodes exist but sit `NotReady` — expected, there's no
CNI yet. Don't troubleshoot this; it's step 2.

---

## Phase 2 — Cilium (manual, one-time) → Flux bootstrap

Full detail: `README.md` steps 4–5.

- [ ] `helm repo add cilium https://helm.cilium.io/ && helm repo update`
- [ ] `helm install cilium cilium/cilium -n kube-system -f infrastructure/cilium/values.yaml`
      — nodes should flip to `Ready` within a minute or two
- [ ] `flux bootstrap github --owner=<you> --repository=<repo> --path=clusters/homelab`

Flux is now running and will start trying to reconcile everything in
`infrastructure/`. **Expect a lot of red/failing/pending status for the
next several steps** — most of it is genuinely waiting on the manual
secrets below, not broken. `flux get kustomizations` and
`flux get helmreleases -A` are your friends for watching this settle.

---

## Phase 3 — Unblock secrets (do this before anything else below)

Everything else in this repo that needs a credential routes through
Infisical via External Secrets Operator. Nothing downstream of this step
will actually work until it's done.

- [ ] **`infrastructure/external-secrets/README.md`** — Infisical project,
      Machine Identity, bootstrap secret. Verify with
      `kubectl get clustersecretstore infisical` showing `Valid` before
      moving on.

---

## Phase 4 — Certificates (unblocks HTTPS for every LAN app at once)

- [ ] **`infrastructure/cert-manager/README.md`** — Cloudflare API token
      → Infisical. This one step unblocks HTTPS for every LAN-routed app
      simultaneously, since Traefik uses one wildcard cert for all of
      them.
- [ ] Verify staging issuance first:
      `kubectl describe certificate traefik-default-cert -n kube-system`
      → look for `Ready: True`
- [ ] Switch `infrastructure/cert-manager/certificate.yaml`'s
      `issuerRef.name` to `letsencrypt-cloudflare-prod`, delete the
      `traefik-default-cert-tls` secret to force reissuance, commit and
      push

---

## Phase 5 — DNS for the LAN routes

Full detail: `infrastructure/gatus/README.md` (the DNS explanation lives
there, applies to all four hostnames below).

- [ ] Confirm Traefik actually got its LoadBalancer IP:
      `kubectl get svc -n kube-system traefik` → should show `10.70.5.90`
- [ ] Add DNS resolution (public Cloudflare `A` record, or local
      Pi-hole/router override — your choice, see the linked README) for:
  - [ ] `gatus.home.dakin.im`
  - [ ] `capacitor.home.dakin.im`
  - [ ] `jellyfin.home.dakin.im`
  - [ ] `immich.home.dakin.im`
  - [ ] `auth.home.dakin.im`

---

## Phase 6 — Remote access (Tailscale)

- [ ] **`infrastructure/tailscale/README.md`** — OAuth client → Infisical
- [ ] Confirm with `kubectl get pods -n tailscale` and check the
      Tailscale admin console for connected devices

---

## Phase 7 — Auth for the apps that need it

- [ ] **`infrastructure/authelia/README.md`** — core secrets, Postgres
      password, your user + password hash, all → Infisical
- [ ] Verify CNPG's database first:
      `kubectl get cluster authelia-postgres -n authelia` → `Cluster
      Healthy`
- [ ] Visit `https://auth.home.dakin.im`, confirm you can log in and
      register a second factor

---

## Phase 8 — Alerts

- [ ] **`infrastructure/monitoring/README.md`** — Pushover credentials →
      Infisical
- [ ] Send the test alert command in that README, confirm a phone
      notification actually arrives

---

## Phase 9 — Verify every app is actually reachable

This is the actual finish line for what you asked for. Check every path
for every app:

| App | Tailscale | LAN | Notes |
|---|---|---|---|
| Jellyfin | `jellyfin.<tailnet>.ts.net` | `https://jellyfin.home.dakin.im` | No Authelia — own login |
| Immich | `immich.<tailnet>.ts.net` | `https://immich.home.dakin.im` | No Authelia — own login |
| Capacitor | `capacitor.<tailnet>.ts.net` | `https://capacitor.home.dakin.im` | LAN route behind Authelia |
| Gatus | `gatus.<tailnet>.ts.net` | `https://gatus.home.dakin.im` | LAN route behind Authelia |
| Grafana | — | — | Internal only: `kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80` |
| Authelia | — | `https://auth.home.dakin.im` | The portal itself — LAN only |

- [ ] Every URL above loads without a certificate warning (confirms
      Phase 4's production cert switch actually took)
- [ ] Gatus's own dashboard (`gatus.home.dakin.im`, once logged into
      Authelia) shows all its configured endpoints green — this is
      genuinely the fastest single way to confirm the whole stack is
      healthy at a glance, since it's already checking Jellyfin, Immich,
      Grafana, and the apiserver's cert for you

---

## Phase 10 — Everything else (not blocking "apps reachable," worth doing soon after)

- [ ] `infrastructure/renovate/README.md` — GitHub token, dependency PRs
- [ ] `infrastructure/tuppr/README.md` — talosconfig, automated Talos/K8s
      upgrades (reuses the same talosconfig from Phase 1's `gen config`
      step — see `talos/README.md`)
- [ ] Set up the Immich external library (`/photos/Public`) — the UI step
      documented in `apps/base/immich/release.yaml`'s comments, not
      something Flux can do for you
- [ ] Read `RUNBOOK.md` end to end *before* you need it, not after
