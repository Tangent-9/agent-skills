# tina-miniapp.json

The manifest is what review reads and what the platform enforces. `npx tina-miniapp
validate` checks it against the platform's own schema, so treat its output as the authority
and this file as the map.

## Required fields

| Field | Notes |
| ----- | ----- |
| `name` | 1 to 120 characters |
| `version` | exact semver, no range |
| `sdk` | the SDK range the app is built against, e.g. `^0.5.0`. Platform dialect: `^0.5.0` accepts any 0.x at or above 0.5, unlike npm |
| `plans` | at least one, at most 8 |

A screen comes from one of two fields, never both:

- **`hosting`**: TINA hosts the page. `{ "frontend": { "dir": "dist" } }` names the build
  output, relative to the manifest and inside the project. `tina-miniapp deploy` publishes it
  and writes `embed_url` into the version it uploads, so the manifest itself has none.
  Uploading a `hosting` manifest any other way is refused.

  `hosting.backend` adds a server TINA builds and runs behind `/api/*` on the same host:

  | Field | Default | Meaning |
  | --- | --- | --- |
  | `dir` | `.` | the Docker build context |
  | `dockerfile` | `Dockerfile` | relative to `dir` |
  | `port` | `8080` | what the server listens on, 1024 to 65535 |
  | `runtime` | `lambda` | `lambda`, or `container` for a server that keeps state itself |
  | `volume.path` | none | `container` only: an absolute path for a disk that survives restarts and redeploys, not a system directory |
- **`embed_url`**: the partner hosts the page, over https (loopback http while developing).
  Leave it out until the page is live: the CLI reads a manifest with the `ui.embed` scope
  and neither field as a self-hosted page waiting for its URL. `validate` passes it with a
  note, and `tina-miniapp url` or `submit` writes it.

An app with neither and no `ui.embed` is agent-only, opens no screen, and must bundle at
least one agent.

## Capabilities (`scopes`)

Every one is shown at consent. Widening them in a new version makes existing subscribers
re-consent before that version reaches them, so ask for what the code uses today.

| Scope | Grants |
| ----- | ------ |
| `user.context` | who is using the app, and their locale |
| `tenant.identity` | the tenant's id, in `context()` and as `tenant_id` in the identity assertion. Sensitive |
| `tenant.roles` | the person's built-in tenant roles, as `tenant_roles` in the identity assertion. Sensitive |
| `ui.embed` | the app may draw a screen in the workspace |
| `ui.chat_card` | the app may post cards into a chat |
| `tasks.create` | the app may create tasks |
| `workflows.create` | the app may create workflows |
| `storage` | per-app key-value storage, with a quota |
| `events` | the page may send its own analytics events with `tina.emit()`. Listening with `useAppEvent` needs no scope |
| `tools.invoke` | the app's interface may call its declared tools. Required whenever `tools` is non-empty |

The schema also accepts `payments.checkout`, `rewards.accrue` and `rewards.balance`, but no
SDK method or endpoint uses them yet. Requesting one adds a line at consent and buys nothing.

## Collections

Each entry has a `key` unique within its own list (`chat_cards` use `type` instead), and
validation rejects duplicates.

- **`agents`** (max 16): `key`, `name`, `persona`, `instructions`, `skills`, `rules`, `mcp`.
  Anything in `mcp` must name a server this manifest declares.
- **`mcp_servers`** (max 16): the servers the app's agents may reach. Auth is `none`,
  `static`, `oauth2` or `tina_identity`. Credentials stay in TINA's vault, never in the
  page. Submit refuses a server on a port every HTTP client refuses, 4190 among them.
  `tina_identity` is for a server the app runs itself: nobody connects anything; TINA
  signs the acting person into every call as `x-mcp-<server key>-authorization: Bearer
  <assertion>`, with the key lowercased. A key with other characters arrives under two
  names, each run folded to `_` and to `-`: server `health-api` gets
  `x-mcp-health_api-authorization` and `x-mcp-health-api-authorization`. Proxies and CDNs,
  TINA's hosting included, drop the underscore name, so read the dashed one, or use a key of
  letters and digits only (`validate` warns otherwise). It carries
  the same Ed25519 assertion the page sends the backend. Serve it with
  `createMcpHandler`, or verify it with `tina.verifyMcpIdentity(headers, key)`. On a
  hosted backend, or on a self-hosted page that serves `/api/*` itself, its `url` is a path,
  such as `/api/mcp`. It requires the `user.context` and
  `tenant.identity` scopes, and consent to the live version is re-checked on every call.
- **`tools`** (max 64): operations the app's interface may invoke, each naming a `tool` on
  one declared `mcp` server. `consequential: true` makes the workspace ask the person to
  confirm before the call runs. Set it on anything that spends, sends or deletes.
- **`workflow_actions`** (max 16): preset steps for the workflow builder, each running one
  of this app's own agents.
- **`chat_cards`** (max 32): card types the app may post.
- **`roles`** (max 16): what a person is inside this app. Publishing provisions each as a
  tenant role that a tenant admin assigns. They grant nothing in TINA. At most one is
  `default`.
- **`notifications`** (max 16): the reasons the app may interrupt somebody, each one they
  can switch off.
- **`plans`** (1 to 8): `key`, `price_cents`, optional `name`, `currency` (default `sgd`),
  and `interval`: `month` (the default), `year` or `one_time`.

## `ui`

```json
"ui": {
  "chrome": "shell",
  "modules": [
    { "key": "today", "label": "Today", "icon": "☀️", "default": true },
    { "key": "settings", "label": "Settings", "icon": "⚙️" }
  ],
  "info": {
    "tagline": "One line for the App info sheet.",
    "groups": [{ "name": "Screens", "items": ["Today", "Settings"] }],
    "note": "What the app actually does."
  }
}
```

`modules` is the menu the workspace draws above the frame, and the page cannot change it.
A module's `roles` limits who sees it, and each must name a declared role.
`chrome` defaults to `shell`, where the workspace draws the header. `sdk` lets the SDK draw
it inside the frame. Upload refuses `sdk` unless a platform admin has marked the app
trusted for it, and every launch checks again, so an app whose trust is withdrawn gets
`shell`. Build the page to work under both.

## `hooks`

```json
"hooks": {
  "url": "/api/hooks/tina",
  "events": ["subscription.started", "subscription.ended", "consent.changed"]
}
```

`url` is a path under `/api/` for a hosted backend, or for a self-hosted page that serves
`/api/*` on its own origin, and an https URL otherwise. On a self-hosted page the CLI
resolves the path against `embed_url` before uploading, because the platform takes only full
URLs from it; keep the path in the file.

The endpoint must answer a registration challenge by echoing the nonce, both at submit and
again at publish. A version whose endpoint does not answer is not submitted. Handle the
challenge before any of the app's own logic, and run `npx tina-miniapp doctor` to check it
while it is still cheap to fix.

The three obligation events (`user.deletion_requested`, `tenant.offboarding`,
`data.export_requested`) are delivered to every app with a `hooks.url`, whether or not
`events` lists them. An app with no `hooks` block gets none, and the sandbox refuses to
send them.

## Cross-field rules validation enforces

- An agent's `mcp` entries must name declared servers.
- A tool's `mcp` must name a declared server.
- `tools` being non-empty requires the `tools.invoke` scope.
- A workflow action's `agent` must name a declared agent.
- No duplicate keys in any collection.
- `hosting` and `embed_url` are never both set.
- A screen (`embed_url` or `hosting`) and the `ui.embed` scope come together: each requires the other.
- `ui.modules` require a screen.
- `hosting.frontend.dir` stays inside the project (no `..`).
- `hooks.url` or an `mcp_servers` `url` that is a path must be under `/api/`, and needs `hosting.backend`. A self-hosted page's paths reach the platform as full URLs on `embed_url`'s origin, resolved by the CLI.
- `hosting.backend.volume` needs `runtime: "container"`.
- `chat_cards` require the `ui.chat_card` scope.
- A module's `roles` must name declared roles.
- At most one default role and one default module.
- An `mcp_servers` entry with `tina_identity` auth requires `user.context` and `tenant.identity`.
- No screen means at least one agent.
