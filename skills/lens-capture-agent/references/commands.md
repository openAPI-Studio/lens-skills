# Driving Lens from a browser agent

AI browser agents such as Claude in Chrome act through the page: they click,
type, drag and run scripts in it. They cannot press an extension's keyboard
shortcut or open its popup, so out of the box they cannot start a Lens
recording. The browser agent bridge gives them a way in.

## Turning it on

Options → **Allow browser agents to drive Lens**. Off by default.

While it is on, Lens registers a small content script
(`content/automation-bridge.js`) on every page. Any script on a page can then
ask Lens to record that page, so leave it off unless you use an agent. Turning
it off unregisters the script, and the worker re-checks the setting on every
request, so a tab that still has the script loaded is refused.

## Protocol

A page script posts a request and listens for the answer with the same `id`:

```js
// Request
window.postMessage({ lens: 'automation', type: 'request', id, command, ...args }, '*');
// Response
{ lens: 'automation', type: 'response', id, ok: true | false, error?, ...result }
```

A helper an agent can paste into its script tool:

```js
const lens = (command, args = {}) => new Promise((resolve) => {
  const id = Math.random().toString(36).slice(2);
  const timer = setTimeout(() => {
    window.removeEventListener('message', onMessage);
    resolve({ ok: false, error: 'no-bridge' });
  }, 4000);
  function onMessage(event) {
    const d = event.data;
    if (event.source !== window || !d || d.lens !== 'automation' || d.type !== 'response' || d.id !== id) return;
    clearTimeout(timer);
    window.removeEventListener('message', onMessage);
    resolve(d);
  }
  window.addEventListener('message', onMessage);
  window.postMessage({ lens: 'automation', type: 'request', id, command, ...args }, '*');
});
```

| Command | Args | What it does |
| --- | --- | --- |
| `status` | | `{ ok, version, recording }`. `recording.active` says whether a recording is running. |
| `record-gif` | `region`: `{ x, y, width, height }` in viewport CSS pixels, or omitted for the whole viewport | Starts recording straight away, no selector. Same recorder widget, limits and settings as the hotkey. |
| `open-selector` | `mode`: `'gif'` or `'image'` | Opens the region selector, as the hotkey does. The agent drags a region and presses the toolbar button. |
| `stop` | `save`: `false` to discard | Stops the recording. With save on, the GIF converter opens on the page; press **Convert & Save GIF** (`#screensnap-modal-gif-save`) to write it to the active project. |

Errors: `disabled` (setting off), `tab-not-active` (the tab must be the one on
screen, because captures photograph the visible tab), `already-recording`,
`no-session`, `restricted-page`, `unknown-command`, `no-bridge` (from the helper:
nothing answered, so the setting is off or the page loaded before it was
turned on; reload the page).

## Still captures, notes and annotations

| Command | Args | What it does |
| --- | --- | --- |
| `capture-image` | `region` (viewport CSS pixels), or omitted for the viewport | Takes the shot straight away and opens the screenshot saver. |
| `capture-full-page` | | Scrolls the page top to bottom, stitches one tall screenshot and opens it in the saver. Returns as soon as the run starts. Don't scroll, click or type until it ends: any input stops the run and keeps what was captured so far. In the saver, `annotate` coordinates are page CSS pixels from the top of the page (`rect.top + scrollY`), not viewport pixels. |
| `full-page-status` | | `running`, and `last`: how the last run on this page ended. `last.at` is a `Date.now()` timestamp; `status` is `done`, `interrupted`, `limit`, `timeout`, `jumped`, `failed-midway` or `failed`, with `reason`. Also `screens`, `width`, `height`, `partial`, `unloadedImages` (images still blank when shot), `scaledToFit`, `grew` (the page loaded more as it scrolled), `panel` (an inner panel was scrolled, not the page), `maxCaptures` and `flagged` (regions the saver flagged as private). |
| `annotate` | `marks`: a list of `{ kind: 'rect' \| 'circle' \| 'blur', x, y, width, height }`, `{ kind: 'arrow', from: { x, y }, to: { x, y } }` or `{ kind: 'text', x, y, text, fontSize? }`; each may carry `color` and `size` | Draws on the capture in the open screenshot saver. Coordinates are viewport CSS pixels, the same space the page was measured in, and are mapped onto the image. One Undo takes them all back. `blur` rewrites the pixels when saved. |
| `fill-saver` | `name`, `notes` | Fills the open saver's fields: screenshot, GIF, video or guide (a guide takes `name` as its title). |
| `save-open` | | Presses the open saver's save button. |
| `rename-capture` | `name`, `recordId` (default: the last capture) | Renames the file in every store and its timeline entry, keeping the extension. |
| `set-notes` | `notes`, `recordId` (default: the last capture) | Sets the capture's notes in the timeline. |

`rename-capture` and `set-notes` act only on captures taken of the requesting
page's site.

## Guides

| Command | Args | What it does |
| --- | --- | --- |
| `record-steps` | | Starts a walkthrough: one screenshot per click, written up as numbered steps. Skips the briefing dialog. `stop` ends it and opens the guide saver; `fill-saver` sets its title and `save-open` saves it. |
| `list-guides` | `projectId` (default: active) | The project's guides recorded on this site: `{ guides: [{ id, title, stepCount, url, timestamp, path }] }`. |
| `export-guide` | `guideId` (default: the last guide saved), `format`: `markdown` (default), `html` or `bundle`, `includeData` | Writes `guide.md`, `guide.html` or a `.zip` bundle into the guide's folder in the project. Markdown also comes back as `markdown`. `includeData` adds the file as `dataUrl`. |
| `check-guide` | `guideId` | Looks for each step's control on the current page, the same way Guide Mode does: `{ steps: [{ n, kind, text, status: 'found' \| 'missing' \| 'other-page' \| 'no-target' }] }`. |
| `play-guide` | `guideId`, `stepIndex`, `autoAdvance` | Starts Guide Mode on this tab. |
| `guide-status` | | Where Guide Mode is: `{ playing, stepIndex, stepCount, stepText, target: 'found' \| 'missing' \| 'searching' \| … }`. |

A page can only reach guides recorded on its own site.

## Projects and the saved file

| Command | Args | What it does |
| --- | --- | --- |
| `list-projects` | | `{ projects: [{ id, name, active, folderLinked }] }` |
| `create-project` | `name`, `activate` (default `true`) | Creates a project and makes it active. It has no folder: linking one needs the folder picker, which only a person can answer, so the project keeps its captures inside Lens until the user links one. |
| `set-active-project` | `projectId` | Makes that project the one saves go to. Open savers redraw to match. |
| `last-capture` | `includeData` (default `false`) | The newest capture saved in this browser session: `{ capture: { filename, projectId, projectName, folderName, writtenToFolder, title, url, timestamp, recordId } }`. With `includeData`, also `dataUrl`, the file itself, up to 24 MB. |

`last-capture` answers only a page on the same site the capture was taken
from, so one tab cannot read a recording made of another.

How an agent uses the file: when the project has a linked folder and
`writtenToFolder` is true, the file is at `<that folder>/<filename>`, where an
agent with file access (Claude Code) can read it. Otherwise `includeData`
hands the page script the bytes, which it can, for example, turn into a
`File` for an upload field without the data passing through the agent.

## Secrets on screen

The bridge uses the same rules as the capture editor's blur hints
(`content/redaction.js`): password and payment fields, fields named like
credentials, and text shaped like API keys, tokens, private keys, emails and
card numbers. The agent is told **what** was found and **where**, never the
value, so the secret never reaches the agent's context.

| Command | Args | What it does |
| --- | --- | --- |
| `scan-secrets` | `includePersonal` (default `true`), `region` | `{ findings: [{ id, label, category: 'secret' \| 'personal', confidence, kind: 'field' \| 'text', rects, masked }] }`. Ids are valid until the next scan. |
| `mask` | one of `ids: ['s1', …]`, `categories: ['secret']`, `all: true` | Covers those findings with a solid grey bar drawn in the page, so it is in every captured frame. `{ masked, missed }`; a miss means the page changed, so scan again. |
| `unmask` | `ids`, `categories`, or `all: true` | Removes masks. `all` also turns auto-mask off. |
| `auto-mask` | `on` (default `true`), `categories` (default `['secret']`) | Keeps masking whatever appears, such as a token revealed by a click. It rescans as the page changes, before the change is painted, at most every 120 ms; on a page that changes faster than that, a new secret can be visible for up to that long. |

Masks are page state: they last until `unmask`, or until the page reloads or
navigates.

## Frame rate

The hotkey counts as invoking the extension on the tab, which lets Lens use
`tabCapture` at 15 fps. A request from a page script does not, so Chrome may
refuse `tabCapture` and Lens falls back to sampling the visible tab at about
4 fps. Recordings still work; motion is less smooth. Pressing the Lens toolbar
icon on the tab once beforehand grants the tab and restores 15 fps.
