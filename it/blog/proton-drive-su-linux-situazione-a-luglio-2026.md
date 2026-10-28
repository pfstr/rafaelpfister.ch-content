---
title: "Proton Drive su Linux: situazione a ottobre 2026"
navTitle: "Proton Drive e Linux"
description: "Il client Linux ufficiale è stato annunciato, ma non è ancora disponibile. Da giugno 2026 esiste la Proton Drive CLI ufficiale per script e server; Proton Drive continua a poter essere montato solo con Rclone. Manca un accesso macchina limitato a singole cartelle o attività."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min di lettura"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-su-linux-situazione-a-luglio-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:18:36.538Z
translationReview: automatic
translationSourceHash: e32f1f479477b6a2eb1ff5ad2fb4e4144953faad34f2ac9302f3b2338ca22047
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/proton-drive-su-linux-situazione-a-luglio-2026
---

Per Windows e macOS, Proton Drive offre client di sincronizzazione propri dal 2023. Su Linux sono finora disponibili l'interfaccia web, strumenti della community e, da giugno 2026, un'applicazione ufficiale a riga di comando, ma non ancora un client di sincronizzazione. Su un server la situazione è ancora più difficile, perché né la sincronizzazione desktop né un accesso interattivo sono adatti.

Questa panoramica descrive la situazione al 1° ottobre 2026. Si basa sulle roadmap pubblicate, sul codice sorgente della Proton Drive CLI e su un test pratico del backend Rclone [come archivio documentale per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Aggiornamento del 1° ottobre 2026:** La prima versione del 26 luglio descriveva l'applicazione a riga di comando solo come strumento nel repository SDK. Tuttavia, Proton l'aveva già pubblicata ufficialmente il 9 giugno 2026 come **Proton Drive CLI**, con build pronte per Windows, macOS e Linux. La relativa sezione e la tabella delle raccomandazioni sono state aggiornate; i dettagli sono disponibili nell'articolo dedicato alla [Proton Drive CLI](/blog/proton-drive-cli).

## Il client Linux è annunciato, ma non ha ancora una data

Nel giugno 2026 Proton ha confermato esplicitamente per la prima volta che è in sviluppo un client Linux. Si basa sul nuovo SDK unificato e utilizzerà la stessa base tecnica delle applicazioni per Windows e macOS. All'inizio di ottobre 2026 non ci sono ancora né una data né una beta pubblica.

Importante per inquadrare la questione: sarà un **client di sincronizzazione desktop**. Per il desktop risolve il problema. Per le applicazioni server, invece, un client di sincronizzazione è lo strumento sbagliato, perché un servizio deve leggere direttamente i file da Proton Drive e scrivervi. Un client di sincronizzazione conserva una copia locale completa, proprio ciò che si vuole evitare quando lo spazio è limitato.

## Rclone resta necessario per mount e mirroring

Su Linux, Rclone con il suo backend `protondrive` è attualmente lo strumento più versatile. Può copiare e sincronizzare file e, come unica soluzione disponibile, rendere Proton Drive disponibile come directory locale tramite un **mount FUSE**. Due limitazioni sono importanti:

**È in beta su un'API ricostruita.** Proton non documenta pubblicamente la propria API Drive; il backend si basa sul reverse engineering. Nel test ha funzionato in modo affidabile, ma ha applicato limitazioni a sequenze rapide di chiamate con elenchi di directory incoerenti.

**Per il funzionamento non supervisionato, Rclone richiede la chiave TOTP.** L'assistente di configurazione chiama il campo `otp_secret_key`. Si intende la chiave permanente della configurazione 2FA, non il codice a sei cifre mostrato in quel momento da un'app Authenticator. Rclone memorizza questo valore in forma offuscata e genera autonomamente un codice TOTP valido a ogni accesso.

Chi inserisce per errore un codice monouso corrente può completare il primo accesso. Tuttavia, la successiva riautenticazione fallisce con l'errore 8002, perché Rclone non può usare di nuovo lo stesso codice.

In questo modo l'account rimane protetto contro una password rubata in modo isolato. Un server compromesso rivela però password e chiave TOTP. Per gli accessi automatizzati è quindi consigliabile un **account Proton dedicato**.

Il comportamento di un simile mount in ambienti Docker, inclusi due problemi non documentati, è trattato nell'[articolo dedicato a Rclone nei container](/blog/rclone-mount-in-docker-container).

## La CLI ufficiale copre script e backup

Il 9 giugno 2026 Proton ha pubblicato la **Proton Drive CLI**, un singolo file eseguibile `proton-drive` per Windows, macOS e Linux. Si basa sullo stesso SDK delle app ufficiali; il codice sorgente è disponibile nel repository SDK pubblico. Attualmente è disponibile la versione 0.9.0 del 5 ottobre 2026, con build per x86-64 (anche senza AVX2), ARM64 e distribuzioni musl come Alpine.

Il modello di accesso è più pulito di quello del backend Rclone:

- `auth login` restituisce un URL di accesso che può essere aperto anche **su un altro dispositivo**; l'accesso avviene regolarmente **inclusa l'autenticazione a due fattori**, quindi anche via SSH su un server senza desktop
- la sessione viene archiviata nel **portachiavi del sistema operativo** (Keychain, Credential Manager, libsecret) oppure, dalla versione 0.6.0, nel gestore di password `pass`, più pratico sui server senza sessione desktop
- in seguito: caricare e scaricare file, spostarli, inviarli nel cestino, gestire condivisioni, inviti e link pubblici, usare Proton Photos; ogni volta con `--json` per un output leggibile dalle macchine

Password e chiave TOTP non devono quindi risiedere sul server. Per backup e artefatti di build, oggi la CLI è perciò la scelta migliore rispetto a Rclone. Restano due limiti: la CLI non può **montare un filesystem** né **creare un mirror con eliminazioni**; carica e scarica, ma non sincronizza. Dalla versione 0.9.0 del 5 ottobre 2026 è stato inoltre pubblicato il comando `takeout` per un'esportazione locale completa.

L'SDK stesso continua a non essere classificato da Proton come pronto per la produzione per applicazioni di terze parti; il rilascio è previsto tra la fine del 2026 e l'inizio del 2027. La CLI non è interessata, perché è pubblicata da Proton stessa.

## La vera lacuna: gli accessi macchina

Il nucleo del problema si trova a un livello più profondo di client o SDK: **Proton non conosce accessi macchina.** Nessuna password per app, nessun account di servizio, nessun token con ambito limitato. Ogni automazione, sia essa uno script di backup, un mount server o un job CI, deve operare con le credenziali complete dell'account.

Per confronto: negli archivi compatibili con S3, le coppie di chiavi di accesso sono la norma, revocabili e limitabili a bucket o prefissi. Google e Microsoft dispongono di password per app e account di servizio. Con Proton, invece, vale tutto o niente: chi vuole dare a un server l'accesso a una cartella gli concede l'intero account.

Con un servizio crittografato end-to-end è più difficile che con S3, perché un accesso limitato dovrebbe implicare anche materiale crittografico limitato. Tuttavia, le sessioni della CLI mostrano che Proton sa gestire tali meccanismi. Una sessione è già un accesso derivato e revocabile, solo con l'intero ambito dell'account. Un ufficiale «token macchina per esattamente questa cartella, in sola lettura» sarebbe il maggiore singolo progresso per l'uso su server, ben prima di qualsiasi client.

## Raccomandazione per caso d'uso

| Caso d'uso | Situazione a ottobre 2026 |
|---|---|
| Sincronizzazione desktop su Linux | Attendere il client annunciato; fino ad allora sincronizzazione Rclone o interfaccia web |
| Backup del server (caricamento di file) | [Proton Drive CLI](/blog/proton-drive-cli) con `filesystem upload` e strategia di conflitto `create-new-revision`; supportata ufficialmente, senza password memorizzata |
| Mirror con eliminazioni | Rclone con `sync`; considerare lo stato beta |
| Mount del filesystem per servizi | Rclone con `mount`, chiave TOTP memorizzata e account dedicato; l'unica [soluzione testata nella pratica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automazione tramite script, gestione delle condivisioni | Proton Drive CLI con `--json`; versione 0.x, i comandi possono ancora cambiare |

Sul desktop Linux si può attendere il client annunciato oppure usare temporaneamente Rclone. Sui server, la CLI ufficiale gestisce ormai backup e automazione; per un mount Rclone resta l'unica soluzione praticabile. Tuttavia, un espediente funzionante diventerà una piattaforma solida solo quando Proton offrirà accessi macchina limitati e un mount supportato ufficialmente.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiornato alla Proton Drive CLI 0.9.0 del 5 ottobre 2026, con la funzione takeout ora pubblicata per un'esportazione locale completa.

## Fonti

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la conferma del giugno 2026 che il client Linux è in sviluppo, senza una data.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la roadmap con il client Linux senza una finestra temporale e l'SDK come fondamento delle app proprietarie.

3.  [ProtonDriveApps/sdk su GitHub](https://github.com/ProtonDriveApps/sdk): il repository SDK pubblico con codice sorgente e changelog della CLI (tra l'altro con il comando `takeout` nella versione 0.9.0).

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): la pubblicazione ufficiale della CLI da parte di Proton il 9 giugno 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): l'attuale versione 0.9.0 del 5 ottobre 2026 con tutte le build per piattaforma.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): la valutazione di Proton stessa: non ancora pronta per la produzione per applicazioni di terze parti.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend con l'avviso beta e l'opzione `otp_secret_key` per l'accesso non supervisionato.
