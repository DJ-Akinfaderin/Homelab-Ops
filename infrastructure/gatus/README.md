# Gatus LAN route — DNS prerequisite (not managed by Flux/Git)

`ingress-lan.yaml` routes `gatus.home.dakin.im` through Traefik, but
nothing in this repo makes that hostname actually resolve anywhere — this
gap has existed silently since the cert-manager wildcard cert was set up
(it covers `*.home.dakin.im` for TLS, but issuing a cert for a name says
nothing about whether that name resolves). You need one of:

**Option A — public DNS record in Cloudflare (simplest):**
Add an `A` record for `gatus.home.dakin.im` → `10.70.5.90` in the
Cloudflare dashboard (same place the nameservers were configured). Since
nothing is port-forwarded from your router, this is only reachable from
inside your LAN regardless — the record being public just means anyone
who knew to look could see that hostname resolves to a private IP, not
that they could reach it.

**Option B — local-only DNS (more private):**
If you run a local DNS resolver on your network (Pi-hole, your router's
DNS, etc.), add a local override there instead: `*.home.dakin.im` →
`10.70.5.90`. Nothing about your internal hostnames touches public DNS at
all this way. More setup, but tighter.

Either way, every future LAN-routed app (not just Gatus) needs the same
resolution to work — worth deciding once which option you want, since
it'll apply to anything else you add to the Traefik path later.

**Update:** `capacitor.home.dakin.im` (infrastructure/capacitor/ingress-lan.yaml)
now needs the same treatment — add it alongside `gatus.home.dakin.im`,
same option (A or B) you picked above.

**Update 2:** `jellyfin.home.dakin.im` and `immich.home.dakin.im` too —
same treatment, same option. Four hostnames total now needing the same
resolution: gatus, capacitor, jellyfin, immich.

**Update 3:** `radar.home.dakin.im` — same treatment. Five hostnames now:
gatus, capacitor, jellyfin, immich, radar.
