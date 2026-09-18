# Goalkeeper Role Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let players be marked as goalkeepers with their own rating, editable after being added, and have team generation automatically split today's two goalkeepers one per side.

**Architecture:** Everything lives in the single existing `index.html` file (no build step, no new files — matches the existing codebase pattern, confirmed with the peer-voting feature already shipped this way). Two new optional fields on a player object (`isGoalkeeper`, `gkRating`) need no schema migration — they're just undefined/falsy on existing players until set. The "one goalkeeper per side" constraint reuses the exact mechanism `generateOptions()` already uses for "no juntos" pairs (an adjacency graph the existing backtracking algorithm already respects) rather than adding new algorithm logic. The app's first per-player *edit* affordance (an expandable row in the roster list) is introduced here, since only add/remove existed before.

**Tech Stack:** Vanilla JS, Firebase Firestore compat SDK (already wired — goalkeeper fields ride along with the existing `players` array sync, no changes needed there), no bundler, no npm dependencies, no test framework. Deployed to Cloudflare Workers static assets, auto-deployed on push to `main` (Workers Builds connected to the GitHub repo).

**Repo:** `tomasgima7/armaequipo`, local clone at `E:\Temp\claude\armaequipo_deploy`.

**Verification approach:** No automated test runner exists in this project — testing is manual, via the Claude Browser tool's `javascript_tool` against a locally served copy of `index.html`. Every task's verification steps give the exact JS to run and the exact expected result. Start (or reuse) the local server with:

```bash
curl -sf http://127.0.0.1:8123/index.html >/dev/null || (cd <repo-root> && nohup npx http-server -p 8123 -c-1 > http.log 2>&1 & disown)
sleep 2
```

Replace `<repo-root>` with whatever working directory this plan is executed from (the main clone, or an isolated worktree if one is set up — see the plan's execution setup).

## Global Constraints

- Single file only: all changes go in `index.html`. No new files.
- The goalkeeper rating (`gkRating`) is **always manual** — never voted on. Do not touch anything in the `?votar=` vote screen or the `votes` array for this feature.
- No edit UI beyond the goalkeeper fields: do not add editing of a player's name or field-player `rating` — those stay add/remove-only, unchanged.
- `p.rating` and the existing Elo/OVR pipeline (`effectiveRating`, `combinedPower`, `recalculateAllElo`, `ovrFor`) must not change behavior for non-goalkeeper use — this feature adds a parallel goalkeeper-specific power calculation, it does not modify the existing one.
- There is exactly one Elo per player, never split by role. A player's `p.elo` is shared regardless of whether they played as goalkeeper or field player that day.
- With 0 or 1 goalkeeper available on a given match day, team generation proceeds with no warnings or blocks — this is a normal case, not an error state.
- With 3+ goalkeepers available, the admin must pick exactly 2 before "Generar equipos" becomes enabled.
- Reuse existing CSS classes (`.field`, `.group-pick`, `.rating-display`, `.btn-ghost`, `.btn-primary`) for all new UI — no new CSS rules needed.
- Do not push to GitHub until the final task — Workers Builds auto-deploys on every push to `main`.

---

## File Structure

Only one file changes: `index.html`. Consistent with the existing codebase (single static file, no build tooling) and the design spec's explicit choice to keep everything there.

---

### Task 1: Add-player form — mark a new player as goalkeeper

**Files:**
- Modify: `index.html` — add-player form HTML (inside the "Tu plantel" panel)
- Modify: `index.html` — `addBtn` click handler and the `pIsGk`/`pGkRating` input wiring

**Interfaces:**
- Produces: two new optional fields on player objects pushed by the add form: `isGoalkeeper: boolean`, `gkRating: number|null` (1.0-10.0, `null` when not a goalkeeper). Every later task that reads a player's goalkeeper status reads exactly these two field names.

- [ ] **Step 1: Add the checkbox + conditional rating slider to the add-player form**

Find (the field wrapping the existing "Calificación" slider, immediately followed by the "Agregar" button):

```html
      <div class="field">
        <label for="prating">Calificación (1.0-10.0)</label>
        <div style="display:flex;align-items:center;gap:10px;">
          <input type="range" id="prating" min="1" max="10" step="0.1" value="6.0">
          <span class="rating-display" id="pratingval">6.0</span>
        </div>
      </div>
      <button class="btn-primary" id="addBtn">Agregar</button>
```

Replace with:

```html
      <div class="field">
        <label for="prating">Calificación (1.0-10.0)</label>
        <div style="display:flex;align-items:center;gap:10px;">
          <input type="range" id="prating" min="1" max="10" step="0.1" value="6.0">
          <span class="rating-display" id="pratingval">6.0</span>
        </div>
      </div>
      <div class="field">
        <label for="pIsGk" style="display:flex;align-items:center;gap:6px;cursor:pointer;text-transform:none;letter-spacing:0;font-size:13px;">
          <input type="checkbox" id="pIsGk">
          <span>¿Es arquero?</span>
        </label>
      </div>
      <div class="field" id="pGkRatingField" style="display:none;">
        <label for="pGkRating">Calificación de arquero (1.0-10.0)</label>
        <div style="display:flex;align-items:center;gap:10px;">
          <input type="range" id="pGkRating" min="1" max="10" step="0.1" value="6.0">
          <span class="rating-display" id="pGkRatingVal">6.0</span>
        </div>
      </div>
      <button class="btn-primary" id="addBtn">Agregar</button>
```

- [ ] **Step 2: Wire the checkbox to show/hide the goalkeeper rating slider, and the slider to update its live display**

Find:

```js
document.getElementById('prating').addEventListener('input', (e)=>{
  document.getElementById('pratingval').textContent = parseFloat(e.target.value).toFixed(1);
});
```

Replace with:

```js
document.getElementById('prating').addEventListener('input', (e)=>{
  document.getElementById('pratingval').textContent = parseFloat(e.target.value).toFixed(1);
});

document.getElementById('pIsGk').addEventListener('change', (e)=>{
  document.getElementById('pGkRatingField').style.display = e.target.checked ? 'flex' : 'none';
});

document.getElementById('pGkRating').addEventListener('input', (e)=>{
  document.getElementById('pGkRatingVal').textContent = parseFloat(e.target.value).toFixed(1);
});
```

- [ ] **Step 3: Store the fields when a player is added, and reset the form afterward**

Find:

```js
document.getElementById('addBtn').addEventListener('click', ()=>{
  const nameInput = document.getElementById('pname');
  const ratingInput = document.getElementById('prating');
  const name = nameInput.value.trim();
  if(!name) { nameInput.focus(); return; }
  const rating = parseFloat(parseFloat(ratingInput.value).toFixed(1));
  players.push({id:uid(), name, rating, elo: initialElo(rating), playing:true, photo: pendingPhoto});
  nameInput.value = '';
  ratingInput.value = 6.0;
  document.getElementById('pratingval').textContent = '6.0';
  pendingPhoto = null;
  document.getElementById('pphoto').value = '';
  document.getElementById('pphotoPreviewWrap').innerHTML = '<div class="avatar-sm avatar-initial">?</div>';
  savePlayers();
  renderAll();
});
```

Replace with:

```js
document.getElementById('addBtn').addEventListener('click', ()=>{
  const nameInput = document.getElementById('pname');
  const ratingInput = document.getElementById('prating');
  const name = nameInput.value.trim();
  if(!name) { nameInput.focus(); return; }
  const rating = parseFloat(parseFloat(ratingInput.value).toFixed(1));
  const gkCheckbox = document.getElementById('pIsGk');
  const gkRatingInput = document.getElementById('pGkRating');
  const isGoalkeeper = gkCheckbox.checked;
  const gkRating = isGoalkeeper ? parseFloat(parseFloat(gkRatingInput.value).toFixed(1)) : null;
  players.push({id:uid(), name, rating, elo: initialElo(rating), playing:true, photo: pendingPhoto, isGoalkeeper, gkRating});
  nameInput.value = '';
  ratingInput.value = 6.0;
  document.getElementById('pratingval').textContent = '6.0';
  gkCheckbox.checked = false;
  document.getElementById('pGkRatingField').style.display = 'none';
  gkRatingInput.value = 6.0;
  document.getElementById('pGkRatingVal').textContent = '6.0';
  pendingPhoto = null;
  document.getElementById('pphoto').value = '';
  document.getElementById('pphotoPreviewWrap').innerHTML = '<div class="avatar-sm avatar-initial">?</div>';
  savePlayers();
  renderAll();
});
```

- [ ] **Step 4: Verify — adding a goalkeeper stores both fields, adding a regular player leaves them falsy**

Server running (command in the plan header), navigate to `http://127.0.0.1:8123/index.html`, run:

```js
document.getElementById('pname').value = 'QA Arquero';
document.getElementById('prating').value = '7.0';
document.getElementById('pIsGk').checked = true;
document.getElementById('pIsGk').dispatchEvent(new Event('change'));
document.getElementById('pGkRating').value = '8.5';
document.getElementById('addBtn').click();

document.getElementById('pname').value = 'QA Jugador';
document.getElementById('prating').value = '5.0';
document.getElementById('addBtn').click();

const gk = players.find(p=>p.name==='QA Arquero');
const field = players.find(p=>p.name==='QA Jugador');
JSON.stringify({
  gkFlags: {isGoalkeeper: gk.isGoalkeeper, gkRating: gk.gkRating, rating: gk.rating},
  fieldFlags: {isGoalkeeper: field.isGoalkeeper, gkRating: field.gkRating}
});
```

Expected: `{"gkFlags":{"isGoalkeeper":true,"gkRating":8.5,"rating":7},"fieldFlags":{"isGoalkeeper":false,"gkRating":null}}`.

Also confirm the checkbox and slider reset after adding:

```js
JSON.stringify({
  checkboxNowUnchecked: document.getElementById('pIsGk').checked === false,
  fieldHidden: document.getElementById('pGkRatingField').style.display === 'none'
});
```

Expected: `{"checkboxNowUnchecked":true,"fieldHidden":true}`.

Clean up:

```js
players = players.filter(p=>!['QA Arquero','QA Jugador'].includes(p.name));
savePlayers();
renderAll();
```

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "$(cat <<'EOF'
Add goalkeeper checkbox and rating to the add-player form

New players can be marked isGoalkeeper with their own gkRating,
independent of the existing field-player rating. Non-goalkeepers get
isGoalkeeper:false, gkRating:null, no behavior change for them.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Edit existing players — expandable row in the roster

**Files:**
- Modify: `index.html` — `renderRoster()` function

**Interfaces:**
- Consumes: `isGoalkeeper`/`gkRating` fields (Task 1).
- Produces: global `let editingPlayerId = null;` (which player's row is currently expanded). This is the app's first per-player edit UI — no other task depends on new function names from this one; `renderRoster()` keeps its existing signature and call sites.

- [ ] **Step 1: Add the "Editar" button and the expandable edit row**

Find the whole `renderRoster()` function:

```js
function renderRoster(){
  const list = document.getElementById('rosterList');
  const empty = document.getElementById('emptyMsg');
  list.innerHTML = '';
  if(players.length === 0){
    empty.style.display = 'block';
  } else {
    empty.style.display = 'none';
  }
  players.forEach(p=>{
    const li = document.createElement('li');
    const received = votes.filter(v=>v.targetId===p.id);
    const voteStatus = received.length > 0
      ? `★ ${(received.reduce((s,v)=>s+v.score,0)/received.length).toFixed(1)} (${received.length} voto${received.length===1?'':'s'})`
      : 'sin votos';
    li.innerHTML = `
      <div class="name-check">
        <div class="avatar-wrap" data-photo-id="${p.id}">${avatarHtml(p, 'avatar-sm')}</div>
        <span class="pname">${escapeHtml(p.name)}</span>
        <span style="color:var(--chalk-dim);font-size:12px;white-space:nowrap;flex-shrink:0;">${escapeHtml(voteStatus)}</span>
      </div>
      <button class="btn-ghost" data-id="${p.id}">Quitar</button>
    `;
    li.querySelector('.avatar-wrap').addEventListener('click', ()=>{
      photoEditTargetId = p.id;
      document.getElementById('playerPhotoInput').click();
    });
    li.querySelector('.btn-ghost').addEventListener('click', ()=>{
      players = players.filter(x=>x.id!==p.id);
      groups = groups
        .map(g=>({...g, playerIds: g.playerIds.filter(id=>id!==p.id)}))
        .filter(g=>g.playerIds.length >= 2);
      avoidPairs = avoidPairs.filter(g=>!g.playerIds.includes(p.id));
      votes = votes.filter(v => v.voterId !== p.id && v.targetId !== p.id);
      savePlayers();
      renderAll();
    });
    list.appendChild(li);
  });
}
```

Replace with:

```js
let editingPlayerId = null;

function renderRoster(){
  const list = document.getElementById('rosterList');
  const empty = document.getElementById('emptyMsg');
  list.innerHTML = '';
  if(players.length === 0){
    empty.style.display = 'block';
  } else {
    empty.style.display = 'none';
  }
  players.forEach(p=>{
    const li = document.createElement('li');
    const received = votes.filter(v=>v.targetId===p.id);
    const voteStatus = received.length > 0
      ? `★ ${(received.reduce((s,v)=>s+v.score,0)/received.length).toFixed(1)} (${received.length} voto${received.length===1?'':'s'})`
      : 'sin votos';
    li.innerHTML = `
      <div class="name-check">
        <div class="avatar-wrap" data-photo-id="${p.id}">${avatarHtml(p, 'avatar-sm')}</div>
        <span class="pname">${escapeHtml(p.name)}${p.isGoalkeeper ? ' 🧤' : ''}</span>
        <span style="color:var(--chalk-dim);font-size:12px;white-space:nowrap;flex-shrink:0;">${escapeHtml(voteStatus)}</span>
      </div>
      <div style="display:flex;gap:6px;flex-shrink:0;">
        <button class="btn-ghost" data-edit-id="${p.id}">Editar</button>
        <button class="btn-ghost" data-id="${p.id}">Quitar</button>
      </div>
    `;
    li.querySelector('.avatar-wrap').addEventListener('click', ()=>{
      photoEditTargetId = p.id;
      document.getElementById('playerPhotoInput').click();
    });
    li.querySelector('[data-edit-id]').addEventListener('click', ()=>{
      editingPlayerId = (editingPlayerId === p.id) ? null : p.id;
      renderRoster();
    });
    li.querySelector('[data-id]').addEventListener('click', ()=>{
      players = players.filter(x=>x.id!==p.id);
      groups = groups
        .map(g=>({...g, playerIds: g.playerIds.filter(id=>id!==p.id)}))
        .filter(g=>g.playerIds.length >= 2);
      avoidPairs = avoidPairs.filter(g=>!g.playerIds.includes(p.id));
      votes = votes.filter(v => v.voterId !== p.id && v.targetId !== p.id);
      savePlayers();
      renderAll();
    });
    list.appendChild(li);

    if(editingPlayerId === p.id){
      const editLi = document.createElement('li');
      const gkRatingValue = (p.gkRating !== undefined && p.gkRating !== null) ? p.gkRating : 6.0;
      editLi.innerHTML = `
        <div style="display:flex;flex-direction:column;gap:10px;width:100%;padding:4px 0;">
          <label style="display:flex;align-items:center;gap:8px;font-size:13px;cursor:pointer;">
            <input type="checkbox" id="gkToggle-${p.id}" ${p.isGoalkeeper ? 'checked' : ''}>
            <span>¿Es arquero?</span>
          </label>
          <div class="field" id="gkRatingField-${p.id}" style="display:${p.isGoalkeeper ? 'flex' : 'none'};flex-direction:row;align-items:center;gap:10px;">
            <label>Calificación de arquero</label>
            <input type="range" id="gkRatingSlider-${p.id}" min="1" max="10" step="0.1" value="${gkRatingValue}">
            <span class="rating-display" id="gkRatingVal-${p.id}">${gkRatingValue.toFixed(1)}</span>
          </div>
        </div>
      `;
      const toggle = editLi.querySelector(`#gkToggle-${p.id}`);
      const ratingField = editLi.querySelector(`#gkRatingField-${p.id}`);
      const slider = editLi.querySelector(`#gkRatingSlider-${p.id}`);
      const val = editLi.querySelector(`#gkRatingVal-${p.id}`);
      toggle.addEventListener('change', (e)=>{
        p.isGoalkeeper = e.target.checked;
        if(p.isGoalkeeper && (p.gkRating === undefined || p.gkRating === null)) p.gkRating = 6.0;
        ratingField.style.display = p.isGoalkeeper ? 'flex' : 'none';
        savePlayers();
        renderAll();
      });
      slider.addEventListener('input', (e)=>{
        const v = parseFloat(parseFloat(e.target.value).toFixed(1));
        p.gkRating = v;
        val.textContent = v.toFixed(1);
      });
      slider.addEventListener('change', ()=>{
        savePlayers();
      });
      list.appendChild(editLi);
    }
  });
}
```

Note: the "Quitar" button's selector changed from `li.querySelector('.btn-ghost')` (which would now ambiguously match the first `.btn-ghost`, i.e. "Editar") to `li.querySelector('[data-id]')`, which is unambiguous since only the "Quitar" button carries a bare `data-id` attribute ("Editar" carries `data-edit-id` instead).

- [ ] **Step 2: Verify — Editar expands the row, toggling the checkbox updates the player, and it survives a re-render**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
players.push({id:'qa1', name:'QA Roster', rating:5.0, elo: initialElo(5.0), playing:true, photo:null, isGoalkeeper:false, gkRating:null});
renderAll();
document.querySelector('[data-edit-id="qa1"]').click();
JSON.stringify({
  toggleExists: !!document.getElementById('gkToggle-qa1'),
  ratingFieldHiddenInitially: document.getElementById('gkRatingField-qa1').style.display === 'none'
});
```

Expected: `{"toggleExists":true,"ratingFieldHiddenInitially":true}`.

Then tick the checkbox and confirm the player object and the roster's 🧤 badge update:

```js
const toggle = document.getElementById('gkToggle-qa1');
toggle.checked = true;
toggle.dispatchEvent(new Event('change'));
const p = players.find(x=>x.id==='qa1');
JSON.stringify({
  isGoalkeeper: p.isGoalkeeper,
  gkRatingDefaulted: p.gkRating,
  badgeShown: document.getElementById('rosterList').innerHTML.includes('QA Roster 🧤'),
  stillExpanded: !!document.getElementById('gkToggle-qa1')
});
```

Expected: `{"isGoalkeeper":true,"gkRatingDefaulted":6,"badgeShown":true,"stillExpanded":true}` — confirms the default 6.0 kicked in, the badge appears, and the edit row survives the `renderAll()` the toggle handler triggers (since `editingPlayerId` persists).

Then drag the slider and confirm it persists to localStorage:

```js
const slider = document.getElementById('gkRatingSlider-qa1');
slider.value = '9.0';
slider.dispatchEvent(new Event('input'));
slider.dispatchEvent(new Event('change'));
const stored = JSON.parse(localStorage.getItem('armaequipos_players')).find(x=>x.id==='qa1');
JSON.stringify({liveDisplay: document.getElementById('gkRatingVal-qa1').textContent, storedGkRating: stored.gkRating});
```

Expected: `{"liveDisplay":"9.0","storedGkRating":9}`.

Clean up:

```js
editingPlayerId = null;
players = players.filter(p=>p.id!=='qa1');
savePlayers();
renderAll();
```

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "$(cat <<'EOF'
Add expandable edit row for existing players' goalkeeper status

Introduces the app's first per-player edit affordance: an "Editar"
button expands the roster row inline to toggle isGoalkeeper and set
gkRating, scoped to just those two fields. Name/field-rating editing
is intentionally out of scope.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: "Elegí 2 arqueros" picker for match day

**Files:**
- Modify: `index.html` — "Armar el partido" panel HTML
- Modify: `index.html` — `updateCounter()` function
- Modify: `index.html` — `renderAll()` (add the new render call)

**Interfaces:**
- Consumes: `isGoalkeeper` (Task 1/2).
- Produces: global `let selectedGkIds = new Set();` (ids of goalkeepers explicitly chosen for today, only meaningful/consulted when 3+ are available) and `renderGkPickSection()`. Task 4 (`generateOptions`) reads `selectedGkIds` directly by name.

- [ ] **Step 1: Add the picker's HTML container**

Find (the end of the "Armar el partido" panel body):

```html
    <p style="color:var(--chalk-dim);font-size:13px;margin:2px 0 10px;">Tildá quiénes juegan hoy (por defecto están todos marcados).</p>
    <ul class="roster" id="pickList"></ul>
    </div>
  </div>
```

Replace with:

```html
    <p style="color:var(--chalk-dim);font-size:13px;margin:2px 0 10px;">Tildá quiénes juegan hoy (por defecto están todos marcados).</p>
    <ul class="roster" id="pickList"></ul>
    <div id="gkPickSection" style="display:none;margin-top:14px;padding-top:14px;border-top:1px dashed var(--pitch-line);">
      <p style="color:var(--chalk-dim);font-size:13px;margin:0 0 8px;">Elegí exactamente 2 arqueros para hoy:</p>
      <div class="group-pick" id="gkPickList"></div>
    </div>
    </div>
  </div>
```

- [ ] **Step 2: Add `selectedGkIds` and `renderGkPickSection()`**

Find:

```js
function updateCounter(){
```

Replace with:

```js
let selectedGkIds = new Set();

function renderGkPickSection(){
  const section = document.getElementById('gkPickSection');
  const list = document.getElementById('gkPickList');
  const candidates = players.filter(p=>p.isGoalkeeper && p.playing);
  if(candidates.length < 3){
    section.style.display = 'none';
    return;
  }
  section.style.display = 'block';
  list.innerHTML = '';
  candidates.forEach(p=>{
    const label = document.createElement('label');
    label.innerHTML = `
      <input type="checkbox" value="${p.id}" ${selectedGkIds.has(p.id) ? 'checked' : ''}>
      <span>${escapeHtml(p.name)}</span>
    `;
    label.querySelector('input').addEventListener('change', (e)=>{
      if(e.target.checked) selectedGkIds.add(p.id); else selectedGkIds.delete(p.id);
      updateCounter();
    });
    list.appendChild(label);
  });
}

function updateCounter(){
```

- [ ] **Step 3: Gate `genBtn` on a valid goalkeeper selection**

Find the whole `updateCounter()` function:

```js
function updateCounter(){
  const size = parseInt(document.getElementById('formatSel').value, 10);
  const needed = size*2;
  const selected = players.filter(p=>p.playing).length;
  document.getElementById('selCount').textContent = selected;
  document.getElementById('neededCount').textContent = needed;
  const warn = document.getElementById('warnMsg');
  const genBtn = document.getElementById('genBtn');
  if(selected === needed && needed > 0){
    warn.textContent = '';
    genBtn.disabled = false;
  } else {
    warn.textContent = selected < needed
      ? `Necesitás ${needed - selected} jugador(es) más marcado(s) para este formato.`
      : `Tenés ${selected - needed} jugador(es) de más marcado(s) para este formato.`;
    genBtn.disabled = true;
  }
}
```

Replace with:

```js
function updateCounter(){
  const size = parseInt(document.getElementById('formatSel').value, 10);
  const needed = size*2;
  const selected = players.filter(p=>p.playing).length;
  document.getElementById('selCount').textContent = selected;
  document.getElementById('neededCount').textContent = needed;
  const warn = document.getElementById('warnMsg');
  const genBtn = document.getElementById('genBtn');
  const gkCandidates = players.filter(p=>p.isGoalkeeper && p.playing);
  const validSelectedCount = Array.from(selectedGkIds).filter(id=>gkCandidates.some(p=>p.id===id)).length;
  const gkOk = gkCandidates.length <= 2 || validSelectedCount === 2;
  if(selected !== needed || needed === 0){
    warn.textContent = selected < needed
      ? `Necesitás ${needed - selected} jugador(es) más marcado(s) para este formato.`
      : `Tenés ${selected - needed} jugador(es) de más marcado(s) para este formato.`;
    genBtn.disabled = true;
  } else if(!gkOk){
    warn.textContent = 'Elegí exactamente 2 arqueros para hoy.';
    genBtn.disabled = true;
  } else {
    warn.textContent = '';
    genBtn.disabled = false;
  }
}
```

- [ ] **Step 4: Call `renderGkPickSection()` from `renderAll()`**

Find:

```js
function renderAll(){
  recalculateAllElo();
  renderRoster();
  renderPickList();
  renderGroupPickList();
```

Replace with:

```js
function renderAll(){
  recalculateAllElo();
  renderRoster();
  renderPickList();
  renderGkPickSection();
  renderGroupPickList();
```

- [ ] **Step 5: Verify — the picker appears only with 3+ candidates, and gates the generate button correctly**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
['qa1','qa2'].forEach((id,i)=>{
  players.push({id, name:'QA GK'+i, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.0});
});
renderAll();
JSON.stringify({ sectionHiddenWith2: document.getElementById('gkPickSection').style.display === 'none' });
```

Expected: `{"sectionHiddenWith2":true}` — with only 2 candidates, no picker needed.

Add a third and confirm the picker appears and gates the button:

```js
players.push({id:'qa3', name:'QA GK2', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.0});
renderAll();
JSON.stringify({
  sectionVisible: document.getElementById('gkPickSection').style.display === 'block',
  checkboxCount: document.querySelectorAll('#gkPickList input[type=checkbox]').length
});
```

Expected: `{"sectionVisible":true,"checkboxCount":3}` — the section renders with one checkbox per candidate once there are 3 or more.

Now check both goalkeeper-count and player-count gating in isolation using a controlled 5v5 pool:

```js
players = players.filter(p=>!['qa1','qa2','qa3'].includes(p.id));
for(let i=0;i<7;i++){ players.push({id:'f'+i, name:'Field'+i, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:false, gkRating:null}); }
['g1','g2','g3'].forEach(id=>{ players.push({id, name:'GK-'+id, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.0}); });
document.getElementById('formatSel').value = '5';
selectedGkIds = new Set();
renderAll();
const stateNoneSelected = { genDisabled: document.getElementById('genBtn').disabled, warn: document.getElementById('warnMsg').textContent };

document.querySelector('#gkPickList input[value="g1"]').checked = true;
document.querySelector('#gkPickList input[value="g1"]').dispatchEvent(new Event('change'));
document.querySelector('#gkPickList input[value="g2"]').checked = true;
document.querySelector('#gkPickList input[value="g2"]').dispatchEvent(new Event('change'));
const stateTwoSelected = { genDisabled: document.getElementById('genBtn').disabled, warn: document.getElementById('warnMsg').textContent };

JSON.stringify({stateNoneSelected, stateTwoSelected});
```

Expected: `stateNoneSelected.genDisabled` is `true` with `warn` containing "Elegí exactamente 2 arqueros para hoy." (10 players are playing — 7 field + 3 GK — matching the needed 10 for 5v5, so the only blocker is the goalkeeper count). `stateTwoSelected.genDisabled` is `false` with `warn` empty.

Clean up:

```js
players = players.filter(p=>!p.id.startsWith('f') && !p.id.startsWith('g') && !['qa1','qa2','qa3'].includes(p.id));
selectedGkIds = new Set();
savePlayers();
renderAll();
```

- [ ] **Step 6: Commit**

```bash
git add index.html && git commit -m "$(cat <<'EOF'
Add "choose 2 goalkeepers for today" picker

With 0-2 goalkeepers playing, nothing changes — they're implicitly
"today's goalkeepers". With 3+, a picker appears and Generar equipos
stays disabled until exactly 2 are chosen.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Split today's goalkeepers across teams in `generateOptions`

**Files:**
- Modify: `index.html` — add `gkCombinedPower()` near `combinedPower()`
- Modify: `index.html` — `generateOptions()` function

**Interfaces:**
- Consumes: `isGoalkeeper`/`gkRating` (Task 1/2), `selectedGkIds` (Task 3).
- Produces: `gkCombinedPower(p)` (goalkeeper-specific power, same shape as `combinedPower(p)`). Modifies the `renderResults(pool, picks, size, shirtLabels)` call site to pass a 5th argument, a `Set` of today's goalkeeper ids — Task 5 reads this 5th parameter by position.

- [ ] **Step 1: Add `gkCombinedPower()`**

Find:

```js
function combinedPower(p){
  const eloScore = eloToScore10(p.elo !== undefined ? p.elo : initialElo(effectiveRating(p)));
  return (effectiveRating(p) + eloScore) / 2;
}
```

Replace with:

```js
function combinedPower(p){
  const eloScore = eloToScore10(p.elo !== undefined ? p.elo : initialElo(effectiveRating(p)));
  return (effectiveRating(p) + eloScore) / 2;
}

// Poder de un jugador jugando de arquero hoy: usa gkRating en vez de la
// calificación de jugador de campo, pero el mismo Elo (no hay Elo separado
// por rol). Si nunca se le cargó gkRating, se usa 6.0 como default.
function gkCombinedPower(p){
  const eloScore = eloToScore10(p.elo !== undefined ? p.elo : initialElo(effectiveRating(p)));
  const gk = (p.gkRating !== undefined && p.gkRating !== null) ? p.gkRating : 6.0;
  return (gk + eloScore) / 2;
}
```

- [ ] **Step 2: Compute today's goalkeepers and a role-aware power function**

Find:

```js
function generateOptions(){
  const size = parseInt(document.getElementById('formatSel').value, 10);
  const pool = players.filter(p=>p.playing);
  const n = pool.length;
  const warn = document.getElementById('warnMsg');
  if(n !== size*2) return;

  const poolIds = new Set(pool.map(p=>p.id));
  const usedIds = new Set();
  const units = []; // {players:[...], size, sum}
  const playerToUnit = {}; // playerId -> unit index

  groups.forEach(g=>{
    const members = pool.filter(p=>g.playerIds.includes(p.id));
    if(members.length >= 2){
      const idx = units.length;
      units.push({players: members, size: members.length, sum: members.reduce((s,p)=>s+combinedPower(p),0)});
      members.forEach(p=>{ usedIds.add(p.id); playerToUnit[p.id] = idx; });
    }
  });
  pool.forEach(p=>{
    if(!usedIds.has(p.id)){
      const idx = units.length;
      units.push({players:[p], size:1, sum:combinedPower(p)});
      playerToUnit[p.id] = idx;
    }
  });
```

Replace with:

```js
function generateOptions(){
  const size = parseInt(document.getElementById('formatSel').value, 10);
  const pool = players.filter(p=>p.playing);
  const n = pool.length;
  const warn = document.getElementById('warnMsg');
  if(n !== size*2) return;

  const gkCandidates = pool.filter(p=>p.isGoalkeeper);
  const todaysGoalkeepers = gkCandidates.length <= 2 ? gkCandidates : gkCandidates.filter(p=>selectedGkIds.has(p.id));
  const todaysGkIdSet = new Set(todaysGoalkeepers.map(p=>p.id));
  function powerOf(p){ return todaysGkIdSet.has(p.id) ? gkCombinedPower(p) : combinedPower(p); }

  const poolIds = new Set(pool.map(p=>p.id));
  const usedIds = new Set();
  const units = []; // {players:[...], size, sum}
  const playerToUnit = {}; // playerId -> unit index

  groups.forEach(g=>{
    const members = pool.filter(p=>g.playerIds.includes(p.id));
    if(members.length >= 2){
      const idx = units.length;
      units.push({players: members, size: members.length, sum: members.reduce((s,p)=>s+powerOf(p),0)});
      members.forEach(p=>{ usedIds.add(p.id); playerToUnit[p.id] = idx; });
    }
  });
  pool.forEach(p=>{
    if(!usedIds.has(p.id)){
      const idx = units.length;
      units.push({players:[p], size:1, sum:powerOf(p)});
      playerToUnit[p.id] = idx;
    }
  });
```

- [ ] **Step 3: Split the 2 goalkeepers across teams, with the same contradiction check style already used for avoid-pairs**

Find:

```js
  if(contradiction){
    warn.textContent = contradiction;
    document.getElementById('resultsPanel').innerHTML = '';
    return;
  }

  // canonicalizar: unidad 0 siempre en el equipo A
```

Replace with:

```js
  if(contradiction){
    warn.textContent = contradiction;
    document.getElementById('resultsPanel').innerHTML = '';
    return;
  }

  if(todaysGoalkeepers.length === 2){
    const [gkA, gkB] = todaysGoalkeepers;
    const uA = playerToUnit[gkA.id], uB = playerToUnit[gkB.id];
    if(uA === uB){
      warn.textContent = 'Los dos arqueros de hoy están en el mismo grupo fijo, no se pueden separar. Corregí la selección de arqueros o el grupo.';
      document.getElementById('resultsPanel').innerHTML = '';
      return;
    }
    adj[uA].add(uB);
    adj[uB].add(uA);
  }

  // canonicalizar: unidad 0 siempre en el equipo A
```

- [ ] **Step 4: Use `powerOf()` for the total-power sum, and thread `todaysGkIdSet` into the results renderer**

Find:

```js
  const totalSum = pool.reduce((s,p)=>s+combinedPower(p),0);
```

Replace with:

```js
  const totalSum = pool.reduce((s,p)=>s+powerOf(p),0);
```

Find:

```js
  renderResults(pool, picks, size, shirtLabels);
}
```

Replace with:

```js
  renderResults(pool, picks, size, shirtLabels, todaysGkIdSet);
}
```

- [ ] **Step 5: Verify — two goalkeepers always end up on different teams, and power balance uses gkRating for them**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
for(let i=0;i<8;i++){ players.push({id:'f'+i, name:'Field'+i, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:false, gkRating:null}); }
players.push({id:'gkX', name:'GK X', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:9.0});
players.push({id:'gkY', name:'GK Y', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:9.0});
document.getElementById('formatSel').value = '5';
selectedGkIds = new Set();
renderAll();
generateOptions();
const opposite = lastPicksData.every(pick=>{
  const aHasX = pick.teamA.some(p=>p.id==='gkX');
  const bHasX = pick.teamB.some(p=>p.id==='gkX');
  const aHasY = pick.teamA.some(p=>p.id==='gkY');
  const bHasY = pick.teamB.some(p=>p.id==='gkY');
  return (aHasX && bHasY) || (bHasX && aHasY);
});
JSON.stringify({ optionsGenerated: lastPicksData.length, alwaysOppositeTeams: opposite });
```

Expected: `optionsGenerated` greater than 0, `alwaysOppositeTeams: true` — every generated option has the two goalkeepers on opposite sides.

Verify the power calculation used `gkRating` (9.0) rather than `rating` (5.0) for the two goalkeepers by checking one option's team averages are pulled noticeably higher than they'd be with 8 players at 5.0 and 2 at 5.0 (which would average exactly 5.0 across the board):

```js
const opt = document.querySelector('#resultsPanel .option .tavg');
JSON.stringify({ firstTeamAvgText: opt ? opt.textContent : null });
```

Expected: an average visibly above 5.0 (the two 9.0-rated-as-goalkeeper players pull their team's average up) — confirms `powerOf()` used `gkCombinedPower` for them, not `combinedPower`.

Clean up:

```js
players = players.filter(p=>!p.id.startsWith('f') && p.id!=='gkX' && p.id!=='gkY');
selectedGkIds = new Set();
document.getElementById('resultsPanel').innerHTML = '';
savePlayers();
renderAll();
```

- [ ] **Step 6: Commit**

```bash
git add index.html && git commit -m "$(cat <<'EOF'
Split today's 2 goalkeepers across teams in generateOptions

Reuses the existing avoid-pair adjacency mechanism (a temporary edge,
not persisted to avoidPairs) so the backtracking algorithm's own logic
needs zero changes. A goalkeeper's contribution to team-power balance
now uses gkRating via the new gkCombinedPower(), everyone else is
unaffected.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Show a 🧤 indicator for each team's goalkeeper in the results

**Files:**
- Modify: `index.html` — `renderResults()` function

**Interfaces:**
- Consumes: the 5th parameter added to the `renderResults()` call site in Task 4 (a `Set` of today's goalkeeper ids).

- [ ] **Step 1: Accept the new parameter and render the icon**

Find:

```js
function renderResults(pool, picks, size, shirtLabels){
  const panel = document.getElementById('resultsPanel');
  if(picks.length === 0){
    panel.innerHTML = '';
    return;
  }
  shirtLabels = shirtLabels || { A: 'EQUIPO A', B: 'EQUIPO B' };
```

Replace with:

```js
function renderResults(pool, picks, size, shirtLabels, gkIds){
  const panel = document.getElementById('resultsPanel');
  if(picks.length === 0){
    panel.innerHTML = '';
    return;
  }
  shirtLabels = shirtLabels || { A: 'EQUIPO A', B: 'EQUIPO B' };
  gkIds = gkIds || new Set();
```

Find:

```js
            <ul>${teamA.map(p=>`<li><span class="pn">${escapeHtml(p.name)}${streakBadge(agg[p.id])}</span></li>`).join('')}</ul>
          </div>
          <div class="vs">VS</div>
          <div class="team">
            <div class="team-head"><span class="tname">${shirtLabels.B}</span><span class="tavg">prom ${avgB}</span></div>
            <ul>${teamB.map(p=>`<li><span class="pn">${escapeHtml(p.name)}${streakBadge(agg[p.id])}</span></li>`).join('')}</ul>
```

Replace with:

```js
            <ul>${teamA.map(p=>`<li><span class="pn">${gkIds.has(p.id) ? '🧤 ' : ''}${escapeHtml(p.name)}${streakBadge(agg[p.id])}</span></li>`).join('')}</ul>
          </div>
          <div class="vs">VS</div>
          <div class="team">
            <div class="team-head"><span class="tname">${shirtLabels.B}</span><span class="tavg">prom ${avgB}</span></div>
            <ul>${teamB.map(p=>`<li><span class="pn">${gkIds.has(p.id) ? '🧤 ' : ''}${escapeHtml(p.name)}${streakBadge(agg[p.id])}</span></li>`).join('')}</ul>
```

- [ ] **Step 2: Verify — the icon shows for exactly the two goalkeepers, on both teams, across every generated option**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
for(let i=0;i<8;i++){ players.push({id:'f'+i, name:'Field'+i, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:false, gkRating:null}); }
players.push({id:'gkX', name:'GK X', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:9.0});
players.push({id:'gkY', name:'GK Y', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:9.0});
document.getElementById('formatSel').value = '5';
selectedGkIds = new Set();
renderAll();
generateOptions();
const html = document.getElementById('resultsPanel').innerHTML;
JSON.stringify({
  hasGkXIcon: html.includes('🧤 GK X'),
  hasGkYIcon: html.includes('🧤 GK Y'),
  fieldPlayerHasNoIcon: !html.includes('🧤 Field0')
});
```

Expected: `{"hasGkXIcon":true,"hasGkYIcon":true,"fieldPlayerHasNoIcon":true}`.

Clean up:

```js
players = players.filter(p=>!p.id.startsWith('f') && p.id!=='gkX' && p.id!=='gkY');
selectedGkIds = new Set();
document.getElementById('resultsPanel').innerHTML = '';
savePlayers();
renderAll();
```

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "$(cat <<'EOF'
Show a 🧤 icon next to each team's goalkeeper in results

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: End-to-end verification against the spec's test plan

**Files:** none (verification only; fix forward in `index.html` only if something actually fails)

Walks through the design spec's "Plan de pruebas" as one continuous flow, on top of everything from Tasks 1-5, to catch anything the per-task checks might have missed in combination.

- [ ] **Step 1: Add a goalkeeper with a distinct rating, confirm both are stored (spec test 1)**

Already covered in depth by Task 1 Step 4. Re-run briefly to confirm it still holds after Tasks 2-5 landed:

```js
document.getElementById('pname').value = 'E2E GK';
document.getElementById('prating').value = '4.0';
document.getElementById('pIsGk').checked = true;
document.getElementById('pIsGk').dispatchEvent(new Event('change'));
document.getElementById('pGkRating').value = '8.0';
document.getElementById('addBtn').click();
const p = players.find(x=>x.name==='E2E GK');
JSON.stringify({rating: p.rating, gkRating: p.gkRating});
```

Expected: `{"rating":4,"gkRating":8}`.

- [ ] **Step 2: Edit an existing player to become a goalkeeper after being added (spec test 2)**

```js
document.getElementById('pname').value = 'E2E Field';
document.getElementById('prating').value = '5.0';
document.getElementById('pIsGk').checked = false;
document.getElementById('addBtn').click();
renderAll();
const fieldP = players.find(x=>x.name==='E2E Field');
document.querySelector(`[data-edit-id="${fieldP.id}"]`).click();
const toggle = document.getElementById(`gkToggle-${fieldP.id}`);
toggle.checked = true;
toggle.dispatchEvent(new Event('change'));
JSON.stringify({ nowGoalkeeper: fieldP.isGoalkeeper, gkRatingField: !!document.getElementById(`gkRatingField-${fieldP.id}`) });
```

Expected: `{"nowGoalkeeper":true,"gkRatingField":true}`.

- [ ] **Step 3: Un-mark a goalkeeper and confirm it drops out as a candidate without losing its stored `gkRating` (spec test 3)**

```js
const toggle2 = document.getElementById(`gkToggle-${fieldP.id}`);
toggle2.checked = false;
toggle2.dispatchEvent(new Event('change'));
const stillHasRating = players.find(x=>x.id===fieldP.id).gkRating;
const candidateNow = players.filter(p=>p.isGoalkeeper && p.playing).some(p=>p.id===fieldP.id);
editingPlayerId = null;
JSON.stringify({ stillHasRating, candidateNow });
```

Expected: `stillHasRating` is a number (not null/undefined — the value from when it was checked, default 6 in this case since it was never set before toggling), `candidateNow: false`.

- [ ] **Step 4: 2 goalkeepers available — always split across teams (spec test 4)**

Already covered in depth by Task 4 Step 5. Re-run with the current roster (which now includes E2E GK as a goalkeeper) plus one more to make exactly 2, and enough field players for a valid format:

```js
players = players.filter(p=>p.name!=='E2E Field'); // remove the toggled-off one from Step 3 to avoid it being a 3rd candidate
for(let i=0;i<8;i++){ players.push({id:'e2ef'+i, name:'E2E Field'+i, rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:false, gkRating:null}); }
players.push({id:'e2eGk2', name:'E2E GK 2', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.5});
document.getElementById('formatSel').value = '5';
selectedGkIds = new Set();
renderAll();
generateOptions();
const gk1 = players.find(x=>x.name==='E2E GK');
const opposite = lastPicksData.length > 0 && lastPicksData.every(pick=>{
  const aHas1 = pick.teamA.some(p=>p.id===gk1.id);
  const aHas2 = pick.teamA.some(p=>p.id==='e2eGk2');
  return aHas1 !== aHas2;
});
JSON.stringify({ optionsGenerated: lastPicksData.length, alwaysOpposite: opposite });
```

Expected: `optionsGenerated > 0`, `alwaysOpposite: true`.

- [ ] **Step 5: 3+ goalkeepers — Generar equipos is blocked until exactly 2 are picked (spec test 5)**

```js
players.push({id:'e2eGk3', name:'E2E GK 3', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:6.5});
selectedGkIds = new Set();
renderAll();
const beforePick = document.getElementById('genBtn').disabled;
document.querySelector(`#gkPickList input[value="${gk1.id}"]`).checked = true;
document.querySelector(`#gkPickList input[value="${gk1.id}"]`).dispatchEvent(new Event('change'));
document.querySelector(`#gkPickList input[value="e2eGk2"]`).checked = true;
document.querySelector(`#gkPickList input[value="e2eGk2"]`).dispatchEvent(new Event('change'));
const afterPick = document.getElementById('genBtn').disabled;
JSON.stringify({ beforePick, afterPick });
```

Expected: `{"beforePick":true,"afterPick":false}`.

- [ ] **Step 6: 1 goalkeeper available — no warnings, generates normally, only one team shows 🧤 (spec test 6)**

```js
players = players.filter(p=>!['e2eGk2','e2eGk3'].includes(p.id));
selectedGkIds = new Set();
renderAll();
generateOptions();
const warnEmpty = document.getElementById('warnMsg').textContent === '';
const html = document.getElementById('resultsPanel').innerHTML;
const gk1IconCount = (html.match(/🧤/g) || []).length;
JSON.stringify({ warnEmpty, gk1IconCount, optionsGenerated: lastPicksData.length });
```

Expected: `warnEmpty: true`, `optionsGenerated > 0`. `gk1IconCount` should equal the number of rendered options (one 🧤 per option, since only one goalkeeper is playing today and appears once per option) — confirm it's greater than 0 and not, say, double per option.

- [ ] **Step 7: 0 goalkeepers available — behaves exactly as before this feature (spec test 7)**

```js
players = players.filter(p=>p.name!=='E2E GK');
selectedGkIds = new Set();
renderAll();
generateOptions();
JSON.stringify({
  warnEmpty: document.getElementById('warnMsg').textContent === '',
  hasGkIcon: document.getElementById('resultsPanel').innerHTML.includes('🧤'),
  optionsGenerated: lastPicksData.length
});
```

Expected: `{"warnEmpty":true,"hasGkIcon":false,"optionsGenerated":<some number greater than 0>}`.

- [ ] **Step 8: Two goalkeepers already in the same fixed group — contradiction warning, no equipos generated (spec test 8)**

```js
players.push({id:'e2eGkA', name:'E2E GK A', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.0});
players.push({id:'e2eGkB', name:'E2E GK B', rating:5.0, elo:initialElo(5.0), playing:true, photo:null, isGoalkeeper:true, gkRating:7.0});
groups.push({id: uid(), playerIds:['e2eGkA','e2eGkB']});
selectedGkIds = new Set();
renderAll();
generateOptions();
JSON.stringify({
  warnText: document.getElementById('warnMsg').textContent,
  resultsCleared: document.getElementById('resultsPanel').innerHTML === ''
});
```

Expected: `warnText` is `"Los dos arqueros de hoy están en el mismo grupo fijo, no se pueden separar. Corregí la selección de arqueros o el grupo."`, `resultsCleared: true`.

- [ ] **Step 9: Confirm the peer-voting flow (built previously) is completely unaffected**

Navigate to `http://127.0.0.1:8123/index.html?votar=e2e-gk-plan-2026`, wait for connection, and confirm no console errors and the vote screen renders (`read_console_messages` with `onlyErrors: true`, and check `document.getElementById('voteWrap').style.display !== 'none'`). Clean up the disposable Firestore doc afterward (`await db.collection('armaequipos').doc('e2e-gk-plan-2026').delete();`, with the listener torn down first via `if(unsubscribe) unsubscribe();`).

- [ ] **Step 10: Clean up all QA data**

```js
players = players.filter(p=>!p.id.startsWith('e2e') && !p.name.startsWith('E2E'));
groups = [];
selectedGkIds = new Set();
editingPlayerId = null;
document.getElementById('resultsPanel').innerHTML = '';
savePlayers();
renderAll();
```

- [ ] **Step 11: If any check above failed, fix the code in `index.html` now and re-run the failing scenario until it passes, then commit the fix**

```bash
git add index.html && git commit -m "fix: address end-to-end verification findings for goalkeeper role"
```

(Skip this step entirely if everything passed on the first run — nothing to commit.)

---

### Task 7: Push and verify the live deploy

**Files:** none (deployment step)

- [ ] **Step 1: Review the full diff before publishing**

```bash
git log --oneline <merge-base>..HEAD
git diff <merge-base>..HEAD -- index.html
```

Confirm the diff only contains the changes from Tasks 1-6 (add form, roster edit, goalkeeper picker, team-generation split, results icon, any e2e fixes).

- [ ] **Step 2: Ask the user for explicit confirmation before pushing/merging**

This project auto-deploys to production on every push to `main` (Cloudflare Workers Builds). Per this project's established workflow, confirm with the user before pushing or merging — do not do it automatically. If this plan was executed on a feature branch/worktree, follow the same integrate-then-confirm-then-push flow used for the peer-voting feature (open a PR, or merge locally, per the user's choice) rather than pushing straight to `main`.

- [ ] **Step 3: After merging to `main`, wait for the Cloudflare Workers Build to finish and confirm success**

```bash
COMMIT=$(git rev-parse main)  # or the merge commit sha
for i in $(seq 1 20); do
  RESULT=$(gh api repos/tomasgima7/armaequipo/commits/$COMMIT/check-runs --jq '.check_runs[] | select(.app.slug=="cloudflare-workers-and-pages") | .status + " " + (.conclusion // "null")')
  echo "attempt $i: $RESULT"
  if echo "$RESULT" | grep -q completed; then break; fi
  sleep 6
done
```

Expected: eventually prints `completed success`.

- [ ] **Step 4: Smoke-test the live site**

Navigate the Browser pane to `https://armaequipo.tomasgima7.workers.dev`, confirm no console errors, and confirm the new goalkeeper checkbox is present in the add-player form:

```js
JSON.stringify({ hasGkCheckbox: !!document.getElementById('pIsGk'), hasGkFn: typeof gkCombinedPower === 'function' });
```

Expected: `{"hasGkCheckbox":true,"hasGkFn":true}`.

- [ ] **Step 5: Report to the user**

Tell the user the goalkeeper feature is live: they can mark a player as arquero when adding them or by editing an existing one, and team generation will automatically split today's two goalkeepers one per side.
