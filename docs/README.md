# MOSFET Scanner — web app

The browser UI for the [Arduino MOSFET Scanner](../README.md), served by GitHub
Pages from this folder. It replicates the Python tools (3-phase scan, live
charts, CSVs, guided bring-up) with no install and no build step.

**Use it:** open the Pages URL in Chrome, connect the Uno, run.
- **Desktop** (Chrome/Edge): connects directly via **Web Serial**.
- **Android**: plain Chrome can't open wired serial ports, so use the
  [shell app](../android/) — it loads this same page in a WebView and adds a
  native USB bridge this page auto-prefers. See
  [../how-to-install.md](../how-to-install.md).
- **Demo:** append `?demo` (or tap **Demo**) for a simulated rig — no hardware.

Features: scans save to in-app History (IndexedDB); CSVs download on demand with
the same columns/filenames as the Python tools; installable as a PWA (works
offline after first load); a wake lock keeps the screen on during scans.

## Dev

No build step — this folder *is* the source (vanilla ES modules, hand-rolled
canvas charts). Serve it locally:

```
python -m http.server 8123 --directory docs    # from the repo root
# http://localhost:8123/?demo
```

| file | role |
| --- | --- |
| `js/transport.js` | Web Serial + WebUSB-CDC + Android-bridge byte transports |
| `js/protocol.js`  | line protocol, command queue, `Rig` driver |
| `js/mock.js`      | simulated firmware + synthetic FET (demo/testing; `bench` selects socket contents) |
| `js/convert.js`   | dual-ref pick, measure-point math, CSV format |
| `js/scan.js`      | 3-phase cycle engine with abort + live callbacks |
| `js/bringup.js`   | wizard steps + limits |
| `js/chart.js`     | canvas line-family + leak-bar charts (viridis) |
| `js/store.js`     | IndexedDB history, CSV builders, params persistence |
| `js/app.js`       | UI wiring + service-worker registration |

Caveat: background tabs get timer-throttled by Chrome (~1 s per `setTimeout`), so
keep the app foregrounded during scans — the wake lock handles that on phones.

> **Deploying a rename:** the service worker and manifest use **relative** paths,
> so the app is repo-name-agnostic *except* the GitHub link in `index.html` and
> the `APP_URL` compiled into the [Android wrapper](../android/).
