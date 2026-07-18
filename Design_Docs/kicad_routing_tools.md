# KiCAD Routing Tools & Techniques

Reference for interactive routing, autorouting, plugins, and workflow tips.
Scoped to KiCAD 8/9/10 conventions. Written during the PN102-A4 config-standardization
session — see `.konnect/project.json` for this project's netclass/component-class setup
that these tools operate against.

---

## Built-in Interactive Router

The main routing tool. Press `X` to start.

### Modes (cycle with `Shift+R`)

| Mode | Behavior | Use when |
|---|---|---|
| **Highlight collisions** | Shows conflicts, doesn't force through | Tight regions, careful placement |
| **Shove** | Pushes existing traces to make space | Mid-density, routing thick power |
| **Walkaround** | Auto-routes around obstacles (default) | Single connections, sparse areas |

### While routing

| Key | Action |
|---|---|
| `W` | Cycle through `track_widths` preset in `.kicad_pro` |
| `V` | Place a via (switch layer) |
| `/` | Toggle 45° / 90° / free angle |
| `E` | Edit segment properties mid-route |
| `D` | Drag mode (rubber-band existing tracks) |
| `Esc` | Cancel current segment |

### Push-and-shove caveat

Shove doesn't cross netclass clearance boundaries — a wide `Motor` trace won't shove a
tight `Analog` trace out of its (wider) clearance envelope. Route the tighter-clearance
class first when they must coexist.

---

## Differential Pair Router

Press `6` (or menu) to start a diff pair route. Both traces route in parallel maintaining
the netclass `diff_pair_gap`. Uses `diff_pair_dimensions` presets from `.kicad_pro`.

Auto-detects pair candidates from net names ending in `_P` / `_N`, `_+` / `_-`, or
matching regex patterns (we have `^/?[A-Z][0-9]+_[PN]$` registered).

---

## Length Tuning

| Key | Tool |
|---|---|
| `7` | Single-track length tuning (meander) |
| `8` | Diff pair length tuning (both together) |
| `9` | Diff pair skew (P-N length matching) |

Adds serpentine wiggles to hit a length target. Configuration lives in
`.kicad_pro.board.design_settings.tuning_pattern_settings`.

Only useful when:
- A netclass or per-net constraint declares a length range, OR
- You're manually matching to a known-good reference trace length

---

## Track Edits

| Action | How |
|---|---|
| Delete segment | Hover + `Del` |
| Rip up whole net | Select net + `Ctrl+Del` |
| Merge collinear tracks | Draw new segment on top of existing (same net + layer) |
| Slice/split | Right-click track → various edit options |
| Batch delete | Select multiple + `Del` |

---

## Visibility & Focus

| Feature | How | Purpose |
|---|---|---|
| Ratsnest toggle per net | `Nets` panel (`Ctrl+K`) | Hide clutter for nets you've routed |
| Highlight net | Click pad + `` ` `` (backtick); clear with `~` | Trace where a signal goes |
| Hide layer | `Ctrl+H` or Layers panel | Focus on active layer only |
| Layer opacity | Layers panel slider | See through top layer to bottom |

---

## Copper Zones (Pours)

- Draw filled area on any copper layer, assign net (usually GND)
- **`B`** — Fill all zones (compute actual copper)
- **`Ctrl+B`** — Unfill (visualize routing without pours in the way)
- Right-click zone → Zone Properties for per-zone overrides (thermal relief, pad connection)
- Zone defaults: `.kicad_pro.board.design_settings.defaults.zones`
  - `thermal_relief_gap: 0.4mm`
  - `thermal_relief_spoke_width: 0.4mm`
  - `min_thickness: 0.2mm`
  - `min_island_area: 2mm²` (catches slivers, keeps small legitimate pours)

---

## Autoroute — Freerouting

External Java-based autorouter, KiCAD-integrated via Specctra DSN export/import.

### Manual invocation

1. **File → Export → Specctra DSN** (from PCB editor)
2. Run Freerouting on the `.dsn` file
3. **File → Import → Specctra Session** (imports the routed `.ses`)

### Via Konnect

The `integration` toolset (not loaded by default) has a `run_freerouting` tool that
handles the round-trip automatically. Load with `load_toolset("integration")`.

### Notes

- Respects netclass constraints — all the netclass work in `.kicad_pro` transfers
- Handles moderate complexity well; not as good as commercial routers but free
- KiCAD's own autorouter was removed after v5 — Freerouting is the only free option

---

## Konnect PCB Routing Tools (`pcb_routing` toolset)

Programmatic routing for agent-driven or script-driven workflows.

| Tool | Purpose |
|---|---|
| `route_trace` | Add a trace segment (start XY, end XY, layer) |
| `route_pad_to_pad` | Route between two named component pads with L-bend logic |
| `route_differential_pair` | Matched-pair route with declared gap |
| `add_via` | Plunge a via at coordinates, assign to net |
| `add_copper_pour` | Programmatic zone insertion |
| `modify_trace` | Replace a track's parameters by delete + re-add |
| `query_traces` | Inspect existing routing (filter by net/layer) |
| `get_nets_list` | Full net inventory |
| `create_netclass` | Programmatic netclass creation |
| `assign_net_to_class` | Route a net to a specific class |

Use case: "Route all decoupling cap ground pads to the nearest EPAD" as an agent
operation across dozens of caps.

---

## Community Plugins

### Via Stitching
- **jsreynaud/kicad-via-stitching** on GitHub — grid-fills a selected zone with GND vias
- Fills the "no native stitching density enforcement" gap
- KiCAD 9+ has partial built-in via-fill in the Fill Zones dialog

### InteractiveHtmlBom
- Generates a clickable HTML BOM highlighting components on a rendered board image
- Invaluable during routing when you need to find "which cap is C47?" quickly
- Also useful during assembly and debug

### Length Matching Utilities
- Various community scripts report per-net length statistics
- Useful *before* running length tuning to know how much you actually need to add
- Search: `kicad length matching plugin`

### KiKit
- Panelization for volume production (multi-up boards)
- Generates fab-ready panels from single-board files
- Adds V-scores, mouse bites, tooling holes, fiducials
- Not routing per se, but complementary at the tail end

### Saturn PCB Toolkit (Windows external)
- Trace impedance calculator
- Current-carrying capacity calculator (temp rise vs width vs copper weight)
- Via inductance estimator
- Standalone tool, not a plugin — free download

---

## Routing Workflow (order of operations)

Sequence that minimizes rework:

1. **Verify component placement first.** Routing quality is 80% determined by placement.
   Two chips that need a wide power link belong close together, not corners.

2. **Route power first.** Wide, direct paths from regulators to loads. Use the
   Motor/Heater/Line/power-rail netclasses for width. Establishes the ground pour
   structure implicitly.

3. **Route clocks and critical signals second.** Shortest path, impedance-controlled
   where relevant. Analog signals fall here for this board.

4. **Route buses third.** Address/data buses together, one layer per bus direction
   where possible.

5. **Route general signals fourth.** Walkaround router handles most of this.

6. **Fill copper pours last.** `B` to fill after routing is done; the pour will
   flow around your routed tracks rather than the other way around.

---

## Techniques That Save Time

### Rip-up-and-retry beats "fix in place"

If a region's routing is 60% good, ripping the entire region and re-routing with
better ordering is faster than tweaking individual traces around each other.

### Use netclass colors as visual DRC

Per-netclass PCB colors give at-a-glance verification without needing DRC to run.
This board's palette:

| Class | Color |
|---|---|
| `+12V` | cyan |
| `+24V` | red |
| `+3V3` | magenta |
| `+3V3A` | blue |
| `Motor` | orange |
| `Heater` | red |
| `Line` | purple |
| `Analog` | green |

If a trace is the wrong color for its function, it's in the wrong class.

### Route from congested to sparse

BGA / QFN / connector clusters constrain the routing pattern. Everything else adapts
to whatever comes out of those clusters. If you route the sparse area first and then
try to escape a QFN into a fully-routed neighborhood, you're rebuilding.

### Escape routing on fine-pitch parts

For dense QFN / BGA parts:
- Fanout with short traces + via to inner layer
- Use `route_pad_to_pad` from Konnect for systematic escape patterns
- Consider dedicated "escape row" of vias just outside the courtyard

### Layer discipline

- Top layer: primary routing direction (e.g., horizontal)
- Bottom layer: perpendicular (e.g., vertical) with GND pour where possible
- On this board specifically (analog + digital): top for analog signals, bottom
  for digital return with GND pour — see `.konnect/project.json.design_rules`

---

## Keyboard Quick-Reference

| Key | Action |
|---|---|
| `X` | Start routing |
| `V` | Add via mid-route |
| `W` | Cycle track width preset |
| `Shift+R` | Cycle router mode (highlight / shove / walkaround) |
| `/` | Toggle angle mode (45 / 90 / free) |
| `E` | Edit properties (routing or selection) |
| `D` | Drag with rubber-band |
| `6` | Route differential pair |
| `7` / `8` / `9` | Length tune single / diff pair / skew |
| `B` | Fill all zones |
| `Ctrl+B` | Unfill zones |
| `` ` `` | Highlight net (backtick) |
| `~` | Clear highlight (tilde) |
| `Ctrl+H` | Toggle layer visibility |
| `Ctrl+K` | Open Nets panel |
| `Ctrl+Del` | Rip up whole net |
| `Esc` | Cancel |

---

## What KiCAD Can't Do Natively

Documenting the gaps so you don't waste time looking:

- **Autoroute** — Removed in v6+. Use Freerouting.
- **Stitching density enforcement** — No "GND vias every 5mm" rule. Manual or plugin.
- **Return via near signal via** — No proximity rule. Manual placement.
- **Via-in-pad detection with warning severity** — `disallow via` maps to `items_not_allowed`
  which is currently `error` globally; can't override per-rule. See note in `.kicad_dru`.
- **Auto-add creepage slots** — You draw the slot; DRC (edge-aware in KiCAD 8+) checks the result.
- **Enforce isolation barrier not crossed** — No native rule; use copper keep-out zones manually.
- **Per-rule severity override** — All `disallow` rules share `items_not_allowed` severity.
