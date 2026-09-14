# Peer Voting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let players in the `armaequipo` roster rate each other from a simple `?votar=<codigo>` link, replacing the Google Forms workflow, with the vote average automatically feeding the existing rating/Elo/OVR pipeline.

**Architecture:** Everything lives in the single existing `index.html` file (no build step, no new files — see File Structure below for why). A new global `votes` array (`{voterId, targetId, score}`) is synced through the same Firestore group document as `players`/`matches`. A query-param (`?votar=<codigo>`) switches the page into a standalone three-step voting UI (who are you → ballot → done) that reuses existing CSS classes. The vote average is read by a new `effectiveRating()` helper that the existing `combinedPower()`/`recalculateAllElo()`/`ovrFor()` functions call instead of the raw manual `rating` field — so match-history Elo recalculation keeps working with zero changes to its own logic.

**Tech Stack:** Vanilla JS, Firebase Firestore (compat SDK, already wired), no bundler, no npm dependencies, no test framework. Deployed to Cloudflare Workers static assets via `wrangler`, auto-deployed on push to `main` (Workers Builds is already connected to the GitHub repo).

**Repo:** `tomasgima7/armaequipo`, working copy at `E:\Temp\claude\armaequipo_deploy`. Every task below assumes that directory as the working directory.

**Verification approach:** This project has no automated test runner — testing today is manual, via the Claude Browser tool's `javascript_tool` (`window`-scoped JS execution) against a locally served copy of `index.html`. Every task's verification steps give the exact JS to run and the exact expected result — treat these the same as you would a `pytest` assertion. Start (or reuse) the local server with:

```bash
curl -sf http://127.0.0.1:8123/index.html >/dev/null || (cd /e/Temp/claude/armaequipo_deploy && nohup npx http-server -p 8123 -c-1 > http.log 2>&1 & disown)
sleep 2
```

This is idempotent — safe to run at the start of every task even if the server is already running from a previous task.

## Global Constraints

- Single file only: all changes go in `index.html`. No new HTML/JS/CSS files (spec decision: Option A, to avoid duplicating CSS/Firebase init across files).
- No authentication, no PIN, no password for voters — identify by picking a name from a list. This is intentional (spec's Non-objetivos), do not add any auth mechanism.
- Votes can never be edited or resubmitted once sent — a `voterId` that has cast any vote is permanently locked out of the ballot form.
- Voters may skip players they don't know; a skipped slider produces no vote entry (do not default it to a mid-value vote).
- No minimum-vote threshold: as soon as a player has 1 or more votes, the average of those votes replaces their manual rating for all Elo/OVR calculations.
- `groups` and `avoidPairs` remain intentionally non-persisted (existing behavior, already implemented) — `votes` is different and **is** persisted to both `localStorage` and Firestore, same as `players`/`matches`.
- Reuse existing CSS classes (`.panel`, `.panel-body`, `.counter`, `.group-pick`, `.roster`, `.rating-display`, `.btn-primary`) for all new UI — no new CSS rules are needed for this feature.
- Do not push to GitHub until the final task — Workers Builds auto-deploys on every push to `main`, so intermediate/incomplete states must stay local commits only.

---

## File Structure

Only one file changes: `index.html`. This is consistent with the existing codebase (a single static file, no build tooling, no other JS/CSS files) and with the spec's explicit choice of Option A over a second `votar.html` file. No new files are created.

---

### Task 1: Vote data plumbing (localStorage + Firestore)

**Files:**
- Modify: `index.html` — global variable block (search for `let players = [];`)
- Modify: `index.html` — `loadPlayers()` function
- Modify: `index.html` — `savePlayers()` function
- Modify: `index.html` — `connectToGroup()` function

**Interfaces:**
- Produces: global `votes` array of `{voterId: string, targetId: string, score: number}`, persisted under `localStorage` key `'armaequipos_votes'` and Firestore field `votesJson` on the same `armaequipos/{code}` document used for `players`/`matches`.

- [ ] **Step 1: Add the `votes` global and its storage key**

Find this block near the top of the `<script>` section:

```js
let players = [];
let groups = []; // [{id, playerIds:[...]}] - deben jugar juntos
let avoidPairs = []; // [{id, playerIds:[a,b]}] - no deben jugar juntos
let matches = []; // [{id, date, size, label, teamAIds:[], teamBIds:[], scoreA, scoreB, winner}]
const STORAGE_KEY = 'armaequipos_players';
const GROUPS_KEY = 'armaequipos_groups';
const AVOID_KEY = 'armaequipos_avoid';
const MATCHES_KEY = 'armaequipos_matches';
const CODE_KEY = 'armaequipos_code';
```

Replace it with:

```js
let players = [];
let groups = []; // [{id, playerIds:[...]}] - deben jugar juntos
let avoidPairs = []; // [{id, playerIds:[a,b]}] - no deben jugar juntos
let matches = []; // [{id, date, size, label, teamAIds:[], teamBIds:[], scoreA, scoreB, winner}]
let votes = []; // [{voterId, targetId, score}] - votos de un jugador sobre otro
const STORAGE_KEY = 'armaequipos_players';
const GROUPS_KEY = 'armaequipos_groups';
const AVOID_KEY = 'armaequipos_avoid';
const MATCHES_KEY = 'armaequipos_matches';
const VOTES_KEY = 'armaequipos_votes';
const CODE_KEY = 'armaequipos_code';
```

- [ ] **Step 2: Load votes from localStorage in `loadPlayers()`**

Find this block inside `loadPlayers()`:

```js
  try{
    const rawMatches = localStorage.getItem(MATCHES_KEY);
    if(rawMatches){
      matches = JSON.parse(rawMatches);
    }
  }catch(e){
    matches = [];
  }
  renderAll();
```

Replace it with:

```js
  try{
    const rawMatches = localStorage.getItem(MATCHES_KEY);
    if(rawMatches){
      matches = JSON.parse(rawMatches);
    }
  }catch(e){
    matches = [];
  }
  try{
    const rawVotes = localStorage.getItem(VOTES_KEY);
    if(rawVotes){
      votes = JSON.parse(rawVotes);
    }
  }catch(e){
    votes = [];
  }
  renderAll();
```

- [ ] **Step 3: Persist votes in `savePlayers()`**

Find the whole `savePlayers()` function:

```js
function savePlayers(){
  // groups y avoidPairs son intencionalmente efímeros: nunca se escriben a
  // localStorage ni a Firestore, ni siquiera después de armar los equipos.
  try{
    localStorage.setItem(STORAGE_KEY, JSON.stringify(players));
    localStorage.setItem(MATCHES_KEY, JSON.stringify(matches));
  }catch(e){
    console.error('No se pudo guardar', e);
  }
  if(currentCode && db && !applyingRemote){
    db.collection('armaequipos').doc(currentCode).set({playersJson: JSON.stringify(players), matchesJson: JSON.stringify(matches)})
      .catch(err=>{
        console.error(err);
        setSyncStatus('Error guardando en la nube. Se guardó localmente.', 'error');
      });
  }
}
```

Replace it with:

```js
function savePlayers(){
  // groups y avoidPairs son intencionalmente efímeros: nunca se escriben a
  // localStorage ni a Firestore, ni siquiera después de armar los equipos.
  // votes sí se persiste: es la base de la calificación por votación de amigos.
  try{
    localStorage.setItem(STORAGE_KEY, JSON.stringify(players));
    localStorage.setItem(MATCHES_KEY, JSON.stringify(matches));
    localStorage.setItem(VOTES_KEY, JSON.stringify(votes));
  }catch(e){
    console.error('No se pudo guardar', e);
  }
  if(currentCode && db && !applyingRemote){
    db.collection('armaequipos').doc(currentCode).set({playersJson: JSON.stringify(players), matchesJson: JSON.stringify(matches), votesJson: JSON.stringify(votes)})
      .catch(err=>{
        console.error(err);
        setSyncStatus('Error guardando en la nube. Se guardó localmente.', 'error');
      });
  }
}
```

- [ ] **Step 4: Sync votes through `connectToGroup()`**

Find this block inside `connectToGroup()`:

```js
  unsubscribe = db.collection('armaequipos').doc(code).onSnapshot(doc=>{
    applyingRemote = true;
    if(doc.exists && doc.data().playersJson){
      players = JSON.parse(doc.data().playersJson);
      matches = doc.data().matchesJson ? JSON.parse(doc.data().matchesJson) : [];
      // groups/avoidPairs nunca viajan por Firestore: cada dispositivo mantiene
      // sus propias restricciones/grupos solo para esta sesión.
    } else {
      // el grupo no existe todavía: lo creamos con lo que tengamos localmente
      db.collection('armaequipos').doc(code).set({playersJson: JSON.stringify(players), matchesJson: JSON.stringify(matches)});
    }
    renderAll();
    applyingRemote = false;
    setSyncStatus(`Sincronizado con el grupo "${code}". Cualquiera con este código ve el mismo plantel.`, 'ok');
  }, err=>{
```

Replace it with:

```js
  unsubscribe = db.collection('armaequipos').doc(code).onSnapshot(doc=>{
    applyingRemote = true;
    if(doc.exists && doc.data().playersJson){
      players = JSON.parse(doc.data().playersJson);
      matches = doc.data().matchesJson ? JSON.parse(doc.data().matchesJson) : [];
      votes = doc.data().votesJson ? JSON.parse(doc.data().votesJson) : [];
      // groups/avoidPairs nunca viajan por Firestore: cada dispositivo mantiene
      // sus propias restricciones/grupos solo para esta sesión.
    } else {
      // el grupo no existe todavía: lo creamos con lo que tengamos localmente
      db.collection('armaequipos').doc(code).set({playersJson: JSON.stringify(players), matchesJson: JSON.stringify(matches), votesJson: JSON.stringify(votes)});
    }
    renderAll();
    applyingRemote = false;
    setSyncStatus(`Sincronizado con el grupo "${code}". Cualquiera con este código ve el mismo plantel.`, 'ok');
  }, err=>{
```

- [ ] **Step 5: Verify — votes round-trip through localStorage**

Start the local server (command above), then navigate the Browser pane to `http://127.0.0.1:8123/index.html` and run:

```js
votes.push({voterId: 'qa1', targetId: 'qa2', score: 8.5});
savePlayers();
JSON.parse(localStorage.getItem('armaequipos_votes'));
```

Expected: returns `[{"voterId":"qa1","targetId":"qa2","score":8.5}]`.

Then reload the page (`navigate` to the same URL again) and run:

```js
JSON.stringify(votes);
```

Expected: `[{"voterId":"qa1","targetId":"qa2","score":8.5}]` — confirms it survived the reload via `loadPlayers()`.

Clean up so later tasks start from a clean slate:

```js
votes = [];
savePlayers();
```

- [ ] **Step 6: Commit (local only — do not push)**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Add votes data plumbing (localStorage + Firestore)

Introduces the votes array ({voterId, targetId, score}) and wires it
through loadPlayers/savePlayers/connectToGroup the same way matches
already works, as groundwork for peer voting.
EOF
)"
```

---

### Task 2: Wire vote average into rating/Elo/OVR

**Files:**
- Modify: `index.html` — ELO section (search for `function combinedPower`)

**Interfaces:**
- Consumes: global `votes` array (Task 1).
- Produces: `effectiveRating(p)` — returns the average of `p`'s received votes if any exist, else `p.rating`. Used internally by `combinedPower()`, `recalculateAllElo()`, and `ovrFor()`.

- [ ] **Step 1: Add `effectiveRating()` and use it in `combinedPower()`**

Find:

```js
function uid(){ return 'p'+Math.random().toString(36).slice(2,9); }

// ---- ELO ----
function initialElo(rating){ return 1000 + rating*100; } // rating 5.0 -> 1500
function eloToScore10(elo){ return (elo-1000)/100; }
function combinedPower(p){
  const eloScore = eloToScore10(p.elo !== undefined ? p.elo : initialElo(p.rating));
  return (p.rating + eloScore) / 2;
}
```

Replace with:

```js
function uid(){ return 'p'+Math.random().toString(36).slice(2,9); }

// ---- ELO ----
function initialElo(rating){ return 1000 + rating*100; } // rating 5.0 -> 1500
function eloToScore10(elo){ return (elo-1000)/100; }

// Si el jugador tiene votos de sus compañeros, su calificación efectiva es el
// promedio de esos votos. Si no tiene ninguno, se usa la calificación manual
// (la que se le puso al agregarlo). El campo p.rating manual nunca se pisa.
function effectiveRating(p){
  const received = votes.filter(v=>v.targetId===p.id);
  if(received.length === 0) return p.rating;
  return received.reduce((sum,v)=>sum+v.score, 0) / received.length;
}

function combinedPower(p){
  const eloScore = eloToScore10(p.elo !== undefined ? p.elo : initialElo(effectiveRating(p)));
  return (effectiveRating(p) + eloScore) / 2;
}
```

- [ ] **Step 2: Use `effectiveRating()` as the Elo baseline in `recalculateAllElo()`**

Find:

```js
function recalculateAllElo(){
  const eloMap = {};
  players.forEach(p=>{ eloMap[p.id] = initialElo(p.rating); });
  const K = 24;
```

Replace with:

```js
function recalculateAllElo(){
  const eloMap = {};
  players.forEach(p=>{ eloMap[p.id] = initialElo(effectiveRating(p)); });
  const K = 24;
```

- [ ] **Step 3: Use `effectiveRating()` in `ovrFor()`'s fallback path**

Find:

```js
function ovrFor(p){
  const score10 = eloToScore10(p.elo !== undefined ? p.elo : initialElo(p.rating));
  return Math.max(1, Math.min(99, Math.round(score10*10)));
}
```

Replace with:

```js
function ovrFor(p){
  const score10 = eloToScore10(p.elo !== undefined ? p.elo : initialElo(effectiveRating(p)));
  return Math.max(1, Math.min(99, Math.round(score10*10)));
}
```

- [ ] **Step 4: Verify — votes shift the Elo baseline correctly**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
players.push({id:'qa1', name:'QA1', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa2', name:'QA2', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
recalculateAllElo();
const before = players.find(p=>p.id==='qa1').elo;
JSON.stringify({before, expectedBefore: initialElo(5.0)});
```

Expected: `before` equals `expectedBefore` (1500) — no matches played yet, no votes yet, baseline is the manual rating.

Then:

```js
votes.push({voterId:'qa2', targetId:'qa1', score: 9.0});
recalculateAllElo();
const after = players.find(p=>p.id==='qa1').elo;
JSON.stringify({after, expectedAfter: initialElo(9.0)});
```

Expected: `after` equals `expectedAfter` (1900) — the single vote of 9.0 became the effective rating with no match history to offset it.

- [ ] **Step 5: Verify — existing match-history deltas still apply on top of the new baseline**

```js
matches.push({id:'qam1', date:new Date().toISOString(), size:5, label:'qa', teamAIds:['qa2'], teamBIds:['qa1'], scoreA:1, scoreB:0, winner:'A'});
recalculateAllElo();
const eloQa1WithVote = players.find(p=>p.id==='qa1').elo;
const deltaFromVoteBaseline = eloQa1WithVote - initialElo(9.0);

votes = votes.filter(v=>v.targetId!=='qa1'); // simulate "no vote" to compare deltas
recalculateAllElo();
const eloQa1NoVote = players.find(p=>p.id==='qa1').elo;
const deltaFromManualBaseline = eloQa1NoVote - initialElo(5.0);

JSON.stringify({deltaFromVoteBaseline, deltaFromManualBaseline});
```

Expected: both deltas are equal (qa1 lost the same match either way, so the Elo *movement* from its own baseline is identical — only the baseline itself differs). This confirms `recalculateAllElo()`'s match-replay logic didn't need to change; it just now starts from a different number.

Clean up:

```js
players = players.filter(p=>!['qa1','qa2'].includes(p.id));
matches = matches.filter(m=>m.id!=='qam1');
votes = [];
savePlayers();
```

- [ ] **Step 6: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Feed peer-vote average into rating/Elo/OVR calculation

Adds effectiveRating(), which returns the average of a player's
received votes when any exist, otherwise the manual rating.
combinedPower/recalculateAllElo/ovrFor now read through it, so
match-history Elo replay keeps working unchanged from a new baseline.
EOF
)"
```

---

### Task 3: Show vote status in the admin roster panel

**Files:**
- Modify: `index.html` — `renderRoster()` function

**Interfaces:**
- Consumes: `votes` (Task 1), `effectiveRating()` is not needed here — this displays raw vote count/average text, independent of the calculation itself.

- [ ] **Step 1: Add a vote-status label per player row**

Find:

```js
  players.forEach(p=>{
    const li = document.createElement('li');
    li.innerHTML = `
      <div class="name-check">
        <div class="avatar-wrap" data-photo-id="${p.id}">${avatarHtml(p, 'avatar-sm')}</div>
        <span class="pname">${escapeHtml(p.name)}</span>
      </div>
      <button class="btn-ghost" data-id="${p.id}">Quitar</button>
    `;
```

Replace with:

```js
  players.forEach(p=>{
    const li = document.createElement('li');
    const received = votes.filter(v=>v.targetId===p.id);
    const voteStatus = received.length > 0
      ? `★ ${(received.reduce((s,v)=>s+v.score,0)/received.length).toFixed(1)} (${received.length} voto${received.length===1?'':'s'})`
      : 'sin votos todavía (usando calificación manual)';
    li.innerHTML = `
      <div class="name-check">
        <div class="avatar-wrap" data-photo-id="${p.id}">${avatarHtml(p, 'avatar-sm')}</div>
        <span class="pname">${escapeHtml(p.name)}</span>
        <span style="color:var(--chalk-dim);font-size:12px;white-space:nowrap;">${escapeHtml(voteStatus)}</span>
      </div>
      <button class="btn-ghost" data-id="${p.id}">Quitar</button>
    `;
```

- [ ] **Step 2: Verify — vote status text appears correctly**

Server running, navigate to `http://127.0.0.1:8123/index.html`, run:

```js
players.push({id:'qa1', name:'QA Sin Votos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa2', name:'QA Con Votos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
votes.push({voterId:'qa1', targetId:'qa2', score:7.0});
votes.push({voterId:'other', targetId:'qa2', score:9.0});
renderAll();
const html = document.getElementById('rosterList').innerHTML;
JSON.stringify({
  hasNoVotesText: html.includes('sin votos todavía'),
  hasAverageText: html.includes('★ 8.0 (2 votos)')
});
```

Expected: both `true` (average of 7.0 and 9.0 is 8.0).

Clean up:

```js
players = players.filter(p=>!['qa1','qa2'].includes(p.id));
votes = [];
savePlayers();
renderAll();
```

- [ ] **Step 3: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Show vote count/average in the admin roster panel

Each player row now shows either their vote average and count, or a
note that they're still using the manual rating, so the admin can
see voting progress without leaving the app.
EOF
)"
```

---

### Task 4: Vote-mode HTML scaffold and query-param detection

**Files:**
- Modify: `index.html` — `<div class="wrap">` opening tag (add `id`)
- Modify: `index.html` — insert new markup after the existing `.wrap` closes, before `<script>`
- Modify: `index.html` — global variable block (mode detection)
- Modify: `index.html` — `renderAll()` (add `renderVoteScreen()` call + stub)
- Modify: `index.html` — bottom of script (branch `loadPlayers()` vs `connectToGroup()`)

**Interfaces:**
- Produces: `voteMode` (boolean), `voteCode` (string|null), `selectedVoterId` (string|null, mutable) globals. `renderVoteScreen()` stub (filled in by Task 5).
- Consumes: nothing new yet.

- [ ] **Step 1: Give the existing app wrapper a stable id**

Find (this exact string appears once, right after the `<div class="scoreboard">` header block):

```html
<div class="wrap">

  <div class="panel collapsed" id="syncPanel">
```

Replace with:

```html
<div class="wrap" id="appWrap">

  <div class="panel collapsed" id="syncPanel">
```

- [ ] **Step 2: Insert the vote-mode markup before the closing `</script>` tag's opening**

Find (the end of the existing `.wrap`, right before the `<script>` tag):

```html
    <div id="historyList"></div>
    </div>
  </div>

</div>

<script>
```

Replace with:

```html
    <div id="historyList"></div>
    </div>
  </div>

</div>

<div class="wrap" id="voteWrap" style="display:none;">

  <div class="panel" id="voteStepWho">
    <div class="panel-body">
      <div class="counter" id="voteCounter">Cargando...</div>
      <p style="color:var(--chalk-dim);font-size:13px;margin:14px 0 8px;">¿Quién sos?</p>
      <div class="group-pick" id="voteWhoList"></div>
    </div>
  </div>

  <div class="panel" id="voteStepBallot" style="display:none;">
    <div class="panel-body">
      <p style="color:var(--chalk-dim);font-size:13px;margin:0 0 12px;">Puntuá del 1 al 10 a tus compañeros. Si no conocés bien a alguien, dejalo sin tocar y no se cuenta.</p>
      <ul class="roster" id="voteBallotList"></ul>
      <button class="btn-primary" id="voteSubmitBtn" style="margin-top:16px;">Enviar votos</button>
    </div>
  </div>

  <div class="panel" id="voteStepDone" style="display:none;">
    <div class="panel-body">
      <p style="color:var(--chalk);font-size:16px;">¡Gracias! Tu voto ya quedó registrado.</p>
    </div>
  </div>

</div>

<script>
```

- [ ] **Step 3: Add vote-mode detection globals**

Find:

```js
let db = null;
let unsubscribe = null;
let currentCode = null;
let applyingRemote = false; // evita loops al recibir datos remotos

function firebaseReady(){
```

Replace with:

```js
let db = null;
let unsubscribe = null;
let currentCode = null;
let applyingRemote = false; // evita loops al recibir datos remotos

// ---- Modo votación (?votar=<codigo>) ----
function getVoteCodeFromUrl(){
  const raw = new URLSearchParams(location.search).get('votar');
  if(!raw) return null;
  return raw.trim().toLowerCase().replace(/\s+/g, '-');
}
const voteCode = getVoteCodeFromUrl();
const voteMode = !!voteCode;
let selectedVoterId = null;
if(voteMode){
  document.getElementById('appWrap').style.display = 'none';
  document.getElementById('voteWrap').style.display = '';
}

function firebaseReady(){
```

- [ ] **Step 4: Call a `renderVoteScreen()` stub from `renderAll()`**

Find:

```js
function renderAll(){
  recalculateAllElo();
  renderRoster();
  renderPickList();
  renderGroupPickList();
  renderGroupChips();
  renderAvoidPickList();
  renderAvoidChips();
  renderManualPickLists();
  renderHistoryAndRanking();
  seedPartidoDomingo();
  updateCounter();
}
```

Replace with:

```js
function renderAll(){
  recalculateAllElo();
  renderRoster();
  renderPickList();
  renderGroupPickList();
  renderGroupChips();
  renderAvoidPickList();
  renderAvoidChips();
  renderManualPickLists();
  renderHistoryAndRanking();
  seedPartidoDomingo();
  updateCounter();
  renderVoteScreen();
}

function renderVoteScreen(){
  if(!voteMode) return;
  // Se completa en la Tarea 5 (pantalla "¿quién sos?").
}
```

- [ ] **Step 5: Branch the startup call between admin mode and vote mode**

Find (the very last line of the script):

```js
loadPlayers();
</script>
```

Replace with:

```js
if(voteMode){
  connectToGroup(voteCode);
} else {
  loadPlayers();
}
</script>
```

- [ ] **Step 6: Verify — normal mode is unaffected, vote mode toggles the right wrapper**

Server running. Navigate to `http://127.0.0.1:8123/index.html` (no query param) and run:

```js
JSON.stringify({
  appWrapVisible: document.getElementById('appWrap').style.display !== 'none',
  voteWrapVisible: document.getElementById('voteWrap').style.display !== 'none'
});
```

Expected: `{"appWrapVisible":true,"voteWrapVisible":false}`.

Then navigate to `http://127.0.0.1:8123/index.html?votar=qa-plan-2026` and run the same check:

```js
JSON.stringify({
  appWrapVisible: document.getElementById('appWrap').style.display !== 'none',
  voteWrapVisible: document.getElementById('voteWrap').style.display !== 'none'
});
```

Expected: `{"appWrapVisible":false,"voteWrapVisible":true}`.

Also check the console for errors:

Use `read_console_messages` with `onlyErrors: true` — expected: no errors (the `renderVoteScreen()` stub must not throw even though it does nothing yet).

- [ ] **Step 7: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Add vote-mode HTML scaffold and ?votar= detection

Introduces the standalone three-panel voting layout (hidden by
default) and the query-param switch that hides the admin app and
shows it instead, connecting straight to the given group code.
EOF
)"
```

---

### Task 5: "¿Quién sos?" step

**Files:**
- Modify: `index.html` — `renderVoteScreen()` (replace stub from Task 4)

**Interfaces:**
- Consumes: `voteMode`, `selectedVoterId` (Task 4), `players`, `votes` (Task 1).
- Produces: `showVoteStep(step)` — toggles which of the three vote panels (`'who'|'ballot'|'done'`) is visible. Later tasks (6, 7) depend on this exact function name and its three string values.

- [ ] **Step 1: Implement the "who are you" screen**

Find:

```js
function renderVoteScreen(){
  if(!voteMode) return;
  // Se completa en la Tarea 5 (pantalla "¿quién sos?").
}
```

Replace with:

```js
function renderVoteScreen(){
  if(!voteMode) return;
  const votedIds = new Set(votes.map(v=>v.voterId));
  const total = players.length;
  document.getElementById('voteCounter').innerHTML = `Ya votaron <b>${votedIds.size}</b> de <b>${total}</b>`;

  if(selectedVoterId){
    // Ya eligió quién es: no repintar la pantalla actual (ballot o done) por
    // actualizaciones en tiempo real de votos de otras personas.
    if(votedIds.has(selectedVoterId)) showVoteStep('done');
    return;
  }

  const whoList = document.getElementById('voteWhoList');
  whoList.innerHTML = '';
  players.forEach(p=>{
    const already = votedIds.has(p.id);
    const label = document.createElement('label');
    label.innerHTML = `<span>${escapeHtml(p.name)}${already ? ' (ya votó)' : ''}</span>`;
    if(already){
      label.classList.add('disabled');
    } else {
      label.addEventListener('click', ()=>{
        selectedVoterId = p.id;
        showBallotFor(p.id);
      });
    }
    whoList.appendChild(label);
  });
  showVoteStep('who');
}

function showVoteStep(step){
  document.getElementById('voteStepWho').style.display = step==='who' ? '' : 'none';
  document.getElementById('voteStepBallot').style.display = step==='ballot' ? '' : 'none';
  document.getElementById('voteStepDone').style.display = step==='done' ? '' : 'none';
}
```

Note: this references `showBallotFor`, which doesn't exist until Task 6. That's fine — it's a `function` declaration referenced only inside a click handler, and JS hoists declarations, but the function body itself isn't defined yet. Do not click a name in this task's verification; only inspect the DOM.

- [ ] **Step 2: Verify — roster renders correctly, already-voted names are disabled**

Server running. Navigate to `http://127.0.0.1:8123/index.html?votar=qa-plan-2026` and run:

```js
players.push({id:'qa1', name:'QA Uno', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa2', name:'QA Dos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
votes.push({voterId:'qa1', targetId:'qa2', score:7.0});
renderAll();
const html = document.getElementById('voteWhoList').innerHTML;
JSON.stringify({
  counterText: document.getElementById('voteCounter').textContent,
  hasQa1Disabled: html.includes('QA Uno (ya votó)'),
  hasQa2Enabled: html.includes('QA Dos') && !html.includes('QA Dos (ya votó)')
});
```

Expected: `counterText` is `"Ya votaron 1 de 2"`, `hasQa1Disabled` is `true`, `hasQa2Enabled` is `true`.

Clean up:

```js
players = players.filter(p=>!['qa1','qa2'].includes(p.id));
votes = [];
savePlayers();
```

- [ ] **Step 3: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Implement the vote screen's "who are you" step

Renders the player-picker with already-voted names disabled, plus
the showVoteStep() helper that later steps (ballot, done) use to
switch panels.
EOF
)"
```

---

### Task 6: Ballot step (rate your teammates)

**Files:**
- Modify: `index.html` — add `showBallotFor()` next to `showVoteStep()`

**Interfaces:**
- Consumes: `showVoteStep()` (Task 5), `players`, `escapeHtml()`.
- Produces: `showBallotFor(voterId)` — renders one slider row per other player into `#voteBallotList`, each `<input type=range>` carrying `data-target` (player id) and `data-touched` (`'true'`/`'false'`), and switches to the ballot step. Task 7 reads these `data-*` attributes on submit.

- [ ] **Step 1: Implement `showBallotFor()`**

Find:

```js
function showVoteStep(step){
  document.getElementById('voteStepWho').style.display = step==='who' ? '' : 'none';
  document.getElementById('voteStepBallot').style.display = step==='ballot' ? '' : 'none';
  document.getElementById('voteStepDone').style.display = step==='done' ? '' : 'none';
}
```

Replace with:

```js
function showVoteStep(step){
  document.getElementById('voteStepWho').style.display = step==='who' ? '' : 'none';
  document.getElementById('voteStepBallot').style.display = step==='ballot' ? '' : 'none';
  document.getElementById('voteStepDone').style.display = step==='done' ? '' : 'none';
}

function showBallotFor(voterId){
  const list = document.getElementById('voteBallotList');
  list.innerHTML = '';
  players.filter(p=>p.id!==voterId).forEach(p=>{
    const li = document.createElement('li');
    li.innerHTML = `
      <span class="pname">${escapeHtml(p.name)}</span>
      <div style="display:flex;align-items:center;gap:10px;">
        <input type="range" min="1" max="10" step="0.1" value="5.5" data-target="${p.id}" data-touched="false">
        <span class="rating-display">–</span>
      </div>
    `;
    const range = li.querySelector('input[type=range]');
    const display = li.querySelector('.rating-display');
    range.addEventListener('input', ()=>{
      range.dataset.touched = 'true';
      display.textContent = parseFloat(range.value).toFixed(1);
    });
    list.appendChild(li);
  });
  showVoteStep('ballot');
}
```

- [ ] **Step 2: Verify — picking a name shows every other player with an untouched slider**

Server running. Navigate to `http://127.0.0.1:8123/index.html?votar=qa-plan-2026` and run:

```js
players.push({id:'qa1', name:'QA Uno', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa2', name:'QA Dos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa3', name:'QA Tres', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
renderAll();
// simular click en "QA Uno" en el paso "¿quién sos?"
document.querySelectorAll('#voteWhoList label')[0].click();
const rows = document.querySelectorAll('#voteBallotList input[type=range]');
JSON.stringify({
  ballotVisible: document.getElementById('voteStepBallot').style.display !== 'none',
  rowCount: rows.length,
  allUntouched: Array.from(rows).every(r=>r.dataset.touched==='false'),
  targets: Array.from(rows).map(r=>r.dataset.target).sort()
});
```

Expected: `ballotVisible: true`, `rowCount: 2` (everyone except QA Uno), `allUntouched: true`, `targets: ["qa2","qa3"]`.

Then simulate touching one slider:

```js
const range = document.querySelector('#voteBallotList input[data-target="qa2"]');
range.value = '9.0';
range.dispatchEvent(new Event('input'));
JSON.stringify({
  touched: range.dataset.touched,
  displayText: range.nextElementSibling.textContent
});
```

Expected: `{"touched":"true","displayText":"9.0"}`.

Clean up:

```js
players = players.filter(p=>!['qa1','qa2','qa3'].includes(p.id));
selectedVoterId = null;
votes = [];
savePlayers();
renderAll();
```

- [ ] **Step 3: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Implement the vote screen's ballot step

Renders a 1.0-10.0 slider per teammate (excluding the voter), with
data-touched tracking so untouched sliders never produce a vote.
EOF
)"
```

---

### Task 7: Submit votes

**Files:**
- Modify: `index.html` — add the `voteSubmitBtn` click handler, right before the final startup branch

**Interfaces:**
- Consumes: `selectedVoterId` (Task 4/5), the `data-target`/`data-touched` sliders from `showBallotFor()` (Task 6), `votes` and `savePlayers()` (Task 1), `showVoteStep()` (Task 5).
- Produces: appends `{voterId, targetId, score}` entries to `votes` for every touched slider, persists via `savePlayers()`, shows the "done" step.

- [ ] **Step 1: Add the submit handler**

Find (the very end of the script):

```js
if(voteMode){
  connectToGroup(voteCode);
} else {
  loadPlayers();
}
</script>
```

Replace with:

```js
document.getElementById('voteSubmitBtn').addEventListener('click', ()=>{
  if(!selectedVoterId) return;
  const inputs = document.querySelectorAll('#voteBallotList input[type=range]');
  const newVotes = [];
  inputs.forEach(inp=>{
    if(inp.dataset.touched === 'true'){
      newVotes.push({
        voterId: selectedVoterId,
        targetId: inp.dataset.target,
        score: parseFloat(parseFloat(inp.value).toFixed(1))
      });
    }
  });
  votes = votes.concat(newVotes);
  savePlayers();
  showVoteStep('done');
});

if(voteMode){
  connectToGroup(voteCode);
} else {
  loadPlayers();
}
</script>
```

- [ ] **Step 2: Verify — submitting writes only touched votes and locks the voter out**

Server running. Navigate to `http://127.0.0.1:8123/index.html?votar=qa-plan-2026` and run:

```js
players.push({id:'qa1', name:'QA Uno', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa2', name:'QA Dos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'qa3', name:'QA Tres', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
renderAll();
document.querySelectorAll('#voteWhoList label')[0].click(); // elige "QA Uno"
const range = document.querySelector('#voteBallotList input[data-target="qa2"]');
range.value = '8.0';
range.dispatchEvent(new Event('input'));
// el slider de qa3 queda sin tocar a propósito
document.getElementById('voteSubmitBtn').click();
JSON.stringify({
  doneVisible: document.getElementById('voteStepDone').style.display !== 'none',
  votesForQa1AsVoter: votes.filter(v=>v.voterId==='qa1'),
  localStorageHasVote: JSON.parse(localStorage.getItem('armaequipos_votes')).some(v=>v.voterId==='qa1')
});
```

Expected: `doneVisible: true`; `votesForQa1AsVoter` has exactly one entry, `{"voterId":"qa1","targetId":"qa2","score":8}` (nothing for `qa3`, since that slider was never touched); `localStorageHasVote: true`.

Then verify the lock-out by resetting `selectedVoterId` and re-rendering (simulating a page reload for the same person):

```js
selectedVoterId = null;
renderAll();
const html = document.getElementById('voteWhoList').innerHTML;
JSON.stringify({ qa1ShowsAsVoted: html.includes('QA Uno (ya votó)') });
```

Expected: `{"qa1ShowsAsVoted":true}` — confirms a reload can't be used to vote again, since lock-out is derived from `votes` data (Firestore-backed), not local session state.

Clean up (including the real Firestore test document, so no QA residue is left in the live project):

```js
players = players.filter(p=>!['qa1','qa2','qa3'].includes(p.id));
selectedVoterId = null;
votes = [];
savePlayers();
await db.collection('armaequipos').doc('qa-plan-2026').delete();
renderAll();
```

- [ ] **Step 3: Commit**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "$(cat <<'EOF'
Wire up vote submission

Collects every touched ballot slider into the votes array on submit,
persists it, and shows the thank-you step. Skipped sliders produce
no vote, and a voterId that already voted stays locked out even
after a reload, since the lock-out is derived from stored vote data.
EOF
)"
```

---

### Task 8: End-to-end verification against the spec's test plan

**Files:** none (verification only; fix forward in `index.html` if something fails)

This task walks through every scenario in the spec's "Plan de pruebas" section as one continuous flow, using the real local server and a fresh Firestore test group code, to catch anything the per-task unit checks above might have missed in combination.

- [ ] **Step 1: Fresh vote recalculates OVR (spec test 1 & 2)**

Server running. Navigate to `http://127.0.0.1:8123/index.html` (admin mode, no query param) and run:

```js
players.push({id:'e2e1', name:'E2E Uno', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'e2e2', name:'E2E Dos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
savePlayers();
renderAll();
const ovrBefore = ovrFor(players.find(p=>p.id==='e2e1'));
votes.push({voterId:'e2e2', targetId:'e2e1', score:9.5});
savePlayers();
renderAll();
const ovrAfter = ovrFor(players.find(p=>p.id==='e2e1'));
JSON.stringify({ovrBefore, ovrAfter});
```

Expected: `ovrAfter` is noticeably higher than `ovrBefore` (rating moved from 5.0 to 9.5), and the admin roster panel (`#rosterList`) now shows "★ 9.5 (1 voto)" for E2E Uno — check with:

```js
document.getElementById('rosterList').innerHTML.includes('★ 9.5 (1 voto)');
```

Expected: `true`.

- [ ] **Step 2: Voter lock-out end-to-end through the real vote UI (spec test 3)**

Navigate to `http://127.0.0.1:8123/index.html?votar=e2e-plan-2026`. Wait for the Firestore snapshot to arrive (poll `players.length` a couple of times a second apart if needed, or just re-run after a 1-2s pause), then run:

```js
players.push({id:'e2e1', name:'E2E Uno', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
players.push({id:'e2e2', name:'E2E Dos', rating:5.0, elo: initialElo(5.0), playing:true, photo:null});
savePlayers();
renderAll();
document.querySelectorAll('#voteWhoList label')[0].click();
document.getElementById('voteSubmitBtn').click();
JSON.stringify({ done: document.getElementById('voteStepDone').style.display !== 'none' });
```

Expected: `{"done":true}`.

Now reload the same URL (`navigate` to `http://127.0.0.1:8123/index.html?votar=e2e-plan-2026` again) and, once players load, check:

```js
document.getElementById('voteWhoList').innerHTML.includes('(ya votó)');
```

Expected: `true` — the voter who already submitted stays locked out after a fresh page load.

- [ ] **Step 3: Skipped sliders never produce a vote (spec test 4)**

Still in the same session (or re-navigate to `?votar=e2e-plan-2026`), pick the *other* player and submit without touching any slider:

```js
selectedVoterId = null;
renderAll();
document.querySelectorAll('#voteWhoList label')[0].click(); // el único no bloqueado
const before = votes.length;
document.getElementById('voteSubmitBtn').click();
JSON.stringify({ votesAdded: votes.length - before });
```

Expected: `{"votesAdded":0}` — no sliders were touched, so nothing was appended.

- [ ] **Step 4: Admin mode is unaffected (spec test 5)**

Navigate to `http://127.0.0.1:8123/index.html` (no query param) and confirm normal functionality still works:

```js
JSON.stringify({
  appWrapVisible: document.getElementById('appWrap').style.display !== 'none',
  voteWrapVisible: document.getElementById('voteWrap').style.display !== 'none',
  rosterHasPlayers: document.getElementById('rosterList').children.length > 0
});
```

Expected: `{"appWrapVisible":true,"voteWrapVisible":false,"rosterHasPlayers":true}`.

Check the console for errors too (`read_console_messages`, `onlyErrors: true`) — expected: none.

- [ ] **Step 5: Clean up all QA data**

```js
players = players.filter(p=>!p.id.startsWith('e2e'));
votes = [];
selectedVoterId = null;
savePlayers();
await db.collection('armaequipos').doc('e2e-plan-2026').delete();
renderAll();
```

- [ ] **Step 6: If any check above failed, fix the code in `index.html` now and re-run the failing scenario until it passes, then commit the fix**

```bash
cd /e/Temp/claude/armaequipo_deploy && git add index.html && git commit -m "fix: address end-to-end verification findings"
```

(Skip this step entirely if everything passed on the first run — nothing to commit.)

---

### Task 9: Push and verify the live deploy

**Files:** none (deployment step)

- [ ] **Step 1: Review the full diff before publishing**

```bash
cd /e/Temp/claude/armaequipo_deploy && git log --oneline origin/main..HEAD && git diff origin/main..HEAD -- index.html
```

Confirm the diff only contains the changes from Tasks 1-8 (votes plumbing, rating calc, admin display, vote screen, submit handler, any e2e fixes).

- [ ] **Step 2: Ask the user for explicit confirmation before pushing**

This pushes to the public `tomasgima7/armaequipo` repo on `main`, which triggers Cloudflare's connected auto-deploy. Per this project's established workflow, confirm with the user before pushing — do not push automatically.

- [ ] **Step 3: Push**

```bash
cd /e/Temp/claude/armaequipo_deploy && git push origin main
```

- [ ] **Step 4: Wait for the Cloudflare Workers Build to finish and confirm success**

```bash
COMMIT=$(git rev-parse HEAD)
for i in $(seq 1 20); do
  RESULT=$(gh api repos/tomasgima7/armaequipo/commits/$COMMIT/check-runs --jq '.check_runs[] | select(.app.slug=="cloudflare-workers-and-pages") | .status + " " + (.conclusion // "null")')
  echo "attempt $i: $RESULT"
  if echo "$RESULT" | grep -q completed; then break; fi
  sleep 6
done
```

Expected: eventually prints `completed success`.

- [ ] **Step 5: Smoke-test the live site**

Navigate the Browser pane to `https://armaequipo.tomasgima7.workers.dev` and `https://armaequipo.tomasgima7.workers.dev/?votar=smoke-test-2026`, confirming (via `read_page` or `read_console_messages`) that the admin app loads normally on the first URL and the voting screen ("¿Quién sos?") loads on the second, with no console errors on either.

Clean up the smoke-test Firestore doc if it was created:

```js
await db.collection('armaequipos').doc('smoke-test-2026').delete();
```

- [ ] **Step 6: Report the live link to the user**

Tell the user the feature is live, and that the voting link to share with friends is `https://armaequipo.tomasgima7.workers.dev/?votar=<su-codigo-de-grupo>` — using whatever group code they already use to sync devices for that plantel.
