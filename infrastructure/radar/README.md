# Radar — one-time setup (not managed by Flux/Git)

## 1. Generate the session secret
```
openssl rand -hex 64
```
In Infisical, create this in a new `radar` folder of your `homelab`
project (`prod` environment): `AUTH_SECRET` → the generated value.

## 2. Verify

```
kubectl get externalsecret radar-auth-secret -n radar
```
Should show `SecretSynced` / `READY: True`.

```
kubectl get pods -n radar
```
Should show the Radar pod `Running`, `1/1`.

## 3. DNS
Same as every other LAN-routed app — `radar.home.dakin.im` needs the
same DNS treatment as `gatus.home.dakin.im`. See
`infrastructure/gatus/README.md` for the two options (Cloudflare record
vs local override).

## 4. First login
Visit `https://radar.home.dakin.im`. You should be redirected to
Authelia's login first (same as Gatus/Capacitor), then land on Radar's
actual dashboard after authenticating — not a "No Namespace Access"
screen. If you do see that screen, the `radar-admin-group`
ClusterRoleBinding didn't apply correctly, or Authelia's `Remote-Groups`
header value doesn't match `admin` exactly (check your actual group name
in `infrastructure/authelia`'s `users_database.yml` in Infisical).

## Worth knowing
- **Single instance only, hard-enforced by the chart itself** — unlike
  Authelia's DaemonSet issue earlier, this chart refuses to even start
  with more than one replica when SQLite persistence is enabled, so this
  specific category of bug can't recur here.
- **Write/exec capability is off by default** — `podExec`, `secrets`,
  `helm`, and `viewRBAC` were all left at the chart's own safe defaults
  (`false`). Radar can browse, view logs, and show topology, but can't
  exec into pods or view Secret values as currently configured. If you
  want more capability later, these are explicit opt-ins in
  `infrastructure/radar/release.yaml`'s `rbac:` block — enable
  deliberately, not by default.
- **The `strip-identity-headers` Traefik Middleware is not optional** —
  it's what prevents anyone on your LAN from spoofing the `Remote-User`/
  `Remote-Groups` headers and impersonating a user. Both are chained on
  `radar-lan`'s ingress annotation in the correct order (strip, then
  Authelia) — don't remove or reorder this if you ever edit that file.
