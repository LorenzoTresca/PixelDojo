# Piano di Sviluppo — Nuovi Giochi PixelDojo

## Indice
1. [Architettura condivisa (Firebase)](#architettura-condivisa)
2. [Nomi in Codice](#nomi-in-codice)
3. [Impostore](#impostore)
4. [Integrazioni con il sito esistente](#integrazioni)
5. [Ordine di sviluppo consigliato](#ordine-di-sviluppo)

---

## Architettura condivisa

Entrambi i giochi richiedono sincronizzazione real-time tra dispositivi diversi. La soluzione scelta è **Firebase Realtime Database** (piano Spark gratuito), accessibile via CDN senza build tools.

### Setup Firebase (una-tantum, condiviso tra entrambi i giochi)

1. Creare un progetto su [console.firebase.google.com](https://console.firebase.google.com)
2. Attivare **Realtime Database** (modalità test per lo sviluppo)
3. Copiare la `firebaseConfig` (apiKey, databaseURL, ecc.)
4. Inserire la config in un file condiviso o inline in ogni gioco

```js
// Inclusione via CDN (nessun npm necessario)
import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.x.x/firebase-app.js';
import { getDatabase, ref, set, get, update, onValue, push, remove }
  from 'https://www.gstatic.com/firebasejs/10.x.x/firebase-database.js';
```

### Pattern stanza (room) — uguale per entrambi i giochi

- **Codice stanza**: 6 caratteri alfanumerici maiuscoli generati casualmente (es. `XK7F2A`)
- **Host** crea la stanza, riceve il codice, condivide link/QR
- **Giocatori** aprono `games/nomefile.html?room=XK7F2A` dal proprio device
- La stanza si auto-elimina da Firebase dopo N ore di inattività (Firebase TTL rule)
- Ogni giocatore genera localmente un `playerId` (UUID random salvato in `sessionStorage`)

```
URL di join: pixeldojo.com/games/nomefile.html?room=XK7F2A
QR: punta allo stesso URL
```

---

## Nomi in Codice

### Concept
Adattamento del classico Codenames in italiano. Due squadre (Rosso e Blu) si sfidano su una griglia 5×5 di parole. Ogni squadra ha un **Suggeritore** che vede la mappa colori segreta e dà indizi in una parola + numero. Gli **Indovinatori** della squadra toccano le carte cercando di trovare le parole del proprio colore. Chi tocca l'Assassino (carta nera) perde immediatamente.

### Giocatori e ruoli

| Ruolo | Numero | Visibilità mappa colori |
|---|---|---|
| Suggeritore Rosso | 1 | ✅ Sì |
| Indovinatori Rosso | 1–4 | ❌ No |
| Suggeritore Blu | 1 | ✅ Sì |
| Indovinatori Blu | 1–4 | ❌ No |

Totale: **4–10 giocatori** consigliati (minimo 4, 2 per squadra).

### Struttura della griglia

- 25 carte (5×5) pescate casualmente da un vocabolario di 400+ parole italiane generiche
- Distribuzione colori: **9 Rosso + 8 Blu** (chi inizia ne ha una in più) **+ 7 Neutro + 1 Assassino**
- La squadra che inizia è estratta casualmente e ha 9 parole (una in più = svantaggio, perché inizia lei)

### Fasi di gioco

```
LOBBY → SETUP → [TURNO SUGGERITORE → TURNO INDOVINATORI] × N → FINE
```

| Fase | Descrizione |
|---|---|
| `lobby` | Giocatori si connettono, scelgono squadra e ruolo |
| `setup` | Host avvia la partita, griglia generata, mappa assegnata |
| `clue` | Il suggeritore della squadra attiva inserisce indizio + numero |
| `guessing` | Gli indovinatori toccano le carte entro il limite di tentativi |
| `end` | Fine partita — vittoria/sconfitta con reveal completo |

### Struttura dati Firebase

```
/rooms/{roomCode}/
  phase: 'lobby' | 'clue' | 'guessing' | 'end'
  host: playerId
  currentTeam: 'rosso' | 'blu'
  firstTeam: 'rosso' | 'blu'          // chi inizia (ha 9 parole)
  currentClue: { parola, numero } | null
  guessesLeft: number
  winner: null | 'rosso' | 'blu'
  lostBy: null | playerId             // chi ha toccato l'Assassino
  grid: string[25]                    // le 25 parole
  colorMap: string[25]                // solo visibile ai suggeritori
  revealed: boolean[25]               // quali carte sono state scoperte
  revealedColors: string[25]          // colore rivelato (null se non ancora scoperta)
  createdAt: timestamp

/rooms/{roomCode}/players/{playerId}/
  nickname: string
  team: 'rosso' | 'blu'
  role: 'suggeritore' | 'indovinatore'
  online: boolean
  lastSeen: timestamp
```

### Schermate UI (viste per dispositivo)

**1. Home / Creazione stanza**
- Pulsante "Crea stanza" → genera codice, host entra in lobby
- Campo "Entra con codice" → join stanza esistente
- (Alternativa: QR code generato lato client con libreria `qrcode.js`)

**2. Lobby**
- Lista giocatori connessi con avatar colorato
- Pulsanti "Squadra Rosso / Blu" + "Ruolo Suggeritore / Indovinatore"
- Indicatore "Pronto" per ogni giocatore
- Solo l'host vede il pulsante "Inizia partita" (attivato quando squadre bilanciate)

**3. Vista Suggeritore** *(solo chi ha ruolo = suggeritore)*
- Griglia 5×5 con colori visibili (rosso/blu/grigio/nero)
- Contatore parole rimaste per ogni squadra
- Form: campo testo "Indizio" + selettore numero (1–8 + ∞)
- Pulsante "Dai indizio" (attivo solo nel proprio turno)

**4. Vista Indovinatori** *(tutti gli altri)*
- Griglia 5×5 parole — nessun colore, solo le carte già scoperte mostrano il colore
- Riquadro indizio corrente (parola + numero)
- Tocco/click su carta = proposta di selezione → conferma collettiva o voto
- Pulsante "Passa turno" (rinuncia ai tentativi rimasti)

**5. Schermata fine partita**
- Reveal animato di tutte le carte rimaste
- Punteggio e messaggio vittoria/sconfitta
- Pulsanti "Rivincita" (stessa stanza) / "Nuova partita"

### Contenuto — Vocabolario italiano

Lista di ~400 parole comuni italiane suddivise in macro-categorie per garantire varietà:

- **Oggetti** (libro, sedia, finestra, chiave, specchio…)
- **Animali** (volpe, aquila, serpente, orso, balena…)
- **Luoghi** (castello, porto, deserto, ponte, foresta…)
- **Persone/Professioni** (capitano, dottore, spia, soldato, re…)
- **Natura** (neve, fulmine, radice, marea, vulcano…)
- **Concetti** (ombra, silenzio, tempo, confine, memoria…)

Tutte hardcoded nell'HTML come array JS — nessuna richiesta di rete per le parole.

### Sfide tecniche da considerare

- **Visibilità selettiva della mappa**: il `colorMap` su Firebase deve essere leggibile solo dai suggeritori. Soluzione: regole di sicurezza Firebase per ruolo, oppure criptare lato client con una chiave derivata dal ruolo.
- **Sincronizzazione tocco carta**: quando un indovinatore tocca una carta, tutti devono vederla. Usare `update()` atomico su Firebase per `revealed[i]` e `revealedColors[i]`.
- **Gestione disconnessioni**: usare `onDisconnect()` di Firebase per segnare il giocatore come offline.
- **QR code**: libreria `qrcode.min.js` (CDN, ~40KB) per generare il QR in-browser.

### File da creare

```
games/
  codenames.html     # ~2000 righe — tutto inline (CSS + HTML + JS)
```

### Fasi di sviluppo

| # | Task | Complessità |
|---|---|---|
| 1 | Setup Firebase + sistema stanze (crea/join/abbandona) | Media |
| 2 | Lobby: lista giocatori, scelta squadra/ruolo, ready check | Media |
| 3 | Generazione griglia (vocabolario + distribuzione colori) | Bassa |
| 4 | Vista Suggeritore: griglia colorata + form indizio | Media |
| 5 | Vista Indovinatori: griglia interattiva + logica turni | Alta |
| 6 | Logica vittoria/sconfitta (assassino, tutte le carte) | Media |
| 7 | Schermata fine partita + rivincita | Bassa |
| 8 | UI/UX design con token PixelDojo, responsive, tema | Media |
| 9 | QR code per link di join | Bassa |
| 10 | Integrazione `Scores` (punteggio su classifica) | Bassa |

**Stima totale**: ~4–6 sessioni di sviluppo.

---

## Impostore

### Concept
Gioco sociale di deduzione. Ogni round viene scelto un **tema segreto** (es. *"Personaggi di Harry Potter"*). Tutti i giocatori ricevono sul proprio device una lista di 5 affermazioni vere su quel tema. **Un solo giocatore — l'Impostore** — riceve la lista con **una affermazione falsa/infiltrata** al posto di una vera. I giocatori si confrontano a voce, discutono e votano chi pensano sia l'Impostore. L'Impostore vince se non viene trovato.

### Giocatori

- **Minimo**: 3 giocatori
- **Massimo**: 10 giocatori
- **Tutti su device separati** (ogni giocatore vede la propria lista privata)
- Nessun ruolo assegnato visivamente — il ruolo di Impostore è segreto

### Esempio di round

> **Tema**: Marchi di automobili italiane
>
> **Giocatori normali ricevono** (5 affermazioni vere):
> - Ferrari è fondata a Maranello
> - Lamborghini ha il logo del toro
> - Alfa Romeo ha origini milanesi
> - Maserati usa il simbolo del tridente
> - Lancia ha prodotto la Stratos
>
> **L'Impostore riceve** (4 vere + 1 falsa infiltrata, evidenziata solo a lui):
> - Ferrari è fondata a Maranello
> - Lamborghini ha il logo del toro
> - ~~Alfa Romeo ha origini milanesi~~ → **"Alfa Romeo è nata a Torino"** *(falsa — è nata a Milano)*
> - Maserati usa il simbolo del tridente
> - Lancia ha prodotto la Stratos

### Fasi di gioco

```
LOBBY → DISTRIBUZIONE → LETTURA → DISCUSSIONE → VOTO → REVEAL → [prossimo round]
```

| Fase | Durata | Descrizione |
|---|---|---|
| `lobby` | — | Join, nickname, attesa |
| `deal` | ~5s | Distribuzione affermazioni via Firebase |
| `reading` | 30–60s | Ogni giocatore legge privatamente la propria lista |
| `discussion` | libero | I giocatori discutono a voce (nessuna meccanica automatica) |
| `voting` | 60s | Ognuno vota un giocatore (non se stesso) |
| `reveal` | — | Si rivela l'Impostore e l'affermazione falsa |
| `score` | — | Punti assegnati, opzione round successivo |

### Sistema di punteggio

| Evento | Punti |
|---|---|
| Giocatori normali trovano l'Impostore | +2 a testa |
| Impostore non viene trovato (o voto diviso) | +4 all'Impostore |
| Voto unanime sull'Impostore | +3 a testa ai normali |
| L'Impostore vota se stesso 🤦 | −1 |

### Struttura dati Firebase

```
/rooms/{roomCode}/
  phase: 'lobby' | 'deal' | 'reading' | 'discussion' | 'voting' | 'reveal' | 'score'
  host: playerId
  round: number
  topicId: string                      // id del tema corrente
  topicLabel: string                   // "Marchi di automobili italiane"
  impostoreId: playerId                // segreto — lato client non mostrato agli altri
  readyPlayers: string[]               // chi ha finito la fase di lettura
  votes: { [voterId]: targetPlayerId } // visibili solo nella fase reveal
  scores: { [playerId]: number }
  createdAt: timestamp

/rooms/{roomCode}/players/{playerId}/
  nickname: string
  online: boolean
  lastSeen: timestamp

/rooms/{roomCode}/affermazioni/{playerId}/
  list: string[5]                      // le 5 affermazioni (una è falsa per l'impostore)
  falsaIdx: number | null              // indice della falsa (null per i giocatori normali)
```

> **Nota sicurezza**: `affermazioni/{playerId}` deve essere leggibile solo dal giocatore stesso (Firebase rules: `auth.uid === playerId` oppure usare regole per `sessionId`). `impostoreId` e `falsaIdx` non vengono mai inviati ai giocatori normali.

### Schermate UI

**1. Home / Join**
- Stessa logica di Codenames: crea stanza o inserisci codice

**2. Lobby**
- Lista giocatori connessi (nickname + avatar)
- Solo host vede "Inizia round" (attivo con ≥3 giocatori online)
- Punteggi cumulativi dei round precedenti visibili

**3. Fase lettura** *(ogni giocatore vede solo la propria lista)*
- Le 5 affermazioni mostrate a schermata intera
- L'Impostore vede l'affermazione falsa evidenziata in rosso con etichetta "INFILTRATA"
- Timer + pulsante "Ho letto" → entra in `readyPlayers`
- Quando tutti pronti → fase discussione

**4. Fase discussione**
- Schermata neutra con timer (regolabile dall'host, default 3 minuti)
- Lista giocatori visibile
- Nessuna meccanica automatica — la discussione avviene a voce
- Pulsante "Vai al voto" (solo host, attivabile prima della fine del timer)

**5. Fase voto**
- Lista di tutti i giocatori (tranne se stessi)
- Tocco = voto (un voto a testa, modificabile prima della conferma)
- Timer 60s, poi voto automatico casuale per chi non ha votato
- I voti sono nascosti finché non tutti hanno votato

**6. Reveal**
- Animazione drammatica di reveal: "L'Impostore era… 🎭 *[nickname]*"
- Mostra l'affermazione falsa e quella corretta
- Mostra tutti i voti (chi ha votato chi)
- Punteggi del round + totale cumulativo

### Contenuto — Database temi

Struttura del database hardcoded (JSON nell'HTML):

```js
const TEMI = [
  {
    id: 'auto_italiane',
    label: 'Marchi di automobili italiane',
    affermazioni: [
      'Ferrari è fondata a Maranello',
      'Lamborghini ha il logo del toro',
      // ... 12-15 affermazioni vere
    ],
    false: [
      'Alfa Romeo è nata a Torino',   // falsa (è nata a Milano)
      'Ferrari produce solo auto da corsa', // falsa
      // ... 4-5 false plausibili
    ]
  },
  // 40-60 temi totali
]
```

**Categorie di temi da sviluppare:**
- Anime & Manga (es. "Personaggi di One Piece")
- Sport (es. "Record di Usain Bolt")
- Geografia (es. "Capitali europee")
- Cinema (es. "Film Marvel")
- Videogiochi (es. "Giochi Nintendo")
- Cucina italiana
- Storia italiana
- Scienza & Natura
- Musica pop
- Serie TV

### Sfide tecniche da considerare

- **Privacy delle affermazioni**: ogni giocatore deve vedere solo la propria lista. Su Firebase senza autenticazione si usano regole basate su `playerId` (cifrato o come segreto condiviso al momento del join).
- **Reveal sicuro dell'impostore**: `impostoreId` non deve mai raggiungere i client prima del reveal. Opzione: l'host (o una Cloud Function) gestisce il reveal e scrive il valore solo quando la fase diventa `reveal`.
- **Timer sincronizzato**: il timer deve essere uguale per tutti. Soluzione: salvare `timerStartedAt` su Firebase e calcolare il tempo rimanente lato client come `timerStartedAt + durata - Date.now()`.
- **Voto simultaneo**: tutti votano in parallelo, i risultati si nascondono finché non hanno votato tutti (o scade il timer). Usare transaction Firebase per il conteggio.

### File da creare

```
games/
  impostore.html     # ~1500 righe — tutto inline (CSS + HTML + JS)
```

### Fasi di sviluppo

| # | Task | Complessità |
|---|---|---|
| 1 | Setup Firebase + sistema stanze (crea/join) | Media |
| 2 | Lobby: join con nickname, lista giocatori online | Bassa |
| 3 | Database temi: struttura JSON + 20 temi iniziali | Media |
| 4 | Logica distribuzione: scegli impostore, assegna affermazioni | Media |
| 5 | Schermata lettura privata (affermazioni per device) | Media |
| 6 | Fase discussione: timer sincronizzato, controllo host | Bassa |
| 7 | Sistema di voto: invio, attesa, reveal coordinato | Alta |
| 8 | Schermata reveal animata + reveal impostore | Media |
| 9 | Sistema punteggi multi-round + classifica in-game | Media |
| 10 | UI/UX design con token PixelDojo, responsive, tema | Media |
| 11 | Espansione database temi (da 20 a 50+) | Bassa |
| 12 | Integrazione `Scores` con classifica globale sito | Bassa |

**Stima totale**: ~4–5 sessioni di sviluppo.

---

## Integrazioni con il sito esistente

Entrambi i giochi vanno aggiunti a `giochi.html` e `index.html` con le rispettive card.

### Schede giochi da aggiungere

**Nomi in Codice** (in `giochi.html` e nella sezione giochi di `index.html`):
```
Emoji/Thumb: 🕵️
Titolo: Nomi in Codice
Tag: Multiplayer, 4-10 giocatori, Squadre
Descrizione: Due squadre si sfidano su 25 parole nascoste.
  Indovina le parole della tua squadra prima di toccare l'Assassino.
```

**Impostore** (in `giochi.html` e nella sezione giochi di `index.html`):
```
Emoji/Thumb: 🎭
Titolo: Impostore
Tag: Multiplayer, 3-10 giocatori, Deduzione
Descrizione: Tutti hanno lo stesso tema… tranne uno.
  Trova chi sta mentendo prima che sia troppo tardi.
```

### `Scores` system

Entrambi i giochi al termine di ogni partita/round chiameranno:
```js
Scores.submit('codenames', { player: nickname, score: punti, squadra: 'rosso'|'blu' });
Scores.submit('impostore',  { player: nickname, score: punti, rounds: n });
```

Aggiungere i due `gameId` all'oggetto `GAMES` in `classifica.html`:
```js
const GAMES = {
  // ... esistenti
  codenames: { label: 'Nomi in Codice', emoji: '🕵️', color: '#6366f1' },
  impostore:  { label: 'Impostore',     emoji: '🎭', color: '#ef4444' },
};
```

### `CLAUDE.md` — aggiornamenti necessari

Quando i giochi vengono sviluppati, aggiornare `CLAUDE.md` con:
- La `firebaseConfig` (solo variabili pubbliche, non secret)
- La struttura Firebase di ciascun gioco
- Le regole di sicurezza Firebase
- Le dipendenze CDN aggiunte (Firebase SDK, qrcode.js)

---

## Ordine di sviluppo consigliato

Sviluppare **Impostore prima di Nomi in Codice** per i seguenti motivi:

1. **Meno complesso** — nessuna griglia interattiva, logica turni più semplice
2. **Ottimo banco di prova** per il sistema Firebase stanze (che poi si riutilizza in Codenames)
3. **Contenuto più veloce** da produrre (temi testuali vs vocabolario da 400 parole)
4. **Alta coinvolgibilità** — il gioco virale porta utenti nuovi, generando traffico per gli altri giochi

**Roadmap complessiva:**

```
Sprint 1: Setup Firebase condiviso + sistema stanze base
Sprint 2: Impostore — lobby, distribuzione, lettura
Sprint 3: Impostore — voto, reveal, punteggi, UI finale
Sprint 4: Nomi in Codice — lobby, griglia, vista suggeritore
Sprint 5: Nomi in Codice — logica turni, indovinatori, fine partita
Sprint 6: Nomi in Codice — UI finale, QR code, polish
Sprint 7: Integrazioni (giochi.html, classifica, Scores), aggiornamento CLAUDE.md
```
