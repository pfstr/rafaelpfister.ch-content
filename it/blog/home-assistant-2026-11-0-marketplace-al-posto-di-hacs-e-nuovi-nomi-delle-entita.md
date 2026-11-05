---
title: "Home Assistant 2026.11.0: Marketplace al posto di HACS e nuovi nomi delle entità"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 integra HACS come Marketplace, ricompone i nomi delle entità a partire da dispositivo ed entità e introduce numerose modifiche non compatibili con le versioni precedenti. Prima dell’aggiornamento, controllate dashboard, template e il rilevamento della presenza tramite FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant e IoT"
timeToRead: "6 min di lettura"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-al-posto-di-hacs-e-nuovi-nomi-delle-entita"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/it/blog/home-assistant-2026-11-0-marketplace-al-posto-di-hacs-e-nuovi-nomi-delle-entita
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:24:32.401Z
translationReview: automatic
---

# Home Assistant 2026.11.0: Marketplace al posto di HACS e nuovi nomi delle entità

Home Assistant 2026.11.0 è stato rilasciato il 4 novembre 2026 e segue la versione 2026.10.4. La release integra l’ex Home Assistant Community Store (HACS) come Marketplace direttamente in Home Assistant, modifica la formazione dei nomi delle entità e contiene un lungo elenco di modifiche non compatibili con le versioni precedenti, tra cui modifiche ai Device Tracker di FRITZ!Box Tools e all’autenticazione tramite indirizzi non locali. Secondo il blog della release, vengono aggiunte 38 nuove integrazioni. Le note di rilascio non menzionano una correzione di sicurezza con CVE. La release è rivolta a tutti gli operatori di Home Assistant, in particolare alle installazioni con Custom Integrations da HACS, come quelle usate negli articoli su [Midea PortaSplit](/blog/midea-portasplit-home-assistant) e [digitalSTROM](/blog/digitalstrom-home-assistant).

## Le versioni in sintesi

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Release mensile | HACS diventa il Marketplace, nuovi nomi delle entità, Device Tracker degli scanner di rete sempre associati a un dispositivo, controllo dell’autenticazione per indirizzi non locali, nessuna cartella `deps` più, 38 nuove integrazioni |

## Marketplace: HACS diventa parte di Home Assistant

Con la versione 2026.11, secondo il blog della release, HACS diventa parte di Home Assistant e ora si chiama Marketplace. Gli amministratori lo trovano in Impostazioni > Marketplace e vi installano integrazioni, schede dashboard, temi e modelli; non è necessario un account GitHub. Dopo l’installazione di un’integrazione non è più necessario riavviare, mentre gli aggiornamenti richiedono ancora un riavvio.

Home Assistant migra le installazioni HACS esistenti al primo avvio dopo l’aggiornamento. I progetti installati restano installati e continuano a ricevere aggiornamenti; vengono trasferiti anche Custom Repositories ed entità di aggiornamento. Secondo il produttore, non è necessario alcun intervento. L’installer HACS termina con Home Assistant 2026.11, su questa versione non è più possibile un’installazione HACS separata e il precedente HACS non riceverà nuove release. In futuro, segnalate gli errori nell’issue tracker di Home Assistant.

Al primo accesso, il Marketplace mostra un avviso secondo cui i progetti della community non sono testati, verificati né supportati dal progetto Home Assistant. Dovete confermare questo avviso prima di installare o aggiornare. I progetti esterni al catalogo possono essere aggiunti tramite “Add from link”; a questo scopo è necessario un collegamento a GitHub mediante Device Sign-in, che legge solo informazioni pubbliche.

Per Midea PortaSplit, questo riguarda la Custom Integration `Midea AC LAN` e la scheda `apexcharts-card`, se entrambe sono installate tramite HACS. In base alle informazioni sulla migrazione, rimangono installate e continuano a ricevere aggiornamenti. Il rischio della supply chain delle Custom Integrations, descritto nell’articolo sulla [protezione di PortaSplit](/blog/midea-portasplit-home-assistant-absichern), rimane anche con il Marketplace, come mostra l’avviso sui rischi del produttore. Dopo l’aggiornamento, controllate in Impostazioni > Marketplace che siano presenti tutti i progetti e i Custom Repositories precedentemente installati tramite HACS.

## Nuovi nomi delle entità

A partire dalla versione 2026.11, i nomi delle entità sono composti dal nome dell’entità e dal suo contesto, di norma il nome del dispositivo. Anche i nomi assegnati manualmente vengono migrati secondo le stesse regole; il vecchio nome viene mantenuto come alias. Le schede, i badge e le righe senza un nome proprio ora mostrano il nome del dispositivo seguito dal nome dell’entità invece del Friendly Name. Secondo le note di rilascio, Entity ID, automazioni e script non sono interessati.

Per i template, il produttore raccomanda le funzioni `entity_name()`, `device_name()` e `area_name()`; l’attributo `friendly_name` sarà in seguito contrassegnato come deprecato. Dopo l’aggiornamento, controllate le vostre dashboard per individuare etichette modificate e adattate i template che valutano `friendly_name`.

## FRITZ!Box Tools e Device Tracker

I tracker degli scanner di rete, tra cui sono esplicitamente menzionati FRITZ!Box Tools, UniFi Network, NETGEAR e Nmap Tracker, ora dispongono sempre di un dispositivo. Le entità tracker rinominate ricevono il nome del dispositivo all’inizio del proprio nome. I nuovi tracker di queste integrazioni vengono sempre creati disattivati; finora erano attivi se un’altra integrazione conosceva già l’indirizzo MAC del dispositivo. I tracker esistenti mantengono il loro stato, attivato o disattivato.

Per le automazioni di presenza tramite FRITZ!Box, come descritto nell’[articolo su digitalSTROM](/blog/digitalstrom-home-assistant) per l’uscita automatica, ciò significa che i tracker esistenti continuano a funzionare. Il tracker di uno smartphone appena aggiunto, invece, deve essere prima attivato affinché fornisca uno stato nelle automazioni.

Il changelog elenca inoltre diverse correzioni per FRITZ!Box Tools, tra cui la creazione degli interruttori WLAN, l’esclusione dagli Device Tracker degli host FRITZ!Box privi di indirizzo IP, un ciclo di ricaricamento per dispositivi senza dati sul traffico e un messaggio di errore più comprensibile in caso di credenziali non valide nella finestra di configurazione. In caso di autorizzazioni mancanti, l’integrazione FRITZ!Box Call Monitor ora segnala un errore di configurazione.

## Midea e digitalSTROM

I miglioramenti a Midea nel blog della release riguardano l’integrazione ufficiale `Midea`, che fa parte di Core da Home Assistant 2026.8. La Custom Integration `Midea AC LAN` (dominio `midea_ac_lan`) è separata da essa. Sono nuovi i controlli e un sensore di avanzamento per lavatrici e asciugatrici, i sensori porta per frigoriferi, le entità per scaldacqua, un interruttore di accensione/spegnimento per purificatori d’aria e un interruttore per il segnale acustico dei deumidificatori. Per i climatizzatori, il changelog menziona un pulsante `reset_filter`, con cui il filtro può essere contrassegnato come pulito. L’integrazione ha raggiunto il livello di qualità Platinum. Secondo la pagina dell’integrazione, il controllo e le interrogazioni di stato funzionano localmente; il Cloud è necessario solo per token e chiave dei dispositivi V3.

Il changelog della versione 2026.11.0 non contiene alcuna voce relativa a digitalSTROM. L’integrazione descritta nell’[articolo su digitalSTROM](/blog/digitalstrom-home-assistant) è una Custom Integration; se è inclusa tramite HACS come Custom Repository, si applica la migrazione nel Marketplace descritta sopra.

## Altre modifiche non compatibili con le versioni precedenti

L’elenco completo è disponibile nel blog della release. Per il funzionamento sono rilevanti soprattutto questi punti:

- **Auth:** In caso di accesso tramite un indirizzo IP non locale, schema, host e porta nel browser devono corrispondere all’indirizzo visto da Home Assistant, altrimenti l’autenticazione fallisce con “Invalid client id”. Il controllo precedente accettava tali indirizzi a causa di un errore, non appena contenevano una porta. In questo caso, inserite il vostro Reverse Proxy tra i Trusted Proxies dell’integrazione HTTP oppure utilizzate un nome di dominio.
- **Dipendenze Python:** Home Assistant non installa più pacchetti nella cartella `deps` della directory di configurazione. Sono interessate solo le installazioni Core al di fuori di un ambiente virtuale; non le installazioni OS, Supervised e Container. Dopo l’aggiornamento, la cartella `deps` può essere eliminata.
- **SMTP:** Un allegato non leggibile ora causa un errore; in precedenza il messaggio veniva inviato senza allegato.
- **Telegram:** La vecchia piattaforma `notify` viene rimossa. Utilizzate le entità Notify del bot Telegram e l’azione `notify.send_message`.
- **Z-Wave:** Sono richiesti zwave-js-server 3.11.0 o successivo (schema 51), l’app Z-Wave JS dalla versione 1.9.0 oppure Z-Wave JS UI dalla versione v11.24.2.
- **Uptime Kuma:** Le versioni precedenti alla 2.0.0 non sono più supportate.
- **Philips Hue:** L’effetto luce `"None"` viene rimosso; al suo posto si applica `"off"`.
- **Matter ed Electra Smart:** `climate.turn_on` riprende l’ultima modalità attiva. Per una modalità specifica, utilizzate `climate.set_hvac_mode`.
- **Intervallo di polling:** Per Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato e Risco, un intervallo personalizzato viene ignorato e viene utilizzato quello predefinito. Control4 esegue sempre il polling ogni 5 secondi; per un intervallo diverso, disattivate il polling e richiamate `homeassistant.update_entity` da un’automazione.

L’integrazione Flipr è stata rimossa perché l’azienda che la gestiva è stata liquidata.

## Nuove funzioni

Oltre al Marketplace, il blog della release cita tra l’altro l’esclusione di singole entità, dispositivi, aree, piani o etichette da un target, il copia e incolla delle sezioni della dashboard, una scheda data e una scheda avviso con editor visivo. La dashboard Energia può derivare un sensore di energia da un sensore di potenza. Per la risoluzione dei problemi sono disponibili un filtro “Triggered by” nelle attività, log live per integrazione e l’eliminazione delle statistiche in un intervallo temporale. Backup e file multimediali possono essere salvati su supporti dati locali. Sono ora supportate le telecamere Matter 1.5 con WebRTC, nonché LoRaWAN (ChirpStack, The Things Network, SenseCAP) e Zigbee Green Power.

Nella scheda avviso, l’opzione `color` ora si chiama `alert_color`; dovete adattare solo le dashboard create personalmente in YAML. La nuova integrazione Debug console esegue codice Python all’interno di Home Assistant, ha accesso completo e deve quindi essere attivata in `configuration.yaml`; può essere usata solo dagli amministratori.

## Procedura consigliata

La versione di destinazione è la 2026.11.0. Prima dell’aggiornamento, leggete le modifiche non compatibili con le versioni precedenti nel blog della release per le integrazioni utilizzate. Dopo l’aggiornamento, verificate nel Marketplace che tutti i precedenti progetti HACS siano stati trasferiti, controllate dashboard e template per i nuovi nomi delle entità e, se necessario, attivate i Device Tracker appena creati. In caso di accesso tramite un Reverse Proxy, testate l’autenticazione. Al 5 novembre 2026, gli avvisi di sicurezza nel repository GitHub di Home Assistant Core non contengono alcuna voce associata alla versione 2026.11.0; il blog della release non contiene indicazioni sui Known Issues.

## Fonti

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Blog della release con Marketplace, nomi delle entità, nuove integrazioni e modifiche non compatibili con le versioni precedenti.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Changelog completo, comprese le voci su FRITZ!Box Tools, FRITZ!Box Call Monitor e Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Tag della release con pubblicazione il 4 novembre 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Documentazione dell’integrazione ufficiale Midea, introdotta nella versione 2026.8, con controllo locale.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Avvisi di sicurezza pubblicati, al 5 novembre 2026 senza voce relativa alla versione 2026.11.0.

6.  [Midea PortaSplit in Home Assistant: configurazione e dashboard](/blog/midea-portasplit-home-assistant): Installazione di Midea AC LAN e apexcharts-card tramite HACS.

7.  [Proteggere Midea PortaSplit in Home Assistant](/blog/midea-portasplit-home-assistant-absichern): Rischio della supply chain delle Custom Integrations.

8.  [Collegare digitalSTROM a Home Assistant](/blog/digitalstrom-home-assistant): Integrazione locale digitalSTROM e presenza tramite FRITZ!Box.
