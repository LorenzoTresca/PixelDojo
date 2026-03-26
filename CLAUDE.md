# PixelDojo — Guida al Codebase

Sito multi-pagina per giochi in-browser, scritto interamente in HTML/CSS/JS vanilla (nessun framework, nessun bundler). Ogni pagina è un file HTML self-contained con CSS e JS inline.

---

## Struttura delle cartelle

```
PixelDojo/
├── index.html          # Home page — vetrina dei giochi
├── giochi.html         # Catalogo dei giochi (elenco schede)
├── classifica.html     # Classifica globale (leaderboard)
├── login.html          # Login / Registrazione
└── games/
    ├── trivial.html    # Gioco Trivial Otaku (~2700 righe)
    ├── quiz.html       # Gioco Quiz Flash
    └── impostore.html  # Gioco Impostore (multiplayer locale/remoto via BroadcastChannel)
```

---

## Design System condiviso

Ogni pagina replica il design system all'inizio del `<style>`. I token CSS sono definiti in `:root` e sovrascritti dalle varianti tema.

### Token globali (`:root`)
| Token | Uso |
|---|---|
| `--ink` / `--ink2` / `--ink3` | Testo primario / secondario / terziario |
| `--surf` / `--surf2` / `--surf3` | Sfondo card / pagina / accento |
| `--neon` / `--neon2` | Viola primario `#7c3aed` / scuro |
| `--volt` / `--volt2` | Verde-lime `#c8f135` / scuro |
| `--fire` / `--ice` / `--gold` / `--mint` | Colori accento |
| `--border` | Bordi e divisori |
| `--r` / `--r2` / `--r3` | Border radius 10/16/24px |
| `--shadow` / `--shadow-neon` | Ombre box |

### Sistema tema (light/dark)

**Pagine principali** (`index.html`, `giochi.html`, `classifica.html`, `trivial.html`) usano il sistema manuale con `data-theme`:

```css
body[data-theme="dark"]  { --ink: #e8eaf2; --surf: #1c2030; ... }
body[data-theme="light"] { --ink: #1a1f2e; --surf: #ffffff; ... }
```

Il tema viene salvato/letto da `localStorage` con chiave `pd_theme` ed è **condiviso tra tutte le pagine**.

**Pagine secondarie** (`login.html`, `quiz.html`) usano ancora `@media (prefers-color-scheme: dark)` — **non hanno il toggle manuale**, sono da aggiornare al sistema `data-theme` se serve coerenza.

`impostore.html` usa il sistema `data-theme` ma con una variante semplificata (nessun `initTheme()` separato — l'init è in una IIFE in fondo al file).

### Funzioni tema (presenti in ogni pagina che ha il toggle)

```js
function initTheme() {
  const saved = localStorage.getItem('pd_theme') || 'dark';
  document.body.dataset.theme = saved;
  const btn = document.getElementById('themeToggleBtn');
  if (btn) btn.textContent = saved === 'dark' ? '🌙' : '☀️';
  // 🌙 = tema scuro attivo; ☀️ = tema chiaro attivo
}

function toggleTheme() {
  const next = document.body.dataset.theme === 'dark' ? 'light' : 'dark';
  document.body.dataset.theme = next;
  localStorage.setItem('pd_theme', next);
  // ...aggiornamenti specifici per pagina...
}
```

Il toggle va chiamato in `DOMContentLoaded`: `window.addEventListener('DOMContentLoaded', () => { initTheme(); ... })`.

> **Convenzione emoji toggle:** l'emoji rappresenta il tema **correntemente attivo**, non quello di destinazione. Quindi: `saved === 'dark' ? '🌙' : '☀️'`. In `impostore.html` il pattern è identico ma usa `id="themeBtn"` invece di `id="themeToggleBtn"`.

---

## localStorage — chiavi usate

| Chiave | Tipo | Contenuto |
|---|---|---|
| `pd_theme` | `string` | `'dark'` o `'light'` |
| `pd_users` | JSON object | Database utenti `{ email: { nickname, passwordHash, xp, level } }` |
| `pd_session` | JSON object | Utente loggato corrente |
| `pd_scores` | JSON object | Punteggi per gioco `{ gameId: [ {player, score, date} ] }` |
| `pd_imp_{CODE}` | JSON object | Stato completo della stanza Impostore (scritto dall'host, letto dai guest) |

---

## Sistema Auth (`login.html`)

Autenticazione client-side simulata (solo localStorage, nessun backend).

```js
Auth.register(email, password, nickname)  // → { ok, error }
Auth.login(email, password)               // → { ok, error }
Auth.logout()
Auth.currentUser()                        // → { email, nickname, xp, level } | null
Auth.isLoggedIn()                         // → boolean
Auth.addXP(amount)                        // aggiunge XP e aggiorna localStorage
Auth.requireLogin()                       // redirect a login.html se non loggato
```

XP e livelli sono calcolati con una curva esponenziale. La navbar mostra l'avatar utente se loggato, altrimenti il pulsante login.

---

## Classifica (`Scores`)

Modulo disponibile in tutte le pagine che lo includono:

```js
Scores.submit(gameId, entry)              // { player, score, date, ... }
Scores.getBoard(gameId)                   // array top-10
Scores.getAllBoards()
Scores.renderBoard(gameId, container)     // inietta HTML classifica nel container
```

---

## `trivial.html` — Trivial Otaku

Il file più complesso (~2700 righe). Tutto in un singolo HTML: CSS game-specific, HTML struttura, JS game logic.

### Categorie

```js
const CATS = {
  anime:  { label:"Anime",        emoji:"⛩️", color:"#f87171" },
  sport:  { label:"Sport",        emoji:"🏅", color:"#34d399" },
  giochi: { label:"Videogiochi",  emoji:"🎮", color:"#a78bfa" },
  musica: { label:"Musica",       emoji:"🎵", color:"#fbbf24" },
  series: { label:"Serie TV",     emoji:"📺", color:"#60a5fa" },
  pop:    { label:"Pop Culture",  emoji:"⭐", color:"#f472b6" },
};
const CAT_IDS = Object.keys(CATS); // ['anime','sport','giochi','musica','series','pop']
```

### Geometria del tabellone (SVG `viewBox="0 0 600 600"`)

```js
const CX = 300, CY = 300   // centro
const RING_R = 230          // raggio anello esterno
const SPOKE_STEPS = 4       // nodi per raggio (esclusi C e HQ)
const RING_N = 36           // nodi totali sull'anello
const HQ_IDXS = [0, 6, 12, 18, 24, 30]  // posizioni QG sull'anello
```

**Tipi di posizione:**
- `'C'` — centro
- `'R{n}'` — nodo anello (n=0..35)
- `'{PREFIX}{s}'` — nodo raggio (PREFIX ∈ `AN SP GV MU SR PK`, s=0..3)

**Mapping PREFIX → categoria:** `AN=anime, SP=sport, GV=giochi, MU=musica, SR=series, PK=pop`

### Stato di gioco (`G`)

```js
let G = {
  phase: 'idle',       // 'setup'|'idle'|'moving'|'dir'|'spoke'|'question'|'streak'|'win'
  players: [],         // array di player objects
  curIdx: 0,           // indice giocatore corrente
  stepsLeft: 0,        // passi rimasti dopo il lancio del dado
  diceVal: 0,          // ultimo valore del dado
  pendingDir: null,    // 'cw'|'ccw'|'center' — direzione scelta sulla direzione ring
  usedQ: {},           // { playerIdx: [questionIds usate] }
  turnCount: 0,
  streakQueue: [],     // coda domande per la streak finale
  streakCount: 0,
  qTimer: null,
}
```

**Player object:**
```js
{
  name: string,
  color: string,              // da PLAYER_COLORS
  position: string,           // 'C' | 'R{n}' | '{PREFIX}{s}'
  wedges: { anime:false, sport:false, ... },  // spicchi guadagnati
  onSpoke: null | { prefix, idx },
}
```

### Fasi di gioco (`G.phase`)

| Fase | Descrizione |
|---|---|
| `idle` | In attesa del lancio dado — mostra overlay dado |
| `moving` | Animazione movimento pedina |
| `dir` | Scelta direzione (orario/antiorario/centro) sul ring |
| `spoke` | Scelta raggio (quale QG raggiungere) |
| `question` | Domanda in corso |
| `streak` | Sfida finale al centro (serie di domande) |
| `win` | Partita vinta |

### Layout CSS del gioco

```css
.game-layout { display:grid; grid-template-columns:185px 1fr 210px; height:calc(100vh - 48px); }
/* tablet: */ grid-template-columns:140px 1fr 165px;
/* mobile: */ display:flex; flex-direction:column;
```

**Breakpoint:** 960px (tablet), 640px (mobile).

Il tabellone SVG è sempre quadrato:
```css
#boardSvg {
  aspect-ratio: 1 / 1;
  height: 100%;
  width: auto;
  max-width: 100%;
  max-height: 100%;
}
```

Lo sfondo del `board-area` usa un colore solido che coincide con il bordo esterno del gradiente SVG (dark: `#0a0812`, light: `#d4cef0`) — così non ci sono barre visibili attorno al tabellone.

### Funzioni chiave di `trivial.html`

| Funzione | Descrizione |
|---|---|
| `startGame()` | Inizializza `G`, chiama `drawBoard()`, `setPhase('idle')` |
| `drawBoard()` | Disegna tutto il tabellone SVG (è theme-aware) |
| `renderTokens()` | Aggiorna le pedine SVG sul tabellone |
| `setPhase(ph)` | Cambia `G.phase`, aggiorna UI, mostra/nasconde overlay dado |
| `rollDice()` | Anima il dado, imposta `G.stepsLeft`, chiama `processMove()` |
| `processMove()` | Router principale del movimento (decide se muovere ring/raggio/centro) |
| `moveOnRing()` | Sposta la pedina di 1 passo sull'anello |
| `moveSpokeForward()` | Avanza di 1 passo verso HQ sul raggio |
| `moveSpokeBackward()` | Arretra di 1 passo verso il centro sul raggio |
| `showDirModal()` | Mostra scelta direzione inline nell'`actionPanel` con preview SVG |
| `showSpokeChoice()` | Mostra scelta raggio quando si è su un HQ |
| `enterSpoke(prefix, cat)` | Entra su un raggio specifico |
| `landOn(pos)` | Gestisce l'atterraggio su una casella (domanda, QG, centro…) |
| `askQuestion(catId, isBonus, isHq)` | Mostra la modale con la domanda |
| `answerQ(chosen, correct, isHq, isBonus, catId)` | Gestisce la risposta |
| `startStreak()` | Avvia la sfida finale al centro |
| `endTurn()` | Passa al giocatore successivo |
| `renderPlayers()` | Aggiorna il pannello giocatori e le pedine |
| `toggleTheme()` | Cambia tema + ridisegna il tabellone |

### Regola centro / "Verso il Centro"

Il pulsante "Verso il Centro" (`chooseDir('center')`) appare **solo quando il giocatore ha tutti e 6 gli spicchi** ed è su un HQ. Una volta scelto, `G.pendingDir = 'center'` e `processMove()` forza sempre `moveSpokeBackward()` finché non si arriva al centro:

```js
const hasAllWedges = CAT_IDS.every(c => player.wedges[c]);
if (hasAllWedges || G.pendingDir === 'center') {
  G.pendingDir = 'center'; // persiste tra i turni
  moveSpokeBackward();
}
```

### Dice SVG — sistema a doppio target

Il dado viene renderizzato in due posti contemporaneamente: sidebar destra e overlay centrale.

```js
_buildDiceBase(svg, idSuffix)              // crea gradiente+ombra (theme-aware, IDs univoci via suffix)
_renderDiceInto(svg, value, suffix)        // disegna i punti sul dado
_renderPlaceholderInto(svg, suffix)        // disegna il placeholder "?"

// Shortcut per la sidebar:
renderDice(value)           // usa suffix 's'
renderDicePlaceholder()     // usa suffix 'sp'

// Per l'overlay:
_renderDiceInto(document.getElementById('diceOverlaySvg'), G.diceVal, 'or')
```

Gli `id` dei gradienti SVG usano il suffix per evitare collisioni DOM (es. `grad_s`, `grad_or`).

### Overlay dado (`#diceOverlay`)

Mostrato quando `G.phase === 'idle'`, nascosto dopo il lancio. Il click/tap su `#diceOverlayArea` chiama `handleDiceClick()` → `rollDice()`.

---

## Pattern comuni

### Aggiungere il tema toggle a una nuova pagina

1. Aggiungere CSS: `body[data-theme="dark"] { ... }` e `body[data-theme="light"] { ... }` con gli stessi token di `index.html`
2. Aggiungere `.theme-toggle` CSS (pulsante emoji rotondo)
3. Aggiungere il `<button class="theme-toggle" id="themeToggleBtn" onclick="toggleTheme()">` nella navbar
4. Aggiungere le funzioni `initTheme()` e `toggleTheme()` nel JS
5. Chiamare `initTheme()` in `DOMContentLoaded`

### Aggiungere un nuovo gioco

1. Creare `games/nomgioco.html` (auto-contained)
2. Aggiungere una scheda in `giochi.html`
3. Registrare il gameId in `Scores.GAMES` dentro `classifica.html` e nelle pagine che usano `Scores`

---

## `impostore.html` — Gioco Impostore

Gioco multiplayer in-browser (locale o remoto su stessa rete/browser). Architettura completamente client-side: un giocatore fa da **host** e mantiene lo stato autorevole su `localStorage`; tutti gli altri sono **guest** e comunicano tramite `BroadcastChannel`.

### Trasporto (`BroadcastChannel`)

```js
// Canale: 'pd_imp_{roomCode}'
// HOST → tutti: { type:'STATE', from, payload: roomObj }
// GUEST → HOST:  { type:'ACTION', from, payload: actionObj }
// GUEST → HOST:  { type:'REQ_STATE', from, payload:null }   // richiesta stato iniziale
```

L'host scrive sempre con `hostWrite(room)` che aggiorna `G.room`, `localStorage` e invia `STATE` sul canale. I guest inviano azioni con `playerSend(action)` che, se è l'host stesso, chiama direttamente `applyAction()`.

### Stato globale client (`G`)

```js
let G = {
  playerId: string,       // ID univoco per sessione (sessionStorage 'pd_pid')
  roomCode: string,       // codice stanza 5 lettere
  isHost: boolean,
  nickname: string,
  room: roomObj | null,   // copia locale dell'ultimo stato ricevuto
  myVote: string | null,  // targetId del voto corrente
  voteConfirmed: boolean,
  readyConfirmed: boolean,
  guessConfirmed: boolean,
  _localRound: number,    // traccia il round per resettare lo stato locale
  _localVoteSeq: number,  // traccia la sequenza di voto (rivoto incluso)
}
```

### Stato della stanza (`room`)

```js
{
  phase: string,          // vedi fasi sotto
  round: number,          // numero del round corrente (incrementato da startRound)
  wordId: string,         // id della parola pescata da PAROLE[]
  readyPlayers: string[], // ids che hanno premuto "Sono pronto"
  votes: { [voterId]: targetId },
  voteSeq: number,        // incrementato ad ogni rivoto; usato per resettare myVote sui client
  tieIds: string[] | null,// candidati del rivoto (null = nessun pareggio in corso)
  scores: { [playerId]: number },
  revealData: object | null,
  guessData: object | null,
  settings: { impostori: number, infiltrati: number },
  usedWords: string[],
  turnOrder: string[],    // ordine casuale usato per assegnare i ruoli
  discOrder: string[],    // ordine casuale per la discussione (generato all'entrata in 'discussion')
  roles: { [playerId]: 'normale'|'impostore'|'infiltrato' },
  messages: Message[],    // chat della discussione, resettata ad ogni round
  players: { [id]: { id, nickname, isHost, online, color } },
}
```

### Fasi di gioco (`room.phase`)

| Fase | Descrizione |
|---|---|
| `'lobby'` | Sala d'attesa — host configura, guest si uniscono |
| `'word'` | Ogni giocatore vede la propria parola in privato e preme "Sono pronto" |
| `'discussion'` | Chat pubblica con ordine obbligato; l'host passa alle votazioni solo quando tutti hanno scritto |
| `'voting'` | Votazione chi è l'impostore; se pareggio → rivoto automatico tra i pareggianti |
| `'guess'` | Chi è stato scoperto prova a indovinare la parola per guadagnare punti bonus |
| `'reveal'` | Risultati del round con ruoli, voti, delta punti e risultato indovinello |
| `'end'` | Classifica finale |

### Azioni (`ACTION` dal guest all'host)

| `action.type` | Quando | Payload extra |
|---|---|---|
| `JOIN` | Guest entra in lobby | `nickname` |
| `MARK_READY` | Fase `word`, giocatore pronto | — |
| `VOTE` | Fase `voting` | `targetId` |
| `SUBMIT_GUESS` | Fase `guess` | `word` |
| `SEND_MESSAGE` | Fase `discussion` | `text` (max 200 car.) |
| `LEAVE` | Qualsiasi | — |

### Logica punteggio

**Votazione normale (nessun pareggio):**
- Voter che scopre un impostore: **+3 pt** ciascuno
- Voter che scopre un infiltrato: **+2 pt** ciascuno
- Bonus unanimità (tutti i normali votano lo stesso speciale): **+1 pt** per ogni normale
- Impostore sopravvive (non scoperto): **+4 pt**
- Infiltrato sopravvive: **+3 pt**

**Indovinello post-scoperta:**
- Impostore indovina la parola: **+2 pt** bonus; i suoi voter vengono ridotti da 3 → **1 pt** (−2)
- Infiltrato indovina la parola: **+1 pt** bonus; i suoi voter mantengono i **2 pt** originali

**Pareggio nelle votazioni:**
- Al primo pareggio: `doReveal` salva i candidati in `r.tieIds`, azzera `r.votes`, incrementa `r.voteSeq` e rimanda alla fase `'voting'` (rivoto solo tra i pareggianti)
- Se dopo il rivoto c'è ancora pareggio: nessuno viene trovato, gli speciali sopravvivono con i loro punti

### Discussione — regole di turno

- Quando tutti segnalano "pronto" (fase `word`), l'host genera `r.discOrder`: shuffle casuale di tutti i giocatori, specifico per quel round.
- L'ordine viene mostrato a tutti nella schermata discussione con indicatore ✅/⏳ per ogni giocatore.
- Il pulsante "Vai alle votazioni" dell'host è **disabilitato** finché ogni giocatore non ha inviato almeno un messaggio in chat. Il controllo è: `discOrder.every(id => spokeSet.has(id))` dove `spokeSet = new Set(messages.map(m => m.pid))`.

### Funzioni chiave di `impostore.html`

| Funzione | Descrizione |
|---|---|
| `createRoom()` | Crea stanza, sceglie codice, inizializza stato, chiama `hostWrite()` |
| `joinRoom()` | Entra in stanza esistente via codice, invia `JOIN` |
| `hostWrite(room)` | Salva stato su `G.room` + `localStorage` + BroadcastChannel |
| `playerSend(action)` | Invia azione all'host (o chiama direttamente `applyAction` se è host) |
| `applyAction(fromId, action)` | Router azioni lato host; modifica `r` e chiama `hostWrite` |
| `startRound(r)` | Pesca parola, assegna ruoli, resetta tutti i campi di round, incrementa `r.round` |
| `doReveal(r)` | Conta voti, rileva pareggi (→ rivoto) o scoperta, calcola delta, decide fase successiva |
| `finalizeGuesses(r)` | Valuta le risposte dell'indovinello, applica bonus/penalità e transita a `'reveal'` |
| `hostGoVoting()` | Transita a `'voting'` solo se tutti hanno scritto in chat |
| `resetLocalIfNewRound()` | Resetta `myVote`, `voteConfirmed`, `guessConfirmed` al cambio round; resetta anche il DOM del campo indovinello; gestisce anche il cambio `voteSeq` per i rivoti |
| `renderScreen()` | Dispatcher: chiama il render corretto in base a `G.room.phase` |
| `renderDiscussion()` | Mostra ordine turni + indicatori "ha parlato" + chat + barra progresso + stato bottone host |
| `renderVoting()` | Mostra griglia voti; in caso di rivoto mostra solo i candidati pareggianti con banner giallo |
| `renderGuess()` | Mostra form indovinello; gestisce stato DOM (enabled/disabled) in base a `G.guessConfirmed` |
| `toggleTheme()` | Cambia tema, salva in `localStorage`, aggiorna emoji pulsante (🌙 = dark, ☀️ = light) |

### Struttura `Message` (chat discussione)

```js
{
  pid: string,       // playerId del mittente
  nickname: string,
  color: string,     // colore hex del giocatore
  text: string,      // max 200 caratteri
}
```

I messaggi vengono azzerati (`r.messages = []`) all'inizio di ogni nuovo round in `startRound()`.

---

## Note tecniche importanti

- **Nessun build tool.** Aprire i file direttamente nel browser. Non serve `npm install` o compilazione.
- **Nessun framework JS.** Tutto è vanilla DOM API.
- **CSS e JS inline.** Ogni file HTML è completamente self-contained — nessun import esterno tranne Google Fonts.
- **SVG programmatico.** Il tabellone è costruito con `document.createElementNS` via `svgEl(tag, attrs)`.
- **Auth client-side.** Non c'è backend — `login.html` usa solo `localStorage`. Non adatto alla produzione senza un server reale.
- **Quiz/domande hardcoded.** Le domande di Trivial Otaku sono definite direttamente nel JS di `trivial.html` dentro `getTrivialQuestion()`.
- **Git attivo.** La cartella ha un repo `.git` con branch `main` e remote `origin`.
