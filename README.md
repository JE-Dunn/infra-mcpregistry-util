# Private MCP registry (GitHub Pages)

Static [MCP registry v0.1](https://registry.modelcontextprotocol.io/docs) for GitHub Copilot
(Enterprise > AI Controls > MCP > MCP Registry URL), hosted on GitHub Pages. It registers
Microsoft Work IQ as `io.github.microsoft/workiq`. The server ID must exactly match the ID of the
installed server ([enforcement docs](https://docs.github.com/en/copilot/reference/enterprise-administrators/mcp-private-registry-enforcement)).

## Layout (everything under `site/`, published as the Pages root)

```
site/.nojekyll
site/v0.1/servers/index.html                                   -> GET /v0.1/servers (list; JSON body)
site/v0.1/servers/index.json                                   -> identical copy, served as application/json
site/v0.1/servers/io.github.microsoft/workiq/versions/latest   -> GET .../versions/latest
site/v0.1/servers/io.github.microsoft/workiq/versions/0.2.8    -> GET .../versions/0.2.8
```

Pages serves files literally, so the raw-slash directory layout matches `.../io.github.microsoft/workiq/...`.
A `%2F`-encoded request (`io.github.microsoft%2Fworkiq`) is expected to be decoded by Pages to the same
path. **This must be verified after the first deploy.**

## Limitations (GitHub Pages)

- **Headers:** Pages sends `Access-Control-Allow-Origin: *` by default. `Access-Control-Allow-Methods` and
  `Access-Control-Allow-Headers` cannot be set. Simple GET requests work; a preflight that sends `Authorization`
  may not be satisfied by browser clients (Copilot's non-browser clients are unaffected).
- **Content-Type:** Pages derives it from the file extension. Extensionless files (`latest`, `0.2.8`) are expected
  to be served as `application/octet-stream` (not verified; I could not test before deploying).
  `/v0.1/servers` cannot be a file, because `servers/` must also be a directory, so it redirects to
  `/v0.1/servers/` and serves `index.html` as `text/html` with a JSON body. `index.json` is included so you
  can confirm `application/json` is only obtained with a `.json` extension. If clients reject non-JSON
  content types, Pages cannot satisfy the spec; use Azure Static Web Apps (custom headers/rewrites) or a
  Function/Worker in front instead.
- Use `curl -L` for `/v0.1/servers` (301 to trailing slash).

## Add a server

1. Add `site/v0.1/servers/<namespace>/<name>/versions/<version>` and `.../versions/latest`
   (extensionless, same `{ "server": {...}, "_meta": {...} }` content as the existing files).
2. Append the entry to `servers` in `site/v0.1/servers/index.json` and `index.html` (keep identical) and bump `metadata.count`.
3. `name` must be the server's canonical ID (`namespace/name`).
4. Open a PR; the workflow validates the JSON on merge to `main`.

## Enable Pages and deploy

Repo Settings > Pages > Source: **GitHub Actions**. Then merge to `main` (or run the workflow manually).
The registry URL is `https://<owner>.github.io/<repo>` (set that as the MCP Registry URL). Note that Pages
sites for private repos require a plan that supports it, and are public by default unless access control is configured.

## Test

```
H=https://<owner>.github.io/<repo>
curl -siL $H/v0.1/servers
curl -si  $H/v0.1/servers/index.json
curl -si  "$H/v0.1/servers/io.github.microsoft/workiq/versions/latest"
curl -si  "$H/v0.1/servers/io.github.microsoft%2Fworkiq/versions/latest"
curl -si  "$H/v0.1/servers/io.github.microsoft%2Fworkiq/versions/0.2.8"
# CORS and content type
curl -si "$H/v0.1/servers/io.github.microsoft%2Fworkiq/versions/latest" | grep -i -E '^HTTP|content-type|access-control'
```

## Protect the registry

- Branch protection on `main`: require a pull request with at least one approving review, dismiss stale approvals, block force pushes.
- Keep `CODEOWNERS` pointing at real owners and enable "Require review from Code Owners".
- Restrict who can edit the `github-pages` environment/deployments.
