# Disaster Recovery Runbook

What this covers, and what it doesn't:

| Failure | Covered? | How |
|---|---|---|
| A worker node (PC2/PC3) dies | Yes | Re-image with Talos, rejoin — pods reschedule to the surviving worker automatically |
| The control-plane node (PC1) dies | Yes, with data loss back to last snapshot | Restore etcd from the last `etcd-snapshot` backup |
| Accidental `kubectl delete` of something important | Partially | Flux re-applies anything git-managed on its next reconcile; anything not in Git (PVC contents) is not recoverable this way |
| **PVC data lost or corrupted** (Immich library, Jellyfin config, Grafana/Prometheus) | **No** | No backup mechanism — see note below |
| Your NAS dies | **No** | Nothing here protects against this |
| Your house burns down | **No** | Nothing here is off-site |

## Scenario A: One worker node's hardware is gone

The simplest case — Talos nodes are stateless/immutable by design, so this
is mostly re-running the original bootstrap for one node:

1. Re-image the replacement PC with the same GPU-enabled worker schematic
   from Image Factory (`talos/extensions/worker-extensions.yaml`)
2. Apply the same worker config patches (`talos/patches/cluster-network.yaml`
   + `talos/patches/worker.yaml`, with a unique hostname)
3. `talosctl apply-config` to the new node — it joins the existing cluster
4. Flux reconciles nothing new here; the cluster just has its capacity back.
   Pods that were on the dead node get rescheduled automatically once
   Kubernetes notices it's gone (a few minutes, via the default pod
   eviction timeout)

No data loss for anything NFS-backed (Jellyfin config, Immich library,
Grafana/Prometheus) — that data was never on the worker node's local disk
to begin with.

## Scenario B: The control-plane node (PC1) is gone

This is the one that actually loses state — etcd only exists on PC1, and
there's no HA control plane in this setup (deliberate tradeoff made early
on, see the top-level README).

1. Get your most recent etcd snapshot — check `/backups` on the
   `etcd-backups` PVC (mount it from any pod, or `kubectl cp` it off
   before wiping anything)
2. Re-image PC1 (or its replacement) with the control-plane schematic
3. Apply `talos/patches/cluster-network.yaml` + `talos/patches/controlplane.yaml`
   via `talosctl apply-config`, but do **not** run the normal
   `talosctl bootstrap` —  instead:
   ```
   talosctl bootstrap --recover-from=./etcd-snapshot-<date>.db --nodes <PC1-IP>
   ```
4. This restores the ENTIRE cluster state as of that snapshot — every
   HelmRelease, every Secret (Tailscale OAuth, GitHub token, Pushover
   creds, Cloudflare token, the talosconfig itself), everything. You are
   not manually recreating those six-plus secrets
   from scratch.
5. Once etcd is healthy (`talosctl etcd status`), the workers reconnect on
   their own, and Flux resumes reconciling from wherever Git is — anything
   committed after the snapshot but not yet applied gets picked up
   automatically.

**Data loss window:** anything that changed between the last etcd
snapshot (daily, 3:30am) and the failure. Config changes are safe either
way since they're re-derived from Git; what's actually at risk is Secret
rotations or manually-run one-off commands that never made it to a
snapshot.

## Scenario C: Everything is gone — full rebuild

Worst case, all three PCs and their disks are gone, but your Git repo and
your NAS both survived (NAS surviving matters a lot here — see below):

1. Provision 3 new PCs, re-run the full bootstrap from the top-level
   README (Image Factory → gen config → apply → bootstrap → Cilium →
   Flux)
2. Once Flux is running and has reconciled `infrastructure/`, every
   secret-bearing tool (Tailscale, Renovate, Tuppr, Pushover,
   cert-manager) needs its out-of-band secret recreated — follow each
   `infrastructure/*/README.md` in order
3. If the NAS itself survived: PVC data (Immich library, Jellyfin config,
   Grafana/Prometheus) was never stored on the PCs to begin with — once
   `nfs-subdir-external-provisioner` reconnects to the same NFS export,
   apps pick their data back up automatically. No restore step needed.
4. If the NAS did *not* survive: that data is gone. There is currently no
   backup of it anywhere else — see the gap below.

This is meaningfully slower than Scenario B (no etcd snapshot to recover
from means recreating every secret by hand), which is exactly why the
etcd snapshot path is worth keeping current.

## Known gap: PVC data has no backup at all

This is the one to take seriously, not just a nice-to-have: **Immich's
photo library, Jellyfin's config, and Grafana/Prometheus data exist in
exactly one place — your NAS.** Nothing in this repo backs that up
anywhere else. If the NAS fails, gets corrupted, or is stolen/destroyed,
that data is gone regardless of how well the Kubernetes-level recovery
above works — etcd snapshots restore cluster *state* (what should be
running), not PVC *contents* (the actual files).

Previously this repo used Velero + a self-hosted Garage instance to cover
exactly this gap; it was removed by request. If you want this covered
again later, that's the mechanism to bring back — or use your NAS's own
snapshot/replication features (most NAS OSes, e.g. Synology/TrueNAS, have
this built in) as a separate, non-Kubernetes solution. Either way, treat
this as a real open gap, not a hypothetical one — it's the most likely
source of genuinely irreplaceable loss (the photos, specifically) in this
whole setup.

## Test this before you need it

An untested backup is a hope, not a plan. Once this is running:
- Confirm an etcd snapshot actually lands in `/backups` after the first
  scheduled run
- Actually practice Scenario B on a spare node if you ever have one free —
  the Talos docs explicitly recommend this rather than finding out during
  a real failure
