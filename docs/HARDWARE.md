# Hardware e collegamenti

## Componenti

| Componente | Note |
|---|---|
| ESP8266: **NodeMCU v3** (LoLin, CH340) oppure **Wemos D1 mini** | Il progetto è stato sviluppato e testato su NodeMCU v3 |
| Modulo CAN **MCP2515 + TJA1050** (il classico modulo blu) | Controllare il quarzo: questo progetto usa **8 MHz** |
| Cavo CAN verso la centralina | Il manuale MTDC indica una lunghezza massima di **3 m** |
| Resistenza da **120 Ω** (oppure un secondo modulo MCP2515, vedi sotto) | Per la terminazione lato centralina |
| Alimentatore USB 5 V per l'ESP | |

## Schema dei collegamenti

```
  NodeMCU v3                 MCP2515                    SOREL MTDC
  ──────────                 ───────                    ──────────
  VU (5V USB) ─────────────► VCC
  G           ─────────────► GND
  D5 (GPIO14) ─────────────► SCK
  D7 (GPIO13) ─────────────► SI
  D6 (GPIO12) ◄───────────── SO
  D8 (GPIO15) ─────────────► CS
                             INT   (non collegato)
                             H  ◄──────────────────────► CAN-H  ┐ presa CAN1
                             L  ◄──────────────────────► CAN-L  ┘ o CAN2
                             J1 = chiuso (120 Ω)        120 Ω sull'altra presa CAN
```

Con il **Wemos D1 mini** i pin hanno gli stessi nomi (D5–D8); i 5 V si prendono dal pin `5V`.
Nella configurazione basta cambiare `board: nodemcuv2` in `board: d1_mini`.

## Note importanti

### Terminazione del bus (fondamentale)
Un bus CAN vuole una resistenza da 120 Ω **a ciascuno dei due capi**.
Con la sola resistenza del modulo (J1) il bus **non funzionava**: nessun frame ricevuto.
Aggiunta la seconda terminazione lato centralina, i dati sono arrivati subito.

- Sul modulo MCP2515: ponticello **J1 chiuso**.
- Lato centralina: 120 Ω tra H e L (sulla seconda presa CAN o sui morsetti).
  Un trucco se non si ha una resistenza: un **secondo modulo MCP2515 non alimentato**,
  con J1 chiuso e collegato solo con H e L, funziona da resistenza da 120 Ω.
- **Verifica**: a tutto spento, tra H e L si devono misurare **circa 60 Ω**.
  120 Ω = manca una terminazione; circuito aperto = nessuna terminazione.

### Livelli logici 5 V / 3,3 V
Il modulo blu è alimentato a 5 V (il TJA1050 lo richiede), quindi il pin **SO** del MCP2515
porta 5 V verso D6 dell'ESP8266, che lavora a 3,3 V. In pratica funziona, ma è fuori specifica.
Per un'installazione definitiva si consiglia un partitore sul solo SO (1 kΩ in serie + 2 kΩ verso GND),
oppure un modulo con transceiver a 3,3 V (es. SN65HVD230).

### Pin D8 (GPIO15) e avvio
D8 deve essere basso all'accensione. Se la scheda non si avvia o non si lascia programmare,
scollegare CS durante il caricamento oppure spostare CS su un altro pin (es. D2) e aggiornare `cs_pin`.

### Il LED del modulo
Il LED rosso acceso fisso sul MCP2515 è solo la spia di alimentazione, non indica errori.

### Quarzo
Leggere la scritta sul quarzo argentato del modulo: `8.000` → `clock: 8MHZ`, `16.000` → `clock: 16MHZ`.
Con il valore sbagliato la velocità del bus risulta errata e non si riceve nulla.

## Procedura di diagnosi consigliata

1. **ESP ↔ MCP2515**: caricare [`esphome/test-loopback.yaml`](../esphome/test-loopback.yaml)
   con la centralina scollegata. Ogni 5 s nel log deve comparire `ricevuto ... DE AD BE EF`.
   Se non compare: controllare SPI (D5–D8) e alimentazione.
2. **Quarzo**: verificare 8 / 16 MHz (il loopback non lo rileva).
3. **Terminazione**: misurare ~60 Ω tra H e L a tutto spento.
4. **H e L**: se ancora non arriva nulla, provare a invertirli (non si danneggia niente).
5. Caricare la configurazione principale: entro pochi secondi devono arrivare i riepiloghi
   (la centralina li ripete ogni ~10 s) e il sensore "Centralina online" passa a Connesso.
