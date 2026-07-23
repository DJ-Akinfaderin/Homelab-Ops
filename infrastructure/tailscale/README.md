# Tailscale operator — one-time setup (not managed by Flux/Git)

The operator needs an OAuth client to register itself and create proxies on
your tailnet.

1. In the Tailscale admin console: **Settings → OAuth clients → Generate
   client**. Scopes needed: `devices:core` (write). Add the tag
   `tag:k8s-operator` to the client.

2. In Infisical, create two secrets in the `tailscale` folder of your
   `homelab` project (`prod` environment):
   - `CLIENT_ID` → the OAuth Client ID
   - `CLIENT_SECRET` → the OAuth Client Secret

3. `external-secret.yaml` already references `/tailscale/CLIENT_ID` and
   `/tailscale/CLIENT_SECRET` — no further editing needed once those exist
   in Infisical.

4. In the Tailscale admin console, also add an ACL grant so the operator's
   tag is allowed to create tagged devices — the default ACL usually
   already permits this, but double check under **Access controls**.

Once External Secrets Operator and the ClusterSecretStore are running
(see `infrastructure/external-secrets/README.md` first if you haven't set
those up yet), `kubectl get externalsecret operator-oauth -n tailscale`
should show `SecretSynced` within a minute or so of applying.
