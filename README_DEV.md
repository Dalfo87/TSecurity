# Door Sensus Control Unit — README_DEV.md

## Project Overview
Single-file HTML web interface for the **Door Sensus Control Unit**, an embedded door monitoring and alarm management system. Runs on Tibbo TPP2-G2 hardware with severely limited resources.

Manufacturer: Aretes S.r.l. — Contact: dev@aretes.net

---

## File Structure
```
Frontend Tibbo/
├── README_DEV.md                      — This file
└── files/
    ├── DSControlUnit_UI.html          — The single deployable file (CSS + HTML + JS)
    └── DSControlUnit_extracted/       — Original Tibbo firmware HTML fragments (reference only)
        ├── html/                      — Legacy page fragments
        └── tpp_demo.css               — Original Tibbo CSS reference
```

---

## Architecture Decisions

### Single-File Design
**No build tools. No npm. No frameworks.** The file is served directly from the Tibbo embedded webserver's filesystem. This is intentional:
- The hardware cannot serve multiple files efficiently
- SRAM is shared between firmware and webserver — extremely limited (see Hardware Constraints)
- A single file eliminates TFTP multi-file upload complexity

### Vanilla JS Only
No React, Vue, Angular, or jQuery. No external CDN links. All logic is hand-written ES5-compatible JavaScript. Inline `onclick=` handlers are used throughout (preferred over `addEventListener` for performance on this embedded target).

### Logo
The logo is a `🔐` emoji used in both the topbar and the login page. **Do not replace with SVG or binary assets** — it costs only a few bytes.

### CSS Variables
All colors and sizes are defined in `:root {}` at the top of `<style>`. Modify theme colors there only, not scattered through the file.

### Notifications and Dialogs
- `toast(msg)` is a thin wrapper around the native `alert()` — **do not restore a custom toast component**
- `showConfirm(msg, cb)` uses the native `confirm()` — **do not restore a custom modal**
- These were simplified during the 30KB optimization and must stay minimal

---

## Hardware Constraints — Tibbo TPP2-G2 (REAL SPECS)
> Source: tibbo.com/store/tps/tpp2g2.html

- **CPU**: 32-bit architecture, TiOS, PLL software-controllable
- **SRAM total**: **66KB** — shared between firmware, application, webserver buffer, and heap
- **Flash filesystem**: 1MB dedicated (separate from SRAM — the HTML file lives here)
- **Network**: 10/100BaseT Ethernet **only** (no WiFi on TPP2-G2 — do not add WiFi config)
- **Max concurrent sessions**: 32 (UDP/TCP/HTTP)
- **Webserver**: Tibbo embedded httpd — buffers HTTP response in SRAM during transmission
- **Available SRAM for webserver**: estimated ~20–35KB after firmware + app
- **Gzip**: NOT supported by TiOS for HTML files — file size is uncompressed physical size
- **File size target**: Keep under **31KB** — current file is ~28.8KB
- **Do NOT add external dependencies** — they would require internet access which is never guaranteed

---

## Data Model

```js
DOOR_STATE_OPTS = [
  'Notifiche Disabilitate', 'Chiuso', 'Allarme Locale',
  'Allarme Remoto', 'Acquisito', 'Silenziato', 'Manutenzione'
]

STATE = {
  inputs: [                          // 9 inputs total (IN1–IN9)
    {
      name: string,                  // Display name (user editable)
      milestoneName: string,         // Milestone VMS channel binding name
      linkOutput: number,            // Index into STATE.outputs (0 = "Nessuno")
      doorState: number,             // Index into DOOR_STATE_OPTS
      inputType: 'NO'|'NC',          // Normally Open or Normally Closed
      alarmOnMuted: 'YES'|'NO',
      state: 'closed'|'open'|'alarm', // Runtime state (from device poll)
      nc: boolean                    // Legacy compat (mirrors inputType)
    }
  ],
  outputs: [                         // 6 outputs total (OUT1–OUT6)
    {
      name: string,                  // Display name
      on: boolean,                   // Current relay state
      type: 'NO'|'NC'               // Relay wiring type
    }
  ],
  logEntries: [                      // Rolling log (max 200)
    { ts: string, port: string, ev: string, cls: string }
  ],
  appType: string                    // 'AllFunction'|'DoorAlarms'|'Tvcc'|'ControFlusso'
}
```

---

## Pages and Their Purpose

| Page ID | Sidebar Section | Label | Purpose |
|---|---|---|---|
| `#p-ingressi` | Configurazione | Ingressi | Configure 9 digital inputs (name, type, alarm behavior, Milestone binding) |
| `#p-uscite` | Configurazione | Uscite | Configure and manually control 6 relay outputs |
| `#p-rete` | Configurazione | Rete | Ethernet DHCP, IP, subnet, gateway, NTP |
| `#p-integrazioni` | Configurazione | Integrazioni | **Merged**: Milestone XProtect VMS + Xovis people counter (two cards, one page) |
| `#p-manutenzione` | Sistema | Manutenzione | App type selector, system info, reboot, factory reset, password change + **Licenza** content (merged at bottom) |
| `#p-log` | Sistema | Log eventi | Rolling event log (last 100 events shown) |

> **Note**: `#p-milestone`, `#p-xovis`, and `#p-licenza` no longer exist as separate pages. Their content was merged into `#p-integrazioni` and `#p-manutenzione` respectively during the 30KB optimization.

---

## Key JavaScript Functions

| Function | Purpose |
|---|---|
| `renderInputsTable()` | Builds inline-editable config table for all 9 inputs |
| `saveInput(i)` | Per-row save handler for input config |
| `saveAllInputs()` | Global "Salva Tutti" handler for input config |
| `renderOutputsTable()` | Builds output relay table with ON/OFF/Impulso controls |
| `renderStatusBar()` | Updates the bottom real-time status bar (inputs + outputs) |
| `renderLog()` | Renders the event log table |
| `addLog(port, ev, cls)` | Appends entry to STATE.logEntries and re-renders log |
| `toggleOutput(i)` | Toggles relay on/off, logs event, re-renders outputs + status bar |
| `pulseOutput(i)` | Activates relay for 2 seconds (impulso), then auto-deactivates |
| `showPage(id, el)` | Navigates to page, updates active nav item, refreshes rendered pages |
| `initApp()` | Called after login: renders inputs table, outputs, status bar |
| `toggleDHCP(sel)` | Shows/hides manual IP fields based on DHCP selection |
| `toast(m)` | Wrapper for native `alert(m)` — keep minimal |
| `showConfirm(m, cb)` | Wrapper for native `confirm(m)` — keep minimal |

> **Removed**: `simulatePoll()` and its `setInterval` have been removed. The status bar is static until production polling is implemented.

---

## Status Bar (Bottom)
The bottom status bar (`#status-bar`) shows real-time state of all inputs and outputs. Populated by `renderStatusBar()`, called on login and on every toggle/pulse action.

**This bar MUST always remain visible. Never remove it.**

Input states: `closed` (green), `open` (red), `alarm` (red)
Output states: toggle switch (green = ON, gray = OFF)

---

## Polling / Simulation
`simulatePoll()` has been **removed** to save space. In production, implement an HTTP fetch to the device API (Tibbo BASIC endpoint returning JSON state). The fetch should only mutate `STATE.inputs[i].state` and call `renderStatusBar()` — it must NOT re-render the config table. Keep any polling implementation minimal.

---

## Login
- Demo password: `1234` (hardcoded as `DEMO_PW`)
- Empty password also accepted (no-password scenario)
- In production: replace `doLogin()` logic with actual API call

---

## How to Run / Test
1. Open `files/DSControlUnit_UI.html` directly in a browser (no server needed)
2. Enter password `1234` (or press Enter with empty field)
3. The UI runs fully offline with demo log data

## How to Deploy to Device
1. Connect to Tibbo device via TFTP
2. Upload `DSControlUnit_UI.html` as configured in the Tibbo BASIC firmware (typically `index.htm`)
3. Access via device IP in browser

---

## Integration Points

### Milestone XProtect VMS (in `#p-integrazioni`)
- Server: IP + Port (default: 1238) + Username + Password
- Each input can be mapped to a Milestone channel name via `milestoneName` field in STATE (configured in Ingressi table)
- Protocol: Milestone generic event ingestion API

### Xovis People Counter (in `#p-integrazioni`)
- REST API over HTTP (port 4321 default)
- Uses `LineCrossing` detection command
- Direction: forward / backward / both

### Application Types (set in Manutenzione)
- `AllFunction` — full door alarm + TVCC + flow control
- `DoorAlarms` — door alarms only
- `Tvcc` — TVCC/camera integration only
- `ControFlusso` — people flow control only

---

## Contributing Guidelines
1. **Do NOT introduce external dependencies** (no CDN, no npm, no framework)
2. Keep all changes within the single HTML file
3. Follow the CSS variable naming convention (`--bg-*`, `--text-*`, `--amber`, etc.)
4. Test in Chrome and Firefox
5. After changes, verify file size stays under **31KB** (`wc -c DSControlUnit_UI.html`)
6. Use `toast(msg)` for user feedback — it calls `alert()` internally, keep it that way
7. Use `showConfirm(msg, cb)` for destructive actions — it calls `confirm()` internally, keep it that way
8. All user-visible text in **Italian** (labels, toasts, page titles) except technical identifiers
9. Inline `onclick=` handlers are acceptable and preferred over `addEventListener`
10. Keep the status bar — it must always be present
11. No WiFi configuration — the TPP2-G2 hardware has no WiFi
12. **Do not add CSS animations, @keyframes, or custom scrollbars** — removed intentionally to save bytes
13. **Do not add badge components or custom toast/modal HTML** — removed intentionally
14. **Do not re-add SVG logo** — replaced with emoji `🔐` to save ~2KB

---

## Removed Sections (by design)
The following were present in earlier versions and intentionally removed:

### Removed pages / merged:
- **`#p-milestone`** — Merged into `#p-integrazioni` (Integrazioni page)
- **`#p-xovis`** — Merged into `#p-integrazioni` (Integrazioni page)
- **`#p-licenza`** — Merged into `#p-manutenzione` (bottom of Manutenzione page)

### Removed features:
- **Porte Seriali** — Modbus serial port configuration (not needed for DoorSensus)
- **Routing Modbus** — Modbus ID routing table (not needed for DoorSensus)
- **SNMP** — Network management protocol config (out of scope)
- **WiFi** — Hardware has no WiFi (TPP2-G2 is Ethernet only)
- **Milestone channel names** — Moved to per-input `milestoneName` field in Ingressi config
- **Firmware update** — Removed from Manutenzione UI (performed via TFTP only)
- **Tipo dispositivo** — Moved to Application Type selector in Manutenzione
- **simulatePoll / setInterval** — Demo simulation removed; status bar is static pending real polling
- **SVG logo** — Replaced with 🔐 emoji (~2KB saving)
- **Custom toast component** — Replaced with native `alert()`
- **Custom confirm modal** — Replaced with native `confirm()`
- **CSS animations (@keyframes)** — Removed (pulse-dot, blink, blink-light)
- **Badge/dot-indicator system** — Removed; output state shown as colored text
- **Scrollbar styling** — Removed; browser default used
- **Topbar badge + separator** — Removed; topbar is minimal (logo + title + IP + logout)

---

*Last updated: 2026-02-19*
