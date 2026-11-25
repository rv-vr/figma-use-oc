# figma-use-oc

OpenCode skill that makes AI models reliably able to drive the Figma MCP
server — placing frames, cloning screens, binding variables, auto-layout —
in fresh sessions, where they usually fumble. `oc` = OpenCode.

A field-tested playbook for `tools.figma.*` inside `execute`, plus
plugin-sandbox gotchas the official docs don't state: parent-relative
coordinates, whole-batch revert on an uncaught throw, variables failing
silently when looked up by bare name.

## Prerequisites

A working `figma` MCP server connection in OpenCode — this skill doesn't set
it up:

```sh
opencode mcp add figma --global --url https://mcp.figma.com/mcp
```

Auth caveat: Figma's OAuth allowlist doesn't include OpenCode yet
([#5636](https://github.com/sst/opencode/issues/5636),
[#5583](https://github.com/sst/opencode/issues/5583)), so `/mcps` sign-in may
fail. [mcp-auth-helper](https://github.com/sdaoudi/mcp-auth-helper) works
around it but writes legacy storage current OpenCode doesn't read. Auth is
out of scope here.

## Install

```sh
git clone https://github.com/rv-vr/figma-use-oc ~/.config/opencode/skills/figma-use-oc  # global
git clone https://github.com/rv-vr/figma-use-oc .opencode/skills/figma-use-oc           # project
```

Folder name = skill ID. Skills hot-reload — no restart needed.

## Contents

- Tool access model — Code Mode `execute` mechanics, mutating-call flags
- Workflow — inspect → guards → one mutation batch with same-call verify → screenshot
- Sandbox rules — fonts, fills, layout sizing, page context
- 12 field-tested gotchas

Verified end-to-end in a fresh session: page placement, parent-relative
clones, auto-layout gaps. The sandbox gotchas apply to any Figma plugin
client; the tool-access section is Code Mode-specific.

## License

MIT — see [LICENSE](LICENSE).
