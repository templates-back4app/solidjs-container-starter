# Deploy through the Back4app MCP — solidjs-container-starter — 2026-11-10

Repo commit uploaded: `cde6338` (branch `main`, clean tree for Part A).

## Context (same for the 9 starters measured this day)

- Date: 2026-11-10 (all times UTC).
- Account kind: permanent, signed in (`whoami` -> `accountKind: "permanent"`, `limits: {freeAppsUsed: 19, maxFreeApps: 50}` before this run). Apps created on the Free plan.
- Client: Claude Code driving the Back4app MCP connector; terminal on macOS (Node v25.8.1, curl 8.7.1); validation requests left from Sao Paulo (CloudFront POP `GRU1-P5`).
- Upload CLI: `@back4app/cli` 0.3.2, run as `npx -y @back4app/cli upload --url <upload-link>` inside the starter folder. The upload link carries a one-time token and is not recorded here. `create_upload` returned `expiresAt` 15 minutes after the call.
- "create -> READY" below = deployment `createdAt` to the `updatedAt` of the `get_deployment` answer that first said `READY`. It is the platform's own clock, not the wall time of the polling.

## Tools called, in order

Part A: `create_upload` (no appId) -> CLI upload in the folder -> `create_web_app` -> `get_web_app` -> `get_deployment` -> `get_runtime_logs`

Part B4: `create_upload` (with the appId) -> CLI upload in the folder -> `deploy_web_app` -> `get_deployment` -> `get_web_app` -> `list_deployments`

## Part A — first deploy

### Upload (CLI output)

```text
Packing .../solidjs-container-starter ...
27 file(s), 159 KB (2 left out)
Sending 12 new file(s) ...
Upload ready.
uploadId: 9f7d35bf-1597-4caf-87b0-73b74fbea4f2
manifestSha256: e93516aec9656082b9088f228284213d59061cdd72af53d8560cb02594a9ff1b
```

- Wall time: 7 s. No "may hold secrets" message for this folder.
- The repo tracks 20 files; the CLI packed 27, which is every file in the folder outside `node_modules/` and `.git/` (local `dist/` included: the CLI does not read `.gitignore`).
- Only 12 of the 27 files were sent although this folder had never been uploaded through MCP today: the CLI sends only files Back4app does not hold yet. Where the other 15 came from (other starters, earlier uploads on the account) was not investigated.

### App and deployment

- `create_web_app` arguments: `name: "mcp-solidjs"`, `port: 8080`, `uploadId`, `manifestSha256`; `builder` not specified.
- appId `73b98d68-16e2-4f43-8ae3-7ad480239066` · deploymentId `81670f28-e28e-47ac-975d-ed62a26e2eeb`
- App `createdAt` 12:53:42.014Z · deployment `createdAt` 12:53:42.219Z
- Status seen: `BUILDING` / `BUILDING_IMAGE` (12:53:42) -> `READY` (`updatedAt` 12:54:00.685Z) -> **18.5 s**
- Public URL: 404 at 12:53:50, first 200 at 12:54:03.

Settings reported by `get_web_app`:

- `builder`: `DOCKERFILE` · `port`: `8080` · `startCommand`: `null` · `healthCheckPath`: `"/"` · `isStaticSite`: `false` · `rootDir`: `""` · `runtimeVersion`: `null` · `envVarNames`: `[]` · `plan`: `Free` · `source.kind`: `UPLOAD`
- `url`: https://mcpsolidjs-x8zjq0c7.back4.app
- `urlExpiresAt`: 2026-11-10T13:53:42.103Z (60 minutes after the app was created)
- `get_deployment` returned no `suggestions` and no `portDetected` on this deployment.

### Validation on the public URL (12:54Z)

- `GET /` -> 200, `content-type: text/html`, `server: nginx/1.27.5`, `via: 1.1 ... .cloudfront.net (CloudFront)`, no `content-encoding` (475 B document).
- JS referenced by the HTML: `/assets/index-CDaOe7bN.js` -> 200, **7,657 B** transferred with `Accept-Encoding: gzip` (15,031 B uncompressed). Headers: `content-encoding: gzip`, `server: nginx/1.27.5`, `cache-control: public, max-age=31536000, immutable`, `via: ... (CloudFront)`.
- Deep route: `GET /some/deep/route` -> 200, 475 B (`index.html`).
- Missing asset: `GET /assets/nope.js` -> 404.

### Who compressed the JS: the platform edge, not the starter's nginx

- The nginx access log (`$body_bytes_sent`, counted after nginx's own compression) shows 15,031 B sent for the JS file, which is its uncompressed size, while the client received 7,657 B with `content-encoding: gzip`. nginx sent it uncompressed; the gzip was applied in front of the container.
- Most likely cause, with the mechanism checked locally on 2026-11-10 with the preact starter (same `nginx.conf` gzip lines, `nginx:1.27-alpine`, Docker 29.5.2): a request with `Accept-Encoding: gzip` gets 7,335 B gzip; the same request plus a `Via:` header gets 17,970 B and no `content-encoding`. nginx's `gzip_proxied` defaults to `off`, which disables compression for requests that arrive through a proxy, and the Back4app edge is one.
- Consequence: `gzip on; gzip_comp_level 6;` in the starter's `nginx.conf` does nothing on this platform as written. The reader still gets gzip (from the edge), at a weaker ratio than nginx level 6 would give.
- The fix would be `gzip_proxied any;` in `nginx.conf`. NOT applied and NOT tested on Back4app.

### Runtime logs

`get_runtime_logs`: 26 lines (nginx entrypoint, nginx notices, access log). Lines with the same millisecond came back in a shuffled order. First three as returned:

```text
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
```

## Part B4 — update flow (`create_upload` with appId -> `deploy_web_app`)

1. Created `public/update-check.txt` containing `mcp-update-1794315717`, not committed (`git status`: `?? public/update-check.txt`). Before the deploy, `GET /update-check.txt` on the live app answered 200 `text/html` 475 B: the SPA fallback, not the file.
2. `create_upload` with `appId: 73b98d68-...`. Its `nextAction` changed to "pass both to deploy_web_app or update_web_app".
3. CLI in the folder:

```text
28 file(s), 159 KB (2 left out)
Sending 1 new file(s) ...
Upload ready.
uploadId: 3e7123b7-69d6-472b-a0c0-b7f65d70bfaf
manifestSha256: 054ffb580bf269fc49b75f787fcf877d78c58395f0e6b28fa95708922c4c36a7
```

   Wall time 7 s. The upload is the folder as it is on disk: the uncommitted file went along, and only that file was sent.
4. `deploy_web_app` with `appId`, `uploadId`, `manifestSha256` -> `{ "deploymentId": "f8e7b623-bea7-4b11-b623-a7164d59740a" }`. No confirmation step.
5. New deployment `createdAt` 13:02:17.762Z -> `READY` (`updatedAt` 13:02:38.325Z) -> **20.6 s**.
6. URL: the same, https://mcpsolidjs-x8zjq0c7.back4.app. `urlExpiresAt` also the same (13:53:42.103Z): deploying again did not extend the 60 minutes.
7. `GET /update-check.txt` -> 200 `text/plain`, body `mcp-update-1794315717`, first seen at 13:02:44 (polled every 5 s). `GET /` answered 200 on every poll during the switch; no error was observed.
8. `list_deployments`: the new one `READY` / `active: true`; the first one (`81670f28-...`) `REMOVED` / `active: false`, `updatedAt` 13:02:38.496Z.
9. `public/update-check.txt` deleted afterwards; the tree is clean and the file was never committed. The live app keeps serving it until its next deploy.

## Not tested

- `update_web_app` with a confirmationId, `rollback_web_app` to the first deployment, `deploy_web_app` with no code (redeploy of the same upload).
- `gzip_proxied any;` on Back4app.
- Whether in-flight requests are dropped during the switch (only one request every 5 s was sent).
- Not tested anywhere in this series: `confirm_action`, custom domains, env vars at creation, GitHub as the code source through MCP, behaviour after `urlExpiresAt`, a temporary (not signed in) account, the `files` upload path for clients with no terminal.

## Compression fix verified on the public URL (2026-11-10)

- Cause: requests reach the container through the platform's CDN with a `Via` header, and nginx's `gzip_proxied` is off by default, so nginx answered them uncompressed and the CDN compressed the response itself.
- Fix: `gzip_proxied any;` in nginx.conf (commit 1c640f2), redeployed to the same app with `deploy_web_app`.
- Main JS transferred from the public URL (`Accept-Encoding: gzip`): before 7,657 B, after 5,665 B. The "after" figure equals this bundle compressed by nginx at gzip_comp_level 6.
