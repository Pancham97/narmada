# Per-field Multiple Images Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let every free-text field (`textarea.narr`) hold multiple embedded images with a drag-and-drop add surface, thumbnail reordering, and a masonry layout that paginates safely in print.

**Architecture:** Single self-contained HTML file. `NARRIMG[id]` changes from one data-URI string to an array of data-URI strings. Each field's injected `.narr-attach` box renders a drop surface + a CSS-columns masonry of `.narr-fig` thumbnails. Reordering uses HTML5 drag-and-drop distinguished from desktop file drops by a private `text/narr-reorder` dataTransfer type. Print CSS keeps every figure unbreakable while allowing the field/section to flow across pages, retiring the "content off the page" risk.

**Tech Stack:** HTML + CSS + vanilla JS, no build step, no runtime deps, no network. Verification via headless Google Chrome.

## Global Constraints

- Single file `Narmada_Daily_Monsoon_Report.html` — no build step, no runtime dependencies, no network calls (Google Fonts link stays optional).
- Images embedded as base64 data-URIs so the JSON snapshot stays self-contained.
- Backward compatible with existing saves: `_narrImg[id]` may be an old single string OR a new array.
- No automated test framework — verify with headless Chrome (`--dump-dom` reading assertions from `document.title`; `--print-to-pdf` / `--screenshot` for print/visual). Add `--allow-file-access-from-files` for local images.
- A pre-change backup already exists at `Narmada_Daily_Monsoon_Report.html.bak` — do not overwrite it.

---

## File Structure

Everything lives in `Narmada_Daily_Monsoon_Report.html`. Regions touched:

- **CSS** (`<style>`, ~lines 190–253): `.narr-attach` block (~212–216) and the `@media print` block (~243–252).
- **JS — image module** (~lines 797–828): `NARRIMG`, `renderNarrImg`, `renderAllNarrImg`, `addNarrImg`, `removeNarrImg`, `initNarrImg`.
- **JS — export/import** (`snapshot` ~730–738, `importJSON` ~745–759).
- **JS — print** (`hideEmptyForPrint` ~846–863).

A throwaway verification harness (Task 0) lives in the scratchpad, not the repo.

---

### Task 0: Verification harness (setup)

**Files:**
- Create: `<SCRATCH>/verify.mjs` (throwaway; `<SCRATCH>` = the session scratchpad dir)

This harness injects an assertion snippet into a copy of the report and reads the result from `document.title`. Every later task's "run to verify" step uses it. Node ships with macOS toolchains; if absent, the same copy-inject-dump can be done by hand.

- [ ] **Step 1: Write the harness**

```js
// <SCRATCH>/verify.mjs  — usage: SCRATCH=<dir> node verify.mjs inject.js
import { readFileSync, writeFileSync } from 'fs';
import { execSync } from 'child_process';
const html   = readFileSync('Narmada_Daily_Monsoon_Report.html', 'utf8');
const inject = readFileSync(process.argv[2], 'utf8');
// app init script is the last <script> before </body>, so our block runs after init:
const out = html.replace('</body>',
  `<script>try{\n${inject}\n}catch(e){document.title='FAIL:'+(e&&e.message||e);}</script></body>`);
const tmp = (process.env.SCRATCH || '.') + '/harness.html';
writeFileSync(tmp, out);
const CHROME = '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome';
const dom = execSync(
  `"${CHROME}" --headless=new --no-sandbox --allow-file-access-from-files --virtual-time-budget=4000 --dump-dom "file://${tmp}"`,
  { encoding: 'utf8' });
const m = dom.match(/<title>([^<]*)<\/title>/);
console.log(m ? m[1] : '(no title)');
```

- [ ] **Step 2: Confirm Chrome is present**

Run: `ls "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"`
Expected: the path prints (no "No such file").

Note: top-level `function`/`let` bindings in one `<script>` are visible to later `<script>` blocks in the same document, so injected assertions can call `renderNarrImg`, mutate `NARRIMG`, etc. directly. No commit — scratchpad only.

---

### Task 1: Data model — `normNarr`, snapshot, import

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` (`NARRIMG` decl ~798; `snapshot` ~735; `importJSON` ~752–753)

**Interfaces:**
- Produces: `normNarr(v) -> string[]` (Array→filtered copy, non-empty string→`[str]`, else `[]`). `NARRIMG` is now `{ [id:string]: string[] }`. `snapshot()._narrImg` is `{ [id]: string[] }` with empty fields omitted. `importJSON` normalizes incoming `_narrImg` values.

- [ ] **Step 1: Add `normNarr` and switch `NARRIMG` semantics**

Replace the declaration line (currently `let NARRIMG = {};   // ...`) with:

```js
let NARRIMG = {};   // textarea id -> array of base64 data-URIs (optional, per field)
function normNarr(v){
  if (Array.isArray(v)) return v.filter(s => typeof s === 'string' && s);
  if (typeof v === 'string' && v) return [v];   // migrate an old single-image save
  return [];
}
```

- [ ] **Step 2: Update `snapshot()` to write normalized arrays**

Replace `o._narrImg=NARRIMG;` with:

```js
o._narrImg={};
Object.keys(NARRIMG).forEach(k=>{const a=normNarr(NARRIMG[k]);if(a.length)o._narrImg[k]=a;});
```

- [ ] **Step 3: Update `importJSON()` to normalize on load**

Replace:

```js
    NARRIMG=(o._narrImg&&typeof o._narrImg==='object')?o._narrImg:{};
    renderAllNarrImg();
```

with:

```js
    NARRIMG={};
    if(o._narrImg&&typeof o._narrImg==='object')
      Object.keys(o._narrImg).forEach(k=>{NARRIMG[k]=normNarr(o._narrImg[k]);});
    renderAllNarrImg();
```

- [ ] **Step 4: Write the failing verification snippet**

Create `<SCRATCH>/t1.js`:

```js
// old string form migrates; array stays; junk drops
const okString = JSON.stringify(normNarr('data:x')) === JSON.stringify(['data:x']);
const okArray  = JSON.stringify(normNarr(['a','','b'])) === JSON.stringify(['a','b']);
const okJunk   = JSON.stringify(normNarr(42)) === JSON.stringify([]);
document.title = (okString && okArray && okJunk) ? 'PASS-T1' : 'FAIL-T1';
```

- [ ] **Step 5: Run — expect FAIL before edits, PASS after**

Run: `SCRATCH=<SCRATCH> node <SCRATCH>/verify.mjs <SCRATCH>/t1.js`
Expected before Steps 1–3: `FAIL:` (normNarr undefined). After: `PASS-T1`.

- [ ] **Step 6: Commit**

```bash
git add Narmada_Daily_Monsoon_Report.html
git commit -m "feat: NARRIMG holds an array of images per field (normNarr migration)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Render masonry + add/remove multiple (screen)

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` (CSS `.narr-attach` block ~212–216; JS `renderNarrImg`/`addNarrImg`/`removeNarrImg` ~799–818)

**Interfaces:**
- Consumes: `normNarr` (Task 1).
- Produces: `renderNarrImg(id)` rebuilds the box from `NARRIMG[id]` (drop surface `.narr-drop` + `.narr-masonry` of `.narr-fig[data-i]`). `addNarrImgs(id, files)` appends each image file. `addNarrImg(id, ev)` = picker wrapper. `removeNarrImg(id, index)` splices by index.

- [ ] **Step 1: Replace the `.narr-attach` CSS block**

Replace lines ~212–216 (`.narr-attach{margin-top:8px}` through `.narr-attach.drag{...}`) with:

```css
  .narr-attach{margin-top:8px}
  .narr-drop{display:flex;align-items:center;gap:10px;font-size:12px;color:var(--muted);
    border:1px dashed var(--line);border-radius:8px;padding:7px 10px;background:#FCFDFD}
  .narr-drop .dz{flex:1}
  .narr-attach.drag .narr-drop{border-color:var(--water);background:#F4F9FC;color:var(--water)}
  .narr-masonry{column-width:150px;column-gap:10px;margin-top:8px}
  .narr-fig{break-inside:avoid;position:relative;margin:0 0 10px;border:1px solid var(--line);
    border-radius:8px;overflow:hidden;background:#fff;box-shadow:var(--shadow);max-width:100%}
  .narr-fig img{display:block;width:100%;height:auto}
  .narr-fig[draggable="true"]{cursor:grab}
  .narr-fig.dragging{opacity:.4}
  .narr-fig.drop-target{outline:2px dashed var(--water);outline-offset:-2px}
```

- [ ] **Step 2: Replace `renderNarrImg` / `addNarrImg` / `removeNarrImg`**

Replace lines ~799–818 (`function renderNarrImg(id){` … end of `removeNarrImg`) with:

```js
function renderNarrImg(id){
  const box=document.querySelector('.narr-attach[data-for="'+id+'"]');
  if(!box) return;
  const arr=NARRIMG[id]=normNarr(NARRIMG[id]);
  const figs=arr.map((src,i)=>
    `<figure class="narr-fig" draggable="true" data-i="${i}">`+
      `<img src="${src}" alt="illustration ${i+1}">`+
      `<button class="clip-rm no-print" title="Remove image" onclick="removeNarrImg('${id}',${i})">×</button>`+
    `</figure>`).join('');
  box.innerHTML =
    `<div class="narr-drop no-print"><span class="dz">Drop image(s) here</span>`+
      `<label class="btn-lite" style="cursor:pointer">+ Add images`+
        `<input type="file" accept="image/*" multiple onchange="addNarrImg('${id}',event)" style="display:none"></label>`+
    `</div>`+
    (figs?`<div class="narr-masonry">${figs}</div>`:'');
  box.classList.toggle('has-img', arr.length>0);
}
function renderAllNarrImg(){document.querySelectorAll('.narr-attach').forEach(b=>renderNarrImg(b.dataset.for));}
function addNarrImgs(id,files){
  [...files].filter(f=>f.type&&f.type.startsWith('image/')).forEach(f=>{
    const r=new FileReader();
    r.onload=()=>{(NARRIMG[id]=normNarr(NARRIMG[id])).push(r.result);renderNarrImg(id);};
    r.readAsDataURL(f);
  });
}
function addNarrImg(id,ev){addNarrImgs(id,ev.target.files);ev.target.value='';}
function removeNarrImg(id,i){const a=NARRIMG[id]=normNarr(NARRIMG[id]);a.splice(i,1);renderNarrImg(id);}
```

(Note: the old `renderAllNarrImg` at ~810 is now included above; delete the standalone duplicate if it remains.)

- [ ] **Step 3: Write the verification snippet**

Create `<SCRATCH>/t2.js`:

```js
NARRIMG.n7 = ['data:1','data:2','data:3'];
renderNarrImg('n7');
const box = document.querySelector('.narr-attach[data-for="n7"]');
const three = box.querySelectorAll('.narr-fig').length === 3;
const hasDrop = !!box.querySelector('.narr-drop');
const hasMasonry = !!box.querySelector('.narr-masonry');
removeNarrImg('n7', 1);                       // remove the middle
const nowTwo = box.querySelectorAll('.narr-fig').length === 2;
const survivor = box.querySelector('img').getAttribute('src') === 'data:1'
              && box.querySelectorAll('img')[1].getAttribute('src') === 'data:3';
document.title = (three&&hasDrop&&hasMasonry&&nowTwo&&survivor) ? 'PASS-T2' : 'FAIL-T2';
```

- [ ] **Step 4: Run — expect PASS**

Run: `SCRATCH=<SCRATCH> node <SCRATCH>/verify.mjs <SCRATCH>/t2.js`
Expected: `PASS-T2` (3 figures render; removing index 1 leaves `data:1`,`data:3`).

- [ ] **Step 5: Visual check (optional but recommended)**

Run: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --no-sandbox --allow-file-access-from-files --virtual-time-budget=4000 --screenshot=<SCRATCH>/t2.png --window-size=1400,2200 "file://<SCRATCH>/harness.html"`
Expected: masonry thumbnails tile in ~2 columns inside field n7; drop surface visible above them.

- [ ] **Step 6: Commit**

```bash
git add Narmada_Daily_Monsoon_Report.html
git commit -m "feat: render per-field images as a masonry with add/remove-by-index

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Drop surface (add files by dragging onto a field)

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` (JS `initNarrImg` ~819–828; add `wireNarrDnD`)

**Interfaces:**
- Consumes: `addNarrImgs` (Task 2).
- Produces: `wireNarrDnD(box, id)` attaches delegated drag/drop listeners to a `.narr-attach` box; `initNarrImg()` calls it once per box. Listeners survive `renderNarrImg`'s `innerHTML` rewrites because they sit on the persistent box.

- [ ] **Step 1: Add `wireNarrDnD` and call it from `initNarrImg`**

Replace `initNarrImg` (~819–828) with:

```js
let narrDrag=null;   // {id, from} while a thumbnail is being dragged to reorder
function wireNarrDnD(box,id){
  box.addEventListener('dragover',ev=>{
    if(narrDrag){ return; }                                   // reorder handled in Task 4
    if([...ev.dataTransfer.types].includes('Files')){ ev.preventDefault(); box.classList.add('drag'); }
  });
  box.addEventListener('dragleave',ev=>{ if(!box.contains(ev.relatedTarget)) box.classList.remove('drag'); });
  box.addEventListener('drop',ev=>{
    if(ev.dataTransfer.files && ev.dataTransfer.files.length){
      ev.preventDefault(); box.classList.remove('drag'); addNarrImgs(id, ev.dataTransfer.files);
    }
  });
}
function initNarrImg(){
  document.querySelectorAll('textarea.narr').forEach(t=>{
    if(!t.id) return;
    if(t.nextElementSibling && t.nextElementSibling.classList.contains('narr-attach')) return;
    const d=document.createElement('div');
    d.className='narr-attach'; d.dataset.for=t.id;
    t.after(d);
    wireNarrDnD(d,t.id);
    renderNarrImg(t.id);
  });
}
```

- [ ] **Step 2: Write the verification snippet (synthesize a file drop)**

Create `<SCRATCH>/t3.js`:

```js
const box = document.querySelector('.narr-attach[data-for="n8"]');
// build a fake image File and a DataTransfer to simulate a desktop drop
const png = 'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNk+M8AAAMBAQDJ/pLvAAAAAElFTkSuQmCC';
const bytes = Uint8Array.from(atob(png), c=>c.charCodeAt(0));
const file = new File([bytes], 't.png', {type:'image/png'});
const dt = new DataTransfer(); dt.items.add(file);
const ev = new DragEvent('drop', {bubbles:true, cancelable:true, dataTransfer:dt});
box.dispatchEvent(ev);
setTimeout(()=>{                                   // FileReader is async
  document.title = box.querySelectorAll('.narr-fig').length===1 ? 'PASS-T3' : 'FAIL-T3';
}, 400);
```

- [ ] **Step 3: Run — expect PASS**

Run: `SCRATCH=<SCRATCH> node <SCRATCH>/verify.mjs <SCRATCH>/t3.js`
Expected: `PASS-T3` (dropped file becomes one thumbnail in field n8). The 4s virtual-time-budget covers the 400ms timeout.

- [ ] **Step 4: Commit**

```bash
git add Narmada_Daily_Monsoon_Report.html
git commit -m "feat: drop-to-add image surface on each free-text field

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Reorder thumbnails within a field

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` (JS `wireNarrDnD` from Task 3; add `moveNarrImg`)

**Interfaces:**
- Consumes: `NARRIMG`, `normNarr`, `renderNarrImg`, `narrDrag`.
- Produces: `moveNarrImg(id, from, to)` inserts the item at `from` immediately before the item at `to` (indices in the pre-move array); drop on empty surface appends to end. Reorder is scoped to a single field via the `text/narr-reorder` payload carrying `id`.

- [ ] **Step 1: Add `moveNarrImg`**

Add near `removeNarrImg`:

```js
function moveNarrImg(id,from,to){        // insert dragged item before the item currently at `to`
  const a=NARRIMG[id]=normNarr(NARRIMG[id]);
  if(from<0||from>=a.length||from===to)return;
  const [img]=a.splice(from,1);
  const dest = to>from ? to-1 : to;      // items left of `to` shifted after the removal
  a.splice(Math.max(0,Math.min(dest,a.length)),0,img);
  renderNarrImg(id);
}
```

- [ ] **Step 2: Extend `wireNarrDnD` with reorder listeners**

Replace the `wireNarrDnD` body from Task 3 with the full version:

```js
function wireNarrDnD(box,id){
  box.addEventListener('dragstart',ev=>{
    const fig=ev.target.closest('.narr-fig'); if(!fig)return;
    narrDrag={id, from:+fig.dataset.i};
    ev.dataTransfer.effectAllowed='move';
    ev.dataTransfer.setData('text/narr-reorder', id+':'+fig.dataset.i);
    fig.classList.add('dragging');
  });
  box.addEventListener('dragend',()=>{
    box.querySelectorAll('.narr-fig').forEach(f=>f.classList.remove('dragging','drop-target'));
    box.classList.remove('drag'); narrDrag=null;
  });
  box.addEventListener('dragover',ev=>{
    if(narrDrag && narrDrag.id===id){                       // internal reorder
      ev.preventDefault(); ev.dataTransfer.dropEffect='move';
      const fig=ev.target.closest('.narr-fig');
      box.querySelectorAll('.narr-fig').forEach(f=>f.classList.toggle('drop-target', f===fig));
    } else if(!narrDrag && [...ev.dataTransfer.types].includes('Files')){   // external files
      ev.preventDefault(); box.classList.add('drag');
    }
  });
  box.addEventListener('dragleave',ev=>{ if(!box.contains(ev.relatedTarget)) box.classList.remove('drag'); });
  box.addEventListener('drop',ev=>{
    box.querySelectorAll('.narr-fig').forEach(f=>f.classList.remove('drop-target'));
    if(ev.dataTransfer.files && ev.dataTransfer.files.length){               // add files
      ev.preventDefault(); box.classList.remove('drag'); addNarrImgs(id, ev.dataTransfer.files); return;
    }
    if(narrDrag && narrDrag.id===id){                                       // reorder
      ev.preventDefault();
      const fig=ev.target.closest('.narr-fig');
      if(fig){ moveNarrImg(id, narrDrag.from, +fig.dataset.i); }
      else { const a=NARRIMG[id]=normNarr(NARRIMG[id]); a.push(a.splice(narrDrag.from,1)[0]); renderNarrImg(id); }
    }
  });
}
```

- [ ] **Step 3: Write the verification snippet (exercise `moveNarrImg` directly)**

Create `<SCRATCH>/t4.js`:

```js
NARRIMG.n9 = ['A','B','C','D']; renderNarrImg('n9');
moveNarrImg('n9', 0, 2);                          // A before C  -> B,A,C,D
const r1 = NARRIMG.n9.join('') === 'BACD';
moveNarrImg('n9', 3, 1);                          // D before A  -> B,D,A,C
const r2 = NARRIMG.n9.join('') === 'BDAC';
// DOM reflects order
const domOrder = [...document.querySelectorAll('.narr-attach[data-for="n9"] img')]
  .map(i=>i.getAttribute('alt')).join('');        // alts are "illustration 1..4"
document.title = (r1 && r2) ? 'PASS-T4' : 'FAIL-T4:'+NARRIMG.n9.join('');
```

- [ ] **Step 4: Run — expect PASS**

Run: `SCRATCH=<SCRATCH> node <SCRATCH>/verify.mjs <SCRATCH>/t4.js`
Expected: `PASS-T4` (insert-before semantics: `BACD` then `BDAC`).

- [ ] **Step 5: Commit**

```bash
git add Narmada_Daily_Monsoon_Report.html
git commit -m "feat: drag thumbnails to reorder images within a field

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Print pagination safety + empty-field hiding

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` (`@media print` block ~243–252; `hideEmptyForPrint` ~846–863)

**Interfaces:**
- Consumes: `normNarr`, `NARRIMG`.
- Produces: print CSS where each `.narr-fig` is unbreakable while the field's section and container flow; `hideEmptyForPrint` hides a field only when it has neither text nor images.

- [ ] **Step 1: Update the print CSS**

In the `@media print` block, replace the three lines:

```css
    .grid2>div{break-inside:avoid;page-break-inside:avoid}
    .narr-fig{box-shadow:none !important;break-inside:avoid;max-width:100%}
    .narr-attach:not(.has-img){display:none !important}
```

with:

```css
    /* narrative fields can now carry many images: let their section + container flow across
       pages (each image stays whole via figure break-inside:avoid) instead of being clipped */
    .section:has(.narr){break-inside:auto;page-break-inside:auto}
    .grid2>div{break-inside:auto;page-break-inside:auto}
    .s-head{break-after:avoid;page-break-after:avoid}   /* heading stays with its text */
    .narr{break-inside:avoid;page-break-inside:avoid}
    .narr-masonry{column-count:2;column-width:auto;column-gap:10px}
    .narr-fig{box-shadow:none !important;break-inside:avoid;page-break-inside:avoid;max-width:100%}
    .narr-attach:not(.has-img){display:none !important}
    .narr-drop{display:none !important}
```

- [ ] **Step 2: Update `hideEmptyForPrint` empty test**

In `hideEmptyForPrint` (~848), replace:

```js
    const empty = !t.value.trim() && !NARRIMG[t.id];   // keep the block if it has an image, even with no text
```

with:

```js
    const empty = !t.value.trim() && normNarr(NARRIMG[t.id]).length===0;   // keep the block if it has any image
```

- [ ] **Step 3: Write the print stress harness (8 tall images on n7)**

Create `<SCRATCH>/t5.js`:

```js
// eight tall SVG data-URIs so the field is forced to paginate
const img = n => 'data:image/svg+xml,'+encodeURIComponent(
  `<svg xmlns="http://www.w3.org/2000/svg" width="260" height="360"><rect width="260" height="360" fill="#cfe3ef"/><text x="20" y="60" font-size="40">${n}</text></svg>`);
NARRIMG.n7 = Array.from({length:8}, (_,i)=>img(i+1));
document.getElementById('n7').value = 'Stress: eight images should paginate whole.';
renderNarrImg('n7');
// hide-empty + section-renumber path the print handler runs:
if (typeof hideEmptyForPrint==='function') hideEmptyForPrint();
const kept = document.querySelector('.grid2>div .narr-fig');   // still present after hideEmptyForPrint
document.title = (NARRIMG.n7.length===8 && kept) ? 'PASS-T5-setup' : 'FAIL-T5';
```

- [ ] **Step 4: Verify the setup logic**

Run: `SCRATCH=<SCRATCH> node <SCRATCH>/verify.mjs <SCRATCH>/t5.js`
Expected: `PASS-T5-setup` (8 images set; field kept for print).

- [ ] **Step 5: Print to PDF and inspect pagination (the important check)**

Run: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --no-sandbox --allow-file-access-from-files --virtual-time-budget=4000 --print-to-pdf=<SCRATCH>/t5.pdf --print-to-pdf-no-header "file://<SCRATCH>/harness.html"`
Then open `<SCRATCH>/t5.pdf`.
Expected: every one of the 8 n7 images renders **whole** (none clipped/slivered); when the group exceeds a page it continues onto the next page; the n7 heading sits with its text; the rest of the narrative grid and other sections are unbroken and nothing runs off the page edge.

- [ ] **Step 6: If — and only if — slivering appears, apply fallback B**

Add inside `@media print` (per-field masonry becomes a single stacked column; no nested multi-column across a page break):

```css
    .narr-masonry{column-count:1}
```

Re-run Step 5 and confirm each image is full-width, stacked, whole, and paginates. Screen layout is unaffected.

- [ ] **Step 7: Regression sweep**

Run: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --no-sandbox --allow-file-access-from-files --virtual-time-budget=4000 --screenshot=<SCRATCH>/t5-screen.png --window-size=1400,3400 "file://Narmada_Daily_Monsoon_Report.html"`
Expected: Media Clippings, reservoir gauges, KPI tiles, and comparison controls look unchanged; no console-visible layout collapse.

- [ ] **Step 8: Commit**

```bash
git add Narmada_Daily_Monsoon_Report.html
git commit -m "feat: print-safe pagination for multi-image fields (figures whole, field flows)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review

**1. Spec coverage**

| Spec section | Task |
|---|---|
| §1 Data model (`normNarr`, string→array, snapshot/import) | Task 1 |
| §2 Per-field UI: drop surface + `multiple` picker + masonry | Tasks 2 (masonry/picker), 3 (drop surface) |
| §3 Reorder interaction (`text/narr-reorder`, scoped to field) | Task 4 |
| §4 Print layout: figures unbreakable, container/section flow, heading kept, empty hidden, `.narr-drop` no-print | Task 5 |
| §5 Migration & snapshot; `removeNarrImg(id,index)`; reorder mutate + re-render | Tasks 1, 2, 4 |
| §6 Verification (headless Chrome, 8-image stress, old-save migration, regressions) | Task 0 harness + Tasks 1/5 |
| Fallback B (single-column print if slivering) | Task 5 Step 6 |

Old-save migration is exercised structurally by `normNarr` (Task 1) and end-to-end by importing a string-form `_narrImg` — covered by the `normNarr('data:x')→['data:x']` assertion; the import path reuses the same function.

**2. Placeholder scan:** No TBD/TODO; every code step shows full code; every verify step gives an exact command and expected output. Fallback B is a concrete one-line rule, not "handle edge cases."

**3. Type consistency:** `NARRIMG[id]` is `string[]` throughout. `normNarr` used in Tasks 1/2/4/5 with the same signature. `renderNarrImg(id)`, `addNarrImgs(id,files)`, `addNarrImg(id,ev)`, `removeNarrImg(id,index)`, `moveNarrImg(id,from,to)`, `wireNarrDnD(box,id)`, `narrDrag={id,from}` are named identically across tasks. Reorder payload type `text/narr-reorder` matches between set (Task 4 Step 2) and the branch guard (`narrDrag.id===id`).

**Deviation noted vs spec §4:** the spec described wrapping heading+text in a `.narr-keep` block; the plan achieves the same "heading never orphaned" guarantee with CSS only (`.s-head{break-after:avoid}` + `.narr{break-inside:avoid}`), avoiding invasive DOM restructuring of all 10 fields. Same outcome, less churn.
