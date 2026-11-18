---
title: "Paperless-ngx 3.4.1, 3.4.2 e 3.5.0: ALLOWED_HOSTS più restrittivo, OIDC-2FA corretto, operazioni bulk asincrone"
navTitle: "Paperless-ngx 3.5.0"
description: "Paperless-ngx 3.5.0, quando PAPERLESS_URL è impostato, consente solo il relativo hostname e localhost, corregge il valore predefinito AMR inefficace della versione 3.4.x ed esegue le operazioni bulk in modo asincrono. Prima dell'aggiornamento, verificate tramite quali hostname si accede a Paperless-ngx e aggiungeteli a PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min di lettura"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-e-3-5-0-allowed-hosts-piu-restrittivo-oidc-2fa-corretto-operazioni"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/it/blog/paperless-ngx-3-4-1-3-4-2-e-3-5-0-allowed-hosts-piu-restrittivo-oidc-2fa-corretto-operazioni
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:22:12.356Z
translationReview: automatic
---

# Paperless-ngx 3.4.1, 3.4.2 e 3.5.0: ALLOWED_HOSTS più restrittivo, OIDC-2FA corretto, operazioni bulk asincrone

Dalla versione 3.4.0, il progetto Paperless-ngx ha pubblicato tre versioni: le release correttive 3.4.1 il 31 ottobre 2026 e 3.4.2 il 3 novembre 2026, nonché la release funzionale 3.5.0 il 17 novembre 2026. Le due versioni patch correggono soprattutto errori di visualizzazione dell'interfaccia web rielaborata e aggiornano Django e altre dipendenze. La 3.5.0 modifica il valore predefinito per gli hostname consentiti, risolve un errore per cui l'impostazione `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` introdotta nella 3.4.0 rimaneva inefficace, esegue tutte le operazioni bulk in modo asincrono, elimina automaticamente le sessioni scadute e introduce workflow attivabili manualmente. Nessuna delle tre release menziona un avviso di sicurezza con CVE o GHSA. Lo stato precedente è descritto nell'articolo [Paperless-ngx 3.4.0: login OIDC senza doppia 2FA, miniature senza ImageMagick](/blog/paperless-ngx-3-4-0). Gli aggiornamenti sono rivolti a tutti gli amministratori di Paperless-ngx 3.x; sono particolarmente interessate le installazioni raggiunte tramite hostname interni e quelle con autenticazione OIDC.

## Panoramica delle versioni

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Release correttiva | Correzioni allo schema API e all'interfaccia web, minore consumo di memoria in inattività |
| 3.4.2 | 03.11.2026 | Release correttiva | Task di modifica bulk nei worker Celery, installazione delle lingue Tesseract prima della creazione dell'indice, filelock 4.2.6, aggiornamenti di Django e delle dipendenze frontend |
| 3.5.0 | 17.11.2026 | Release funzionale | ALLOWED_HOSTS senza wildcard con PAPERLESS_URL impostato, valore predefinito AMR corretto, operazioni bulk asincrone, pulizia automatica delle sessioni, trigger manuale dei workflow, gerarchia dei tag senza django-treenode |

## 3.4.1 e 3.4.2: correzioni dopo la release funzionale

La 3.4.1 corregge il posizionamento di una correzione dello schema per i campi `id` (PR #14604), rimuove i controlli dei permessi duplicati nel pulsante di apertura delle schede dei documenti (PR #14601), nasconde in gran parte i nuovi controlli PDF (PR #14595), sposta il pulsante di anteprima nell'elenco dei documenti (PR #14594) e corregge la visualizzazione del checksum in modalità chiara nonché la posizione del menu a discesa dei campi personalizzati sui dispositivi mobili (PR #14590). Inoltre, Paperless-ngx carica più moduli solo quando necessario, riducendo il consumo di memoria in inattività (PR #14599).

La 3.4.2 contiene undici correzioni di errori. Rilevanti per l'esercizio sono:

- Le attività di importazione da `bulk_edit` vengono eseguite in modo affidabile nei worker Celery (PR #14653).
- La creazione dell'indice di ricerca e dell'indice LLM attende finché non sono installati i pacchetti linguistici di Tesseract (PR #14670).
- filelock passa alla versione 4.2.6; le note di rilascio motivano l'aggiornamento con correzioni relative alle ACL (PR #14656).
- La cronologia dei documenti mostra i nomi dei tag anziché gli ID (PR #14625), i pulsanti di zoom non inviano più i moduli (PR #14624) e il nome di una vista salvata viene mantenuto durante il salvataggio e la chiusura (PR #14667).

Tra le dipendenze, la 3.4.2 aggiorna Django, django-guardian, django-filter e drf-spectacular-sidecar (PR #14661), turbohtml alla versione 1.15.3 (PR #14662) e pdfjs-dist alla versione 6.4.299 (PR #14644). Il progetto sostituisce la libreria ngx-bootstrap-icons, non più mantenuta, con una soluzione propria (PR #14665). Nei file Compose di esempio, Gotenberg passa alla versione 8.38 (PR #14634) e nginx alla 1.31.7 (PR #14633). Le note di rilascio della 3.4.1 e della 3.4.2 non classificano nessuna di queste voci come aggiornamento di sicurezza.

## 3.5.0: ALLOWED_HOSTS senza wildcard

Le note di rilascio della 3.5.0 iniziano con l'avviso che alcuni utenti devono adattare la propria impostazione `PAPERLESS_ALLOWED_HOSTS` (PR #14735). Fino alla 3.4.2, il progetto documentava per `PAPERLESS_ALLOWED_HOSTS` il valore predefinito `*`, ossia tutti gli host. Dalla 3.5.0, secondo la documentazione della configurazione:

- Se non sono impostati né `PAPERLESS_URL` né `PAPERLESS_ALLOWED_HOSTS`, Paperless-ngx continua ad accettare qualsiasi host.
- Se è impostato solo `PAPERLESS_URL`, Paperless-ngx accetta soltanto l'hostname di questo URL e `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` aggiunge altri host a `localhost` e all'hostname di `PAPERLESS_URL`. `localhost` rimane sempre consentito perché necessario al controllo di integrità Docker.
- `PAPERLESS_ALLOWED_HOSTS=*` consente qualsiasi host, anche se `PAPERLESS_URL` è impostato.

Sono interessate le installazioni con `PAPERLESS_URL` impostato che vengono raggiunte anche tramite altri nomi, ad esempio un hostname interno, un indirizzo IP o il nome del servizio in una rete di container attraverso cui altri servizi richiamano l'API. Dopo l'aggiornamento Django rifiuta tali richieste. Prima dell'aggiornamento, inserite tutti i nomi aggiuntivi in `PAPERLESS_ALLOWED_HOSTS`, ad esempio:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Dovreste impostare il valore wildcard `*` solo se Paperless-ngx è raggiungibile esclusivamente in una rete isolata; per le installazioni su Internet aperto, la documentazione consiglia esplicitamente di definire gli host consentiti.

## 3.5.0: autenticazione OIDC e sessioni

Con la 3.4.0 è stata introdotta l'impostazione `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, con cui Paperless-ngx salta la richiesta del secondo fattore locale quando il provider OIDC segnala un secondo fattore affidabile nel claim `amr`. Secondo l'issue #14692, nella serie 3.4.x l'elenco dei valori affidabili rimaneva sempre vuoto: il codice passava `mfa` come separatore anziché come valore predefinito. Senza una variabile impostata e anche con `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa`, Paperless-ngx continuava quindi a richiedere il secondo fattore locale. La PR #14693 risolve il problema nella 3.5.0. Il valore predefinito documentato è `mfa`; gli accessi con password continuano a richiedere il codice di Paperless-ngx.

Di conseguenza, il cambiamento di comportamento descritto nell'articolo sulla 3.4.0 si verifica effettivamente solo con la 3.5.0: gli utenti con 2FA configurata localmente, il cui Identity Provider fornisce `mfa` nel claim `amr`, dopo l'aggiornamento non vedono più una seconda richiesta. Prima dell'aggiornamento, verificate quali valori AMR invia il vostro provider. La documentazione richiede di inserire soltanto valori che indicano che durante l'accesso è stato effettivamente utilizzato un secondo fattore. Allo stesso tempo, django-allauth passa alla versione 65.21.1 (PR #14672).

Finora Paperless-ngx non rimuoveva mai dal database le sessioni scadute. Il motivo della modifica è stato l'issue #14721: con `PAPERLESS_AUTO_LOGIN_USERNAME` veniva creata ogni 30 secondi una nuova sessione, che non veniva mai eliminata. Dalla 3.5.0, un'attività pianificata rimuove le sessioni scadute (PR #14722). L'orario è controllato da `PAPERLESS_SESSION_CLEANUP_TASK_CRON`; il valore predefinito è `0 3 * * *`, ovvero ogni giorno alle 03:00.

## 3.5.0: operazioni bulk, workflow e tag

- **Operazioni bulk asincrone:** tutte le operazioni bulk sui documenti e la modifica bulk dei metadati vengono eseguite tramite la coda delle attività (PR #14773, #14711) e vengono visualizzate nell'elenco delle attività. Secondo la documentazione API, le operazioni bulk accodate rispondono con `{"result":"OK","task_id":"…"}`; potete richiedere l'esito tramite `/api/tasks/?task_id=<uuid>`. Una volta conclusa l'attività bulk, le attività successive, come la rielaborazione dei file, sono solo pianificate e non ancora completate. Adattate di conseguenza gli script che elaborano il risultato immediatamente dopo la chiamata.
- **Trigger manuale dei workflow:** con il nuovo tipo di trigger, i workflow possono essere applicati manualmente ai documenti selezionati, nella modifica bulk e nei dettagli del documento (PR #14704).
- **Gerarchia dei tag:** Paperless-ngx salva ora la gerarchia dei tag come lista di adiacenza e rinuncia a django-treenode (PR #14612). L'aggiornamento include una migrazione del database a tale scopo. In un prototipo con 6000 tag su SQLite, l'autore misura tempi nettamente inferiori, circa 1,3 anziché 4409 millisecondi per la creazione di un tag; i valori non sono stati verificati in modo indipendente. Quando elimina un tag, Paperless-ngx verifica inoltre i permessi sui tag subordinati (PR #14687).
- **Altre novità:** le note sui documenti vengono visualizzate come Markdown (PR #14626). Un modello di embedding Hugging Face memorizzato localmente viene caricato senza connessione all'Hub (PR #14728) e la chat IA filtra anche oltre 32 700 documenti (PR #14754).

## 3.5.0: ulteriori correzioni di errori

- **Argomenti OCR:** l'API rifiuta gli argomenti che Paperless-ngx stesso passa a OCRmyPDF (`input_file_or_options`, `output_file` e `sidecar`); in `PAPERLESS_OCR_USER_ARGS` vengono ignorati (PR #14734). Verificate se la vostra configurazione contiene questi argomenti.
- **Workflow:** il filtro del nome file dei workflow pianificati utilizza un'espressione regolare compatibile con PostgreSQL (PR #14783).
- **Posta:** le e-mail salvate vengono scritte con terminazioni di riga CRLF (PR #14745) e la conversione PDF/A degli archivi e-mail viene eseguita una sola volta (PR #14698).
- **Miniature:** per le immagini senza versione archiviata, Paperless-ngx genera nuovamente miniature (PR #14741).
- **Permessi e account:** i permessi della vista di configurazione sono stati uniformati (PR #14709), durante la registrazione del primo utente Paperless-ngx blocca le righe del database interessate (PR #14688) e le password vengono visualizzate con una maschera costante (PR #14784). L'autore della PR #14709 non classifica esplicitamente la modifica come vulnerabilità.

## Dipendenze e sicurezza

Non esiste alcun Security Advisory del progetto relativo alla 3.4.1, alla 3.4.2 o alla 3.5.0 (stato al 17 novembre 2026). L'Advisory più recente, GHSA-c8f3-pqr2-rr24 del 16 ottobre 2026, riguarda i workflow di classificazione IA su documenti senza proprietario ed è risolto dalla 3.3.2; i dettagli sono disponibili nell'articolo sulla 3.4.0. Anche i sistemi su una versione 3.x precedente chiudono questa lacuna aggiornando alla 3.5.0.

## Problemi noti

- **2FA locale con OIDC nonostante AMR `mfa` (dalla 3.4.0 alla 3.4.2):** risolto nella 3.5.0 (issue #14692, PR #14693).
- **Problemi noti della 3.5.0:** le note di rilascio non ne indicano alcuno (stato al 17 novembre 2026). L'avviso relativo a `PAPERLESS_ALLOWED_HOSTS` è un cambiamento di comportamento intenzionale.

## Procedura consigliata

La versione di destinazione è la 3.5.0. Prima dell'aggiornamento, definite tramite quali hostname viene raggiunto Paperless-ngx e aggiungeteli a `PAPERLESS_ALLOWED_HOSTS`; verificate inoltre i valori in `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Interrompete le elaborazioni in corso, come l'importazione di documenti, e create un backup, come richiesto dalla documentazione amministrativa. Nelle installazioni Docker con immagine da Docker Hub, fermate lo stack con `docker compose down`, scaricate la nuova immagine con `docker compose pull` e avviate con `docker compose up`; le migrazioni del database, inclusa la nuova migrazione della gerarchia dei tag, vengono eseguite automaticamente. Nelle installazioni bare metal, aggiornate i pacchetti Python con `pip install -r requirements.txt` ed eseguite le migrazioni nella directory `src` con `python3 manage.py migrate`; successivamente, `python3 manage.py document_index reindex --if-needed` e `python3 manage.py document_llmindex migrate` aggiornano, se necessario, l'indice di ricerca e l'indice LLM. Per un aggiornamento diretto dalla 3.3.x valgono inoltre le indicazioni dell'articolo sulla 3.4.0, in particolare quelle relative a `pdftoppm`. Un'installazione Docker con archivio documenti esterno è descritta nell'articolo [Gestire Paperless-ngx con poco spazio di archiviazione](/blog/paperless-dokumente-clouddienst-auslagern).

## Fonti

1.  [GitHub: Paperless-ngx Release v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): note di rilascio con avviso su `PAPERLESS_ALLOWED_HOSTS`, nuove funzionalità e correzioni di errori.

2.  [GitHub: Paperless-ngx Release v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): correzioni di errori e dipendenze aggiornate.

3.  [GitHub: Paperless-ngx Release v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): correzioni di errori e minore consumo di memoria in inattività.

4.  [GitHub API: Paperless-ngx Releases](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): date di pubblicazione della 3.4.1, della 3.4.2 e della 3.5.0.

5.  [Paperless-ngx: documentazione della configurazione 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): nuovo comportamento di `PAPERLESS_ALLOWED_HOSTS`, valore predefinito di `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` e argomenti OCR ignorati.

6.  [Paperless-ngx: documentazione della configurazione 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): precedente valore predefinito `*` per `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: issue #14692, MFA trusted AMR sempre vuoto](https://github.com/paperless-ngx/paperless-ngx/issues/14692): causa e sintomi del valore predefinito AMR inefficace nella 3.4.x.

8.  [GitHub: PR #14722, pulizia automatica delle sessioni](https://github.com/paperless-ngx/paperless-ngx/pull/14722): contesto sulle sessioni mai eliminate con l'accesso automatico.

9.  [GitHub: PR #14612, gerarchia dei tag come lista di adiacenza](https://github.com/paperless-ngx/paperless-ngx/pull/14612): sostituzione di django-treenode, migrazione e valori misurati dall'autore.

10.  [Paperless-ngx: documentazione API 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): operazioni bulk asincrone e interrogazione tramite `task_id`.

11.  [Paperless-ngx: documentazione amministrativa 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): procedura di aggiornamento per Docker e bare metal.

12.  [GitHub: Paperless-ngx Security Advisories](https://github.com/paperless-ngx/paperless-ngx/security/advisories): panoramica degli advisory, nessuno relativo dalla 3.4.1 alla 3.5.0.
