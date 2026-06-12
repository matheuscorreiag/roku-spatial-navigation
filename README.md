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

The recommended way is a **base screen**: your screens extend `SpatialNavScreen`,
which owns focus and arrow navigation for everything beneath it.

1. Make your screen `extends SpatialNavScreen`.
2. Mark focusable nodes in XML with `spatialFocusable="true"`.
3. Activate it once: a router calls `handleFocus` on the screen when its route
   activates, or you call `focusDefault()` yourself after mounting.

On each arrow press the screen reads the marked nodes' rects and moves focus to the
nearest eligible one. A candidate is only eligible if it overlaps the current node on
the perpendicular axis (so Left never jumps diagonally to something that's really
_above_ you), which keeps movement predictable while still allowing big gap jumps
within a row/column.

> Even a single-screen app uses a `SpatialNavScreen` — make your screen extend it and
> land focus with one `focusDefault()` call (see [Quick start](#quick-start)).

---

## Install (copy the files)

No package manager yet — grab the files from [`src/`](src/) (download the repo zip
and drag them in) and drop them into your channel:

| File                       | Put it in                         |
| -------------------------- | --------------------------------- |
| `src/SpatialNav.bs`        | `source/SpatialNav.bs`            |
| `src/SpatialNavScreen.xml` | `components/SpatialNavScreen.xml` |
| `src/SpatialNavScreen.bs`  | `components/SpatialNavScreen.bs`  |

`SpatialNavScreen.bs` imports the engine as `pkg:/source/SpatialNav.bs`, so keep
`SpatialNav.bs` in `source/` (or update that one import).

> Written in **BrighterScript** (`.bs`) — the standard modern Roku toolchain
> ([@rokucommunity/brighterscript](https://github.com/rokucommunity/brighterscript)).
> For a plain‑BrightScript project, transpile with `bsc` or rename to `.brs` and
> swap the namespace calls.

---

## Quick start

**HomeScreen.xml** — a screen is just a `SpatialNavScreen` with focusables in it:

```xml
<component name="HomeScreen" extends="SpatialNavScreen">
    <children>
        <!-- any focusable components; markers opt them in -->
        <MyButton text="One"   translation="[80, 100]" spatialFocusable="true" spatialDefault="true" />
        <MyButton text="Two"   translation="[80, 200]" spatialFocusable="true" />
        <MyButton text="Three" translation="[80, 300]" spatialFocusable="true" />
    </children>
</component>
```

**Activate it.** If you use a router, it calls `handleFocus` on the screen for you on
every route change — nothing to write. Without a router, land focus once from
whatever hosts the screen:

```xml
<!-- MainScene.xml -->
<component name="MainScene" extends="Scene">
    <children>
        <HomeScreen id="home" />
    </children>
</component>
```

```brightscript
' MainScene.bs
sub init()
    m.top.findNode("home").callFunc("focusDefault")
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

| Field              | Meaning                                                                                                                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spatialFocusable` | Opt this node into navigation. It's treated as a **leaf** — the walk won't descend into it, so a node that owns its own internals (a custom button, a `MarkupList`) is registered whole. |
| `spatialDefault`   | This node gets focus when its scope activates (`focusDefault` / `sendFocusTo`).                                                                                                          |
| `spatialScope`     | This whole subtree is a **separate** scope (a modal/sheet). The base walk skips it; you activate it with `sendFocusTo` (see below).                                                      |

You can also set these in code for built‑in nodes, e.g.
`myMarkupList.addFields({ spatialFocusable: true })`.

---

## Activating a screen

A screen needs exactly one nudge to build its scope and land initial focus, because
SceneGraph won't run an inherited `init()` once your screen defines its own. Two ways:

- **`handleFocus(data)`** — the hook a router calls when a route activates. Most
  routers (e.g. sgRouter) call it automatically, so router apps write _zero_ focus
  code and get correct focus on every navigation.
- **`focusDefault()`** — call it once yourself after the screen is mounted (see Quick
  start) when there's no router.

After that it's hands-off. The screen also **re-discovers focusables on every arrow
press** (a cheap re-walk; the focused node is recovered from the live focus chain),
so nodes you add or remove at runtime just work — no `refresh()` needed. `refresh()`
still exists for forcing a rebuild outside a keypress.

---

## Modals & sheets (trapped focus)

Focus is a **stack** of scopes. Opening a modal should _trap_ focus inside it
(arrows mustn't reach what's underneath) and closing it should restore focus to
where it was. From inside the screen, that's two inherited calls:

```brightscript
' open: trap focus inside the sheet's subtree
m.sheet.callFunc("open")
sendFocusTo(m.sheet)

' close: pop back and restore focus to where the parent scope left off
m.sheet.callFunc("close")
releaseFocus()
```

Mark the sheet's root `spatialScope="true"` so the base navigation ignores it until
you open it. Non-arrow keys (like **Back**) land in the screen's `onScreenKey`
override — close the sheet from there:

```brightscript
' the base consumes arrows itself; override onScreenKey for everything else
function onScreenKey(key as string) as boolean
    if key = "back" and m.sheetOpen
        closeSheet()   ' -> releaseFocus()
        return true
    end if
    return false
end function
```

---

## Using a router (e.g. sgRouter)

If your router mounts screens and calls `handleFocus` on activation, the base screen
plugs straight in — every route gets focus and arrow navigation for free, and the
modal stack lives on the screen where its lifecycle already is.

One Roku constraint to know: a screen can extend **one** base. If your router
requires its screens to extend _its_ view base (sgRouter's screens extend
`sgrouter_View`), you can't also `extends SpatialNavScreen`. In that case, copy
`SpatialNavScreen.bs`'s body into your router's base view — it's self-contained (only
imports `SpatialNav.bs`), so your `Screen extends sgrouter_View` gains the exact same
behavior. Your concrete screens then declare focusables and override `onScreenKey`,
unchanged.

---

## Run the example

A complete, sideloadable demo lives in [`examples/minimal`](examples/minimal): a
`DemoContent` screen (a `SpatialNavScreen`) with a 2‑D grid of focusable cards plus a
trapped modal, hosted by a thin scene that lands initial focus.

```bash
cd examples/minimal
npm install
npm run build         # -> out/spatialnav-demo.zip
```

Sideload `out/spatialnav-demo.zip` from your Roku's Development Application
Installer (`http://<roku-ip>/` → Upload). Then:

- **Arrow keys** move focus across the grid by geometry.
- **OK** on _Open Modal_ opens the sheet and traps focus in it.
- **Back** releases the trap and restores focus.

---

## API

**`SpatialNavScreen`** (base component) — subclass it; override / `callFunc` these:

| Member              | Description                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `handleFocus(data)` | Router activation hook: (re)builds the scope and lands default focus. Returns `true`. Routers call this for you.  |
| `focusDefault()`    | Land focus on the `spatialDefault` node (or the first focusable). Call once after mounting if you have no router. |
| `onScreenKey(key)`  | **Override** for non-arrow keys (Back, etc.). Default returns `false`. The base handles arrows before this.       |
| `sendFocusTo(node)` | Open a trapped sub‑scope over `node` (a modal/sheet).                                                             |
| `releaseFocus()`    | Close the top sub‑scope and restore focus underneath.                                                             |
| `refresh()`         | Force a scope rebuild. Rarely needed — focusables are re-discovered each arrow press.                             |

**`SpatialNav`** (engine namespace) — `scope(root)`, `create()`, `register(nav,
node)`, `registerAll(nav, nodes)`, `clear(nav)`, `focus(nav, node)`,
`focusDefault(nav)`, `handleKey(nav, key)`.

---

## License

MIT — see [LICENSE](LICENSE).
