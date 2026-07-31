# Tuppr — one-time setup (not managed by Flux/Git)

## 1. Enable Talos API access from Kubernetes (required — this is the actual fix)

Tuppr's chart needs `talos.dev/v1alpha1 ServiceAccount` to exist as a
real API resource — it doesn't, by default. This is a genuine machine
config change, not a Git/YAML-only fix, so it needs applying to your
actual running nodes directly:

```
talosctl --talosconfig ./talosconfig patch mc \
  --endpoints 10.70.5.103 \
  --nodes 10.70.5.103,10.70.5.81,10.70.5.208 \
  --patch @talos/patches/kubernetes-talos-api-access.yaml
```

(`--endpoints` is the control plane specifically — the single, reliable
connection point talosctl talks to, which then proxies the actual patch
out to each node listed in `--nodes`. Same pattern as every other
multi-node talosctl command in this repo — `--nodes` alone isn't
sufficient, matching the "failed to determine endpoints" issue hit
earlier with the etcd-snapshot job.)

(All three nodes — the controller pod and tuppr's per-node upgrade Jobs
can each land on different nodes over the course of an upgrade, so every
node needs this, not just wherever the controller happens to run today.)

Verify it actually took before moving on — check each node individually,
`--endpoints` matching `--nodes` this time since each command targets
just one node directly:
```
talosctl --talosconfig ./talosconfig get machineconfig --endpoints 10.70.5.103 --nodes 10.70.5.103 -o yaml | grep -A5 kubernetesTalosAPIAccess
talosctl --talosconfig ./talosconfig get machineconfig --endpoints 10.70.5.81 --nodes 10.70.5.81 -o yaml | grep -A5 kubernetesTalosAPIAccess
talosctl --talosconfig ./talosconfig get machineconfig --endpoints 10.70.5.208 --nodes 10.70.5.208 -o yaml | grep -A5 kubernetesTalosAPIAccess
```
Should show `enabled: true` on all three — if any show nothing, that
node's patch didn't apply and tuppr's HelmRelease will still fail with
the original error for operations touching that node specifically.

## 2. The talosconfig secret

Tuppr needs a `talosconfig` (the same client credential file `talosctl`
itself uses) to actually call the Talos API and drive upgrades.

You already have this file from the original cluster bootstrap
(`talos/README.md` at the repo root) — it's whatever `talosctl` uses to
talk to your nodes, typically `~/.talos/config` unless you pointed it
elsewhere.

1. In Infisical, create a secret in the `tuppr` folder of your `homelab`
   project (`prod` environment):
   - `TALOSCONFIG` → paste the entire contents of your talosconfig file
     as-is (it's YAML, multi-line values are fine in Infisical)

   `external-secret.yaml` already references `/tuppr/TALOSCONFIG` — no
   further editing needed once it exists in Infisical. (This is simpler
   than the Keeper version of this setup was — Infisical stores it as a
   plain text secret value, no attachment-vs-field ambiguity to work
   around.)

## 3. Verify

```
flux get helmrelease tuppr -n kube-system-upgrade
```
Should show `Ready: True` — if it's still showing the original
`ServiceAccount`/`talos.dev` error after step 1, double-check the patch
actually landed on all three nodes (see the verification command above).

## Before relying on this for real

- Trigger one upgrade manually first and watch it end-to-end (`kubectl get
  talosupgrade cluster -w`) before assuming it'll behave correctly
  unattended overnight.
- Since your control plane is a single dedicated node (PC1), it has no
  redundancy during its own upgrade — a Kubernetes-level upgrade on the
  control plane is a moment where cluster management is briefly
  unavailable even though workloads on the two workers keep running. Worth
  scheduling Renovate's merge/upgrade window for a time you can be around,
  at least until you trust the process.
