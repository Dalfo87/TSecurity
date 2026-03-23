Ipotesi di integrazione tra DemoMaster e Progetto definitivo Tdoor.

1. Nuova Mappatura I/O Rivista
Configurazione Hardware Aggiornata

| Socket  | Tibbit | Tipo          | Linee  | Funzione               |
| ------- | ------ | ------------- | ------ | ---------------------- |
| **S1**  | #03.1  | 2 Relè        | L1, L2 | **OUT1, OUT2**         |
| **S3**  | #03.1  | 2 Relè        | L1, L2 | **OUT3, OUT4**         |
| **S5**  | #03.1  | 2 Relè        | L1, L2 | **OUT5, OUT6**         |
| **S7**  | #54    | 4 Input       | L1-L4  | **IN1, IN2, IN3, IN4** |
| **S9**  | #54    | 4 Input       | L1-L4  | **IN5, IN6, IN7, IN8** |
| **S11** | —      | Alimentazione | —      | Power supply           |

Mapping GPIO Completo
| Funzione | GPIO                  | Slot | Linea | Note                 |
| -------- | --------------------- | ---- | ----- | -------------------- |
| **OUT1** | PL\_IO\_NUM\_9\_TX0   | S1   | L1    | Tibbit #03.1 Relay 1 |
| **OUT2** | PL\_IO\_NUM\_8\_RX0   | S1   | L2    | Tibbit #03.1 Relay 2 |
| **OUT3** | PL\_IO\_NUM\_11\_TX1  | S3   | L1    | Tibbit #03.1 Relay 1 |
| **OUT4** | PL\_IO\_NUM\_10\_RX1  | S3   | L2    | Tibbit #03.1 Relay 2 |
| **OUT5** | PL\_IO\_NUM\_13\_TX2  | S5   | L1    | Tibbit #03.1 Relay 1 |
| **OUT6** | PL\_IO\_NUM\_12\_RX2  | S5   | L2    | Tibbit #03.1 Relay 2 |
| **IN1**  | PL\_IO\_NUM\_15\_TX3  | S7   | L1    | Tibbit #54 Input 1   |
| **IN2**  | PL\_IO\_NUM\_14\_RX3  | S7   | L2    | Tibbit #54 Input 2   |
| **IN3**  | PL\_IO\_NUM\_3        | S7   | L3    | Tibbit #54 Input 3   |
| **IN4**  | PL\_IO\_NUM\_19\_INT3 | S7   | L4    | Tibbit #54 Input 4   |
| **IN5**  | PL\_IO\_NUM\_32       | S9   | L1    | Tibbit #54 Input 1   |
| **IN6**  | PL\_IO\_NUM\_33       | S9   | L2    | Tibbit #54 Input 2   |
| **IN7**  | PL\_IO\_NUM\_4        | S9   | L3    | Tibbit #54 Input 3   |
| **IN8**  | PL\_IO\_NUM\_20\_INT4 | S9   | L4    | Tibbit #54 Input 4   |

2. Associazione Input/Output Libera
L'associazione N:M (non 1:1) è già prevista nel documento PROGETTO.md:
' Logica firmware per associazione libera
for i = 0 to 7  ' 8 ingressi (0-based)
  linked_out = get_csv_field(stg_get("IN0"+str(i+1)), 5)  ' campo linkOutput
  if linked_out > 0 and input_state(i) = ALARM then
    call set_output(linked_out - 1, ON)
  end if
next i




Esempi di configurazione possibili:
| Configurazione   | Descrizione                               |
| ---------------- | ----------------------------------------- |
| IN1 → OUT1       | Allarme porta 1 attiva sirena 1           |
| IN1 → OUT1, OUT2 | Allarme porta 1 attiva sirena + strobe    |
| IN1, IN2 → OUT1  | Due porte attivano la stessa sirena       |
| IN3 → Nessuno    | Ingresso solo per logging, nessuna uscita |
| IN5 → OUT6       | Ingresso 5 controlla uscita 6             |

3. Analisi UI di Riferimento (TdoorUI.html)
Struttura della Dashboard
┌─────────────────────────────────────────────────────────────┐
│ 🔐 Door Sensus Control Unit    192.168.1.40 — DSControlUnit │  ← Topbar
├──────────┬──────────────────────────────────────────────────┤
│          │                                                  │
│  Sidebar │              Content Area                        │
│          │                                                  │
│ Config   │  ┌────────────────────────────────────────────┐  │
│ ├── ⬤    │  │ Ingressi / Uscite / Rete / Integrazioni  │  │
│ ├── ◉    │  └────────────────────────────────────────────┘  │
│ ├── ◈    │                                                  │
│ └── ◈    │                                                  │
│          │                                                  │
│ Sistema  │                                                  │
│ ├── ⚙    │                                                  │
│ └── ≡    │                                                  │
│          │                                                  │
├──────────┴──────────────────────────────────────────────────┤
│ IN ●●●●●●●●●●●●●●●● | OUT ◯◯◯◯◯◯                          │  ← Status Bar


--bg-deep: #0b0e14      /* Sfondo principale */
--bg-panel: #111520     /* Sidebar, topbar, status bar */
--bg-card: #161b28      /* Card contenuto */
--amber: #f0a030        /* Brand color / accent */
--green: #28c76f        /* OK / Chiuso / ON */
--red: #ea4a5a          /* Allarme / Aperto / Errore */
--yellow: #f5c518       /* Warning */


Pagine UI da Implementare
| Pagina           | Funzionalità                                                                                         |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| **Ingressi**     | Tabella 8 ingressi con: Nome, Nome Milestone, Link Output, Stato Porta, Tipo (NC/NO), Alarm On Muted |
| **Uscite**       | Tabella 6 uscite con: Nome, Tipo, Stato ON/OFF, Toggle, Impulso (2s)                                 |
| **Rete**         | DHCP toggle, IP statico, Gateway, Netmask, Server NTP                                                |
| **Integrazioni** | Milestone VMS (IP, Porta, User, Pass), Xovis (Porta, Comando, Direzione)                             |
| **Manutenzione** | Tipo App, Riavvio, Reset Fabbrica, Password, Licenza                                                 |
| **Log Eventi**   | Tabella 100 eventi con timestamp, porta, evento                                                      |

Status Bar (Aggiornamento ogni 3s)
// Polling I/O ogni 3 secondi
setInterval(function() {
  fetch('api.html?e=io&action=get&password=' + PW_SESSION)
    .then(r => r.json())
    .then(data => updateStatusBar(data));
}, 3000);

5. Schema Completo Settings EEPROM
| Pos   | Key           | Max | Default         | Contenuto              |
| ----- | ------------- | --- | --------------- | ---------------------- |
| 1     | `DN`          | 8   | —               | Nome dispositivo       |
| 2     | `ON`          | 8   | —               | Nome proprietario      |
| 3     | `PW`          | 16  | `^`             | Password               |
| 4     | `NETCP`       | 1   | `1`             | DHCP                   |
| 5     | `NETIP`       | 16  | `10.80.0.10`    | IP statico             |
| 6     | `NETNM`       | 16  | `255.255.255.0` | Netmask                |
| 7     | `NETGW`       | 16  | `10.80.0.1`     | Gateway                |
| 8     | `stg1`        | 80  | —               | Stato runtime (legacy) |
| 9     | `SCHV`        | 4   | `1`             | Schema version         |
| 10-17 | `IN01`-`IN08` | 80  | —               | Config ingressi CSV    |
| 18-23 | `OUT1`-`OUT6` | 40  | —               | Config uscite CSV      |
| 24    | `MVIP`        | 20  | —               | Milestone IP           |
| 25    | `MVPT`        | 6   | `1238`          | Milestone porta        |
| 26    | `MVUN`        | 32  | —               | Milestone user         |
| 27    | `MVPW`        | 32  | —               | Milestone password     |
| 28    | `XVPT`        | 6   | `4321`          | Xovis porta            |
| 29    | `XVCM`        | 32  | `LineCrossing`  | Xovis comando          |
| 30    | `XVDI`        | 10  | `forward`       | Xovis direzione        |
| 31    | `NTPIP`       | 20  | `192.168.0.3`   | NTP server             |
| 32    | `APTP`        | 20  | `AllFunction`   | Tipo app               |

6. Prossimi Passi Consigliati
1.    Confermo che i Tibbit #03.1 e #54 sono già montati su S1, S3, S5, S7, S9
2.    Aggiorna settings.xtxt: Definisci i 32 settings con le nuove dimensioni
3.    Implementa index_to_io_num(): Usa la mappatura GPIO rivista sopra
4.    Crea handler ?e=io: Per lettura stato I/O e controllo uscite
5.    Sviluppa UI: Basata su TdoorUI.html con chiamate HTTP reali
6.    Test integrazione: Verifica associazione libera input/output

