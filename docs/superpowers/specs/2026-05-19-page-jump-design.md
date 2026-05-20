# Direct Page Jump in PDF Pager

## Goal

Let the user jump to an arbitrary page in a loaded multi-page PDF by typing the page number directly, instead of clicking the prev/next buttons repeatedly.

## Current behavior

The top-bar pager renders as `[←] [<span id="pcur">1</span> / <span id="ptot">N</span>] [→]`. Navigation is limited to:

- Prev/next buttons (`#pprev`, `#pnext`)
- Arrow keys (`ArrowLeft`, `ArrowRight`) handled globally on `window`

## Proposed behavior

Replace the static `<span id="pcur">` with an `<input id="pcur">` styled to be visually indistinguishable from the surrounding `1 / N` text. The input is always present (no DOM swap).

Interaction:

- **Click** the page number: input gains focus, contents selected for overtype.
- **Enter**: parse as integer, clamp to `[1, pdfDoc.numPages]`, call `renderPage(n)` if `n !== curPage`.
- **Escape**: blur and revert displayed value to `curPage`.
- **Blur** (click elsewhere): revert to `curPage` without navigating. Required because typing a number without pressing Enter must not silently reroute the user.
- **Invalid input** (non-numeric, empty): revert silently. The constraint `/ N` is visible adjacent.

## Implementation

In `index.html`:

1. **Markup** — Change `<span id="pcur">1</span>` to:
   ```html
   <input id="pcur" type="text" inputmode="numeric" value="1">
   ```

2. **Styling** — Add a rule scoped to `#pcur` matching the existing pager text style: transparent background, no border, inherits font-weight/color/tabular-nums from the parent. Width derived dynamically in JS from `String(pdfDoc.numPages).length` using `ch` units so the pager doesn't reflow as page count changes. Subtle hover background (`#1d1d22`, matching `.btn:hover`) to hint interactivity. Focus state uses the accent color outline matching the modal input (`border-color:var(--accent)` equivalent — implemented via `outline` since input has no border).

3. **Render hook** — In `renderPage(n)`, replace `$('pcur').textContent = n` with `$('pcur').value = n`. Also blur the input if focused, so a successful jump returns the user to the canvas (allows arrow-key navigation to resume).

4. **Input handler** — Single keydown listener on `#pcur`:
   - `Enter`: parse `parseInt(value, 10)`. If finite and in `[1, pdfDoc.numPages]` and `!== curPage`, call `renderPage(n)`. Otherwise reset `value = curPage` and blur.
   - `Escape`: reset and blur.
   - All other keys: pass through (allows typing).

5. **Focus handler** — On focus, `select()` the input contents so the user can immediately overtype.

6. **Width update** — When a new PDF loads (`$('f').onchange`), set `$('pcur').style.width = (String(pdfDoc.numPages).length + 1) + 'ch'`.

## Edge cases

- **No PDF loaded** — Pager hidden via existing `$('pager').style.display = 'none'`. Input is inside the pager so it's also hidden.
- **Single-page PDF** — Pager hidden entirely (existing behavior, preserved).
- **Typed value === current page** — No-op, just blur. Avoids needless re-render.
- **Out-of-range value** — Revert silently (no toast). Constraint is visible.
- **Non-numeric input** — Browser allows it (we use `type="text"` not `type="number"` to avoid the spinner UI). Validation happens on Enter; invalid parses revert.
- **Global arrow-key handler** — Already guards with `if(e.target.tagName==='INPUT') return;`. This continues to work: while focused on `#pcur`, left/right arrows move the cursor within the input rather than triggering page nav. Confirmed by reading existing line in `index.html`.

## Out of scope

- Page thumbnails / outline panel
- Keyboard shortcut to focus the input (e.g. `Cmd+G`) — can add later if requested
- Page label support (PDF logical page numbers vs. physical indices) — uses physical indices, matching existing behavior

## Testing

Manual verification in browser:

1. Load a single-page PDF → pager hidden, no input visible.
2. Load a multi-page PDF (e.g., 10 pages) → pager visible, input shows `1`, width fits 2 digits.
3. Click the input → contents select.
4. Type `7`, press Enter → page 7 renders, prev/next button enable states update.
5. Click input, type `999`, press Enter → reverts to current page (clamped/rejected).
6. Click input, type `abc`, press Enter → reverts to current page.
7. Click input, type `5`, press Escape → reverts to current page, no navigation.
8. Click input, type `5`, click elsewhere → reverts, no navigation.
9. Click input, press left arrow → cursor moves in input (does NOT page back).
10. After successful jump, press right arrow on body → page advances (global handler still works).
