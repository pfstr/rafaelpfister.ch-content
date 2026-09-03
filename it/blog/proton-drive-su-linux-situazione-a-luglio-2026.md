---
title: "Proton Drive su Linux: situazione a luglio 2026"
navTitle: "Proton Drive e Linux"
description: "Il client Linux ufficiale è stato annunciato, ma non è ancora disponibile. Sui server, Proton Drive può attualmente essere integrato con Rclone; il nuovo SDK indica la direzione tecnica. Ciò che continua a mancare è un accesso macchina limitato a singole cartelle o attività."
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
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:26:46.733Z
translationReview: automatic
translationSourceHash: dc35729208664efc8971642a0b5e67d38634e3871b66e1a448a41d14cd2d67b3
url: https://rafaelpfister.ch/it/blog/proton-drive-su-linux-situazione-a-luglio-2026
---

Per Windows e macOS, Proton Drive offre client di sincronizzazione propri dal 2023. Su Linux esistono finora solo l'interfaccia web, strumenti della community e un SDK ufficiale in fase di anteprima. Su un server la situazione è ancora più difficile, poiché né una sincronizzazione desktop né un accesso interattivo sono adatti.

Questa panoramica descrive la situazione a luglio 2026. Oltre alle roadmap pubblicate, si basa su un test pratico del backend Rclone [come archivio di documenti per Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Il client Linux è annunciato, ma senza una data

Nel giugno 2026 Proton ha confermato per la prima volta esplicitamente che è in sviluppo un client Linux. Sarà basato sul nuovo SDK unificato e utilizzerà la stessa base tecnica delle applicazioni per Windows e macOS. Non esistono ancora né una data né una beta pubblica.

Importante per contestualizzare: sarà un **client di sincronizzazione desktop**. Per il desktop risolve il problema. Per le applicazioni server, invece, un client di sincronizzazione è lo strumento sbagliato, perché un servizio deve leggere file direttamente da Proton Drive e scrivervi. Un client di sincronizzazione mantiene una copia locale completa, proprio ciò che si vuole evitare quando lo spazio di archiviazione è limitato.

## Oggi Rclone svolge il lavoro pratico

Su Linux, Rclone con il suo backend `protondrive` è attualmente lo strumento più versatile. Può copiare e sincronizzare file e, come unica soluzione disponibile, rendere Proton Drive disponibile come directory locale tramite **mount FUSE**. Due limitazioni sono importanti:

**È in beta su un'API ricostruita.** Proton non documenta pubblicamente la propria API Drive; il backend si basa sul reverse engineering. Nel test ha funzionato in modo affidabile, ma con sequenze rapide di chiamate applicava limitazioni restituendo elenchi di directory incoerenti.

**Per il funzionamento non presidiato, Rclone richiede la chiave TOTP.** L'assistente di configurazione chiama il campo `otp_secret_key`. Si intende la chiave permanente della configurazione 2FA, non il codice a sei cifre visualizzato in quel momento da un'app Authenticator. Rclone salva questo valore in forma offuscata e genera autonomamente un codice TOTP valido a ogni accesso.

Chi inserisce per errore un codice monouso attuale può completare il primo accesso. La successiva nuova autenticazione fallisce però con errore 8002, perché Rclone non può utilizzare nuovamente lo stesso codice.

In questo modo l'account rimane protetto da una password rubata isolatamente. Un server compromesso espone tuttavia sia la password sia la chiave TOTP. Per gli accessi automatizzati è quindi consigliabile un **account Proton dedicato**.

Il comportamento di un simile mount negli ambienti Docker, compresi due problemi non documentati, è descritto nell'[articolo dedicato a Rclone nei container](/blog/rclone-mount-in-docker-container).

## L'SDK ufficiale mostra la direzione dello sviluppo

Parallelamente, Proton sta migrando le proprie applicazioni a un **SDK ufficiale** per JavaScript e C#, con binding per Swift e Kotlin. Il repository pubblico contiene anche uno strumento a riga di comando. Il suo modello di accesso è più pulito di quello del backend Rclone:

- `auth login` apre il browser; l'accesso avviene regolarmente **inclusa l'autenticazione a due fattori**
- la sessione finisce nel **portachiavi del sistema operativo** (Keychain, Credential Manager, libsecret), e l'SDK la rinnova autonomamente
- successivamente: elencare, caricare file e verificare le condivisioni con output JSON leggibile dalle macchine

In questo modo password e chiave TOTP non devono essere presenti in un file di configurazione. Per l'uso su server rimangono comunque tre limiti: la CLI **non può montare un file system**, l'accesso apre un browser e Proton non considera ancora l'SDK pronto per la produzione per applicazioni di terze parti. Il rilascio è previsto tra la fine del 2026 e l'inizio del 2027.

## La vera lacuna: gli accessi macchina

Il nocciolo del problema si trova a un livello più profondo di client o SDK: **Proton non prevede accessi macchina.** Nessuna password per app, nessun account di servizio, nessun token con ambito limitato. Ogni automazione, sia essa uno script di backup, un mount server o un job CI, deve operare con le credenziali complete dell'account.

Per confronto: negli archivi compatibili con S3, le coppie di chiavi di accesso sono la norma, revocabili e limitabili a bucket o prefissi. Google e Microsoft offrono password per app e account di servizio. Con Proton, invece, vale tutto o niente: chi vuole dare a un server accesso a una cartella gli dà accesso all'intero account.

Con un servizio crittografato end-to-end ciò è più difficile che con S3, perché un accesso limitato dovrebbe implicare anche materiale crittografico limitato. Le sessioni SDK mostrano però che Proton padroneggia tali meccanismi. Una sessione è già un accesso derivato e revocabile. Un ufficiale «token macchina per esattamente questa cartella, in sola lettura» sarebbe il maggiore progresso individuale per l'uso su server, ben prima di qualsiasi client.

## Raccomandazione per caso d'uso

| Caso d'uso | Situazione a luglio 2026 |
|---|---|
| Sincronizzazione desktop su Linux | Attendere il client annunciato; nel frattempo sincronizzazione Rclone o interfaccia web |
| Backup server (caricamento file) | Rclone con `copy` o `sync`; funziona, tenere conto dello stato beta |
| Mount del file system per servizi | Rclone con `mount`, chiave TOTP memorizzata e account dedicato; l'unica [soluzione collaudata nella pratica](/blog/paperless-dokumente-clouddienst-auslagern) |
| Automazione tramite script con accesso pulito | Tenere d'occhio la CLI dell'SDK; ancora troppo presto per la produzione |

Sul desktop Linux si può attendere il client annunciato oppure usare per ora Rclone. Sui server, Rclone resta l'unica soluzione di mount praticabile. Tuttavia, un espediente funzionante diventerà una piattaforma affidabile solo quando Proton offrirà accessi macchina limitati e un mount ufficialmente supportato.

## Fonti

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): la conferma di giugno 2026 che il client Linux è in sviluppo, senza una data.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): la roadmap con il client Linux senza finestra temporale e l'SDK come base delle proprie app.

3.  [ProtonDriveApps/sdk su GitHub](https://github.com/ProtonDriveApps/sdk): il repository SDK pubblico, inclusa la CLI con accesso tramite browser e sessione nel portachiavi.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): la valutazione di Proton: non ancora pronto per la produzione per applicazioni di terze parti.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): il backend con l'avviso beta e l'opzione `otp_secret_key` per l'accesso non presidiato.
