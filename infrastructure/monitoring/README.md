# Alertmanager → Pushover — one-time setup (not managed by Flux/Git)

1. Create a free Pushover account, then in the dashboard:
   - Note your **User Key** (shown on the main dashboard)
   - Create an **Application/API Token**: pushover.net/apps/build →
     name it something like "Homelab Alerts" → note the token it gives you
   - Install the Pushover app on your phone and log in with the same account

2. In Infisical, create two secrets in the `pushover` folder of your
   `homelab` project (`prod` environment):
   - `USER_KEY` → your Pushover User Key
   - `TOKEN` → the Application/API Token

3. `external-secret.yaml` already references `/pushover/USER_KEY` and
   `/pushover/TOKEN` — no further editing needed once those exist in
   Infisical.

4. **Verify the config actually took effect.** The kube-prometheus-stack
   chart has a history of silently keeping the default Alertmanager config
   if the values aren't shaped exactly right for the chart version you're
   on. After Flux reconciles, check either:
   - Alertmanager's web UI → Status page (shows the live loaded config), or
   - `kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 -- amtool config show`

   If it's still showing the default config instead of the `pushover`
   receiver, the chart version's expected values shape has likely shifted —
   check `helm show values prometheus-community/kube-prometheus-stack`
   against what's in `release.yaml`.

5. Send a test alert to confirm delivery end-to-end before trusting it:
   ```
   kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 -- \
     amtool alert add alertname=TestAlert severity=warning --annotation=summary="test from amtool"
   ```
   You should get a phone notification within a few seconds.
