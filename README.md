# Trading Live Viewer

A lightweight browser-based trading viewer for crypto pairs.

## Features
- Live polling with Binance primary source and Coinbase fallback
- Symbol selection (example: `BTCUSDT`, `ETHUSDT`)
- 1s / 2s / 5s refresh intervals
- Price sparkline-style chart rendered with canvas
- 24h change, 24h volume, and last update metadata

## Run locally
```bash
python -m http.server 8000
```
Then open <http://localhost:8000>.

## Capture a screenshot
Because this repository is pure static HTML, you can capture a UI screenshot from any browser without extra dependencies:
1. Start the local server (`python -m http.server 8000`).
2. Open <http://localhost:8000> in your browser.
3. Use your browser's built-in screenshot feature (or OS-level screenshot hotkey) and save the image.
