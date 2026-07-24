# cert-manager + Cloudflare — one-time setup (not managed by Flux/Git)

## 1. Nameservers — already done
`dakin.im` already points at Cloudflare's nameservers, so this step is
skipped. (Registration stays at Namecheap regardless — only DNS hosting
moved.) Just confirm the domain is added as a site in your Cloudflare
dashboard if you haven't already, since the API token in step 2 is scoped
to a specific zone there.

## 2. Create a scoped Cloudflare API token
Don't use your Global API Key — create a scoped token instead:
**Cloudflare dashboard → My Profile → API Tokens → Create Token → Custom
Token**, with:
- Permissions: `Zone / DNS / Edit`, `Zone / Zone / Read`
- Zone Resources: limit to the specific domain, not "All zones"

## 3. Create an Infisical secret
In Infisical, create a secret in the `cloudflare` folder of your
`homelab` project (`prod` environment):
- `API_TOKEN` → the scoped Cloudflare API token from step 2

`external-secret.yaml` already references `/cloudflare/API_TOKEN` — no
further editing needed once it exists in Infisical.

## 4. Edit the remaining placeholder
- `cluster-issuer.yaml` — set your real email in both ClusterIssuers
  (Let's Encrypt uses this for expiry notices as a backup to the alert
  we've now got wired to Pushover). `certificate.yaml` is already filled
  in for `home.dakin.im` / `*.home.dakin.im`.

## 5. Test against staging first
`certificate.yaml` defaults to `letsencrypt-cloudflare-staging`. Confirm
it issues before touching production:
```
kubectl describe certificate traefik-default-cert -n kube-system
```
Look for `Ready: True`. Staging certs show as browser-untrusted — that's
expected, it's just confirming the DNS-01 flow works end-to-end. Once
confirmed, switch `certificate.yaml`'s `issuerRef.name` to
`letsencrypt-cloudflare-prod` and delete the `traefik-default-cert-tls`
secret so cert-manager reissues against the real Let's Encrypt API.
