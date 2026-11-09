# Live deploy — Back4app Containers — 2026-11-09

App: spoke-solidjs (free plan), created fresh from back4app-templates/container-deploy-lab @ rotation (commit 8db363b = this repo @ 6c731b5 + stamp 566ae090-e41a-4825-9b69-396088ad90c8). Port 8080 at creation.

## Public URL (client São Paulo; CloudFront edge; fresh deploy)
- Live at 13:35:33 UTC (creation ~13:34).
- Deep route /some/deep/route: 200 — nginx try_files verified live.
- First-load JS: 7,657 B transferred (1 file, gzip via CloudFront) vs 5,665 B from our nginx locally at level 6 — the CDN's compression level decides the public transfer (consistent with the React and Angular deploys).
- TTFB on / (3 requests, illustrative): 1.099 / 0.990 / 0.988 s.

Local measurements: docs/evidence/local-measurements-2026-11-06.md
