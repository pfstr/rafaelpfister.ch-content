---
title: "Midea PortaSplit in Home Assistant: perché token e key sono decisivi"
navTitle: "PortaSplit e token"
description: "Il controllo locale richiede due valori dal cloud Midea. Ecco come ottenere token e key, perché perderli è problematico e come i proprietari possono proteggere la configurazione esistente."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "9 min di lettura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-einrichten
  - serverloser-newsletter-cloudflare-workers-d1
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali"
translationOf: "midea-portasplit-home-assistant"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: 93933b82cdbb4151fe6dc6ac73a356fc752f120f41461268af1c8e484b62652c
translatedAt: 2026-09-04T08:28:26.698Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/it/blog/midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali
---

<aside class="article-update">
  <p class="article-update__label">Cosa dovrebbero fare ora i proprietari di PortaSplit</p>
  <p>Durante la configurazione, Home Assistant ottiene il token e la key specifici del dispositivo tramite interfacce cloud private di Midea. Il progetto Midea AC LAN avverte dal 19 maggio 2025 di possibili modifiche. Tuttavia, non è documentata una data concreta di disattivazione da parte del produttore. Per i proprietari ciò significa:</p>
  <ol>
    <li><strong>Non rimuovere inutilmente la configurazione esistente.</strong> Solo il reperimento delle credenziali richiede il cloud Midea. Future modifiche all'endpoint privato potrebbero rendere più difficile una nuova configurazione.</li>
    <li><strong>Salvare in modo cifrato token, key e configurazione.</strong> Se in seguito il recupero non dovesse più funzionare, il backup rimane il modo più affidabile per il ripristino.</li>
    <li><strong>Non annullare l'abbinamento senza necessità.</strong> Il ripristino delle impostazioni di fabbrica, la rimozione dall'account Midea o la sostituzione del modulo Wi-Fi impongono un nuovo reperimento del token, che in futuro potrebbe fallire.</li>
  </ol>
  <p>I dispositivi già configurati vengono controllati localmente. Le modifiche all'interfaccia cloud riguardano quindi anzitutto l'aggiunta e il ripristino, non ogni comando di controllo in esecuzione. I passaggi concreti sono descritti nel <a href="/blog/midea-portasplit-home-assistant-einrichten">contributo pratico su integrazione e protezione</a>.</p>
</aside>

![Esempio di dashboard Home Assistant di una Midea PortaSplit con temperatura ambiente e impostata, umidità dell'aria, assorbimento di potenza, consumo energetico e tempi di funzionamento del compressore nelle ultime 24 ore.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Il controllo locale della Midea PortaSplit si basa su due valori specifici del dispositivo: token e key. Durante la configurazione, l'integrazione Home Assistant recupera entrambi tramite un endpoint cloud privato di Midea. Successivamente invia i comandi di controllo direttamente nella rete locale.

Il progetto Midea AC LAN avverte di possibili modifiche a queste interfacce cloud. Analisi più recenti mostrano tuttavia che da ciò non si può dedurre né una roadmap confermata del produttore né una data concreta di disattivazione. Questo articolo spiega il rapporto tecnico di dipendenza; l'[analisi dettagliata dell'API](/blog/midea-v2-cloud-api-portasplit-home-assistant) inquadra le diverse denominazioni «V2» e lo stato attuale.

## La questione del token nel dettaglio

### Perché Home Assistant ha potuto ottenere il token finora?

La community non ha mai calcolato il token. Ha piuttosto analizzato il traffico di rete dell'app ufficiale, scoprendo che l'app non genera autonomamente il token, bensì lo ottiene dal cloud:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

L'integrazione Home Assistant ha reimplementato proprio questa chiamata cloud. Accede al cloud con gli stessi endpoint e lo stesso processo dell'app, ottenendo così lo stesso token e la stessa key. Il vero fondamento è dunque un recupero ricostruito, non un calcolo. Se l'endpoint viene meno, viene meno anche il reperimento.

### Si potrebbe estrarre il token dall'app ufficiale?

In teoria sì. L'app deve prima o poi conoscere il token, altrimenti non potrebbe comunicare localmente con il dispositivo. Le vie possibili in linea di principio sarebbero:

- reverse engineering dell'app,
- intercettazione del traffico di rete, se questo non è ulteriormente protetto,
- strumentazione dell'app in fase di esecuzione, ad esempio con Frida o Objection,
- hooking delle funzioni che elaborano il token.

È proprio a questo che si riferisce lo sviluppatore di Midea AC LAN quando afferma che l'attuale design costituisce un problema di sicurezza dal punto di vista di Midea: un segreto a lunga durata, estraibile con uno sforzo ragionevole da un'app distribuita su larga scala, è difficile da controllare. Per il singolo utente, tuttavia, queste vie sono complesse e non sostituiscono il comodo recupero dal cloud.

### Si potrebbe ottenere il token direttamente dal dispositivo?

Sarebbe la soluzione più elegante. Se il dispositivo scambiasse una chiave pubblica durante il primo abbinamento locale o utilizzasse un codice di abbinamento monouso via Bluetooth, il cloud non sarebbe affatto necessario. Molti dispositivi IoT moderni fanno esattamente questo.

Midea ha però progettato diversamente il protocollo LAN originale: il dispositivo accetta comandi locali solo con le credenziali appropriate, legate al cloud. Non esiste un meccanismo di abbinamento locale documentato che fornisca il token senza passare dal cloud. Il cloud non è quindi solo una comodità, ma l'unica via prevista architetturalmente per ottenere il token.

### La community potrebbe aggirare modifiche all'endpoint del token?

Sarebbe possibile solo trovando una delle seguenti opzioni:

- una nuova API cloud che continui a fornire token,
- un metodo di abbinamento locale finora sconosciuto,
- una vulnerabilità nel dispositivo,
- oppure Midea stessa pubblica un giorno un'API locale ufficiale.

Al contrario, semplicemente «ricalcolare» il token molto probabilmente non funzionerà. Se fosse possibile, la community lo avrebbe presumibilmente già implementato da tempo e non sarebbe mai dipesa dall'API cloud. Il fatto stesso che sia stato realizzato il passaggio tramite cloud è l'indizio più forte che non esista una via locale più semplice.

## L'avvertimento di Midea AC LAN

Il repository di `Midea AC LAN` contiene un avviso «Important Notice» collocato in evidenza. Secondo lo sviluppatore, Midea ha già chiuso le API dei token lato server nei cloud Meiju e SmartHome. L'integrazione accede quindi attualmente alle interfacce token del cloud NetHome Plus, e anche queste dovrebbero essere chiuse gradualmente. La conseguenza sarebbe che i dispositivi già configurati continuerebbero a funzionare localmente, ma non sarebbe più possibile aggiungerne di nuovi. Lo sviluppatore si spinge oltre e scrive che Midea intende passare a lungo termine a una nuova Cloud-Control-API, rendendo così inutilizzabile l'attuale V1-LAN-API.

L'avvertimento ha una breve storia. La «Important Notice» in evidenza è stata inserita nel README il 19 maggio 2025 (Pull Request #578) e allora indicava il cloud SmartHome come soluzione di riserva per l'aggiunta di nuovi dispositivi. Il 14 luglio 2025 (#639) è stata aggiornata; da allora rimanda al cloud NetHome Plus, perché Midea aveva chiuso altri endpoint. Il nucleo è rimasto invariato in entrambe le versioni: le interfacce token scompaiono poco a poco, cambia solo il cloud ancora utilizzabile di volta in volta.

Questo va considerato con le dovute distinzioni. Si tratta della valutazione di un progetto open source, non di una roadmap vincolante di Midea, e la tempistica è sconosciuta. Un futuro aggiornamento del firmware può modificare le funzioni locali; un token già salvato può continuare a funzionare, ma non necessariamente per sempre. Un ripristino delle impostazioni di fabbrica, una sostituzione del modulo Wi-Fi o un nuovo dispositivo possono richiedere un nuovo reperimento del token.

Da ciò derivano i tre passaggi riportati nel riquadro all'inizio dell'articolo, ciascuno con la propria motivazione:

- **Non sostituire senza motivo una configurazione funzionante.** Il reperimento del token è l'unico passaggio che avviene necessariamente tramite il cloud Midea. Le modifiche all'endpoint privato possono colpire soprattutto una successiva nuova configurazione.
- **Proteggere le credenziali.** Home Assistant salva token e key localmente. Un sistema guasto, un ripristino non riuscito o un'integrazione eliminata accidentalmente possono comunque rendere inutilizzabile il controllo locale se non è disponibile un backup esterno.
- **Non annullare l'abbinamento con leggerezza.** Non è completamente documentato se un ripristino di fabbrica o la rimozione dall'account Midea impongano nuove credenziali per ogni modello. Un backup prima di tali modifiche è pertanto indispensabile.

Il funzionamento in corso non ne è inizialmente interessato: il controllo locale utilizza i valori già salvati e non ha più bisogno dell'endpoint del token. Rimane un rischio residuo nel caso in cui un firmware successivo modifichi il protocollo locale o l'autenticazione. Il [contributo pratico sulla configurazione](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration) spiega come proteggere token, key e configurazione.

## Cosa significa per la sicurezza

Oltre alla disponibilità, l'avvertimento ha un nucleo legato alla sicurezza. Secondo `Midea AC LAN`, la precedente architettura LAN si basa su un presupposto problematico: in origine la comunicazione del client era considerata sufficientemente protetta, per cui i token emessi dal cloud non avevano una scadenza.

Un token senza scadenza non è di per sé una vulnerabilità. Diventa problematico se finisce in log o backup non protetti, arriva a terzi oppure non può essere né revocato né ruotato. Lo sviluppatore di `Midea AC LAN` ipotizza che Midea stia reagendo a questi rischi con modifiche ai servizi token e un'architettura maggiormente basata sul cloud. Tuttavia, non è documentato alcun annuncio del produttore con una tempistica.

La precisione linguistica è importante. L'integrazione della community non «hackera» il climatizzatore. Implementa un protocollo proprietario ricostruito tramite reverse engineering. Il problema di sicurezza deriva dal fatto che segreti a lunga durata possono essere utilizzati e salvati al di fuori dell'app originariamente prevista.

Per l'uso nella propria rete, ciò che conta soprattutto è cosa consentono token e key. Entrambi autenticano la comunicazione locale con il dispositivo. Se finiscono nelle mani sbagliate, un aggressore potrebbe, a seconda del protocollo e della sua posizione nella rete, rilevare il dispositivo, autenticarsi presso di esso, leggere informazioni di stato, modificare impostazioni, accendere o spegnere il climatizzatore, cambiare modalità operative e modificare la temperatura impostata. Di norma, l'aggressore deve comunque poter stabilire una connessione di rete con il dispositivo; il solo possesso di token e key non consente un attacco da tutta Internet. Token e key devono quindi essere trattati come una password. Il [secondo capitolo](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben) tratta di come integrare il dispositivo nella rete in modo che questi valori causino pochi danni anche in caso di incidente.

## Cosa resta in pratica

Il controllo locale della PortaSplit dipende interamente da token e key, che al momento possono essere ottenuti solo tramite il cloud Midea. Questo passaggio fa parte del design del protocollo: i comandi locali sono vincolati a credenziali legate al cloud. Poiché l'endpoint è privato e non documentato, la disponibilità a lungo termine dell'integrazione non ufficiale resta incerta.

In pratica ciò significa: proteggere credenziali e configurazione, non annullare inutilmente un abbinamento funzionante e monitorare le modifiche all'integrazione e al firmware. I dispositivi già configurati continuano a funzionare localmente. Il [contributo pratico sulla PortaSplit](/blog/midea-portasplit-home-assistant-einrichten) descrive configurazione, backup e protezione della rete.

## Fonti

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN` con la «Important Notice» (dal 19 maggio 2025, aggiornata il 14 luglio 2025), la motivazione relativa ai token senza scadenza e alla crittografia del client ricostruita, nonché la descrizione del recupero del token basato sul cloud.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: descrizione del recupero di token e key basato sul cloud nei dispositivi V3 e del salvataggio locale dei valori.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): informazioni del produttore sull'ecosistema SmartHome e sugli standard di sicurezza e protezione dei dati citati.
