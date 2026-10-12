# Changelog

## 1.0.0 – 2026-10-12
Prima versione pubblica.

- Lettura di sonde S1–S4, relè R1–R2, velocità pompa, potenza solare.
- Ore di esercizio R1/R2 ed energia solare per giorno, settimana, mese, anno e totale (R1).
- Diagnostica: centralina online, ultimo riavvio, messaggi della centralina, programma.
- Ultimi valori conservati in RAM RTC dopo riavvio/OTA dell'ESP.
- Filtro degli slot del riepilogo (giorno della settimana / periodo corrente).
- Notifica di riavvio: automazione Home Assistant e flow Node-RED.
- Test di loopback per verificare ESP ↔ MCP2515.
