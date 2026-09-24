---
name: tina-miniapp
description: Build a TINA mini app end to end. Scaffold it, write its tina-miniapp.json manifest, wire the browser SDK and the app's own backend, run it locally in the sandbox, then check and submit it. Use this whenever someone wants to build, extend, debug, test or ship an app for the TINA workspace, or mentions tina-miniapp.json, @tangent-9/miniapps-sdk, create-tina-miniapp, tina-sandbox, app scopes and plans, lifecycle hooks, a mini app agent, or an app that gets framed by TINA. Reach for it even when the request says "an app for TINA" or names a template instead of saying "mini app".
---

# Build a TINA mini app

A mini app is three things: a page the TINA workspace frames in an iframe, a manifest
saying what the app may do, and usually a backend of your own. The division of labour is
the thing to hold onto. TINA owns the subscription, the session and the consent. Your
service owns the records the app is actually about. That is true of the apps TINA
publishes itself, so a design that wants TINA to store the app's data is fighting the
platform.

## Scaffold, do not hand-write

The CLI ships templates that already validate against the platform schema. Starting from
one is faster than assembling files and finding out at submit which fields were wrong.

```bash
npx @tangent-9/create-tina-miniapp <dir> --template <name>
```

| Template | Choose it when |
| ----------------- | ------------------------------------------------------ |
| `react-vite` (default) | the app has a screen and an agent |
| `fullstack` | the app also keeps records, so it needs its own backend |
| `agent-companion` | the app is an agent and nothing else |

`agent-companion` is a manifest and a README, nothing to install. That matters for one
thing only: with no `node_modules` there is no local `tina-miniapp` binary, so its commands
are `npx @tangent-9/create-tina-miniapp <cmd>`. The other two templates depend on the CLI,
so `npx tina-miniapp <cmd>` resolves locally.

After `npm install`, read `node_modules/@tangent-9/miniapps-sdk/README.md`. It is the SDK's
own reference, it ships with the exact version installed, and it is more current than
anything restated here. Read it before writing page code.

## The loop

```bash
npx tina-miniapp dev --sandbox  # the page, its backend and a local stand-in for TINA
npx tina-miniapp dev            # the page and its backend, for a real workspace to frame
npx tina-miniapp doctor         # framing headers and the hook endpoint, against a running app
npx tina-miniapp validate       # the manifest against the platform's own schema
npx tina-miniapp pack           # a .tinapkg of the manifest and listing assets
TINA_PARTNER_KEY=tinapk_… npx tina-miniapp submit
```

Run `validate` after every manifest edit rather than at the end. It carries the platform's
schema, so what it accepts is what upload accepts, and its messages name the field.

The listing's logo is `assets/icon.png` (or `.jpg`, `.webp`): square, up to 512 KB, and
`submit` uploads it. SVG is refused, because the platform serves logos to anyone and an
SVG can carry script. Without one, the catalogue shows the app's initial.

`doctor` needs the app running. It catches the two failures that otherwise surface as a
blank frame in front of a customer: a missing `frame-ancestors` header, and a hook endpoint
that does not answer the registration challenge.

## Run it before TINA knows about it

`dev --sandbox` also starts `tina-sandbox`, which ships in the SDK, and points the backend
at it for that run. Open `http://localhost:4100`. The sandbox frames the page the way the
workspace does and answers the bridge from fixtures, so the app runs with no account, no
partner key and nothing published. It plays TINA to the backend too. It publishes a key
set and signs identity assertions and hooks with a throwaway key, so `verifyHook` and
`verifyIdentity` run unchanged.

Use it for what is hard to reach on a real workspace:

- Switch the person, their roles and what they granted, then relaunch. A declined scope
  should degrade, not break the screen.
- Kill or expire the session, and check the app shows its ended state.
- Send any hook, obligations included, and watch the acknowledgement land:
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

What the host serving the page must do is permit framing:

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
`Masked`, `ToastProvider`) is styled by `theme.css` and matches the workspace, so an app
built on it does not look bolted on. On Tailwind v4, import
`@tangent-9/miniapps-sdk/tailwind.css` for the same tokens as utilities.

## Writing the backend

The server half verifies what TINA sends. It needs no secret, because both message types
are Ed25519-signed and checked against a public key set.

```ts
import { createTina } from "@tangent-9/miniapps-sdk/server";
const tina = createTina({ appId: process.env.TINA_APP_ID, issuer: process.env.TINA_ISSUER });
```

Three things reliably go wrong here:

**Hook signatures cover raw bytes.** Verify against `req.rawBody`. A body that has been
through `JSON.parse` and back is a different string and will not verify, and most web
frameworks parse before your handler sees it. Configure the raw body first.

**Identity assertions live two minutes and describe that moment.** Verify per request and
do not cache the claims. Holding them past `exp` keeps a revoked scope alive, which is why
`exp` is enforced with no tolerance.

**Obligations arrive whether or not the manifest asks for them.**
`user.deletion_requested`, `tenant.offboarding` and `data.export_requested` carry data
protection duties TINA cannot discharge, because TINA does not hold the app's records. Each
arrives with an `ack_token`. Act, then acknowledge, even when the answer is
`not_applicable`. One nobody answers is escalated to platform staff with the app named.

Hooks retry eight times over about a day on any non-2xx except 4xx, which stops
immediately, so handlers must be idempotent on `deliveryId`. `event.sequence` counts up per
app, and a gap means a delivery was missed and the app should reconcile.

Authorization is a backend decision against a verified token:

```ts
const { roles } = await tina.verifyIdentity(bearerToken);
if (!roles.includes("hr-admin")) return reply.code(403).send();
```

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
- The app works in the sandbox with each scope declined in turn, and every obligation hook
  gets acknowledged.
- `npx tina-miniapp doctor` passes against the running app.
- Opening `embed_url` directly shows the "has to run inside the TINA workspace" state.
  That is correct, not a bug: there is no session until the workspace frames the app.
- Every scope in the manifest is one the code actually uses.
