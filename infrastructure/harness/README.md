# Harness Delegate — one-time setup (not managed by Flux/Git)

1. Sign up for Harness (free tier is fine for a homelab):
   https://app.harness.io

2. In the UI: **Account Settings → Account Resources → Delegates → New
   Delegate → Kubernetes → Helm Chart**. This screen shows your real
   `accountId`, `managerEndpoint`, the current `delegateDockerImage` tag,
   and a generated delegate token — copy the non-token values into
   `release.yaml`.

   Worth knowing: **this download screen is the actual source of truth**
   for the exact Helm values schema — Harness's delegate chart values have
   changed shape before across versions. Treat `release.yaml` here as a
   Flux-shaped starting point, and reconcile it against whatever the UI
   generates for your account before applying.

3. In Infisical, create a secret in the `harness` folder of your
   `homelab` project (`prod` environment):
   - `DELEGATE_TOKEN` → the delegate token from step 2

   `external-secret.yaml` already references `/harness/DELEGATE_TOKEN` —
   no further editing needed once it exists in Infisical.

4. Once the delegate pod is `Running` and shows as connected in the
   Harness UI, you're ready to build a CD pipeline there pointing at this
   cluster.
