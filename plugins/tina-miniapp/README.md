# tina-miniapp

A Claude Code plugin carrying one skill: building a TINA mini app.

The skill covers the whole path, from `npx @tangent-9/create-tina-miniapp` through the
`tina-miniapp.json` manifest, the browser SDK and the app's own backend, the local sandbox,
and `validate` / `doctor` / `submit`.

Claude loads it on its own when a request mentions a TINA mini app, the manifest, the SDK or
the sandbox. You can also name it directly.

- [SKILL.md](../../skills/tina-miniapp/SKILL.md)
- [Manifest reference](../../skills/tina-miniapp/references/manifest.md)

## Install

With the [skills](https://github.com/vercel-labs/skills) CLI, for Claude Code or any other
agent it supports:

```bash
npx skills add Tangent-9/agent-skills --skill tina-miniapp
```

Or as a Claude Code plugin:

```bash
claude plugin marketplace add Tangent-9/agent-skills
```

```bash
claude plugin install tina-miniapp@tangent-9-agent-skills
```
