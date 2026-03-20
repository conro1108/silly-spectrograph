# Silly Spectrograph v2 - Design Document

## Product Vision

A beautiful, mesmerizing real-time audio spectrograph web app you can pull up at the park and just *watch* as birds chirp, dogs bark, people talk, and the world hums around you. The spectrograph is the star — lush color palettes, smooth scrolling, and enough visual detail to distinguish a cardinal from a crow. Layered on top: playful, delightful "silly" features that surprise and reward you without getting in the way of the core zen experience.

## Target User

Someone sitting on a park bench with their phone or laptop, watching sound paint itself across the screen. They want beauty first, silliness second, zero friction to start.

---

## Core Features

### 1. Spectrograph (Primary)

The full-screen, continuously-scrolling frequency spectrogram. This is 90% of the app.

- **High-quality FFT visualization** — 4096-point FFT for detailed frequency resolution (hear the difference between bird species)
- **Multiple color palettes** — cycle through: Thermal (warm), Ocean (cool blues/greens), Neon (synthwave), Grayscale, Park Mode (earth tones optimized for nature sounds)
- **Logarithmic frequency scale** — maps frequency bins to a perceptually-accurate log scale so birdsong and bass are both visible
- **Adjustable parameters:**
  - Gain (sensitivity)
  - Scroll speed
  - FFT smoothing (temporal smoothing for dreamier vs. crispier look)
  - Brightness/contrast
- **Frequency labels** — subtle tick marks on the Y-axis showing Hz values (toggleable)
- **Auto-gain** — optional automatic gain adjustment based on ambient noise floor

### 2. Silly Features

Fun, opt-in features that layer on top without obstructing the spectrograph.

#### a) Sound Detective (replaces speech recognition)
- Uses Web Audio frequency analysis (not speech recognition API) to detect sound *patterns*:
  - **Bird chirps** — high-frequency transients → bird emoji rain (🐦🐤🦜)
  - **Dog barks** — mid-frequency bursts → dog emoji rain (🐕🐶🦴)
  - **Laughter** — characteristic spectral pattern → laughing emojis (😂🤣😆)
  - **Clapping** — broadband transient → clap/celebration emojis (👏🎉✨)
  - **Bass/music** — sustained low-frequency energy → music emojis (🎵🎶🔊)
- Emoji particles rain down as a translucent overlay, physics-based with gravity and slight wind drift
- Toggle on/off with a single tap

#### b) Vibe Check
- A floating indicator that reads the "mood" of the ambient sound:
  - "Peaceful 🌿" (quiet, nature sounds)
  - "Lively 🎉" (lots of activity)
  - "Musical 🎵" (sustained tonal content)
  - "Chaotic 🌪️" (loud, unpredictable)
- Smooth transitions between states, shown as a small pill in the corner

#### c) Sound Snapshots
- Tap a camera button to "screenshot" the current spectrograph view
- Saves as a PNG with a subtle watermark ("Silly Spectrograph")
- Downloads directly to device

#### d) Time Markers
- Tap to drop a pin on the spectrograph timeline
- Shows a vertical line with a label ("Cool bird!", auto-timestamped)
- Markers scroll with the spectrograph

### 3. Controls UI

Minimal, non-intrusive overlay that auto-hides after 3 seconds of no interaction:
- **Top bar** (auto-hiding): palette selector, settings gear
- **Bottom bar** (always visible, minimal): Sound Detective toggle, Vibe Check pill, snapshot button
- **Settings drawer** (slides from right): gain, speed, smoothing, frequency labels toggle, auto-gain toggle
- Tap anywhere on the spectrograph to show/hide top bar

---

## User Flows

### First Visit
1. Landing screen: app title, one-line description, big "Start Listening" button
2. Browser requests microphone permission
3. **If granted:** spectrograph begins immediately, controls auto-hide after 3s
4. **If denied:** friendly error message with instructions to enable mic in browser settings
5. **If not supported (no getUserMedia):** static message suggesting a supported browser

### Returning Visit
- Preferences (palette, gain, speed, etc.) persisted in localStorage
- Spectrograph starts immediately on tap (AudioContext resume needed per browser policy)

### Error States
- **Mic disconnected mid-session:** pause spectrograph, show reconnect prompt
- **Browser tab backgrounded:** pause rendering (save battery), resume on focus
- **No audio detected for 30s:** subtle "No audio detected — is your mic on?" indicator

### Loading State
- Minimal — app is a single page with no external dependencies to load
- Show a pulsing waveform animation while AudioContext initializes

---

## Architecture

### Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Node.js 20 LTS | Render supports it natively |
| Server | Express.js | Minimal static file server, room to add API later |
| Frontend | Vanilla JS + Canvas API | Zero dependencies, fast, full control over rendering |
| Build | None | Single HTML file approach, server just serves static |
| Deployment | Render Web Service | Free tier, auto-deploy from GitHub |

### Project Structure

```
/
├── server.js          # Express static server (8 lines)
├── package.json       # Node.js manifest for Render
├── public/
│   └── index.html     # The entire app (HTML + CSS + JS)
├── render.yaml        # Render blueprint (optional)
├── DESIGN.md
└── README.md
```

### Why This Architecture

- **Single HTML file in `public/`**: The app is entirely client-side. No API needed. No database. No WebSocket. The server exists solely because Render needs a process to run.
- **No build step**: No bundler, no transpiler, no framework. The app is ~600 lines of vanilla JS. This is a feature, not a limitation — it keeps deployment trivial and the codebase greppable.
- **No external dependencies (frontend)**: Zero CDN links, zero npm packages on the client side. Everything is Web APIs (AudioContext, Canvas, getUserMedia).

### Server (server.js)

Minimal Express server:
```js
const express = require('express');
const app = express();
app.use(express.static('public'));
app.listen(process.env.PORT || 3000);
```

That's it. Render runs `node server.js`, serves the static files over HTTPS.

### Render Deployment

- **Build command:** `npm install`
- **Start command:** `node server.js`
- **Environment:** Node.js
- **Auto-deploy:** on push to `main`

---

## Data Model

No persistent data. All state is ephemeral and client-side:

| State | Storage | Lifetime |
|---|---|---|
| User preferences (palette, gain, speed) | localStorage | Persistent across sessions |
| Spectrograph pixel data | Canvas buffer | Current session only |
| Emoji particles | JS array in memory | Transient (seconds) |
| Time markers | JS array in memory | Current session only |
| Sound snapshots | Downloaded as PNG | User's device |

---

## API Design

No API. The server serves static files. If we later want features like:
- Sharing snapshots → add a `/api/share` endpoint
- Sound identification → integrate a classification API

...the Express server is already there to extend.

---

## Performance Considerations

- **requestAnimationFrame** for render loop (not setInterval)
- **OffscreenCanvas** for the spectrograph strip generation (if supported, fallback to regular canvas)
- **Typed arrays** for FFT data and palette lookup (Uint8ClampedArray)
- **Batch particle updates** — single pass, splice dead particles
- **Pause rendering when tab is backgrounded** (Page Visibility API)
- **Log-scale frequency mapping** precomputed once on resize, not per-frame

## Security

- Microphone access requires HTTPS (Render provides this)
- No user data collected or transmitted
- No cookies, no tracking, no analytics
- CSP headers set by Express for defense-in-depth

---

## Tradeoffs & Decisions

| Decision | Alternative | Why This Way |
|---|---|---|
| Vanilla JS over React/Vue | Framework would add build step | App is a single interactive canvas — no component tree needed |
| Frequency analysis for sound detection over speech recognition | SpeechRecognition API is unreliable cross-browser | Frequency heuristics work everywhere getUserMedia works |
| Single HTML file over multi-file | Separate JS/CSS files | Keeps it greppable, no import issues, trivial to understand |
| Express over static site hosting | Render Static Sites | Express gives us room to grow + Render Web Service is more flexible |
| localStorage over IndexedDB | IndexedDB for larger data | We're storing ~5 key-value pairs, localStorage is simpler |
