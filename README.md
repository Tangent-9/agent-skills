# agent-skills

Agent skills from Tangent 9, packaged so Claude Code can install them as plugins.

## Skills

| Skill | What it does |
| ----- | ------------ |
| [tina-miniapp](plugins/tina-miniapp/skills/tina-miniapp/SKILL.md) | Build a TINA mini app end to end: scaffold, manifest, SDK, backend, sandbox, submit |

## Install as a plugin

Add this repo as a marketplace, then install from it:

```bash
claude plugin marketplace add Tangent-9/agent-skills
```

```bash
claude plugin install tina-miniapp@tangent-9-agent-skills
```

From inside Claude Code, `/plugin` opens the same thing with a picker.

To work against a local checkout instead of GitHub, point the marketplace at the directory:

```bash
claude plugin marketplace add /Users/hmu/Codes/t9/agent-skills
```

Updating the repo and running `claude plugin marketplace update tangent-9-agent-skills` pulls
new versions of anything installed from it.

## Install a skill without the plugin

Skills are plain directories, so a symlink works:

```bash
ln -s "$PWD/plugins/tina-miniapp/skills/tina-miniapp" ~/.claude/skills/tina-miniapp
```

## Layout

```
.claude-plugin/marketplace.json     the marketplace, listing every plugin below
plugins/<plugin>/
  .claude-plugin/plugin.json        the plugin's own manifest
  skills/<skill>/SKILL.md           the skill, with its references/ beside it
```

One plugin per directory under `plugins/`, listed in `marketplace.json` by relative path. A
plugin can carry more than one skill; each gets its own directory under `skills/`.

## Adding a skill

1. Create `plugins/<plugin>/skills/<skill>/SKILL.md` with `name` and `description` frontmatter.
   The description is what Claude matches against, so write it as the trigger.
2. Add or update `plugins/<plugin>/.claude-plugin/plugin.json`.
3. Add the plugin to the `plugins` array in `.claude-plugin/marketplace.json`.
4. Bump the plugin `version` so installed copies pick the change up.
