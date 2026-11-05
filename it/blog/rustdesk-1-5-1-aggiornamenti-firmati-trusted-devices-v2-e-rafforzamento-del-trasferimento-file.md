---
title: "RustDesk 1.5.1: aggiornamenti firmati, Trusted Devices V2 e rafforzamento del trasferimento file"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 verifica gli aggiornamenti desktop tramite metadati firmati, introduce Trusted Devices V2 e rafforza il trasferimento file, l'installer Windows e l'accesso automatico. Si aggiungono circa 35 correzioni, soprattutto per l'acquisizione DRM e Wayland. Si consiglia di aggiornare i client; il server rimane invariato."
date: "2026-11-02"
kategorie: "Assistenza remota e supporto"
timeToRead: "5 min di lettura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-aggiornamenti-firmati-trusted-devices-v2-e-rafforzamento-del-trasferimento-file"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/it/blog/rustdesk-1-5-1-aggiornamenti-firmati-trusted-devices-v2-e-rafforzamento-del-trasferimento-file
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:35:10.540Z
translationReview: automatic
---

# RustDesk 1.5.1: aggiornamenti firmati, Trusted Devices V2 e rafforzamento del trasferimento file

RustDesk ha pubblicato la versione 1.5.1 del client il 1° novembre 2026, poco più di un mese dopo la 1.5.0 (30 settembre 2026). La release introduce cinque nuove funzionalità, tra cui la verifica dei metadati di aggiornamento firmati e una nuova versione dei Trusted Devices per l'autenticazione a due fattori, tre modifiche e circa 35 correzioni. Diverse di queste rafforzano processi rilevanti per la sicurezza: convalida dei percorsi di download, trasferimento protetto dell'installer su Windows, avviso per server API non crittografati e un accesso automatico più restrittivo. Il produttore non menziona né una CVE né un Security Advisory (situazione al 2 novembre 2026). La release è destinata a chiunque utilizzi client RustDesk su Windows, macOS, Linux, Android o iOS, in particolare con un proprio server. La configurazione e l'utilizzo di un server RustDesk proprio sono descritti nell'articolo [RustDesk: configurare l'alternativa open source a TeamViewer](/blog/rustdesk-teamviewer-alternative).

## Le versioni in sintesi

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Release client | metadati di aggiornamento firmati, Trusted Devices V2, scorciatoie da tastiera, convalida dei percorsi di download, rafforzamento dell'installer su Windows, avviso per server API HTTP, correzioni per DRM, Wayland, X11 e QSV |

Il server open source (`rustdesk-server`) non è interessato da questa release. La sua ultima versione è la 1.1.16 del 20 luglio 2026.

## Metadati di aggiornamento firmati

Con la PR #15914, l'updater desktop su Windows e macOS ottiene le proprie informazioni da metadati di release firmati. Ogni release include gli asset `rustdesk-update.json` (versione, URL, nome file, dimensione e SHA-256 per artefatto) e `rustdesk-update.json.sig` (firma Ed25519). Il client verifica la firma prima di elaborare i metadati e accetta soltanto URL di download nella release di `rustdesk/rustdesk`. Se la verifica fallisce, l'aggiornamento viene interrotto; non è previsto alcun fallback a dati non firmati. Gli installer scaricati vengono conservati in una directory protetta fino all'avvio e sottoposti nuovamente ad hash prima dell'esecuzione con privilegi elevati.

Secondo la PR, la firma attesta autenticità e integrità, ma non l'attualità: non include una protezione contro la reinstallazione di release precedenti correttamente firmate (rollback). La variante portabile su Windows continua a estrarre e avviarsi da una directory scrivibile dall'utente; secondo la PR, il relativo rafforzamento è stato rinviato a un branch separato.

## Trusted Devices V2

La PR #16548 sostituisce la precedente relazione di attendibilità basata sull'ID hardware con token emessi dall'host. Per una nuova registrazione sono ancora necessari la password permanente e TOTP; in seguito, un token Trusted Device evita solo l'inserimento del TOTP. L'host memorizza soltanto l'hash SHA-256 del token in forma crittografata, mentre il controller associa il proprio token alla chiave pubblica verificata dell'host. Per le connessioni IP dirette non viene offerta alcuna relazione di attendibilità, poiché in quel caso l'identità dell'host non viene verificata.

Per l'esercizio sono rilevanti i seguenti punti della PR:

- **Validità:** configurabile su 1, 7, 30 o 90 giorni oppure illimitata; il valore predefinito è 90 giorni. Le relazioni di attendibilità possono essere rinnovate o rimosse singolarmente nelle impostazioni, oppure eliminate tutte insieme.
- **Revoca:** una modifica della password, della 2FA o dell'impostazione Trusted Device revoca i token interessati. L'host mantiene al massimo 128 voci.
- **Versioni miste:** le relazioni di attendibilità esistenti e non scadute restano valide dopo l'aggiornamento dell'host. Le nuove registrazioni richiedono la versione 1.5.1 su entrambi i lati; nelle sessioni con versioni miste RustDesk richiede il TOTP e non offre “Trust this device”.
- **Downgrade:** secondo la PR non è sicuro, poiché le versioni precedenti non possono leggere le relazioni di attendibilità importate.
- **Limiti noti:** una revoca viene prima salvata localmente e poi trasferita in modo asincrono nella configurazione root; un riavvio prima della sincronizzazione può ripristinare una relazione di attendibilità precedente. La data di scadenza dipende dall'orologio di sistema dell'host.

Pianificate quindi l'aggiornamento in modo che host e controller passino alla 1.5.1 in tempi ravvicinati ed evitate di tornare alla 1.5.0.

## Ulteriori correzioni rilevanti per la sicurezza

- **Percorsi di download (PR #16601):** i download rifiutano nomi file e percorsi non sicuri che escono dalla directory di destinazione selezionata o passano tramite link simbolici. Gli elenchi di file remoti vengono verificati prima del download; la sovrascrittura di file esistenti e la ripresa di download interrotti richiedono una conferma.
- **Installer Windows (PR #16554, #16562):** gli script per il trasferimento dell'installer con privilegi elevati non si trovano più in `System32`, bensì in una directory sotto Windows Temp con DACL limitata (Administrators e SYSTEM). Il servizio viene gestito tramite il controllo nativo dei servizi. Tra le ragioni vi è stato anche il blocco dello script batch temporaneo da parte di Bitdefender (Issue #16557).
- **Server API non crittografato (PR #16616):** le impostazioni del server mostrano un avviso se il server API è vuoto o utilizza un indirizzo HTTP remoto. HTTPS e gli indirizzi locali non attivano alcun avviso. La connessione non viene bloccata.
- **Accesso automatico (PR #16520):** l'accesso automatico inserisce la password del sistema operativo soltanto nella schermata di accesso o di blocco.
- **Privacy Mode (PR #16507):** una richiesta Privacy Mode rifiutata non disattiva più il Privacy Mode di un'altra connessione.

Se utilizzate un server proprio con un indirizzo `http://` come server API, il nuovo avviso apparirà agli utenti. Passate il server API a HTTPS prima di distribuire la 1.5.1, per evitare richieste di chiarimento.

## Nuove funzionalità e modifiche

Oltre ai punti già citati, la 1.5.1 introduce scorciatoie da tastiera (PR #14933), un pulsante nella rubrica per aprire la console web e, su Android, il trasferimento diretto di file nelle cartelle SAF. Sono stati modificati la codifica dei contenuti di immagini statiche dopo lo scorrimento (visualizzazione più nitida) e i budget di refresh per MediaCodec. L'elaborazione di `ConfigUpdate` è stata rimossa.

## Correzioni per piattaforma

- **Linux, acquisizione DRM:** dimensioni corrette con display ruotati, nessun risveglio dei display in standby all'avvio, eliminazione dei frame duplicati e frequenza dei fotogrammi adattata all'encoder.
- **Linux, Wayland:** acquisizione PipeWire senza tempi di attesa, corretta associazione dello stream del portale all'output, layout di tastiera durante l'immissione e una richiesta del portale aperta non blocca più il server.
- **Linux, generale:** utente attivo corretto con Flatpak, arresto anomalo alla chiusura della finestra GTK, richiesta sudo senza il falso “Sorry, try again.” e con supporto per sudo-rs, nomi di output xrandr, interrogazione dei monitor RandR sotto X11.
- **Windows:** lettura di `host=` da percorsi di programma con caratteri non ASCII, executable esplicitamente denominato per la finestra Privacy, blocchi dell'encoder QSV.
- **macOS:** link alle impostazioni di Screen Recording, rilascio del pulsante del mouse.
- **Mobile:** nessun Mouse-Down ritardato dopo il rilascio del pulsante fluttuante.
- **Rete:** errore API HTTP 407 tramite proxy HTTPS, server ID IPv6 senza parentesi durante la derivazione del server API, un messaggio di errore invece di due quando entrambi i tentativi di connessione WebRTC falliscono.
- **Ulteriori:** le stampanti eliminate rimangono conservate, replay buffer nel terminale, navigazione da tastiera nell'elenco file, caricamento dei peer senza directory dell'app impostata, clic sulle schede dei peer nel client web.

## Procedura consigliata

La versione di destinazione per tutti i client è la 1.5.1. Aggiornate host e controller possibilmente nello stesso periodo, affinché le nuove registrazioni Trusted Device funzionino; secondo la PR #16548, un downgrade alla 1.5.0 non è sicuro. Prima del rollout, verificate che il vostro server API sia raggiungibile tramite HTTPS. Il server open source non deve essere aggiornato per questa release. Le release notes non contengono indicazioni su Known Issues o termini di supporto (situazione al 2 novembre 2026).

## Fonti

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): note di rilascio con data, nuove funzionalità, modifiche e correzioni.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): versione precedente del 30 settembre 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): processo di verifica della firma, protezione rollback assente, variante portabile.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): modello basato su token, compatibilità con versioni precedenti, downgrade e limiti noti.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): verifica di nomi file, percorsi e link simbolici durante i download.

6.  [GitHub: PR #16554, script dell'installer in directory Temp protette](https://github.com/rustdesk/rustdesk/pull/16554): spostamento da System32, DACL e relazione con Bitdefender.

7.  [GitHub: PR #16616, avviso per server API non sicuro](https://github.com/rustdesk/rustdesk/pull/16616): condizioni e testo del nuovo avviso.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): nessun advisory pubblicato (situazione al 2 novembre 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): ultima versione del server 1.1.16 del 20 luglio 2026.

10.  [RustDesk: configurare l'alternativa open source a TeamViewer](/blog/rustdesk-teamviewer-alternative): configurazione del client e del server proprio.
