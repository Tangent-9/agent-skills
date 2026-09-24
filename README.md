# agent-skills

Agent skills from Tangent 9. Install them with `npx skills add`, or as Claude Code plugins.

## Skills

| Skill | What it does |
| ----- | ------------ |
| [tina-miniapp](plugins/tina-miniapp/skills/tina-miniapp/SKILL.md) | Build a TINA mini app end to end: scaffold, manifest, SDK, backend, sandbox, submit |

## Install with the skills CLI

Works with Claude Code, Codex, Cursor and the rest of the agents the
[skills](https://github.com/vercel-labs/skills) CLI supports:

```bash
npx skills add Tangent-9/agent-skills
```

```bash
npx skills add Tangent-9/agent-skills --skill tina-miniapp -a claude-code
```

Add `-g` to install to the user directory instead of the current project, and `--list` to see
what a repo carries without installing it.

## Install as a Claude Code plugin

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

## Install a skill by hand

Skills are plain directories, so a symlink works:

```bash
ln -s "$PWD/plugins/tina-miniapp/skills/tina-miniapp" ~/.claude/skills/tina-miniapp
```

## Layout

```
.claude-plugin/marketplace.json     the marketplace, listing every plugin below
plugins/<plugin>/
  .claude-plugin/plugin.json        the plugin's own manifest
  skills/<skill>/SKILL.md           the skill itself, with its references/ beside it
skills/<skill>                      symlink to the skill above
```

One plugin per directory under `plugins/`, listed in `marketplace.json` by relative path. A
plugin can carry more than one skill; each gets its own directory under its `skills/`.

The skill files live inside the plugin because a plugin manifest cannot reference a path
outside its own directory. The top-level `skills/` holds symlinks so the skills CLI and
anyone reading the repo find skills where they expect them. Claude Code reads the real files
and never follows the symlinks.

## Adding a skill

1. Create `plugins/<plugin>/skills/<skill>/SKILL.md` with `name` and `description` frontmatter.
   The description is what an agent matches against, so write it as the trigger.
2. Symlink it: `ln -s ../plugins/<plugin>/skills/<skill> skills/<skill>`.
3. Add or update `plugins/<plugin>/.claude-plugin/plugin.json`.
4. Add the plugin to the `plugins` array in `.claude-plugin/marketplace.json`.
5. Bump the plugin `version` so installed copies pick the change up.
6. Check both installers still see it:

```bash
claude plugin validate . && npx skills add . --list
```
