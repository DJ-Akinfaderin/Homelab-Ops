# Tuppr — one-time setup (not managed by Flux/Git)

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
