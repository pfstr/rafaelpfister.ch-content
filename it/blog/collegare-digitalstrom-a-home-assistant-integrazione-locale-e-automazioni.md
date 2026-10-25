---
title: "Collegare digitalSTROM a Home Assistant: integrazione locale e automazioni"
navTitle: "digitalSTROM e HA"
description: "Un’integrazione locale di Home Assistant per il server digitalSTROM, con luci a movimento, modalità doccia, musica con 4× tocchi e uscita automatica tramite FRITZ!Box. Inclusi i problemi emersi nella pratica."
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
translationSourceHash: 36ff960cb636859ab9f7ed1cccb3152c61e6657a074546cef23685dbde2ec8fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:48:21.092Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/collegare-digitalstrom-a-home-assistant-integrazione-locale-e-automazioni
---

Nelle installazioni digitalSTROM, luci, tapparelle e altri carichi sono collegati a morsetti nel quadro elettrico e controllati tramite pulsanti e un server digitalSTROM (dSS20). Quando si aggiungono altri sistemi come Philips Hue, Sonos e una FRITZ!Box, è naturale riunire tutto localmente in Home Assistant, senza un account cloud e senza salvare in Home Assistant la password del dSS. A questo scopo ho scritto una piccola integrazione e l’ho pubblicata, insieme alle automazioni appropriate, come [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) con licenza MIT.

In breve: il dSS offre una valida API JSON con eventi. Usandola con moderazione, è possibile portare in Home Assistant pulsanti, scene e le attività Uscita e Arrivo senza ritardi, per attivare automazioni che digitalSTROM da solo non conosce.

## Come funziona l’integrazione

Il dSS rende disponibile la propria API via HTTPS sulla porta 8080, con un certificato autofirmato. Per i programmi digitalSTROM prevede token per app: il programma richiede un token, l’utente lo autorizza nel Configurator in Sistema > Autorizzazione di accesso, quindi il programma effettua l’accesso con esso. La password del dSS rimane all’utente.

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
| `-s` | nessun indicatore di avanzamento |
| `-k` | accetta il certificato autofirmato del dSS |
| `requestApplicationToken` | crea un token per app che deve essere autorizzato nel Configurator |
| `loginApplication` | scambia il token per app autorizzato con un token di sessione |
| `event/subscribe` | sottoscrive un evento (`callScene`, `stateChange`, `buttonClick` …) con un ID scelto liberamente |
| `event/get` con `timeout` | attende fino a 30 secondi nuovi eventi (long polling) |

</details>

L’integrazione mantiene permanentemente aperta una connessione di long polling di questo tipo. Ogni scena attivata da un pulsante, dall’app o dal Configurator arriva come evento. Stati quali «luce accesa nella stanza» o la presenza sono disponibili nella cache del dSS in `/usr/states` e possono essere interrogati senza gravare sui morsetti.

Le interrogazioni dirette dei valori di uscita (`device/getOutputValue`) passano invece attraverso il bus dS485 fino al morsetto e richiedono da mezzo secondo a un secondo per valore. Molte interrogazioni di questo tipo a intervalli brevi possono sovraccaricare i meter. L’integrazione legge quindi direttamente solo le posizioni delle tapparelle e le uscite Joker, ogni 15 minuti e circa un minuto dopo un movimento.

| Piattaforma | Contenuto |
|---|---|
| `light` | una luce per stanza, controllata tramite scene della stanza come i pulsanti; luminosità con morsetti dimmerabili, solo acceso/spento con quelli commutati |
| `cover` | tapparelle con su, giù, stop e posizione |
| `scene` | atmosfere nominate nel Configurator |
| `button` | Uscita (scena 72) e Arrivo (scena 71) |
| `binary_sensor` | presenza, allarme vento, sensori di movimento, uscite Joker |
| `sensor` | consumo totale |

Inoltre, l’integrazione segnala ogni scena della stanza, nonché Uscita e Arrivo, come evento `digitalstrom_local_event` a Home Assistant. In questo modo è possibile assegnare liberamente 2× o 4× tocchi su un normale interruttore della luce.

## Automazioni come blueprint

Le seguenti automazioni sono incluse nel repository come blueprint e possono essere importate in Home Assistant tramite il pulsante di importazione nel README.

### Luce a movimento che non spegne la luce accesa manualmente

Quando viene rilevato un movimento, la luce si accende e si spegne di nuovo dopo un tempo configurabile senza movimento. Facoltativamente, ciò avviene solo sotto una soglia di luminosità, solo in una fascia oraria oppure con luce attenuata di notte. Se qualcuno accende la luce dal pulsante, rimane accesa. Un interruttore di supporto (`input_boolean`) ricorda se è stata l’automazione ad accendere la luce; solo in quel caso la spegne di nuovo.

Un dettaglio emerge solo durante l’uso: il trigger «sensore inattivo da 2 minuti» è un contatore che Home Assistant elimina a ogni riavvio. Se il sensore ha rilevato l’ultimo movimento prima del riavvio, non segue un nuovo passaggio a «inattivo» e la luce rimane accesa. Il blueprint verifica quindi anche ogni minuto se una luce accesa da esso è ancora accesa benché il sensore sia inattivo da abbastanza tempo.

### Spegnere la luce quando si esce dalla stanza

Il tempo di spegnimento della luce a movimento è un compromesso: troppo breve, e la luce si spegne mentre qualcuno è fermo nella stanza; troppo lungo, e rimane accesa per minuti dopo l’uscita. Un altro blueprint utilizza quindi un secondo sensore fuori dalla stanza, tipicamente nel corridoio. Se rileva movimento, il sensore nella stanza aveva rilevato movimento poco prima (entro 30 secondi) e poi rimane inattivo, la luce nella stanza si spegne immediatamente. Con i sensori Hue, che segnalano «inattivo» circa 10 secondi dopo l’ultimo movimento, ciò avviene all’incirca da 10 a 15 secondi dopo l’uscita.

Per evitare che qualcuno resti al buio, la regola interviene solo a determinate condizioni: la luce deve essere stata accesa dall’automazione di movimento, una modalità doccia non deve essere attiva e può esserci al massimo una persona in casa. Il numero di persone deriva dai telefoni che Home Assistant riconosce come persone. La condizione dei 30 secondi protegge inoltre dal caso in cui qualcuno rimanga fermo a lungo nella stanza e un’altra persona passi nel corridoio.

### Modalità doccia con 2× tocchi

Nella doccia, un sensore di movimento spesso non rileva nessuno e, dopo il tempo impostato, tutto diventa buio. 2× tocchi sul pulsante del bagno attivano in digitalSTROM l’atmosfera 2 (scena 17). L’automazione riconosce questa scena nella stanza e attiva una modalità doccia che blocca lo spegnimento. I movimenti nei primi 2 minuti vengono ignorati (spogliarsi, entrare). Il primo movimento successivo termina la modalità doccia, dopodiché si applica di nuovo il normale tempo di spegnimento. Dopo una durata massima configurabile (20 minuti per impostazione predefinita), termina automaticamente.

### 4× tocchi avviano Sonos

Toccando rapidamente più volte un pulsante, digitalSTROM scorre in sequenza le atmosfere: scena 5, 17, 18 e 19 al quarto tocco. Se nel Configurator si imposta la scena 19 su «non modificare l’uscita» per tutte le lampade, essa è libera per altri scopi. Un blueprint reagisce a questa scena, spegne la luce della stanza e avvia l’altoparlante Sonos nella stanza oppure, nelle stanze senza altoparlante, più altoparlanti insieme.

Lo script presta attenzione a tre punti: se un altoparlante è già in riproduzione, la nuova stanza viene aggiunta al suo gruppo affinché la riproduzione resti sincronizzata. Se non è possibile riprendere nulla, viene riprodotta una stazione radio dalla directory Radio Browser. E prima di ogni avvio viene impostato lo stesso volume iniziale, altrimenti una stanza suona piano e l’altra al volume al quale qualcuno ha ascoltato musica l’ultima volta.

### Uscita e Arrivo tramite FRITZ!Box

L’integrazione FRITZ!Box Tools segnala se un telefono è nella WLAN; valuta l’elenco dei dispositivi del router e copre quindi contemporaneamente la rete a 2,4 GHz, quella a 5 GHz e la LAN. Se non c’è più nessuno in casa e Uscita non è stato premuto, un blueprint attiva Uscita. Al rientro segue Arrivo e, se è buio, una luce di benvenuto.

In aggiunta, un’automazione può spegnere le lampade di altri sistemi (per esempio Hue) e mettere in pausa Sonos quando si attiva Uscita. Le lampade digitalSTROM vengono spente dal dSS stesso.

## Pianta come dashboard

Una rappresentazione chiara in Home Assistant è una pianta con la scheda integrata `picture-elements`. Per ogni stanza viene sovrapposto alla pianta un SVG trasparente, sostituito da uno leggermente giallo quando la luce è accesa; toccando la stanza si accende o spegne la luce. Lampade, altoparlanti, tapparelle e sensori sono rappresentati da simboli nella loro posizione. Le istruzioni con una configurazione di esempio si trovano nel repository in `docs/floor-plan-dashboard.md`.

## Problemi dalla pratica

| Problema | Causa | Soluzione |
|---|---|---|
| Pulsante Uscita senza effetto | dSS non in rete, le attività su più circuiti elettrici passano dal server | ripristinata la connessione di rete del dSS |
| Le tapparelle non reagiscono | allarme vento impostato su «attivo», nonostante non fosse presente alcun sensore del vento | attivata la scena 87 («nessun vento») per tutte le stanze |
| Le tapparelle non si alzano con Uscita | scena 72 su «non modificare l’uscita» (`dontCare`) | `device/setSceneMode` con `dontCare=0`; il valore `false` veniva accettato, ma ignorato |
| Non è possibile regolare la luminosità della luce | tubo fluorescente su un morsetto commutato (modalità uscita 35) | l’integrazione rileva i morsetti commutati e offre lì solo acceso/spento |
| La luce rimane accesa dopo il riavvio | il contatore per «inattivo da X minuti» viene perso al riavvio | verifica aggiuntiva ogni minuto |
| Il telefono risulta assente | iPhone usa un indirizzo WLAN privato variabile e disconnette brevemente la WLAN in standby | indirizzo WLAN privato impostato su «Fisso», periodo di tolleranza di 10 minuti |
| DECT trasmette continuamente | «DECT Eco» non è disponibile non appena viene registrato un dispositivo FRITZ! Smart Home | rinunciare alle prese DECT se si desidera DECT Eco |

Una ricerca dei dispositivi sul bridge Hue aggiunge tutti i dispositivi che si trovano in quel momento in modalità di associazione. Controllare quindi nell’elenco dei dispositivi che siano state aggiunte solo le proprie lampade.

Le tabelle delle scene dei morsetti possono essere lette e modificate tramite l’API, ad esempio con `device/getSceneMode` e `device/saveScene`. Ognuna di queste chiamate passa dal bus. Prima delle modifiche, salvare i vecchi valori, per esempio in un file CSV, per poterli ripristinare se necessario.

## Fonti

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): integrazione e blueprint di questo articolo, licenza MIT.

2.  [digitalSTROM: Manuale d’uso e configurazione](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): attività Uscita (tenere premuto per 3 secondi), impostazione «non modificare l’uscita» nel capitolo 3.5.2.

3.  [digitalSTROM: Documentazione dei pulsanti dei morsetti](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): impostazioni di fabbrica delle funzioni dei pulsanti per tipo di morsetto.

4.  [digitalSTROM: Manuali d’uso](https://www.digitalstrom.com/bedienungsanleitungen/): panoramica dei manuali, documentazione per progettisti e installatori.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): rilevamento della presenza, interruttori WLAN, opzioni dell’integrazione.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): base della dashboard della pianta.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importazione e utilizzo dei blueprint.

8.  [FRITZ! Knowledge Base: La presa FRITZ! perde la connessione](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indicazioni sulla portata e sulla potenza radio DECT delle prese Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): directory libera di flussi radio, integrata in Home Assistant come sorgente multimediale.
