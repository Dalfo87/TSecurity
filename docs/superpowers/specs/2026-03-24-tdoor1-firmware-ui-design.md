# TDoor_1 — Firmware API + UI Design Spec
**Data:** 2026-03-24
**Progetto:** Door Sensus Control Unit — Tibbo TPP2-G2
**Scope:** TDoor_1 (dispositivo di campo) — strato API firmware + UI embedded (config/admin)
**Escluso da questo spec:** TMaster firmware, PSIM integration, protocollo low-level TDoor↔TMaster

---

## 1. Architettura di sistema (contesto)

Il sistema è composto da tre componenti:

| Componente | Ruolo | UI operatore |
|---|---|---|
| **TDoor_1** | Dispositivo di campo: legge sensori porta, controlla uscite relè locali | Config/admin (tecnico) |
| **TMaster** | Dispositivo control room: sirena CR, indicatori locali | Pannello operatore backup |
| **PSIM** | Software gestione sicurezza fisica | Interfaccia operatore quotidiana |

**Canali di comunicazione di TDoor_1:**
- Verso PSIM: HTTP API (primario, operazioni quotidiane)
- Verso TMaster: protocollo low-level TCP/UDP (backup operativo, socket S1 già presente nel firmware a `10.80.0.105:6480`)

Quando scatta un allarme, TDoor_1 apre **entrambi i canali** simultaneamente.
Il comando dell'operatore (Silenzia, Ack, ecc.) può arrivare da PSIM via HTTP o da TMaster via protocollo basso livello — TDoor_1 esegue l'azione e aggiorna EVST in entrambi i casi.

---

## 2. Macchina a stati evento (per ingresso)

### 2.1 Stati

| Valore | Nome | Output campo | Nota |
|---|---|---|---|
| `0` | **Chiuso** | OFF | Stato normale, nessun evento attivo |
| `1` | **Allarme** | ON | Input attivo, sirena campo ON, notifica TMaster CR siren ON |
| `2` | **Silenziato** | ON | Operatore ha silenziato CR (TMaster siren OFF), sirena campo rimane ON |
| `3` | **Manutenzione** | OFF | Input temporaneamente ignorato, accessibile da qualsiasi stato |
| `4` | **Disabilitato** | OFF | Input escluso completamente — solo admin |

> **Nota chiave:** "Silenzia" non spegne l'uscita di campo di TDoor_1. Invia solo il comando di spegnimento alla sirena CR del TMaster. L'uscita di campo rimane ON finché non viene eseguito Ack con input a riposo.

### 2.2 Transizioni di stato

```
[0] Chiuso
  → input va attivo                          → [1] Allarme
                                                output campo ON
                                                notifica TMaster: ALLARME

[1] Allarme
  → Silenzia                                 → [2] Silenziato
                                                output campo: rimane ON
                                                notifica TMaster: SILENZIA (CR siren OFF)
  → Ack + input LOW (o lockOnActive=N)       → [0] Chiuso
                                                output campo OFF
                                                notifica TMaster: CHIUSO
  → Ack + input HIGH + lockOnActive=Y        → [1] Allarme (nessun cambio stato)
                                                output campo: rimane ON
                                                notifica TMaster: ALLARME (CR siren torna ON)

[2] Silenziato
  → Ack + input LOW (o lockOnActive=N)       → [0] Chiuso
                                                output campo OFF
                                                notifica TMaster: CHIUSO
  → Ack + input HIGH + lockOnActive=Y        → [1] Allarme
                                                output campo: rimane ON
                                                notifica TMaster: ALLARME (CR siren torna ON)

[qualsiasi stato] → Manutenzione (operatore) → [3] Manutenzione
                                                output campo OFF
                                                notifica TMaster: MANUTENZIONE
[3] → riabilita (operatore)                  → [0] Chiuso
                                                notifica TMaster: CHIUSO

[qualsiasi stato] → Disabilita (solo admin)  → [4] Disabilitato
                                                output campo OFF
                                                notifica TMaster: DISABILITATO
[4] → riabilita (solo admin)                 → [0] Chiuso
                                                notifica TMaster: CHIUSO
```

### 2.3 Flag `lockOnActive` (nel CSV di configurazione ingresso, campo 6)

| lockOnActive | Comportamento Ack con input HIGH |
|---|---|
| `Y` (default, porte critiche) | Ack respinto: stato torna a Allarme, operatore deve ripetere da Silenzia |
| `N` (porte non critiche) | Ack accettato: evento chiuso anche con input fisicamente attivo |

### 2.4 Logica `evaluate_inputs()` — rispetto degli stati EVST

`evaluate_inputs()` viene chiamato ogni secondo dal timer. Deve rispettare gli stati EVST gestiti dall'operatore. Regole:

```
per ogni ingresso i (0..NUM_INPUTS-1):

  se EVST[i] = '3' (Manutenzione) o '4' (Disabilitato):
      → skip: non aggiornare in_state(i), non toccare output
      → continua al prossimo ingresso

  se EVST[i] = '1' o '2' (evento aperto):
      → aggiorna in_state(i) con lo stato fisico corrente (per uso UI/log)
      → NON modificare EVST[i] (solo set_event() può farlo)
      → NON toccare l'output (rimane ON)
      → continua

  se EVST[i] = '0' (Chiuso):
      → se input va attivo (LOW→HIGH per NO, HIGH→LOW per NC):
          EVST[i] = '1'
          set_output(get_linked_output(i), ON)
          notifica TMaster: "ALLARME i"
          stg_set("EVST", 0, EVST)
          scrivi log: "ALLARME ingresso i"
      → altrimenti: nessuna azione
```

### 2.5 Persistenza stato — setting EVST

Lo stato eventi di tutti e 8 gli ingressi è salvato in un singolo setting EEPROM `EVST`:
- Formato: stringa di 8 caratteri, uno per ingresso (es. `"10000000"`)
- Valore default: `"00000000"` (tutti chiusi — da specificare nel default `settings.xtxt`)
- Scrittura: ad ogni transizione di stato (non ad ogni poll — protegge cicli EEPROM)
- Lettura: al boot, ripristina `out_state[]` coerentemente
- Non visibile nel log eventi operatore

**Entry in `settings.xtxt`:**
```
>>EVST E S 1 0 8 00000000 Stato eventi ingressi
```
*(Il campo default è `00000000` — tutti gli stati a Chiuso)*

**Logica boot (`boot.tbs`):**
```basic
EVST = stg_get("EVST", 0)
if len(EVST) < 8 then EVST = "00000000"   ' protezione stringa corrotta
' ripristino uscite coerente con stati salvati
dim i as byte
for i = 0 to NUM_INPUTS - 1
    dim st as byte = val(mid(EVST, i + 1, 1))
    dim linked_out as byte = get_linked_output(i)
    if linked_out > 0 then
        if st = 1 or st = 2 then
            set_output(linked_out - 1, YES)
        else
            set_output(linked_out - 1, NO)
        end if
    end if
next i
```

**Dichiarazione in `global.tbh` (da aggiungere):**
```basic
declare function get_linked_output(idx as byte) as byte
' ritorna 0 se nessuna uscita collegata, 1-6 per OUT1..OUT6
' implementata in io_manager.tbs — legge campo linkOutput dal CSV IN0x
```

---

## 3. API Firmware

> **Nota implementativa:** Tibbo BASIC non dispone di librerie JSON. Tutte le risposte JSON sono stringhe costruite manualmente con concatenazione.

### 3.1 Endpoint esistenti — mantenere con fix

| Endpoint | Fix richiesto |
|---|---|
| `?e=w` | Nessuno |
| `?e=v&action=get&name=X` | Nessuno |
| `?e=v&action=set&name=X&value=Y` | **Fix**: dopo `stg_set(name, 0, value)` aggiornare immediatamente la variabile RAM corrispondente (IN01..IN08, OUT01..OUT06, TMIP, TMPT, TMKY, ecc.) |
| `?e=i` | Nessuno |
| `?e=io&action=set_out&idx=N&state=S` | Nessuno |

### 3.2 Endpoint nuovo — `?e=io&action=get`

Ritorna snapshot JSON completo dello stato I/O + eventi.

**Request:** `GET ?e=io&action=get&password=VALUE`

**Response:**
```json
{
  "in":   [1, 0, 0, 0, 0, 0, 0, 0],
  "out":  [1, 0, 0, 0, 0, 0],
  "evst": "10000000",
  "ts":   1234567
}
```

| Campo | Tipo | Descrizione |
|---|---|---|
| `in` | array[8] byte | Stato fisico ingressi (0=LOW/riposo, 1=HIGH/attivo) |
| `out` | array[6] byte | Stato fisico uscite (0=OFF, 1=ON) |
| `evst` | string[8] | Stato evento per ingresso (0–4, vedi §2.1) |
| `ts` | integer | Uptime in secondi — fonte: `sys.timems \ 1000` |

**Costruzione JSON in Tibbo BASIC (esempio):**
```basic
dim resp as string
resp = "{""in"":[" + str(in_state(0)) + "," + str(in_state(1)) + ...
resp = resp + "],""out"":[" + str(out_state(0)) + ...
resp = resp + "],""evst"":""" + EVST + """,""ts"":" + str(sys.timems \ 1000) + "}"
```

### 3.3 Endpoint nuovo — `?e=io&action=set_event&idx=N&st=S`

Gestisce le transizioni di stato evento.

**Request:** `GET ?e=io&action=set_event&idx=N&st=S&password=VALUE`

| Parametro | Valori | Accesso |
|---|---|---|
| `idx` | 0–7 (ingresso) | — |
| `st` | `0`=Ack, `2`=Silenzia, `3`=Manutenzione, `4`=Disabilita | `st=4` solo admin |

**Response generica (successo per st=2, 3, 4):**
```json
{"ok": 1, "evst_idx": N}
```

**Logica firmware per `st=2` (Silenzia):**
```
EVST[idx] = '2'
' output campo: rimane invariato (ON se era ON)
notifica TMaster: "SILENZIA idx"
stg_set("EVST", 0, EVST)
scrivi log: "SILENZIATO ingresso idx"
risposta: {"ok":1,"evst_idx":2}
```

**Logica firmware per `st=0` (Ack):**
```
se in_state_raw(idx) = LOW oppure lockOnActive(idx) = "N":
    EVST[idx] = '0'
    set_output(get_linked_output(idx) - 1, NO)
    notifica TMaster: "CHIUSO idx"
    stg_set("EVST", 0, EVST)
    scrivi log: "CHIUSO ingresso idx"
    risposta: {"ok":1,"evst_idx":0}

se in_state_raw(idx) = HIGH e lockOnActive(idx) = "Y":
    EVST[idx] = '1'
    set_output(get_linked_output(idx) - 1, YES)   ' rimane ON
    notifica TMaster: "ALLARME idx"
    stg_set("EVST", 0, EVST)
    scrivi log: "ACK RESPINTO ingresso idx (input attivo)"
    risposta: {"ok":0,"evst_idx":1,"msg":"input attivo"}
```

**Logica firmware per `st=3` (Manutenzione):**
```
EVST[idx] = '3'
dim lo3 as byte = get_linked_output(idx)
if lo3 > 0 then set_output(lo3 - 1, NO)
notifica TMaster: "MANUTENZIONE idx"
stg_set("EVST", 0, EVST)
scrivi log: "MANUTENZIONE ingresso idx"
risposta: {"ok":1,"evst_idx":3}
```

**Logica firmware per `st=4` (Disabilita — solo admin):**
```
' verificare password admin prima di procedere
EVST[idx] = '4'
dim lo4 as byte = get_linked_output(idx)
if lo4 > 0 then set_output(lo4 - 1, NO)
notifica TMaster: "DISABILITATO idx"
stg_set("EVST", 0, EVST)
scrivi log: "DISABILITATO ingresso idx"
risposta: {"ok":1,"evst_idx":4}
```

**Response Ack respinto (input ancora attivo):**
```json
{"ok": 0, "evst_idx": 1, "msg": "input attivo"}
```

### 3.4 Nuovi setting EEPROM da aggiungere

Conteggio attuale: 27 settings esistenti + 4 nuovi = **31 totali** (limite: 40).

| Nome | Tipo | Len | Default | Descrizione |
|---|---|---|---|---|
| `EVST` | string | 8 | `00000000` | Stato eventi ingressi (§2.5) |
| `TMIP` | string | 16 | *(vuoto)* | IP TMaster (16 char per compatibilità con pattern esistente NETIP/MVIP) |
| `TMPT` | string | 5 | `6480` | Porta TMaster |
| `TMKY` | string | 32 | *(vuoto)* | Chiave cifratura TMaster *(salvata, non usata in logica v1)* |

---

## 4. Struttura UI TDoor_1

> **Ruolo UI:** configurazione, admin e manutenzione per tecnico/admin.
> L'interfaccia operatore quotidiana è PSIM. Il pannello operatore backup è la UI di TMaster.

### 4.1 Polling status

La UI effettua un poll ogni **3 secondi** su `?e=io&action=get`.
Il risultato aggiorna:
- Status bar (sempre visibile in fondo)
- Dashboard se pagina attiva
- Badge alert nella sidebar se ci sono eventi in stato 1 o 2

### 4.2 Codice colore stati evento

| Stato | Colore | Comportamento |
|---|---|---|
| Chiuso (0) | Verde | Statico |
| Allarme (1) | Rosso | Lampeggiante |
| Silenziato (2) | Arancione | Statico |
| Manutenzione (3) | Giallo | Statico |
| Disabilitato (4) | Grigio | Statico |

### 4.3 Pagine

#### Dashboard (nuova — pagina iniziale)
- 8 card ingresso: stato fisico (HIGH/LOW) + stato evento (colore EVST)
- 6 card uscita: ON/OFF
- Pulsanti emergenza per tecnico/admin: Silenzia, Ack, Manutenzione per singolo ingresso
- Accesso Disabilita solo con conferma e profilo admin

#### Ingressi
- Lista 8 ingressi configurabili
- Campi CSV: `nome, NC/NO, doorState, alarmOnMuted, linkOutput, milestoneName, lockOnActive`
- Aggiornamento a 8 righe (rimosso IN09)
- `lockOnActive` con label esplicita: "Blocca chiusura su input attivo"

#### Uscite
- Lista 6 uscite configurabili
- Campo: `nome` (stringa 40 char)
- Nessuna modifica strutturale per v1

#### Rete
- Invariata rispetto all'esistente (IP, subnet, gateway, DNS)

#### Integrazioni
Tre sottosezioni nella stessa pagina:

**Milestone VMS** (esistente — invariato)

**Xovis** (esistente — invariato)

**TMaster** (nuova sottosezione):
| Campo UI | Setting | Len | Note |
|---|---|---|---|
| IP TMaster | `TMIP` | 16 | Indirizzo IP dispositivo TMaster |
| Porta | `TMPT` | 5 | Default: 6480 |
| Chiave cifratura | `TMKY` | 32 | Salvata, non usata in logica v1 |

#### Manutenzione
- Toggle per-ingresso "Metti in manutenzione" → chiama `?e=io&action=set_event&idx=N&st=3`
- Sezione esistente: reboot, factory reset, log di sistema
- Nessuna rimozione rispetto all'esistente

#### Log eventi
- Invariato (read-only)

#### Licenza
- Invariato

### 4.4 Status bar (critica — non rimuovere)
Aggiornamento: leggere `evst` da `?e=io&action=get` ogni 3s.
Se uno o più ingressi sono in stato 1 o 2 → mostrare badge rosso/arancione con conteggio eventi aperti.

---

## 5. Modifiche file firmware richieste

| File | Modifica |
|---|---|
| `settings.xtxt` | Aggiungere EVST, TMIP, TMPT, TMKY (vedi §3.4) |
| `global.tbh` | Dichiarare `dim EVST as string(8)`, `dim TMIP as string(16)`, `dim TMPT as string(5)`, `dim TMKY as string(32)`; dichiarare `function get_linked_output(idx as byte) as byte` |
| `boot.tbs` | Caricare EVST con protezione stringa corrotta; ripristinare out_state[] al boot (vedi §2.5) |
| `device.tbs` | (1) Aggiungere handler `?e=io&action=get` con JSON hand-crafted; (2) Aggiungere handler `?e=io&action=set_event`; (3) Fix RAM update su `stg_set` per TMIP/TMPT/TMKY |
| `io_manager.tbs` | (1) Implementare `get_linked_output(idx)` — legge campo `linkOutput` dal CSV di IN0x+1; (2) Aggiornare `evaluate_inputs()` con logica EVST (§2.4) |
| `main.tbs` | Nessuna modifica strutturale |

---

## 6. Vincoli tecnici

- File HTML UI: single file, no framework, Vanilla JS ES5, inline CSS — target < 100KB
- Tutto il testo UI in italiano
- Colori tramite variabili CSS `:root`
- Toast: `toast(msg, isError?)` — mai `alert()`
- Confirm: `showConfirm(msg, cb)` — mai `confirm()`
- Hardware: Tibbo TPP2-G2, Ethernet only, no WiFi
- EEPROM: `STG_MAX_NUM_SETTINGS` = 40 — 27 esistenti + 4 nuovi = **31 totali**
- Cicli EEPROM: scrittura EVST solo su transizione stato (non a ogni poll)
- JSON: nessuna libreria disponibile in Tibbo BASIC — costruzione manuale con concatenazione stringhe
- Uptime: `sys.timems \ 1000` per campo `ts` nelle risposte API
- Naming: evitare nomi che coincidono con keyword Tibbo BASIC (es. `ON` è già usato nel codebase come variabile owner name — non introdurre nuovi conflitti)
