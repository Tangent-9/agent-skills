---
name: tina-miniapp
description: Build a TINA mini app end to end. Scaffold it, write its tina-miniapp.json manifest, wire the browser SDK and the app's own backend and MCP server, run it locally in the sandbox, deploy the page and backend to TINA's own hosting (Lambda, or a container with a disk) or host them on the partner's own server, then check and submit it. Use this whenever someone wants to build, extend, debug, test, deploy, migrate or ship an app for the TINA workspace, or is working in a project that has a tina-miniapp.json, or mentions tina-miniapp.json, @tangent-9/miniapps-sdk, create-tina-miniapp, tina-miniapp login, deploy, env or url, tina-sandbox, a hosting block, embed_url, hosting.backend, app scopes and plans, lifecycle hooks, a tina_identity MCP server, a mini app agent, or an app that gets framed by TINA. Reach for it even when the request says "an app for TINA", "put my app on TINA", "move my app off my own server", "host it ourselves" or names a template instead of saying "mini app".
---

# Build a TINA mini app

A mini app is three things: a page the TINA workspace frames in an iframe, a manifest
saying what the app may do, and usually a backend of your own. TINA can host both
(`tina-miniapp deploy`): the page as static files, the backend built from its Dockerfile.
Or the partner hosts them, and tells TINA the page's URL once it is live.
The division of labour is the thing to hold onto. TINA owns the subscription, the session and the consent. Your
service owns the records the app is actually about. That is true of the apps TINA
publishes itself, so a design that wants TINA to store the app's data is fighting the
platform.

## Scaffold, do not hand-write

The CLI ships templates that already validate against the platform schema. Starting from
one is faster than assembling files and finding out at submit which fields were wrong.

```bash
npx @tangent-9/create-tina-miniapp <dir> --template <name>
```

Run by a person at a terminal, it asks whether the app has a screen and a server of its
own, recommends a template, asks who hosts the page, and offers to install. A flag answers
its own question and the rest are still asked. From an agent, pass every answer, because
nobody is there to give one:

```bash
npx @tangent-9/create-tina-miniapp <dir> --template fullstack --hosting tina --install
```

`--hosting tina` (the default) lets TINA host the page; `--hosting self` is the partner's
own server, covered below. Ask the person which they want if the request does not say.
`--yes` takes every default and installs nothing.

The CLI installs with, and prints commands for, whichever package manager ran it: npm,
pnpm, yarn or bun. The project's README uses the same one. Where this skill writes
`npx tina-miniapp`, a pnpm project runs `pnpm exec tina-miniapp`, yarn `yarn tina-miniapp`
and bun `bunx tina-miniapp`. Use the one the project's lockfile names; mixing them leaves
two lockfiles.

| Template | Choose it when |
| ----------------- | ------------------------------------------------------ |
| `react-vite` (default) | the app has a screen and an agent |
| `fullstack` | the app also keeps records or gives its agent tools, so it needs a server. The agent's tools are an MCP server at `/api/mcp` |
| `agent-companion` | the app is an agent and nothing else |

`agent-companion` is a manifest and a README, nothing to install. That matters for one
thing only: with no `node_modules` there is no local `tina-miniapp` binary, so its commands
are `npx @tangent-9/create-tina-miniapp <cmd>`. The other two templates depend on the CLI,
so `npx tina-miniapp <cmd>` resolves locally. `agent-companion` has no `dev` scripts either,
so `dev --sandbox` does not apply to it. Run the sandbox on its own with
`npx -p @tangent-9/miniapps-sdk tina-sandbox`.

After `npm install`, read `node_modules/@tangent-9/miniapps-sdk/README.md`. It is the SDK's
own reference, it ships with the exact version installed, and it is more current than
anything restated here. Read it before writing page code.

## The loop

```bash
npx tina-miniapp dev --sandbox  # the page, its backend and a local stand-in for TINA
npx tina-miniapp validate       # the manifest against the platform's own schema
npx tina-miniapp login          # once per machine; the person approves it in the portal
npx tina-miniapp deploy         # hosted apps: build, upload, and publish on TINA
npx tina-miniapp env set K v    # a hosted backend's settings and secrets
npx tina-miniapp url https://…  # self-hosted apps: where the live page is (embed_url)
npx tina-miniapp doctor         # framing headers and the hook endpoint, against a running app
npx tina-miniapp submit         # send the version for review
```

`submit`, `deploy` and `doctor` find the app by the manifest's `name`, so a listing with
exactly that name must already exist in the partner portal. Create it there first, or record
its id once with `npx tina-miniapp link --app <id>`.

**You cannot sign in for the person.** `login` prints a code and opens the partner portal,
where they check the code, confirm their password and approve. Ask them to run it, then
carry on. `npx tina-miniapp whoami` tells you whether they have. CI uses an API key from the
portal's Developers page as `TINA_PARTNER_KEY` instead; never ask the person to paste one
into the conversation.

Run `validate` after every manifest edit rather than at the end. It carries the platform's
schema, so what it accepts is what upload accepts, and its messages name the field.

The listing's logo is `assets/icon.png` (or `.jpg`, `.jpeg`, `.webp`), up to 512 KB. Make it
square: nothing checks, but the catalogue draws it in a square. `submit` uploads it. SVG is refused, because the platform serves logos to anyone and an
SVG can carry script. Without one, the catalogue shows the app's initial.

`doctor` needs the app running. For a hosted app it skips the framing check, because TINA
sets those headers. Otherwise it checks framing against the workspace origin
(`--workspace` or `TINA_WORKSPACE_URL`, default `http://localhost:3911`). It fails on
`X-Frame-Options: DENY` or `SAMEORIGIN` and on a `frame-ancestors` that leaves the workspace
out, and warns when there is no `frame-ancestors` at all. Those surface as a blank frame in
front of a customer. With `TINA_PARTNER_KEY` set it also sends the registration challenge
to the hook endpoint, as it does when the person is signed in. Without either it only
checks that an unsigned hook is refused.

## Put it on TINA

TINA hosts the page, and the backend if there is one. The manifest says where each is,
and has no `embed_url`:

```json
{
  "hosting": {
    "frontend": { "dir": "dist" },
    "backend": { "dir": ".", "dockerfile": "Dockerfile", "port": 4181 }
  },
  "hooks": { "url": "/api/hooks/tina", "events": [] },
  "mcp_servers": [{ "key": "notes", "url": "/api/mcp", "auth": "tina_identity" }]
}
```

Keep the `ui.embed` scope. `hosting` and `embed_url` together are refused, because TINA
writes the URL.

**The backend lives under `/api/` on the page's own host.** TINA routes `/api/*` there, so
the page calls `/api/...` on its own origin: no backend URL to configure and no CORS. Serve
every backend route, hooks and MCP included, under `/api/`; nothing else reaches it. Written
that way, `hooks.url` and an own `mcp_servers[].url` may be paths, and the deploy writes the
full address into the version. A path is refused without `hosting.backend`. Locally, proxy
`/api` from the dev server to the backend (the `fullstack` template's `vite.config.ts` does),
so the page works the same way.

**Pick the runtime by what the server keeps.**

| `runtime` | Choose it when |
| --- | --- |
| `lambda` (default) | the server keeps its state in a database or another service. It starts per request, may run as many copies as there are requests, and keeps nothing on disk or in memory |
| `container` | the server keeps state itself: a SQLite file, files on disk, an in-memory cache it cannot lose, a websocket. One container runs all the time on a machine of the app's own |

A `container` may ask for a disk: `"volume": { "path": "/data" }` is mounted there and
survives restarts and every redeploy. Point the app's database file at it. A container
must listen on `0.0.0.0` (TINA sets `HOST` to it), not `127.0.0.1`, or nothing outside
the container reaches it. Its first deploy takes a few minutes longer while the machine
starts. A `lambda` backend with a SQLite file loses it, and each copy has its own.

`deploy` tars the backend's `dir` (leaving out `node_modules`, `.git`, `.env*` and whatever
`.dockerignore` names; 8 MB at most) and TINA builds the Dockerfile for arm64. A build that
fails, or a server that does not answer on `port`, fails the deploy with the end of its
log. Read that output before changing anything.

**Settings and secrets go in `tina-miniapp env`, never in the image or the repo.**

```bash
npx tina-miniapp env set DATABASE_URL postgres://…
cat key.pem | base64 | npx tina-miniapp env set PRIVATE_KEY_B64   # values are one line
npx tina-miniapp env                                              # names only
```

Values are sealed and never shown again, and reach the backend on the next deploy. TINA
sets `TINA_APP_ID`, `TINA_ISSUER`, `TINA_API_URL`, `PORT` and `HOST` itself, which is
everything `createTina()` needs; `TINA_*`, `AWS_*`, `PORT` and `HOST` cannot be set.

`npx tina-miniapp deploy` runs the project's `build` script (skip it with `--skip-build`),
uploads the files TINA does not already have, builds the backend, and waits until the deploy
is live. It prints the app's address, which moves with every deploy, and this deploy's own.
Each deploy also uploads a **draft version** pointing at that deploy's own address. That
draft is what the person runs from the listing's **Sandbox** tab in the partner portal, and
what `submit` sends for a hosted app. Deploying the same `version` again replaces the
draft; once it has been submitted, bump `version` first or the deploy fails with "Bump the
version".

`npx tina-miniapp deployments` lists recent deploys, with `*` beside the current one, and
`npx tina-miniapp rollback <id>` moves the app's address back without rebuilding. A
`lambda` backend moves back with the page. A `container` runs the older image again on the
same machine and disk.

Limits: 2,000 files, 8 MB each, 100 MB in all, and an `index.html` at the root of the build
output. A path with no file extension serves `index.html`, so client-side routing works.

A hosted page opened in a tab of its own shows a notice rather than the app: it runs inside
TINA only. Do not treat that as a bug.

### Moving an app that runs on its own host

An app already running elsewhere with `embed_url` moves in five edits, and the agent should
make all of them before the first deploy:

1. Replace `embed_url` with `hosting`: `frontend.dir` is the page's build output, and
   `backend` names the Dockerfile and the port the server listens on. A server that keeps a
   SQLite file or other local state is `runtime: "container"` with a `volume` at the
   directory the file lives in.
2. Move every backend route the manifest names under `/api/` (hooks, MCP), keeping the old
   paths too while the old host is still in use.
3. Write `hooks.url` and the app's own `mcp_servers[].url` as those paths.
4. Point the page at its own origin for the backend (`window.location.origin`), not a
   hard-coded host.
5. Move settings the old host held as environment variables into `tina-miniapp env`.
   Bump `version`.

A server that serves the page itself as well can keep doing so; on TINA only `/api/*`
reaches it. Frame headers it sets for its old host no longer matter: TINA sets them.

## Host it on your own server

Scaffolded with `--hosting self`, the manifest has neither `hosting` nor `embed_url`. That
is deliberate: the page has no address until the partner deploys it, and nothing needs one
before then. `validate` passes with a note, `dev --sandbox` frames the dev server, and
`doctor` checks the hooks through the dev server's `/api` proxy (`localhost:4180`, so
`dev` has to be running).

**Never invent the URL.** Do not write a placeholder, an example host or a localhost
address into `embed_url` to make a check pass. When the page is live, ask the person where,
then record it:

```bash
npx tina-miniapp url https://shop.example.com/   # writes embed_url, checks framing
```

`url` adds that one line to the manifest, checks the host's `frame-ancestors` header, and
prints where the manifest's paths now point. `submit` asks for the URL if it was never set,
and takes `--url <https://…>` where nobody can answer (CI, or an agent). `pack` refuses until
it is set. `deploy` does not apply: it is for pages TINA hosts.

**Keep `/api/...` paths in `hooks.url` and `mcp_servers[].url`.** The platform takes only
full URLs from a self-hosted app, and the CLI resolves each path against `embed_url`'s
origin before it uploads the manifest. The file keeps the paths, so it never names a
production host and the local sandbox resolves them against the dev server. Replacing them
with full URLs points local hooks at production.

What the host has to do, all on one https origin:

- Serve the built page (`dist/`) with `Content-Security-Policy: frame-ancestors` naming the
  workspace. The template's `vite.config.ts` sends it in development only.
- Send `/api/*` to the backend, so the page reaches it on its own origin with no CORS. The
  `fullstack` server reads `PORT` and listens on `127.0.0.1`; change `server.listen` if the
  proxy runs on another machine. Its `Dockerfile` builds the server.
- Set `TINA_APP_ID` (the listing's `app_…` id) and `TINA_ISSUER` (the TINA API origin) in
  the backend's environment. Hooks and identity assertions verify against that issuer.

To switch to TINA hosting later, remove `embed_url`, add the `hosting` block from "Put it
on TINA", and `deploy`.

## Run it before TINA knows about it

`dev --sandbox` also starts `tina-sandbox`, which ships in the SDK, and points the backend
at it for that run. Open `http://localhost:4100`. The sandbox frames the page the way the
workspace does and answers the bridge from fixtures, so the app runs with no account, no
partner key and nothing published. It plays TINA to the backend too. It publishes a key
set and signs identity assertions and hooks with a throwaway key, so `verifyHook` and
`verifyIdentity` run unchanged.

The sandbox frames the page named by `npx tina-sandbox --url <page>`, which `dev --sandbox`
passes as the dev server (`http://localhost:4180/`, the templates' port). It wins over
`embed_url`, so a self-hosted app is worked on locally, not against its live site. With no
`--url`, a hosted app frames `:4180` and any other app its `embed_url`.

Use it for what is hard to reach on a real workspace:

- Switch the person, their roles and what they granted, then relaunch. A declined scope
  should degrade, not break the screen.
- Kill or expire the session, and check the app shows its ended state.
- Send any hook, obligations included (they need a `hooks.url`), and watch the acknowledgement land:
  `npx tina-sandbox hook user.deletion_requested`.
- Run the check submit runs: `npx tina-sandbox challenge`.

The sandbox refuses what TINA refuses, with the same error codes, so treat a refusal there
as a bug in the app. `tina-sandbox.json` beside the manifest overrides the fixtures: the
people, canned tool answers and starting storage.

It stands in for the workspace but is not it. Its key is not TINA's, and `doctor` against
the running app is still the check for framing headers.

## The manifest is the contract

`tina-miniapp.json` is what review reads and what the platform enforces. Everything
declarative lives there rather than being announced at runtime, which is what makes the
menu a person sees the same menu a reviewer approved.

The fields, what each one governs, and the full capability list are in
[references/manifest.md](references/manifest.md). Read it when writing or changing a
manifest.

Two rules worth knowing before you write one:

**Ask for the narrowest scopes the app can work with.** Every scope is shown to the person
at consent, and a new version that widens them makes every existing subscriber re-consent
before it reaches them. A scope added "in case we need it later" costs a re-consent from
the whole install base.

**What `connect()` grants can be narrower than what the manifest requests.** People revoke
individual capabilities. Check `tina.has("storage")` and degrade, rather than assuming the
manifest was honoured in full.

## How the page reaches TINA, and why it matters

The page runs cross-origin inside the workspace and does not `fetch` TINA. The SDK
`postMessage`s the workspace shell, which makes the call first-party on the app's behalf.

Two consequences, both deliberate, and both things a coding agent will otherwise get wrong:

- **The page never holds a credential.** Do not invent a token exchange, an API key in the
  page, or a login screen. A compromised page can ask for what the user granted, and
  nothing more.
- **CORS does not apply.** There is no cross-origin request to TINA to allow, so do not add
  CORS configuration for it. The app's own backend is a different matter: the page does
  call that directly, through `useBackend()`.

A page TINA hosts is already framable by the workspace; skip the rest of this section.
A page on the partner's own host must permit framing:

```
Content-Security-Policy: frame-ancestors https://app.tina.example https://<workspace origin>;
```

Without it the app works everywhere except inside TINA. `X-Frame-Options: DENY` breaks it
the same way, so remove it rather than adding the CSP beside it.

`embed_url` must be https, except on a loopback host (`localhost`, `127.0.0.1`, `*.localhost`)
so a dev server works before there is a certificate. Publishing a loopback `embed_url`
outside development is refused: it would frame whatever happens to run on that port.

## Writing the page

Use the React layer unless there is a reason not to. `TinaProvider` owns the session and
renders children only while it is live, so an app cannot forget to disable itself when the
session ends. Do not reimplement that with your own state.

```tsx
import "@tangent-9/miniapps-sdk/theme.css";
import { TinaProvider, AppShell, Module } from "@tangent-9/miniapps-sdk/react";
```

The menu above the frame comes from `ui.modules` in the manifest. `<AppShell>` renders
whichever module the header opened and `useNavigate()` moves between them. A page cannot
add to the menu, rename an item, or reorder the platform's entries. If a design calls for
that, the answer is a manifest change, not a runtime one.

The kit (`Button`, `Card`, `Dialog`, `Tabs`, `ListRow`, `Fields`, `FilterBar`, `ListState`,
`Masked`, `ToastProvider`) is styled by `theme.css`, which carries the workspace's own
tokens, type and component shapes. Build from it and the app reads as part of the page it
is framed in. On Tailwind v4, import `@tangent-9/miniapps-sdk/tailwind.css` for the same
tokens as utilities.

None of it is required. A partner with their own design system should use it: redefining
`--tina-*` on `:root` carries the kit along, and an app styled entirely its own way passes
review and runs fine. Ask which the partner wants rather than assuming the default.

One thing the stylesheet cannot do for you. It names the faces the workspace uses and
falls back to the system stack, but it loads neither, because a stylesheet that fetches
fonts leaves an app no way out. The templates carry the link in `index.html`, so a
scaffolded project is already right; a page you built another way needs it:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:wght@600;700;800&family=Hanken+Grotesk:wght@400;500;600;700&family=Space+Mono:wght@400;700&display=swap" />
```

Without it the app renders in the system face inside chrome that does not, and the seam
shows. Self-hosting the files is the same change on one line; all three families are OFL.

Read `node_modules/@tangent-9/miniapps-sdk/DESIGN.md` before designing a screen. It says
what each token is for, which component does which job, and which few things the workspace
genuinely owns: the header and its menu, the permission and exit sheets, and confirmation
of a consequential action. Those hold whatever the app looks like. The rest of that file is
a default, not a rule.

## Writing the backend

The server half verifies what TINA sends. It needs no secret, because both message types
are Ed25519-signed and checked against a public key set.

```ts
import { createTina } from "@tangent-9/miniapps-sdk/server";
const tina = createTina({ appId: process.env.TINA_APP_ID!, issuer: process.env.TINA_ISSUER! });
```

Add `credential: process.env.TINA_APP_CREDENTIAL` (a `tinaak_…` a platform admin issues)
only when the backend calls TINA back: `tina.api.notify()` under a declared `notifications`
category, or `tina.api.subscribers()`. Verifying needs no credential.

**Give the app's agent tools with an MCP server of its own.** Declare it with
`auth: "tina_identity"` and serve it with `createMcpHandler`. TINA signs the person the
agent acts for into every call, so nobody connects anything and the handler checks it:

```ts
import { createMcpHandler } from "@tangent-9/miniapps-sdk/server";

const mcp = createMcpHandler({
  tina,
  serverKey: "notes",                     // its key in mcp_servers
  tools: [{
    name: "read_note",
    description: "The person's note.",
    inputSchema: { type: "object", properties: {} },
    readOnly: true,
    run: (_args, { claims }) => ({ note: notes.get(claims.sub) ?? "" }),
  }],
});
// POST /api/mcp
const { status, body } = await mcp({ headers: req.headers, body: rawBody });
```

It is plain JSON-RPC over POST with no session, so it runs on Lambda. The identity is not
in `Authorization`: it is in `x-mcp-<key>-authorization`, the key lower-cased. A key with
other characters arrives twice, folded to `_` (`x-mcp-sales_desk-authorization`) and to `-`
(`x-mcp-sales-desk-authorization`). CloudFront, nginx and Caddy drop a header name with an
underscore, so behind TINA's hosting, or the partner's own proxy, only the dashed one
arrives. `createMcpHandler` reads either. On another MCP library, `await
tina.verifyMcpIdentity(headers, "sales-desk")` does the same and returns the claims; a
server that reads the header itself must read the dashed name. A key of letters and
digits has one name and avoids the question, and `validate` warns about any other.
`tina_identity` requires `user.context` and `tenant.identity`.

**Answer the registration challenge first.** At submit and again at publish TINA sends a
challenge to `hooks.url`. A plain 200 fails. Echo the nonce before any of the app's logic:

```ts
const { event, deliveryId, challenge } = await tina.verifyHook({ headers: req.headers, rawBody: req.rawBody });
if (challenge) return reply.send({ challenge });
```

Three more things reliably go wrong here:

**Hook signatures cover raw bytes.** Verify against `req.rawBody`. A body that has been
through `JSON.parse` and back is a different string and will not verify, and most web
frameworks parse before your handler sees it. Configure the raw body first.

**Identity assertions live two minutes and describe that moment.** Verify per request and
do not cache the claims. Holding them past `exp` keeps a revoked scope alive, which is why
`exp` is enforced with no tolerance.

**Obligations arrive at any app with a `hooks.url`, whether or not `hooks.events` lists
them.** `user.deletion_requested`, `tenant.offboarding` and `data.export_requested` carry
data protection duties TINA cannot discharge, because TINA does not hold the app's records.
Each arrives with an `ack_token`. Act, then acknowledge within 30 days, even when the answer
is `not_applicable`. The ack token is the only credential this needs:

```ts
await tina.api.acknowledge(deliveryId, event.ack_token!, { outcome: "done" }); // or not_applicable, or refused with a note
```

One nobody answers is escalated to platform staff with the app named. An app that keeps
records about people needs a `hooks` block for this reason alone.

A hook gets eight attempts over about 21 hours on any non-2xx. A 4xx other than 408 or
429 stops at once, and a redirect is not followed and counts as a failure. Handlers must be
idempotent on `deliveryId`. `event.sequence` counts up per
app, and a gap means a delivery was missed and the app should reconcile.

Authorization is a backend decision against a verified token:

```ts
const claims = await tina.verifyIdentity(bearerToken);
const roles = Array.isArray(claims.roles) ? (claims.roles as string[]) : [];
if (!roles.includes("hr-admin")) return reply.code(403).send();
```

`roles` holds the manifest's own role keys. The `IdentityClaims` type does not declare it,
so read it as above rather than destructuring.

`useRoles()` and `<RequireRole>` in the page are for layout only. A page cannot authorize
itself, and treating those hooks as a permission check is the classic mistake.

The assertion's `tenant_type` says what kind of workspace the person is in, and not which
one. An app that keeps an organisation's own records, such as a staff directory, needs to
know which one to keep each organisation's data apart. It requests `tenant.identity` for a
`tenant_id` claim, and `tenant.roles` for `tenant_roles`, the person's built-in roles such
as `tenant_admin`. Both are sensitive and shown at consent. Request them only when the app
partitions by tenant or offers admin actions. Without the grant the claim is absent.

## Before saying it is done

- `npx tina-miniapp validate` passes with no warnings that matter.
- The app works in the sandbox with each scope declined in turn. If it has `hooks`, the
  challenge answers and every obligation hook gets acknowledged.
- `npx tina-miniapp doctor` passes against the running app.
- Opening the page directly shows the "has to run inside the TINA workspace" state, or, for
  a hosted app, TINA's "open this app in TINA" notice. Both are correct: there is no session
  until the workspace frames the app.
- For a hosted app: `npx tina-miniapp deploy` finishes and the person has run the draft from
  the partner portal's Sandbox tab. Say so if they have not signed in yet, rather than
  claiming it is deployed.
- For a self-hosted page: the person gave its URL, `npx tina-miniapp url` recorded it, and
  `doctor` passes against the live host. If the page is not live yet, say so and leave
  `embed_url` unset rather than filling it in.
- For a hosted backend: every route it needs is under `/api/`, it answers there after the
  deploy (a 401 without an identity is an answer), and a server that keeps files is a
  `container` with a `volume`, not a `lambda`.
- Every scope in the manifest is one the code actually uses.
- If the app was meant to match the workspace, its type in the sandbox is the same face as
  the header above it. A different face means the font link is missing. An app with its own
  design is supposed to look different, so this one does not apply.
