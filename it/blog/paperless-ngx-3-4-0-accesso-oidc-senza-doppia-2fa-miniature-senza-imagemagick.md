---
title: "Paperless-ngx 3.4.0: accesso OIDC senza doppia 2FA, miniature senza ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 salta la 2FA locale quando il provider OIDC segnala un secondo fattore attendibile, genera miniature PDF con Poppler anziché ImageMagick e aggiorna diverse dipendenze per ragioni di sicurezza. Prima dell’aggiornamento, verificate la nuova impostazione AMR e, sui sistemi bare metal, che pdftoppm sia installato."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min di lettura"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-accesso-oidc-senza-doppia-2fa-miniature-senza-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/it/blog/paperless-ngx-3-4-0-accesso-oidc-senza-doppia-2fa-miniature-senza-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:19:25.930Z
translationReview: automatic
---

# Paperless-ngx 3.4.0: accesso OIDC senza doppia 2FA, miniature senza ImageMagick

Il progetto Paperless-ngx ha pubblicato la versione 3.4.0 il 29 ottobre 2026, una settimana dopo la 3.3.4. Questa release di funzionalità introduce un’impostazione che elimina la richiesta locale di autenticazione a due fattori per gli accessi OIDC quando l’Identity Provider ha già confermato un secondo fattore. Paperless-ngx ora genera le miniature dei documenti PDF con gli strumenti Poppler; ImageMagick non è più una dipendenza. Sono inoltre presenti estensioni per workflow e campi personalizzati, modifiche al caricamento e alle notifiche nell’interfaccia web, otto correzioni di bug e aggiornamenti delle dipendenze frontend e Python che il progetto classifica come aggiornamenti di sicurezza. La release non cita un avviso di sicurezza specifico con CVE o GHSA. Si rivolge a tutti gli operatori di Paperless-ngx 3.x, in particolare alle installazioni con accesso OIDC e alle installazioni bare metal i cui pacchetti di sistema vengono gestiti autonomamente.

## Panoramica delle versioni

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Release di funzionalità | saltare la 2FA locale con AMR OIDC attendibile, miniature PDF tramite Poppler, ImageMagick rimosso, campi personalizzati nei titoli dei workflow, aggiornamenti di sicurezza delle dipendenze |

## Accesso OIDC: 2FA locale con AMR attendibile

Durante l’accesso tramite un provider OIDC, Paperless-ngx ora valuta il claim `amr` del token ID (PR #14403). Se contiene uno dei valori di `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, Paperless-ngx salta la propria richiesta di autenticazione a due fattori per quell’accesso. Secondo la documentazione di configurazione, il valore predefinito della variabile è `mfa`. La documentazione cita come esempi Authelia, che segnala `mfa`, e Pocket ID, che segnala `phr` per le passkey. Per i codici monouso via e-mail, Pocket ID invia il valore `otp`; la documentazione lo considera non attendibile.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Poiché il valore predefinito è già impostato, dopo l’aggiornamento il comportamento cambia per gli utenti con 2FA configurata localmente non appena il loro Identity Provider fornisce `mfa` nel claim `amr`. Verificate quindi quali valori AMR invia effettivamente il vostro provider e se tali valori vengono impostati soltanto con un vero secondo fattore. La documentazione richiede esplicitamente di inserire solo valori che indicano che durante l’accesso è stato effettivamente utilizzato un secondo fattore. Parallelamente, la 3.4.0 aggiorna django-allauth alla 65.20 e oauthlib alla 4.0 (PR #14570).

## Miniature PDF con Poppler, ImageMagick rimosso

Finora, per la miniatura di un PDF Paperless-ngx richiamava ImageMagick (`convert -density 300`) per renderizzare la prima pagina tramite Ghostscript a 300 DPI. Dalla 3.4.0, `pdftoppm` degli strumenti Poppler renderizza la prima pagina, Pillow ridimensiona l’immagine e la salva come WebP (PR #14420). Paperless-ngx calcola la risoluzione in base alle dimensioni della pagina, rilevate tramite `pdfinfo`, fino a un massimo di 300 DPI. Il fallback Ghostscript per le miniature viene eliminato, mentre la riparazione tramite qpdf rimane. L’autore della PR ha misurato tempi di esecuzione molto più brevi e un minore utilizzo di memoria: per un documento A0 vuoto, circa 0,21 anziché 12,9 secondi; i valori non sono stati verificati indipendentemente.

Con la PR #14440, ImageMagick viene eliminato completamente. Le attività rimanenti, principalmente la rimozione di un canale alfa dalle immagini, sono gestite da un helper basato su Pillow; i file WebP vengono salvati senza perdita di dati. La policy di ImageMagick che finora doveva essere adattata per l’elaborazione PDF non è quindi più necessaria. Ghostscript rimane installato perché OCRmyPDF continua a richiederlo.

Per gli amministratori, questo comporta due conseguenze:

- **Bare metal:** `pdftoppm` deve essere disponibile. Se manca, Paperless-ngx emette un avviso all’avvio. Confrontate i pacchetti di sistema installati con l’elenco aggiornato delle dipendenze nella documentazione di installazione.
- **Configurazione:** `PAPERLESS_CONVERT_MEMORY_LIMIT` e `PAPERLESS_CONVERT_TMPDIR` sono, secondo la PR #14420, deprecati e privi di effetto; anche `PAPERLESS_GS_BINARY` è stato contrassegnato come deprecato. Potete rimuovere queste voci da `paperless.conf` o dal file `.env`.

## Workflow, campi personalizzati e interfaccia web

- **Workflow:** I modelli di titolo nei workflow possono utilizzare campi personalizzati (PR #14564). Nei campi tag dei workflow è possibile selezionare tag superiori e subordinati indipendentemente l’uno dall’altro (PR #14400).
- **Campi personalizzati:** Le query sui campi data possono essere relative (PR #14549).
- **Caricamento e notifiche:** La barra di navigazione dispone di un pulsante di caricamento (PR #14442). Il pulsante mostra il numero di caricamenti in corso e rimanda alla panoramica; il numero delle notifiche diminuisce (PR #14501, #14500).
- **Interfaccia:** I dettagli del documento sono stati rielaborati (PR #14485), il badge delle attività si carica più rapidamente (PR #14552) e la chat IA visualizza Markdown (PR #14409, #14415).

## Correzioni di bug

- Il `document_exporter` si interrompeva durante l’esportazione in una cartella con `--delete` se era presente un `manifest.json.tmp` obsoleto (PR #14526). Chi utilizza l’esportatore con `--delete` per i backup dovrebbe verificare dopo l’aggiornamento che le esportazioni vengano nuovamente completate.
- Durante la modifica di un documento, l’indice di ricerca viene aggiornato solo dopo l’esecuzione dei workflow (PR #14558).
- Se un codice a barre non determina una separazione dei documenti, i tag non vengono più assegnati in ritardo (PR #14497).
- Le attività annullate o scadute vengono visualizzate in modo più comprensibile (PR #14566).
- La vista dello schema dell’API non attiva più controlli di autorizzazione sui documenti (PR #14503), mentre il tipo elenco dei filtri per gli oggetti documento nello schema API è stato corretto (PR #14524).
- Le denominazioni di versione lunghe vengono abbreviate nel menu a discesa e le dimensioni dei menu a discesa sono state uniformate (PR #14578, #14510).

## Dipendenze e sicurezza

Le note di rilascio elencano gli aggiornamenti delle dipendenze frontend (PR #14581), nonché di brotli, djangorestframework, filelock, msgpack e setuptools (PR #14575), come aggiornamenti di sicurezza. Le note di rilascio non indicano quali vulnerabilità vengano chiuse in questi pacchetti. I componenti Angular sono alla versione 22.2.2. Non esiste un Security Advisory specifico del progetto per la 3.4.0 (situazione al 30 ottobre 2026).

L’advisory più recente, GHSA-c8f3-pqr2-rr24 del 16 ottobre 2026, riguarda le versioni dalla 3.0 fino a prima della 3.3.2 ed è risolto nella 3.3.2. Con le funzionalità IA attivate, i workflow di classificazione IA per documenti senza proprietario potevano trasmettere estratti e metadati di documenti privati di altri utenti al fornitore IA configurato (CVSS 3.1: 4.3, nessuna CVE). Anche i sistemi ancora in esecuzione su una versione 3.x precedente chiudono questa lacuna aggiornando alla 3.4.0.

## Procedura consigliata

La versione di destinazione è la 3.4.0. Prima dell’aggiornamento, interrompete le elaborazioni in corso, come l’importazione dei documenti, e create un backup come richiesto dalla documentazione di amministrazione. Nelle installazioni Docker con immagine da Docker Hub, arrestate lo stack con `docker compose down`, scaricate la nuova immagine con `docker compose pull` e avviate con `docker compose up`, durante il quale le migrazioni del database vengono eseguite automaticamente; se tutto funziona, avviate lo stack in background con `-d`. Nelle installazioni bare metal, installate prima le nuove dipendenze di sistema, in particolare Poppler con `pdftoppm`, aggiornate i pacchetti Python con `pip install -r requirements.txt` ed eseguite le migrazioni nella directory `src` con `python3 manage.py migrate`. Prima del primo accesso OIDC dopo l’aggiornamento, stabilite quali valori debbano figurare in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. L’articolo [Eseguire Paperless-ngx con poco spazio di archiviazione](/blog/paperless-dokumente-clouddienst-auslagern) descrive un’installazione Docker con archiviazione dei documenti esternalizzata. Le note di rilascio della 3.4.0 non menzionano problemi noti o modifiche incompatibili (situazione al 30 ottobre 2026).

## Fonti

1.  [GitHub: Paperless-ngx Release v3.4.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): note di rilascio con data, nuove funzionalità, correzioni di bug e dipendenze.

2.  [Paperless-ngx: Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): changelog nella documentazione del progetto con le voci relative alla 3.3.x e alla 3.4.0.

3.  [GitHub: PR #14403, skip local 2FA for trusted OIDC AMR](https://github.com/paperless-ngx/paperless-ngx/pull/14403): nuova valutazione del claim AMR durante l’accesso OIDC.

4.  [Paperless-ngx: documentazione di configurazione](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): descrizione e valore predefinito di `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, miniature PDF tramite Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): passaggio a `pdftoppm`, impostazioni deprecate e valori misurati dall’autore.

6.  [GitHub: PR #14440, ImageMagick rimosso come dipendenza](https://github.com/paperless-ngx/paperless-ngx/pull/14440): sostituzione delle restanti chiamate a ImageMagick con Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): divulgazione di informazioni nei workflow di classificazione IA, interessate dalla 3.0 fino a prima della 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: documentazione di amministrazione](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): procedura di aggiornamento per Docker e bare metal, opzione dell’esportatore `--delete`.

9.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): date di pubblicazione della 3.3.4 e della 3.4.0.
