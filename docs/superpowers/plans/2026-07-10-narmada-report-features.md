# Narmada Daily Monsoon Report — Feature Additions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add four capabilities to `Narmada_Daily_Monsoon_Report.html` — arbitrary-date comparison mode, corrected reservoir thresholds (98/75), auto-hiding of empty free-text in the PDF, and attachable auto-arranged media clippings — without altering the plain daily report.

**Architecture:** The file is a single, offline, self-contained HTML tool. State lives in DOM inputs keyed by `id`; `recompute()` runs on every edit to derive all outputs; `snapshot()`/`exportJSON()`/`importJSON()` carry a day as JSON; printing produces the PDF. All new features extend these mechanisms in-place. New comparison figures live in parallel `cmp*` inputs plus a `CMP` snapshot object; attached images live in a `CLIPS` array of base64 data-URIs. Everything is toggled visible only when in use.

**Tech Stack:** Plain HTML + CSS + vanilla JS, no dependencies, no build, no network. Google Fonts is the only external link and must stay optional (report works offline once cached).

## Global Constraints

- Single file only: `Narmada_Daily_Monsoon_Report.html`. No new files except backup + these docs.
- No new runtime dependencies, no network calls, no build step. Fully offline.
- When `Compare` is off and no clippings are attached, the on-screen report **and the printed PDF must be identical to today's**.
- Images embedded as base64 data-URIs (`FileReader.readAsDataURL`) so the JSON snapshot stays self-contained.
- Bilingual labels (English + Gujarati `.guj`) preserved; new sections follow the same pattern.
- Delta colour semantics everywhere: current **higher** than comparison → `up` = red (`--alert`); **lower** → `dn` = green (`--normal`); equal → `flat` = muted.
- Reservoir thresholds: `≥98%` red/ALERT, `75–97%` amber/WATCH, `<75%` green/NORMAL.
- Rainfall comparison metric = **cumulative** season total per taluka. KPI deltas exact only when a full report file is loaded into `CMP`.
- No automated test framework exists or should be added; verify each task by opening the file in a browser.

---

### Task 1: Backup + reservoir thresholds (98/75)

**Files:**
- Create: `Narmada_Daily_Monsoon_Report.html.bak` (verbatim backup)
- Modify: `Narmada_Daily_Monsoon_Report.html` — CSS `.danger` rule (~line 144); JS `gaugeColor`/`gaugeWord` (~lines 433-434)

**Interfaces:**
- Produces: `gaugeColor(pct)` and `gaugeWord(pct)` with the 98/75 thresholds, used by the gauge render loop inside `recompute()`.

- [ ] **Step 1: Make the backup**

Run:
```bash
cp "/Users/panchamkhaitan/work/narmada-reports/Narmada_Daily_Monsoon_Report.html" \
   "/Users/panchamkhaitan/work/narmada-reports/Narmada_Daily_Monsoon_Report.html.bak"
```
Expected: backup file exists (`ls` shows it).

- [ ] **Step 2: Move the red danger marker to the 98% line**

The tank fill grows from the bottom; `.danger` is currently pinned to the top (100%). Replace the rule:

Find:
```css
  .tank .danger{position:absolute;left:-3px;right:-3px;top:0;height:0;border-top:2px dashed var(--alert)}
```
Replace with:
```css
  .tank .danger{position:absolute;left:-3px;right:-3px;top:2%;height:0;border-top:2px dashed var(--alert)}
```
(`top:2%` = the 98% fill line, since fill height is % of danger level.)

- [ ] **Step 3: Update the colour + word thresholds**

Find:
```js
function gaugeColor(pct){return pct>=90?'--alert':pct>=75?'--watch':'--normal';}
function gaugeWord(pct){return pct>=90?'ALERT':pct>=75?'WATCH':'NORMAL';}
```
Replace with:
```js
function gaugeColor(pct){return pct>=98?'--alert':pct>=75?'--watch':'--normal';}
function gaugeWord(pct){return pct>=98?'ALERT':pct>=75?'WATCH':'NORMAL';}
```

- [ ] **Step 4: Verify in browser**

Open `Narmada_Daily_Monsoon_Report.html`. In §3 River & Reservoir:
- Confirm the dashed red line sits just below each tank's top rim (the 98% mark), not exactly at the rim.
- Set a dam's `current` so fill is, e.g., 96% of danger → status reads **WATCH** amber. Set `current` = `danger` (100%) → **ALERT** red. Set `current` low (e.g. 50%) → **NORMAL** green.
- Seed data check: Narmada Dam (126.97/138.68 = 91.6%) should now read **WATCH** amber (was ALERT under old 90 rule). Karjan (102.82/116.10 = 88.6%) → WATCH amber. Nana Kakadi (182.00/187.71 = 96.9%) → WATCH amber. All below 98, so none red — correct.

---

### Task 2: Empty free-text fields drop out of the PDF

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` — print CSS block (~lines 171-184); add JS before/after-print handlers near the INIT block (~line 558)

**Interfaces:**
- Consumes: existing narrative textareas with class `narr` (`execSummary`, `reliefNote`, `n7`–`n14`), their enclosing `.section` (for §1, §6) or `.grid2 > div` (for §§7–14), and the `.grid2` wrapper.
- Produces: `hideEmptyForPrint()` / `restoreAfterPrint()`, wired to `beforeprint`/`afterprint`.

- [ ] **Step 1: Add the print-hidden CSS rule**

Inside the existing `@media print{ ... }` block, add this rule (e.g. right after `body{background:#fff}`):
```css
    .print-hidden{display:none !important}
```
Scoping to `@media print` means the class only hides during printing, so the screen is never affected even if `afterprint` doesn't fire.

- [ ] **Step 2: Add the before/after-print handlers**

Just before the `/* ---------- INIT ---------- */` comment, add:
```js
/* ---------- HIDE EMPTY FREE-TEXT ON PRINT ---------- */
function hideEmptyForPrint(){
  document.querySelectorAll('textarea.narr').forEach(t=>{
    const empty = !t.value.trim();
    const block = t.closest('.grid2 > div') || t.closest('.section');
    if(block) block.classList.toggle('print-hidden', empty);
  });
  const grid=document.querySelector('.grid2');
  if(grid){
    const anyVisible=[...grid.children].some(c=>!c.classList.contains('print-hidden'));
    const sec=grid.closest('.section');
    if(sec) sec.classList.toggle('print-hidden', !anyVisible);
  }
}
function restoreAfterPrint(){
  document.querySelectorAll('.print-hidden').forEach(e=>e.classList.remove('print-hidden'));
}
window.addEventListener('beforeprint',hideEmptyForPrint);
window.addEventListener('afterprint',restoreAfterPrint);
```

- [ ] **Step 3: Verify in browser (print preview)**

Open the file. Clear the §9 Vulnerable Locations textarea and the §11 Media Monitoring textarea. Open Print preview (Cmd-P):
- Confirm §9's and §11's heading **and** box are absent from the preview.
- Confirm other filled sections still print normally.
- Clear all of §§7–14 → confirm the whole two-column block disappears from the preview.
- Clear §1 Executive Summary → confirm §1 section absent. Cancel the dialog; confirm on-screen editor is fully intact (all boxes back).

---

### Task 3: Media clippings — attach multiple images, masonry auto-layout

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` — add CSS (near end of `<style>`), add the new section HTML after the §§7–14 `.section` and before `.foot`, add JS (`CLIPS`, render/add/remove/drag), extend `snapshot()`/`importJSON()`.

**Interfaces:**
- Produces: `CLIPS` (array of data-URI strings), `renderClips()`, `addClips(ev)`, `addFiles(fileList)`, `removeClip(i)`. `snapshot()` gains key `_clips`; `importJSON()` restores it.

- [ ] **Step 1: Add clippings CSS**

Immediately before the `/* responsive */` comment in `<style>`, add:
```css
  /* media clippings */
  .btn-lite{font:inherit;font-size:12px;font-weight:600;cursor:pointer;background:#F3F6F6;color:var(--ink);
    border:1px solid var(--line);border-radius:8px;padding:5px 11px;display:inline-flex;align-items:center;gap:6px}
  .btn-lite:hover{background:#E8EEEE;border-color:var(--water-soft)}
  .clips{column-width:220px;column-gap:14px;margin-top:6px}
  .clips figure{break-inside:avoid;margin:0 0 14px;position:relative;border:1px solid var(--line);
    border-radius:8px;overflow:hidden;background:#fff;box-shadow:var(--shadow)}
  .clips img{width:100%;height:auto;display:block}
  .clip-rm{position:absolute;top:6px;right:6px;width:24px;height:24px;border-radius:50%;border:none;
    background:rgba(15,42,56,.72);color:#fff;font-size:15px;line-height:1;cursor:pointer;
    display:flex;align-items:center;justify-content:center}
  .clip-rm:hover{background:var(--alert)}
  .clips-empty{font-size:12.5px;color:var(--muted);border:1px dashed var(--line);border-radius:8px;
    padding:18px;text-align:center;background:#FCFDFD;margin-top:6px}
  #clipSection.clips-drag{outline:2px dashed var(--water);outline-offset:-8px;background:#F4F9FC}
```
Then, inside the `@media print{ ... }` block, add:
```css
    #clipSection:not(.has-clips){display:none !important}
    .clips{column-width:250px}
```

- [ ] **Step 2: Add the new section HTML**

Immediately after the closing `</div>` of the §§7–14 narrative `.section` (the one containing `.grid2`) and before `<!-- FOOTER -->`, insert:
```html
  <!-- 15. MEDIA CLIPPINGS -->
  <div class="section" id="clipSection">
    <div class="s-head">
      <span class="s-num">15</span>
      <h2 class="s-title">Media Clippings <span class="guj">અખબાર કટિંગ</span></h2>
      <label class="btn-lite no-print" style="margin-left:auto">+ Add images
        <input type="file" accept="image/*" multiple onchange="addClips(event)" style="display:none">
      </label>
    </div>
    <div class="clips" id="clipsGrid"></div>
    <div class="clips-empty no-print" id="clipsEmpty">Drag newspaper images here, or use “+ Add images”. They embed into the report and print with it.</div>
  </div>
```

- [ ] **Step 3: Add the clippings JS**

Just before the `/* ---------- INIT ---------- */` comment, add:
```js
/* ---------- MEDIA CLIPPINGS ---------- */
let CLIPS = [];
function renderClips(){
  const grid=document.getElementById('clipsGrid');
  grid.innerHTML=CLIPS.map((src,i)=>
    `<figure><button class="clip-rm no-print" title="Remove" onclick="removeClip(${i})">×</button>`+
    `<img src="${src}" alt="clipping ${i+1}"></figure>`).join('');
  document.getElementById('clipsEmpty').style.display=CLIPS.length?'none':'';
  document.getElementById('clipSection').classList.toggle('has-clips',CLIPS.length>0);
}
function addFiles(files){
  [...files].filter(f=>f.type&&f.type.startsWith('image/')).forEach(f=>{
    const r=new FileReader();
    r.onload=()=>{CLIPS.push(r.result);renderClips();};
    r.readAsDataURL(f);
  });
}
function addClips(ev){addFiles(ev.target.files);ev.target.value='';}
function removeClip(i){CLIPS.splice(i,1);renderClips();}
(function(){
  const sec=document.getElementById('clipSection');
  ['dragenter','dragover'].forEach(t=>sec.addEventListener(t,ev=>{ev.preventDefault();sec.classList.add('clips-drag');}));
  sec.addEventListener('dragleave',ev=>{if(!sec.contains(ev.relatedTarget))sec.classList.remove('clips-drag');});
  sec.addEventListener('drop',ev=>{ev.preventDefault();sec.classList.remove('clips-drag');addFiles(ev.dataTransfer.files);});
})();
```

- [ ] **Step 4: Persist clippings in save/load**

In `snapshot()`, before `return o;`, add:
```js
  o._clips=CLIPS;
```
In `importJSON()`, inside the `r.onload` try block after the existing `Object.keys(o).forEach(...)` loop, add:
```js
    if(Array.isArray(o._clips)){CLIPS=o._clips;renderClips();}
```

- [ ] **Step 5: Initialise render on load**

In the INIT block, on the line that calls the build functions (`buildRF();buildDMG();buildROADS();buildGauges();`), append:
```js
renderClips();
```

- [ ] **Step 6: Verify in browser**

Open the file, scroll to §15 Media Clippings:
- Click **+ Add images**, select 2–3 of the `WhatsApp Image ….jpeg` clippings (mixed shapes). Confirm they appear in masonry columns, each keeping its aspect ratio (no cropping), empty-prompt gone.
- Drag another image file onto the section → confirm dashed highlight then insertion.
- Hover an image → `×` removes it.
- **Save data (JSON)**, remove all images, then **Load** that JSON → images return.
- Remove all images, open Print preview → confirm §15 is absent. Add one image → confirm it prints.

---

### Task 4: Comparison mode — scaffolding, data plumbing, KPI refactor

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` — toolbar (~lines 189-196); add compare banner after masthead (~line 233); add CSS; add `CMP`/toggle/load/helpers JS; refactor the aggregate math in `recompute()` into `deriveMetrics(get)`; extend `snapshot()`/`importJSON()`.

**Interfaces:**
- Produces:
  - `CMP` (object, a full saved-report snapshot or `{}`)
  - `toggleCompare(on?)` — toggles `body.comparing`, updates toolbar label, shows/hides load button, calls `recompute()`
  - `loadComparison(ev)` — parses a saved JSON into `CMP`, prefills `cmp*` inputs, sets `cmpDate`, turns compare on, recomputes
  - `setCmpInput(id, v)` — sets a comparison input's value if it exists
  - `cval(id)` → number or `NaN` (reads a `cmp*` input; blank = `NaN`)
  - `deltaHTML(cur, cmp, unit, showPct)` → `<span class="delta …">` string, or `'—'` when `cmp` is `NaN`
  - `cmpDateLabel()` → the comparison-date string or `'comparison'`
  - `deriveMetrics(get)` → `{n,pTot,tTot,cTot,lTot,maxCum,hiVal,hiName,aff,rows,dams,damsWatch,dmgTot,houses,humanCas,humanInj,livestock,cattleShed,rTot,evac,camps,sdrf}` where `get(id)` returns a number
  - `liveGet(id)` (= `val(id)`) and `cmpGet(id)` (reads `CMP`)
- Consumes (in Task 5): all of the above.

- [ ] **Step 1: Add comparison CSS**

Before the `/* responsive */` comment in `<style>`, add:
```css
  /* comparison mode */
  body:not(.comparing) .cmp-only{display:none !important}
  .cmp-banner{display:flex;align-items:center;gap:10px;flex-wrap:wrap;padding:9px 30px;
    background:#F0F6FA;border-bottom:1px solid var(--line);font-size:12.5px;color:var(--water);font-weight:600}
  .cmp-banner input{font:inherit;font-size:12.5px;font-weight:600;color:var(--ink);
    border:1px solid var(--line);border-radius:6px;padding:2px 8px;width:120px;background:#fff}
  .cmp-banner .cmp-note{color:var(--muted);font-weight:400}
  .cmp-in{width:60px;font:inherit;font-size:13px;text-align:right;border:1px solid var(--water-soft);
    border-radius:6px;padding:3px 6px;background:#F4F9FC;color:var(--text);font-variant-numeric:tabular-nums}
  .cmp-in:focus{outline:none;border-color:var(--water);box-shadow:0 0 0 3px rgba(29,111,165,.12)}
  .delta{font-size:12px;font-weight:700;white-space:nowrap}
  .delta.up{color:var(--alert)} .delta.dn{color:var(--normal)} .delta.flat{color:var(--muted)}
  .delta small{font-weight:600;font-size:10.5px;opacity:.85}
  .k-delta{margin-top:3px} .k-delta .k-vs{font-weight:500;color:var(--muted);font-size:10px}
  .gdelta{margin-top:2px}
```

- [ ] **Step 2: Add the toolbar controls**

In `.toolbar`, immediately after the "Load yesterday" `<label>…</label>`, add:
```html
  <button class="btn" id="cmpToggle" onclick="toggleCompare()">⇄ Compare: Off</button>
  <label class="btn" id="cmpLoadBtn" style="cursor:pointer" hidden>⭱ Load comparison report
    <input type="file" accept="application/json" onchange="loadComparison(event)" style="display:none">
  </label>
```

- [ ] **Step 3: Add the compare banner**

Immediately after the closing `</div>` of `.masthead` and before `<!-- KPI DASHBOARD -->`, add:
```html
  <!-- COMPARE BANNER -->
  <div class="cmp-banner cmp-only" id="cmpBanner">
    <span>⇄ Comparing against</span>
    <input id="cmpDate" placeholder="dd/mm/yyyy" oninput="recompute()">
    <span class="cmp-note">type each “vs” figure below, or load a saved report to prefill every field</span>
  </div>
```

- [ ] **Step 4: Add comparison helpers + state**

Just after the `STATUS` constant definition (~line 390), add:
```js
/* ---------- COMPARISON STATE + HELPERS ---------- */
let CMP = {};
const liveGet = id => val(id);
const cmpGet  = id => (CMP && id in CMP) ? (parseFloat(CMP[id])||0) : 0;
function cval(id){const e=document.getElementById(id);if(!e)return NaN;const s=(e.value||'').trim();return s===''?NaN:parseFloat(s);}
function cmpDateLabel(){const e=document.getElementById('cmpDate');return (e&&e.value.trim())||'comparison';}
function deltaHTML(cur,cmp,unit,showPct){
  if(cmp===null||cmp===undefined||isNaN(cmp)) return '—';
  const d=cur-cmp, cls=d>0?'up':d<0?'dn':'flat', arrow=d>0?'▲':d<0?'▼':'▪';
  const dp=(unit==='pts')?1:0;
  let pct='';
  if(showPct&&cmp!==0) pct=` <small>(${d>0?'+':''}${(d/cmp*100).toFixed(0)}%)</small>`;
  return `<span class="delta ${cls}">${arrow} ${d>0?'+':''}${d.toFixed(dp)}${unit?' '+unit:''}${pct}</span>`;
}
function setCmpInput(id,v){const e=document.getElementById(id);if(e)e.value=Math.round(v*100)/100;}
function toggleCompare(on){
  const now=(typeof on==='boolean')?on:!document.body.classList.contains('comparing');
  document.body.classList.toggle('comparing',now);
  document.getElementById('cmpToggle').textContent='⇄ Compare: '+(now?'On':'Off');
  document.getElementById('cmpLoadBtn').hidden=!now;
  recompute();
}
function loadComparison(ev){
  const f=ev.target.files[0];if(!f)return;const r=new FileReader();
  r.onload=()=>{try{
    const o=JSON.parse(r.result); CMP=o;
    RF.forEach((_,i)=>setCmpInput('cmpRf'+i,(parseFloat(o['rf'+i+'p'])||0)+(parseFloat(o['rf'+i+'t'])||0)));
    DAMS.forEach((_,i)=>setCmpInput('cmpDg'+i,parseFloat(o['dgC'+i])||0));
    DMG.forEach((_,i)=>setCmpInput('cmpDm'+i,parseFloat(o['dm'+i+'_0'])||0));
    ROADS.forEach((_,i)=>setCmpInput('cmpRd'+i,parseFloat(o['rd'+i])||0));
    if(o.dateTo)document.getElementById('cmpDate').value=o.dateTo;
    toggleCompare(true);
  }catch(e){alert('Could not read that comparison file — is it a saved report?');}};
  r.readAsText(f);ev.target.value='';
}
```

- [ ] **Step 5: Extract `deriveMetrics(get)` and rewrite `recompute()` to use it**

Replace the entire existing `recompute()` function (from `function recompute(){` through its closing `}`) with the following. This preserves all current outputs but sources the aggregate numbers from `deriveMetrics`, so the same math can be reused for the comparison dataset. **Comparison rendering hooks are added in Task 5 — the `/* CMP-HOOK */` comments mark exactly where.**

```js
/* ---------- METRICS (pure: reads via get(id)) ---------- */
function deriveMetrics(get){
  const n=RF.length; let pTot=0,tTot=0,cTot=0,lTot=0,maxCum=0,hiVal=0,hiName='',aff=0;
  const rows=RF.map((r,i)=>{const p=get('rf'+i+'p'),t=get('rf'+i+'t'),l=get('rf'+i+'l'),c=p+t;
    pTot+=p;tTot+=t;cTot+=c;lTot+=l;if(c>maxCum)maxCum=c;if(t>hiVal){hiVal=t;hiName=r[0];}if(t>0)aff++;
    return {p,t,l,c,name:r[0],guj:r[1]};});
  let damsWatch=0; const dams=DAMS.map((d,i)=>{const dg=get('dgD'+i),cu=get('dgC'+i);
    const pct=dg>0?Math.min(cu/dg*100,100):0; if(pct>=75)damsWatch++; return{pct,danger:dg,current:cu};});
  const dmgTot=[0,0,0,0,0,0]; DMG.forEach((r,i)=>{for(let j=0;j<6;j++)dmgTot[j]+=get('dm'+i+'_'+j);});
  const houses=get('dm4_0')+get('dm5_0')+get('dm6_0')+get('dm7_0');
  const humanCas=get('dm0_0'),humanInj=get('dm2_0'),livestock=get('dm1_0'),cattleShed=get('dm8_0');
  let rTot=0; ROADS.forEach((r,i)=>rTot+=get('rd'+i));
  return {n,pTot,tTot,cTot,lTot,maxCum,hiVal,hiName,aff,rows,dams,damsWatch,dmgTot,
    houses,humanCas,humanInj,livestock,cattleShed,rTot,
    evac:get('kEvac'),camps:get('kCamps'),sdrf:get('kSdrf')};
}

/* ---------- RECOMPUTE (runs on every edit) ---------- */
function recompute(){
  const comparing=document.body.classList.contains('comparing');
  const M=deriveMetrics(liveGet);
  const MC=(comparing && CMP && Object.keys(CMP).length)?deriveMetrics(cmpGet):null;
  const n=M.n, scale=Math.max(M.maxCum,1);

  // rainfall table cumulative + totals
  M.rows.forEach((r,i)=>set('rf'+i+'c',r.c.toFixed(0)));
  set('rfPrevTot',M.pTot.toFixed(0));set('rfTodayTot',M.tTot.toFixed(0));
  set('rfCumTot',`${M.cTot.toFixed(0)} <span class="derived" style="font-weight:600">(avg ${(M.cTot/n).toFixed(0)})</span>`);
  set('rfLyTot',M.lTot.toFixed(0));
  /* CMP-HOOK RAINFALL (Task 5) */

  // rainfall bars
  set('rfChart', M.rows.map((r,i)=>{
    const w=(r.c/scale*100);
    const cmpC=cval('cmpRf'+i);
    const tickVal=(comparing&&!isNaN(cmpC))?cmpC:r.l;         /* CMP-HOOK TICK */
    const tickX=tickVal/scale*100;
    const tickTitle=(comparing&&!isNaN(cmpC))?'comparison':'last year';
    return `<div class="bar-row">
      <div class="bl">${r.name}<small class="guj">${r.guj}</small></div>
      <div class="track">
        <div class="fill" style="width:${w}%">
          ${r.t>0?`<div class="today-cap" style="right:0;width:${(r.t/Math.max(r.c,1)*100)}%"></div>`:''}
        </div>
        <div class="ly-tick" title="${tickTitle}" style="left:${tickX}%"></div>
      </div>
      <div class="bar-val">${r.c.toFixed(0)}${r.t>0?`<small> +${r.t.toFixed(0)}</small>`:''}</div>
    </div>`;
  }).join(''));

  // dam gauges
  M.dams.forEach((dm,i)=>{
    const pct=dm.pct,col=gaugeColor(pct),word=gaugeWord(pct),free=(dm.danger-dm.current);
    const L=document.getElementById('gL'+i);
    L.style.height=pct+'%'; L.style.background=`linear-gradient(180deg, color-mix(in srgb, ${cssv(col)} 55%, white), ${cssv(col)})`;
    set('gP'+i,pct.toFixed(0)+'%');
    set('gS'+i,`<span style="color:${cssv(col)}">${word}</span>`);
    set('gF'+i,(free>=0?free.toFixed(2)+' m below':Math.abs(free).toFixed(2)+' m ABOVE')+` danger`);
    /* CMP-HOOK GAUGE (Task 5) */
  });

  // damage totals
  M.dmgTot.forEach((v,j)=>set('dmgT'+j,v));
  // roads
  set('roadTot',M.rTot);
  /* CMP-HOOK DAMAGE+ROADS (Task 5) */

  // reservoir KPI already inside M.damsWatch

  // KPI STRIP
  const kpis=[
    {l:'24-hr Rainfall · આજ',v:M.tTot.toFixed(0),small:'mm',sub:`avg ${(M.tTot/n).toFixed(1)} mm across ${n} talukas`,cmp:MC&&MC.tTot,unit:'mm',pct:true},
    {l:'Highest Station',v:M.hiVal.toFixed(0),small:'mm',sub:M.hiName||'—'},
    {l:'Season Cumulative',v:M.cTot.toFixed(0),small:'mm',sub:`avg ${(M.cTot/n).toFixed(0)} mm`,cmp:MC&&MC.cTot,unit:'mm',pct:true},
    {l:'Affected Talukas',v:M.aff,small:`/ ${n}`,sub:'received rain today',cmp:MC&&MC.aff},
    {l:'Reservoirs on Watch',v:M.damsWatch,small:`/ ${DAMS.length}`,sub:'≥75% of danger level',cls:M.damsWatch>0?'warn':'',cmp:MC&&MC.damsWatch},
    {l:'Human Casualties',v:M.humanCas,sub:`injuries ${M.humanInj}`,cls:M.humanCas>0?'flag':'',cmp:MC&&MC.humanCas},
    {l:'Houses Damaged',v:M.houses,sub:'partial + full',cls:M.houses>0?'warn':'',cmp:MC&&MC.houses},
    {l:'Livestock Lost',v:M.livestock,sub:'cattle sheds '+M.cattleShed,cmp:MC&&MC.livestock},
    {l:'Roads Blocked',v:M.rTot,sub:'R&B network',cls:M.rTot>0?'flag':'',cmp:MC&&MC.rTot},
    {l:'Persons Evacuated',v:val('kEvac'),sub:'edit →',edit:'kEvac',cmp:MC&&MC.evac},
    {l:'Relief Camps',v:val('kCamps'),sub:'operational',edit:'kCamps',cmp:MC&&MC.camps},
    {l:'SDRF / NDRF',v:val('kSdrf'),sub:'teams deployed',edit:'kSdrf',cmp:MC&&MC.sdrf},
  ];
  document.getElementById('kpiStrip').innerHTML = kpis.map(k=>{
    let dl='';
    if(comparing && MC && k.cmp!==undefined && k.cmp!==null && !isNaN(k.cmp)){
      dl=`<div class="k-delta cmp-only">${deltaHTML(parseFloat(k.v)||0,k.cmp,k.unit||'',!!k.pct)} <span class="k-vs">vs ${cmpDateLabel()}</span></div>`;
    }
    return `<div class="kpi ${k.cls||''}">
      <div class="k-label">${k.l}</div>
      <div class="k-val">${k.edit?`<input id="${k.edit}" value="${k.v}" oninput="recompute()">`:k.v}${k.small?` <small>${k.small}</small>`:''}</div>
      <div class="k-sub">${k.sub}</div>${dl}
    </div>`;
  }).join('');

  // alert status band color
  const lvl=document.getElementById('alertLevel').value, sc=cssv(STATUS[lvl].c);
  document.getElementById('statusDot').style.background=sc;
  document.querySelector('.status-pill').style.borderColor=sc;
  document.querySelector('.masthead').style.background=`linear-gradient(160deg, var(--ink), color-mix(in srgb, ${sc} 22%, var(--ink-2)))`;

  // exec summary (only auto-draft if untouched)
  const es=document.getElementById('execSummary');
  if(!es.dataset.touched){
    es.value=`As of ${document.getElementById('asOf').value} on ${document.getElementById('dateTo').value}, the district recorded ${M.tTot.toFixed(0)} mm of rainfall across ${M.aff} of ${n} talukas in the past 24 hours (avg ${(M.tTot/n).toFixed(1)} mm), highest at ${M.hiName} (${M.hiVal.toFixed(0)} mm). Season cumulative stands at ${M.cTot.toFixed(0)} mm (avg ${(M.cTot/n).toFixed(0)} mm). ${M.damsWatch} of ${DAMS.length} reservoirs are at or above 75% of danger level and under watch. Reported impact to date: ${M.humanCas} human casualties, ${M.houses} houses damaged, ${M.rTot} roads blocked, no evacuations. Overall district status: ${STATUS[lvl].t}.`;
    autosize(es);
  }
}
```

Note: the editable KPI values now read `val('kEvac')` etc. (was hardcoded `0`), so evacuation/camps/SDRF entries persist across recomputes — required for their comparison deltas to mean anything.

- [ ] **Step 6: Persist comparison state in save/load**

In `snapshot()`, before `return o;`, add:
```js
  o._cmp=CMP; o._comparing=document.body.classList.contains('comparing');
```
In `importJSON()`, inside the try block after the existing keys loop (and after the `_clips` restore from Task 3), add:
```js
    if(o._cmp&&typeof o._cmp==='object')CMP=o._cmp;
    if(o._comparing)toggleCompare(true); else toggleCompare(false);
```

- [ ] **Step 7: Verify in browser (scaffolding)**

Open the file:
- With Compare **Off**, confirm the report looks exactly as before (no banner, no extra columns, gauges/KPIs normal).
- Click **⇄ Compare: Off** → label becomes **On**, the blue "Comparing against" banner appears, and **⭱ Load comparison report** button appears. No table columns yet (added in Task 5) — that's expected. No console errors.
- Type a value into a KPI evac field, edit an unrelated rainfall cell → confirm the evac value is retained (not reset to 0).
- Toggle Off → banner and load button hide; report normal.

---

### Task 5: Comparison rendering across all sections

**Files:**
- Modify: `Narmada_Daily_Monsoon_Report.html` — `numIn` neighbour helper `cmpIn`; `buildRF`/`buildDMG`/`buildROADS`/`buildGauges` (add `cmp-only` cells); thead/tfoot of the rainfall, damage, roads tables (add `cmp-only` headers/cells); the `/* CMP-HOOK … */` points inside `recompute()`.

**Interfaces:**
- Consumes: `cval`, `deltaHTML`, `cmpDateLabel`, and metric objects `M`/`MC` from Task 4.
- Produces: comparison inputs `cmpRf<i>`, `cmpDg<i>`, `cmpDm<i>`, `cmpRd<i>` and delta cells `dRf<i>`, `dDg<i>`, `dDm<i>`, `dRd<i>`, plus footers `cmpRfTot`/`dRfTot`, `cmpDmTot`/`dDmTot`, `cmpRdTot`/`dRdTot`.

- [ ] **Step 1: Add the `cmpIn` helper**

Right after the existing `numIn` function, add:
```js
function cmpIn(id){return `<input class="cmp-in" id="${id}" inputmode="decimal" oninput="recompute()">`;}
```

- [ ] **Step 2: Rainfall table — headers, cells, footer**

In the rainfall `<thead>`, after `<th>Cumulative (mm)</th>`, add:
```html
        <th class="cmp-only">vs date (cum mm)</th>
        <th class="cmp-only">Change</th>
```
In `buildRF`, change the row template so the cumulative `<td>` is followed by the two comparison cells (insert before the existing "same day last yr" `<td>`):
```js
function buildRF(){
  document.getElementById('rfBody').innerHTML = RF.map((r,i)=>`
    <tr>
      <td class="tname">${r[0]}<small class="guj">${r[1]}</small></td>
      <td>${numIn('rf'+i+'p',r[2])}</td>
      <td>${numIn('rf'+i+'t',r[3])}</td>
      <td class="derived" id="rf${i}c">—</td>
      <td class="cmp-only">${cmpIn('cmpRf'+i)}</td>
      <td class="cmp-only derived" id="dRf${i}">—</td>
      <td>${numIn('rf'+i+'l',r[4])}</td>
    </tr>`).join('');
}
```
In the rainfall `<tfoot>`, after `<td id="rfCumTot"></td>`, add:
```html
        <td class="cmp-only" id="cmpRfTot"></td><td class="cmp-only" id="dRfTot"></td>
```

- [ ] **Step 3: Rainfall — render deltas at the CMP-HOOK**

Replace the line `/* CMP-HOOK RAINFALL (Task 5) */` in `recompute()` with:
```js
  let cmpRfTot=0,cmpRfHas=false;
  M.rows.forEach((r,i)=>{
    const cc=cval('cmpRf'+i);
    set('dRf'+i, comparing?deltaHTML(r.c,cc,'mm',true):'—');
    if(!isNaN(cc)){cmpRfTot+=cc;cmpRfHas=true;}
  });
  set('cmpRfTot', cmpRfHas?cmpRfTot.toFixed(0):'—');
  set('dRfTot', (comparing&&cmpRfHas)?deltaHTML(M.cTot,cmpRfTot,'mm',true):'—');
```

- [ ] **Step 4: Reservoir gauges — comparison input + delta line**

In `buildGauges`, add a `cmp-only` block after the existing `.glevels` div:
```js
function buildGauges(){
  document.getElementById('gauges').innerHTML = DAMS.map((d,i)=>`
    <div class="gauge">
      <div class="gname">${d[0]}<br><span class="guj" style="font-weight:400;color:var(--muted);font-size:10.5px">${d[1]}</span></div>
      <div class="tank"><div class="liquid" id="gL${i}"></div><div class="danger"></div><div class="pct" id="gP${i}"></div></div>
      <div class="gstat" id="gS${i}"></div>
      <div class="gfree" id="gF${i}"></div>
      <div class="glevels">danger ${numIn('dgD'+i,d[2],52)}<br>current ${numIn('dgC'+i,d[3],52)}</div>
      <div class="glevels cmp-only">vs ${cmpIn('cmpDg'+i)}<div class="gdelta" id="dDg${i}">—</div></div>
    </div>`).join('');
}
```
Note: `.cmp-in` is 60px wide; inside the narrow gauge it wraps acceptably. (If it looks cramped during verification, add `style="width:46px"` — optional.)

- [ ] **Step 5: Reservoir — render delta at the CMP-HOOK**

Replace `/* CMP-HOOK GAUGE (Task 5) */` with:
```js
    const cLvl=cval('cmpDg'+i);
    const cPct=(!isNaN(cLvl)&&dm.danger>0)?Math.min(cLvl/dm.danger*100,100):NaN;
    set('dDg'+i, comparing?deltaHTML(pct,cPct,'pts',false):'—');
```

- [ ] **Step 6: Damage table — headers, cells, footer, deltas**

In the damage `<thead>`, after `<th>Pay pending</th>`, add:
```html
<th class="cmp-only">vs date (cases)</th><th class="cmp-only">Change</th>
```
In `buildDMG`, add two cells after the existing six numeric cells:
```js
function buildDMG(){
  document.getElementById('dmgBody').innerHTML = DMG.map((r,i)=>`
    <tr>
      <td class="tname">${r[0]}<small class="guj">${r[1]}</small></td>
      ${[2,3,4,5,6,7].map((c,j)=>`<td>${numIn('dm'+i+'_'+j,r[c])}</td>`).join('')}
      <td class="cmp-only">${cmpIn('cmpDm'+i)}</td>
      <td class="cmp-only derived" id="dDm${i}">—</td>
    </tr>`).join('');
}
```
In the damage `<tfoot>`, after `<td id="dmgT5"></td>`, add:
```html
        <td class="cmp-only" id="cmpDmTot"></td><td class="cmp-only" id="dDmTot"></td>
```

- [ ] **Step 7: Roads table — headers, cells, footer, deltas**

In the roads `<thead>`, after `<th>Roads blocked</th>`, add:
```html
<th class="cmp-only">vs date</th><th class="cmp-only">Change</th>
```
Update `buildROADS`:
```js
function buildROADS(){
  document.getElementById('roadBody').innerHTML = ROADS.map((r,i)=>`
    <tr><td class="tname">${r[0]} <span class="guj" style="color:var(--muted);font-weight:400">${r[1]}</span></td>
    <td>${numIn('rd'+i,r[2],70)}</td>
    <td class="cmp-only">${cmpIn('cmpRd'+i)}</td>
    <td class="cmp-only derived" id="dRd${i}">—</td></tr>`).join('');
}
```
In the roads `<tfoot>`, after `<td id="roadTot"></td>`, add:
```html
<td class="cmp-only" id="cmpRdTot"></td><td class="cmp-only" id="dRdTot"></td>
```

- [ ] **Step 8: Damage + roads — render deltas at the CMP-HOOK**

Replace `/* CMP-HOOK DAMAGE+ROADS (Task 5) */` with:
```js
  let cmpDmTot=0,cmpDmHas=false;
  DMG.forEach((r,i)=>{
    const cc=cval('cmpDm'+i), cur=val('dm'+i+'_0');
    set('dDm'+i, comparing?deltaHTML(cur,cc,'',true):'—');
    if(!isNaN(cc)){cmpDmTot+=cc;cmpDmHas=true;}
  });
  set('cmpDmTot', cmpDmHas?cmpDmTot.toFixed(0):'—');
  set('dDmTot', (comparing&&cmpDmHas)?deltaHTML(M.dmgTot[0],cmpDmTot,'',true):'—');
  let cmpRdTot=0,cmpRdHas=false;
  ROADS.forEach((r,i)=>{
    const cc=cval('cmpRd'+i), cur=val('rd'+i);
    set('dRd'+i, comparing?deltaHTML(cur,cc,'',false):'—');
    if(!isNaN(cc)){cmpRdTot+=cc;cmpRdHas=true;}
  });
  set('cmpRdTot', cmpRdHas?cmpRdTot.toFixed(0):'—');
  set('dRdTot', (comparing&&cmpRdHas)?deltaHTML(M.rTot,cmpRdTot,'',false):'—');
```
Note: `M.dmgTot[0]` is the summed **Total cases** column, matching the per-row `dm<i>_0` comparison.

- [ ] **Step 9: Verify in browser (full comparison)**

Open the file, click **⇄ Compare** on:
- Confirm each table now shows a **vs** column (editable, blue) and a **Change** column, plus a delta line under the reservoir tanks and KPI tiles.
- Type a comparison cumulative for a taluka lower than its current → Change shows green ▼ with mm and %. Higher → red ▲. Blank → `—`.
- **Save** the current report to JSON first. Then **⭱ Load comparison report** and pick that JSON: every `vs` field prefills, `cmpDate` fills from the file's date, KPI tiles show deltas (all ~0 since same file). Now edit a current figure → its Change updates live.
- Reservoir: set a dam's comparison level so its % differs → delta shows `pts`.
- Toggle Compare **Off** → all vs/Change columns, gauge deltas, KPI deltas vanish; report identical to plain. Print preview in this state → no comparison columns in PDF.
- Toggle **On**, Print preview → comparison columns/deltas appear in the PDF.
- **Save data**, reload the page, **Load yesterday** that file → comparison values, `CMP`, compare state, and clippings all restored.

---

## Self-Review

**1. Spec coverage:**
- Comparison mode, all sections, absolute+% with arrows, toggle+load+editable → Tasks 4 & 5 (rainfall, reservoirs, damage, roads, KPIs). ✓
- Reservoir 98/75 thresholds + red marker at 98% → Task 1. ✓
- Empty free-text hidden in PDF (heading + box; full sections and grid items) → Task 2. ✓
- Media clippings: full-width section, no captions, masonry by dimensions, base64, drag+drop, hidden-when-empty in PDF, persisted, comparison-load doesn't clobber → Task 3 (+ `loadComparison` in Task 4 only touches `CMP`/`cmp*`, never `CLIPS`). ✓
- Confirmed defaults (rainfall=cumulative; KPI deltas exact only on full-file load) → encoded in Tasks 4–5. ✓

**2. Placeholder scan:** The `/* CMP-HOOK … */` markers are intentional anchors introduced in Task 4 Step 5 and each is explicitly replaced with real code in Task 5 (Steps 3, 5, 8). No other placeholders; every code step shows complete code.

**3. Type/name consistency:** Verified across tasks — comparison inputs `cmpRf/cmpDg/cmpDm/cmpRd<i>`, delta cells `dRf/dDg/dDm/dRd<i>`, footers `cmpRfTot/dRfTot`, `cmpDmTot/dDmTot`, `cmpRdTot/dRdTot`; helpers `cval`, `deltaHTML(cur,cmp,unit,showPct)`, `cmpIn(id)`, `deriveMetrics(get)`, `liveGet`/`cmpGet`, `toggleCompare(on?)`, `renderClips`/`addFiles`/`CLIPS`. `deriveMetrics` field names used by the KPI array (`tTot,cTot,aff,damsWatch,humanCas,houses,livestock,rTot,evac,camps,sdrf`) all exist in its return object.

**Cross-task ordering note:** Task 5 depends on Task 4 (helpers, `recompute` hooks, metrics). Tasks 1–3 are independent and may be done in any order. `.cmp-in`/`.cmp-only`/`.delta` styles are defined in Task 4 Step 1 before Task 5 uses them.
