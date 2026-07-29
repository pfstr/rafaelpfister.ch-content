---
title: "Proton Drive su Linux: situazione a luglio 2026"
navTitle: "Proton Drive e Linux"
description: "Il client Linux ufficiale è stato annunciato, ma non è ancora disponibile. Sui server Proton Drive può attualmente essere montato con Rclone; il nuovo SDK indica la direzione tecnica. Continua invece a mancare un accesso macchina limitato a singole cartelle o attività."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min di lettura"
themen:
  - proton-drive
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-su-linux-situazione-a-luglio-2026"
translationOf: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/it/blog/proton-drive-su-linux-situazione-a-luglio-2026"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:50:48.084Z
translationReview: automatic
translationSourceHash: 1b0af572e102121912376d523c1785ed1563e4ca5c17eee8d605c5000096b57e
---

Per Windows e macOS, Proton Drive offre client di sincronizzazione dedicati dal 2023. Su Linux finora esistono solo l'interfaccia web, strumenti della comunità e un SDK ufficiale in fase di anteprima. Su un server la situazione è ancora più difficile, perché né la sincronizzazione desktop né un accesso interattivo sono adatti.

Questa panoramica descrive la situazione a luglio 2026. Oltre alle roadmap pubblicate, si basa su un test pratico del backend Rclone [come archivio documentale per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Il client Linux è annunciato, ma senza una data

Nel giugno 2026 Proton ha confermato per la prima volta esplicitamente lo sviluppo di un client Linux. Sarà basato sul nuovo SDK unificato e utilizzerà la stessa base tecnica delle applicazioni per Windows e macOS. Non esistono ancora una data né una beta pubblica.

Importante per inquadrare la questione: sarà un **client di sincronizzazione desktop**. Per il desktop risolve il problema. Per le applicazioni server, tuttavia, un client di sincronizzazione è lo strumento sbagliato, poiché un servizio deve leggere i file direttamente da Proton Drive e scrivervi. Un client di sincronizzazione mantiene una copia locale completa, proprio ciò che si vuole evitare quando lo spazio di archiviazione è limitato.

## Oggi Rclone svolge il lavoro pratico

Su Linux, Rclone con il suo backend `protondrive` è attualmente lo strumento più versatile. Può copiare e sincronizzare file e, quale unica soluzione disponibile, rendere Proton Drive accessibile come una directory locale tramite **mount FUSE**. Due limitazioni sono importanti:

**È in beta su un'API ricostruita.** Proton non documenta pubblicamente la sua API Drive; il backend si basa sul reverse engineering. Nel test ha funzionato in modo affidabile, ma con sequenze di chiamate rapide applicava limitazioni con elenchi di directory incoerenti.

**Per il funzionamento non sorvegliato Rclone richiede la chiave TOTP.** L'assistente di configurazione indica il campo come `otp_secret_key`. Si intende la chiave permanente della configurazione 2FA, non il codice a sei cifre mostrato in quel momento da un'app Authenticator. Rclone salva questo valore in forma offuscata e genera autonomamente un codice TOTP valido a ogni accesso.

Chi inserisce per errore un codice monouso corrente può completare il primo accesso. La successiva nuova autenticazione fallisce però con l'errore 8002, perché Rclone non può utilizzare di nuovo lo stesso codice.

In questo modo l'account resta protetto da una password rubata isolatamente. Un server compromesso espone però password e chiave TOTP. Per gli accessi automatizzati è quindi consigliabile un **account Proton dedicato**.

Il comportamento di un simile mount negli ambienti Docker, incluse due insidie non documentate, è descritto nell'[articolo dedicato a Rclone nei container](/blog/rclone-mount-in-docker-container).

## L'SDK ufficiale mostra la direzione dello sviluppo

Parallelamente, Proton sta migrando le proprie applicazioni a un **SDK ufficiale** per JavaScript e C#, con binding per Swift e Kotlin. Il repository pubblico contiene anche uno strumento da riga di comando. Il suo modello di accesso è più pulito di quello del backend Rclone:

- `auth login` apre il browser; l'accesso avviene regolarmente **inclusa l'autenticazione a due fattori**
- la sessione viene memorizzata nel **portachiavi del sistema operativo** (Keychain, Credential Manager, libsecret), e l'SDK la rinnova autonomamente
- in seguito: elencare file, caricarli e verificare le condivisioni con output JSON leggibile dalle macchine

Password e chiave TOTP non devono quindi essere presenti in un file di configurazione. Per l'uso su server restano comunque tre limiti: la CLI **non può montare un filesystem**, l'accesso apre un browser e Proton non considera ancora l'SDK pronto per la produzione nelle applicazioni di terze parti. Il rilascio è previsto tra la fine del 2026 e l'inizio del 2027.

## La vera lacuna: gli accessi macchina

Il nucleo del problema è a un livello più profondo di client o SDK: **Proton non prevede accessi macchina.** Nessuna password per app, nessun account di servizio, nessun token con ambito limitato. Qualsiasi automazione, sia essa uno script di backup, un mount server o un job CI, deve operare con le credenziali complete dell'account.

A confronto: negli archivi compatibili con S3, le coppie di chiavi di accesso sono la norma, revocabili e limitabili a bucket o prefissi. Google e Microsoft offrono password per app e account di servizio. Con Proton, invece, è tutto o niente: chi vuole concedere a un server l'accesso a una cartella, gli concede l'intero account.

A onor del vero, per un servizio con crittografia end-to-end è più difficile che con S3, perché un accesso limitato richiederebbe anche materiale crittografico limitato. Le sessioni dell'SDK mostrano però che Proton gestisce tali costruzioni. Una sessione è già un accesso derivato e revocabile. Un ufficiale «token macchina per questa specifica cartella, in sola lettura» sarebbe il più grande singolo progresso per l'uso su server, molto prima di qualsiasi client.

## Raccomandazione per caso d'uso

| Caso d'uso | Situazione a luglio 2026 |
|---|---|
| Sincronizzazione desktop su Linux | Attendere il client annunciato; nel frattempo sincronizzazione Rclone o interfaccia web |
| Backup server (caricamento di file) | Rclone con `copy` o `sync`; funziona, tenere conto dello stato beta |
| Mount filesystem per servizi | Rclone con `mount`, chiave TOTP memorizzata e account dedicato; l'unica [soluzione sperimentata nella pratica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automazione tramite script con accesso pulito | Tenere d'occhio la CLI dell'SDK; ancora troppo presto per la produzione |

Sul desktop Linux si può attendere il client annunciato o utilizzare per ora Rclone. Sui server, Rclone rimane l'unica soluzione di mount praticabile. Tuttavia, un espediente funzionante diventerà una piattaforma affidabile solo quando Proton offrirà accessi macchina limitati e un mount ufficialmente supportato.

## Fonti

1.  [OMG Ubuntu: il client Proton Drive sta (finalmente) arrivando su Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la conferma del giugno 2026 che il client Linux è in sviluppo, senza una data.

2.  [Proton: roadmap dei prodotti per la primavera e l'estate 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la roadmap con il client Linux senza una finestra temporale e l'SDK come fondamento delle proprie app.

3.  [ProtonDriveApps/sdk su GitHub](https://github.com/ProtonDriveApps/sdk): il repository SDK pubblico, inclusa la CLI con accesso via browser e sessione nel portachiavi.

4.  [Anteprima dell'SDK Proton Drive](https://proton.me/blog/proton-drive-sdk-preview): la valutazione di Proton: non ancora pronto per la produzione nelle applicazioni di terze parti.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend con l'avviso beta e l'opzione `otp_secret_key` per l'accesso non sorvegliato.
