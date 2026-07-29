# Renovate — one-time setup (not managed by Flux/Git)

Renovate needs a GitHub token with repo write access to open PRs against
your homelab repo.

1. **Use a classic PAT, not fine-grained.** This changed from the
   original guidance here after a real failure (`platform-unknown-error`
   at repo init) — Renovate's own maintainers have an open, acknowledged
   GitHub issue admitting their fine-grained token documentation is
   stale and incomplete. A classic PAT with the **`repo`** scope is the
   explicitly recommended, reliable default in Renovate's own docs, not
   a workaround.

   GitHub → Settings → Developer settings → Personal access tokens →
   Tokens (classic) → Generate new token → check **`repo`**.

2. In Infisical, create a secret in the `renovate` folder of your
   `homelab` project (`prod` environment):
   - `GITHUB_TOKEN` → the GitHub token from step 1

   `external-secret.yaml` already references `/renovate/GITHUB_TOKEN` — no
   further editing needed once it exists in Infisical.

3. Edit `cronjob.yaml`'s `args` to point at your actual repo
   (`your-username/homelab-repo`).

4. First run: trigger it manually instead of waiting for 3am —
   ```
   kubectl create job --from=cronjob/renovate renovate-manual-run -n renovate
   ```

If it finds updates (new Cilium/Traefik/Immich chart versions, new
Jellyfin image tags), you'll get PRs against this repo. Merge one,
and Flux picks up the change automatically on its next reconcile.

