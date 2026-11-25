# figma-opencode-mcp

An [OpenCode](https://opencode.ai) skill that makes AI models reliably able to
drive the [Figma MCP server](https://mcp.figma.com) — placing frames, cloning
screens, binding variables, converting to auto-layout — without the fumbling
that fresh sessions usually show.

**Why this exists:** most models fumble the Figma MCP in a new session. The
figma tools are only reachable inside Code Mode (`execute`), and the plugin
sandbox has behaviors the official docs don't state — parent-relative
coordinates that produce phantom readings, whole-batch reverts on an uncaught
throw, variables that fail silently when looked up by bare name. This skill
injects a field-tested playbook on demand.

## Install

### 1. Connect the Figma MCP server

```sh
opencode mcp add figma --global --url https://mcp.figma.com/mcp
```

Then sign in: `/mcps` → figma → sign in (or `opencode mcp auth figma`).

> **Sign-in fails?** Figma's OAuth client allowlist doesn't include OpenCode
> yet ([sst/opencode#5636](https://github.com/sst/opencode/issues/5636),
> [#5583](https://github.com/sst/opencode/issues/5583)). The community
> [mcp-auth-helper](https://github.com/sdaoudi/mcp-auth-helper) works around
> that, but it writes tokens to the legacy `mcp-auth.json` file, which current
> OpenCode no longer reads — V2 keeps MCP credentials in its SQLite database
> instead. Until Figma allowlists OpenCode, browser sign-in may fail.

### 2. Add the skill

Global (every project):

```sh
git clone https://github.com/rv-vr/figma-opencode-mcp ~/.config/opencode/skills/figma-opencode-mcp
```

Or project-only:

```sh
git clone https://github.com/rv-vr/figma-opencode-mcp .opencode/skills/figma-opencode-mcp
```

That's it. A directory containing `SKILL.md` is one skill; the folder name is
the skill ID. OpenCode hot-reloads skills, so no restart is needed — ask your
model to use Figma and it loads the playbook before the first call.

## What's inside

- **Tool access model** — how figma tools are called inside `execute`, which
  calls every workflow needs, and the flags mutating calls must carry.
- **Standard workflow** — inspect first, guard against your own edits between
  rounds, one mutation batch with same-call verification, screenshots for you
  (the model verifies structurally).
- **Plugin sandbox rules** — fonts, fills, layout sizing, page context,
  return values.
- **Twelve field-tested gotchas** — the ones that cost real sessions:
  parent-relative `x`/`y`, revert-on-throw, clone placement, variable lookup
  and binding, corner radii, art rescale, and more.

Cross-harness note: the sandbox gotchas apply to any client driving Figma's
plugin sandbox. The tool-access section is specific to OpenCode's Code Mode —
if you port this to another harness, rewrite that section for its
tool-calling convention.

## Provenance

Distilled from production wireframe-building sessions on a real Figma file,
then verified in a fresh OpenCode session with no prior context: frame
placement on the correct page, parent-relative clone coordinates, occupancy
guards, auto-layout with correct gaps, and the revert-on-throw safety pattern
all behaved as documented.

## License

MIT — see [LICENSE](LICENSE).
