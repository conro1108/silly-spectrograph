# Silly Spectrograph v2 - Design Document

## Scope

This is a **ground-up rewrite**, not an incremental revamp. The v1 app used the SpeechRecognition API to detect profanity and trigger emoji rain. v2 replaces that entirely with frequency-based sound detection and focuses on the spectrograph as the primary experience. The v1 code is reference only.

## Product Vision

A beautiful, mesmerizing real-time audio spectrograph web app you can pull up at the park and just *watch* as birds chirp, dogs bark, people talk, and the world hums around you. The spectrograph is the star — lush color palettes, smooth scrolling, and enough visual detail to distinguish a cardinal from a crow. Layered on top: playful, delightful "silly" features that surprise and reward you without getting in the way of the core zen experience.

## Target User

Someone sitting on a park bench with their phone or laptop, watching sound paint itself across the screen. They want beauty first, silliness second, zero friction to start.

---

## Core Features

### 1. Spectrograph (Primary)

The full-screen, continuously-scrolling frequency spectrogram. This is 90% of the app.

- **High-quality FFT visualization** — 4096-point FFT for detailed frequency resolution
- **Logarithmic frequency scale** — precomputed lookup table mapping pixel rows to FFT bins using log scale so birdsong (2-8 kHz) and bass are both clearly visible:
  ```
  freq = minFreq * (maxFreq / minFreq) ^ (percent)
  bin = freq / (sampleRate / fftSize)
  ```
- **5 color palettes** with gradient swatch preview in selector:
  - Thermal (warm reds/oranges/yellows)
  - Ocean (cool blues/greens)
  - Neon (synthwave purples/pinks/cyans)
  - Grayscale
  - Park Mode (earth tones optimized for nature sounds)
- **Adjustable parameters:**
  - Gain (sensitivity) — with real-time amplitude indicator for feedback
  - Scroll speed
  - FFT smoothing (temporal smoothing for dreamier vs. crispier look)
- **Frequency labels** — subtle tick marks on the Y-axis showing Hz values (toggleable)
- **Auto-gain** — optional automatic gain adjustment based on ambient noise floor

### 2. Silly Features

Fun, opt-in features that layer on top without obstructing the spectrograph.

#### a) Sound Detective
Uses Web Audio frequency analysis to make *playful guesses* (not identifications) about ambient sounds. These are fun, not scientific.

**Detection algorithm specs:**
| Sound | Frequency Range | Trigger Condition | Emojis |
|---|---|---|---|
| Bird chirps | 2000-8000 Hz | Spectral centroid > 3kHz AND transient onset (energy spike > 2x rolling avg in 50ms window) | 🐦🐤🦜 |
| Dog barks | 300-2000 Hz | Mid-band energy burst > 3x ambient, duration 100-500ms | 🐕🐶🦴 |
| Bass/music | 40-250 Hz | Sustained low-freq energy > ambient for 2+ seconds | 🎵🎶🔊 |
| Loud transient | Broadband | Full-spectrum energy spike > 4x ambient in < 100ms | 👏🎉✨ |

**False positive mitigation:**
- 3-second cooldown between triggers of the same category
- Require 2 consecutive matching analysis frames before triggering
- Particle cap: max 200 active particles (new triggers ignored if at cap)
- Particles use swap-and-pop removal instead of splice for O(1) removal

Emoji particles rain down as a translucent overlay, physics-based with gravity and slight wind drift. Toggle on/off with a single tap.

#### b) Vibe Check
- An `aria-live="polite"` floating pill that reads the ambient "mood":
  - "Peaceful 🌿" — low RMS, high spectral centroid (nature-like)
  - "Lively 🎉" — high RMS variance
  - "Musical 🎵" — sustained tonal content (low spectral flux)
  - "Chaotic 🌪️" — high RMS + high spectral flux
- Smooth CSS transitions between states (min 5s between changes)

#### c) Sound Snapshots
- Camera button captures the current spectrograph canvas as PNG
- Overlay: subtle "Silly Spectrograph" watermark + timestamp
- Downloads directly via `<a download>` — no server needed

#### d) Time Markers
- Tap to drop a timestamped pin on the spectrograph timeline
- Vertical line with auto-timestamp label, scrolls with spectrograph
- **Persisted to localStorage** so they survive page refreshes within a session
- Included in snapshot PNGs when captured

### 3. Controls UI

**Persistent affordance model** (no auto-hiding):

- **Top-right corner**: single gear icon (44x44px touch target), taps to expand settings panel
- **Settings panel** (slides from right): palette selector with gradient swatches, gain slider with amplitude indicator, speed slider, smoothing slider, frequency labels toggle, auto-gain toggle. `role="dialog"` + `aria-modal="true"`. Closes on Escape or tap outside. Focus trapped when open.
- **Bottom bar** (always visible, minimal, respects safe areas): Sound Detective toggle, Vibe Check pill, snapshot button, time marker button
- **Zen Mode**: long-press anywhere hides ALL UI. Long-press again to restore. Visual hint on first use.

All interactive elements: minimum 44x44px touch targets.

---

## Accessibility

### Keyboard Navigation
- **Tab order**: Start button → bottom bar buttons (left to right) → gear icon
- **Settings panel**: focus trapped when open, Escape closes
- **Spacebar**: toggles pause
- **Enter**: triggers focused button action
- **Arrow keys**: adjust sliders (native `<input type="range">` behavior)

### ARIA
- Spectrograph canvas: `role="img"` with dynamic `aria-label` (e.g., "Live audio spectrograph showing moderate activity")
- Vibe Check pill: `aria-live="polite"` for mood announcements
- Status indicator ("No audio detected"): `aria-live="assertive"`
- All icon buttons: explicit `aria-label` attributes
- Settings panel: `role="dialog"`, `aria-modal="true"`

### Color Contrast
- Minimum 4.5:1 contrast ratio for ALL text at every size (WCAG AA)
- UI text: `#e2e8f0` on `rgba(0,0,0,0.8)` backgrounds minimum
- No text smaller than 12px

### Viewport
- Remove `user-scalable=no` and `maximum-scale=1.0` — use `touch-action: manipulation` on canvas instead to prevent double-tap zoom without blocking pinch-to-zoom
- Add `viewport-fit=cover` for edge-to-edge rendering

---

## User Flows

### First Visit
1. Landing screen: app title, one-line description, big "Start Listening" button
2. Browser requests microphone permission
3. **If granted**: pulsing sine wave loading animation (rendered in selected palette for continuity) → spectrograph begins
4. **If denied**: persistent in-page error state (NOT alert()) with:
   - Browser-specific instructions (detect UA for iOS Safari vs Chrome vs Firefox)
   - "Try Again" button that re-calls `getUserMedia`
   - Visual diagram showing where to find the permission toggle
5. **If not supported**: static message suggesting Chrome, Edge, or Safari

### Returning Visit
- All preferences restored from localStorage (palette, gain, speed, smoothing, toggles)
- Single tap on "Start Listening" to resume (AudioContext browser policy)

### Error States
- **Mic disconnected mid-session**: pause spectrograph, show inline reconnect prompt with "Try Again" button
- **Browser tab backgrounded**: Page Visibility API pauses rendering AND suspends AudioContext (critical for mobile battery)
- **No audio detected for 30s**: `aria-live="assertive"` indicator: "No audio detected — is your mic on?"

### Loading State
- Sine wave animation that starts low amplitude and gradually increases, rendered in the active color palette
- Seamless crossfade into live spectrograph when AudioContext is ready

---

## Architecture

### Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Hosting | Render Static Site | Zero cold starts, global CDN, no server process needed |
| Frontend | Vanilla JS + Canvas API | Zero dependencies, fast, full control over rendering |
| Build | None | Single HTML file, no bundler/transpiler |

### Project Structure

```
/
├── public/
│   └── index.html     # The entire app (HTML + CSS + JS)
├── render.yaml        # Render Static Site blueprint
├── DESIGN.md
└── README.md
```

### Why This Architecture

- **Render Static Site instead of Web Service**: The app is 100% client-side. No API, no database, no WebSocket. A Static Site deployment gives us instant loads from CDN with zero cold starts — critical for the "pull up at the park" use case. No Express server to maintain. We can always migrate to a Web Service later if we need server-side features.
- **Single HTML file**: No bundler, no transpiler, no framework. ~800 lines of vanilla JS. Keeps deployment trivial and the codebase greppable.
- **No external dependencies (frontend)**: Zero CDN links, zero npm packages. Everything is Web APIs (AudioContext, Canvas, getUserMedia).

### Render Deployment (render.yaml)

```yaml
services:
  - type: web
    name: silly-spectrograph
    runtime: static
    buildCommand: ""
    staticPublishPath: ./public
    headers:
      - path: /*
        name: Content-Security-Policy
        value: "default-src 'self'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src 'self' blob: data:; media-src 'self' blob:"
      - path: /*
        name: X-Content-Type-Options
        value: nosniff
      - path: /*
        name: X-Frame-Options
        value: DENY
```

**Note on CSP**: `'unsafe-inline'` is required because the single-file architecture inlines all JS and CSS. This weakens CSP significantly vs. external scripts with hash/nonce. Accepted tradeoff: there is no user-generated content or dynamic HTML injection in this app, so the XSS attack surface is minimal. If we split into separate files later, we can tighten CSP.

---

## Data Model

No persistent server-side data. All state is client-side:

| State | Storage | Lifetime |
|---|---|---|
| User preferences (palette, gain, speed, smoothing, toggles) | localStorage | Persistent across sessions |
| Time markers | localStorage | Persistent across sessions |
| Spectrograph pixel data | Canvas buffer | Current session only |
| Emoji particles | JS array (max 200) | Transient (seconds) |
| Sound snapshots | Downloaded as PNG | User's device |

### localStorage Schema
```json
{
  "spectrograph_prefs": {
    "palette": "thermal",
    "gain": 3.0,
    "speed": 2,
    "smoothing": 0.5,
    "freqLabels": false,
    "autoGain": false,
    "soundDetective": true
  },
  "spectrograph_markers": [
    { "timestamp": "2026-03-19T14:32:00Z", "label": "Cool bird!" }
  ]
}
```

---

## Mobile Considerations

### Safe Areas
All UI overlays use `env(safe-area-inset-*)` for positioning:
- Bottom bar: `padding-bottom: env(safe-area-inset-bottom)`
- Gear icon: `top: calc(8px + env(safe-area-inset-top))`
- Status indicators respect notch/Dynamic Island

### Battery & Thermal
- **Page Visibility API**: suspend AudioContext + stop rAF loop when tab backgrounded
- **Low Power mode**: optional toggle that reduces to 30fps, drops FFT to 2048, disables particles
- Efficient silence detection: sample bins 0, 100, 500 instead of `reduce()` over all 2048 bins

### Orientation
- Layout works in both portrait and landscape
- Landscape recommended for longer time axis — subtle prompt on first portrait use

---

## Performance Considerations

- **requestAnimationFrame** for render loop
- **Typed arrays** for FFT data and palette lookup (Uint8ClampedArray)
- **Log-scale frequency map** precomputed once on resize, stored as Uint16Array
- **Particle cap at 200** with swap-and-pop removal (O(1) per particle)
- **Page Visibility API** pauses everything when backgrounded
- **Efficient silence detection** via spot-checking frequency bins

## Security

- Microphone access requires HTTPS (Render Static Sites serve over HTTPS)
- No user data collected or transmitted
- No cookies, no tracking, no analytics
- CSP + X-Content-Type-Options + X-Frame-Options headers via render.yaml
- `'unsafe-inline'` required for single-file architecture — acceptable given no user-generated content

---

## Tradeoffs & Decisions

| Decision | Alternative | Why This Way |
|---|---|---|
| Vanilla JS over React/Vue | Framework would add build step | App is a single interactive canvas — no component tree needed |
| Frequency heuristics over SpeechRecognition | SpeechRecognition API | Frequency analysis works everywhere getUserMedia works; cross-browser reliable |
| Single HTML file over multi-file | Separate JS/CSS files | Greppable, no import issues, trivial to understand. Tradeoff: requires `unsafe-inline` CSP |
| Render Static Site over Web Service | Express server | No cold starts, CDN delivery, zero server maintenance. Can migrate to Web Service if API needed later |
| localStorage over IndexedDB | IndexedDB for larger data | Storing ~5 key-value pairs + small marker array, localStorage is simpler |
| Persistent UI affordance over auto-hide | Auto-hide after 3s | Auto-hide creates frustrating toggle loops on mobile; persistent gear icon is discoverable without being intrusive |
| 44x44px touch targets | Smaller native defaults | Apple HIG / WCAG 2.5.5 compliance; critical for outdoor one-handed use |
| Playful sound detection framing | "Identification" framing | Frequency heuristics produce false positives outdoors; "playful guesses" sets correct expectations |
