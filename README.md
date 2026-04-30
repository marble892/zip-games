# Zipped Games

A single-file HTML launcher for collections of small browser games shipped as zip archives. Open `index.html`, drop a zip on the page, and the games inside show up as a tiled library you can browse, search, favorite, and play.

There's no build step, no install, no server. The whole thing(including UI, zip parser (JSZip), storage layer, command palette, gamepad support) lives in one HTML file. You can host it on a static server, run it from a USB stick, or just double-click it locally.

## Quick start

1. Open `index.html` in a modern browser (Chrome, Edge, Firefox, Safari all work).
2. Drag a `.zip` onto the window, or click **Pick a zip**.
3. Pick a game from the grid. It opens in a sandboxed iframe.
4. Press `Esc` or `Alt`+`M` while a game is running to bring up the in-game menu (resume, fullscreen, clear save, exit).

Packs are remembered in IndexedDB, so next time you open the launcher your library is already there. You can load multiple packs and switch between them from the dropdown in the header.

## How to bundle a zip

A pack is just a regular `.zip` file with a `data.json` manifest at the root and the game files anywhere you like inside it. Here's the minimum you need:

```
mypack.zip
├── data.json
├── games/
│   ├── snake.html
│   ├── pong.html
│   └── breakout.html
└── covers/
    ├── snake.png
    ├── pong.png
    └── breakout.png
```

### The manifest (`data.json`)

```json
{
  "fileBase": "games/",
  "coverBase": "covers/",
  "games": [
    {
      "id": "snake-v1",
      "fileName": "snake.html",
      "displayName": "Snake",
      "description": "The classic. Eat the apples, don't bite yourself.",
      "cover": "snake.png",
      "tags": ["arcade", "classic"]
    },
    {
      "id": "pong-v1",
      "fileName": "pong.html",
      "displayName": "Pong",
      "cover": "pong.png",
      "tags": ["arcade", "2-player"]
    }
  ]
}
```

### Reference

Top level:

| Field | Required | What it does |
| --- | --- | --- |
| `games` | yes | Array of game entries (see below). |
| `fileBase` | no | Prefix prepended to every `fileName`. Use it if your HTML files live in a subfolder. Trailing slash matters (`"games/"`, not `"games"`). |
| `coverBase` | no | Same idea, but for `cover` images. |

Per-game:

| Field | Required | What it does |
| --- | --- | --- |
| `fileName` | yes | Path inside the zip (after `fileBase`) to the game's HTML entry point. |
| `displayName` | no | Title shown on the tile. Falls back to `fileName` if missing. |
| `description` | no | Shown on the tile and in the details view. |
| `cover` | no | Path inside the zip (after `coverBase`) to a cover image. PNG, JPEG, WebP, GIF all work. If omitted, the tile gets a generated placeholder. |
| `tags` | no | Array of strings. Used for the tag filter chips. Anything goes — `"arcade"`, `"2-player"`, `"wip"`, etc. |
| `id` | no, but recommended | Stable identifier used for favorites, recents, and per-game saves. If you omit it, `fileName` is used, which means renaming the file will detach saves and favorites. Pick something and stick with it. |

### Game files

Each entry should be a self-contained HTML file. The launcher loads it inside a sandboxed iframe with these flags:

```
allow-scripts allow-same-origin allow-pointer-lock allow-popups allow-forms
```

That means:

- JavaScript runs.
- The game can `fetch()` its own assets from inside the zip (the launcher rewrites the iframe's blob URL so relative paths work).
- Pointer lock and fullscreen requests work.
- The game can't reach into the launcher or read other origins.

Keep assets relative. Don't hardcode `file://` paths or absolute URLs to your dev machine.

### Actually building the zip

No special tooling. Any zip program works:

- **Windows Explorer**: select the files → right-click → Send to → Compressed (zipped) folder.
- **macOS Finder**: select → right-click → Compress.
- **`zip` CLI** (Linux/macOS/WSL):
  ```
  cd mypack
  zip -r ../mypack.zip .
  ```
- **7-Zip**: add to archive, format `zip`. Compression level doesn't matter, store-only is fine and parses faster.

One thing to watch: don't zip the *parent* folder. The `data.json` needs to be at the root of the archive, not at `mypack/data.json` inside it. If you double-click your zip and see a single folder, you've got the wrong layout.

### Validating a pack

Use the command palette (`Ctrl`/`⌘`+`K` → "Run pack diagnostics") on a loaded pack. It walks every entry, checks that the file exists in the zip, flags missing covers, and reports tag counts. You can also load an obviously broken pack, the launcher will route the failure through the error log with a recovery action that opens diagnostics directly.

## Features

### Library
- Multiple packs at once. Switch via the pack dropdown; each pack remembers its own state.
- Search across name, description, and tags. Live, no submit needed.
- Tag chips you can toggle on and off; multiple tags AND together.
- Sorts: name, recently added, recently played, most played, favorites first.
- Favorites — click the star on any tile, or press `S` while it's focused.
- Recents — last few you played, shown at the top when sorted that way.
- Cover images lazy-load via `IntersectionObserver` so packs with hundreds of games don't choke the initial render.

### Playing
- Sandboxed iframe with pointer lock and fullscreen.
- In-game menu (`Esc` or `Alt`+`M`): resume, fullscreen (`F`), clear save, exit.
- Crash recovery: if the game throws or rejects a promise, the menu pops up with the error message instead of leaving you with a frozen iframe.
- Per-game save storage — see "Saves bridge" below.

### Input
- Keyboard navigation throughout the grid.
- Gamepad support: D-pad or left stick to move, A to open, B to back out, Start for the in-game menu.
- Command palette (`Ctrl`/`⌘`+`K`) for fuzzy-finding actions and games.

### Storage and privacy
- All data is local. No telemetry, no network calls, no analytics.
- Packs live in IndexedDB (`zipGamesDB`). Preferences live in `localStorage` under the `zg.` prefix.
- Settings export/import: dumps everything except the packs themselves to a JSON file you can carry between browsers.

### Errors
- Centralized error log accessible from the header (or `E`, or the palette).
- Each error type knows what recovery actions make sense (retry, pick another file, remove pack, run diagnostics, clear settings, reload, copy details).
- Errors badge the toolbar button so you notice them without a popup taking over.
- Toast notifications for transient stuff; the modal log keeps the last 100 entries across sessions.

### Themes
- Light and dark. Toggle with `T` or the sun/moon button. Choice persists.

## Keyboard shortcuts

| Key | Does |
| --- | --- |
| `/` | Focus search |
| `↑` `↓` `←` `→` | Move between tiles |
| `Enter` | Open the focused game |
| `Ctrl`+`Click` | Open in a new tab |
| `Esc` / `Alt`+`M` | Toggle in-game menu |
| `F` | Fullscreen (in-game) |
| `T` | Toggle theme |
| `?` | Show keyboard help |
| `Ctrl`/`⌘`+`K` | Command palette |
| `S` | Favorite the focused tile |
| `E` | Open the error log |

## Gamepad

| Input | Does |
| --- | --- |
| D-pad / left stick | Move between tiles |
| A | Open focused game |
| B | Clear search / back |
| Start | Open in-game menu (or palette in library) |

Repeat-rate is debounced so holding a direction scrolls at a sane speed.

## Saves bridge

Games running inside the launcher can persist data without using `localStorage` (which is shared across the whole document and would collide). The launcher exposes a `postMessage` API instead.

Send messages to `window.parent` with `id: "zg-save"`:

```js
// From inside a game's HTML:
function call(op, payload) {
  return new Promise((resolve) => {
    const reqId = Math.random().toString(36).slice(2);
    function onReply(ev) {
      const d = ev.data;
      if (!d || d.id !== "zg-save-reply" || d.reqId !== reqId) return;
      window.removeEventListener("message", onReply);
      resolve(d);
    }
    window.addEventListener("message", onReply);
    window.parent.postMessage({ id: "zg-save", reqId, op, ...payload }, "*");
  });
}

// Examples:
await call("set", { key: "highscore", value: 4200 });
const { value } = await call("get", { key: "highscore" });
await call("del", { key: "highscore" });
await call("clear", {});
```

Operations:

| `op` | Payload | Reply |
| --- | --- | --- |
| `get` | `{ key }` | `{ value }` (or `null`) |
| `set` | `{ key, value }` | `{ ok: true }` |
| `del` | `{ key }` | `{ ok: true }` |
| `clear` | — | `{ ok: true }` |

`value` can be anything that survives `JSON.stringify`. Saves are namespaced by the game's `id` (or `fileName` if no id), so two games can use the same key without colliding. The user can wipe a save from the in-game menu or the error log's "clear settings" recovery.

## Settings export/import

Help modal → "Export settings" downloads a JSON file containing your theme, sort, recents, favorites, active tags, and per-game saves. "Import settings" merges an exported file back in. Pack contents (the actual zip data) are not included, re-add the packs and your saves and favorites snap back into place because they're keyed by game id.

## Optional service worker

If you drop a `sw.js` next to `index.html`, the launcher will register it on load. There's no service worker shipped, it's just a hook for people who want to make the page installable as a PWA or pre-cache assets. If the file isn't there, registration silently does nothing.

## Troubleshooting

**"Couldn't read that file. Is it a valid .zip?"**
The file isn't a zip, or it's corrupted. Try unzipping and rezipping it. Encrypted zips aren't supported.

**"Pack is missing data.json"**
`data.json` has to be at the root of the archive. If you accidentally zipped a folder, the manifest is one level too deep.

**"data.json is malformed"**
Run it through any JSON validator. Trailing commas and comments are the usual culprits.

**"Missing game file: foo.html"**
Either the path is wrong (check `fileBase` + `fileName` against the actual zip contents) or the file got dropped during zipping. Run pack diagnostics from the palette to see exactly which entries can't be resolved.

**Pop-up blocked**
Browsers block `Ctrl`+`Click` "open in new tab" if the user hasn't interacted recently. Click anywhere on the page first, then try again.

**Fullscreen denied**
Some browsers require a direct user gesture for fullscreen and reject it from the in-game menu. Press `F` on the tile or inside the game directly.

**The library disappeared**
Clearing browser storage wipes IndexedDB. Re-add the pack. Saves are preserved separately as long as you didn't clear `localStorage` too.

**Browser says IndexedDB is unavailable**
Private/incognito modes sometimes restrict it. The launcher will still run but packs won't persist, you'll need to re-load them each session.

## Layout of the source

Everything is in `index.html`. Roughly, top to bottom:

1. JSZip (vendored, minified).
2. Stylesheet.
3. Markup for the library, modals (help, diagnostics, error log, in-game menu), and toast container.
4. App script: state, IndexedDB layer, pack loading, rendering, search/sort, keyboard, gamepad, command palette, error system, saves bridge, settings export/import, drag-and-drop wiring, init.

## License

The launcher itself is yours to use however you want. JSZip is MIT/GPLv3 (see the header at the top of the file). Games you bundle keep whatever license they shipped with, the launcher doesn't relicense anything.
