---
title: "Paperless-ngx: pipeline documentale, OCR e gestione dell’archivio"
blatt: "paperless-ngx"
description: "Paperless-ngx per amministratori DMS, di piattaforma e messaggistica: architettura Django/Angular, pipeline Consumer e OCR, file originali e di archivio, metadati, ricerca Tantivy, Celery e Redis, PostgreSQL, Tika/Gotenberg, autorizzazioni, recupero e-mail, API, esercizio, backup, ripristino e storia tecnica."
fakten:
  - label: Ruolo del sistema
    wert: sistema di gestione documentale self-hosted per acquisizione, OCR, classificazione, ricerca, conservazione ed esportazione
    href: https://docs.paperless-ngx.com/
  - label: Stack tecnologico
    wert: Python e Django nel backend, Angular nel browser, task Celery, OCRmyPDF/Tesseract e indice di ricerca Tantivy
    href: https://github.com/paperless-ngx/paperless-ngx
  - label: Servizi obbligatori
    wert: processi web/task, broker compatibile con Redis, database e directory Data/Media persistenti
    href: https://docs.paperless-ngx.com/configuration/
  - label: Database
    wert: PostgreSQL consigliato per le nuove installazioni; sono supportati anche MariaDB e SQLite
    href: https://docs.paperless-ngx.com/setup/
  - label: Oggetto documento
    wert: record del database più originale, PDF di archivio opzionale, miniatura, indice full-text e checksum
    href: https://docs.paperless-ngx.com/faq/
  - label: Ingressi
    wert: directory Consume, upload da browser/API e recupero e-mail IMAP basato su regole
    href: https://docs.paperless-ngx.com/usage/
  - label: OCR
    wert: OCRmyPDF orchestra rasterizzazione e Tesseract; modalità, lingua, limite di pagine e formato di archivio sono parametri operativi
    href: https://docs.paperless-ngx.com/configuration/
  - label: Office ed e-mail
    wert: conversione opzionale tramite Apache Tika e Gotenberg; i due servizi costituiscono un proprio ambito di guasto
    href: https://docs.paperless-ngx.com/configuration/#optional-services
  - label: Ricerca
    wert: indice Tantivy accanto al database relazionale; l’indice è derivabile, database e originali sono autorevoli
    href: https://docs.paperless-ngx.com/migration-v3/
  - label: Autorizzazioni
    wert: permessi Django globali più permessi Owner, View e Change a livello di oggetto per utenti e gruppi
    href: https://docs.paperless-ngx.com/usage/#permissions
  - label: Nucleo del backup
    wert: esportazione coerente oppure backup coordinato di database, Media, Data, configurazione e segreti
    href: https://docs.paperless-ngx.com/administration/#making-backups
  - label: Regola di accettazione
    wert: ripristino in un’istanza vuota e compatibile; devono funzionare accesso, numero di documenti, campioni, ricerca, checksum e una nuova esecuzione Consume
    href: https://docs.paperless-ngx.com/administration/#restoring
werbung:
  - newsletter
ctaThemen:
  - paperless-ngx
translationSourceHash: a172bf8c7ed80cecd7c990d657cfdf7d06afe1c0b349350655549c21ed144dfc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:33:19.037Z
translationReview: automatic
---

# Paperless-ngx: pipeline documentale, OCR e gestione dell’archivio

Paperless-ngx è un sistema di gestione documentale self-hosted per file in entrata e documenti cartacei scansionati. Riceve documenti da una directory monitorata, dal Web o dalla REST API oppure da caselle e-mail, genera testo ricercabile e, se necessario, una copia d’archivio normalizzata, assegna metadati e rende il patrimonio ricercabile tramite interfaccia Web e API. La [panoramica del progetto](https://docs.paperless-ngx.com/) definisce il servizio un archivio online ricercabile; il [repository sorgente](https://github.com/paperless-ngx/paperless-ngx) lo identifica al contempo come successore community di Paperless e Paperless-ng.

Per gli amministratori, Paperless-ngx non è una singola applicazione Web, bensì un insieme di database, broker, processi Web e in background, patrimonio di file, indice di ricerca e convertitori esterni. Uno stato HTTP positivo dimostra soltanto che il processo Web risponde. Non indica se il Consumer rileva nuovi file, se i task Celery vengono consegnati, se l’OCR funziona, se vengono scritti file d’archivio, se l’indice è aggiornato o se un’esportazione può essere reimportata.

La spiegazione accompagna un documento dall’ingresso, attraverso Consumer, parser e OCR, fino a database, file d’archivio e indice di ricerca. Seguono quindi automazione, sicurezza, esercizio e un ripristino che verifica insieme tutte le parti.

Paperless-ngx elabora un documento come pipeline: rilevare l’ingresso, acquisire il file, generare OCR e metadati, aggiornare database e indice di ricerca e fornire il risultato tramite Web o API. L’articolo segue questo percorso e ne deduce esercizio, backup e diagnosi.

## Architettura: percorso di controllo, di lavoro e dei dati

L’architettura può essere suddivisa in tre percorsi:

1. **Percorso di controllo:** browser o client API comunicano tramite HTTP o HTTPS con l’applicazione Django. Il frontend Angular è il client interattivo; Django fornisce autenticazione, logica aziendale, interfaccia di amministrazione e REST API. La [documentazione di sviluppo](https://docs.paperless-ngx.com/development/) documenta che il progetto è costituito da un’applicazione Django e da un frontend Angular.
2. **Percorso di lavoro:** Consumer, Scheduler e Worker generano ed elaborano task asincroni. Un broker compatibile con Redis media tra i processi; Celery esegue, tra l’altro, acquisizione, recupero e-mail, manutenzione dell’indice e manutenzione ricorrente. La [documentazione d’uso](https://docs.paperless-ngx.com/usage/) descrive Consumer, Task Processor e broker come ruoli separati.
3. **Percorso dei dati:** il database relazionale conserva oggetti documento, metadati, utenti, regole e stato dei task. Originali, versioni d’archivio e miniature sono nella directory Media; stati ausiliari e dati di ricerca sono nella directory Data. L’indice di ricerca è una proiezione di questo patrimonio autorevole.

La separazione è operativamente decisiva. Database e directory Media devono appartenere allo stesso momento logico. È possibile ricostruire un nuovo indice, ma non un file originale perso a partire dall’indice. Analogamente, un backup del database senza i file associati non è sufficiente per un ripristino DMS.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-paperless-ngx.svg?v=20260813" title="Interaktive Infografik: Paperless-ngx von Eingang, Consumer und Task-Broker über Parser, OCR, Archivierung, Metadaten und Suche bis zu Datenbank, Dateien, Backup und Restore" loading="lazy">
  <a href="/images/kb-interaktiv-paperless-ngx.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Stack tecnologico e ruoli dei processi

Il codice backend è implementato prevalentemente in Python e si basa su Django. Il frontend per browser utilizza Angular. Le dipendenze quali Django, [Celery](https://docs.celeryq.dev/en/stable/), OCRmyPDF e altri parser sono consultabili nel [manifest del progetto Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml); il [Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile) mostra la costruzione dell’immagine e le dipendenze OCR/PDF native. Per l’esercizio sono rilevanti i seguenti ruoli:

| Ruolo | Compito principale | Fonte dati determinante | Guasto tipico |
|---|---|---|---|
| Server Web | UI, REST API, autenticazione, upload | Sessione e transazione DB | 4xx/5xx, errori proxy o CSRF |
| Consumer | rilevare nuovi file e avviare il task | directory Consume e task | il file rimane, polling/inotify o permessi |
| Task Worker | parser, OCR, classificazione e passaggi di archiviazione | stato task, file temporanei, DB | task pending/failed, limite memoria/CPU |
| Scheduler | recupero e-mail e manutenzione periodica | pianificazione nel sistema task | nessun job periodico |
| Broker | consegna di task asincroni | dati volatili di coda/broker | il Web funziona, l’elaborazione è ferma |
| Database | documenti, metadati, identità, regole | patrimonio primario relazionale | errori di transazione, schema o capacità |
| Archiviazione file | originale, archivio, miniatura | file Media | file mancante, sola lettura, mount incoerente |
| Indice di ricerca | proiezione full-text | dati indice derivabili | documento presente, ricerca incompleta |
| Tika/Gotenberg | conversione opzionale Office/e-mail | convertito temporaneo | falliscono solo determinati tipi di file |

La [riferimento alla configurazione](https://docs.paperless-ngx.com/configuration/) indica il [broker compatibile con Redis](https://redis.io/docs/latest/) come servizio obbligatorio. [PostgreSQL](https://www.postgresql.org/docs/current/), MariaDB e SQLite sono supportati come database; per le nuove installazioni la [guida di installazione](https://docs.paperless-ngx.com/setup/) raccomanda PostgreSQL. Tika e Gotenberg sono opzionali e sono entrambi necessari per formati Office e l’elaborazione di file e-mail ([servizi opzionali](https://docs.paperless-ngx.com/configuration/#optional-services)).

## Inventario dei servizi e configurazione effettiva

Un inventario affidabile include immagine o release, tipo di database, destinazione del broker, mount, variabili d’ambiente effettive, UID/GID, reverse proxy, fuso orario, lingue OCR e parser esterni. Docker Compose è un packaging diffuso, ma non costituisce una prova dell’architettura: più ruoli di processo possono essere contenuti in un’immagine di container, mentre i loro stati risiedono in volumi e servizi separati.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose config --services
docker compose ps
docker compose images
docker compose config
docker inspect paperless-webserver</code></pre>
  </div>
</div>

[`docker compose config`](https://docs.docker.com/reference/cli/docker/compose/config/) rende la configurazione Compose effettivamente unita; [`docker compose ps`](https://docs.docker.com/reference/cli/docker/compose/ps/) mostra processi e porte pubblicate; [`docker compose images`](https://docs.docker.com/reference/cli/docker/compose/images/) identifica le immagini; [`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/) fornisce mount, rete e parametri di runtime. Gli output possono contenere segreti e vanno inseriti nei ticket solo dopo redazione.

## Ingressi e confini di consegna

Paperless-ngx conosce più ingressi che, prima della pipeline comune, hanno differenti ambiti di guasto:

- La **directory Consume** viene monitorata tramite eventi del filesystem. Su filesystem senza inotify affidabile, come alcuni mount NFS, occorre attivare il polling; lo indica la [documentazione di configurazione](https://docs.paperless-ngx.com/setup/#file-systems-without-inotify-support-eg-nfs).
- **Browser e REST API** scrivono gli upload attraverso il percorso Web. Reverse proxy, limite di dimensione della richiesta, timeout e spazio temporaneo si trovano quindi prima del sistema task.
- Il **recupero e-mail** legge da account configurati in base a regole e genera task per gli allegati corrispondenti. Connessione IMAP, TLS, OAuth/password, cartella, criterio di ricerca e azione di post-elaborazione fanno parte del contratto di consegna.

La directory Consume è una zona di trasferimento e non un archivio documentale. I file acquisiti con successo vengono rimossi da lì, mentre Paperless genera nuovi file gestiti nel patrimonio Media. Le [FAQ](https://docs.paperless-ngx.com/faq/) avvertono esplicitamente di non gestire manualmente la directory Media.

Una volta noti processi e ingressi, inizia l’elaborazione vera e propria. Ogni passaggio genera file, record di database e segnali di errore diversi.

## Pipeline Consume: dall’ingresso all’oggetto documento

Il commit funzionale non avviene al rilevamento di un file, ma solo dopo più passaggi. A seconda del tipo di file e della configurazione, la pipeline comprende tipicamente:

1. **Rilevamento e stabilizzazione:** vengono verificati nome file, tipo consentito, dimensione del file e completamento della scrittura.
2. **Generazione del task:** il Consumer o il server Web trasferisce un task tramite il broker.
3. **Parsing e pre-elaborazione:** i parser estraggono il testo esistente o convertono il documento in ingresso. File Office ed e-mail possono richiedere Tika e Gotenberg.
4. **OCR:** OCRmyPDF decide il riconoscimento testuale in base alla modalità OCR; Tesseract riconosce le lingue configurate.
5. **Versione d’archivio e miniatura:** vengono generate una visualizzazione/file d’archivio e dati di anteprima, nella misura prevista da tipo di file e impostazioni.
6. **Persistenza:** originale e derivati vengono scritti; vengono salvati oggetto database, checksum, testo e proprietà tecniche.
7. **Metadati e workflow:** le regole assegnano tag, corrispondente, tipo di documento, percorso di archiviazione, proprietario o altri campi.
8. **Indicizzazione:** il patrimonio full-text viene aggiornato nell’indice di ricerca.
9. **Conclusione:** stato del task e log mostrano successo o errore; solo ora il documento è funzionalmente reperibile.

Gli script pre- e post-consume possono estendere il percorso. La [documentazione sugli hook Consume](https://docs.paperless-ngx.com/advanced_usage/#consume-hooks) chiarisce che tali script vengono eseguiti nel contesto di sicurezza e di errore del processo Paperless. Devono essere versionati, eseguiti con timeout e codici di uscita univoci; un hook difettoso può bloccare l’intera acquisizione.

## OCR, formato di archivio e limiti qualitativi

OCRmyPDF orchestra l’elaborazione PDF e l’OCR Tesseract. La modalità decide se il testo esistente viene saltato, rielaborato o forzato. Lingua, limite di pagine, pulizia dell’immagine, correzione della rotazione e formato di output influenzano tempo CPU, dimensione del file e qualità di riconoscimento. Le impostazioni sono descritte nella [configurazione OCR di Paperless](https://docs.paperless-ngx.com/configuration/#ocr-settings) e nella [documentazione OCRmyPDF](https://ocrmypdf.readthedocs.io/en/latest/); i modelli linguistici e il comportamento del motore sono documentati da [Tesseract](https://tesseract-ocr.github.io/tessdoc/).

L’originale rimane conservato come oggetto probatorio. Un PDF d’archivio è una versione derivata, rigenerabile tramite successive modifiche OCR/di archiviazione. Questa distinzione deve rientrare nei concetti di conservazione e valore probatorio:

| Oggetto | Scopo | Ripristinabilità |
|---|---|---|
| Originale | file di ingresso invariato | non ricostruibile dal testo OCR |
| File d’archivio | versione normalizzata, spesso ricercabile | rigenerabile dall’originale e dalla pipeline adatta |
| Miniatura | anteprima UI | derivabile |
| Testo estratto | ricerca e classificazione | riestraibile, il risultato può differire in base alla versione |
| Checksum | confronto di integrità | probatorio solo insieme ad algoritmo e riferimento |

Il successo dell’OCR non significa leggibilità funzionale. I controlli a campione devono confrontare numero di pagine, orientamento, copertura del testo, caratteri speciali, tabelle, firme incorporate e dimensione del file. I picchi di CPU e RAM si verificano spesso durante il rendering di PDF grandi o ricchi di immagini, non nella scrittura finale nel database.

## Originali, archivi, miniature e formato dei percorsi

Un documento Paperless è composto da più parti collegate. Il database conosce ID interno, titolo, data di creazione e di aggiunta, corrispondente, tipo di documento, tag, percorso di archiviazione, contenuto, proprietario e altri campi. Nella directory Media si trovano l’originale e, se presenti, file di archivio e miniatura. Nomi file e Storage Paths configurabili modificano la disposizione fisica, non l’identità dell’oggetto database.

Gli amministratori evitano rinomini o spostamenti diretti nella directory Media. Il [`document_renamer`](https://docs.paperless-ngx.com/administration/#document-renamer) applica in modo coerente il formato configurato. Uno strumento esterno di sincronizzazione file può eseguire il backup del patrimonio, ma non riorganizzarlo autonomamente accanto all’applicazione in esecuzione. Per le basi dello storage, vedere [Storage](/kb/storage), per copia e verifica [Rclone](/kb/rclone).

## Metadati, matching e workflow

Corrispondenti, tipi di documento, tag, percorsi di archiviazione, campi personalizzati e valori di data formano un ordinamento sfaccettato. Non sostituiscono un albero di file, ma descrivono un documento lungo più dimensioni indipendenti. Il matching automatico fornisce proposte o assegnazioni sulla base del contenuto riconosciuto e dei dati esistenti; non è un servizio di verità immutabile.

I workflow sono composti da trigger, filtri opzionali e azioni sequenziali. Possono impostare metadati, autorizzazioni e altre proprietà durante il consumo o in occasione di altri eventi. La [documentazione sui workflow](https://docs.paperless-ngx.com/usage/#workflows) descrive trigger e azioni. Per gli amministratori ne derivano tre regole:

- Ogni automazione necessita di un insieme di input definito e di un risultato osservabile.
- Ordine delle regole e corrispondenze multiple vengono testati come codice di policy.
- Dopo le modifiche, i documenti esistenti vengono rivalutati in modo controllato solo con il [`document_retagger`](https://docs.paperless-ngx.com/administration/#document-retagger).

## Ricerca: query sul database e proiezione Tantivy

La ricerca full-text utilizza un indice Tantivy separato. Il database rimane autorevole per oggetti documento e metadati; l’indice accelera ricerca e ranking. Pertanto un documento può essere visibile nel database ma mancare nelle ricerche full-text. La migrazione a Tantivy e i campi di ricerca modificati sono documentati dalla [guida alla migrazione](https://docs.paperless-ngx.com/migration-v3/#search-index-whoosh-tantivy).

Dopo gli upgrade o in caso di sospetta deriva, l’indice può essere ricostruito tramite il comando di gestione. La reindicizzazione è una riparazione di una proiezione, non un ripristino degli originali. La pianificazione della capacità considera dimensione dell’indice, spazio temporaneo durante una ricostruzione e carico I/O parallelo al consumo normale.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Suche und Wartung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec webserver document_index list
docker compose exec webserver document_index reindex --if-needed
docker compose exec webserver document_sanity_checker
docker compose exec webserver document_thumbnails --help</code></pre>
  </div>
</div>

I comandi di gestione vengono avviati con Compose tramite [`docker compose exec`](https://docs.docker.com/reference/cli/docker/compose/exec/) nel container Webserver. Le installazioni bare metal utilizzano `python3 manage.py`; la forma di chiamata supportata e gli aiuti integrati sono documentati nell’[amministrazione](https://docs.paperless-ngx.com/administration/#management-utilities).

I file OCR e i metadati da soli non costituiscono ancora un servizio coerente. Database, broker dei task e proiezione di ricerca devono corrispondere allo stesso stato di elaborazione.

## Database, broker e coerenza

PostgreSQL è il percorso standard per le nuove installazioni. MariaDB e SQLite sono possibili, ma differiscono per strumenti di backup, comportamento in parallelo e limiti operativi. Le migrazioni del database fanno parte di un upgrade di Paperless; una vecchia versione binaria ripristinata non è necessariamente compatibile con uno schema già migrato. L’esercizio del database viene quindi trattato come un lifecycle separato ([backup e ripristino PostgreSQL](https://www.postgresql.org/docs/current/backup.html), [backup e ripristino MariaDB](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)).

Il broker trasporta i task tra processi. La persistenza Redis/Valkey non sostituisce il backup di database e documenti. Dopo la perdita del broker, lavori in corso o consegnati possono essere nuovamente avviati, persi oppure visibili come stato di task incoerente. L’accettazione avviene quindi sull’oggetto documento e sui suoi file, non su una coda vuota.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Datenbank- und Brokertests">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec db pg_isready
docker compose exec broker redis-cli PING
docker compose logs --since 30m webserver
docker compose logs --since 30m broker db</code></pre>
  </div>
</div>

[`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) verifica l’accettazione delle connessioni PostgreSQL, non lo schema o le query funzionali. [`redis-cli`](https://redis.io/docs/latest/develop/tools/cli/) testa la raggiungibilità del broker con `PING`. [`docker compose logs`](https://docs.docker.com/reference/cli/docker/compose/logs/) raccoglie log di servizio limitati nel tempo; credenziali, titoli dei documenti e percorsi vengono redatti prima della condivisione.

## Configurazione, Secret Key e risoluzione dei percorsi

Con Docker, le impostazioni vengono tipicamente fornite come file Environment o Compose Env; `paperless.conf` non è applicabile in questo caso. Le installazioni bare metal cercano la configurazione secondo un ordine documentato. Alcune impostazioni possono inoltre essere memorizzate nell’applicazione e sovrascrivere i valori Environment. La configurazione effettiva è quindi più di un singolo file ([ordine di configurazione](https://docs.paperless-ngx.com/configuration/#configuration)).

`PAPERLESS_SECRET_KEY` firma sessioni e valori sensibili ed è obbligatorio. `PAPERLESS_URL`, Allowed Hosts, impostazioni CSRF e CORS definiscono il confine Web. Credenziali per database, e-mail, OAuth, Tika e AI opzionale sono segreti. Le configurazioni Docker possono usare per numerosi valori la variante `_FILE` e quindi segreti montati; lo documenta la [guida di configurazione](https://docs.paperless-ngx.com/setup/#docker-compose-install).

Il backup della configurazione include almeno:

- definizione Compose/servizio e riferimento dell’immagine fissato,
- Environment senza copie in chiaro in repository non protetti,
- Secret Key e riferimenti esterni ai segreti,
- UID/GID, assegnazione di volumi e bind mount,
- configurazione di proxy, DNS, certificati e account e-mail,
- lingue OCR, formato dei nomi file, workflow e hook personalizzati.

## Confine Web, TLS e reverse proxy

L’applicazione dovrebbe essere raggiungibile dietro un reverse proxy correttamente configurato tramite HTTPS. Header Host, URL di origine, CSRF Trusted Origins, Forwarded Headers e limiti di upload devono essere coerenti. Il proxy può impostare header Remote User solo se autentica in modo affidabile l’identità a monte e rimuove header client con lo stesso nome.

La [documentazione sulla sicurezza](https://docs.paperless-ngx.com/configuration/#hosting-security) descrive parametri Secret Key, URL, Host, CORS e CSRF. Le basi sui certificati e il trasporto si trovano in [TLS](/kb/tls), la protezione sistematica in [Hardening](/kb/haertung). Il progetto avverte nel [repository](https://github.com/paperless-ngx/paperless-ngx#important-note) che i documenti vengono memorizzati in chiaro e che Paperless-ngx non deve essere eseguito su un host non attendibile.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Web- und Portdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection paperless.example.net -Port 443
curl.exe -sS -D - -o NUL https://paperless.example.net/
Resolve-DnsName paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp
curl -sS -D - -o /dev/null https://paperless.example.net/
getent ahosts paperless.example.net
docker compose port webserver 8000</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) verifica la raggiungibilità TCP in Windows; [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostra i listener locali; [`curl`](https://curl.se/docs/manpage.html) rende visibili stato, redirect e header; [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) o [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) verificano la risoluzione dei nomi; [`docker compose port`](https://docs.docker.com/reference/cli/docker/compose/port/) mostra la mappatura effettiva delle porte.

## Identità, diritti globali e diritti sugli oggetti

Paperless-ngx utilizza il modello utenti e gruppi di Django. Le autorizzazioni globali controllano quali tipi di oggetto e aree UI/API può utilizzare un account. Le autorizzazioni riferite agli oggetti controllano Owner, View e Change per un documento, tag o altro oggetto specifico. La [documentazione sulle autorizzazioni](https://docs.paperless-ngx.com/usage/#permissions) sottolinea che i diritti di un tag non vengono trasferiti automaticamente ai documenti contrassegnati.

I superuser possono vedere tutti gli oggetti. Lo stato Staff/Admin deve essere distinto e abilita, tra l’altro, viste backend e di sistema. Per l’operatività quotidiana si usano account e gruppi normali; gli account di servizio ricevono soltanto i diritti API e sugli oggetti necessari. I workflow possono impostare Owner e diritti durante il consumo, aspetto particolarmente importante per documenti senza proprietario provenienti da directory Consume.

L’autenticazione può avvenire localmente, tramite Reverse-Proxy-Remote-User o mediante procedure Social/OIDC configurate. Remote User viene attivato solo dietro un confine proxy chiuso. Ciclo di vita di MFA, sessioni e token viene incluso nel concetto generale di identità.

## REST API e confine dell’automazione

La REST API è raggiungibile sotto il prefisso dell’applicazione; una vista schema interattiva si trova in `/api/schema/view/`. Sono supportate, tra le altre, autenticazione Basic, Session, Token, Remote User e Headless OIDC. La [documentazione API](https://docs.paperless-ngx.com/api/) descrive ricerca, autorizzazioni sugli oggetti, operazioni bulk e header di versione.

Un token API è un segreto e viene assegnato separatamente per ogni automazione. I client rispettano paginazione, filtri, autorizzazioni, versione API e ripetizioni idempotenti. Una risposta di upload positiva non equivale ancora a OCR completato; le automazioni correlano ID documento o task fino allo stato finale funzionale. I principi su contratto, autenticazione e comportamento di retry sono descritti in [API](/kb/apis).

## Recupero e-mail come interfaccia di messaggistica

Account e regole e-mail collegano Paperless-ngx a una casella IMAP. Lo Scheduler controlla periodicamente gli account configurati per impostazione predefinita, applica le regole e trasferisce gli allegati corrispondenti alla pipeline Consume. Il comando di gestione `mail_fetcher` può avviare manualmente il recupero; lo documenta l’[amministrazione del recupero e-mail](https://docs.paperless-ngx.com/administration/#fetching-e-mail).

Per gli amministratori di messaggistica, i seguenti valori devono essere inclusi nel runbook:

- FQDN del server, porta e modalità TLS,
- metodo di autenticazione, scadenza di segreti/token e account di servizio,
- cartella IMAP, criterio di ricerca, tipi di file e dimensione massima del messaggio,
- azione dopo importazione riuscita o fallita,
- gestione di messaggi consegnati più volte e allegati identici,
- correlazione tra Message-ID, task Paperless e ID documento.

La semantica IMAP è definita da [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051). Un recupero IMAP riuscito dimostra solo l’ingresso nel percorso task; completamento OCR e archivio vengono monitorati separatamente.

## Documenti Office, Tika e Gotenberg

Apache Tika estrae contenuti e metadati da numerosi tipi di file; Gotenberg fornisce servizi di conversione. Paperless necessita di entrambi quando è attivata l’elaborazione opzionale Office/e-mail. I loro endpoint sono configurati separatamente ([Tika Server](https://tika.apache.org/3.2.3/gettingstarted.html#Using_Tika_as_a_service), [configurazione Gotenberg](https://gotenberg.dev/docs/configuration)).

I due servizi aumentano superficie d’attacco, fabbisogno di risorse e spazio degli errori. Vengono resi raggiungibili solo internamente, ricevono limiti CPU/RAM/timeout e non vengono pubblicati come API di conversione generica. Se i PDF funzionano ma DOCX o EML falliscono sistematicamente, la diagnosi inizia su questo percorso laterale e non da Tantivy o dal database.

## Esercizio container, bare metal e storage

I percorsi ufficiali sono script di installazione, Docker Compose e bare metal. Compose fornisce definizioni di servizi riproducibili e volumi separati; bare metal offre maggiore controllo su processi, pacchetti e systemd, ma richiede una gestione esplicita del lifecycle. La [matrice di configurazione](https://docs.paperless-ngx.com/setup/#overview) classifica i percorsi per impegno e utilizzo.

Le directory persistenti non vengono confuse con il lifecycle del container. Con Compose, Media, Data, Consume, Export e percorso del database sono risolti consapevolmente. I filesystem remoti o di rete vengono verificati per comportamento POSIX, rinomina atomica, blocchi, latenza, inotify e disponibilità. Altrimenti una breve interruzione può sembrare una perdita permanente di documenti. Le basi sui mount sono disponibili in [Container](/kb/container) e [Storage](/kb/storage).

In caso di guasto si determina anzitutto in quale fase della pipeline si trova il documento. Quindi vengono verificati in modo mirato coda, Worker, convertitore esterno, database e archiviazione file.

## Osservabilità e diagnosi dei task

Una panoramica operativa utile separa quattro segnali:

- **Salute del servizio:** processo in esecuzione, porta in ascolto, database e broker raggiungibili.
- **Throughput:** nuovi task, documenti completati, tempi di esecuzione e profondità della coda.
- **Errori:** task per fase e tipo di file, exit di parser/OCR, errori e-mail e storage.
- **Integrità del patrimonio:** originali/archivi/miniatire mancanti o danneggiati, file orfani, deriva dell’indice.

Il Sanity Checker integrato rileva, tra l’altro, file mancanti, illeggibili o modificati rispetto al checksum memorizzato, nonché file Media orfani. L’elenco completo è riportato nella [documentazione del Sanity Checker](https://docs.paperless-ngx.com/administration/#sanity-checker). Un allarme sul numero di errori senza ID documento/task e fase della pipeline è difficilmente azionabile.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Task- und Storage-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose logs --since 1h --timestamps webserver
docker compose exec webserver document_sanity_checker
docker compose exec webserver mail_fetcher
docker stats --no-stream
docker system df -v</code></pre>
  </div>
</div>

[`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) mostra l’utilizzo istantaneo di CPU/memoria, [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) l’occupazione di storage Docker. Entrambi sono istantanee. Per le cause vengono correlati serie temporali, ID task e log Paperless. La metodologia generale è descritta in [Troubleshooting](/kb/troubleshooting).

## Capacità, parallelismo e limiti

La pianificazione della capacità inizia con pagine al giorno, formati di ingresso, dimensione media e massima dei file, lingue OCR, task paralleli, tempo di conservazione e volume di ricerca. Un documento può generare contemporaneamente originale, PDF d’archivio, miniatura, record database, indice full-text e file temporanei di rendering. Lo spazio libero deve considerare anche reindicizzazione, esportazione e upgrade.

Un maggiore parallelismo dei Worker accelera l’OCR vincolato alla CPU solo fino alla saturazione di CPU, RAM e I/O. Allo stesso tempo aumenta il numero di file temporanei e connessioni al database. Il recupero e-mail e il Consumer possono generare più task di quanti i Worker possano elaborare; pertanto viene osservata l’età della coda invece della sola lunghezza momentanea. I limiti vengono testati con documenti rappresentativi, non con piccoli PDF di testo.

L’ambito del backup segue il modello dei dati: originali, archivi, database, configurazione, Secret Key ed eventualmente servizi aggiuntivi devono poter essere ripristinati insieme.

## Backup: quali oggetti vengono salvati insieme

La [documentazione sul backup](https://docs.paperless-ngx.com/administration/#making-backups) ufficiale indica due percorsi principali: Document Exporter oppure un backup coordinato di volumi/directory e database. L’Exporter scrive documenti, miniature, metadati e contenuto del database in una directory portabile. I token API non vengono trasferiti; esportazione e importazione dovrebbero essere compatibili per versione.

Un set di ripristino completo contiene:

| Oggetto | Perché necessario | Regola di coerenza |
|---|---|---|
| Database | ID, metadati, utenti, regole, riferimenti task | corrispondente allo snapshot Media |
| Media | originali, archivi, miniature | nessuna scrittura incompleta |
| Data | indice e stati ausiliari | in parte derivabile, ma utile per un riavvio rapido |
| Configurazione | percorsi, URL, OCR, e-mail, proxy, database | stesso contratto di nomi e mount |
| Segreti | Secret Key, credenziali DB/e-mail/OIDC | protetti, separati e ripristinabili |
| Manifest di esportazione | assegnazione portabile tra oggetti e file | conservare insieme ai file esportati |
| Runbook | sequenza, versione, test di accettazione | disponibili offline |

L’Exporter può aggiornare esportazioni esistenti e confrontare opzionalmente le checksum. Ciò fa risparmiare tempo, ma non sostituisce una copia separata e versionata: con `--delete` i documenti eliminati possono essere rimossi dalla destinazione di esportazione. Dettagli e opzioni sono documentati dal [`document_exporter`](https://docs.paperless-ngx.com/administration/#document-exporter).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paperless-Export und Recoveryprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker compose exec -T webserver document_exporter ../export --compare-checksums
Get-ChildItem -Recurse .\export | Measure-Object -Property Length -Sum
Get-FileHash .\export\manifest.json -Algorithm SHA256
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker compose exec -T webserver document_exporter ../export --compare-checksums
du -sh ./export
sha256sum ./export/manifest.json
docker compose exec webserver document_sanity_checker</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) garantiscono la prova di trasferimento del manifest. Una checksum dimostra soltanto l’uguaglianza di due insiemi di byte, non l’importabilità o la completezza funzionale. Le basi del backup sono disponibili in [Backup e Disaster Recovery](/kb/backup-dr).

## Ripristino e accettazione

Il [`document_importer`](https://docs.paperless-ngx.com/administration/#document-importer) richiede un’installazione vuota oppure, con `--data-only`, un database vuoto. Un test di ripristino utilizza pertanto percorsi di destinazione isolati, una versione Paperless compatibile con il backup e nessun volume di produzione.

L’accettazione comprende almeno:

1. Applicare configurazione e segreti nell’ambiente di destinazione.
2. Fornire database vuoto e percorsi Media/Data vuoti.
3. Importare l’esportazione oppure ripristinare database e file in modo coerente.
4. Eseguire in modo controllato migrazioni ed eventualmente la costruzione dell’indice.
5. Verificare accesso utente, gruppi e autorizzazioni.
6. Confrontare numero di documenti, metadati e campioni di originale e archivio.
7. Testare ricerca full-text e visualizzazione delle miniature.
8. Eseguire il Sanity Checker senza risultati inspiegati.
9. Acquisire un nuovo documento di test attraverso l’ingresso reale.
10. Documentare RPO e RTO con timestamp.

Un ripristino è considerato riuscito solo dopo questo test end-to-end. «Il container si avvia» non è un criterio di accettazione.

## Aggiornamenti, migrazioni del database e limite di rollback

Prima di un aggiornamento si completano i task Consume attivi, si crea un backup e si controllano le release note per breaking changes. La [guida all’aggiornamento](https://docs.paperless-ngx.com/administration/#updating-paperless) separa procedure Docker e bare metal e descrive migrazione del database e reindicizzazione secondo necessità. Un’immagine viene fissata a una serie approvata o a un digest; `latest` non è un punto di rollback riproducibile.

Dopo una migrazione del database, un rollback normalmente consiste nel ripristino di database e patrimonio file corrispondente, non soltanto nell’avvio della vecchia immagine. Nuovi archivi o metadati creati dopo il punto di backup devono essere trattati funzionalmente prima del ritorno. I principi generali su release e modifiche sono disponibili in [Release](/kb/releases) e [Migrazione](/kb/migration).

Il [changelog](https://docs.paperless-ngx.com/changelog/) è la fonte dipendente dal tempo per nuove funzionalità, breaking changes e correzioni. Questo articolo descrive invece principi stabili di architettura ed esercizio.

## Scenari di errore per fase della pipeline

Poiché un documento attraversa più fasi di elaborazione, il sintomo visibile spesso mostra solo l’ultima fase riuscita. La tabella conduce dal sintomo alla successiva evidenza affidabile.

| Sintomo | Confine probabile | Prime evidenze |
|---|---|---|
| Il file rimane nella directory Consume | watcher, polling, UID/GID, completamento scrittura | mount, permessi, log Consumer |
| L’upload risponde, ma non viene creato alcun documento | broker o Task Worker | ID task, PING broker, log Worker |
| Falliscono solo DOCX/EML | Tika/Gotenberg | endpoint, timeout, log convertitore |
| OCR molto lento o processo terminato | CPU/RAM, numero pagine, pacchetti linguistici | statistiche container, evento kernel/runtime, dimensione pagine |
| Documento visibile, full-text mancante | parser/OCR o indice di ricerca | testo estratto, log task, stato reindicizzazione |
| Miniatura mancante, originale presente | file derivato | Sanity Checker, comando miniatura |
| Login/POST errato dietro proxy | URL, Host, CSRF, Forwarded Header | header HTTP, configurazione effettiva |
| Recupero e-mail periodico fermo | Scheduler, broker, autenticazione e-mail | log Scheduler, `mail_fetcher`, test IMAP |
| File scomparsi dopo il riavvio | mount errato o volatile | `docker inspect`, inventario volumi |
| Il ripristino non importa | condizione di versione, percorso o vuoto | manifest di esportazione, versione destinazione, log importazione |

La raccolta ufficiale di [Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/) integra messaggi di errore concreti. Prima delle modifiche, gli amministratori salvano sempre ID task, nome documento, finestra temporale, configurazione effettiva, stato di container/processi e stato di tutti i servizi dipendenti.

## Storia tecnica

Il progetto Paperless originale ha stabilito il modello di base composto da directory Consume, OCR e archivio ricercabile. Paperless-ng ha proseguito lo sviluppo come fork. Paperless-ngx è nato in seguito come successore ufficiale della community per distribuire manutenzione e responsabilità tra più collaboratori; questa discendenza è descritta nel [README del progetto](https://github.com/paperless-ngx/paperless-ngx#paperless-ngx).

La piattaforma si è evoluta da un’applicazione Django principalmente lato server a un DMS con frontend Angular, REST API, autorizzazioni a livello di oggetto, motore di workflow ed elaborazione parallela dei task. OCRmyPDF/Tesseract sono rimasti la base OCR; Tika e Gotenberg hanno ampliato i formati di ingresso supportati. Il percorso di ricerca è passato da Whoosh a Tantivy, rendendo visibile l’indice come dominio autonomo di migrazione ed esercizio ([migrazione Tantivy](https://docs.paperless-ngx.com/migration-v3/)).

Questa storia spiega l’architettura odierna: un documento Paperless-ngx è contemporaneamente oggetto relazionale, insieme di file gestiti, risultato OCR e proiezione di ricerca. Manutenzione e recovery devono considerare tutte e quattro le prospettive.

## Checklist per amministratori

In conclusione vengono verificati insieme ingressi, Worker, storage, ricerca e backup. La checklist aiuta a non trascurare dipendenze esterne al container Paperless.

- **Inventario:** immagine/commit, applicazione Django/Angular, Worker, Scheduler, broker, database, Tika/Gotenberg, proxy e mount.
- **Oggetti dati:** denominare separatamente database, originale, archivio, miniatura, indice, configurazione, segreti e manifest di esportazione.
- **Ingressi:** documentare Consume, upload/API ed e-mail con regole per dimensioni, tipi, autenticazione ed errori.
- **Pipeline:** correlare l’ID task attraverso parser, OCR, archiviazione, metadati e indice fino al completamento.
- **Autorizzazioni:** testare diritti globali e a livello di oggetto, superuser, gruppi e assegnazioni dei workflow.
- **Capacità:** monitorare età della coda, durata OCR, spazio temporaneo, crescita Media/indice e connessioni al database.
- **Sicurezza:** includere HTTPS, fiducia nel proxy, Secret Key, token, storage in chiaro e parser esterni nel modello di minaccia.
- **Backup:** salvare in modo coerente database e Media; non dimenticare configurazione e segreti.
- **Ripristino:** verificare istanza di destinazione vuota, versione compatibile, campioni, ricerca, Sanity Checker e nuova esecuzione Consume.
- **Modifiche:** definire in anticipo release note, migrazione schema/indice, spazio libero e vero limite di rollback.

## Fonti

- [Paperless-ngx – Documentazione](https://docs.paperless-ngx.com/)
- [Paperless-ngx – Repository GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx – Sviluppo](https://docs.paperless-ngx.com/development/)
- [Paperless-ngx – Utilizzo](https://docs.paperless-ngx.com/usage/)
- [Celery](https://docs.celeryq.dev/en/stable/)
- [Paperless-ngx – Manifest del progetto Python](https://github.com/paperless-ngx/paperless-ngx/blob/dev/pyproject.toml)
- [Paperless-ngx – Dockerfile](https://github.com/paperless-ngx/paperless-ngx/blob/dev/Dockerfile)
- [Paperless-ngx – Configurazione](https://docs.paperless-ngx.com/configuration/)
- [Broker compatibile con Redis](https://redis.io/docs/latest/)
- [PostgreSQL](https://www.postgresql.org/docs/current/)
- [Paperless-ngx – Installazione](https://docs.paperless-ngx.com/setup/)
- [Docker – compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker – compose ps](https://docs.docker.com/reference/cli/docker/compose/ps/)
- [Docker – compose images](https://docs.docker.com/reference/cli/docker/compose/images/)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Paperless-ngx – FAQ](https://docs.paperless-ngx.com/faq/)
- [Paperless-ngx – Consume Hooks](https://docs.paperless-ngx.com/advanced_usage/)
- [OCRmyPDF – Documentazione](https://ocrmypdf.readthedocs.io/en/latest/)
- [Tesseract – Documentazione](https://tesseract-ocr.github.io/tessdoc/)
- [Paperless-ngx – Document Renamer](https://docs.paperless-ngx.com/administration/)
- [Paperless-ngx – Migrazione Tantivy](https://docs.paperless-ngx.com/migration-v3/)
- [Docker – compose exec](https://docs.docker.com/reference/cli/docker/compose/exec/)
- [PostgreSQL – Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [MariaDB – Backup and Restore](https://mariadb.com/docs/server/server-usage/backing-up-and-restoring-databases/)
- [PostgreSQL – pg_isready](https://www.postgresql.org/docs/current/app-pg-isready.html)
- [Redis – redis-cli](https://redis.io/docs/latest/develop/tools/cli/)
- [Docker – compose logs](https://docs.docker.com/reference/cli/docker/compose/logs/)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [curl – Manuale](https://curl.se/docs/manpage.html)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Docker – compose port](https://docs.docker.com/reference/cli/docker/compose/port/)
- [Paperless-ngx – REST API](https://docs.paperless-ngx.com/api/)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Apache Tika – Esercizio del server](https://tika.apache.org/3.2.3/gettingstarted.html)
- [Gotenberg – Configurazione](https://gotenberg.dev/docs/configuration)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Docker – system df](https://docs.docker.com/reference/cli/docker/system/df/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – utility sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Paperless-ngx – Changelog](https://docs.paperless-ngx.com/changelog/)
- [Paperless-ngx – Troubleshooting](https://docs.paperless-ngx.com/troubleshooting/)
