# Authelia — one-time setup (not managed by Flux/Git)

## 1. Generate the three core secrets
These are random strings, not passwords you choose:
```
openssl rand -hex 64   # run 3 times, once each for:
                        # JWT_SECRET, SESSION_SECRET, STORAGE_ENCRYPTION_KEY
```
In Infisical, create three secrets in the `authelia` folder of your
`homelab` project (`prod` environment): `JWT_SECRET`, `SESSION_SECRET`,
`STORAGE_ENCRYPTION_KEY` — one generated value each.

## 2. Set the Postgres password
Pick a strong password (this one you do choose, for the CNPG-managed
database) and create it as `/authelia/POSTGRES_PASSWORD` in Infisical,
same folder.

## 3. Create your user and generate a password hash
Authelia needs a bcrypt (or argon2) hash, not a plaintext password. Using
the Authelia CLI via a temporary pod (no local install needed):
```
kubectl run authelia-hash --rm -it --restart=Never \
  --image=ghcr.io/authelia/authelia:latest -- \
  authelia crypto hash generate argon2 --password 'your-actual-password'
```
Copy the resulting hash (starts with `$argon2id$...`).

Build a `users_database.yml` with your user, e.g.:
```yaml
users:
  yourusername:
    displayname: "Your Name"
    password: "$argon2id$...(the hash from above)"
    email: [email protected]
    groups:
      - admins
```
Paste this entire file's contents as one Infisical secret at
`/authelia/USERS_DATABASE_YML` in the same folder.

## 4. Verify
```
kubectl get externalsecret -n authelia
kubectl get cluster authelia-postgres -n authelia   # should show Cluster Healthy
kubectl get pods -n authelia
```
Then visit `https://auth.home.dakin.im` — you should see Authelia's login
portal. Log in with the user from step 3.

## 5. Protecting Gatus (already wired)
`gatus/ingress-lan.yaml` already has the Traefik middleware annotation
and `authelia/release.yaml`'s `access_control.rules` already has a
`two_factor` rule for `gatus.home.dakin.im`. Since you only set up a
password (not 2FA) in step 3, you'll need to register a second factor on
first login, or change that rule's `policy` to `one_factor` if you'd
rather skip 2FA for now.

## Known uncertainty, worth checking rather than assuming
The exact way `secrets.existingSecret` and `config.storage.postgres`'s
password get wired together in the chart is something I'm reasonably but
not fully confident about — it's Authelia's own well-established `_FILE`
env var convention under the hood, but if the Authelia pod crash-loops on
first start, check `kubectl logs -n authelia deploy/authelia` for a
config-parsing error before assuming something else is wrong.
