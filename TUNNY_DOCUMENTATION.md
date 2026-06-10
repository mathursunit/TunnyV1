# Tunny — The Hyderabadi Card Game

**Comprehensive Technical & Gameplay Documentation**

Version: **v1.1.0**
Author: **Sunit Mathur**
Dedicated to: **The Narayan Mathur Family** — *Hyderabad · Then & Always*
Tagline: *ایک کھیل، ہزار یادیں · Ek khel, hazaar yaadein* ("One game, a thousand memories")
Live (single-player): **https://tunny.sunitmathur.com**

---

## 1. Overview

Tunny is a digital revival of a traditional Hyderabadi trick-taking card game, built as a single self-contained HTML file per version (HTML + CSS + vanilla JavaScript, no build step, no external framework). It is a 4-player, 2-versus-2 partnership game played with a stripped 24-card deck. The application ships in two forms that share an identical rules engine, AI, and UI:

| Version | File | Players | Hosting |
|---|---|---|---|
| **Single-player** | `index.html` | 1 human vs 3 AI bots | GitHub Pages (`tunny.sunitmathur.com`) |
| **Multiplayer** | `index-mp.html` | Up to 4 humans (AI fills empty seats) | Self-hosted Node `ws` server in Docker on the user's **OCI** instance, at `tunny2.sunitmathur.com` |

Both files are intentionally framework-free: a single `<style>` block, a single `<script>` block, and inline HTML. The card-game logic, AI module, rendering, and (in the multiplayer build) the networking layer all live in one file each.

The visual identity is explicitly Hyderabadi — the landing screen carries icons for the Charminar (🕌), Hyderabadi cuisine (🍖), roses (🌹), chai (🫖) and playing cards (🎴); card backs use a real cropped, dimmed photo of the Charminar (`charminar_back.webp`); and toast/flavor text is written in Hyderabadi Hindi-Urdu ("Khelo!", "Wah bhai wah!", "Masha'Allah!").

---

## 2. Game Rules (shared by both versions)

### 2.1 The deck

- **24 cards**: four suits (♥ hearts, ♦ diamonds, ♣ clubs, ♠ spades) × six ranks.
- **Ranks, high → low: `J, 9, A, 10, K, Q`** (Jack is the highest card, Queen the lowest). Internally `RANK_ORDER = { J:6, 9:5, A:4, 10:3, K:2, Q:1 }`.
- Hearts and diamonds render red; clubs and spades render black.

### 2.2 Card point values (Hyderabadi scoring)

Each card carries a fractional point value used to score rounds:

| Card | Points |
|---|---|
| J | 3.0 |
| 9 | 2.0 |
| A | 1.0 |
| 10 | 1.0 |
| K | 0.3 |
| Q | 0.2 |

Total points in the deck = 4 × (3 + 2 + 1 + 1 + 0.3 + 0.2) = **30.0**.
Round-win threshold for the counting team = **10.5** (≈35% of the deck).

### 2.3 Seating & teams

- Four seats, indexed `0..3`. The local human is always seat 0 in single-player (and is rotated to the bottom of the screen in multiplayer).
- **Partners sit opposite each other**: seats 0 ↔ 2 form Team 0; seats 1 ↔ 3 form Team 1 (`TEAMS = [[0,2],[1,3]]`). This was a fix in v1.1.0 (partners are visually placed across the table, not adjacent).
- Visual layout: bottom = you, left = an opponent, top = your partner, right = an opponent.

### 2.4 Round structure (state machine)

A round (called a "deal") flows through an explicit phase machine: `draw → deal4 → bid → trump → deal2 → hold → play → score`.

1. **Deal 4** — Each player is dealt 4 cards. A dealer is chosen (random on the first round, then rotated). The dealer badge (🃏) appears on the dealer's name tag, and bidding is announced to start with the player to the dealer's left.
2. **Bid** — Players bid based only on their first 4 cards (see §2.5).
3. **Trump** — The bid winner secretly chooses a trump suit from a suit they hold (see §2.6).
4. **Deal 2** — Each player receives 2 more cards (final hand = 6 cards).
5. **Hold** — A ~5-second window in which the human can declare a **Tunny** (see §2.9).
6. **Play** — Six tricks ("hands") are played out (see §2.7).
7. **Score** — Points are tallied, balls are awarded, and the game-over condition is checked (see §2.8).

### 2.5 Bidding

- Order is **clockwise starting from the player left of the dealer; the dealer bids last** (`order = [dealer+1, dealer+2, dealer+3, dealer]`).
- Bid values offered to the human: `10, 20, 30, 40, 50, 60, 70, 80, 90, 100, 104` — only values above the current high bid are shown. A player may also **Pass**.
- The highest bidder wins the right to choose trump.
- **If nobody bids, the dealer is forced to take the bid at 10** (the dealer's Pass button is disabled in that case).
- Winning the bid at a non-zero amount means the round is scored "with compensation" — the opposing (counting) team gets a points handicap proportional to the bid (`bidAmount × 0.1`), and a successful bid is worth more balls.

### 2.6 Trump selection

- The bid winner picks a trump suit **from a suit they actually hold** (the picker only shows suits present in their hand, annotated with the count and whether they hold the J or 9 of that suit).
- **Trump is kept secret** ("Shh — raaz rakho!"). It is **revealed automatically the first time any trump card is played to the table**, after which the trump indicator lights up in the score bar and trump cards in the human's hand are highlighted with a golden border.

### 2.7 Playing tricks (the six "hands")

Each of the 6 tricks proceeds clockwise from the current leader (the first leader is the player left of the dealer; thereafter the winner of each trick leads the next).

**Legal-move / follow-suit rules** (enforced in `getLegalMoves`):

- **Follow suit**: if you hold the led suit, you must play it. Playing off-suit when you could follow is illegal (a "colour cut," which the rules text describes as a 4-ball penalty).
- **Undercut (over-trump) rule**:
  - If the *led suit is trump* and a higher trump is already on the table, you must play a still-higher trump if you have one.
  - If you *cannot follow* the led suit and trump has already been played, you must play a higher trump than the highest on the table if you hold one.
- If you can neither follow nor are forced to over-trump, you may play any card.

**Trick resolution** (`determineTrickWinner` / `cardValue` / `beats`): a trump card beats any non-trump; among the same category the higher `RANK_ORDER` wins. `cardValue` = `100 + rank` for trump, `rank` for the led suit, `0` otherwise. The winning card's slot is highlighted, and the winning team accumulates the points of all four cards in the trick.

**Last-hand bonus**: the winner of the 6th (final) trick scores an extra **+1 point**.

### 2.8 Scoring, balls, and winning the game

- After all 6 tricks, the **counting team** (the team that did *not* win the bid) needs **≥ 10.5 points** (plus any bid compensation) to "win the round."
- **Ball awards**:
  - If the counting team reaches the threshold: they earn **2 balls** if a bid was placed (non-zero), otherwise **1 ball**.
  - If the counting team falls short: the **bidding team earns 1 ball**.
- The first team to **12 balls** (`BALLS_TO_WIN`) wins the game.
- **Corner house**: a team sitting at **11 balls** (one away from winning) is "on the corner" — its final ball slot pulses on the score bar. Certain calls (e.g. Double) are banned while on the corner.
- **Two to clear**: if **both** teams are one ball from winning (e.g. 11–11) the game does not end on a single ball; a team must achieve a **2-ball lead** to actually win.

### 2.9 Special calls

These high-risk, high-reward declarations are part of the traditional game and are implemented (some in simplified form):

- **Jodie** — Declared after winning trick 1 or trick 3 if you hold the **K + Q of one suit**. Worth **K+Q = 2 pts**, **K+Q+J = 3 pts**, doubled if the suit is trump. The declaration opens a **3-second challenge window** with a visible timer bar; opponents may shout **"Marials!"** to challenge a suspected bluff, or **Accept**. (In the current build the AI only ever declares a *real* Jodie, so Marials always fails and the Jodie stands; true bluffing is flagged as a TODO.)
- **Tunny** — Offered to the human during the Hold window (and when leading or holding ≥5 trumps). You bet you will personally win **all 6 tricks**. Success = **4 balls** to your team; failure = **4 balls** to opponents; if your *partner* wins a trick ("partner catch") = **8 balls** to opponents. The Tunny prompt auto-dismisses after 5 seconds if ignored.
- **Double** — Offered before the 6th trick if your team has won the first five. You bet you'll win the last trick yourself. Win = **2 balls**; a "backward double" (when you are the counting team) pays **4 balls**; failure = **4 balls** to opponents. **Banned when your team is on the corner house.**
- **Khanack** — (Defined in the rules panel) Requires a Jodie *and* the opponents having won at least one trick. Win = **3 balls** (6 if "backward"); failure = **4 balls**; it raises the round target to 13.

### 2.10 Player names (flavour)

Each game randomly assigns affectionate Hyderabadi family/relative names. The partner is drawn from `['Nana Ji','Tau Ji','Chacha Ji','Bauji','Bade Papa','Dada Ji']` and opponents from `['Jeeja Ji','Mama Ji','Saala Sahab','Fufa Ji','Nana Ka Yaar','Shaadi Wale Bhaiya']`. The human's name comes from the optional name input (default "Aap").

---

## 3. The AI (single-player bots)

The `AI` module drives all three bots (and any unfilled multiplayer seats). It is deterministic, lightweight, and rule-following rather than search-based:

- **`handStrength(cards, suit)`** — scores a suit (J = +35, 9 = +25, A = +12, 10 = +10, others = +3, plus length bonuses of +10 for 3 cards and +15 for 4), capped at 100. Used for bidding and trump choice.
- **`bestSuit(cards)`** — the suit with the highest `handStrength`.
- **`decideBid(hand4, currentHighBid, position, isDealer)`** — passes below a strength threshold (40 normally, 30 for the dealer); bids aggressively (up to 100) on very strong hands (strength ≥ 80), moderately (≤ 60) on strength ≥ 60, and minimally (10) otherwise.
- **`chooseCard(...)`** — card-play heuristics:
  - **Leading**: lead the highest trump if it's a J or 9, else lead the highest non-trump.
  - **Partner already winning**: throw off the lowest-point card.
  - **Can win**: win with the *minimum* card sufficient to take the trick.
  - **Can't win**: discard the lowest-point card.
- **`shouldDeclareJodie(hand, trump, handNum)`** — after trick 1 or 3, declares K+Q (or K+Q+J) if present, flagging whether the suit is trump.

The bots respect the same `getLegalMoves` constraints (follow-suit and over-trump) as the human.

---

## 4. User Interface & UX

The two builds share the same polished interface. Key elements:

- **Screens** (`.screen`): Landing, Game, and — in multiplayer — a Lobby screen.
- **Landing screen**: animated Hyderabadi logo, a fan of sample cards, a rotating carousel of ~10 Hyderabadi flavour quotes (changing every 4s), an optional name input, primary "Khelo! (vs AI)" button, How-to-Play and About buttons, and a **Classic ↔ Modern theme toggle**.
- **Themes**: a CSS-variable theme system. *Classic* is a green felt table with gold accents; *Modern* (`body.theme-modern`) is a dark navy/indigo table with purple accents. Suits, buttons, glow colours, and fonts are all variable-driven.
- **Game table**: a CSS-grid felt table with radial-gradient lighting. Four player zones (top/left/right/bottom), a central play area with four positional card slots, a top score bar showing each team's **balls as illuminated dots** plus the (revealed) trump indicator, a top nav (Hints / Quit), and a bottom action bar with contextual instructions.
- **Cards**: rendered as styled `<div>`s with rank, suit pip, large center pip, and an inverted bottom rank. Playable cards lift and scale on hover; the selected card glows. Opponents' hands show **face-down Charminar-photo card backs** (a fanned row for the top partner, vertical stacks for the side opponents). The active player's name tag highlights and shows a "thinking…" animation.
- **Hand sorting**: the human's hand is auto-sorted with trump first, then by suit, then by rank.
- **Animations**: cards deal in with a scale/rotate-in, trick winners pulse, and toasts slide in from the top.
- **Overlays/modals**: Bid, Trump-select, Jodie challenge bar (with countdown), Hold timer, Tunny/special-call prompt, Round result, Game over, Quit confirm, and About (with the family dedication).
- **Hints panel**: a slide-in side panel ("Tunny ke Qayde") summarising every rule — card ranking, bidding, trump, follow-suit, undercut, scoring, balls, Jodie, Tunny, Double, Khanack, and Two-to-clear.
- **Toasts & flavour**: a `HYD_TOASTS` table provides randomised Hyderabadi reactions for every event (bid won/passed, trump set/revealed, Jodie, hand win, round/game win/lose, corner house).
- **Responsive**: a `@media (max-width: 600px)` block shrinks cards and tightens the fan layout for phones.

---

## 5. Single-Player Version (`index.html`)

### 5.1 Behaviour

- One human (seat 0, bottom) versus three AI bots. The human's partner sits at the top; the two opponents are left and right.
- View rotation is fixed (`_viewRotation = 0`) — the human is always seat 0.
- The full game runs entirely in the browser; there is no network dependency.
- Entry point: the **"🃏 Khelo! (vs AI)"** button calls `Game.start()`.

### 5.2 Deployment — GitHub Pages

- **Repository**: `https://github.com/mathursunit/TunnyV1` (remote `origin`).
- **Custom domain**: the repo contains a `CNAME` file with `tunny.sunitmathur.com`, so GitHub Pages serves the site at that domain over HTTPS.
- **`.nojekyll`** is present to disable Jekyll processing so all files (including the `.webp` card back) are served verbatim.
- **Assets in the repo**: `index.html`, `charminar_back.webp` (5 KB optimised card back), `Charminar.png` (full-resolution source image, ~1.8 MB), `CNAME`, `.nojekyll`.
- **Publishing model**: GitHub Pages serves the static files straight from the repository — pushing to the default branch updates the live single-player game. No build pipeline is involved.
- **Version history (git log)** shows the evolution: initial commit (v0.1), randomised Hyderabadi names, several gameplay/crash fixes, the About modal + family dedication (v1.0.4), the Charminar SVG then real-photo card back (v1.0.5 / v1.0.6), and v1.1.0 (team-seating fix so partners sit opposite, dealer badge, required name for multiplayer, public lobby).

---

## 6. Multiplayer Version (`index-mp.html`)

The multiplayer build is a **superset** of the single-player file: the entire rules engine, AI, and UI are identical, with an added lobby system and a networking layer (`MP` module). Empty seats are automatically filled with the same AI bots, so a game can run with anywhere from 1 to 4 humans.

### 6.1 Networking architecture — "client-authoritative relay"

The design (labelled *"Architecture 1 — Client-authoritative relay"* in the code) is a **host-authoritative** model relayed through a thin WebSocket server:

- The room creator is **seat 0 = the host**. The host's browser runs the *complete* game loop (`Game.runPhase()`), exactly as in single-player.
- **Non-host clients are thin**: they do not run the game loop. They render what the host tells them and send back only their own actions (bids, trump choice, card plays).
- The server itself is a **dumb relay**: it manages rooms and forwards messages between clients. The authoritative game state lives in the host's browser, not on the server.

**Connection** (`WS_URL`): the client opens a WebSocket to **its own origin** — `wss://<page-host>` when served over HTTPS, `ws://<page-host>` otherwise (`${proto}//${location.host}`). In other words, the same OCI host that serves `index-mp.html` must also run the WebSocket relay (or terminate/proxy it on the same host:port), upgraded to `wss://` behind TLS.

**Message envelope**: every message is JSON `{ type, payload }`.

Client → server message types:

| Type | Purpose |
|---|---|
| `create_room` | Create a room (`{name, roomName, isPublic}`) |
| `join_room` | Join by code (`{code, name}`) |
| `list_rooms` | Request the public-room list |
| `player_ready` | Mark ready / signal start |
| `game_action` | Wrapper for all in-game actions (carries an `action` field) |
| `chat` | Lobby chat message |

Server → client message types handled in `_onMessage`:

`room_created`, `room_joined`, `player_joined`, `player_left`, `game_start`, `game_action`, `chat`, `rooms_list`, `error`.

In-game `game_action` sub-actions (host ↔ clients), handled in `_handleRemoteAction`:

| Action | Direction | Meaning |
|---|---|---|
| `force_start` | host → all | Begin the game; carries `humanSeats` (which seats are humans vs AI) |
| `deal_hands` | host → clients | Each client's cards (+ dealer, trump when known) |
| `your_turn_bid` | host → a client | It's your turn to bid (`highBid`, `isDealer`) |
| `bid` | client → host | A client's bid result |
| `your_turn_trump` | host → a client | You won the bid — pick trump |
| `set_trump` | client → host | Trump selection |
| `your_turn` | host → a client | Your turn to play a card (`ledSuit`) |
| `play_card` | both directions | A card was played (`seat`, `cardId`) — used both as the host's broadcast for visual updates and as the client's reply |

The host blocks on remote players via `waitForRemote()`, which returns a promise resolved when the matching `_handleRemoteAction` reply arrives (one pending action at a time via `_pendingResolve`).

### 6.2 View rotation

Because every player should see themselves at the bottom of the table, `startMultiplayer(mySeat, humanSeats)` sets `_viewRotation = mySeat`. The helper `_visualSlot(canonicalSeat)` maps canonical seats `0..3` to the four on-screen positions (`handBottom/handLeft/handTop/handRight`) relative to the viewer, so seat assignments stay consistent across all four screens while each player sees their partner across from them.

### 6.3 Lobby system

- **Name required**: multiplayer requires a player name (`_requireName()` shakes the input and toasts "Pehle apna naam daalo!" if empty) — a v1.1.0 addition.
- **Create Room**: opens a modal pre-filled with "*<name>*'s Room", with a **Private 🔒 / Public 🌐** toggle. On confirm it connects and sends `create_room`.
- **Join Room**: prompts for a 4-letter room code and sends `join_room` (codes are uppercased).
- **Browse Public Rooms**: a modal listing open public rooms, **auto-refreshing every 5 seconds** (`list_rooms`), each row showing room name, code, and player count with a Join button.
- **Lobby screen**: shows the room name, the 4-letter join **code**, four **seat slots** (filled/empty, with "you" highlighted), a live **chat** box, a status line ("Full house!" / "*n*/4 players joined"), and Leave / Start buttons.
- **Starting with fewer than 4 humans**: only the host can start. The Start button reads "Start Game →" when full, or "Start with AI fill (*n*/4) →" otherwise; the host broadcasts `force_start` with the list of human seats and the remaining seats are played by the AI.
- **Resilience**: if the socket drops mid-game the client toasts "⚠️ Connection lost!"; leaving the lobby cleanly closes the socket and resets all `MP` state.

### 6.4 Deployment — self-hosted on OCI

The multiplayer client and its **WebSocket relay server** run together on the user's own **Oracle Cloud Infrastructure (OCI)** instance. The following is the *actual deployed configuration* (verified by inspecting the live OCI host), not a reconstruction.

**Host & infrastructure**
- **OCI instance**: Ubuntu 24.04 LTS, **ARM64 (aarch64)**, public IP `132.145.145.128` (SSH as `ubuntu`).
- **Edge / TLS**: **Caddy** is the public web server, listening on ports 80 and 443 and providing automatic TLS. The multiplayer site is served at the dedicated subdomain **`tunny2.sunitmathur.com`** (separate from the single-player `tunny.sunitmathur.com`). Its Caddy site block simply reverse-proxies to the app:
  ```
  tunny2.sunitmathur.com {
      reverse_proxy localhost:3010
      tls internal
  }
  ```
  Caddy terminates HTTPS and transparently upgrades the WebSocket to the backend, which is why the client's `wss://<location.host>` connection works against the same origin that serves the page. (The code comment "use relative WS when served from tunny2" refers to exactly this subdomain.)

**The application — `tunny2` Docker container**
- Runs as a **Docker** container named **`tunny2`**, image **`node:20-alpine`**, restart policy `unless-stopped`, publishing **port 3010**. Source lives on the host at **`/home/ubuntu/tunny2`**, bind-mounted to `/app`.
- `docker-compose.yml` command: `sh -c "npm install --production && node server.js"`, `NODE_ENV=production`.
- **`package.json`** (`tunny2-server` v1.0.0) has just two dependencies: **`ws`** (`^8.16.0`, the WebSocket server) and **`uuid`** (`^9.0.0`, for connection IDs). No framework.
- **`public/`** holds the served static assets: the multiplayer build (deployed as **`index.html`**, byte-identical in size to the repo's `index-mp.html`) and `charminar_back.webp`.

**`server.js` — what the relay actually does (~150 lines, Node `http` + `ws`)**
- **One process serves both** the static files (a tiny `http.createServer` file server with a small MIME map and a path-traversal guard, falling back to `index.html`) **and** the WebSocket server (`new WebSocket.Server({ server })`) on the same port 3010.
- **Rooms** are held in an in-memory `Map` (`code → room`); there is **no database** — state is ephemeral and lost on restart. Each room is `{ code, roomName, isPublic, players[], state }` where `state` is `'waiting'` or `'playing'`.
- **Room codes**: 4 characters from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (note: visually ambiguous characters I/O/0/1 are deliberately excluded), regenerated until unique.
- **Seat assignment**: the creator gets **seat 0** (host); each joiner takes the next index. Joins are rejected with an `error` if the room is **full (4 players)**, **not found**, or **already playing**.
- **Message handling** mirrors the client protocol in §6.1: `create_room` → `room_created`; `join_room` → `room_joined` to the joiner plus `player_joined` broadcast; `list_rooms` → `rooms_list` (only public, waiting, non-full rooms); `player_ready` (and an auto-`game_start` once all 4 are ready); `chat`; and `ping`/`pong`.
- **`game_action` relay**: the server is a pure pass-through — it re-broadcasts each `game_action` to **all other players in the room** (excluding the sender) and stamps it with the sender's `fromSeat`. It never inspects or validates game logic; the **host's browser remains authoritative** (as described in §6.1).
- **Connection lifecycle**: a **30-second heartbeat** (`ping`/`pong`, terminating dead sockets) keeps connections alive; on `close`, the player is removed, a `player_left` is broadcast, and an **empty room is deleted automatically**.

**Other services on the same OCI box** (for context, unrelated to Tunny): the instance also runs Caddy-fronted Docker containers for `sslcheck` (frontend/backend), `uptime-kuma`, `vaultwarden`, and `n8n`, plus a local `ollama` — Tunny is one of several apps multiplexed behind Caddy by subdomain.

**To redeploy/update the multiplayer build:** copy the new `index-mp.html` to `/home/ubuntu/tunny2/public/index.html` on the OCI host; static files are served live (no rebuild needed). Restart only if `server.js` or dependencies change (`docker compose up -d` in `/home/ubuntu/tunny2`).

---

## 7. Differences at a glance: single-player vs multiplayer

| Aspect | Single-player (`index.html`) | Multiplayer (`index-mp.html`) |
|---|---|---|
| Opponents | Always 3 AI bots | Up to 3 other humans; AI fills any empty seats |
| Game loop | Runs locally in the one browser | Runs in the **host's** browser; guests are thin clients |
| Networking | None | WebSocket relay (`{type,payload}` JSON) |
| Lobby | None | Create / Join / Browse public rooms, 4-letter codes, chat, seat picker |
| View rotation | Fixed (human = seat 0, bottom) | Rotated so each player sees themselves at the bottom |
| Entry buttons | "Khelo! (vs AI)" | Same, plus Create Room / Join Room / Browse Public Rooms |
| Hosting | GitHub Pages @ `tunny.sunitmathur.com` | OCI: Docker `node:20-alpine` + `ws` on :3010, behind Caddy @ `tunny2.sunitmathur.com` (HTTPS + WSS) |
| Name | Optional (defaults to "Aap") | **Required** to enter multiplayer |

---

## 8. Technical summary

- **Stack**: pure HTML + CSS + vanilla JavaScript, single file per version. No frameworks, no bundler, no dependencies. Multiplayer adds a browser-side `WebSocket` client only.
- **Key modules**: card model (`makeDeck`, `shuffle`, `cardValue`, `beats`), `AI`, the `Game` state machine (`runPhase` over the `draw → … → score` phases), rendering helpers (`renderPlayerHand`, `renderSideHand`, `playCardToTable`, `renderScoreBar`, `_visualSlot`), UI utilities (screens/overlays/toasts/themes), and — in multiplayer — the `MP` networking module.
- **Constants**: `BALLS_TO_WIN = 12`, `WIN_THRESHOLD = 10.5`, `DECK_TOTAL = 30.0`, `TEAMS = [[0,2],[1,3]]`.
- **Assets**: `charminar_back.webp` (card back), `Charminar.png` (source image).
- **Version constant**: `TUNNY_VERSION = 'v1.1.0'`, surfaced on the landing page and in the About modal alongside `tunny.sunitmathur.com`.

### Client/server notes observed during analysis
- **Two start paths exist.** The host client drives starts by sending `force_start` (via `game_action`), which lets the host begin with fewer than 4 players and AI-fill the empty seats. The server *also* has an independent auto-start path that fires `game_start` only when all 4 players have sent `player_ready`. In practice the host-driven `force_start` is what launches games (and is what enables AI-fill); the server's 4-ready auto-start is a secondary path.
- The server relays `game_action` to **all** other players in the room and adds a `fromSeat` field; the client targets the intended recipient by checking `seat`/`mySeat` inside `_handleRemoteAction`.

### Known simplifications / TODOs (present in code comments)
- Jodie **bluffing is not yet implemented** — the AI only declares real Jodies, so a "Marials!" challenge always fails.
- The Double/Khanack "won all previous hands" check is simplified (assumes the condition rather than fully verifying trick-by-trick).
- The deal-4 routine contains a redundant first pass that is overwritten by a clean 4-cards-each deal.

---

*Document generated from source analysis of `index.html` and `index-mp.html` (both v1.1.0).*
