# Per-field multiple images — design

**Date:** 2026-07-10
**File touched:** `Narmada_Daily_Monsoon_Report.html` (single self-contained file — no build step, no deps, no network)

## Goal

Today each free-text field (`textarea.narr`) can carry **one** embedded image via
`NARRIMG[id]` → single data-URI string. Extend this to **multiple images per field**, add a
**drag-and-drop surface** per field (drop image files to add), let the user **reorder** the
thumbnails, lay the images out as a **masonry**, and — above all — **never let content run off
the printed page**.

The Media Clippings section (`CLIPS`) already implements the multi-image + drop-surface +
CSS-columns masonry pattern. This work brings that pattern to each field, plus thumbnail
reordering, while keeping print pagination safe.

## Decisions (confirmed with user)

- **Draggable surface = drop-to-add *and* reorder.** Drop desktop image files onto a field to
  add them; drag the added thumbnails to reorder them within that field.
- **Print overflow = flow to next page.** Each individual image stays whole (never clipped);
  when a field has more images than fit, the image group paginates onto the next page.

## 1. Data model

`NARRIMG[id]` changes from a single data-URI **string** to an **array** of data-URI strings.

```js
function normNarr(v){
  if (Array.isArray(v)) return v.filter(s => typeof s === 'string' && s);
  if (typeof v === 'string' && v) return [v];   // old single-image save
  return [];
}
```

- On load (`importJSON`): normalize each entry of the incoming `_narrImg` object through
  `normNarr`, so both old (string) and new (array) saves work.
- On save (`snapshot`): `_narrImg` holds arrays under the same key. No other JSON change.
- Old single-image saves render as a one-thumbnail masonry with zero user action.

## 2. Per-field UI (screen)

Each injected `.narr-attach` box (one per `textarea.narr`, from `initNarrImg()`) becomes:

- An always-present **drop surface**: dashed outline that highlights on drag-over (same visual
  language as the Clippings section's `#clipSection.clips-drag`), containing a
  **"+ Add images"** `<label>` file button with `multiple`.
- Below it, a **masonry** of thumbnails: `column-width`-based (~2 columns within a half-page
  field), `column-gap` small. Each thumbnail is a `<figure class="narr-fig">` with:
  - the `<img>` (max-width:100%, height:auto),
  - a **× remove** button (`.clip-rm no-print`),
  - `draggable="true"` for reordering.

Adding images has two entry points — the `multiple` file picker and dropping files on the
surface — both routed through one function:

```js
function addNarrImgs(id, files){
  [...files].filter(f => f.type && f.type.startsWith('image/')).forEach(f => {
    const r = new FileReader();
    r.onload = () => { (NARRIMG[id] ||= []).push(r.result); renderNarrImg(id); };
    r.readAsDataURL(f);
  });
}
```

(FileReader is async; each onload pushes then re-renders, matching how `addFiles` works for
clippings. Order of appended images may vary with file size, which is acceptable — the user can
reorder.)

`renderNarrImg(id)` rebuilds the box from `NARRIMG[id]` (array): the drop surface + button, then
one `.narr-fig` per data-URI. `box.classList.toggle('has-img', arr.length > 0)`.

## 3. Reorder interaction (HTML5 drag-and-drop)

- **`dragstart`** on a thumbnail: stash `{id, from}` in a module-level `dragState` and set a
  private dataTransfer type `text/narr-reorder` (value = `id + ':' + from`). This marks the drag
  as an *internal reorder* so it is distinguishable from a desktop **file** drop. Add a faint
  "lifting" class to the dragged figure.
- **`dragover`** on a thumbnail (only when the drag is an internal reorder for the *same* field):
  `preventDefault()` and show an insertion cue on the hovered target.
- **`drop`** on a thumbnail: compute target index, splice the array — remove `from`, insert at the
  target position — then `renderNarrImg(id)`.
- **`dragend`**: clear `dragState` and any cue/lifting classes.
- The **field-level drop handler** branches:
  - `ev.dataTransfer.files.length` → treat as **add files** (`addNarrImgs`).
  - else if the drag carries `text/narr-reorder` for this field → let the thumbnail handlers do
    the move (field handler ignores).

Reordering only operates within a single field (the `id` in the reorder payload must match).
Cross-field drags are ignored.

## 4. Print layout (Approach A, with B as tested fallback)

Core principle: **each figure is unbreakable; the field container is breakable.**

- Wrap each field's **heading + textarea text** in a `break-inside:avoid` keep-block so a heading
  never gets orphaned from its text at a page boundary. (For `n7`–`n14` the "heading" is the
  existing `.s-head`; for `execSummary`/`reliefNote` there is a preceding `.s-head` in the same
  section — the keep-block covers head + text.)
- The image masonry is a **breakable** sibling. **Every `.narr-fig` figure gets
  `break-inside:avoid; page-break-inside:avoid`** — so no image is ever split/clipped.
- **Remove `break-inside:avoid` from `.grid2 > div`** (currently line ~248). This is the central
  change: a tall, image-heavy field must be allowed to flow across a page rather than be clipped
  (Chrome clips, rather than shrinks, an avoid-break block taller than the page — the failure the
  project memory records for landscape clippings). Text-only fields are short and will still
  rarely split in practice.
- **Screen masonry = print masonry:** `.narr-masonry{column-width:~150px;column-gap:10px}` for
  both; figures avoid-break. The narrative section prints **portrait** (~1032px content height),
  which per the memory is tall enough for multi-column figures to paginate whole.
- `no-print` on the drop surface, "+ Add images" button, and × controls.
- A field with **no text and no images** stays hidden in print — update `hideEmptyForPrint` to
  test `normNarr(NARRIMG[id]).length === 0` instead of the old truthy-string check. `renumberSections`
  is unaffected.

**Fallback B (only if verification shows slivering):** for the per-field masonry *in print only*,
drop to a single column (`.narr-masonry{column-count:1}` inside `@media print`) so each image is
full field-width and stacked — no nested multi-column across a page break at all. Screen stays
masonry. This is bulletproof pagination at the cost of vertical space.

## 5. Functions changed / added

| Function | Change |
|---|---|
| `NARRIMG` | now `id → string[]` |
| `normNarr(v)` | **new** — normalize string/array/other → `string[]` |
| `renderNarrImg(id)` | render drop-surface + button + masonry of N figures from the array |
| `renderAllNarrImg()` | unchanged (iterates `.narr-attach`) |
| `addNarrImgs(id, files)` | **new/renamed** — append each dropped/picked image to the array |
| `addNarrImg(id, ev)` | becomes a thin wrapper → `addNarrImgs(id, ev.target.files)`; clears input |
| `removeNarrImg(id, index)` | now takes an index; splices the array; re-renders |
| reorder handlers | **new** — dragstart/dragover/drop/dragend on thumbnails |
| `initNarrImg()` | also wire the per-field drop surface (dragenter/over/leave/drop) |
| `snapshot()` | `_narrImg` = arrays (no code change if `NARRIMG` already arrays) |
| `importJSON()` | normalize incoming `_narrImg` values via `normNarr` |
| `hideEmptyForPrint()` | empty test uses array length |

## 6. Verification (headless Chrome — no test framework by design)

Per project convention (`--headless=new --screenshot` for visuals, `--dump-dom` +
`document.title` for logic), verify:

1. **Add multiple** — pick/drop 3 images on a field → 3 thumbnails render in masonry.
2. **Reorder** — simulate a reorder → array order changes, re-render reflects it.
3. **Remove by index** — remove the middle image → correct one gone, others intact.
4. **Old-save migration** — load a JSON with `_narrImg` as `{n7:"data:..."}` (string) → renders
   one thumbnail; re-save → `_narrImg.n7` is now an array.
5. **Print stress (the important one)** — put ~8 images on `n7`, print to PDF via headless
   Chrome, inspect: no image clipped, group flows to a second page, heading stays with its text,
   no field content off the page edge, other fields/layout unbroken. If slivered → apply
   fallback B and re-verify.
6. **No regressions** — Media Clippings, comparison mode, section renumbering, empty-field
   hiding still behave.

## 7. Constraints honored

- Single file; no build step, no runtime deps, no network calls.
- Images embedded as base64 data-URIs → JSON snapshot stays self-contained.
- Backward compatible with existing saves (string and future array forms).
- A pre-change backup already exists at `Narmada_Daily_Monsoon_Report.html.bak`.
