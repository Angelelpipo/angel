# Trading Live Viewer

A lightweight browser-based trading viewer for crypto pairs.

## Features
- Live polling of Binance 24h ticker endpoint
- Symbol selection (example: `BTCUSDT`, `ETHUSDT`)
- 1s / 2s / 5s refresh intervals
- Price sparkline-style chart rendered with canvas
- 24h change, 24h volume, and last update metadata

## Run locally
```bash
python -m http.server 8000
```
Then open <http://localhost:8000>.
