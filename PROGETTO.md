# Door Sensus Control Unit — Documento di Progetto

> Documento di riferimento permanente. Contiene tutto ciò che serve per mantenere il focus
> sull'obiettivo senza dover rileggere chat, file sparsi o documentazione esterna.

---

## 1. Obiettivo

Costruire un'**interfaccia web embedded** per il dispositivo **Door Sensus Control Unit**, basato
su hardware Tibbo TPP2-G2. L'interfaccia permette di:

- Configurare e monitorare **9 ingressi digitali** (porte, sensori)
- Controllare **6 uscite relè**
- Configurare la rete Ethernet
- Integrare con Milestone VMS e Xovis people counter
- Visualizzare il log eventi in tempo reale
- Gestire il dispositivo (password, riavvio, licenza)

Il dispositivo è **Ethernet-only** (no WiFi). La UI gira direttamente sull'HTTP server del
firmware ed è l'unica interfaccia utente — non esiste app mobile né cloud.

---

## 2. Hardware — TPP2-G2

### Mainboard
- **Tibbo TPP2(G2)** — controller programmabile industriale
- CPU: processore Tibbo TiOS (BASIC-like)
- Connettività: **solo Ethernet** (RJ45), no WiFi integrato
- HTTP server integrato su porta 80

### Tibbits installati (moduli di espansione)
| Socket | Modulo | Tipo | Stato | Funzione |
|--------|--------|------|-------|----------|
| 1 | Tibbit 03.1 | Ingresso digitale isolato | **ATTIVO** | Ingressi 1–2 |
| 3 | Tibbit 03.1 | Ingresso digitale isolato | **ATTIVO** | Ingressi 3–4 |
| 5 | Tibbit 03.1 | Ingresso digitale isolato | **ATTIVO** | Ingressi 5–6 |
| 7 | Tibbit 54 | GPIO | Disabilitato | (previsto per ingressi 7–9) |
| 9 | Tibbit 54 | GPIO | Disabilitato | (previsto per uscite) |

> **Nota**: Il firmware attuale ha 6 ingressi attivi. L'obiettivo finale è 9 ingressi + 6 uscite,
> che richiede estensione del firmware con attivazione dei Tibbit 54.

### Vincoli hardware
- Nessun WiFi → nessun BLE, nessuna connettività mobile diretta
- Flash storage limitato → file UI ≤ **100KB** (attualmente ~29KB)
- Firmware in Tibbo BASIC (TiOS), file `.tbs`

---

## 3. AppBlocks — Ruolo nel progetto

### Cos'è AppBlocks
AppBlocks è la **piattaforma di sviluppo ufficiale Tibbo** composta da:

- **AppBlocks Designer (ABD)**: editor browser-based per creare firmware visualmente (flowchart,
  no-code). Alternativa a scrivere manualmente il codice Tibbo BASIC.
- **AppBlocks Cloud (ABC)**: piattaforma IoT cloud per monitoring remoto, OTA update,
  dashboard centralizzato. Richiede connettività internet dal dispositivo.
- **LUIS**: sistema di UI via Bluetooth Low Energy. Richiede modulo WA2000 (WiFi/BLE).
  Non applicabile al TPP2-G2 senza WiFi.

### Tutorial rilevanti per questo progetto (appblocks.io/tutorials)

| Tutorial | URL | Rilevanza | Note |
|----------|-----|-----------|------|
| **AC3: Door Sensor & Alarm** | `/tutorials/accesscontrol3` | ⭐⭐⭐ Massima | Door sensor + alarm relay + buzzer + disable_alarm button. Logica identica al DSControlUnit. |
| **AC6: Real Inputs** | `/tutorials/accesscontrol6` | ⭐⭐⭐ Massima | Tre modalità input: Standard (1s), Button (100x/s+debounce, max 8), Interrupt (hardware, solo 4a linea). Tibbit #54 = 4 dry contact optoisolati. |
| **Sprinkler 6: Multiple Zones** | `/tutorials/sprinkler6` | ⭐⭐⭐ Massima | **Pattern For-Next + Record Offset** per N zone/ingressi in parallelo senza che la ricerca si fermi al primo record. Modello diretto per il nostro loop su 9 ingressi → N uscite. |
| **AC2: Timer su Relay** | `/tutorials/accesscontrol2` | ⭐⭐ Alta | `_lock_activation_duration_`: relay attivo per durata definita. Pattern per impulso output (2s) e per "relay non disattivabile se ingresso ancora attivo". |
| **PWMLights 2: Custom Blocks** | `/tutorials/pwmlights2` | ⭐⭐ Alta | Custom Function in Tibbo BASIC — incapsula logica in funzioni riutilizzabili (es. `evaluate_inputs()`, `serialize_csv()`). Riduce complessità del codice firmware. |
| **AC5: Event Logging** | `/tutorials/accesscontrol5` | ⭐⭐ Alta | Logging eventi — allineato alla nostra pagina Log |
| **Sprinkler 5: Web Dashboard** | `/tutorials/sprinkler5` | ⭐ Bassa | Dashboard via LUIS (BLE) — non applicabile (no BLE su TPP2-G2). La nostra UI HTML sostituisce questo approccio. |
| **Sprinkler 3: LCD Status** | `/tutorials/sprinkler3` | ⭐ Bassa | LCD 320×240 px su TPS2L(G2) — non applicabile al TPP2-G2. Rilevante solo se si aggiunge display LCD Tibbit in futuro. |
| **AC1: Minimal System** | `/tutorials/accesscontrol1` | ⭐ Media | Wiegand reader + relay lock — base del sistema access control |
| **4G Switchover** | `/tutorials/4gswitchover` | ⭐ Bassa | Utile solo se si aggiunge Tibbit #45 per connettività 4G futura |

**Note chiave da AC6 — Tre modalità input:**
- **Standard Mode**: polling ogni 1s, usa "On Variable Changed". OK per sensori porta (risposta lenta accettabile).
- **Button Mode**: polling 100x/s con debouncing. Max 8 input (4 se LCD attivo). Per pulsanti fisici.
- **Interrupt Mode**: hardware interrupt, immediato. Solo su 4a linea IO di certi slot. Per rilevazione ultra-rapida.
- **Nostra scelta**: Standard Mode (1s) — sufficiente per sensori porte, compatibile con Tibbit 03.1.
- AC3 conferma: l'allarme si disabilita solo tramite comando esplicito dalla web dashboard.

**Pattern sprinkler6 applicato al nostro firmware (Tibbo BASIC):**
```basic
' Loop su tutti gli ingressi — nessuno viene saltato anche se più allarmi sono simultanei
for i = 0 to 8  ' 9 ingressi (0-based)
  call evaluate_single_input(i)  ' trova uscita associata, valuta, attiva/disattiva relè
next i
```

### Perché NON usiamo AppBlocks per questo progetto

| Aspetto | AppBlocks | Nostra scelta |
|---------|-----------|---------------|
| Firmware | ABD genera firmware (no-code) | Tdoor custom (Tibbo BASIC manuale) |
| UI | Web Interface Builder (BLE/WiFi) | HTML embedded su HTTP server |
| Cloud | ABC richiede connettività internet | Nessun cloud, tutto on-device |
| Controllo API | Limitato (endpoints fissi ABD) | Totale (definiamo noi l'API HTTP) |
| Offline-first | No (cloud-dipendente) | Sì (tutto on-device, LAN) |

**Conclusione**: AppBlocks è ottimo per nuovi progetti IoT cloud-connected. Per il DSControlUnit —
dispositivo Ethernet-only, UI embedded, API HTTP custom — il Tdoor custom con interfaccia HTML
proprietaria è la scelta corretta.

### AppBlocks Cloud — opzione futura
Se in futuro si volesse aggiungere monitoring remoto/OTA, sarebbe necessario:
1. Un **Tibbit 4G** (es. Tibbit 43) per connettività cellulare
2. Integrazione con AppBlocks Cloud (ABC)
3. Non richiede modifiche alla UI attuale

---

## 4. Firmware Tdoor — API Contract

Il firmware è in `/Users/bronc/Frontend Tibbo/Tdoor/` (Tibbo BASIC, TiOS).
Espone un HTTP server su porta 80 con endpoint su `api.html`.

### Autenticazione
```
GET api.html?e=ENDPOINT&action=ACTION&password=PASSWORD
```
- Setting `PW` memorizza la password (max 16 chars, default `^`)
- Ogni richiesta deve includere `?password=VALUE`
- Validazione: se risponde HTTP 200 → password corretta

### Endpoint esistenti (già implementati)

#### `e=w` — Batch read variabili
```
GET api.html?e=w&action=get&password=PW
→ {"DN":"nome","ON":"owner","NETCP":"1","NETIP":"...","NETNM":"...","NETGW":"...","stg1":"..."}
```

#### `e=v` — Read/Write singola variabile
```
GET api.html?e=v&action=get&variable=DN&password=PW
GET api.html?e=v&action=set&variable=DN&value=VALUE&password=PW
```

#### `e=i` — Info sistema
```
GET api.html?e=i&action=get&password=PW
→ {versione firmware, IP, MAC, uptime, timestamp}
```

#### `e=t` — Tabelle (LOG, DEBUG)
```
GET api.html?e=t&action=rows&table=LOG&offset=0&password=PW
GET api.html?e=t&action=count&table=LOG&password=PW
GET api.html?e=t&action=clear&table=LOG&password=PW
```
LOG table: max 200 record, campi: `timestamp` (DWORD Unix) + `value` (string 122 chars)

### Settings attualmente nel firmware

| Chiave | Max | Default | Uso |
|--------|-----|---------|-----|
| `DN` | 8 | — | Nome dispositivo |
| `ON` | 8 | — | Nome proprietario |
| `PW` | 16 | `^` | Password |
| `NETCP` | 1 | `1` | DHCP (0=statico, 1=DHCP) |
| `NETIP` | 16 | `10.80.0.10` | IP statico |
| `NETNM` | 16 | `255.255.255.0` | Netmask |
| `NETGW` | 16 | `10.80.0.1` | Gateway |
| `stg1` | 80 | — | Stato ingressi (da estendere) |

### Estensioni firmware da implementare

**`STG_MAX_NUM_SETTINGS = 40`** (33 usati + 7 riservati — over-provisioned per evitare future
schema change sulla costante stessa). Vedere tabella completa in §7.

#### Nuovi settings — Ingressi (9, posizioni 10–18)
Formato CSV: `"NomePorta,NC,1,NO,0,MilestoneName,N"`

| Campo | Valori | Significato |
|-------|--------|-------------|
| nome | string max 20 | Nome porta (es. "Porta1") |
| tipo | `NC` / `NO` | Normally Closed / Normally Open |
| doorState | `0`–`6` | Stato porta associato (0=disabilitato…6=manutenzione) |
| alarmOnMuted | `Y` / `N` | Allarme anche quando silenziato |
| linkOutput | `0`–`6` | Uscita associata (0=nessuna, 1–6=OUT corrispondente) |
| milestoneName | string max 20 | Nome canale Milestone VMS |
| **lockOnActive** | **`Y` / `N`** | **Se Y: il relè associato NON può essere disattivato finché l'ingresso è ancora attivo** |

> ⚠️ **lockOnActive** è un requisito di sicurezza: impedisce la disattivazione manuale del relè
> quando la condizione di allarme è ancora presente sull'ingresso fisico.

| Setting | Max len | Ingresso |
|---------|---------|---------|
| `IN01`…`IN09` | **80** | Ingressi 1–9 |

#### Nuovi settings — Uscite (6, posizioni 19–24)
Formato CSV: `"NomeUscita,NO"`

| Setting | Max len | Uscita |
|---------|---------|--------|
| `OUT1`…`OUT6` | **40** | Uscite 1–6 |

#### Nuovi settings — Integrazioni (posizioni 25–31)
| Setting | Max | Default | Contenuto |
|---------|-----|---------|-----------|
| `MVIP` | 20 | — | Milestone VMS IP |
| `MVPT` | 6 | `1238` | Milestone porta |
| `MVUN` | 32 | — | Milestone username |
| `MVPW` | 32 | — | Milestone password |
| `XVPT` | 6 | `4321` | Xovis porta |
| `XVCM` | 32 | `LineCrossing` | Xovis comando |
| `XVDI` | 10 | `forward` | Xovis direzione (forward/backward/both) |

#### Nuovi settings — Sistema (posizioni 32–33)
| Setting | Max | Default | Contenuto |
|---------|-----|---------|-----------|
| `NTPIP` | 20 | `192.168.0.3` | Server NTP (ora hardcoded) |
| `APTP` | 20 | `AllFunction` | Tipo app (AllFunction/DoorAlarms/Tvcc/ControFlusso) |

### Endpoint da aggiungere al firmware

#### `e=io` — Stato I/O in tempo reale
```
GET api.html?e=io&action=get&password=PW
→ {"in":[0,1,0,0,0,1,0,0,0],"out":[0,0,0,0,0,0]}
  valori: 0=chiuso/OFF, 1=aperto/ON, 2=allarme
```

#### `e=io` — Controllo uscite
```
GET api.html?e=io&action=set_out&idx=N&val=0/1&password=PW
GET api.html?e=io&action=pulse_out&idx=N&password=PW   (impulso 2s)
```

---

## 5. UI — DSControlUnit_UI.html

### File
`/Users/bronc/Frontend Tibbo/files/DSControlUnit_UI.html`

### Stato attuale
- **100% mock** — nessuna chiamata HTTP reale
- Login: password hardcoded `'1234'`
- Tutti i "Salva" sono `alert()` cosmetici
- Dati: hardcoded nello `STATE` object
- Dimensione: ~29KB (budget: 100KB)

### Vincoli tecnici (NON derogabili)
- Vanilla JavaScript **ES5-compatible** (no ES6+, no arrow functions, no const/let)
- **Single file** — nessun file esterno (CSS, JS, immagini → tutto inline)
- **No build tool** — no npm, no webpack, no framework
- Tutti i testi in **italiano**
- Colori CSS via variabili `:root` (mai valori hardcoded)
- `toast(msg, isError?)` per messaggi — **mai `alert()`**
- `showConfirm(msg, cb)` per conferme — **mai `confirm()`**
- STATUS BAR in fondo — **non rimuovere mai**

### Struttura pagine (sidebar)
```
Configurazione
├── Ingressi     → tabella 9 ingressi con config completa
├── Uscite       → tabella 6 relè con toggle/impulso
├── Rete         → IP, DHCP, gateway, DNS, NTP
└── Integrazioni → Milestone VMS + Xovis

Sistema
├── Manutenzione → tipo app, riavvio, reset, password, licenza
└── Log eventi   → tabella 100 eventi con timestamp
```

### STATUS BAR (bottom)
Mostra in tempo reale lo stato di tutti gli ingressi e uscite.
Deve aggiornarsi ogni 3 secondi via polling `?e=io`.

### CSS — Variabili root chiave
```css
--bg-deep: #0b0e14    /* sfondo principale */
--bg-panel: #111520   /* sidebar, topbar, status bar */
--amber: #f0a030      /* colore brand / accent */
--green: #28c76f      /* stato OK / chiuso / ON */
--red: #ea4a5a        /* stato allarme / aperto / errore */
--sidebar-w: 210px
--top-h: 52px
--status-h: 82px
```

### Obiettivo finale UI
Login → fetch config dal device → popola tutti i form → polling I/O ogni 3s → operazioni di
salvataggio scrivono sulla vera API firmware.

---

## 6. Piano di integrazione — Fasi

### Fase 1 — Fondamenta
**Firmware:**
1. `settings.xtxt`: `STG_MAX_NUM_SETTINGS` → 40, aggiungere tutti i nuovi settings + `SCHV`
2. `device.tbs`: aggiungere handler per `?e=io` (get, set_out, pulse_out)

**UI:**
3. Implementare `toast()` e `showConfirm()` inline (eliminare `alert()`/`confirm()` nativi)
4. Login reale: `fetch('api.html?e=w&action=get&password='+pw)` → 200 = valido
5. Sessione password: variabile `var PW_SESSION = ''` usata in tutte le chiamate successive

### Fase 2 — Config persistence
6. Pagina **Rete**: load da `?e=w`, save via `?e=v&action=set` per ogni campo
7. Pagina **Manutenzione**: PW, DN (max 8), ON (max 8), APTP, riavvio/reset
8. Pagina **Ingressi**: load/parse IN01-IN09 da CSV → save serializzato
9. Pagina **Uscite**: load/parse OUT1-OUT6 da CSV → save serializzato
10. Pagina **Integrazioni**: load/save MVIP/MVPT/MVUN/MVPW/XVPT/XVCM/XVDI

### Fase 3 — Real-time
11. `setInterval` ogni 3s → `?e=io&action=get` → aggiorna STATUS BAR
12. Toggle uscite → `?e=io&action=set_out`
13. Impulso uscite → `?e=io&action=pulse_out`
14. Pagina **Log**: load da `?e=t&action=rows&table=LOG`, cancella con `action=clear`
15. Indicatore connessione in topbar (punto verde/rosso con testo `ONLINE`/`OFFLINE`)

---

## 7. Decisioni architetturali

| Decisione | Scelta | Motivo |
|-----------|--------|--------|
| UI tecnologia | Vanilla JS ES5, single HTML | Device ha RAM e flash limitati; no build environment |
| Firmware tool | Tdoor custom (Tibbo BASIC manuale) | Controllo totale sull'API HTTP esposta |
| AppBlocks ABD | Non usato | ABD genera API fissa; noi necessitiamo endpoint custom `?e=io` |
| AppBlocks Cloud | Non usato | TPP2-G2 è Ethernet-only, nessuna connettività internet diretta |
| LUIS (BLE UI) | Non usato | Richiede modulo WA2000 non installato |
| File singolo | Sì, ≤100KB | Serve file servibile direttamente dall'HTTP server del firmware |
| Lingua UI | Italiano | Prodotto destinato a mercato italiano |

### EEPROM — Strategia scrittura (wear prevention)

L'EEPROM del TPP2-G2 ha cicli di scrittura limitati (~100.000 per cella).
**Regola fondamentale: scrivere in EEPROM solo dati che cambiano raramente (configurazione),
mai dati che cambiano frequentemente (stato runtime).**

**✅ EEPROM — Configurazione (scrittura rara, solo da admin):**
- Config rete: `NETCP`, `NETIP`, `NETNM`, `NETGW`
- Credenziali: `PW`
- Identità: `DN`, `ON`, `APTP`
- Config ingressi: `IN01`–`IN09` (nome, tipo, settings allarme)
- Config uscite: `OUT1`–`OUT6` (nome, tipo)
- Integrazioni: `MVIP`, `MVPT`, `MVUN`, `MVPW`, `XVPT`, `XVCM`, `XVDI`
- NTP: `NTPIP`

**❌ NON in EEPROM — Stato runtime (cambiano ad ogni evento):**
- Stato ingressi → RAM: letti direttamente dai pin hardware al boot e ad ogni poll
- Stato uscite → RAM: boot safe = tutte OFF (vedi sezione sotto)

**📋 Log eventi → TABLE library (flash separata, nessun impatto su EEPROM)**

### Schema change — Strategia per dispositivi in campo

Tibbo valida ogni setting confrontando nome + tipo + max_length con lo schema corrente.
Un setting "invalido" viene **resettato al default di fabbrica** — inclusi IP e password.
**Una scheda in campo con IP resettato diventa irraggiungibile.**

#### Le 4 regole obbligatorie (da non violare mai)

**Regola 1 — APPEND-ONLY**
Aggiungere nuovi settings SOLO in fondo alla lista. Mai rimuovere, mai riordinare.
Se un setting diventa obsoleto → lasciarlo con commento `[DEPRECATED]`, non eliminarlo.
Rimuovere o riordinare può invalidare a cascata tutti i settings successivi.

**Regola 2 — MAX_LENGTH: solo aumentare, mai diminuire**
Definire max_length generosi fin dal giorno 0 (over-provisioning).
Ridurre la max_length di un setting esistente causa schema change su quel setting.

**Regola 3 — MAI modificare nome, tipo o posizione di un setting esistente**
Qualsiasi modifica a nome o tipo di un setting esistente lo invalida e lo resetta.

**Regola 4 — Setting `SCHV` (Schema Version)**
Il setting in posizione 9 (`SCHV`) traccia la versione dello schema.
In `boot.tbs`, dopo l'init, confrontare `SCHV` con la versione corrente del firmware.
Se diverge → i nuovi settings sono già a default (Tibbo li ha inizializzati),
i settings della baseline (1–8) sono intatti. Eseguire eventuale logica di migrazione.

```basic
' boot.tbs — dopo init settings:
if stg_get("SCHV") <> "2" then
  ' Nuovi settings già inizializzati da Tibbo ai loro default
  ' Eventuali migrazioni dati custom qui...
  call stg_set("SCHV", "2")
end if
```

#### Tabella settings — baseline CONGELATA + nuovi settings

| Pos | Key | Tipo | Max len | Default | Stato |
|-----|-----|------|---------|---------|-------|
| 1 | `DN` | String | 8 | — | **FROZEN** |
| 2 | `ON` | String | 8 | — | **FROZEN** |
| 3 | `PW` | String | 16 | `^` | **FROZEN** |
| 4 | `NETCP` | Float | 1 | `1` | **FROZEN** |
| 5 | `NETIP` | String | 16 | `10.80.0.10` | **FROZEN** |
| 6 | `NETNM` | String | 16 | `255.255.255.0` | **FROZEN** |
| 7 | `NETGW` | String | 16 | `10.80.0.1` | **FROZEN** |
| 8 | `stg1` | String | 80 | — | **FROZEN** |
| 9 | `SCHV` | String | 4 | `1` | Schema version — âncora di migrazione |
| 10–18 | `IN01`–`IN09` | String | 80 | — | Config ingressi CSV |
| 19–24 | `OUT1`–`OUT6` | String | 40 | — | Config uscite CSV |
| 25 | `MVIP` | String | 20 | — | Milestone VMS IP |
| 26 | `MVPT` | String | 6 | `1238` | Milestone porta |
| 27 | `MVUN` | String | 32 | — | Milestone username |
| 28 | `MVPW` | String | 32 | — | Milestone password |
| 29 | `XVPT` | String | 6 | `4321` | Xovis porta |
| 30 | `XVCM` | String | 32 | `LineCrossing` | Xovis comando |
| 31 | `XVDI` | String | 10 | `forward` | Xovis direzione |
| 32 | `NTPIP` | String | 20 | `192.168.0.3` | NTP server |
| 33 | `APTP` | String | 20 | `AllFunction` | Tipo applicazione |

**`STG_MAX_NUM_SETTINGS = 40`** — over-provisioned a 40 (33 usati + 7 riservati per future
estensioni) per evitare di dover modificare questa costante e causare ulteriori schema change.

### Boot-time I/O check — DECISIONE CRITICA PER SECURITY

Il sistema è un'applicazione di **security**. Al reboot il comportamento corretto è:

```
Boot
  ↓
Init hardware, rete, settings EEPROM
  ↓
⚡ CHECK IMMEDIATO tutti gli ingressi  ← stessa funzione del polling ciclico
  ↓
Porta aperta o in allarme? → triggera SUBITO l'allarme
Tutto chiuso?              → stato normale
  ↓
Avvia polling ciclico ogni 1s
```

**Motivazione:** Se un allarme è attivo e il dispositivo si riavvia (crash, power cycle,
manutenzione), al boot deve rivalutare immediatamente lo stato reale delle porte.
NON si salva lo stato in EEPROM — si legge dall'hardware.

**Vantaggi rispetto a salvataggio EEPROM:**
- Zero EEPROM wear per stati che cambiano frequentemente
- Stato sempre accurato: riflette hardware fisico, non uno snapshot potenzialmente stale
- Comportamento deterministico e verificabile
- Se porta era aperta durante il reboot → allarme immediato, sempre

**Implementazione `boot.tbs`:**
```basic
' ... fine sequenza init ...
call evaluate_inputs()     ' ← CHECK IMMEDIATO al boot
call start_polling_timer() ' ← poi avvia il polling ciclico
```

### Uscite — safe state al boot = OFF

Tutte le uscite partono **OFF al boot** (nessuna persistenza EEPROM per lo stato).
Se un relè era attivo per un allarme prima del crash → il boot-time check degli ingressi
lo ri-attiverà immediatamente se la condizione di allarme è ancora presente.
Comportamento: prevedibile, sicuro, senza EEPROM wear.

### Mappatura ingressi → uscite: N:M, non 1:1

Ogni ingresso può essere associato a **qualsiasi** uscita tramite il campo `linkOutput` nel suo
setting `IN0X`. Non è una mappatura 1:1 fissa: l'ingresso 3 può pilotare l'uscita 5, l'ingresso 7
può non avere uscita, due ingressi possono pilotare la stessa uscita.

**Logica firmware:**
```basic
for i = 0 to 8
  linked_out = get_csv_field(stg_get("IN0"+str(i+1)), 5)  ' campo linkOutput
  if linked_out > 0 and input_state(i) = ALARM then
    call set_output(linked_out - 1, ON)
  end if
next i
```

### Sicurezza: lockOnActive — relay bloccato se ingresso ancora attivo

Se il campo `lockOnActive` dell'ingresso è `Y`, il firmware **rifiuta** il comando di spegnimento
del relè associato finché l'ingresso fisico è ancora in stato attivo/allarme.

**Logica firmware (endpoint `?e=io&action=set_out`):**
```basic
' Prima di eseguire OFF su un'uscita:
for i = 0 to 8
  if get_csv_field(stg_get("IN0"+str(i+1)), 5) = requested_output then  ' linkOutput corrisponde
    if get_csv_field(stg_get("IN0"+str(i+1)), 7) = "Y" then             ' lockOnActive = Y
      if input_state(i) = ACTIVE then
        ' BLOCCA: restituisce errore HTTP 409 o JSON {"error":"locked"}
        exit sub
      end if
    end if
  end if
next i
```

**Lato UI:** il pulsante OFF dell'uscita deve essere **disabilitato/grigio** con tooltip
"Relè bloccato: ingresso ancora attivo" quando la condizione è presente.

### Timezone — configurazione orologio

⚠️ **Per il corretto funzionamento dei timestamp nel LOG**, configurare il **Time Zone** nelle
proprietà del dispositivo Tibbo (General page → Features tab → Time Zone property).
Senza questo, tutti i timestamp del log saranno in UTC e potrebbero risultare sfasati.
Questo va fatto prima del deploy in campo per ogni dispositivo installato.

### Settings survival: factory reset vs schema change

Questi sono due scenari distinti:

**Schema change accidentale** (nuovo firmware con nuovi settings):
→ Gestito dalle 4 regole APPEND-ONLY + SCHV. I settings FROZEN (1–8) non vengono mai toccati.
→ I nuovi settings vengono inizializzati ai loro default. Nessun danno ai settings di rete.

**Factory reset intenzionale** (utente preme "Reset fabbrica"):
→ TUTTO viene resettato ai default, inclusi IP e password.
→ Questo è il comportamento CORRETTO e atteso per un factory reset.
→ L'utente deve essere consapevole che dopo un factory reset il device torna a `10.80.0.10`
   e la password torna a `^`.
→ **La UI deve mostrare un warning esplicito** prima del factory reset: "Il dispositivo tornerà
   all'indirizzo IP di default 10.80.0.10. Assicurarsi di poterlo raggiungere a quell'indirizzo."

**Settings che NON devono mai essere inizializzati da schema change (FROZEN 1–8):**
Garantito dalla strategia APPEND-ONLY. I settings FROZEN non cambiano mai di posizione/tipo/nome.

---

## 8. Architettura applicativa — Monolitica vs Sub-applicazioni

### Domanda aperta
Conviene creare sub-applicazioni separate (TDoor, TFlux, TControl) selezionabili al boot,
oppure fare tutto in un firmware unico con selezione via `APTP`?

### Analisi

| Aspetto | Monolitica (APTP) | Sub-applicazioni separate |
|---------|-------------------|--------------------------|
| Codebase | Uno solo da mantenere | N codebase separati |
| Feature creep | Rischio di mischiare logiche diverse | Ogni app è focalizzata |
| Dimensione firmware | Cresce con le features | Ogni app è compatta |
| Deploy in campo | Un solo file firmware | N file firmware diversi |
| Bug isolation | Un bug può impattare tutte le modalità | Bug confinato all'app |
| Riuso codice | Facile (tutto nello stesso file) | Richiede librerie condivise |
| Evoluzione | Facile splitare dopo se necessario | Difficile riunire dopo |

### Decisione consigliata: **Monolitica con APTP, codice modulare**

**Fase attuale**: architettura monolitica con setting `APTP` che disabilita al boot le
funzionalità non necessarie per la modalità selezionata. Il codice viene strutturato in
**moduli separati** (file `.tbs` distinti per funzione):

```
firmware/
├── boot.tbs          ← init, network, settings, boot-check I/O
├── main.tbs          ← loop principale, polling timer
├── device.tbs        ← HTTP API handlers
├── io_manager.tbs    ← gestione ingressi/uscite (evaluate_inputs, set_output)
├── alarm.tbs         ← logica allarme (attivazione, lock, silenziamento)
├── integration.tbs   ← Milestone VMS + Xovis
└── global.tbh        ← costanti, variabili globali, enum
```

**Se in futuro** il firmware diventa troppo grande o le modalità troppo diverse:
→ Splittare in sub-applicazioni usando i moduli come base (ogni app include solo i moduli necessari).
→ Le librerie condivise (io_manager, alarm) diventano file comuni includibili.

### Modalità applicative previste via APTP

| Valore APTP | Nome | Funzionalità attive |
|-------------|------|---------------------|
| `AllFunction` | Tutto attivo | Ingressi + Uscite + Milestone + Xovis |
| `DoorAlarms` | Solo allarmi porte | Ingressi + Uscite (no integrazioni esterne) |
| `Tvcc` | Solo TVCC | Ingressi + Milestone VMS (no Xovis) |
| `ControFlusso` | Conteggio persone | Ingressi + Xovis (no Milestone) |

---

## 9. File e percorsi chiave

| File | Percorso | Descrizione |
|------|----------|-------------|
| UI principale | `/Users/bronc/Frontend Tibbo/files/DSControlUnit_UI.html` | Single-file web UI |
| Firmware dir | `/Users/bronc/Frontend Tibbo/Tdoor/` | Sorgenti Tibbo BASIC |
| Settings FW | `/Users/bronc/Frontend Tibbo/Tdoor/settings.xtxt` | Definizione settings EEPROM |
| Tables FW | `/Users/bronc/Frontend Tibbo/Tdoor/tables.xtxt` | Definizione tabelle (LOG, DEBUG) |
| Main FW | `/Users/bronc/Frontend Tibbo/Tdoor/main.tbs` | Entry point firmware |
| Device FW | `/Users/bronc/Frontend Tibbo/Tdoor/device.tbs` | Handler HTTP API |
| Boot FW | `/Users/bronc/Frontend Tibbo/Tdoor/boot.tbs` | Inizializzazione (rete, password) |
| Global FW | `/Users/bronc/Frontend Tibbo/Tdoor/global.tbh` | Costanti e variabili globali |
| Doc sviluppatori | `/Users/bronc/Frontend Tibbo/README_DEV.md` | Istruzioni per sviluppatori |
| Piano integrazione | `.claude/projects/.../memory/project_integration_plan.md` | Roadmap tecnica dettagliata |

---

## 10. Glossario

| Termine | Significato |
|---------|-------------|
| **TPP2-G2** | Tibbo Programmable Platform 2 (Gen 2) — la mainboard hardware |
| **TiOS** | Sistema operativo Tibbo — gira sul TPP2-G2 |
| **Tibbit** | Modulo di espansione I/O (es. relè, ingressi digitali, 4G) |
| **Tdoor** | Nome del progetto firmware ufficiale Tibbo (base per DSControlUnit) |
| **TiBasic / `.tbs`** | Linguaggio Tibbo BASIC — usato per programmare il firmware |
| **ABD** | AppBlocks Designer — IDE visual no-code ufficiale Tibbo |
| **ABC** | AppBlocks Cloud — piattaforma IoT cloud Tibbo |
| **LUIS** | Loadable User Interface System — UI via BLE (non usata in questo progetto) |
| **`?e=io`** | Endpoint HTTP da aggiungere al firmware per stati I/O in tempo reale |
| **`STG_MAX_NUM_SETTINGS`** | Costante firmware che limita il numero di settings — impostare a **40** (over-provisioned) |
| **`IN01`–`IN09`** | Chiavi settings firmware per configurazione ingressi (formato CSV) |
| **`OUT1`–`OUT6`** | Chiavi settings firmware per configurazione uscite (formato CSV) |
| **STATUS BAR** | Barra in fondo alla UI che mostra stato live di ingressi e uscite |
| **mock** | Stato attuale della UI: dati hardcoded, nessuna chiamata HTTP reale |
| **Milestone VMS** | Video Management System per integrazione telecamere |
| **Xovis** | People counter (conta persone) da integrare via API |
| **lockOnActive** | Campo CSV in IN0X: se Y, impedisce spegnimento relè finché ingresso è attivo |
| **APTP** | Setting che seleziona la modalità applicativa al boot (AllFunction, DoorAlarms, Tvcc, ControFlusso) |
| **Standard Mode** | Modalità input: polling ogni 1s. Usata per sensori porte. |
| **Button Mode** | Modalità input: polling 100x/s con debounce. Per pulsanti fisici. Max 8 input. |
| **Interrupt Mode** | Modalità input: hardware interrupt immediato. Solo 4a linea IO di certi slot. |
| **Record Offset** | Pattern AppBlocks (sprinkler6): permette al loop di trovare record multipli consecutivi senza fermarsi sempre al primo |
| **SCHV** | Setting "Schema Version" — traccia la versione dello schema EEPROM per gestire migrazioni |
| **lockOnActive** | Se Y nell'ingresso, il relè associato non può essere spento mentre l'ingresso è ancora attivo (security) |
