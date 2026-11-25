---
name: figma-opencode-mcp
description: "Use whenever operating the Figma MCP server: creating, editing, cloning, syncing, or inspecting designs, wireframes, screens, or components in Figma, or when a figma.com URL appears. Field-tested call mechanics for tools.figma.* inside execute, plus sandbox gotchas (parent-relative coords, revert-on-throw, variable binding) that the official plugin docs don't state. Load before the first figma tool call."
---

# Figma MCP — operational playbook

Distilled from a long production session of building/editing wireframes via the
Figma MCP. These are empirically verified behaviors, not guesses.

## 1. Tool access model

Figma MCP tools are **only callable inside `execute` (Code Mode)** as
`tools.figma.<name>(...)` (or `tools["figma"][...])`. They are NOT direct tools.

Core calls:
```js
const s  = await tools.figma.create_session({ fileKey, description });            // → { sessionId } (UUID)
const r  = await tools.figma.use_figma({ sessionId, fileKey, code, description, skillNames }); // mutating + read-only plugin code
const md = await tools.figma.get_metadata({ sessionId, fileKey, nodeId? });
const sh = await tools.figma.get_screenshot({ sessionId, fileKey, nodeId, maxDimension? });
await tools.figma.search_design_system({ ... });   // only for PUBLISHED libraries
await tools.figma.get_libraries({ ... });
```

- `fileKey`: from URL `figma.com/design/<fileKey>/...`; `node-id=958-390` → `958:390`.
- Create a **fresh session per task**; old sessionIds go stale.
- Pass the same `sessionId` to every file tool call.
- On every mutating `use_figma` include `skillNames: "resource:figma-use,resource:figma-generate-design"` (skills loaded from the server's MCP resources, prefixed `resource:`).
- If the design system is LOCAL (components on canvas, not a published library), skip `search_design_system` and reuse in-file components found via inspection.

## 2. Standard workflow per task

1. `create_session` → keep sessionId.
2. Inspect: `get_metadata`, then a **read-only** `use_figma` dump that returns the node tree (ids, names, sizes, coordinates) of the area you'll touch. Never assume the file matches your last look — users reorganize between rounds.
3. Plan the mutation; check the target area is free (occupancy guard) and the change isn't already applied (idempotency guard).
4. ONE `use_figma` call per coherent batch: build, then **verify inside the same call** (readback of created/mutated ids, bounds/overlap math) and return the results.
5. `get_screenshot` at the end — URLs are short-lived and for the USER; if the model lacks image input, validation is structural (return computed bounds/overflow/overlap from the plugin call), not visual.

## 3. Plugin sandbox rules (figma-use)

- `return` is the only output channel — no `figma.closePlugin()`/`notify()`/`console.log`. Code is auto-wrapped async: top-level `await` OK, no async IIFE.
- `await figma.loadFontAsync({ family, style })` before creating/mutating ANY text node.
- Colors are 0–1 range, not 0–255. Fills/strokes are **read-only arrays**: clone, modify, reassign (`node.fills = [...]`).
- Page context RESETS to the first page every call: set the page every call, max once per call.
- Code < 100 KiB; no base64. On error, obey `safeToRetryWithoutCanvasRead` in the response before retrying.
- Helpers: `figma.createAutoLayout(dir, {props})`, `node.query(selector)`, `node.set(props)`, `node.clone()`, `instance.setProperties(...)`; text overrides on instance children via finding the child TEXT node.
- `layoutSizing` HUG/FILL only after the child is appended to its auto-layout parent; `resize()` before setting sizing modes.
- MUST return all created/mutated node IDs.

## 4. Field-tested gotchas (each cost a failed call — trust these)

1. **x/y getter AND setter are PARENT-RELATIVE**, not absolute. Cross-level math must chain parent offsets explicitly; naive subtraction produces phantom "displacement" readings when the file is fine. Same-call `n.y = n.y + delta` is safe.
2. **use_figma REVERTS all mutations if plugin code throws an uncaught error** (atomic rollback — no strays, no partials). Non-throwing wrong runs DO persist. Consequence: wrap every readback/verification in try/catch so a bad deref can't throw the whole build away.
3. **`node.clone()` does NOT attach to the original's parent** — clones land at page root with junk coordinates. Pattern that works: clone/create → `parent.appendChild(node)` → then set position/props.
4. **`figma.setCurrentPageAsync('<page-id-string>')` fails** ("node with id 'undefined' does not exist"). Use `figma.root.children.find(p => p.name === '<page>')`.
5. **Variable lookup**: `collection.variableIds` + `figma.variables.getVariableByIdAsync(id)` → name→Variable map. `Object.values(collection.variables)` fails (no such property).
6. **Variable names are namespaced** (`palette/rose`, `radius/12`). Bare-name lookup fails SILENTLY → build the map keyed by both full name and post-`/` suffix.
7. **`paint.boundVariables.color` is a VariableAlias with property `.id`**, not `.variableId` — a checker using `.variableId` reports false negatives on bindings that are actually fine.
8. **Bindings**: fills via `figma.variables.setBoundVariableForPaint(p, 'color', v)` — it returns a NEW paint, capture and reassign into `node.fills`. Radii via 2-arg `node.setBoundVariable('topLeftRadius', v)` per corner (fields `topLeftRadius`/`topRightRadius`/`bottomLeftRadius`/`bottomRightRadius`; NO 'Corner' infix; 3-arg form errors).
9. **Scale instance art via `instance.rescale(f)`, not `resize()`** — resize doesn't scale instance children.
10. **Anchor finders on verified node NAMES**, never loose structural predicates (a generic "frame-with-frame-children" matcher grabs the wrong node; an unanchored `/pass/i` regex also matches "PASSWORD CHECKLIST").
11. **Guard before mutating**: dump current state (users edit between rounds — treat current file as truth), check target rects are free of collisions, abort if `/pattern/i` of the intended change already exists.
12. **Text width estimate**: `0.62 × fontSize × charCount` flags likely overflow; flags under ~10% over are safe to ignore.

## 5. Minimal mutating call template

```js
await tools.figma.use_figma({
  sessionId,                       // from create_session
  fileKey,
  skillNames: "resource:figma-use,resource:figma-generate-design",
  description: "Build X in screen Y",
  code: `
    const page = figma.root.children.find(p => p.name === 'Run');
    await figma.setCurrentPageAsync(page);
    await figma.loadFontAsync({ family: 'Nunito Sans', style: 'Regular' });
    const screen = await figma.getNodeByIdAsync('<screenId>');
    const body = screen.findOne(n => n.name === 'Body');
    // ... build: create → appendChild → set props ...
    // verify INSIDE try/catch (a throw reverts the whole call):
    try {
      const made = body.children.map(n => ({ id: n.id, name: n.name, y: n.y, h: n.height }));
      return { made, overflow: body.height - Math.max(...body.children.map(n => n.y + n.height)) };
    } catch (e) { return { verifyError: String(e) }; }
  `,
});
```

If a call fails: fix the cause, then re-run the whole script — mutations are
atomic per call, and if it threw before mutating, a full re-run is safe.
