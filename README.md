# roku-spatial-navigation

LRUD (Left/Right/Up/Down) spatial navigation for Roku SceneGraph — move focus by
**real on-screen geometry**, not hand-maintained focus indices.

You declare which nodes are focusable; the library figures out where focus should
go when the user presses an arrow, by picking the nearest neighbour in that
direction. No `setFocus` bookkeeping, no per-screen key handlers, no index math.

### Motivation

This is heavily inspired by [**bamlab/react-tv-space-navigation**](https://github.com/bamlab/react-tv-space-navigation)
and its declarative "focusables register into a root" model. The difference is the
engine: that library wraps LRUD's tree-walk, whereas Roku gives us
`node.sceneBoundingRect()` — every node's absolute on-screen rect after layout — so
here navigation is **purely geometric** (nearest neighbour in the pressed
direction), which is closer to what "spatial" usually means.

---

## How it works

1. Mark focusable nodes in XML with `spatialFocusable="true"`.
2. Wrap them in a `<SpatialNavRoot>`.
3. Call `focusDefault()` once. Arrow keys now move focus by geometry.

The root walks its subtree once, gathers the marked nodes, and on each arrow press
reads their rects and moves focus to the nearest eligible one. A candidate is only
eligible if it overlaps the current node on the perpendicular axis (so Left never
jumps diagonally to something that's really *above* you), which keeps movement
predictable while still allowing big gap jumps within a row/column.

---

## Install (copy the files)

No package manager yet — grab the files from [`src/`](src/) (download the repo zip
and drag them in) and drop them into your channel:

| File | Put it in |
|------|-----------|
| `src/SpatialNav.bs` | `source/SpatialNav.bs` |
| `src/SpatialNavRoot.xml` | `components/SpatialNavRoot.xml` |
| `src/SpatialNavRoot.bs` | `components/SpatialNavRoot.bs` |

`SpatialNavRoot.bs` imports the engine as `pkg:/source/SpatialNav.bs`, so keep
`SpatialNav.bs` in `source/` (or update that one import).

> Written in **BrighterScript** (`.bs`) — the standard modern Roku toolchain
> ([@rokucommunity/brighterscript](https://github.com/rokucommunity/brighterscript)).
> For a plain‑BrightScript project, transpile with `bsc` or rename to `.brs` and
> swap the namespace calls.

---

## Quick start

**MainScene.xml**

```xml
<component name="MainScene" extends="Scene">
    <children>
        <SpatialNavRoot id="nav">
            <!-- any focusable components; markers opt them in -->
            <MyButton text="One"   translation="[80, 100]"  spatialFocusable="true" spatialDefault="true" />
            <MyButton text="Two"   translation="[80, 200]"  spatialFocusable="true" />
            <MyButton text="Three" translation="[80, 300]"  spatialFocusable="true" />
        </SpatialNavRoot>
    </children>
</component>
```

**MainScene.bs**

```brightscript
sub init()
    ' Land focus once the UI exists. That's the only line you need.
    m.top.findNode("nav").callFunc("focusDefault")
end sub
```

Your focusable component only has to (a) carry the marker fields and (b) let arrow
keys bubble — i.e. handle `OK` and return `false` for everything else. The library
calls `setFocus(true)`; you render the focused look however you like (e.g. observe
`focusedChild`).

```xml
<component name="MyButton" extends="Group">
    <interface>
        <field id="text" type="string" />
        <field id="selected" type="boolean" alwaysNotify="true" />
        <field id="spatialFocusable" type="boolean" value="true" />
        <field id="spatialDefault"   type="boolean" value="false" />
    </interface>
    <!-- ... -->
</component>
```

### Markers

| Field | Meaning |
|-------|---------|
| `spatialFocusable` | Opt this node into navigation. It's treated as a **leaf** — the walk won't descend into it, so a node that owns its own internals (a custom button, a `MarkupList`) is registered whole. |
| `spatialDefault` | This node gets focus when its scope activates (`focusDefault` / `sendFocusTo`). |
| `spatialScope` | This whole subtree is a **separate** scope (a modal/sheet). The base walk skips it; you activate it with `sendFocusTo` (see below). |

You can also set these in code for built‑in nodes, e.g.
`myMarkupList.addFields({ spatialFocusable: true })`.

---

## Modals & sheets (trapped focus)

Focus is a **stack** of scopes. Opening a modal should *trap* focus inside it
(arrows mustn't reach what's underneath) and closing it should restore focus to
where it was. That's two calls:

```brightscript
' open: trap focus inside the sheet's subtree
m.nav.callFunc("sendFocusTo", m.sheet)

' close: pop back and restore focus to where the parent scope left off
m.nav.callFunc("releaseFocus")
```

Mark the sheet's root `spatialScope="true"` so the base navigation ignores it until
you open it. Keys the root doesn't consume (like **Back**) bubble past to your
scene, so you close the sheet from there:

```brightscript
function onKeyEvent(key as string, press as boolean) as boolean
    if press and key = "back" and m.sheetOpen
        closeSheet()   ' -> m.nav.callFunc("releaseFocus")
        return true
    end if
    return false
end function
```

---

## Engine-only usage (no SpatialNavRoot)

`SpatialNavRoot` is just a thin wrapper over [`SpatialNav.bs`](src/SpatialNav.bs).
If you'd rather drive focus from your own component:

```brightscript
import "pkg:/source/SpatialNav.bs"

sub init()
    m.nav = SpatialNav.scope(m.top)   ' gather marked focusables in this subtree
    SpatialNav.focusDefault(m.nav)
end sub

function onKeyEvent(key as string, press as boolean) as boolean
    if not press then return false
    return SpatialNav.handleKey(m.nav, key)   ' true = moved, false = bubble
end function
```

---

## Run the example

A complete, sideloadable demo lives in [`examples/minimal`](examples/minimal): a
2‑D grid of focusable cards plus a trapped modal.

```bash
cd examples/minimal
npm install
npm run build         # -> out/spatialnav-demo.zip
```

Sideload `out/spatialnav-demo.zip` from your Roku's Development Application
Installer (`http://<roku-ip>/` → Upload). Then:

- **Arrow keys** move focus across the grid by geometry.
- **OK** on *Open Modal* opens the sheet and traps focus in it.
- **Back** releases the trap and restores focus.

---

## API

**`SpatialNavRoot`** (component) — `callFunc` these:

| Function | Description |
|----------|-------------|
| `focusDefault()` | Land focus on the `spatialDefault` node (or the first focusable). Call once after the UI exists. |
| `sendFocusTo(node)` | Open a trapped sub‑scope over `node`. |
| `releaseFocus()` | Close the top sub‑scope and restore focus underneath. |
| `refresh()` | Rebuild the base scope after adding/removing focusables at runtime. |

**`SpatialNav`** (engine namespace) — `scope(root)`, `create()`, `register(nav,
node)`, `registerAll(nav, nodes)`, `clear(nav)`, `focus(nav, node)`,
`focusDefault(nav)`, `handleKey(nav, key)`.

---

## License

MIT — see [LICENSE](LICENSE).
