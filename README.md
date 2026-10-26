# Command Center

A single-file, no-build web app: Today / Projects / Deadlines / Waiting on / Library / Decisions / Bots / Priorities / Settings.
Vanilla HTML, CSS and JS. Fonts are embedded, so `index.html` is fully self-contained.

**This repository is code only.** The data shown is labeled EXAMPLE DATA and is made up. No personal data, no tokens, no keys are stored here. Real data lives only in a different, private repo.

## Use it
- **Hosted (GitHub Pages):** `https://cosmopaxus.github.io/command-center/`
- **Local:** download `index.html` and double-click it (works from `file://`).
- **iPhone:** open the Pages URL in Safari > Share > Add to Home Screen. **Android:** Chrome menu > Install app.

Data is saved in that browser's localStorage on that device. **Optional sync:** Settings > Sync connects one JSON file in a separate **private** repo (fine-grained token pasted on each device, stored only in that browser). Steps and the bot instructions are in [SYNC.md](SYNC.md).

## Files
- `index.html` the whole app
- `manifest.webmanifest` home-screen install info (icons are embedded as data URIs)
- `SYNC.md` how to connect the private data repo, and how bots read/write it
- `.nojekyll` tells GitHub Pages to serve the files as-is
