# Renovate — one-time setup (not managed by Flux/Git)

Renovate needs a GitHub token with repo write access to open PRs against
your homelab repo.

1. Generate a GitHub personal access token (fine-grained, scoped to just
   this repo) with **Contents: read/write** and **Pull requests:
   read/write** permissions.

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
