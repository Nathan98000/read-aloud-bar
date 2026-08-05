# Read Aloud Bar

A floating text-to-speech controller for any web page, delivered as a bookmarklet. It extracts the article text, splits it into sentences, and reads them aloud with sentence-level navigation, optional highlighting, and optional auto-scroll.

No build step, no dependencies, no network calls. The whole thing is one HTML file that doubles as the install page and the source.

## Install

1. Open `Read_Aloud_Bar.html` in a browser.
2. Drag the **Read Aloud Bar** button onto your bookmarks bar.
3. If dragging doesn't take, click **Copy code**, create a new bookmark by hand, and paste the copied `javascript:` URL into its URL field.

To upgrade an existing install, overwrite that bookmark's URL with a freshly copied one. The install page regenerates the bookmarklet URL from the live source every time it loads, so editing the file is enough — but the URL already sitting in your bookmark is a snapshot and will not update on its own.

The page also has a **Play this paragraph** demo that runs the real controller against a sample paragraph, so you can try it before installing.

## Controls

The bar appears bottom-right and can be dragged anywhere by its background.

| Control | Behavior |
| --- | --- |
| `◀◀` / `▶▶` | Move exactly one sentence. While playing, narration resumes from the new sentence; while paused, the position moves and playback stays paused. |
| `▶` / `❚❚` | Play / pause. |
| `■` | Stop and reset to the first sentence. |
| `Highlight` | Toggle the tint on the sentence being spoken. |
| `Scroll` | Toggle auto-scroll that keeps the current sentence in view. |
| Voice dropdown | English voices from the platform's speech engine. |
| Speed slider | 0.5×–4.0× in 0.1 steps. |
| Readout | `3.0× · 525 wpm` over `14 / 87 sentences`. |
| `✕` | Tear everything down — cancel speech, remove the bar, overlays, styles, and listeners. |

`Highlight` and `Scroll` are independent; either can be on without the other.

Re-running the bookmarklet on a page that already has a bar cancels the old instance and replaces it.

## What it reads

Priority order:

1. **A selection**, if you made one. Only text intersecting the selected range is read.
2. **Substack posts**, detected by hostname, `.available-content`, or Substack CDN assets. Pulls the title, subtitle, and post body; excludes the byline, date, like/comment/restack counts, and share controls.
3. **A generic article root** — `article`, `[itemprop=articleBody]`, `[class*='article-body']`, `main`, `[role=main]`, `#content`, and similar, plus the page `h1` when it sits outside that root.
4. **The densest block**, if none of the above yields enough text. Scores every candidate container by text length minus link text, weighted by paragraph count.
5. **`document.body`**, as a last resort.

Throughout, elements are skipped when they match junk tags (`nav`, `header`, `footer`, `aside`, `form`, …), junk ARIA roles, junk class/id words (`sidebar`, `cookie`, `newsletter`, `comment`, `related`, `share`, `byline`, …), `aria-hidden="true"`, the `hidden` attribute, or computed invisibility.

## How it works

**Extraction with DOM mapping.** A `TreeWalker` collects text nodes, groups them by block-level ancestor, and concatenates each block. Every resulting sentence keeps start and end `(node, offset)` pairs, so the highlighter can rebuild a `Range` later without ever editing the page's markup.

**Sentence segmentation.** Splits on `.`, `!`, `?`, and `…`, absorbing trailing quotes and brackets. An abbreviation list (`Mr.`, `Dr.`, `e.g.`, `U.S.`, month names, …) and a single-initial rule suppress false boundaries. Punctuation reaches the speech engine untouched, so sentences keep their normal falling intonation and end-of-sentence pause.

**Chunked scheduling.** Utterances are built on demand from a starting sentence, packing sentences up to 520 characters, with 4 chunks queued ahead. Because each chunk is constructed at the moment it's needed, skipping never inherits a boundary decided earlier. `onboundary` events map `charIndex` back to the exact sentence.

**Watchdog.** A 250 ms interval covers engines that emit no boundary events, estimating position from elapsed time at roughly `15.5 × rate` characters per second. If the engine goes silent for more than 2.5 seconds mid-playback, it restarts from the current sentence. It never interrupts healthy playback.

**Highlighting.** Uses the CSS Custom Highlight API (`::highlight(ttsbar)`) where available, and absolutely-positioned overlay rectangles derived from `Range.getClientRects()` otherwise. Neither path modifies the document. The overlay path re-renders on scroll and resize. The tint persists while paused and clears on stop.

**Auto-scroll.** Only intervenes when the current sentence drifts within 70 px of the top or 90 px of the bottom of the viewport, then smooth-scrolls it to about 35% of viewport height.

## Settings and persistence

Voice, speed, highlight, and scroll persist in `localStorage` under the key `readAloudBar.v3`:

```json
{ "voice": "Ava (Online)", "rate": 3, "highlight": true, "autoscroll": true }
```

Defaults on a fresh install are rate `3`, highlight on, scroll on. Voice selection falls back through Ava Online → Ava Natural → any Ava → Samantha → any voice named "online" or "natural" → the first English voice, but a saved choice always wins.

## Development

Everything lives in `Read_Aloud_Bar.html`:

- The `ttsBar(demoRoot)` function is the entire bookmarklet. It takes an optional root element, which the demo uses to scope playback to the sample paragraph.
- The IIFE at the bottom is install-page wiring only — it serializes `ttsBar` into the `javascript:` URL, fills the drag button, the readable source block, and the copy buffer, and hooks up the demo buttons. It is not part of the bookmarklet.
- The page's own CSS and markup are the install page; none of it ships with the bookmarklet.

To syntax-check after editing:

```bash
sed -n '/^<script>/,/^<\/script>/p' Read_Aloud_Bar.html | sed '1d;$d' | node --check /dev/stdin
```

Elements injected into a host page, all removed on close: `#__ttsbar` (the bar), `#__ttsbarhl` (overlay container), `#__ttsbarstyle` (highlight style).

## Browser support and limitations

- Requires the Web Speech API's `speechSynthesis`. Available voices, and whether they emit word-boundary events, vary by platform — the watchdog exists because of that.
- The CSS Custom Highlight API needs Chrome/Edge 105+ or Safari 17.2+; everything else falls back to overlay rectangles.
- Sites with a strict Content Security Policy may block `javascript:` bookmarklets entirely. There is no workaround from inside the bookmarklet.
- Extraction is heuristic. On unusual layouts it may pick the wrong container — selecting a paragraph first bypasses extraction entirely.
