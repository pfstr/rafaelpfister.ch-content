---
title: "Midea PortaSplit in Home Assistant: perché token e chiave sono fondamentali"
navTitle: "PortaSplit e token"
description: "Il controllo locale richiede due valori dal cloud Midea. Ecco come ottenere token e chiave, perché la loro perdita è problematica e come i proprietari possono proteggere la configurazione esistente."
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
url: "https://rafaelpfister.ch/it/blog/midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: a02265cf4b8fde907361c3551326fd3283c83d660cf9fdfb40451a9e78ca690b
translatedAt: 2026-07-29T12:29:38.946Z
---

<aside class="article-update">
  <p class="article-update__label">Cosa dovrebbero fare ora i proprietari di PortaSplit</p>
  <p>Attraverso interfacce cloud private, Home Assistant recupera il token e la chiave specifici del dispositivo durante la configurazione. Il progetto Midea AC LAN avverte dal 19 maggio 2025 di possibili modifiche. Tuttavia, non è documentata una data concreta di disattivazione da parte del produttore. Per i proprietari questo significa:</p>
  <ol>
    <li><strong>Non rimuovere inutilmente la configurazione esistente.</strong> Solo il recupero delle credenziali richiede il cloud Midea. Future modifiche all'endpoint privato potrebbero rendere più difficile una nuova configurazione.</li>
    <li><strong>Salvare token, chiave e configurazione in forma crittografata.</strong> Se in seguito il recupero non dovesse più funzionare, il backup rimane il modo più affidabile per ripristinare tutto.</li>
    <li><strong>Non annullare l'associazione senza necessità.</strong> Il ripristino delle impostazioni di fabbrica, la rimozione dall'account Midea o la sostituzione del modulo Wi-Fi impongono un nuovo recupero del token, che in futuro potrebbe fallire.</li>
  </ol>
  <p>I dispositivi già configurati vengono controllati localmente. Le modifiche all'interfaccia cloud riguardano quindi innanzitutto l'aggiunta e il ripristino, non ogni comando di controllo in esecuzione. I passaggi concreti sono descritti nel <a href="/blog/midea-portasplit-home-assistant-einrichten">contributo pratico su integrazione e protezione</a>.</p>
</aside>

![Dashboard esemplificativa di Home Assistant per una Midea PortaSplit con temperatura ambiente e impostata, umidità dell'aria, assorbimento di potenza, consumo energetico e tempi di funzionamento del compressore nelle ultime 24 ore.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Il controllo locale della Midea PortaSplit si basa su due valori specifici del dispositivo: token e chiave. L'integrazione Home Assistant recupera entrambi durante la configurazione tramite un endpoint cloud privato di Midea. Successivamente, invia i comandi di controllo direttamente sulla rete locale.

Il progetto Midea AC LAN avverte di possibili modifiche a queste interfacce cloud. Analisi più recenti mostrano tuttavia che da ciò non si può dedurre una roadmap confermata del produttore né una data concreta di disattivazione. Questo articolo spiega il rapporto di dipendenza tecnica; l'[analisi dettagliata dell'API](/blog/midea-v2-cloud-api-portasplit-home-assistant) inquadra le diverse denominazioni «V2» e lo stato attuale.

## La questione del token nel dettaglio

### Perché Home Assistant è riuscito finora a ottenere il token?

L'aspetto interessante è che la community non ha mai calcolato il token. Ha invece analizzato il traffico di rete dell'app ufficiale e ha constatato che l'app non genera autonomamente il token, bensì lo recupera dal cloud:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

L'integrazione Home Assistant ha reimplementato esattamente questa chiamata cloud. Accede al cloud usando gli stessi endpoint e lo stesso flusso dell'app, ottenendo così lo stesso token e la stessa chiave. Il fondamento effettivo non è quindi un calcolo ingegnoso, ma un recupero ricostruito. Se l'endpoint viene meno, viene meno anche il recupero.

### Si potrebbe estrarre il token dall'app ufficiale?

In teoria sì. L'app deve conoscere il token a un certo punto, altrimenti non potrebbe comunicare localmente con il dispositivo. Le strade ipotizzabili in linea di principio sarebbero:

- reverse engineering dell'app,
- intercettazione del traffico di rete, se non è ulteriormente protetto,
- strumentazione dell'app in fase di esecuzione, ad esempio con Frida o Objection,
- hooking delle funzioni che elaborano il token.

Proprio a questo mira l'affermazione dello sviluppatore di Midea AC LAN secondo cui il design finora adottato rappresenta un problema di sicurezza dal punto di vista di Midea: un segreto di lunga durata, estraibile con uno sforzo ragionevole da un'app ampiamente distribuita, è difficile da controllare. Per il singolo utente, tuttavia, queste strade sono complesse e non sostituiscono il comodo recupero dal cloud.

### Si potrebbe ottenere il token direttamente dal dispositivo?

Sarebbe la soluzione più elegante. Se il dispositivo scambiasse una chiave pubblica al primo abbinamento locale o utilizzasse via Bluetooth un codice di abbinamento monouso, il cloud non sarebbe affatto necessario. Molti dispositivi IoT moderni funzionano proprio così.

Midea ha però progettato diversamente il protocollo LAN originale: il dispositivo accetta comandi locali solo con le credenziali corrette correlate al cloud. Non esiste un meccanismo di abbinamento locale documentato che fornisca il token senza passare dal cloud. Il cloud non è quindi solo una comodità, ma dal punto di vista architetturale l'unica via prevista per ottenere il token.

### La community potrebbe aggirare modifiche all'endpoint del token?

Sarebbe possibile solo se si trovasse una delle seguenti opzioni:

- una nuova API cloud che continui a fornire token,
- un metodo di abbinamento locale finora sconosciuto,
- una vulnerabilità nel dispositivo,
- oppure Midea pubblica un giorno un'API locale ufficiale.

Al contrario, è molto improbabile che sia possibile semplicemente «ricalcolare» il token. Se lo fosse, la community probabilmente lo avrebbe implementato già da tempo e non sarebbe mai dipesa dall'API cloud. Il fatto stesso che sia stato realizzato il passaggio attraverso il cloud è l'indizio più forte che non esista una via locale più semplice.

## L'avvertimento di Midea AC LAN

Il repository di `Midea AC LAN` contiene un avviso ben visibile intitolato “Important Notice”. Secondo lo sviluppatore, Midea ha già chiuso le API di token lato server nei cloud Meiju e SmartHome. L'integrazione accede quindi attualmente alle interfacce token del cloud NetHome Plus, e anche queste dovrebbero essere chiuse gradualmente. La conseguenza sarebbe che i dispositivi già configurati continuerebbero a funzionare localmente, ma non sarebbe più possibile aggiungerne di nuovi. Lo sviluppatore si spinge oltre e scrive che Midea intende passare a lungo termine a una nuova API Cloud Control, rendendo così inutilizzabile la precedente API LAN V1.

L'avvertimento ha una breve storia. La ben visibile “Important Notice” è stata aggiunta al README il 19 maggio 2025 (Pull Request #578) e allora indicava il cloud SmartHome come soluzione alternativa per aggiungere nuovi dispositivi. Il 14 luglio 2025 (#639) è stata aggiornata; da allora rimanda al cloud NetHome Plus, perché Midea aveva chiuso altri endpoint. Il nucleo è rimasto invariato in entrambe le versioni: le interfacce token scompaiono gradualmente, cambia solo il cloud ancora utilizzabile di volta in volta.

Va considerato in modo differenziato. Si tratta della valutazione di un progetto open source, non di una roadmap vincolante di Midea, e la tempistica è sconosciuta. Un futuro aggiornamento del firmware può modificare le funzioni locali; un token già salvato può continuare a funzionare, ma non necessariamente per sempre. Un ripristino delle impostazioni di fabbrica, la sostituzione del modulo Wi-Fi o un nuovo dispositivo possono rendere necessario un nuovo recupero del token.

Da ciò derivano i tre passaggi del riquadro all'inizio dell'articolo, ciascuno con la relativa motivazione:

- **Non sostituire senza motivo una configurazione funzionante.** Il recupero del token è l'unico passaggio che avviene obbligatoriamente tramite il cloud Midea. Le modifiche all'endpoint privato possono colpire soprattutto una nuova configurazione successiva.
- **Proteggere le credenziali.** Home Assistant salva localmente token e chiave. Un sistema difettoso, un ripristino non riuscito o un'integrazione eliminata accidentalmente possono comunque rendere inutilizzabile il controllo locale se non esiste un backup esterno.
- **Non annullare l'associazione con leggerezza.** Non è completamente documentato se un ripristino di fabbrica o la rimozione dall'account Midea impongano nuove credenziali per ogni modello. Un backup prima di tali modifiche è quindi indispensabile.

Il funzionamento corrente inizialmente non è interessato: il controllo locale utilizza i valori già salvati e non necessita più dell'endpoint del token. Rimane un rischio residuo nel caso in cui un firmware successivo modifichi il protocollo locale o l'autenticazione. Il modo in cui salvare token, chiave e configurazione è descritto nel [contributo pratico sulla configurazione](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Cosa significa per la sicurezza

Oltre alla disponibilità, l'avvertimento ha un nucleo legato alla sicurezza. Secondo `Midea AC LAN`, la precedente architettura LAN si basa su un presupposto problematico: la comunicazione client era originariamente considerata sufficientemente protetta, perciò i token emessi dal cloud non ricevevano una data di scadenza.

Un token che non scade non è di per sé una vulnerabilità. Diventa problematico quando finisce in protocolli o backup non protetti, viene acquisito da terzi oppure non può essere né revocato né ruotato. Lo sviluppatore di `Midea AC LAN` ipotizza che Midea stia reagendo a questi rischi con modifiche ai servizi token e un'architettura più basata sul cloud. Tuttavia, non è documentato un annuncio del produttore corrispondente con una tempistica.

In questo caso la precisione linguistica è importante. L'integrazione della community non «hackerizza» il climatizzatore. Implementa un protocollo proprietario ricostruito tramite reverse engineering. Il problema di sicurezza deriva dal fatto che segreti di lunga durata possono essere utilizzati e salvati al di fuori dell'app originariamente prevista.

Per l'utilizzo nella propria rete è soprattutto rilevante ciò che token e chiave consentono. Entrambi autenticano la comunicazione locale con il dispositivo. Se finiscono nelle mani sbagliate, un aggressore potrebbe, a seconda del protocollo e della sua posizione nella rete, individuare il dispositivo, autenticarsi presso di esso, leggere informazioni di stato, modificare impostazioni, accendere o spegnere il climatizzatore, cambiare modalità operative e modificare la temperatura impostata. A tal fine, l'aggressore deve comunque di norma poter stabilire una connessione di rete con il dispositivo; il possesso del solo token e della chiave non consente un attacco dall'intera Internet. Token e chiave vanno quindi trattati come una password. Come integrare il dispositivo nella rete affinché questi valori causino danni limitati anche in caso di incidente è il tema della [seconda parte](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Cosa resta nella pratica

Il controllo locale della PortaSplit dipende interamente da token e chiave, che al momento possono essere ottenuti solo tramite il cloud Midea. Questo passaggio fa parte del design del protocollo: i comandi locali sono legati a credenziali correlate al cloud. Poiché l'endpoint è privato e non documentato, la disponibilità a lungo termine dell'integrazione non ufficiale rimane incerta.

In pratica, ciò significa: salvare credenziali e configurazione, non annullare inutilmente un'associazione funzionante e monitorare le modifiche all'integrazione e al firmware. I dispositivi già configurati continuano a funzionare localmente. Configurazione, backup e protezione della rete sono descritti nel [contributo pratico sulla PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Fonti

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN` con la “Important Notice” (dal 19 maggio 2025, aggiornata il 14 luglio 2025), la motivazione relativa ai token senza scadenza e alla crittografia client ricostruita, nonché la descrizione del recupero dei token basato sul cloud.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: descrizione del recupero basato sul cloud di token e chiave nei dispositivi V3 e del salvataggio locale dei valori.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): informazioni del produttore sull'ecosistema SmartHome e sugli standard di sicurezza e protezione dei dati citati.
