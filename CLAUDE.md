# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```powershell
npm start              # Launch Electron app
npm run build:win      # Build Windows NSIS installer
npm run build:win:dir  # Build unpacked Windows directory
node --check server.js # Check server.js syntax
```

## Project Overview

Mineradio is a Windows Electron desktop music player with: search/playback, playlists, lyrics stage, particle visualization, 3D playlist shelf, DIY visual console, weather radio, and GitHub auto-update.

**Fork note:** This is a fork of [XxHuberrr/Mineradio](https://github.com/XxHuberrr/Mineradio) with added local music library support (local folder scanning, ID3 parsing, cover cache, LRC lyrics).

## Branch Strategy

- `main` — Tracks upstream (XxHuberrr/Mineradio), clean original code only
- `feature-local-music` — All local music library changes live here

To sync upstream updates:
```bash
git checkout main && git pull upstream main && git push origin main
git checkout feature-local-music && git merge main
```

## Architecture (Big Picture)

### Single-File Frontend
Everything is in `public/index.html` (~27K lines, one monolithic file):

| Line range | Section |
|-----------|---------|
| 1-1884 | CSS (~1884 lines) |
| 1885-2732 | HTML structure |
| 2733-5550 | Core player: playback, queue, search, playlists |
| 5550-5735 | UI helpers (auto-hide, pointer, hits) |
| 5736-6459 | WebGL/Three.js: shaders (vertex + fragment) |
| 6460-7117 | Particle system |
| 7118-7259 | Webcam / effect layers |
| 7260-9418 | 3D playlist shelf — Three.js Canvas-rendered cards |
| 9419-12811 | Visual console: FX panel, DIY controls, beat analysis |
| 12812-13912 | Shelf manager class (rebuild, openContent, placeCard) |
| 13913-15000 | Content list manager (shelf detail panel, track rows) |
| 15001-20000 | Home page, search, playlists panel, lyrics, custom covers |
| 20001-end | Local music (cards, list, playback, progress bar) |

### Backend (server.js, ~4500 lines)
- **Raw Node.js HTTP server** (no Express)
- Route pattern: `if (pn === '/api/...')` sequential checks
- `sendJSON(res, data)` helper for all JSON responses
- `serveStatic(res, filePath)` for static files
- Audio proxy: `/api/audio?url=...` with Range support

### Electron Shell (desktop/)
- `main.js` — Window creation, IPC handlers, login windows, wallpaper/desktop lyrics windows
- `preload.js` — contextBridge exposing `desktopWindow` API

### Two Separate Playlist Systems
1. **`#playlist-panel`** (HTML DOM) — Left-sliding panel with "Playlists" / "Queue" tabs. Playlist cards rendered via `renderUserPlaylistsList()` into `#pl-list`.
2. **3D Shelf** (Three.js Canvas) — 3D floating card shelf on the right side. Cards built from `currentItems()` in the shelf manager, rendered onto Canvas textures via `drawCard()`.

Both systems now include the local music entry. Changes to one system DO NOT affect the other.

## Key Data Flows

### Playback
```
playQueueAt(idx)
  → URL resolution (Netease/QQ API or local direct stream)
  → audio.src = proxyAudioUrl
  → playAudio()
  → fetchLyric(song, token) or local lyric fetch
  → beginListenSession(song)
```

### Local Music APIs (all in server.js)
| Endpoint | Description |
|----------|------------|
| `POST /api/local-music/dir` | Set music directory |
| `POST /api/local-music/scan` | Background incremental scan (poll `/scan/status` for progress) |
| `GET /api/local-music/scan/status` | Scan progress (running/total/current/file) |
| `GET /api/local-music/list` | Full index |
| `GET /api/local-music/:id/stream` | Range 206 streaming |
| `GET /api/local-music/:id/cover` | Cached cover image |
| `GET /api/local-music/:id/lyric` | LRC lyric text |
| `PATCH /api/local-music/:id` | Update liked/playCount |

### Playlist Panel Rendering
```
switchPlaylistTab('playlists')
  → refreshUserPlaylists()
    → await loadLocalMusicIndex()
    → API fetch Netease + QQ playlists
    → renderUserPlaylistsList()
      → localMusicCardHtml() -- always first card
      → groups.map(playlistCardHtml) -- Netease + QQ
```

### Local Music Card Click → Expand
```
click .pl-card[data-playlist-provider="local"]
  → openPlaylistPanelDetail('local', 'local', title)
    → builds tracks from localMusicData.files
    → sets playlistPanelDetailState
    → renderPlaylistPanelDetailState()
    → localMusicCardHtml() + playlistPanelDetailHtmlLocal() renders expanded view
```

## Key Variables / State

| Variable | Location | Purpose |
|----------|----------|---------|
| `localMusicData` | index.html (global) | `{dir, files[]}` — current local music index |
| `localMusicIndexCache` | server.js | Server-side cached index read from disk |
| `localScanJob` | server.js | Async scan progress state |
| `playlistPanelDetailState` | index.html (global) | Current expanded playlist detail state |
| `currentLocalSong` | index.html (global) | Currently playing local song (standalone playback) |
| `playQueue` | index.html (global) | Current play queue array |
| `userPlaylists` | index.html (global) | Netease + QQ playlists combined array |
| `playlistCoverCache` | index.html (global) | 3D shelf cover image cache |

## Important Notes

- **Avoid modifying core playback functions** like `playQueueAt`, `switchPlaylistTab`, `renderUserPlaylistsList` — these are deeply coupled. Add new functions rather than altering existing ones.
- **3D Shelf uses Canvas textures** — `drawCard()` renders text/images onto a canvas, then creates a Three.js `CanvasTexture`. Changes to card appearance go in `drawCard()`.
- **Widevine/Electron quirks**: `mpg123-decoder` is bundled for MP3 decoding. Streaming uses Range requests. Audio proxy wraps URLs via `/api/audio?url=...`.
- **Cover images**: Empty cover strings are handled by `coverUrlWithSize()` and `requestPlaylistCover()` — both guard against falsy values.
- **The app has NO test suite** — validate changes by running `npm start` and testing key interactions manually.
