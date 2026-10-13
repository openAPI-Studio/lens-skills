---
name: lens-capture-agent
description: Capture screenshots, GIFs and step-by-step guides of web pages with the Lens Chrome extension while driving Chrome as an agent (Claude in Chrome or similar). Use when asked to record a GIF, take an annotated screenshot, document a browser flow as a guide, export or replay a Lens guide, or check that a guide still matches the page. Masks secrets before capture and files everything into a Lens project.
---

# Capturing with Lens from a browser agent

Lens (Lens by Vectored) captures regions, records GIFs and turns click-throughs
into step-by-step guides, filed into local projects with a timeline. An agent
can't press Lens's shortcuts or open its popup, so it drives Lens through a
page-script bridge. Every command is a `window.postMessage` from a script run
on the page, for example with Claude in Chrome's `javascript_tool`.

If the user just wants a quick GIF and doesn't care about Lens, the agent's own
recorder (Claude in Chrome's `gif_creator`) is simpler.

## Requirements

- Lens with **Options > Allow browser agents to drive Lens** turned on. Never
  turn it on yourself. If `status` returns `disabled` or `no-bridge`, reload the
  tab once. If it still fails, ask the user to turn the option on.
- The tab you work in must be the active, visible tab, because captures
  photograph what is on screen.

## The helper

Start each script with this helper, then call one or more commands:

```js
const lens = (command, args = {}) => new Promise((resolve) => {
  const id = Math.random().toString(36).slice(2);
  const timer = setTimeout(() => { removeEventListener('message', on); resolve({ ok: false, error: 'no-bridge' }); }, 15000);
  function on(e) {
    const d = e.data;
    if (e.source !== window || !d || d.lens !== 'automation' || d.type !== 'response' || d.id !== id) return;
    clearTimeout(timer); removeEventListener('message', on); resolve(d);
  }
  addEventListener('message', on);
  postMessage({ lens: 'automation', type: 'request', id, command, ...args }, '*');
});
await lens('status');
```

Every answer has `ok`, and `error` when `ok` is false. The full command list is
in [references/commands.md](references/commands.md).

## Every capture: project, then secrets

1. **Project.** Run `lens('list-projects')`.
   - If the user named a project, select it with `set-active-project`.
   - Otherwise create one with `create-project` `{ name }`, which also makes it active.
   - A project created this way has no linked folder, because only the user can
     pick one. Tell them once.
2. **Secrets.** Run `lens('scan-secrets')`. It reports each finding's label and
   position, never its value. Don't try to read the values yourself.
   - By default, mask every `secret`: `lens('mask', { categories: ['secret'] })`.
   - Turn on auto-mask, so secrets that appear mid-flow are covered too:
     `lens('auto-mask', { on: true })`.
   - Mask `personal` findings (emails, card numbers) too when the result will be
     shared outside the user's team. When unsure, ask once and list the labels.
   - If the user says what to show or hide, do that.
   - If `mask` returns `missed > 0`, the page changed under the scan. Scan and
     mask again.
3. **Cleanup.** When you're finished, run `lens('unmask', { all: true })`.

## Screenshot

1. Measure the element with `getBoundingClientRect()` and add about 8 px of
   padding. Then run `lens('capture-image', { region: { x, y, width, height } })`.
   Leave out `region` to capture the whole viewport.
2. Optionally annotate it. Coordinates are viewport CSS pixels, the same space
   you measured in:
   `lens('annotate', { marks: [{ kind: 'rect', x, y, width, height }, { kind: 'arrow', from: {x, y}, to: {x, y} }, { kind: 'text', x, y, text: 'Click here' }] })`.
   - Use `kind: 'blur'` for anything secret that masking missed. A blur
     rewrites the pixels when the image is saved.
3. Name it: `lens('fill-saver', { name, notes })`. Then save it:
   `lens('save-open')`.

## GIF

1. Start recording with `lens('record-gif', { region })`. Leave out `region` to
   record the whole viewport.
2. Do the interaction with normal clicks and typing. Pause about a second
   between steps. These recordings run at about 4 fps, so slow, deliberate
   steps read best. Don't navigate to another page mid-recording.
3. Run `lens('stop')`. Then `fill-saver` and `save-open`. Saving encodes the GIF,
   which takes a few seconds.

## Step-by-step guide

1. Run `lens('record-steps')`. Each click becomes a numbered step with a
   screenshot. Click and type as the user would, one deliberate action at a
   time.
2. Run `lens('stop')`. Then name the guide with
   `lens('fill-saver', { name: '<guide title>' })` and save it with
   `lens('save-open')`.
3. Export it with `lens('export-guide', { format: 'markdown' })`.
   - Use `'html'` for a standalone page, or `'bundle'` for a zip someone else
     can import into Lens.
   - The file is written into the guide's folder in the project. Markdown also
     comes back as `markdown`, so you can paste it into docs.
4. Use `lens('check-guide')` to confirm each step's control is still on the
   page. Steps come back as `found`, `missing` or `other-page`. Report any
   `missing` step as a guide that needs updating.
5. Use `lens('play-guide')` to run Guide Mode for the user, and
   `lens('guide-status')` to read which step it's on.

## Afterwards

- Run `lens('last-capture')` to get the filename, project and whether it was
  written to the linked folder. When `writtenToFolder` is true, the file is at
  `<linked folder>/<filename>`, where an agent with file access can read it.
- To put the file into an upload field on the same site, call
  `lens('last-capture', { includeData: true })` inside the script that uses it,
  and turn `dataUrl` into a `File` there. Never return the data URL as your
  tool result.
- To rename or annotate after saving, use `rename-capture` `{ name }` and
  `set-notes` `{ notes }`.
- Tell the user the filename, the project, and what was masked, by label.

## Errors

| Error | Meaning |
| --- | --- |
| `disabled`, `no-bridge` | The option is off, or the tab loaded before it was turned on. |
| `tab-not-active` | Bring the tab to the front. |
| `already-recording` | Check `status`. Stop only what you started. |
| `restricted-page` | Chrome blocks capture on browser pages and the Web Store. |
| `no-saver` | No save dialog is open yet. Wait a moment and retry. |
| `nothing-saved`, `no-guide` | Nothing from this site to act on. Lens only shows a page what was captured from its own site. |
| `no-active-project` | Pick or create a project first. |
