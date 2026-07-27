# Infisical — one-time setup (not managed by Flux/Git)

This is the only manual `kubectl create secret` left in this repo —
everything else (Tailscale, Renovate, Tuppr, Pushover,
Cloudflare) pulls from Infisical via the ExternalSecret in each of their
folders instead.

## 1. Create an Infisical project
If you don't already have one: sign up at https://app.infisical.com
(or point `hostAPI` in `secretstore.yaml` at your own instance if
self-hosting), then create a project for this cluster — e.g. `homelab`.

Copy the **Project Slug** from Project Settings — this goes in
`secretstore.yaml`'s `projectSlug`.

## 2. Create a Machine Identity (Universal Auth)
This is the credential ESO itself authenticates with — separate from your
own login.

- Org Settings → **Machine Identities → Create Identity**, auth method
  **Universal Auth**
- Grant it access to the `homelab` project with **read-only** access to
  secrets (least privilege — it never needs to write anything)
- Note the generated **Client ID** and **Client Secret** (the secret is
  shown once)

## 3. Create the bootstrap secret
```
kubectl create namespace external-secrets
kubectl create secret generic infisical-credentials \
  -n external-secrets \
  --from-literal=client-id='<your-client-id>' \
  --from-literal=client-secret='<your-client-secret>'
```

## 4. Fill in the project slug and verify
Edit `secretstore.yaml`, replace `<your-infisical-project-slug>` with the
real slug from step 1, then check it connected:
```
kubectl get clustersecretstore infisical
```
Should show `Valid` in the status once Flux reconciles it.

## 5. Add the actual secrets
Every tool's `ExternalSecret` references an absolute path like
`/tailscale/CLIENT_ID` — in Infisical, that maps to a folder (`tailscale`)
containing a secret named `CLIENT_ID`. Follow each tool's own README for
exactly which secrets it needs; create the matching folder + secret in
Infisical's dashboard (or via the `infisical` CLI) under the `prod`
environment (matches `environmentSlug` above — use a different
environment slug here and in every ExternalSecret if you'd rather organize
it that way).
