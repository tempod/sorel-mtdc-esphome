# Protocollo CAN della SOREL MTDC (SCBI): cosa è stato decodificato

Questa è la mappa del traffico osservato sul bus di una **SOREL MTDC v5** (programma idraulico **S10**,
firmware con display in italiano). La base viene dal decoder Python
[ddimension/sorel-canbus](https://github.com/ddimension/sorel-canbus) e da una
[discussione Tasmota](https://github.com/arendst/Tasmota/discussions/20598);
tutto il resto è stato **verificato sul campo** confrontando i frame con il display della centralina.

> ⚠️ Non è documentazione ufficiale SOREL. Altri modelli o firmware potrebbero differire.

## Livello fisico

| Parametro | Valore |
|---|---|
| Velocità | 250 kbit/s |
| Identificatori | estesi, 29 bit |
| Terminazione | 120 Ω a entrambi i capi |
| Iniziativa | la centralina trasmette **da sola**: nessun Data Logger necessario |

## Struttura dell'ID (little-endian)

```
ID 0x10019780  →  byte: 80 97 01 10
                         │  │  │  └─ byte3: bit 0-2 = protocollo, bit 3-4 = tipo messaggio
                         │  │  │            (0x10 = risposta / dato, 0x00-0x01 = richiesta)
                         │  │  └──── byte2: funzione (func)
                         │  └─────── byte1: client = ID della centralina (qui 0x97, varia)
                         └────────── byte0: programma (prog)
```

Il filtro dell'ESP ignora il byte del client, quindi la configurazione funziona con qualsiasi centralina.

## prog 0x80 – Dati (Datalogger)

### func 0x01 – Sonde di temperatura
```
[id][valore int16 LE][tipo][sottotipo]        valore / 10 = °C
```
| id | Sonda |
|---|---|
| 0–3 | S1–S4 |
| 4 | **non** è una sonda: è la potenza solare (W/10) vista come sonda virtuale – ignorata |

- Risoluzione trasmessa: **1 °C** (valori multipli di 10).
- Valori intorno a ±3276 °C (es. `01 80` = −32767) = sonda scollegata o non pronta (all'accensione).
- **Ritmo di invio**: subito a ogni variazione (al massimo una ogni 10 s per sonda) e comunque
  **ogni ~308 s** anche senza variazioni.

Esempio reale: `0x10019780 [00 96 00 00 00]` → S1 = 0x0096 = 150 → **15 °C**

### func 0x02 – Relè e uscite
```
[id][modo][valore][..]
```
| id | Uscita | Valori |
|---|---|---|
| 0 | R1 (pompa solare) | 0 = spento, 255 = acceso |
| 1 | R2 | 0 = spento, 255 = acceso |
| 2 | Uscita PWM / 0-10 V (velocità pompa) | modo 2 = PWM, 0–255 → 0–100 % |

### func 0x03 – Programma idraulico
Testo ASCII, es. `[53 31 30]` = **"S10"**. Visto solo all'accensione della centralina,
e non a ogni accensione.

### func 0x04 – Registro messaggi (menu 2.4)
Due frame consecutivi al verificarsi di un evento:
```
0x01049780  [07 00 00 00 00 00 00 00]          intestazione: codice 7
0x09049780  [FE 52 69 61 76 76 69 6F]          FE + testo ASCII: "Riavvio"
```
Il testo è nella lingua del display. Messaggi più lunghi proseguono presumibilmente in altri frame
con byte3 = 0x09 (gestiti dal codice, non ancora osservati).

### func 0x06 – Statistiche
```
[id][valore int16 LE]
```
| id | Significato |
|---|---|
| 1 | **Potenza solare istantanea in W** (es. `01 1E 10` = 4126 W) – verificato col display |
| 0, 4, 7, 8 | sconosciuti (valori piccoli, es. 1 o 7) |

### func 0x07 – Riepilogo ore di esercizio ed energia (menu 2.1 / 2.2)
```
[tipo<<5 | indice][campo][ore uint16 LE][kWh uint32 LE]
```
| tipo | Periodo | indice |
|---|---|---|
| 1 | giorno | **giorno della settimana**: 0 = lunedì … 6 = domenica |
| 2 | settimana | 0 = corrente, 1 = precedente |
| 3 | mese | 0 = corrente |
| 4 | anno | 0 = corrente |
| 5 | totale | 0 |
| 6, 7 | sconosciuti | |

| campo | Contenuto |
|---|---|
| 0 | ore R1 + **kWh solari** (contabilizzazione calore) |
| 1 | ore R2 (kWh sempre 0) |
| 2 | ore (uguali a R1, probabilmente il contatore del contacalorie) |

- L'intero riepilogo viene **ripetuto ogni ~10,6 s**.
- La centralina invia anche gli slot dei periodi appena chiusi (es. il lunedì manda sia lunedì sia domenica,
  sia la settimana corrente sia la precedente): per questo l'ESP filtra per indice.
- **Totale R2 (`A0 01 …`) non viene mai trasmesso**, anche se il display lo mostra.
- L'energia ha risoluzione di **1 kWh** ed è disponibile solo se la contabilizzazione del calore
  (menu 6.6) è attiva.

Verifica col display (10/10/2026):

| Display | Frame | Valore |
|---|---|---|
| R1 anno 769 h | `80 00 01 03 …` | 0x0301 = 769 ✓ |
| R2 anno 2,58 kh | `80 01 15 0A …` | 0x0A15 = 2581 ✓ |
| R1 mese 19 h / R2 27 h | `60 00 13 …` / `60 01 1B …` | 19 / 27 ✓ |
| R1 totale 5,87 kh | `A0 00 F0 16 …` | 0x16F0 = 5872 ✓ |

## prog 0x0B – Controller

| func | Significato | Esempio |
|---|---|---|
| 0x01 | "sono qui", ogni 60 s | `0x1001000B [97 00 78 00 …]` (ID 0x97, OEM 0x78) |
| 0x08 | **centralina appena riavviata** | `0x1008000B [97 00 78 00 …]` |

## Altri frame osservati (non decodificati)

| ID | Dati | Note |
|---|---|---|
| `0x10019785` | `[00]` / `[01]` ogni 10 s | prog 0x85 = circuito riscaldamento (HCC) |
| `0x100D9785` | `[00 00 00]` | all'accensione |
| `0x10029781`, `0x10019781` | `[01 02 01]`, `[05 01 01]` | prog 0x81, sconosciuto |

## Cosa NON funziona: scrittura / richieste

Sono state provate (solo richieste di **lettura**, innocue):
- richiesta sonde (`0x00019780`, varie forme, anche remote frame);
- "c'è qualcuno?" (`0x0000000B`, `0x0100000B`);
- lettura data/ora (`0x0106970B`, `0x0106000B`) anche con il bit di protocollo a 1, come nelle richieste
  che la centralina stessa invia.

Tutti i frame vengono confermati (ACK) ma **nessuno riceve risposta**. Inoltre impostando data e ora
dal display **non passa nulla sul bus**. Per questo il progetto è in sola lettura e il problema
"conferma data/ora dopo un blackout" è risolto con una **notifica** basata sul frame `0x1008000B`.
