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

`embed_url` is optional. An app without one is agent-only and opens no screen.

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
| `payments.checkout` | the app may start a checkout |
| `tasks.create` | the app may create tasks |
| `workflows.create` | the app may create workflows |
| `storage` | per-app key-value storage, with a quota |
| `rewards.accrue` | the app may award points |
| `rewards.balance` | the app may read a balance |
| `events` | the app hears workspace events, such as `card.actioned` |
| `tools.invoke` | the app's interface may call its declared tools. Required whenever `tools` is non-empty |

## Collections

Each has a `key` unique within its own list, and validation rejects duplicates.

- **`agents`** (max 16): `key`, `name`, `persona`, `instructions`, `skills`, `rules`, `mcp`.
  Anything in `mcp` must name a server this manifest declares.
- **`mcp_servers`** (max 16): the servers the app's agents may reach. Auth is `none`,
  `static`, `oauth2` or `tina_identity`. Credentials stay in TINA's vault, never in the
  page. Submit refuses a server on a port every HTTP client refuses, 4190 among them.
  `tina_identity` is for a server the app runs itself: nobody connects anything; TINA
  signs the acting person into every call as `x-mcp-<server key>-authorization: Bearer
  <assertion>`, the same Ed25519 assertion the page sends the backend, and the server
  verifies it with `tina.verifyIdentity()`. It requires the `user.context` and
  `tenant.identity` scopes, and consent to the live version is re-checked on every call.
- **`tools`** (max 64): operations the app's interface may invoke, each passing through to
  one declared `mcp` server.
- **`workflow_actions`** (max 16): preset steps for the workflow builder, each running one
  of this app's own agents.
- **`chat_cards`** (max 32): card types the app may post.
- **`roles`** (max 16): what a person is inside this app. Publishing provisions each as a
  tenant role that a tenant admin assigns. They grant nothing in TINA.
- **`notifications`** (max 16): the reasons the app may interrupt somebody, each one they
  can switch off.
- **`plans`** (1 to 8): `key`, `name`, `price_cents`, `currency`, and an interval of
  `month`, `year` or `one_time`.

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
`chrome` defaults to `shell`, where the workspace draws the header. `sdk` lets the SDK draw
it inside the frame. Upload refuses `sdk` unless a platform admin has marked the app
trusted for it, and every launch checks again, so an app whose trust is withdrawn gets
`shell`. Build the page to work under both.

## `hooks`

```json
"hooks": {
  "url": "https://api.yourapp.example/hooks/tina",
  "events": ["subscription.started", "subscription.ended", "consent.changed"]
}
```

The endpoint must answer a registration challenge by echoing the nonce, both at submit and
again at publish. A version whose endpoint does not answer is not submitted. Handle the
challenge before any of the app's own logic, and run `npx tina-miniapp doctor` to check it
while it is still cheap to fix.

The three obligation events (`user.deletion_requested`, `tenant.offboarding`,
`data.export_requested`) are delivered whether or not they are listed here.

## Cross-field rules validation enforces

- An agent's `mcp` entries must name declared servers.
- A tool's `mcp` must name a declared server.
- `tools` being non-empty requires the `tools.invoke` scope.
- A workflow action's `agent` must name a declared agent.
- No duplicate keys in any collection.
