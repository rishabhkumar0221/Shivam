# Sketchy – a skribbl.io-style drawing & guessing game

Real-time multiplayer pictionary: create or join a room, take turns drawing a secret word, and race to guess it.

**Live demo:** `https://YOUR-APP.onrender.com`  ← replace after deploying (see below)

## Features
- Public & private rooms, join by 5-letter code or invite link, lobby with ready-up and host start
- Host settings: players (2–20), rounds (2–10), draw time (15–240 s), word choices (1–5), hints (0–5, 0 = off), word mode (Normal / Hidden / Combination), custom words
- Real-time canvas over WebSockets: brush, 12 colours, 3 sizes, eraser, undo, clear (drawer only)
- Turn-based rounds (everyone draws once per round), word picker with auto-pick, timer, letter hints over time
- Scoring (faster guess = more points, drawer earns a bonus), per-round results, final leaderboard, play again
- Chat with close-guess feedback; correct guessers and the drawer chat privately so the word can't leak
- Host moderation: kick (banned from rejoining), avatars

## Run locally
```bash
npm install                 # server deps
cd client && npm install    # client deps
cd ..
npm run dev                 # terminal 1 -> server on :3001
cd client && npm run dev    # terminal 2 -> Vite on :5173 (proxies /socket.io to :3001)
```
Open http://localhost:5173 in two browser windows to play against yourself.

Production-style (single process): `npm run build && npm start` → http://localhost:3001

Automated test (simulates two players through a full 2-round game): `npm test`

## Deploy on Render (free, WebSocket support)
1. Push this folder to a GitHub repo.
2. Render → **New + → Blueprint** (uses `render.yaml`) or **New Web Service** with  
   Build: `npm install && npm run build` · Start: `npm start` · Health check: `/health`
3. Copy the `https://….onrender.com` URL into this README.

Express serves both the API/WebSocket server and the built React app from one service, so there are no CORS or split-deploy issues.
Vercel/Netlify can't host a long-lived Socket.IO server (serverless), so they would only host the frontend; set `VITE_SERVER_URL` to the Render/Railway backend URL at build time if you split them.
Free Render instances sleep when idle (first load can take ~30 s), and rooms live in memory so a restart clears them.

## Architecture
```
client/ (React + Vite)                 server/ (Node, Express, Socket.IO)
 Home / Lobby / GameView                index.js      MessageHandler: one per socket, maps events -> methods
 DrawCanvas (HTML5 canvas)  <──WS──>    Room.js       Room (players, host, ban list, broadcast) + RoomManager
 socket.js                              Game.js       Game (phases, turn order, timers, hints, scoring, strokes)
                                        Player.js     Player (name, avatar, score, ready, hasGuessed)
                                        words.js      categorised word list
```

### Drawing sync
The drawer's pointer events become `draw_start {x,y,color,size}`, `draw_move {x,y}`, `draw_end`. Coordinates are **normalised to 0–1**, so any screen size renders identically on the fixed 800×600 logical canvas. The server checks the sender is the current drawer, stores the stroke (for undo and late joiners), and relays it as `draw_data` to everyone else; the drawer paints locally for zero latency. Undo/late-join send the full stroke list via `canvas_state`, which clients simply redraw.

### Game state
`Game` is a state machine: `lobby → choosing → drawing → roundEnd → (next turn…) → gameOver`.
Each round the server copies the player ids into a queue; popping it gives the next drawer, so everyone draws once per round. When the queue is empty the round counter increments, or the game ends after the last round. A single server-side `setInterval` drives every countdown; the server is the source of truth and sends personalised `game_state` snapshots (the word is only included for the drawer, for players who already guessed, and at round end).

### Scoring
`points = max(50, 100 + 300 × timeLeft/drawTime − 20 × earlierCorrectGuessers)`; the drawer gets 40 % of each guesser's points. The round ends early when everyone has guessed.

### Word matching
Guess and answer are normalised (Unicode accents stripped, lower-cased, trimmed, inner whitespace collapsed) and must match exactly. A guess one typo away (edit distance 1) only tells that player "really close!". Wrong guesses are shown as chat; correct ones announce "X guessed the word!" without revealing the word.

### WebSocket events
| Direction | Events |
|---|---|
| Client → Server | `create_room`, `join_room`, `list_rooms`, `ready`, `start_game`, `word_chosen`, `draw_start`, `draw_move`, `draw_end`, `canvas_clear`, `draw_undo`, `guess`, `chat`, `kick`, `play_again`, `leave_room` |
| Server → Clients | `game_state`, `player_joined`, `player_left`, `round_start`, `word_options`, `word_chosen`, `timer`, `round_end`, `game_over`, `draw_data`, `canvas_clear`, `canvas_state`, `guess_result`, `chat_message`, `kicked`, `error_message` |

Server-side validation: settings are clamped to the allowed ranges, only the drawer can draw/choose words, only the host can start/kick, and coordinates are clamped to 0–1.

## Possible improvements
Reconnect with the same identity, spectator mode, votekick, round replay, persistent rooms (Redis) to scale beyond one instance.
