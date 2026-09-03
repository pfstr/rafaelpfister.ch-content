---
title: "Eseguire Paperless-ngx con poco spazio di archiviazione: esternalizzare i documenti a un servizio cloud"
navTitle: "Paperless con servizio cloud"
description: "Paperless-ngx necessita solo localmente del database, dell’indice di ricerca e delle anteprime; i documenti stessi possono risiedere in un servizio cloud. Cosa ha rivelato il test pratico e come completare la configurazione con il modello pronto in tre comandi."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min di lettura"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-spazio-limitato-documenti-cloud"
translationOf: "paperless-dokumente-clouddienst-auslagern"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:24:35.798Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/it/blog/paperless-ngx-spazio-limitato-documenti-cloud
---

Paperless-ngx archivia i documenti in una directory locale, che cresce con ogni scansione. Tuttavia, Paperless usa raramente i file nell’uso quotidiano: la ricerca interroga il database, l’elenco mostra le anteprime e il file vero e proprio viene letto solo quando lo si apre. Ho quindi verificato se fosse possibile spostare l’archivio in un servizio cloud. Lo strumento per farlo è Rclone, con cui gli utenti di Plex integrano da anni intere raccolte multimediali dal cloud.

Il risultato: **funziona in entrambe le direzioni** e la configurazione si è ormai ridotta a tre comandi. Questo articolo riassume i risultati del test e spiega come realizzare autonomamente il setup. I dettagli tecnici sono trattati in articoli dedicati, collegati alla fine: mount propagation di Docker, peculiarità di AppArmor, autenticazione a due fattori e metodologia di misurazione.

## Il principio: l’hot storage resta locale, il cold storage è nel cloud

| Componente | Posizione | Motivo |
|---|---|---|
| Database (contiene il testo OCR) | locale | richiede un locking reale |
| Indice di ricerca, anteprime | locale | accessi continui |
| **File dei documenti** | **Cloud** | vengono letti raramente |
| Cache (documenti aperti più di recente) | locale, limitata | gli accessi ripetuti restano veloci |

In Paperless, proprio il nome della directory trae in inganno: `archive/` **non è cold storage**, bensì contiene la versione PDF/A fornita a ogni visualizzazione. Nonostante il nome, fa parte dell’hot storage. Gli originali raramente necessari in `originals/` sono il vero cold storage. Se si desidera risparmiare al massimo, è possibile disattivare completamente la copia archivio con `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; la ricerca full-text non ne risente, poiché il testo si trova nel database.

Paperless-ngx non offre peraltro un’integrazione cloud nativa, né S3 né `django-storages`. Al momento, un mount del filesystem tramite Rclone è l’unica soluzione e funziona con ciascuno degli oltre 70 servizi supportati da Rclone. Proton Drive è stata la mia scelta per il test grazie alla crittografia end-to-end; uno storage compatibile con S3 è l’alternativa più robusta.

## Cosa ha rivelato il test

Test eseguito con un’istanza Paperless isolata, 40 PDF di prova generati (13.9 MB) e un account Proton dedicato:

| Operazione | Risultato |
|---|---|
| Aprire un documento per la prima volta (dal cloud) | ~1.8 s |
| Riaprire lo stesso documento (dalla cache) | ~20 ms |
| Acquisire un nuovo documento, fino a quando si trova nel cloud | ~20 s |
| Elenco documenti, ricerca full-text | 39 ms / 272 ms, funziona anche **senza** connessione cloud |
| Verifica dell’integrità (checksum di ogni file) | superata, nessuna discrepanza |
| Interruzione del mount | autoripristino senza riavviare Paperless, verificato |

Il fabbisogno di spazio locale è quindi disaccoppiato dalla dimensione dell’archivio: la raccolta può crescere, il disco no.

## Come configurarlo

La configurazione completa è disponibile come modello su GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). Sul server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

Il wizard chiede il servizio cloud (Proton, S3, Backblaze B2, WebDAV, SFTP o “Not in the list” per qualsiasi altro servizio Rclone), verifica la connessione con un test reale di caricamento e scaricamento e avvia il container di storage. Quindi:

- **Nuova installazione:** `docker compose -f paperless.yml up -d`, fatto.
- **Istanza Paperless esistente:** database e impostazioni restano invariati; la guida [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) descrive il caricamento dei documenti esistenti e la modifica necessaria al file Compose.

Ho volutamente rinunciato a un’interfaccia web. Inizialmente era in uso la GUI web di Rclone, ma tunnel SSH, CORS e mount effimeri la rendevano peggiore della riga di comando che avrebbe dovuto sostituire. Tre domande nel terminale sono più rapide.

## Per mantenere stabile il mount nell’uso quotidiano

Il modello gestisce quattro aspetti che devono essere considerati anche in una configurazione personalizzata:

1. **`propagation: rslave`** sul media bind mount del container Paperless, altrimenti il container non sopravvive al riavvio del mount. Dettagli e il problema AppArmor sottostante: [Mount Rclone nel container Docker](/blog/rclone-mount-in-docker-container).
2. **Arrestare Paperless se il mount manca.** Altrimenti scrive i documenti in una directory locale vuota e il mount che ritorna li copre in modo invisibile. Uno script watchdog è incluso nel modello.
3. **Un account che possa autenticarsi senza supervisione.** Con Proton significa memorizzare la chiave TOTP nella configurazione di Rclone. Perché questo non svuota di significato l’autenticazione a due fattori e qual è la situazione complessiva di Proton su Linux: [Proton Drive su Linux](/blog/proton-drive-linux-status).
4. **Disattivare le attività pianificate di lettura completa** (`PAPERLESS_SANITY_TASK_CRON=disable`), poiché altrimenti il controllo d’integrità legge regolarmente l’intero archivio dal cloud.

## Cosa valutare prima dell’uso

Un documento appena acquisito rimane solo nella cache locale per alcuni secondi, finché il caricamento non è completato. Se la macchina si guasta proprio in questa finestra, il file manca. Il limite della cache è flessibile e può essere superato notevolmente per breve tempo durante picchi di accesso. Inoltre, il backend Proton di Rclone è ufficialmente in beta; con rapide chiamate API ha mostrato sintomi di throttling. Poiché mancano ancora dati a lungo termine dall’uso continuativo, il modello è contrassegnato come sperimentale.

L’articolo sulla metodologia spiega come sono stati ottenuti i valori misurati, quali guasti sono stati simulati e come testare seriamente una configurazione di questo tipo: [Testare i cloud mount con PDF generati](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusione

Paperless-ngx su un disco piccolo con archiviazione cloud è realizzabile e adatto all’uso quotidiano: poco meno di due secondi alla prima apertura, poi velocità della cache; ricerca e interfaccia restano indipendenti dal cloud e la configurazione si autoripristina dopo i guasti. Se si desidera risparmiare solo qualche gigabyte su un server di dimensioni normali, tuttavia, conviene fare i conti: nel mio caso l’intero archivio occupava 71 MB, mentre il sistema operativo diversi gigabyte. Il vantaggio non è lo spazio risparmiato immediatamente, ma il fatto che l’archivio possa crescere senza che debba crescere anche il disco.

## Fonti

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): il modello di questo articolo: setup.sh, wizard.sh, file Compose, watchdog e guida al retrofit.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): gli oltre 70 servizi supportati e le loro funzionalità a confronto.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` e le altre impostazioni utilizzate.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): Sanity Checker, esportazione e importazione, nonché le attività pianificate in background.
