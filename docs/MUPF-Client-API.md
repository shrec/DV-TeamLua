# MUPF Client API

The client side of a plugin is an HTML page (`client/index.html`) rendered **in-game** in its
own window. It is drawn by the client's bundled HTML/CSS engine (**litehtml**) with JavaScript
run by **duktape** — this is *not* a full browser. Read the **Important model** section first;
it is the single biggest difference from web development.

> **Client logic is JavaScript.** There is no client-side Lua runtime — the manifest's
> `client.entry` / `client.lua` is a reserved field that is **not executed** today. The
> architecture is: **front-end = HTML/JS** (your pages) · **logic + data = `server.lua`** ·
> the client DLL's **bridge** connects them (`MUPF.invoke` ↔ `OnInvoke` / `ctx:reply`). You
> never write client-side Lua.

---

## ⚠️ Important model — there is **no live DOM**

litehtml renders HTML but does **not** give JavaScript a mutable document. So:

- `document.getElementById`, `addEventListener`, `el.onclick`, `el.innerHTML = …` → **do not use.**
- `setTimeout` / `setInterval` / `fetch` / `XMLHttpRequest` → **not available.**

Instead, the page works by **re-rendering**: your JS builds a full HTML string and calls
`MUPF.render(html)`. The host re-parses it and repaints. You keep state in **JS globals**,
which survive across `MUPF.render()` (same JS context).

```js
// pattern: entry page defines state + functions + does the first request.
var rows = [], page = 0;                 // <-- persists across MUPF.render

function view() {                        // build the page as a STRING
  var h = '<style>' + CSS + '</style><div id="bd">';
  for (var i = 0; i < rows.length; i++) h += '<div>' + esc(rows[i].name) + '</div>';
  return h + '</div>';
}
function render() { MUPF.render(view()); }   // re-parse + repaint

MUPF.invoke('getData', {}, function(resp){   // first request, on load
  rows = resp.rows || [];
  render();
});
```

> ⚠️ The page you pass to `MUPF.render()` must contain **no `<script>`** — only the *entry*
> `index.html` carries the script. Re-rendered views are pure markup; the JS context (your
> globals + functions) stays alive between renders.

---

## The `MUPF` object (JavaScript)

### `MUPF.invoke(fn, args, cb)` → reqId
Call a server function (`OnInvoke(ctx, fn, args, reqId)` on the server). `args` is any JSON-able
value. `cb(resp)` runs when the reply arrives — `resp` is the parsed reply object, or
`{ error: "…" }` on failure. *(Requires the `net.request` capability.)*

```js
MUPF.invoke('getRanks', { page: 0 }, function(resp){
  if (resp.error) { MUPF.render(errorView(resp.error)); return; }
  rows = resp.rows;
  render();
});
```

### `MUPF.render(html)`
Replace the whole page with `html` (a complete fragment/string). Applied off the JS call
stack (next frame), so it is safe to call from inside a callback.

### `MUPF.close()`
Close the (main) plugin window.

### `MUPF.open([id])` · tier-1 launcher
Open a plugin's **main** window. With no argument, `MUPF.open()` opens **this** plugin's main
window — this is what a **tier-1 launcher** page calls when its button is clicked. With an id,
`MUPF.open("com.x.y")` opens that plugin's main window. *(Requires `ui.window`.)*

```js
// in client/launcher.html — the whole always-on button opens the main window:
function __mupf_click(x, y) { MUPF.open(); }
```

---

## Receiving input — `__mupf_click(x, y)`

Because there is no DOM, **clicks are delivered as coordinates**. Define a global function and
hit-test the regions yourself (client pixels; `0,0` = window top-left):

```js
function __mupf_click(x, y) {
  if (y >= H - 50) {                 // a bottom "pager" strip
    if (x < W / 2) prevPage(); else nextPage();
  }
}
```

- `W` / `H` are your window size (from the manifest `client.window`).
- The host reserves the **top ~38px** as a window **drag strip**, and a **top-right ~38×38**
  area as a built-in **close** hotspot. Draw your title bar there; clicks elsewhere arrive in
  `__mupf_click`.
- There is no hover / mouse-move / key event to JS — design around clicks (and the manifest
  hotkey, which toggles the window).

---

## CSS support (litehtml + GDI)

**Supported:** `background-color` (solid), `border` / `border-*` (solid), `linear-gradient`
backgrounds, text (color, size, weight, `letter-spacing`, `text-transform`, `text-shadow`),
`flex`, tables, `nth-child`, `width/height/padding/margin`, `white-space:nowrap`,
`vertical-align`, `position:absolute/relative`, `overflow:hidden`.

**Not (yet) rendered — avoid relying on these:**

| Feature | Status / workaround |
|---|---|
| `border-radius` (rounded corners) | not drawn — use square panels, or bake rounding into an SVG |
| `box-shadow` | not drawn — fake depth with borders/gradients |
| `radial-gradient` / `conic-gradient` | not drawn — use `linear-gradient` |
| transitions / animations / `:hover` | no — there is no mouse-move to JS |
| `%` heights relative to the viewport | unreliable — use fixed px (the window size is fixed) |

> 💡 Fixed window size means you can lay out against exact pixels (e.g. a 554px-tall content
> area). Hardcode it; the window does not resize.

---

## Fonts & glyphs

Text is drawn with `TextOutW` and **has no font fallback**: a character the chosen font lacks
renders as an empty box (tofu).

> ⚠️ This bites with symbol glyphs. Triangle arrows `◀ ▶` (U+25C0/25B6), for example, are
> **not** in Segoe UI/Tahoma → they show as boxes. Use glyphs the font has, or — better —
> draw the icon as an **SVG** (see below). `✕` (U+2715) and `★` do render.

---

## Images & SVG

`<img>` and CSS `background-image` render via an SVG rasterizer (**nanosvg**):

```html
<img class="crown" src="crown.svg">
<style> .badge { background-image: url(badge.svg); background-repeat: no-repeat;
                 background-size: 26px 26px; } </style>
```

- Reference assets **relative** — `src="crown.svg"` resolves to your `client/crown.svg`.
- Ship the `.svg` files in `client/`; they are packed with the plugin automatically.
- nanosvg handles paths, `rect`/`circle`/`ellipse`/`line`/`polygon`, flat fills, and
  `linearGradient`. It does **not** render SVG `<text>` — put text in HTML on top of the SVG.
- **PNG/JPG are not supported yet** — SVG only.

> 💡 SVG is the reliable way to get crisp icons, gradient badges, arrows, and frames — it
> sidesteps both the font-glyph and the rounded-corner limitations.

---

## Windows: main + tier-1 launcher

**Main window** (`client.window`):
- **Size/title** from the manifest (`width`, `height`, `title`).
- A borderless popup that stays over the game scene, follows the game window when you drag it,
  and **hides when you alt-tab away** (returns when the game regains focus).
- **Open:** the manifest `entryPoints.hotkey` (e.g. `F5`), a tier-1 launcher (`MUPF.open()`), or
  the server. **Close:** the host's top-right close hotspot or `MUPF.close()`. Several can be open.

**Tier-1 launcher** (`client.launcher`, optional):
- A small **always-on, fixed, non-movable, non-closable** button window, positioned by `anchor`
  + `x`/`y` relative to the game window (see
  [MUPF Plugins → 2-tier plugins](MUPF-Plugins.md#2-tier-plugins--an-always-on-launcher-button)).
- Renders its own `launcher.html`; the same `__mupf_click(x, y)` model applies. Its click calls
  `MUPF.open()` to open the main window.

---

## How a click round-trips to the server and back

```
[click] -> __mupf_click(x,y) -> MUPF.invoke(fn, args, cb)
   -> (server) OnInvoke(ctx, fn, args, reqId) -> ctx:sql -> ctx:reply(reqId, data)
   -> (client) cb(data) -> MUPF.render(view(data))     // page repaints with the data
```

---

## Minimal example

```html
<!DOCTYPE html><html><head><meta charset="utf-8"></head>
<body>
  <div id="boot">Loading…</div>
  <script>
    var W = 400, H = 300, rows = [];
    var CSS = 'body{margin:0;font:13px "Segoe UI";color:#e8e4d8;background:#15161a}'
            + '#bd{padding:10px} .n{color:#7fb6ff}';
    function esc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;'); }
    function view(){
      var h = '<style>'+CSS+'</style><div id="bd">';
      for (var i=0;i<rows.length;i++) h += '<div class="n">'+esc(rows[i].name)+'</div>';
      return h + '</div>';
    }
    function __mupf_click(x, y){ if (y < 24) MUPF.close(); }   // click the top strip to close
    MUPF.invoke('getData', {}, function(resp){
      rows = (resp && resp.rows) || [];
      MUPF.render(view());
    });
  </script>
</body></html>
```

See **[MUPF Server API](MUPF-Server-API.md)** for the matching `OnInvoke` handler, and
**[MUPF Plugins](MUPF-Plugins.md)** for the manifest, capabilities, and packaging.
