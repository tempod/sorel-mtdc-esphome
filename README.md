# sorel-mtdc-esphome

**Centralina solare SOREL MTDC → Home Assistant, via bus CAN con ESPHome (ESP8266 + MCP2515).**

*English summary: read-only ESPHome integration for the SOREL MTDC (v5) solar thermal controller
over its CAN bus (SCBI protocol, 250 kbit/s) using an ESP8266 and an MCP2515 module. It exposes
temperatures, relays, pump speed, solar power, solar energy (day/week/month/year/total), operating
hours and diagnostics to Home Assistant. Protocol notes in [docs/PROTOCOLLO.md](docs/PROTOCOLLO.md)
(Italian, tables are self-explanatory).*

---

## Cosa fa

Un ESP8266 collegato alla presa CAN della MTDC **ascolta** i dati che la centralina trasmette
da sola e li pubblica in Home Assistant tramite l'API nativa di ESPHome.
**Sola lettura**: l'ESP non invia comandi e non modifica nessun parametro.

| Gruppo | Entità |
|---|---|
| Temperature | S1, S2, S3, S4 |
| Solare | Potenza solare (kW), Velocità pompa solare (%) |
| Energia | Energia solare oggi / settimana / mese / anno / **totale** (kWh, pronta per la dashboard Energia) |
| Ore di esercizio | R1 oggi / settimana / mese / anno / totale · R2 oggi / settimana / mese / anno |
| Relè | Relè R1, Relè R2 |
| Diagnostica | Centralina online, Ultimo riavvio centralina, Messaggio centralina, Programma, WiFi, Uptime |

In più:
- **notifica dopo un blackout** (la MTDC chiede di confermare data e ora): automazione HA o flow Node-RED;
- gli ultimi valori vengono **ricordati** dopo un riavvio dell'ESP (niente "sconosciuto");
- un **test di loopback** per verificare il cablaggio ESP ↔ MCP2515 senza la centralina.

## Hardware

- NodeMCU v3 (o Wemos D1 mini)
- modulo MCP2515 + TJA1050 (quarzo 8 MHz)
- terminazione 120 Ω **anche lato centralina** ← senza questa il bus non funziona

| NodeMCU | MCP2515 |
|---|---|
| VU (5 V) | VCC |
| G | GND |
| D5 | SCK |
| D7 | SI |
| D6 | SO |
| D8 | CS |

H e L del modulo vanno a CAN-H e CAN-L di una delle due prese CAN della centralina.
Tutti i dettagli, le avvertenze (livelli 5 V, pin D8, quarzo) e la procedura di diagnosi sono in
**[docs/HARDWARE.md](docs/HARDWARE.md)**.

## Installazione

1. Copia [`esphome/sorelmtdc.yaml`](esphome/sorelmtdc.yaml) nella cartella di ESPHome.
2. Crea `secrets.yaml` partendo da [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example).
3. Controlla nel file: `board` (nodemcuv2 / d1_mini) e `clock` del MCP2515 (8 o 16 MHz).
4. Compila e carica. Entro pochi secondi devono arrivare i primi dati
   (le temperature ferme vengono rimandate al massimo ogni ~5 minuti).
5. Opzionale: notifica di riavvio con
   [`homeassistant/automazione_riavvio.yaml`](homeassistant/automazione_riavvio.yaml)
   **oppure** [`nodered/sorel_riavvio.json`](nodered/sorel_riavvio.json) (in Node-RED: Importa, poi seleziona il server HA).

**Requisiti**: ESPHome **2026.9** o successivo per `ota: encryption:`. Con versioni precedenti
sostituiscilo con `password:` (vedi commento nel file).

Se non arriva nulla: [docs/HARDWARE.md → Procedura di diagnosi](docs/HARDWARE.md#procedura-di-diagnosi-consigliata).

## Note e limiti

- Testato su **una** MTDC v5, programma S10, display in italiano. Altre versioni potrebbero differire.
- La centralina trasmette le temperature con risoluzione di **1 °C** e l'energia di **1 kWh**.
- Per energia e potenza serve la **contabilizzazione del calore** attiva (menu 6.6 della centralina).
- L'orologio della centralina dev'essere corretto: le statistiche (e il filtro "oggi") dipendono dalla data.
- Dopo un blackout la centralina può perdere le statistiche delle ultime ore (comportamento della MTDC).
- **Non** è possibile impostare data e ora via CAN: vedi [docs/PROTOCOLLO.md](docs/PROTOCOLLO.md#cosa-non-funziona-scrittura--richieste).
- Il totale ore di R2 e (non sempre) il programma non vengono trasmessi: i sensori restano in ascolto.

## Documentazione

| File | Contenuto |
|---|---|
| [docs/HARDWARE.md](docs/HARDWARE.md) | componenti, schema, terminazione, diagnosi |
| [docs/PROTOCOLLO.md](docs/PROTOCOLLO.md) | mappa del protocollo con frame reali e verifiche col display |
| [docs/STORIA.md](docs/STORIA.md) | da dove siamo partiti e come ci siamo arrivati |
| [CHANGELOG.md](CHANGELOG.md) | versioni |

## Crediti

- **[ddimension/sorel-canbus](https://github.com/ddimension/sorel-canbus)** (MIT): decoder Python del protocollo SCBI,
  base per struttura degli ID, sonde e relè.
- **[Tasmota – discussione #20598 "SOREL CAN BUS Interface (SCBI)"](https://github.com/arendst/Tasmota/discussions/20598)**:
  frame reali, velocità del bus e conferma che la centralina trasmette senza richieste.
- **[ESPHome](https://esphome.io)** e il suo componente `mcp2515`.
- **SOREL GmbH**: manuale d'istruzioni della MTDC (non incluso in questo repository per motivi di copyright).
- Sviluppo, decodifica e documentazione realizzati con l'assistenza di **Claude** (Anthropic);
  test, misure e verifiche sull'impianto reale a cura di **tempod**.

## Avvertenze

Progetto amatoriale, **non affiliato né approvato da SOREL GmbH**. "SOREL" e "MTDC" sono marchi dei rispettivi proprietari.
Il collegamento avviene sul bus CAN a bassa tensione (PELV) della centralina: non aprire la centralina e non intervenire
sulla parte a 230 V se non sei qualificato. Uso a proprio rischio (vedi [LICENSE](LICENSE)).

## Licenza

[MIT](LICENSE)
