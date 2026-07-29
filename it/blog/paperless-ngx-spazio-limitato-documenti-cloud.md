---
title: "Utilizzare Paperless-ngx con poco spazio di archiviazione: esternalizzare i documenti a un servizio cloud"
navTitle: "Paperless con servizio cloud"
description: "Paperless-ngx richiede localmente solo database, indice di ricerca e miniature; i documenti stessi possono risiedere in un servizio cloud. I risultati del test pratico e come completare la configurazione con il modello pronto in tre comandi."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min di lettura"
themen:
  - "paperless-ngx"
related:
  - "rclone-mount-in-docker-container"
  - "proton-drive-linux-status"
  - "cloud-mount-testen-dummy-pdfs"
slug: "paperless-ngx-spazio-limitato-documenti-cloud"
translationOf: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/it/blog/paperless-ngx-spazio-limitato-documenti-cloud"
---

Paperless-ngx archivia i documenti in una directory locale, e questa directory cresce con ogni scansione. Tuttavia, Paperless usa raramente i file nell'uso quotidiano: la ricerca interroga il database, l'elenco mostra le miniature e il file effettivo viene letto solo quando lo si apre. Ho quindi verificato se fosse possibile spostare l'archivio in un servizio cloud. Lo strumento per farlo è Rclone, con cui gli utenti di Plex integrano da anni intere raccolte multimediali dal cloud.

Il risultato: **funziona in entrambe le direzioni** e la configurazione è ormai ridotta a tre comandi. Questo articolo riassume i risultati del test e spiega come creare autonomamente il setup. I dettagli tecnici sono raccolti in articoli separati, linkati alla fine: mount propagation di Docker, insidie di AppArmor, autenticazione a due fattori e metodologia di misurazione.

## Il principio: l'hot storage resta locale, il cold storage è nel cloud

| Componente | Posizione | Motivo |
|---|---|---|
| Database (contiene il testo OCR) | locale | richiede un locking reale |
| Indice di ricerca, miniature | locale | accessi continui |
| **File dei documenti** | **cloud** | vengono letti raramente |
| Cache (documenti aperti più di recente) | locale, limitata | gli accessi ripetuti restano rapidi |

In Paperless, proprio il nome della directory trae in inganno: `archive/` **non è cold storage**, ma contiene la versione PDF/A fornita a ogni visualizzazione. Nonostante il nome, appartiene all'hot storage. Gli originali raramente necessari in `originals/` sono il vero cold storage. Se vuoi risparmiare al massimo, disattiva completamente la copia d'archivio con `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; la ricerca full-text non ne risente, perché il testo si trova nel database.

Paperless-ngx non offre peraltro un'integrazione cloud propria, né S3 né `django-storages`. Un mount del filesystem tramite Rclone è attualmente l'unica soluzione e funziona con ciascuno degli oltre 70 servizi supportati da Rclone. Proton Drive è stata la mia scelta per i test grazie alla crittografia end-to-end; uno storage compatibile con S3 è l'alternativa più robusta.

## Cosa ha mostrato il test

Test effettuato con un'istanza Paperless isolata, 40 PDF di test generati (13,9 MB) e un account Proton dedicato:

| Operazione | Risultato |
|---|---|
| Aprire un documento per la prima volta (dal cloud) | ~1,8 s |
| Riaprire lo stesso documento (dalla cache) | ~20 ms |
| Acquisire un nuovo documento fino alla sua presenza nel cloud | ~20 s |
| Elenco documenti, ricerca full-text | 39 ms / 272 ms, funziona anche **senza** connessione cloud |
| Controllo di integrità (checksum di ogni file) | superato, nessuna discrepanza |
| Interruzione del mount | autoripristino senza riavvio di Paperless, verificato |

Il fabbisogno di spazio locale è quindi disaccoppiato dalle dimensioni dell'archivio: la raccolta può crescere, il disco no.

## Come configurarlo

La configurazione completa è disponibile come modello su GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). Sul server:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # una sola volta, prepara l'host (unico passaggio come root)
./wizard.sh          # guidato: scegliere il provider, credenziali, test completo
```

Il wizard interroga il servizio cloud (Proton, S3, Backblaze B2, WebDAV, SFTP oppure "Non nell'elenco" per qualsiasi altro servizio Rclone), verifica la connessione con un test reale di caricamento e scaricamento e avvia il container di storage. Dopodiché:

- **Nuova installazione:** `docker compose -f paperless.yml up -d`, fatto.
- **Istanza Paperless esistente:** database e impostazioni rimangono invariati; la guida [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) descrive il caricamento dei documenti esistenti e la modifica necessaria al tuo file Compose.

Ho deliberatamente rinunciato a un'interfaccia web. Inizialmente era in uso la GUI web di Rclone, ma tunnel SSH, CORS e mount effimeri la rendevano peggiore della riga di comando che avrebbe dovuto sostituire. Tre domande nel terminale sono più rapide.

## Per mantenere stabile il mount nell'uso quotidiano

Il modello gestisce quattro aspetti che devi considerare anche in una configurazione autonoma:

1. **`propagation: rslave`** sul media bind mount del container Paperless, altrimenti il container non sopravvive a un riavvio del mount. Dettagli e l'insidia di AppArmor alla base: [Mount Rclone nel container Docker](/blog/rclone-mount-in-docker-container).
2. **Arrestare Paperless se manca il mount.** Altrimenti scrive documenti in una directory locale vuota e il mount che torna li copre invisibilmente. Nel modello è incluso uno script watchdog.
3. **Un account che possa autenticarsi senza supervisione.** Con Proton significa memorizzare la chiave TOTP nella configurazione di Rclone. Perché questo non svaluta l'autenticazione a due fattori e qual è la situazione complessiva di Proton su Linux: [Proton Drive su Linux](/blog/proton-drive-linux-status).
4. **Disattivare le attività pianificate di lettura completa** (`PAPERLESS_SANITY_TASK_CRON=disable`), poiché altrimenti il controllo di integrità legge regolarmente dal cloud l'intero archivio.

## Cosa dovresti valutare prima dell'uso

Un documento appena acquisito resta per alcuni secondi solo nella cache locale, fino al completamento del caricamento. Se la macchina si guasta proprio in questa finestra, il file manca. Il limite della cache è flessibile e può essere superato temporaneamente in modo significativo durante picchi di accesso. Inoltre, il backend Proton di Rclone è ufficialmente in beta; con chiamate API rapide ha mostrato sintomi di throttling. Poiché mancano ancora dati a lungo termine dall'esercizio continuo, il modello è contrassegnato come sperimentale.

L'articolo sulla metodologia spiega come sono stati ottenuti i valori misurati, quali guasti sono stati simulati e come testare seriamente una configurazione simile: [Testare i mount cloud con PDF generati](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusione

Paperless-ngx su un piccolo disco con archiviazione cloud è fattibile e adatto all'uso quotidiano: poco meno di due secondi alla prima apertura, poi velocità di cache; ricerca e interfaccia restano indipendenti dal cloud e la configurazione si ripristina autonomamente dopo i guasti. Se invece vuoi risparmiare solo qualche gigabyte su un server di dimensioni normali, dovresti fare i conti: nel mio caso, l'intero archivio occupava 71 MB, il sistema operativo diversi gigabyte. Il vantaggio non sta nello spazio risparmiato subito, ma nel fatto che l'archivio può crescere senza che il disco debba crescere con esso.

## Fonti

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): il modello di questo articolo: setup.sh, wizard.sh, file Compose, watchdog e guida al retrofit.

2.  [Rclone: panoramica dei sistemi di cloud storage](https://rclone.org/overview/): gli oltre 70 servizi supportati e le loro funzionalità a confronto.

3.  [Paperless-ngx: configurazione](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` e le altre impostazioni utilizzate.

4.  [Paperless-ngx: amministrazione](https://docs.paperless-ngx.com/administration/): Sanity Checker, esportazione e importazione, nonché le attività pianificate in background.
