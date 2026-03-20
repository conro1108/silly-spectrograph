# Silly Spectrograph

A beautiful, real-time audio spectrograph web app. Pull it up at the park and watch birds chirp, dogs bark, and the world hum — painted in color across your screen.

## Features

- **Real-time spectrograph** — 4096-point FFT with logarithmic frequency scale, 5 color palettes (Thermal, Ocean, Neon, Grayscale, Park)
- **Sound Detective** — frequency-based detection of bird chirps, dog barks, bass/music, and loud transients, with emoji rain
- **Vibe Check** — ambient mood indicator (Peaceful, Lively, Musical, Chaotic)
- **Sound Snapshots** — capture the spectrograph as a PNG with watermark
- **Time Markers** — drop timestamped pins on the timeline, persisted across page loads
- **Zen Mode** — long-press to hide all UI
- **Low Power Mode** — reduces frame rate and disables effects for battery savings

## Architecture

Single-file vanilla JS app (`public/index.html`) with zero external dependencies. Uses Web Audio API, Canvas API, and getUserMedia. No framework, no build step, no server-side code.

```
public/
  index.html    # The entire app (~1465 lines)
render.yaml     # Render Static Site deployment config
DESIGN.md       # Design document
```

## Local Development

Just open `public/index.html` in a browser. For HTTPS (required for mic access on some browsers):

```bash
npx serve public
```

## Deploy to Render

1. Push to GitHub
2. Connect the repo in [Render](https://render.com)
3. Select **Static Site**
4. Set publish directory to `./public`
5. Or use the `render.yaml` blueprint for automatic configuration

The `render.yaml` configures CSP, X-Content-Type-Options, and X-Frame-Options headers.

## Controls

| Control | Action |
|---|---|
| Gear icon (top-right) | Open settings panel |
| Bottom bar buttons | Sound Detective, Vibe Check, Snapshot, Marker |
| Long-press canvas | Toggle Zen Mode |
| Spacebar | Pause/resume |
| Escape | Close settings panel |

## Settings

- **Palette** — Thermal, Ocean, Neon, Grayscale, Park
- **Gain** — microphone sensitivity (with amplitude indicator)
- **Speed** — scroll speed (1-6 pixels/frame)
- **Smoothing** — FFT temporal smoothing
- **Frequency Labels** — show Hz tick marks on Y-axis
- **Auto-Gain** — automatic sensitivity adjustment
- **Low Power** — 30fps, reduced FFT, no particles

All preferences are saved to localStorage.

## Browser Support

Requires a browser with Web Audio API and getUserMedia:
- Chrome 49+
- Firefox 36+
- Safari 14.1+
- Edge 79+

HTTPS required for microphone access (localhost is exempt).

## Privacy

- Microphone audio is processed entirely on-device
- No data is collected, transmitted, or stored server-side
- No cookies, tracking, or analytics
- localStorage stores only user preferences and time markers
