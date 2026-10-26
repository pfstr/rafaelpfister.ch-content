---
title: "Collegare digitalSTROM a Home Assistant: integrazione locale e automazioni"
navTitle: "digitalSTROM e HA"
description: "Un’integrazione locale di Home Assistant per il server digitalSTROM, con luci di movimento, modalità doccia, musica con 4× tocchi e uscita automatica tramite FRITZ!Box. Inclusi i problemi riscontrati nella pratica."
date: "2026-10-06"
kategorie: "Home Assistant e IoT"
timeToRead: "13 min di lettura"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant
slug: "collegare-digitalstrom-a-home-assistant-integrazione-locale-e-automazioni"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:23:08.969Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/collegare-digitalstrom-a-home-assistant-integrazione-locale-e-automazioni
---

Nelle installazioni digitalSTROM, luci, tapparelle e altri carichi sono collegati a morsetti nel quadro elettrico e controllati tramite pulsanti e un server digitalSTROM (dSS20). Quando si aggiungono altri sistemi come Philips Hue, Sonos e una FRITZ!Box, è naturale riunire tutto localmente in Home Assistant, senza un account cloud e senza memorizzare la password del dSS in Home Assistant. A questo scopo ho scritto una piccola integrazione e l’ho pubblicata, insieme alle automazioni adatte, come [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) con licenza MIT.

In breve: il dSS offre una valida API JSON con eventi. Usandola con moderazione, pulsanti, scene e le attività Uscita e Rientro arrivano in Home Assistant senza ritardi e possono quindi attivare automazioni che digitalSTROM da solo non conosce.

## Come funziona l’integrazione

Il dSS rende disponibile la sua API sulla porta 8080 tramite HTTPS, con un certificato autofirmato. digitalSTROM prevede token delle app per i programmi: il programma richiede un token, l’utente lo autorizza nel Configurator in Sistema > Autorizzazione di accesso, quindi il programma effettua l’accesso con esso. La password del dSS rimane presso l’utente.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-s` | nessuna barra di avanzamento |
| `-k` | accetta il certificato autofirmato del dSS |
| `requestApplicationToken` | crea un token dell’app che deve essere autorizzato nel Configurator |
| `loginApplication` | scambia il token dell’app autorizzato con un token di sessione |
| `event/subscribe` | sottoscrive un evento (`callScene`, `stateChange`, `buttonClick` …) con un ID scelto liberamente |
| `event/get` con `timeout` | attende fino a 30 secondi nuovi eventi (long poll) |

</details>

L’integrazione mantiene permanentemente aperta una connessione di long poll di questo tipo. Ogni scena attivata da un pulsante, dall’app o dal Configurator arriva come evento. Stati come «luce accesa nella stanza» o la presenza sono disponibili nella cache del dSS in `/usr/states` e possono essere interrogati senza gravare sui morsetti.

Le interrogazioni dirette dei valori di uscita (`device/getOutputValue`) passano invece attraverso il bus dS485 fino al morsetto e richiedono da mezzo secondo a un secondo per valore. Molte interrogazioni di questo tipo a intervalli brevi possono sovraccaricare i meter. Per questo l’integrazione legge direttamente solo le posizioni delle tapparelle e le uscite Joker, ogni 15 minuti e circa un minuto dopo un movimento.

| Piattaforma | Contenuto |
|---|---|
| `light` | una luce per stanza, comandata tramite scene della stanza come i pulsanti; luminosità con morsetti dimmerabili, solo acceso/spento con quelli commutati |
| `cover` | tapparelle con su, giù, stop e posizione |
| `scene` | atmosfere denominate nel Configurator |
| `button` | Uscita (scena 72) e Rientro (scena 71) |
| `binary_sensor` | presenza, allarme vento, rilevatori di movimento, uscite Joker |
| `sensor` | consumo totale |

Inoltre, l’integrazione segnala ogni scena della stanza nonché Uscita e Rientro a Home Assistant come evento `digitalstrom_local_event`. In questo modo è possibile assegnare liberamente 2× o 4× tocchi su un normale interruttore della luce.

## Automazioni come blueprint

Le seguenti automazioni sono incluse nel repository come blueprint e possono essere importate in Home Assistant tramite il pulsante di importazione nel README.

### Luce di movimento che non spegne una luce accesa manualmente

In caso di movimento la luce si accende e si spegne nuovamente dopo un tempo configurabile senza movimento. Facoltativamente ciò vale solo al di sotto di una soglia di luminosità, solo in una fascia oraria o di notte con luce attenuata. Se qualcuno accende la luce dal pulsante, questa rimane accesa. Un interruttore ausiliario (`input_boolean`) memorizza se l’automazione ha acceso la luce; solo in tal caso la spegne nuovamente.

Un dettaglio emerge solo durante l’uso: il trigger «rilevatore inattivo da 2 minuti» è un contatore che Home Assistant scarta a ogni riavvio. Se il rilevatore ha rilevato movimento per l’ultima volta prima del riavvio, non segue un nuovo passaggio a «inattivo» e la luce rimane accesa. Il blueprint verifica quindi anche ogni minuto se una luce accesa da esso è accesa nonostante il rilevatore sia inattivo da abbastanza tempo.

### Spegnere la luce quando si esce dalla stanza

Il tempo di spegnimento della luce di movimento è un compromesso: se troppo breve, la luce si spegne mentre qualcuno è fermo nella stanza; se troppo lungo, rimane accesa per minuti dopo l’uscita. Un altro blueprint utilizza quindi un secondo rilevatore esterno alla stanza, in genere nel corridoio. Se questo rileva movimento, il rilevatore nella stanza ha rilevato movimento poco prima (entro 30 secondi) e successivamente rimane inattivo, la luce nella stanza si spegne subito. Con rilevatori Hue, che segnalano «inattivo» circa 10 secondi dopo l’ultimo movimento, ciò avviene circa 10-15 secondi dopo essere usciti.

Per evitare che qualcuno rimanga al buio, la regola si applica solo a determinate condizioni: la luce deve essere stata accesa dall’automazione di movimento, la modalità doccia non deve essere attiva e in casa deve esserci al massimo una persona. Il numero di persone è determinato dai telefoni che Home Assistant riconosce come persone. La condizione dei 30 secondi protegge inoltre dal caso in cui qualcuno rimanga fermo a lungo nella stanza mentre un’altra persona passa nel corridoio.

### Modalità doccia con 2× tocchi

Nella doccia, un rilevatore di movimento di solito non rileva nessuno e, trascorso il tempo impostato, tutto diventa buio. 2× tocchi sul pulsante del bagno attivano in digitalSTROM l’atmosfera 2 (scena 17). L’automazione riconosce questa scena nella stanza e attiva una modalità doccia che blocca lo spegnimento. I movimenti nei primi 2 minuti vengono ignorati (spogliarsi, entrare). Il primo movimento successivo termina la modalità doccia; da quel momento torna a valere il normale tempo di spegnimento. Dopo una durata massima configurabile (20 minuti per impostazione predefinita), termina da sola.

### 4× tocchi avviano Sonos

Toccando rapidamente più volte un pulsante, digitalSTROM passa in sequenza attraverso le atmosfere: scena 5, 17, 18 e 19 al quarto tocco. Se si imposta la scena 19 nel Configurator su tutte le lampade su «non modificare l’uscita», essa è libera per altri scopi. Un blueprint reagisce a questa scena, spegne la luce della stanza e avvia l’altoparlante Sonos nella stanza oppure, nelle stanze senza altoparlante, più altoparlanti insieme.

Lo script presta attenzione a tre aspetti: se un altoparlante è già in funzione, la nuova stanza viene aggiunta al suo gruppo affinché la riproduzione resti sincronizzata. Se non è possibile riprendere nulla, riproduce una stazione radio dalla directory Radio Browser. E prima di ogni avvio viene impostato lo stesso volume iniziale, altrimenti una stanza suona piano e l’altra al volume a cui qualcuno aveva ascoltato musica l’ultima volta.

### Uscita e Rientro tramite FRITZ!Box

L’integrazione FRITZ!Box Tools segnala se un telefono è nella WLAN; valuta l’elenco dei dispositivi del router e copre quindi contemporaneamente la rete a 2,4 GHz, quella a 5 GHz e la LAN. Se non c’è più nessuno in casa e Uscita non è stato premuto, un blueprint attiva Uscita. Al rientro segue Rientro e, se è buio, una luce di benvenuto.

Inoltre, un’automazione può spegnere le lampade di altri sistemi (ad esempio Hue) e mettere in pausa Sonos in caso di Uscita. Il dSS spegne autonomamente le lampade digitalSTROM.

## Pianta come dashboard

Una visualizzazione chiara in Home Assistant è una pianta con la scheda integrata `picture-elements`. Per ogni stanza, un SVG trasparente viene sovrapposto alla pianta e sostituito da uno leggermente giallo quando la luce è accesa; un tocco sulla stanza accende o spegne la luce. Lampade, altoparlanti, tapparelle e rilevatori sono rappresentati da simboli nella loro posizione. Le istruzioni con una configurazione di esempio si trovano nel repository in `docs/floor-plan-dashboard.md`.

## Problemi riscontrati nella pratica

| Problema | Causa | Soluzione |
|---|---|---|
| Pulsante Uscita senza effetto | dSS non in rete, le attività su più circuiti elettrici passano attraverso il server | ripristinata la connessione di rete del dSS |
| Le tapparelle non reagiscono | allarme vento impostato su «attivo», sebbene non fosse presente alcun sensore del vento | attivata la scena 87 («nessun vento») per tutte le stanze |
| Le tapparelle non salgono con Uscita | scena 72 su «non modificare l’uscita» (`dontCare`) | `device/setSceneMode` con `dontCare=0`; il valore `false` è stato accettato, ma ignorato |
| Non è possibile regolare la luminosità della luce | tubo fluorescente su un morsetto commutato (modalità di uscita 35) | l’integrazione riconosce i morsetti commutati e offre solo acceso/spento |
| La luce rimane accesa dopo il riavvio | il contatore per «inattivo da X minuti» viene perso al riavvio | verifica aggiuntiva ogni minuto |
| Il telefono risulta assente | l’iPhone usa un indirizzo WLAN privato variabile e disconnette brevemente la WLAN in standby | indirizzo WLAN privato impostato su «Fisso», periodo di tolleranza di 10 minuti |
| DECT trasmette continuamente | «DECT Eco» non è disponibile non appena è registrato un dispositivo smart home FRITZ! | rinunciare alle prese DECT se si desidera DECT Eco |

Una ricerca dei dispositivi sul bridge Hue aggiunge tutti i dispositivi che si trovano in quel momento in modalità di abbinamento. Verificate poi nell’elenco dei dispositivi che siano state aggiunte solo le vostre lampade.

Le tabelle delle scene dei morsetti possono essere lette e modificate tramite l’API, ad esempio con `device/getSceneMode` e `device/saveScene`. Ognuna di queste chiamate passa attraverso il bus. Prima delle modifiche, salvate i vecchi valori, ad esempio in un file CSV, per poterli ripristinare se necessario.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornato il link delle Fonti al nuovo indirizzo della banca dati FRITZ!.

## Fonti

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): integrazione e blueprint di questo articolo, licenza MIT.

2.  [digitalSTROM: manuale d’uso e configurazione](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): attività Uscita (tenere premuto per 3 secondi), impostazione «non modificare l’uscita» nel capitolo 3.5.2.

3.  [digitalSTROM: documentazione dei pulsanti dei morsetti](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): impostazioni di fabbrica delle funzioni dei pulsanti per tipo di morsetto.

4.  [digitalSTROM: istruzioni per l’uso](https://www.digitalstrom.com/bedienungsanleitungen/): panoramica dei manuali, documentazione per progettazione e installazione.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): rilevamento della presenza, interruttori WLAN, opzioni dell’integrazione.

6.  [Home Assistant: scheda Picture Elements](https://www.home-assistant.io/dashboards/picture-elements/): base del dashboard della pianta.

7.  [Home Assistant: blueprint](https://www.home-assistant.io/docs/automation/using_blueprints/): importazione e utilizzo dei blueprint.

8.  [Banca dati FRITZ!: la presa FRITZ! perde la connessione](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indicazioni sulla portata DECT e sulla potenza radio delle prese smart home.

9.  [Radio Browser](https://www.radio-browser.info/): directory libera di flussi radio, integrata in Home Assistant come sorgente multimediale.
