---
title: "Pi-hole 2026.11.0: FTL 6.9 e Core 6.6 chiudono undici vulnerabilità di sicurezza"
navTitle: "Pi-hole 2026.11.0"
description: "L'immagine Docker Pi-hole 2026.11.0 include FTL v6.9 e Core v6.6 e corregge, tra l'altro, un'esecuzione di codice da parte dell'amministratore tramite files.gravity (CVSS 7.2) e diverse vulnerabilità legate ai symlink nell'installer. I filtri regex, webserver.acl e l'API delle sessioni si comportano in parte diversamente; verificate questi punti dopo l'aggiornamento."
date: "2026-11-12"
kategorie: "Rete e DNS"
timeToRead: "7 min di lettura"
themen:
  - smart-home-iot
produkte:
  - "pihole"
protokolle:
  - "releases"
  - "dns"
  - "container"
slug: "pi-hole-2026-11-0-ftl-6-9-e-core-6-6-chiudono-undici-vulnerabilita-di-sicurezza"
translationId: article-0eb045e8f65bedd0
translationOf: pihole-2026-11-0
url: https://rafaelpfister.ch/it/blog/pi-hole-2026-11-0-ftl-6-9-e-core-6-6-chiudono-undici-vulnerabilita-di-sicurezza
translationSourceHash: 44a4448ed57216f76890c2807ea763f7c340fcdc0fffa70b8e8e50816c28f4c4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:42:30.535Z
translationReview: automatic
---

# Pi-hole 2026.11.0: FTL 6.9 e Core 6.6 chiudono undici vulnerabilità di sicurezza

Pi-hole ha pubblicato l'immagine Docker 2026.11.0 l'11 novembre 2026. Include Pi-hole FTL v6.9 e Pi-hole Core v6.6, pubblicati lo stesso giorno; secondo il produttore, la Web Interface resta alla v6.6.1. Le due release dei componenti correggono undici Security Advisories, di cui quattro con gravità High (CVSS da 7.2 a 7.3). FTL v6.9 aggiorna inoltre dnsmasq, SQLite e il motore regex TRE e modifica il comportamento dei filtri regex, `webserver.acl` e di parti dell'API. La release è rivolta a tutti gli operatori di Pi-hole, sia in un container Docker sia come installazione bare metal. Secondo le Release Notes, la versione precedente 2026.10.1 del 31 ottobre 2026 ha introdotto soprattutto un'immagine di base Alpine aggiornata (3.24.3).

## Panoramica delle versioni

| Versione | Data | Tipo | Punti chiave |
|---|---|---|---|
| Docker 2026.11.0 | 11.11.2026 | Release dell'immagine | include FTL v6.9, Core v6.6 e Web v6.6.1; `setpriv` al posto di `sudo`, logrotate tramite il crontab del container |
| FTL v6.9 | 11.11.2026 | Minor release | sette advisory (uno High), dnsmasq v2.93+42, SQLite 3.54.1, aggiornamento TRE con comportamento regex modificato |
| Core v6.6 | 11.11.2026 | Minor release | cinque advisory (quattro High, uno Medium), l'installer scrive in `/etc/pihole` solo come utente `pihole` |
| Web v6.6.1 | 17.10.2026 | invariata | nessuna nuova versione in questa release |

## Correzioni di sicurezza con gravità High

Gli advisory sono pubblicati dall'11 novembre 2026. Al momento non era stato assegnato un ID CVE per gli advisory esaminati; secondo il testo dell'advisory relativo a `files.gravity`, è stato richiesto (stato al 12 novembre 2026).

- **GHSA-pj9m-9623-2g75 (FTL) e GHSA-cq9w-mv93-mxh7 (Core), CVSS 7.2:** esecuzione di codice da parte di un amministratore autenticato (CWE-78, CWE-89). FTL 6.8 aveva allentato la verifica del percorso, consentendo a `files.gravity` di contenere un apostrofo semplice. `gravity.sh` inseriva questo valore senza protezione in un'istruzione SQL-`ATTACH`; tramite la shell sqlite3 di FTL era così possibile eseguire comandi del sistema operativo come utente `pihole`, secondo l'advisory sia nelle installazioni Docker sia bare metal. Sono interessati Core fino alla 6.5 e FTL 6.8. Core ora effettua l'escape del valore, FTL associa il percorso nel Teleporter come parametro. Il produttore sottolinea che entrambi i pacchetti devono essere aggiornati insieme.
- **GHSA-wp3p-45h3-2v82 (Core), CVSS 7.3:** correzione incompleta delle vulnerabilità symlink di ottobre (CWE-59, CWE-61). `pihole -r` e `pihole -up` continuavano a seguire symlink durante la scrittura di `install.log`, `migration_backup_v6` e `hosts/custom.list` in `/etc/pihole`. Il prerequisito è l'esecuzione di codice come utente `pihole` e un amministratore che successivamente avvii una riparazione o un aggiornamento come root. Sono interessate le versioni fino alla 6.5.
- **GHSA-5w4v-f65m-323f (Core), CVSS 7.3:** root seguiva symlink e percorsi dalla configurazione durante il download di `macvendor.db`, in `pihole checkout` e in `pihole uninstall`.
- **GHSA-jvg9-fm8x-2jqq (Core), CVSS 7.3:** `pihole -d` seguiva symlink in directory di proprietà dell'utente `pihole`. In questo modo root poteva copiare file arbitrari nel log di debug e sovrascriverne altri o modificarne il proprietario.

Core v6.6 scrive quindi `install.log`, il backup della migrazione, `hosts/custom.list` e `macvendor.db` solo come utente `pihole`. Anche `pihole -d` non opera più come root in queste directory.

## Hash delle password nel log di debug

**GHSA-qfq3-88xg-765w (Core), CVSS 5.3:** il log di debug di `pihole -d` e quindi anche i caricamenti al servizio Tricorder contenevano l'hash della password web, l'hash della password dell'app e il segreto TOTP. Core v6.6 rimuove questi valori dal log. Se in passato avete caricato un log di debug, il produttore raccomanda di modificare la password e le password dell'app e di configurare nuovamente l'autenticazione a due fattori.

## Correzioni di sicurezza con gravità Low in FTL

- **GHSA-x6xh-vxcm-g8rm, CVSS 3.8:** le sessioni dell'app senza `webserver.api.app_sudo` e le sessioni CLI potevano elencare tutte le sessioni API e terminarne arbitrariamente una.
- **GHSA-87wp-wm3p-8cxm, CVSS 3.7:** le voci IPv4 in `webserver.acl` non si applicavano ai client IPv4 in un `webserver.port` dual stack come `+80`.
- **GHSA-cg4p-5r3r-m54p, CVSS 3.1:** una modifica della password tramite `pihole-FTL --config` o `pihole setpassword` poteva essere annullata da una scrittura di configurazione simultanea tramite API; la vecchia password e le relative sessioni restavano quindi valide.
- **GHSA-fj2f-pgf2-j888, CVSS 2.2:** le query registrate con `misc.privacylevel` 3 potevano finire nel database a lungo termine dopo aver abbassato il livello.
- **GHSA-9mh9-95wj-4w4q e GHSA-9hh4-j8jf-6c96, entrambi CVSS 2.2:** un arresto anomalo durante la lettura delle metriche nel corso di un ricaricamento del file hosts e un use-after-free durante un'importazione Teleporter in formato v5. Entrambi richiedono una sessione valida.

## Comportamento modificato dopo l'aggiornamento

Secondo le Release Notes, FTL v6.9 non è volutamente una v6.8.1, perché l'aggiornamento di TRE modifica la valutazione di alcuni filtri regex salvati:

- **Regex:** un limite inferiore mancante (`{,n}`, `{,}`) ora significa `{0,n}` o `{0,}`. Vengono rifiutati più di tre gruppi annidati con matching approssimativo e back-reference in ripetizioni illimitate.
- **DNS locale:** le voci in `dns.cnameRecords` con un nome di dominio non valido o una TTL che non rientra in un integer vengono rifiutate.
- **webserver.acl:** le voci IPv4 si applicano a tutte le porte, incluse quelle dual stack. Le voci IPv6, tranne `::ffff:a.b.c.d`, corrispondono ormai solo ai client IPv6; `[::]/0` non consente quindi più client IPv4.
- **API delle sessioni:** le sessioni dell'app senza `webserver.api.app_sudo` e le sessioni CLI ricevono lo stato 403 per `GET /api/auth/sessions` e `DELETE /api/auth/session/{id}`. Una sessione può continuare a terminare sé stessa con `DELETE /api/auth`.
- **Webserver:** le richieste con più di 64 header vengono rifiutate. `/api/info/system` restituisce un errore invece di valori nulli se i load average non sono leggibili.
- **API DHCP:** `DELETE /api/dhcp/leases/{ip}` risponde con 503 invece di 400 se il resolver non ha elaborato la richiesta in tempo.
- **Password:** una password impostata tramite `pihole setpassword` o `pihole-FTL --config` non viene più sovrascritta dalla successiva modifica della configurazione nella Web Interface.

Dopo l'aggiornamento, controllate i filtri regex con limiti inferiori o back-reference, le voci in `webserver.acl` (in particolare `[::]/0`) e gli script o le integrazioni che elencano o terminano sessioni con password dell'app.

## DNS, DHCP e ulteriori correzioni in FTL v6.9

Il dnsmasq integrato è alla v2.93+42 e contiene inoltre una correzione in `blockdata_alloc()` che upstream non ha ancora recepito. Secondo le Release Notes, sono corretti tra l'altro un heap out-of-bounds read in `zone_status()` (DNSSEC), un heap out-of-bounds write in `blockdata_read()`, un overflow del buffer dei nomi DNS, un off-by-one nell'opzione DHCPv6 FQDN, nonché SERVFAIL nelle risposte wildcard più profonde e risposte perse con il TCP pipelining. Con DNSSEC attivato, dnsmasq riserva slot TCP aggiuntivi per le proprie query di convalida, affinché la convalida funzioni anche quando i client esauriscono il limite delle connessioni TCP. L'upstream non filtrato di Quad9 è ora contrassegnato come validante DNSSEC.

Nella configurazione e nell'API, FTL v6.9 corregge tra l'altro modifiche perse in caso di due cambiamenti di configurazione simultanei, un worker TCP che tratteneva `pihole.toml.lock` fino a 300 secondi e variabili `FTLCONF_` ignorate quando FTL veniva avviato senza `pihole.toml`. In precedenza il webserver poteva bloccarsi al 100% di carico CPU e non accettare più connessioni. Nei container, `/api/info/system` finora riportava il carico dell'host. Rilevante per il funzionamento è inoltre che `gravity.db` viene riaperto solo se il file è cambiato; secondo il produttore, questo rappresentava la maggior parte del carico CPU di FTL in inattività. SQLite è alla versione 3.54.1.

## Modifiche in Core v6.6 e nell'immagine Docker

Core v6.6 passa da `sudo` a `setpriv` per l'abbassamento dei privilegi da root all'utente `pihole`; `sudo` non è quindi più una dipendenza. Secondo il produttore, i comandi amministrativi come `pihole -up` continuano a richiedere privilegi root. La rotazione dei log viene ora gestita dal logrotate giornaliero del sistema; la rotazione forzata notturna di Pi-hole viene eliminata poiché entrava in conflitto con l'esecuzione del sistema e ruotava i log due volte. Ulteriori correzioni riguardano `pihole flush quiet`, `pihole updatechecker` e un avviso quando un repository locale per git non è contrassegnato come sicuro (finora `pihole -v` mostrava soltanto N/A). È stata rimossa la capability non necessaria `CAP_IPC_LOCK`.

L'immagine Docker adotta il passaggio a `setpriv` e pianifica la rotazione dei log tramite il crontab del container. L'immagine notturna viene ora compilata dal branch `development`.

## Procedura consigliata

La versione di destinazione è l'immagine Docker 2026.11.0 oppure, per le installazioni bare metal, FTL v6.9 con Core v6.6. Poiché la vulnerabilità tramite `files.gravity` viene chiusa solo con entrambi i componenti, aggiornate FTL e Core insieme. Prima create un backup con Teleporter (Settings, Teleporter nella Web Interface oppure `pihole-FTL --teleporter`), come raccomandato dal produttore, e leggete i changelog a causa delle modifiche di comportamento. Per le installazioni bare metal aggiornate con `pihole -up`. Nel container, secondo la documentazione, `pihole -up` e `pihole -r` sono disattivati; sostituite invece l'immagine. Con Docker Compose, eseguite nella directory della `docker-compose.yml` i seguenti comandi secondo la documentazione:

```bash
docker compose pull
docker compose down
docker compose up -d
```

Per le installazioni con `docker run`, la documentazione indica `docker pull pihole/pihole:latest`, poi `docker stop pihole` e `docker rm pihole` e un nuovo `docker run` con gli argomenti precedenti. Se in passato avete caricato un log di debug, modificate inoltre le password e configurate nuovamente il 2FA. Le Release Notes non contengono indicazioni su problemi noti o termini di supporto (stato al 12 novembre 2026).

## Fonti

1.  [GitHub: pi-hole/docker-pi-hole Release 2026.11.0](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.11.0): Release Notes dell'immagine Docker con versioni dei componenti, modifiche Docker ed elenco degli undici advisory.

2.  [GitHub: pi-hole/FTL Release v6.9](https://github.com/pi-hole/FTL/releases/tag/v6.9): correzioni di sicurezza, aggiornamenti di dnsmasq e SQLite, modifiche di comportamento per regex, ACL e API.

3.  [GitHub: pi-hole/pi-hole Release v6.6](https://github.com/pi-hole/pi-hole/releases/tag/v6.6): correzioni di sicurezza del Core, passaggio a setpriv, modifiche alla rotazione dei log.

4.  [GitHub: pi-hole/web Release v6.6.1](https://github.com/pi-hole/web/releases/tag/v6.6.1): stato della Web Interface del 17 ottobre 2026, invariata in 2026.11.0.

5.  [GitHub: pi-hole/docker-pi-hole Release 2026.10.1](https://github.com/pi-hole/docker-pi-hole/releases/tag/2026.10.1): immagine precedente con Alpine 3.24.3.

6.  [GitHub Advisory GHSA-pj9m-9623-2g75](https://github.com/pi-hole/FTL/security/advisories/GHSA-pj9m-9623-2g75): esecuzione di codice da parte dell'amministratore tramite files.gravity, CVSS 7.2, versioni interessate e corrette, CVE richiesta.

7.  [GitHub Advisory GHSA-wp3p-45h3-2v82](https://github.com/pi-hole/pi-hole/security/advisories/GHSA-wp3p-45h3-2v82): correzione symlink incompleta nell'installer, CVSS 7.3, prerequisiti.

8.  [Pi-hole Blog: Pi-hole FTL v6.9 and Core v6.6 released](https://pi-hole.net/blog/2026/11/11/pi-hole-ftl-v6-9-and-core-v6-6-released/): annuncio con indicazioni di aggiornamento e raccomandazione dopo caricamenti di log di debug.

9.  [Pi-hole Docs: Upgrading Pi-hole in Docker](https://docs.pi-hole.net/docker/upgrading/): comandi di aggiornamento per Docker Compose e docker run, pihole -up disattivato nel container.
