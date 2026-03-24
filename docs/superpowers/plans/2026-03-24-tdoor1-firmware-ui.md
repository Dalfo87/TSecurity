# TDoor_1 Firmware API + UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the event state machine (EVST), new HTTP API endpoints (`?e=io&action=get` with EVST, `?e=io&action=set_event`), TMaster settings, and update the embedded web UI with a Dashboard page and polling.

**Architecture:** TDoor_1 is a Tibbo TPP2-G2 field device. The firmware manages 8 inputs (Tibbit #54) and 6 relay outputs (Tibbit #03.1). A new 5-state event machine (Chiuso/Allarme/Silenziato/Manutenzione/Disabilitato) persists in EEPROM via a single `EVST` setting. The HTTP API exposes I/O state and event control. The UI is a single self-contained HTML file for config/admin.

**Tech Stack:** Tibbo BASIC (ES5-era JS for UI), TIDE IDE, no build tools, no test framework. Verification = compile in TIDE + manual API testing via curl/browser.

**Spec:** `docs/superpowers/specs/2026-03-24-tdoor1-firmware-ui-design.md`

---

## File Structure

### Firmware files (modify existing)

| File | Responsibility | Changes |
|---|---|---|
| `TDoor_1/settings.xtxt` | EEPROM settings descriptor | Add 4 new settings: EVST, TMIP, TMPT, TMKY |
| `TDoor_1/global.tbh` | All declarations and constants | Add dim/declare for EVST, TMIP, TMPT, TMKY, get_linked_output() |
| `TDoor_1/boot.tbs` | Boot sequence and initialization | Load new settings, restore outputs from EVST |
| `TDoor_1/io_manager.tbs` | I/O logic | Add get_linked_output(); rewrite evaluate_inputs() for EVST |
| `TDoor_1/device.tbs` | HTTP API handlers | Extend ?e=io with EVST; add set_event; fix callback_stg_post_set |

### UI file (modify existing)

| File | Responsibility | Changes |
|---|---|---|
| `files/DSControlUnit_UI.html` | Embedded web UI | Add Dashboard page, update polling to use EVST, add TMaster config in Integrazioni |

---

## Task 1: Add EEPROM settings (EVST, TMIP, TMPT, TMKY)

**Files:**
- Modify: `TDoor_1/settings.xtxt` (after line 32, before config defines)
- Modify: `TDoor_1/global.tbh` (after line 464, in schema v1 declarations block)

- [ ] **Step 1: Add 4 settings to settings.xtxt**

Append these 4 lines after line 32 (`>>APTP ...`) and before line 33 (`STG_DESCRIPTOR_FILE`):

```
>>EVST	E	S	1	0	8	A	00000000	Stato eventi ingressi
>>TMIP	E	S	1	0	16	A	^	TMaster IP
>>TMPT	E	S	1	0	5	A	6480	TMaster Porta
>>TMKY	E	S	1	0	32	A	^	TMaster Chiave Cifratura
```

Note: fields are tab-separated. The `A` column (member access) is required — matches existing entries. Default for EVST is `00000000` (8 zeros = all inputs Chiuso).

- [ ] **Step 2: Add RAM variable declarations in global.tbh**

After the existing schema v1 declarations (after `declare APTP as string(20)`, around line 464), add:

```basic
'--- EVST + TMaster settings (v1.2) ---
declare EVST as string(8)
declare TMIP as string(16)
declare TMPT as string(5)
declare TMKY as string(32)
```

- [ ] **Step 3: Declare get_linked_output function in global.tbh**

After the existing io_manager declarations (after `declare sub log_event(byref msg as string)`, around line 482), add:

```basic
declare function get_linked_output(idx as byte) as byte
'ritorna 0 se nessuna uscita collegata, 1-6 per OUT1..OUT6
'legge campo linkOutput (campo 4) dal CSV dell'ingresso idx
```

- [ ] **Step 4: Verify — compile in TIDE**

Open project in TIDE, compile. Expected: 0 errors (declarations only, no implementation yet).

- [ ] **Step 5: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/settings.xtxt TDoor_1/global.tbh
git commit -m "feat: add EVST, TMIP, TMPT, TMKY settings and declarations"
```

---

## Task 2: Implement get_linked_output() helper

**Files:**
- Modify: `TDoor_1/io_manager.tbs` (add new function after get_input_config, around line 93)

- [ ] **Step 1: Add get_linked_output function**

Insert after `get_input_config()` (after line 93 in io_manager.tbs):

```basic
function get_linked_output(idx as byte) as byte
    'Ritorna il numero di uscita collegata all'ingresso idx (0-based)
    'Ritorna 0 se nessuna uscita collegata, 1-6 per OUT1..OUT6
    dim cfg as string = get_input_config(idx)
    if cfg = "" then
        get_linked_output = 0
        exit function
    end if
    dim lo_str as string = get_csv_field(cfg, 4)  'campo linkOutput
    if lo_str = "" then
        get_linked_output = 0
        exit function
    end if
    dim lo_val as byte = val(lo_str)
    if lo_val > NUM_OUTPUTS then lo_val = 0
    get_linked_output = lo_val
end function
```

- [ ] **Step 2: Verify — compile in TIDE**

Expected: 0 errors. The function is callable but not yet called.

- [ ] **Step 3: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/io_manager.tbs
git commit -m "feat: add get_linked_output() helper in io_manager"
```

---

## Task 3: Load new settings at boot + EVST output restore

**Files:**
- Modify: `TDoor_1/boot.tbs` (after line 80, in schema v1 settings load block; after line 306, in output state init)

- [ ] **Step 1: Load EVST, TMIP, TMPT, TMKY at boot**

After line 80 (`APTP = stg_get("APTP", 0)`) add:

```basic
    'TMaster + EVST (v1.2)
    EVST = stg_get("EVST", 0)
    if len(EVST) < 8 then EVST = "00000000"   'protezione stringa corrotta
    TMIP = stg_get("TMIP", 0)
    TMPT = stg_get("TMPT", 0)
    TMKY = stg_get("TMKY", 0)
```

- [ ] **Step 2: Restore output state from EVST after output init**

Replace the output state init loop (lines 301-306):

```basic
    'Ripristino uscite da EVST (stati 1 o 2 = output ON)
    dim oi as byte
    for oi = 0 to NUM_OUTPUTS - 1
        out_state(oi) = false
        pulse_end_ms(oi) = 0
    next oi

    dim ei as byte
    dim evst_char as byte
    dim linked as byte
    for ei = 0 to NUM_INPUTS - 1
        evst_char = val(mid(EVST, ei + 1, 1))
        linked = get_linked_output(ei)
        if linked > 0 then
            if evst_char = 1 or evst_char = 2 then
                set_output(linked - 1, true)
            end if
        end if
    next ei
```

- [ ] **Step 3: Verify — compile in TIDE**

Expected: 0 errors. Boot now loads EVST and restores outputs.

- [ ] **Step 4: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/boot.tbs
git commit -m "feat: load EVST/TMaster settings at boot, restore outputs from EVST"
```

---

## Task 4: Rewrite evaluate_inputs() for EVST state machine

**Files:**
- Modify: `TDoor_1/io_manager.tbs` (replace evaluate_inputs, lines 169-216)

This is the most critical change. The current `evaluate_inputs()` unconditionally overwrites `in_state` and drives outputs. The new version must respect EVST states.

- [ ] **Step 1: Replace evaluate_inputs() with EVST-aware logic**

Replace the entire `evaluate_inputs()` subroutine (lines 169-216) with:

```basic
sub evaluate_inputs()
    dim i as byte
    dim cfg as string
    dim tipo as string
    dim is_active as boolean
    dim raw as boolean
    dim lo as byte
    dim evst_char as byte

    for i = 0 to NUM_INPUTS - 1
        evst_char = val(mid(EVST, i + 1, 1))

        '--- Skip se Manutenzione(3) o Disabilitato(4) ---
        if evst_char = 3 or evst_char = 4 then
            goto next_input
        end if

        '--- Leggi configurazione ---
        cfg = get_input_config(i)
        if cfg = "" then goto next_input

        tipo = get_csv_field(cfg, 1)   'NC o NO
        raw = get_input_state_raw(i)

        '--- Determina se input e' attivo ---
        if tipo = "NC" then
            is_active = raw   'NC: HIGH = aperto = anomalia
        else
            is_active = not raw  'NO: LOW = chiuso = evento
        end if

        '--- Aggiorna stato fisico per UI/debug ---
        if is_active then
            in_state(i) = 1
        else
            in_state(i) = 0
        end if

        '--- Evento aperto (1=Allarme, 2=Silenziato): non modificare EVST, non toccare output ---
        if evst_char = 1 or evst_char = 2 then
            goto next_input
        end if

        '--- Stato Chiuso(0): verifica se scatta allarme ---
        if evst_char = 0 and is_active then
            'Transizione 0 -> 1 (Allarme)
            mid(EVST, i + 1, 1) = "1"
            lo = get_linked_output(i)
            if lo > 0 then
                set_output(lo - 1, true)
            end if
            log_event("ALLARME ingresso " + str(i + 1))
            stg_set("EVST", 0, EVST)
            'TODO: notifica TMaster "ALLARME i"
        end if

        next_input:
    next i
end sub
```

**Key differences from original:**
- Skips inputs in Manutenzione/Disabilitato entirely
- Does NOT modify outputs for inputs in state 1 or 2 (event already open)
- Only triggers new alarms from state 0→1
- Writes EVST to EEPROM only on state transition
- Uses `mid(EVST, i+1, 1) = "1"` to update single char in EVST string

- [ ] **Step 2: Verify — compile in TIDE**

Expected: 0 errors. The evaluate_inputs now respects EVST.

- [ ] **Step 3: Manual test on board**

1. Upload firmware to TPP2-G2
2. All inputs at rest → `curl "http://DEVICE_IP/api.html?e=io&action=get&password=PW"` → `evst` should be `"00000000"`
3. Trigger IN1 physically → observe `in_state[0]=1` and `evst` changes to `"10000000"`
4. Release IN1 → `in_state[0]=0` but `evst` stays `"10000000"` (output remains ON)

- [ ] **Step 4: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/io_manager.tbs
git commit -m "feat: rewrite evaluate_inputs() with EVST state machine logic"
```

---

## Task 5: Extend ?e=io&action=get with EVST and timestamp

**Files:**
- Modify: `TDoor_1/device.tbs` (lines 1764-1785, existing io get handler)

- [ ] **Step 1: Update the io get response to include EVST and ts**

Replace the body of the existing `case "get":` block (lines 1764-1785) inside `select case action`. Keep the `case "get":` line, replace the body:

```basic
                case "get":
                    'Restituisce stato I/O + EVST + uptime
                    'dim j as byte — gia' dichiarato nello scope esterno, riusare
                    sock.setdata("{")
                    sock.setdata("\x22in\x22:[")
                    for j = 0 to NUM_INPUTS - 1
                        sock.setdata(str(in_state(j)))
                        if j < NUM_INPUTS - 1 then sock.setdata(",")
                    next j
                    sock.setdata("],")
                    sock.setdata("\x22out\x22:[")
                    for j = 0 to NUM_OUTPUTS - 1
                        if out_state(j) then
                            sock.setdata("1")
                        else
                            sock.setdata("0")
                        end if
                        if j < NUM_OUTPUTS - 1 then sock.setdata(",")
                    next j
                    sock.setdata("],\x22evst\x22:\x22" + EVST + "\x22,")
                    sock.setdata("\x22ts\x22:" + str(sys.timercountms \ 1000) + "}")
                    sock.send()
```

> **Note:** this replaces only the `case "get":` body within the existing `select case action` block. The `dim j as byte` already exists in the original `case "get":` at line 1767 — riusarlo, non ri-dichiararlo. Il `sys.timercountms` è la proprietà corretta per uptime su TPP2-G2 (usata ovunque nel codebase).

- [ ] **Step 2: Verify — compile in TIDE**

Expected: 0 errors.

- [ ] **Step 3: Manual test**

```
curl "http://DEVICE_IP/api.html?e=io&action=get&password=PW"
```
Expected response:
```json
{"in":[0,0,0,0,0,0,0,0],"out":[0,0,0,0,0,0],"evst":"00000000","ts":123}
```

- [ ] **Step 4: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/device.tbs
git commit -m "feat: extend ?e=io&action=get with EVST and uptime timestamp"
```

---

## Task 6: Add ?e=io&action=set_event endpoint

**Files:**
- Modify: `TDoor_1/device.tbs` (insert new action block after pulse_out, around line 1817)

- [ ] **Step 1: Add set_event action handler**

Insert as a new `case "set_event":` block inside the existing `select case action` — after `case "pulse_out":` (line 1817) and before `case else:` (line 1819). All `dim` statements must be placed at the top of the case block, before any `select case` or branching logic (Tibbo BASIC gotcha: `dim` inside `select case` branches is unreliable).

```basic
                case "set_event":
                    'Gestione stati evento ingresso
                    dim se_idx as byte
                    dim se_st as byte
                    dim se_resp as string
                    dim se_lo as byte
                    dim se_raw as boolean
                    dim se_cfg as string
                    dim se_tipo as string
                    dim se_lock as string
                    dim se_active as boolean
                    dim se_cur as byte

                    se_idx = val(http_server_find_param(params, "idx"))
                    se_st = val(http_server_find_param(params, "st"))

                    if se_idx > NUM_INPUTS - 1 then
                        se_resp = "{\x22ok\x22:0,\x22msg\x22:\x22idx fuori range\x22}"
                        goto se_send
                    end if

                    se_cur = val(mid(EVST, se_idx + 1, 1))

                    select case se_st
                    case 2:
                        '--- Silenzia: EVST[idx] = 2, output campo rimane ON ---
                        mid(EVST, se_idx + 1, 1) = "2"
                        stg_set("EVST", 0, EVST)
                        log_event("SILENZIATO ingresso " + str(se_idx + 1))
                        'TODO: notifica TMaster "SILENZIA se_idx"
                        se_resp = "{\x22ok\x22:1,\x22evst_idx\x22:2}"

                    case 0:
                        '--- Ack / Ripristina ---
                        '--- Se stato corrente e' Manutenzione(3) o Disabilitato(4): ripristino diretto ---
                        if se_cur = 3 or se_cur = 4 then
                            mid(EVST, se_idx + 1, 1) = "0"
                            stg_set("EVST", 0, EVST)
                            log_event("RIPRISTINATO ingresso " + str(se_idx + 1))
                            'TODO: notifica TMaster "CHIUSO se_idx"
                            se_resp = "{\x22ok\x22:1,\x22evst_idx\x22:0}"
                            goto se_send
                        end if

                        '--- Se stato corrente e' Allarme(1) o Silenziato(2): logica Ack con check input ---
                        se_raw = get_input_state_raw(se_idx)
                        se_cfg = get_input_config(se_idx)
                        se_tipo = get_csv_field(se_cfg, 1)
                        se_lock = get_csv_field(se_cfg, 6)

                        if se_tipo = "NC" then
                            se_active = se_raw
                        else
                            se_active = not se_raw
                        end if

                        if se_active = false or se_lock = "N" or se_lock = "" then
                            '--- Ack accettato: chiudi evento ---
                            mid(EVST, se_idx + 1, 1) = "0"
                            se_lo = get_linked_output(se_idx)
                            if se_lo > 0 then set_output(se_lo - 1, false)
                            stg_set("EVST", 0, EVST)
                            log_event("CHIUSO ingresso " + str(se_idx + 1))
                            'TODO: notifica TMaster "CHIUSO se_idx"
                            se_resp = "{\x22ok\x22:1,\x22evst_idx\x22:0}"
                        else
                            '--- Ack respinto: input ancora attivo, torna Allarme ---
                            mid(EVST, se_idx + 1, 1) = "1"
                            se_lo = get_linked_output(se_idx)
                            if se_lo > 0 then set_output(se_lo - 1, true)
                            stg_set("EVST", 0, EVST)
                            log_event("ACK RESPINTO ingresso " + str(se_idx + 1) + " (input attivo)")
                            'TODO: notifica TMaster "ALLARME se_idx"
                            se_resp = "{\x22ok\x22:0,\x22evst_idx\x22:1,\x22msg\x22:\x22input attivo\x22}"
                        end if

                    case 3:
                        '--- Manutenzione: output OFF ---
                        mid(EVST, se_idx + 1, 1) = "3"
                        se_lo = get_linked_output(se_idx)
                        if se_lo > 0 then set_output(se_lo - 1, false)
                        stg_set("EVST", 0, EVST)
                        log_event("MANUTENZIONE ingresso " + str(se_idx + 1))
                        'TODO: notifica TMaster "MANUTENZIONE se_idx"
                        se_resp = "{\x22ok\x22:1,\x22evst_idx\x22:3}"

                    case 4:
                        '--- Disabilita: solo admin, output OFF ---
                        'TODO: verificare password admin separata
                        mid(EVST, se_idx + 1, 1) = "4"
                        se_lo = get_linked_output(se_idx)
                        if se_lo > 0 then set_output(se_lo - 1, false)
                        stg_set("EVST", 0, EVST)
                        log_event("DISABILITATO ingresso " + str(se_idx + 1))
                        'TODO: notifica TMaster "DISABILITATO se_idx"
                        se_resp = "{\x22ok\x22:1,\x22evst_idx\x22:4}"

                    case else
                        se_resp = "{\x22ok\x22:0,\x22msg\x22:\x22st non valido\x22}"
                    end select

                    se_send:
                    sock.setdata(se_resp)
                    sock.send()
```

> **Key fixes vs original draft:**
> - Uses `case "set_event":` inside existing `select case action` (not standalone `if`)
> - All `dim` at top of case block (Tibbo BASIC gotcha: no `dim` inside `select case` branches)
> - **Ripristino da Manutenzione/Disabilitato**: `case 0` now checks `se_cur`: if current state is 3 or 4, does direct unconditional restore to 0 (bypasses lockOnActive check). This matches the spec: exiting Manutenzione/Disabilitato is always allowed.

- [ ] **Step 2: Verify — compile in TIDE**

Expected: 0 errors.

- [ ] **Step 3: Manual test — Silenzia**

```
curl "http://DEVICE_IP/api.html?e=io&action=set_event&idx=0&st=2&password=PW"
```
Expected: `{"ok":1,"evst_idx":2}`
Then: `curl "...?e=io&action=get..."` → `evst` = `"20000000"`

- [ ] **Step 4: Manual test — Ack con input a riposo**

```
curl "http://DEVICE_IP/api.html?e=io&action=set_event&idx=0&st=0&password=PW"
```
Expected: `{"ok":1,"evst_idx":0}` (input a riposo, lockOnActive=Y)

- [ ] **Step 5: Manual test — Manutenzione**

```
curl "http://DEVICE_IP/api.html?e=io&action=set_event&idx=0&st=3&password=PW"
```
Expected: `{"ok":1,"evst_idx":3}`, output goes OFF

- [ ] **Step 6: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/device.tbs
git commit -m "feat: add ?e=io&action=set_event endpoint (silenzia, ack, manutenzione, disabilita)"
```

---

## Task 7: Fix callback_stg_post_set for new settings

**Files:**
- Modify: `TDoor_1/device.tbs` (lines 62-89, callback_stg_post_set schema v1 block)

- [ ] **Step 1: Add RAM sync for EVST, TMIP, TMPT, TMKY**

After line 89 (`if stg_name_or_num = "APTP" then APTP = stg_value`) add:

```basic
    if stg_name_or_num = "EVST" then EVST = stg_value
    if stg_name_or_num = "TMIP" then TMIP = stg_value
    if stg_name_or_num = "TMPT" then TMPT = stg_value
    if stg_name_or_num = "TMKY" then TMKY = stg_value
```

- [ ] **Step 2: Add EVST, TMIP, TMPT, TMKY to ?e=v action=get variable list**

In the `?e=v action=get` select block (around lines 1496-1538), add cases after `APTP`:

```basic
                case "EVST": var_value = "\x22" + EVST + "\x22"
                case "TMIP": var_value = "\x22" + TMIP + "\x22"
                case "TMPT": var_value = "\x22" + TMPT + "\x22"
                case "TMKY": var_value = "\x22" + TMKY + "\x22"
```

- [ ] **Step 3: Add EVST, TMIP, TMPT, TMKY to ?e=v action=set**

In the `?e=v action=set` block (around lines 1561-1573), add after the APTP case:

```basic
                case "EVST","TMIP","TMPT","TMKY":
                    stg_set(var_name, 0, var_val)
```

- [ ] **Step 4: Add new settings to ?e=w action=get (bulk read)**

In the `?e=w` bulk response block (around lines 1577-1684), insert **before** `sock.setdata("}")` at line 1679. Use the same TX buffer overflow protection pattern as existing entries:

```basic
                    value = ",\x22EVST\x22:\x22" + EVST + "\x22"
                    value = value + ","
                    if sock.txfree < len(value) then
                        send_timeout = sys.timercountms + 2000
                        sock.send()
                        while (sock.txlen > 0) AND sys.timercountms < send_timeout
                        wend
                    end if
                    sock.setdata(value)

                    value = "\x22TMIP\x22:\x22" + TMIP + "\x22"
                    value = value + ","
                    if sock.txfree < len(value) then
                        send_timeout = sys.timercountms + 2000
                        sock.send()
                        while (sock.txlen > 0) AND sys.timercountms < send_timeout
                        wend
                    end if
                    sock.setdata(value)

                    value = "\x22TMPT\x22:\x22" + TMPT + "\x22"
                    value = value + ","
                    if sock.txfree < len(value) then
                        send_timeout = sys.timercountms + 2000
                        sock.send()
                        while (sock.txlen > 0) AND sys.timercountms < send_timeout
                        wend
                    end if
                    sock.setdata(value)

                    value = "\x22TMKY\x22:\x22" + TMKY + "\x22"
                    if sock.txfree < len(value) then
                        send_timeout = sys.timercountms + 2000
                        sock.send()
                        while (sock.txlen > 0) AND sys.timercountms < send_timeout
                        wend
                    end if
                    sock.setdata(value)
```

> **Note:** follows the exact same pattern as existing entries (e.g., `NETNM`, `NETGW`, `stg1`). The last entry (`TMKY`) omits the trailing comma since `}` follows. Insert point is immediately before `sock.setdata("}")` at line 1679.

- [ ] **Step 5: Verify — compile in TIDE**

Expected: 0 errors.

- [ ] **Step 6: Manual test — read/write TMIP via API**

```
curl "http://DEVICE_IP/api.html?e=v&action=set&name=TMIP&value=10.80.0.105&password=PW"
curl "http://DEVICE_IP/api.html?e=v&action=get&name=TMIP&password=PW"
```
Expected: `"10.80.0.105"` — verifica che il valore persista dopo reboot.

- [ ] **Step 7: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add TDoor_1/device.tbs
git commit -m "feat: add EVST/TMaster settings to stg_post_set, v/get, v/set, w/get"
```

---

## Task 8: UI — Add Dashboard page with polling

**Files:**
- Modify: `files/DSControlUnit_UI.html`

> **Prerequisite:** Read the current HTML file first. The file is ~1512 lines, single self-contained HTML. All changes must use existing conventions: inline `onclick=`, CSS `:root` variables, `toast()` for notifications, Italian text.

- [ ] **Step 1: Add CSS variables for event state colors**

Find the `:root` CSS block and add:

```css
--ev-chiuso: #4caf50;      /* verde */
--ev-allarme: #f44336;     /* rosso */
--ev-silenziato: #ff9800;  /* arancione */
--ev-manutenzione: #ffeb3b;/* giallo */
--ev-disabilitato: #9e9e9e;/* grigio */
```

- [ ] **Step 2: Add CSS for dashboard cards and blink animation**

Add in the `<style>` block:

```css
.dash-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-bottom: 16px; }
.dash-card { border: 2px solid var(--ev-chiuso); border-radius: 8px; padding: 12px; text-align: center; background: var(--surface); }
.dash-card .label { font-size: 13px; color: var(--text-secondary); }
.dash-card .state { font-size: 18px; font-weight: bold; margin-top: 4px; }
.dash-card.ev-0 { border-color: var(--ev-chiuso); }
.dash-card.ev-1 { border-color: var(--ev-allarme); animation: blink-alarm 0.8s infinite; }
.dash-card.ev-2 { border-color: var(--ev-silenziato); }
.dash-card.ev-3 { border-color: var(--ev-manutenzione); }
.dash-card.ev-4 { border-color: var(--ev-disabilitato); }
@keyframes blink-alarm { 0%,100% { background: var(--surface); } 50% { background: rgba(244,67,54,0.15); } }
.dash-actions { display: flex; gap: 6px; margin-top: 8px; justify-content: center; }
.dash-actions button { font-size: 11px; padding: 4px 8px; border-radius: 4px; cursor: pointer; border: 1px solid var(--border); }
```

- [ ] **Step 3: Add Dashboard page HTML**

Add a new page section in the main content area. Follow the existing page pattern (hidden div with id, shown via `showPage()`):

```html
<div id="page-dashboard" class="page" style="display:none">
  <h2>Dashboard</h2>
  <h3>Ingressi</h3>
  <div class="dash-grid" id="dash-inputs"></div>
  <h3>Uscite</h3>
  <div class="dash-grid" id="dash-outputs"></div>
</div>
```

- [ ] **Step 4: Add sidebar entry for Dashboard**

Add as first item in sidebar navigation:

```html
<li onclick="showPage('dashboard')" id="nav-dashboard">
  <span class="nav-icon">&#9673;</span> Dashboard
</li>
```

- [ ] **Step 5: Add JavaScript — Dashboard rendering and polling**

Add in the `<script>` block:

```javascript
var EVT_LABELS = ['Chiuso','Allarme','Silenziato','Manutenzione','Disabilitato'];
var EVT_COLORS = ['var(--ev-chiuso)','var(--ev-allarme)','var(--ev-silenziato)','var(--ev-manutenzione)','var(--ev-disabilitato)'];
var lastIoData = null;

function renderDashboard(data) {
  lastIoData = data;
  var inHtml = '';
  for (var i = 0; i < 8; i++) {
    var ev = parseInt(data.evst.charAt(i)) || 0;
    var phys = data['in'][i] ? 'HIGH' : 'LOW';
    var name = (STATE.inputs && STATE.inputs[i] && STATE.inputs[i].name) || ('IN' + (i+1));
    inHtml += '<div class="dash-card ev-' + ev + '">';
    inHtml += '<div class="label">' + name + '</div>';
    inHtml += '<div class="state">' + EVT_LABELS[ev] + '</div>';
    inHtml += '<div class="label">Fisico: ' + phys + '</div>';
    if (ev === 1) {
      inHtml += '<div class="dash-actions">';
      inHtml += '<button onclick="sendEvent(' + i + ',2)">Silenzia</button>';
      inHtml += '<button onclick="sendEvent(' + i + ',0)">Ack</button>';
      inHtml += '</div>';
    } else if (ev === 2) {
      inHtml += '<div class="dash-actions">';
      inHtml += '<button onclick="sendEvent(' + i + ',0)">Ack</button>';
      inHtml += '</div>';
    }
    if (ev !== 3 && ev !== 4) {
      inHtml += '<div class="dash-actions">';
      inHtml += '<button onclick="sendEvent(' + i + ',3)">Manutenzione</button>';
      inHtml += '<button onclick="sendEventAdmin(' + i + ',4)">Disabilita</button>';
      inHtml += '</div>';
    } else if (ev === 3) {
      inHtml += '<div class="dash-actions">';
      inHtml += '<button onclick="sendEvent(' + i + ',0)">Ripristina</button>';
      inHtml += '</div>';
    } else if (ev === 4) {
      inHtml += '<div class="dash-actions">';
      inHtml += '<button onclick="sendEventAdmin(' + i + ',0)">Riabilita (admin)</button>';
      inHtml += '</div>';
    }
    inHtml += '</div>';
  }
  document.getElementById('dash-inputs').innerHTML = inHtml;

  var outHtml = '';
  for (var j = 0; j < 6; j++) {
    var on = data.out[j];
    var oname = (STATE.outputs && STATE.outputs[j] && STATE.outputs[j].name) || ('OUT' + (j+1));
    outHtml += '<div class="dash-card" style="border-color:' + (on ? 'var(--ev-allarme)' : 'var(--ev-chiuso)') + '">';
    outHtml += '<div class="label">' + oname + '</div>';
    outHtml += '<div class="state">' + (on ? 'ON' : 'OFF') + '</div>';
    outHtml += '</div>';
  }
  document.getElementById('dash-outputs').innerHTML = outHtml;
}

function sendEvent(idx, st) {
  var msg = 'Confermi azione su ingresso ' + (idx+1) + '?';
  showConfirm(msg, function() {
    apiCall('io', 'set_event', 'idx=' + idx + '&st=' + st, function(resp) {
      if (resp && resp.ok === 1) {
        toast(EVT_LABELS[st] + ' — Ingresso ' + (idx+1));
      } else {
        toast((resp && resp.msg) || 'Errore', true);
      }
      pollIo();
    });
  });
}

function sendEventAdmin(idx, st) {
  var action = st === 4 ? 'DISABILITARE' : 'RIABILITARE';
  var msg = 'Azione admin: ' + action + ' ingresso ' + (idx+1) + '?';
  showConfirm(msg, function() {
    apiCall('io', 'set_event', 'idx=' + idx + '&st=' + st, function(resp) {
      if (resp && resp.ok === 1) {
        toast((st === 4 ? 'Disabilitato' : 'Riabilitato') + ' — Ingresso ' + (idx+1));
      } else {
        toast((resp && resp.msg) || 'Errore', true);
      }
      pollIo();
    });
  });
}
```

- [ ] **Step 6: Update the poll loop to call renderDashboard**

Find the existing poll function (likely `pollStatus` or similar timer) and modify it to:

```javascript
var pollTimer = null;
function pollIo() {
  apiCall('io', 'get', '', function(data) {
    if (data) {
      updateStatusBar(data);
      if (document.getElementById('page-dashboard').style.display !== 'none') {
        renderDashboard(data);
      }
    }
  });
}
function startPoll() { if (!pollTimer) pollTimer = setInterval(pollIo, 3000); }
function stopPoll() { if (pollTimer) { clearInterval(pollTimer); pollTimer = null; } }
```

Call `startPoll()` at page load (in the existing init function).

- [ ] **Step 7: Update status bar badge for open events**

In the `updateStatusBar(data)` function, add event badge logic:

```javascript
function updateStatusBar(data) {
  // ... existing status bar logic ...
  var openEvents = 0;
  if (data.evst) {
    for (var k = 0; k < data.evst.length; k++) {
      var c = parseInt(data.evst.charAt(k));
      if (c === 1 || c === 2) openEvents++;
    }
  }
  var badge = document.getElementById('status-event-badge');
  if (badge) {
    if (openEvents > 0) {
      badge.textContent = openEvents + ' evento/i attivo/i';
      badge.className = 'status-badge status-badge-alarm';
    } else {
      badge.textContent = 'Nessun evento';
      badge.className = 'status-badge status-badge-ok';
    }
  }
}
```

Add the badge element to the status bar HTML:

```html
<span id="status-event-badge" class="status-badge status-badge-ok">Nessun evento</span>
```

- [ ] **Step 8: Set Dashboard as default page**

In the init function, change the default page shown at load to `'dashboard'` instead of the current default.

- [ ] **Step 9: Verify — open in browser with mock data**

Open the HTML file directly in a browser. The Dashboard should render with 8 input cards (all "Chiuso"/green) and 6 output cards (all "OFF"). The poll will fail (no device) but the UI structure should be correct.

- [ ] **Step 10: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add files/DSControlUnit_UI.html
git commit -m "feat: add Dashboard page with event state polling and status bar badge"
```

---

## Task 9: UI — Add TMaster section in Integrazioni page

**Files:**
- Modify: `files/DSControlUnit_UI.html`

- [ ] **Step 1: Add TMaster fieldset in Integrazioni page**

Find the Integrazioni page section (contains Milestone VMS and Xovis). Add after the Xovis fieldset:

```html
<fieldset>
  <legend>TMaster</legend>
  <div class="form-row">
    <label>IP TMaster</label>
    <input type="text" id="cfg-tmip" maxlength="15" placeholder="10.80.0.105"
           onchange="saveSetting('TMIP', this.value)">
  </div>
  <div class="form-row">
    <label>Porta</label>
    <input type="text" id="cfg-tmpt" maxlength="5" placeholder="6480"
           onchange="saveSetting('TMPT', this.value)">
  </div>
  <div class="form-row">
    <label>Chiave cifratura</label>
    <input type="text" id="cfg-tmky" maxlength="32" placeholder="(non implementata)"
           onchange="saveSetting('TMKY', this.value)">
  </div>
</fieldset>
```

- [ ] **Step 2: Load TMaster settings on page open**

In the function that loads Integrazioni settings (called when the page is shown), add:

```javascript
apiCall('v', 'get', 'name=TMIP', function(d) { document.getElementById('cfg-tmip').value = d || ''; });
apiCall('v', 'get', 'name=TMPT', function(d) { document.getElementById('cfg-tmpt').value = d || ''; });
apiCall('v', 'get', 'name=TMKY', function(d) { document.getElementById('cfg-tmky').value = d || ''; });
```

- [ ] **Step 3: Verify — check page renders correctly**

Open in browser. The Integrazioni page should show three sections: Milestone VMS, Xovis, TMaster.

- [ ] **Step 4: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add files/DSControlUnit_UI.html
git commit -m "feat: add TMaster config section (IP, porta, chiave) in Integrazioni page"
```

---

## Task 10: UI — Add Manutenzione per-input toggles

**Files:**
- Modify: `files/DSControlUnit_UI.html`

- [ ] **Step 1: Add per-input maintenance toggles in Manutenzione page**

Find the Manutenzione page section. Add before the existing reboot/factory reset controls:

```html
<fieldset>
  <legend>Manutenzione Ingressi</legend>
  <div id="maint-inputs"></div>
</fieldset>
```

- [ ] **Step 2: Add JavaScript to render maintenance toggles**

```javascript
function renderMaintToggles(evst) {
  var html = '';
  for (var i = 0; i < 8; i++) {
    var ev = parseInt(evst.charAt(i)) || 0;
    var inMaint = (ev === 3);
    var name = (STATE.inputs && STATE.inputs[i] && STATE.inputs[i].name) || ('IN' + (i+1));
    html += '<div class="form-row">';
    html += '<label>' + name + '</label>';
    if (inMaint) {
      html += '<button onclick="sendEvent(' + i + ',0)" class="btn-warning">Ripristina</button>';
      html += ' <span class="tag tag-yellow">In manutenzione</span>';
    } else {
      html += '<button onclick="sendEvent(' + i + ',3)" class="btn-secondary">Metti in manutenzione</button>';
      html += ' <span class="tag">' + EVT_LABELS[ev] + '</span>';
    }
    html += '</div>';
  }
  document.getElementById('maint-inputs').innerHTML = html;
}
```

Call `renderMaintToggles(data.evst)` from the poll loop when Manutenzione page is visible.

- [ ] **Step 3: Verify — open Manutenzione page**

Should show 8 rows with input names and "Metti in manutenzione" buttons.

- [ ] **Step 4: Commit**

```bash
cd "/Users/bronc/Frontend Tibbo"
git add files/DSControlUnit_UI.html
git commit -m "feat: add per-input maintenance toggles in Manutenzione page"
```

---

## Task 11: Final integration test

- [ ] **Step 1: Full compile in TIDE**

Open project, clean build. Expected: 0 errors, 0 warnings (or only pre-existing warnings).

- [ ] **Step 2: Upload to TPP2-G2 and test full flow**

Test sequence:
1. Boot → verify EVST loaded correctly (check debug output)
2. `?e=io&action=get` → all inputs/outputs at rest, evst `"00000000"`
3. Trigger IN1 → evst becomes `"10000000"`, OUT linked goes ON
4. `?e=io&action=set_event&idx=0&st=2` → Silenzia, evst `"20000000"`, output stays ON
5. `?e=io&action=set_event&idx=0&st=0` (input a riposo) → Ack, evst `"00000000"`, output OFF
6. `?e=io&action=set_event&idx=0&st=3` → Manutenzione, evst `"00030000"`, verify input ignored
7. Reboot device → verify EVST persists and outputs restore correctly
8. Open UI in browser → Dashboard shows cards, poll works, status bar shows badge
9. Test Integrazioni → TMaster fields save/load correctly
10. Test Manutenzione → toggle buttons work

- [ ] **Step 3: Final commit with version bump**

Update `APP_VERSION` in global.tbh from `"1.1.0"` to `"1.2.0"`:

```bash
cd "/Users/bronc/Frontend Tibbo"
git add -A
git commit -m "feat: TDoor_1 v1.2.0 — EVST state machine, set_event API, Dashboard UI, TMaster config"
```

- [ ] **Step 4: Push to GitHub**

```bash
git push origin main
```
