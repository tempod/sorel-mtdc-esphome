# Com'è nato il progetto e come ci siamo arrivati

## Il punto di partenza

L'impianto: un bollitore per l'acqua calda sanitaria scaldato da pannelli solari termici,
con una resistenza elettrica da 1500 W come integrazione, il tutto gestito da una centralina
**SOREL MTDC v5**. La centralina funziona bene, ma è un'isola: i dati si vedono solo sul display.

Due esigenze:
1. portare temperature, stato dei relè, potenza ed energia solare in **Home Assistant**, con **ESPHome**;
2. risolvere un fastidio: dopo ogni **blackout** la MTDC chiede di confermare data e ora,
   e se nessuno lo fa le fasce orarie rischiano di non funzionare come previsto.

La MTDC ha due prese **CAN** (pensate per collegare più centraline o il Data Logger SOREL).
Il protocollo (SCBI, *SOREL CAN Bus Interface*) non è pubblico, ma esisteva già un decoder Python,
[ddimension/sorel-canbus](https://github.com/ddimension/sorel-canbus), da cui partire.

## Le tappe

### 1. Primo firmware e primo muro
Prima configurazione ESPHome con un Wemos D1 mini + MCP2515, poi passata a una NodeMCU v3.
Il codice ascoltava solo la sonda S1. Risultato: **nessun dato**, anche invertendo H e L.

### 2. Dividere il problema
- **Test di loopback**: il modulo trasmette e riceve da solo → SPI e alimentazione OK.
- **Quarzo**: 8 MHz, corretto.
- **Terminazione**: tra H e L lato centralina il multimetro non segnava nulla → la centralina
  non aveva terminazione. Non avendo una resistenza, è stato usato un **secondo MCP2515 spento
  con il ponticello chiuso** come resistenza da 120 Ω.

Risultato immediato: il bus si è animato. Era la terminazione.

### 3. Leggere il traffico
Uno sniffer nel log ha mostrato cosa trasmette la centralina: presenza ogni 60 s, relè, statistiche,
riepiloghi… ma all'inizio **nessuna temperatura**. Si è pensato servissero richieste come un Data Logger;
le richieste provate non hanno avuto risposta, ma poco dopo le sonde sono arrivate da sole.
Con un log migliorato (secondi trascorsi tra un invio e l'altro) è emerso lo schema:
**invio a ogni variazione + reinvio ogni ~5 minuti**.

### 4. Decodificare ore ed energia
I riepiloghi avevano numeri come `w=769` e `w=2581`. Confrontandoli con il menu *Statistiche* del display
(R1 anno 769 h, R2 anno 2,58 kh) la struttura è diventata chiara: tipo di periodo, campo (R1/R2), ore.
Un indizio laterale: il "cambio giorno" avveniva alle 00:52 → **l'orologio della centralina era indietro**.

Attivata la **contabilizzazione del calore** (menu 6.6) e aspettata una giornata di sole:
`STAT id=1` è risultata la potenza in W, il secondo valore del riepilogo i **kWh**.
La misteriosa sonda "S5 = 413 °C" era la stessa potenza vista come sonda virtuale.

### 5. Rifinire
- Log ripulito: solo ciò che cambia.
- Ore pubblicate solo quando cambiano (la centralina ripete il riepilogo ogni 10 s).
- Ultimi valori conservati in RAM RTC: dopo un riavvio dell'ESP niente "sconosciuto".
- Sensori aggiunti dal bus: centralina online, velocità pompa, messaggi, programma.
- Bug corretti strada facendo:
  - "Centralina online" che lampeggiava: un `millis() | 1` finiva 1 ms nel futuro e la sottrazione
    senza segno andava in overflow → differenza con segno;
  - ore di "oggi" e "settimana" che ballavano il lunedì: la centralina invia anche gli slot appena chiusi
    → filtro per indice (giorno della settimana per i giorni, 0 per gli altri periodi).

### 6. Data e ora: il tentativo e la scelta
Con un log grezzo è stato registrato tutto il traffico durante un blackout simulato:
- all'accensione la centralina invia **"mi sono riavviata"** (`0x1008000B`) e il messaggio **"Riavvio"**;
- impostando data e ora dal display **non passa nulla sul bus**;
- le richieste di lettura dell'ora (anche nel formato usato dalla centralina) **non hanno risposta**.

Scrivere l'ora alla cieca avrebbe rischiato di corrompere le statistiche (il manuale lo segnala
esplicitamente), quindi ci si è fermati. Soluzione adottata: il sensore **"Ultimo riavvio centralina"**
e una **notifica** in Home Assistant (automazione o flow Node-RED) che ricorda di confermare l'ora.

## Cosa resta aperto
- Scrittura di data/ora: servirebbe la documentazione SCBI o un Data Logger SOREL da "ascoltare".
- Totale ore R2: la centralina non lo trasmette.
- Significato di alcuni frame minori (statistiche 0/4/7/8, riepilogo tipo 6/7, prog 0x81).

Se hai una MTDC (o un'altra centralina SOREL) e riesci ad aggiungere un pezzo, le pull request sono benvenute.
