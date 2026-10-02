---
title: "Proton Drive su Linux: situazione a ottobre 2026"
navTitle: "Proton Drive e Linux"
description: "Il client Linux ufficiale è stato annunciato, ma non è ancora disponibile. Da giugno 2026 esiste la Proton Drive CLI ufficiale per script e server; Proton Drive può ancora essere montato solo con Rclone. Manca un accesso macchina limitato a singole cartelle o attività."
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
translatedAt: 2026-10-02T10:02:08.151Z
translationReview: automatic
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/it/blog/proton-drive-su-linux-situazione-a-luglio-2026
---

Per Windows e macOS, Proton Drive offre client di sincronizzazione propri dal 2023. Su Linux esistono finora l'interfaccia web, strumenti della community e, da giugno 2026, un'applicazione ufficiale da riga di comando, ma non ancora un client di sincronizzazione. Su un server la situazione è ancora più difficile, poiché né la sincronizzazione desktop né un accesso interattivo sono adatti.

Questa panoramica descrive la situazione al 1° ottobre 2026. Si basa sulle roadmap pubblicate, sul codice sorgente della Proton Drive CLI e su un test pratico del backend Rclone [come archivio documentale per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Aggiornamento del 1° ottobre 2026:** La prima versione del 26 luglio descriveva l'applicazione da riga di comando soltanto come strumento nel repository dell'SDK. Tuttavia, Proton l'aveva già pubblicata ufficialmente il 9 giugno 2026 come **Proton Drive CLI**, con build pronte per Windows, macOS e Linux. La relativa sezione e la tabella delle raccomandazioni sono state aggiornate; i dettagli sono disponibili nell'articolo dedicato alla [Proton Drive CLI](/blog/proton-drive-cli).

## Il client Linux è annunciato, ma senza una data

Nel giugno 2026, Proton ha confermato per la prima volta esplicitamente che è in sviluppo un client Linux. Si basa sul nuovo SDK unificato e dovrebbe usare la stessa base tecnica delle applicazioni per Windows e macOS. All'inizio di ottobre 2026 non esistono ancora né una data né una beta pubblica.

Importante per inquadrare la questione: sarà un **client di sincronizzazione desktop**. Per il desktop risolve il problema. Per le applicazioni server, invece, un client di sincronizzazione è lo strumento sbagliato, perché un servizio deve leggere i file direttamente da Proton Drive e scriverli lì. Un client di sincronizzazione mantiene una copia locale completa, proprio ciò che si vuole evitare quando lo spazio è limitato.

## Rclone resta necessario per mount e mirroring

Su Linux, Rclone con il suo backend `protondrive` è attualmente lo strumento più versatile. Può copiare e sincronizzare file e, quale unica soluzione disponibile, rendere Proton Drive disponibile come directory locale tramite **mount FUSE**. Due limitazioni sono importanti:

**È in beta su un'API ricostruita.** Proton non documenta pubblicamente la propria API Drive; il backend si basa sul reverse engineering. Nel test ha funzionato in modo affidabile, ma ha applicato limitazioni di velocità in caso di sequenze rapide di chiamate con elenchi di directory incoerenti.

**Per il funzionamento non supervisionato, Rclone richiede la chiave TOTP.** La procedura guidata di configurazione chiama il campo `otp_secret_key`. Si intende la chiave permanente della configurazione 2FA, non il codice a sei cifre visualizzato in quel momento da un'app Authenticator. Rclone memorizza questo valore in forma offuscata e genera autonomamente un codice TOTP valido a ogni accesso.

Chi inserisce per errore un codice monouso attuale può completare il primo accesso. Tuttavia, la successiva autenticazione fallisce con l'errore 8002, perché Rclone non può usare nuovamente lo stesso codice.

In questo modo l'account rimane protetto da una password rubata in modo isolato. Un server compromesso, tuttavia, espone password e chiave TOTP. Per gli accessi automatizzati è quindi consigliabile un **account Proton dedicato**.

Il comportamento di un simile mount negli ambienti Docker, inclusi due problemi non documentati, è illustrato nell'[articolo dedicato a Rclone nei container](/blog/rclone-mount-in-docker-container).

## La CLI ufficiale copre script e backup

Il 9 giugno 2026 Proton ha pubblicato la **Proton Drive CLI**, un singolo file eseguibile `proton-drive` per Windows, macOS e Linux. Si basa sullo stesso SDK delle app ufficiali; il codice sorgente si trova nel repository SDK pubblico. Attualmente è disponibile la versione 0.8.0 del 13 agosto 2026, con build per x86-64 (anche senza AVX2), ARM64 e distribuzioni musl come Alpine.

Il modello di accesso è più pulito rispetto a quello del backend Rclone:

- `auth login` restituisce un URL di accesso che può essere aperto anche **su un altro dispositivo**; l'accesso avviene regolarmente **inclusa l'autenticazione a due fattori**, quindi anche via SSH su un server senza desktop
- la sessione viene salvata nel **portachiavi del sistema operativo** (Keychain, Credential Manager, libsecret) oppure, dalla versione 0.6.0, nel gestore di password `pass`, più pratico sui server senza sessione desktop
- successivamente: caricare e scaricare file, spostarli, cestinarli, gestire condivisioni, inviti e link pubblici, usare Proton Photos; ogni operazione con `--json` per un output leggibile dalle macchine

Password e chiave TOTP non devono quindi risiedere sul server. Per backup e artefatti di build, la CLI è oggi la scelta migliore rispetto a Rclone. Restano due limiti: la CLI non può **montare un filesystem** né **creare un mirror con eliminazioni**; carica e scarica, ma non sincronizza. Un comando `takeout` per un'esportazione locale completa è già presente nel repository, ma non è ancora stato pubblicato.

Proton continua a classificare l'SDK stesso come non pronto per la produzione per applicazioni di terze parti; il rilascio è previsto tra la fine del 2026 e l'inizio del 2027. La CLI non ne è interessata, poiché è distribuita da Proton stesso.

## La vera lacuna: gli accessi macchina

Il nucleo del problema si trova a un livello più profondo rispetto al client o all'SDK: **Proton non conosce accessi macchina.** Nessuna password per app, nessun account di servizio, nessun token con ambito limitato. Ogni automazione, sia uno script di backup, un mount su server o un job CI, deve operare con le credenziali complete dell'account.

A confronto: negli archivi compatibili con S3, le coppie di chiavi di accesso sono la norma, revocabili e limitabili a bucket o prefissi. Google e Microsoft offrono password per app e account di servizio. Con Proton, invece, vale il tutto o niente: chi vuole dare a un server accesso a una cartella, gli concede l'intero account.

In un servizio crittografato end-to-end ciò è più difficile che con S3, poiché un accesso limitato dovrebbe significare anche materiale di chiave limitato. Tuttavia, le sessioni della CLI mostrano che Proton padroneggia tali costruzioni. Una sessione è già un accesso derivato e revocabile, soltanto con l'ambito completo dell'account. Un ufficiale «token macchina per questa precisa cartella, sola lettura» sarebbe il più grande singolo progresso per l'uso sui server, ben prima di qualsiasi client.

## Raccomandazione per caso d'uso

| Caso d'uso | Situazione a ottobre 2026 |
|---|---|
| Sincronizzazione desktop su Linux | Attendere il client annunciato; fino ad allora, sincronizzazione Rclone o interfaccia web |
| Backup server (caricamento di file) | [Proton Drive CLI](/blog/proton-drive-cli) con `filesystem upload` e strategia di conflitto `create-new-revision`; supportata ufficialmente, senza password memorizzata |
| Mirror con eliminazioni | Rclone con `sync`; considerare lo stato beta |
| Mount del filesystem per servizi | Rclone con `mount`, chiave TOTP memorizzata e account dedicato; l'unica [soluzione collaudata nella pratica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automazione tramite script, gestione delle condivisioni | Proton Drive CLI con `--json`; versione 0.x, i comandi possono ancora cambiare |

Sul desktop Linux si può attendere il client annunciato oppure usare per ora Rclone. Sui server, la CLI ufficiale gestisce ormai backup e automazione; per un mount, Rclone resta l'unica soluzione praticabile. Tuttavia, un espediente funzionante diventerà una piattaforma affidabile soltanto quando Proton offrirà accessi macchina limitati e un mount supportato ufficialmente.

## Fonti

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la conferma del giugno 2026 che il client Linux è in sviluppo, senza una data.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la roadmap con il client Linux senza una finestra temporale e l'SDK come fondamento delle app proprietarie.

3.  [ProtonDriveApps/sdk su GitHub](https://github.com/ProtonDriveApps/sdk): il repository SDK pubblico con il codice sorgente e il changelog della CLI.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): la pubblicazione ufficiale della CLI il 9 giugno 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): l'attuale versione 0.8.0 del 13 agosto 2026 con tutte le build per piattaforma.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): la valutazione di Proton stesso: non ancora pronto per la produzione per applicazioni di terze parti.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend con l'avvertenza beta e l'opzione `otp_secret_key` per l'accesso non supervisionato.
