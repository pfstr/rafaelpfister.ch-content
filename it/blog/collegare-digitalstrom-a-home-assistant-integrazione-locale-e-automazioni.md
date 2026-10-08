---
title: "Collegare digitalSTROM a Home Assistant: integrazione locale e automazioni"
navTitle: "digitalSTROM e HA"
description: "Un'integrazione locale di Home Assistant per il server digitalSTROM, con luce a rilevamento di movimento, modalità doccia, musica con 4× tocchi e uscita automatica tramite FRITZ!Box. Inclusi i problemi emersi nella pratica."
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
url: https://rafaelpfister.ch/it/blog/collegare-digitalstrom-a-home-assistant-integrazione-locale-e-automazioni
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:24:25.436Z
translationReview: required
---

Nelle installazioni digitalSTROM, luci, tapparelle e altre utenze sono collegate a morsetti nel quadro elettrico e controllate tramite pulsanti e un server digitalSTROM (dSS20). Quando si aggiungono altri sistemi come Philips Hue, Sonos e una FRITZ!Box, è naturale riunire tutto localmente in Home Assistant, senza un account cloud e senza memorizzare la password del dSS in Home Assistant. A questo scopo ho scritto una piccola integrazione e l'ho pubblicata, insieme alle relative automazioni, come [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) con licenza MIT.

In breve: il dSS offre una JSON API utilizzabile con eventi. Usandola con cautela, pulsanti, scene e le attività Uscita e Arrivo arrivano in Home Assistant senza ritardi, consentendo di attivare automazioni che digitalSTROM da solo non conosce.

## Come funziona l'integrazione

Il dSS mette a disposizione la sua API sulla porta 8080 tramite HTTPS, con un certificato autofirmato. Per i programmi digitalSTROM prevede token per app: il programma richiede un token, l'utente lo autorizza nel Configurator in Sistema > Autorizzazione di accesso, dopodiché il programma effettua l'accesso con esso. La password del dSS resta all'utente.

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
| `-s` | nessuna indicazione di avanzamento |
| `-k` | accetta il certificato autofirmato del dSS |
| `requestApplicationToken` | crea un token per app che deve essere autorizzato nel Configurator |
| `loginApplication` | scambia il token per app autorizzato con un token di sessione |
| `event/subscribe` | sottoscrive un evento (`callScene`, `stateChange`, `buttonClick` …) con un ID scelto liberamente |
| `event/get` con `timeout` | attende fino a 30 secondi nuovi eventi (long poll) |

</details>

L'integrazione mantiene permanentemente aperta una connessione long poll di questo tipo. Ogni scena attivata da un pulsante, dall'app o dal Configurator arriva come evento. Stati quali «luce accesa nella stanza» o la presenza sono disponibili nella cache del dSS in `/usr/states` e possono essere interrogati senza gravare sui morsetti.

Le interrogazioni dirette dei valori di uscita (`device/getOutputValue`) passano invece attraverso il bus dS485 fino al morsetto e richiedono da mezzo secondo a un secondo per valore. Molte interrogazioni di questo tipo a intervalli brevi possono sovraccaricare i meter. Per questo l'integrazione legge direttamente solo le posizioni delle tapparelle e le uscite Joker, ogni 15 minuti e circa un minuto dopo un movimento.

| Piattaforma | Contenuto |
|---|---|
| `light` | una luce per stanza, comandata tramite scene della stanza come i pulsanti; luminosità con morsetti dimmerabili, solo acceso/spento con quelli commutati |
| `cover` | tapparelle con apertura, chiusura, stop e posizione |
| `scene` | atmosfere denominate nel Configurator |
| `button` | Uscita (scena 72) e Arrivo (scena 71) |
| `binary_sensor` | presenza, allarme vento, rilevatori di movimento, uscite Joker |
| `sensor` | consumo totale |

Inoltre, l'integrazione segnala a Home Assistant ogni scena della stanza nonché Uscita e Arrivo come evento `digitalstrom_local_event`. In questo modo è possibile assegnare liberamente 2× o 4× tocchi su un normale interruttore della luce.

## Automazioni come blueprint

Le seguenti automazioni sono incluse nel repository come blueprint e possono essere importate in Home Assistant tramite il pulsante di importazione nel README.

### Luce con rilevamento di movimento che non spegne una luce accesa manualmente

In caso di movimento la luce si accende e, dopo un tempo configurabile senza movimento, si spegne di nuovo. Facoltativamente, questo vale solo sotto una soglia di luminosità, solo in una fascia oraria o con luce attenuata di notte. Se qualcuno accende la luce dal pulsante, resta accesa. Un interruttore di supporto (`input_boolean`) memorizza se l'automazione ha acceso la luce; solo in tal caso la spegne di nuovo.

Un dettaglio emerge solo durante il funzionamento: il trigger «rilevatore inattivo da 2 minuti» è un contatore che Home Assistant scarta a ogni riavvio. Se il rilevatore ha rilevato movimento per l'ultima volta prima del riavvio, non avviene una nuova transizione a «inattivo» e la luce resta accesa. Per questo il blueprint verifica inoltre ogni minuto se una luce da esso accesa è ancora accesa nonostante il rilevatore sia inattivo da abbastanza tempo.

### Spegnere la luce quando si esce dalla stanza

Il tempo di spegnimento della luce con rilevamento di movimento è un compromesso: troppo breve e la luce si spegne mentre qualcuno resta immobile nella stanza; troppo lungo e rimane accesa per minuti dopo l'uscita. Un ulteriore blueprint utilizza quindi un secondo rilevatore fuori dalla stanza, in genere nel corridoio. Se rileva movimento, il rilevatore nella stanza ha rilevato movimento poco prima (entro 30 secondi) e successivamente lì resta inattivo, la luce nella stanza si spegne subito. Con i rilevatori Hue, che segnalano «inattivo» circa 10 secondi dopo l'ultimo movimento, ciò avviene all'incirca 10-15 secondi dopo essere usciti.

Per evitare che qualcuno resti al buio, la regola si applica solo a determinate condizioni: la luce deve essere stata accesa dall'automazione del movimento, la modalità doccia non deve essere attiva e in casa può esserci al massimo una persona. Il numero di persone deriva dai telefoni che Home Assistant riconosce come persone. La condizione dei 30 secondi protegge inoltre dal caso in cui qualcuno resti immobile a lungo nella stanza e un'altra persona passi nel corridoio.

### Modalità doccia con 2× tocchi

Nella doccia, un rilevatore di movimento solitamente non rileva nessuno e, trascorso il tempo impostato, diventa buio. Con 2× tocchi sul pulsante del bagno, digitalSTROM attiva l'atmosfera 2 (scena 17). L'automazione riconosce questa scena nella stanza e attiva una modalità doccia che blocca lo spegnimento. I movimenti nei primi 2 minuti vengono ignorati (spogliarsi, entrare). Il primo movimento successivo termina la modalità doccia; da quel momento vale di nuovo il normale tempo di spegnimento. Dopo una durata massima configurabile (standard 20 minuti), termina automaticamente.

### 4× tocchi avviano Sonos

Toccando rapidamente più volte un pulsante, digitalSTROM attiva in sequenza le atmosfere: scena 5, 17, 18 e, al quarto tocco, 19. Se nel Configurator si imposta la scena 19 su «non modificare l'uscita» per tutte le lampade, questa resta libera per altri scopi. Un blueprint reagisce a questa scena, spegne la luce della stanza e avvia l'altoparlante Sonos nella stanza oppure, nelle stanze senza altoparlante, più altoparlanti insieme.

Lo script tiene conto di tre aspetti: se un altoparlante è già in funzione, alla sua gruppo viene aggiunta la nuova stanza, affinché la riproduzione resti sincronizzata. Se non è possibile riprendere nulla, viene riprodotta una stazione radio dalla directory Radio Browser. E prima di ogni avvio viene impostato lo stesso volume iniziale, altrimenti una stanza riproduce a basso volume e l'altra al volume a cui qualcuno ha ascoltato musica l'ultima volta.

### Uscita e Arrivo tramite FRITZ!Box

L'integrazione FRITZ!Box Tools segnala se un telefono è nella WLAN; valuta l'elenco dei dispositivi del box e copre quindi contemporaneamente rete a 2,4 GHz, rete a 5 GHz e LAN. Se non c'è più nessuno in casa e non è stato premuto Uscita, un blueprint attiva Uscita. Al rientro segue Arrivo e, se è buio, una luce di benvenuto.

In aggiunta, un'automazione può spegnere le lampade di altri sistemi (per esempio Hue) e mettere in pausa Sonos all'uscita. Il dSS spegne autonomamente le lampade digitalSTROM.

## Pianta come dashboard

Una rappresentazione chiara in Home Assistant è una pianta con la scheda integrata `picture-elements`. Per ogni stanza, un SVG trasparente si sovrappone alla pianta e viene sostituito da uno leggermente giallo quando la luce è accesa; toccando la stanza si accende o spegne la luce. Lampade, altoparlanti, tapparelle e rilevatori sono posizionati come simboli nella loro ubicazione. Le istruzioni con una configurazione di esempio sono disponibili nel repository in `docs/floor-plan-dashboard.md`.

## Problemi riscontrati nella pratica

| Problema | Causa | Soluzione |
|---|---|---|
| Pulsante Uscita senza effetto | dSS non in rete, le attività su più circuiti elettrici passano dal server | ripristinata la connessione di rete del dSS |
| Le tapparelle non reagiscono | l'allarme vento era impostato su «attivo», benché non fosse presente alcun sensore del vento | attivata la scena 87 («nessun vento») per tutte le stanze |
| Le tapparelle non si alzano all'uscita | scena 72 su «non modificare l'uscita» (`dontCare`) | `device/setSceneMode` con `dontCare=0`; il valore `false` è stato accettato, ma ignorato |
| Non è possibile regolare l'intensità della luce | tubo fluorescente su un morsetto commutato (modalità di uscita 35) | l'integrazione riconosce i morsetti commutati e offre solo acceso/spento |
| La luce resta accesa dopo il riavvio | il contatore per «inattivo da X minuti» si perde al riavvio | controllo aggiuntivo ogni minuto |
| Il telefono risulta assente | iPhone usa un indirizzo WLAN privato variabile e in stato di riposo disconnette brevemente la WLAN | indirizzo WLAN privato impostato su «Fisso», periodo di tolleranza di 10 minuti |
| DECT trasmette continuamente | «DECT Eco» non è disponibile non appena è registrato un dispositivo FRITZ! Smart Home | rinunciare alle prese DECT se si desidera DECT Eco |

Una ricerca di dispositivi sul bridge Hue aggiunge tutti i dispositivi che si trovano in quel momento in modalità di associazione. Verificare poi nell'elenco dei dispositivi che siano state aggiunte solo le proprie lampade.

Le tabelle delle scene dei morsetti possono essere lette e modificate tramite l'API, ad esempio con `device/getSceneMode` e `device/saveScene`. Ciascuna di queste chiamate passa attraverso il bus. Prima di apportare modifiche, salvare i vecchi valori, per esempio in un file CSV, per poterli ripristinare se necessario.

## Fonti

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): integrazione e blueprint di questo articolo, licenza MIT.

2.  [digitalSTROM: Manuale d'uso e configurazione](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): attività Uscita (tenere premuto per 3 secondi), impostazione «non modificare l'uscita» nel capitolo 3.5.2.

3.  [digitalSTROM: Documentazione dei pulsanti dei morsetti](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): impostazioni di fabbrica delle funzioni dei pulsanti per tipo di morsetto.

4.  [digitalSTROM: Manuali d'uso](https://www.digitalstrom.com/bedienungsanleitungen/): panoramica dei manuali, documenti per progettazione e installazione.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): rilevamento della presenza, interruttori WLAN, opzioni dell'integrazione.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): base della dashboard della pianta.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importazione e utilizzo dei blueprint.

8.  [FRITZ! Knowledge Base: la presa FRITZ! perde la connessione](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indicazioni sulla portata DECT e sulla potenza di trasmissione delle prese Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): directory libera di flussi radio, integrata in Home Assistant come sorgente multimediale.
