---
title: "SQLite: forstå og forebygg «database is locked»"
navTitle: "SQLite-låser"
description: "Hvorfor SQLite svarer med SQLITE_BUSY og «database is locked», hvordan WAL-modus, busy_timeout, korte transaksjoner og forsøk på nytt unngår låser, og hvorfor trege lagringsmedier og I/O-belastning på en NAS først utløser problemet. Med et praktisk eksempel fra Grafana."
date: "2026-11-12"
kategorie: "Databaser"
timeToRead: "11 min lesetid"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-forsta-og-forebygg-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/no/blog/sqlite-forsta-og-forebygg-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:53:49.063Z
translationReview: automatic
---

SQLite finnes i mange selvhostede tjenester: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin og en rekke andre applikasjoner lagrer som standard dataene sine i én enkelt fil. Det fungerer helt til meldingen `database is locked` vises i loggen. Den sier at en tilgang ikke fikk en lås innenfor den tillatte ventetiden; selve databasefilen er imidlertid intakt. Om dette skjer sjelden eller hele tiden, avhenger av tre ting: journalmodus, applikasjonens oppførsel ved konflikter og lagringsmediets latenstid.

## Praktisk eksempel: Grafana på en NAS

Grafana kjørte som en Docker-container på en Synology-NAS, og SQLite-filen lå i et Named Volume. På samme NAS kjørte Prometheus, Loki, Paperless-ngx og flere restic-sikkerhetskopier. Lagringsmediene var harddisker med Shingled Magnetic Recording (SMR).

Først rapporterte Health-endepunktet til Grafana `"database": "failing"`. I loggen fantes det med korte mellomrom oppføringer av denne typen:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

Meldingen kommer fra Alert-State-History: Grafana skriver som standard tilstandsendringer for varselregler som annotasjoner i sin egen database. Med mange regler oppstår det dermed kontinuerlig små skrivetansaksjoner. Etter en omstart av containeren kom ikke Grafana opp igjen og startet på nytt i en løkke:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

Tilfellet ble løst med to innstillinger i `docker-compose.yml`: aktivering av WAL-modus og at Grafana prøver flere ganger ved låser. Hvorfor nettopp disse to verdiene virker, følger av hvordan SQLite-låsene fungerer.

## Hvordan SQLite låser

SQLite har ingen serverprosess som koordinerer tilgang. Hver tilkobling åpner filen selv og styrer delt tilgang via operativsystemets fillåser. Dette gir den viktigste egenskapen: **Bare én tilkobling kan skrive om gangen.** Det gjelder i alle journalmoduser.

I klassisk rollback-journalmodus (standard, `journal_mode=DELETE`) går en skrivetansaksjon gjennom flere låsenivåer:

| Lås | Betydning |
|---|---|
| `SHARED` | Tilkoblingen leser. Vilkårlig mange lesere samtidig er tillatt. |
| `RESERVED` | Tilkoblingen vil skrive og forbereder endringer. Lesere kan fortsette å lese, men ikke en annen skriver. |
| `PENDING` | Tilkoblingen venter på at pågående lesetilganger avsluttes. Nye lesere avvises. |
| `EXCLUSIVE` | Tilkoblingen skriver til databasefilen. Ingen andre kan lese eller skrive. |

For commit trenger skriveren altså den eksklusive låsen, og den får den først når ingen lesere lenger er aktive. En langvarig lesetilgang forsinker dermed enhver skriver, og under commit blokkeres alle lesere.

Hvis en tilkobling ikke får låsen den trenger, returnerer SQLite feilkoden `SQLITE_BUSY` (5) med teksten `database is locked`. Dette må skilles fra `SQLITE_LOCKED` (6, `database table is locked`): Denne koden gjelder konflikter i samme tilkobling eller i Shared-Cache-modus og har andre årsaker.

Avgjørende er hvor lenge SQLite venter før denne meldingen. Uten konfigurasjon venter SQLite ikke i det hele tatt: Standardverdien for `busy_timeout` er 0, og meldingen kommer straks en annen tilkobling holder låsen. Mange biblioteker angir sin egen verdi; Pythons modul `sqlite3` venter for eksempel 5 sekunder.

## WAL-modus: skill lesere og skrivere

I Write-Ahead-Log-modus (`journal_mode=WAL`) legger SQLite endringer til i en separat fil `<name>-wal` i stedet for å skrive dem direkte til databasefilen. Lesere ser tilstanden ved starten av lesetransaksjonen sin og leser da fra databasefilen og WAL. Dette gir to fordeler:

- Lesere blokkerer ikke skriveren, og skriveren blokkerer ikke leserne.
- En commit er en tilføyelse til én fil i stedet for flere spredte skriveoperasjoner med journal. Hver transaksjon krever mindre I/O, og skrivelåsen holdes kortere.

WAL-modus endrer ikke at bare én skriver kan være aktiv samtidig. Den forkorter imidlertid tiden låsen holdes og eliminerer konflikter mellom lesing og skriving. For en applikasjon som Grafana, som samtidig leverer dashboards og lagrer varseltilstander, er dette det mest effektive tiltaket.

Modusen angis én gang og lagres i filen:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `journal_mode = WAL` | Bytter til Write-Ahead-Log. Innstillingen er persistent; nye tilkoblinger åpner automatisk filen i WAL-modus. |
| `busy_timeout = 5000` | Ventetid i millisekunder der SQLite gjentatte ganger prøver å få en opptatt lås. Gjelder bare den aktuelle tilkoblingen og må angis på nytt ved hver åpning. |
| `synchronous = NORMAL` | I WAL-modus ingen `fsync` per commit, men bare ved sjekkpunkter. Databasen forblir konsistent ved strømbrudd; de sist bekreftede transaksjonene kan gå tapt. Gjelder per tilkobling. |

</details>

WAL-modus har forutsetninger. Alle prosesser som får tilgang til filen, må kjøre på samme maskin, fordi de samordner seg gjennom en delt minnefil `<name>-shm`. Katalogen trenger skrivetillatelser for de to tilleggsfilene, også for prosesser som bare leser. Ved sikkerhetskopiering må filene dessuten behandles samlet (mer om dette nedenfor).

## Sjekkpunkter: når WAL vokser

WAL vokser til et sjekkpunkt overfører endringene til selve databasefilen. SQLite utløser det automatisk så snart WAL omfatter rundt 1000 sider (`wal_autocheckpoint`, med 4-KiB-sider omtrent 4 MB). Sjekkpunktet skriver sidene til plasseringene sine i databasefilen og kaller deretter `fsync`. På et tregt lagringsmedium er nettopp dette trinnet det som tar lengst tid.

Et sjekkpunkt kan bare overføre sider så lenge ingen leser trenger en eldre tilstand. Hvis minst én lesetransaksjon kjører kontinuerlig, forblir sjekkpunktet ufullstendig og WAL fortsetter å vokse. En WAL på flere hundre megabyte er en indikasjon på langvarige lesetransaksjoner eller på sjekkpunkter som ikke blir ferdige på grunn av I/O-belastning. Dette kan kontrolleres slik:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

Utdataene består av tre tall: om sjekkpunktet ble blokkert (`1` = ja), hvor mange sider som finnes i WAL, og hvor mange av dem som ble overført. Med `TRUNCATE` kuttes WAL deretter ned til null byte, forutsatt at sjekkpunktet var fullstendig.

## Busy-timeout og forsøk på nytt

Med en busy-timeout venter SQLite ved en konflikt i stedet for å avbryte umiddelbart. Feilen oppstår da bare hvis låsen holdes lenger enn den angitte tiden. For applikasjoner med få, korte skriveoperasjoner er 5 til 10 sekunder tilstrekkelig.

Det finnes ett tilfelle der busy-timeout ikke gjelder. Hvis en transaksjon begynner med å lese (`BEGIN` uten tillegg, altså `DEFERRED`) og senere vil skrive, mens en annen skriver i mellomtiden har fullført en commit, returnerer SQLite umiddelbart `SQLITE_BUSY`. Transaksjonen arbeider på en utdatert tilstand og kan ikke bli en skrivetansaksjon; venting ville ikke endret det. I rollback-modus finnes en tilsvarende situasjon når to tilkoblinger samtidig vil gå fra `SHARED` til en skrivelås.

Løsningen er å markere skrivetansaksjoner som slike fra starten:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` ber om skrivelåsen ved start. Hvis den er opptatt, gjelder busy-timeout på vanlig måte. I Python ser det slik ut:

```python
import sqlite3

con = sqlite3.connect("app.db", timeout=10.0,
                      isolation_level=None)
con.execute("PRAGMA journal_mode = WAL")
con.execute("BEGIN IMMEDIATE")
con.execute("UPDATE jobs SET status = ? WHERE id = ?",
            ("done", 42))
con.execute("COMMIT")
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `"app.db"` | Sti til databasefilen. |
| `timeout=10.0` | Busy-timeout i sekunder; tilsvarer `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Slår av modulens automatiske transaksjonsstyring, slik at `BEGIN IMMEDIATE` og `COMMIT` kan angis eksplisitt. |

</details>

Forsøk på nytt på applikasjonsnivå utfyller busy-timeout. De fanger opp `SQLITE_BUSY`, venter kort og utfører hele transaksjonen på nytt. Dette dekker også tilfellet med utdatert lesetilstand, fordi gjentakelsen starter med en ny tilstand. Grafana tilbyr egne innstillinger for dette (se avsnittet om løsningen).

## Korte transaksjoner og én skriver

Hvor lenge en skrivelås holdes, bestemmes av applikasjonen. Tre regler holder den kort:

- **Ikke utfør arbeid i transaksjonen som ikke har med databasen å gjøre.** HTTP-kall, filtilganger eller beregninger hører hjemme før `BEGIN` eller etter `COMMIT`.
- **Samle mange små skriveoperasjoner.** 500 enkeltstående commits koster 500 låsesykluser og i rollback-modus 500 ganger flere `fsync`-kall. Én transaksjon med 500 rader holder låsen én gang og litt lenger.
- **Lukk lesecursorer.** Et spørringsresultat som ikke er hentet fullstendig, holder lesetransaksjonen åpen og blokkerer sjekkpunktet i WAL-modus.

Der flere tråder eller prosesser skriver, er én enkelt skriver med kø den mest pålitelige løsningen: Én tråd eller prosess eier den eneste skrivende tilkoblingen, mens alle andre overleverer oppgaver til den. Låskonflikter mellom skrivere forsvinner dermed helt, og lesere fortsetter å arbeide parallelt i WAL-modus.

Flere containere som åpner samme SQLite-fil via et delt volum, er derimot en vanlig årsak til låser. Hver container er en egen skriver, og ingen kjenner til den andre. Hvis en applikasjon skaleres horisontalt, er en databaseserver som PostgreSQL eller MySQL det riktige valget.

## Ingen nettverksfilsystemer

SQLite er avhengig av at fillåser fungerer korrekt og atomisk, og at en `fsync` faktisk skriver dataene til lagringsmediet. Med NFS, SMB/CIFS og mange FUSE-filsystemer er ingen av delene garantert. SQLite-utviklerne fraråder uttrykkelig å bruke en database over et nettverksfilsystem fra flere maskiner. Konsekvensene spenner fra vedvarende låsemeldinger til en skadet database, og WAL-modus fungerer der grunnleggende ikke, fordi delt minne-synkronisering bare er mulig lokalt.

For Docker betyr dette: Et Named Volume eller en bind-mount på et lokalt lagringsmedium er egnet. En bind-mount på en montert SMB- eller NFS-deling er det ikke, selv om den ser ut til å fungere i normal drift.

## I/O-belastning: hvorfor låsen plutselig varer for lenge

Låsen er bare et symptom. Den blir et problem når den holdes lenger enn busy-timeout tillater, og det skyldes nesten alltid skrivelatenstiden til lagringsmediet.

Hver commit avsluttes med en `fsync`. Først når lagringsmediet bekrefter at dataene er skrevet varig, frigir SQLite låsen. På en SSD tar det under ett millisekund, på en rolig harddisk noen millisekunder. Hvis lagringsmediet samtidig må behandle andre oppgaver, venter `fsync` i køen, og låsen forblir aktiv så lenge. Alle andre skrivere venter også, til busy-timeout utløper.

Flere faktorer øker denne latenstiden på en NAS:

- **SMR-harddisker.** Med Shingled Magnetic Recording overlapper sporene hverandre. Tilfeldige skriveoperasjoner havner først i et cacheområde og sorteres senere om i de overlappende sonene. Hvis cacheområdet fylles under vedvarende belastning, øker skrivelatenstiden fra millisekunder til flere sekunder. Databaser med mange små, spredte skriveoperasjoner er den mest ugunstige belastningsprofilen for dette.
- **Parallelle sikkerhetskopier.** restic, Hyper Backup eller rsync leser store datamengder og konkurrerer med hver `fsync` om de samme diskene.
- **Scrubbing og RAID-konsistenskontroller.** En disk-scrub leser hele datamengden og kjører etter tidsplan på mange NAS-systemer, ofte i timevis.
- **Andre databaser på samme volum.** Prometheus og Loki skriver kontinuerlig blokker og komprimerer dem med jevne mellomrom. Disse compaction-operasjonene skaper belastningstopper.
- **Sjekkpunkter i WAL-modus.** Sjekkpunktet skriver mange sider på én gang og avsluttes med `fsync`. Hvis det skjer i en periode med høy belastning, forlenges det tilsvarende.

I det praktiske eksempelet kom flere av disse faktorene samtidig: SMR-harddisker, sikkerhetskopier og et overvåkingsmiljø med flere skrivende tjenester. Grafana kjørte i rollback-modus uten forsøk på nytt. Hver commit i Alert-State-History krevde flere `fsync`-kall, og hvert av dem ventet bak den øvrige belastningen. Ved oppstart leser Grafana blant annet verdier fra nøkkel-verdi-lageret sitt i databasen. Uten gjentakelse avbrøt oppstarten ved den første låskonflikten, og containeren ble startet på nytt.

### Mål latenstid

Om lagringsmediene er problemet, kan måles direkte. Belastningen per lagringsmedium viser `iostat` fra pakken sysstat:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-x` | Utvidet statistikk per enhet, blant annet `r_await`/`w_await` (gjennomsnittlig ventetid i ms) og `%util` (utnyttelse). |
| `5` | Intervall i sekunder; første utdata viser gjennomsnittsverdier siden systemstart og er ikke representativt for den aktuelle situasjonen. |

</details>

Verdier for `w_await` i tresifret millisekundområde og `%util` nær 100 viser at lagringsmediet må køe opp oppgaver. Latenstiden for enkelte synkrone skriveoperasjoner kan kontrolleres med `dd` i databasens katalog:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `if=/dev/zero` | Kilde: nullbyte; bare skriveytelsen måles. |
| `of=./fsync-test` | Målfil i gjeldende katalog; denne må ligge på databasens volum. |
| `bs=4k` | Blokkstørrelse 4 KiB, tilsvarer den vanlige SQLite-sidestørrelsen. |
| `count=500` | Antall blokker, totalt 2 MB. |
| `oflag=dsync` | Hver blokk skrives synkront, sammenlignbart med én `fsync` per skriveoperasjon. |

</details>

`dd` rapporterer varighet og gjennomstrømning til slutt. 500 blokker på 2 til 5 sekunder tilsvarer 4 til 10 ms per synkrone skriveoperasjon og er normalt for harddisker. Hvis den samme testen tar 60 sekunder eller mer under en sikkerhetskopi, ligger årsaken til låsene i lagringsmediet.

## Løsning i det praktiske eksempelet

Grafana støtter begge tiltakene gjennom konfigurasjonsparametere i seksjonen `[database]`, som kan angis som miljøvariabler:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:13.0.2
    environment:
      GF_DATABASE_WAL: "true"
      GF_DATABASE_QUERY_RETRIES: "5"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `GF_DATABASE_WAL` | Tilsvarer `wal = true`: Grafana åpner SQLite-filen i WAL-modus. Standard er `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Tilsvarer `query_retries`: Antall gjentakelser av en spørring når SQLite rapporterer `database is locked`. Standard er `0`. |
| `grafana-data:/var/lib/grafana` | Named Volume på NAS-ens lokale lagringsmedium; der ligger `grafana.db` samt etter omleggingen `grafana.db-wal` og `grafana.db-shm`. |

</details>

For transaksjoner finnes dessuten `transaction_retries` (standard `5`), som ikke måtte tilpasses i det praktiske eksempelet. Etter omstart med begge innstillingene startet Grafana uten feil, Health-endepunktet rapporterte igjen `"database": "ok"`, og SQLITE_BUSY-meldingene fra Alert-State-History uteble. At omleggingen har virket, vises av de nye filene i volumet:

```bash
docker exec grafana ls -l /var/lib/grafana
```

Hvis `grafana.db-wal` og `grafana.db-shm` vises, arbeider databasen i WAL-modus.

På systemer med høy belastning kan ytterligere to tiltak vurderes. Alert-State-History kan skrives til Loki i stedet for databasen (`[unified_alerting.state_history]` med `backend = loki`), hvis Loki allerede kjører. For installasjoner med mange brukere eller flere Grafana-instanser anbefaler Grafana dessuten en ekstern database (MySQL eller PostgreSQL) i stedet for SQLite.

## Sikkerhetskopier uten låskonflikter

En filsikkerhetskopi av `grafana.db` under drift fanger opp en vilkårlig mellomtilstand. I WAL-modus mangler dessuten alle endringer som ennå ikke er overført uten `-wal`-filen. SQLite oppretter selv en konsistent kopi:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

Online Backup API kopierer databasen side for side og tar hensyn til pågående skriveoperasjoner. Alternativt oppretter `VACUUM INTO '/backup/grafana.db'` en kompakt kopi. Begge variantene kan kjøres før selve filsikkerhetskopieringen, som deretter sikkerhetskopierer kopien i stedet for live-filen. Tidsplanen for sikkerhetskopier og scrubs bør legges slik at de ikke overlapper hverandre eller sammenfaller med kjente belastningstopper fra andre tjenester.

## Sjekkliste

- Aktiver WAL-modus, forutsatt at all tilgang kommer fra samme maskin.
- Angi busy-timeout på hver tilkobling, med 5 til 10 sekunder som startverdi.
- Start skrivetansaksjoner med `BEGIN IMMEDIATE` og hold dem korte.
- Sørg for gjentakelser ved `SQLITE_BUSY` på applikasjonsnivå, i Grafana via `query_retries`.
- Kjør bare én skrivende prosess per databasefil.
- Lagre databasefiler bare på lokale filsystemer, aldri på SMB- eller NFS-delte områder.
- Mål skrivelatenstiden hvis låser forekommer ofte; unngå SMR-harddisker for databaser, eller flytt databaser til en SSD.
- Opprett sikkerhetskopier med `.backup` eller `VACUUM INTO` og hold dem tidsmessig atskilt fra scrubs.

## Kilder

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): betydningen av `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) og utvidede koder som `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): låsenivåene SHARED, RESERVED, PENDING og EXCLUSIVE i rollback-modus.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): virkemåte, sjekkpunkter, `wal_autocheckpoint` og begrensningen til lokale filsystemer.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` og `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): forskjellen mellom DEFERRED, IMMEDIATE og EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): hvorfor nettverksfilsystemer er uegnet for SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): feilaktige fillåser og `fsync` som årsaker til skadede databaser.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): konsistente kopier av en database under drift.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): parameterne `timeout` og `isolation_level` for `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, seksjonen database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): parameterne `wal`, `query_retries` og `transaction_retries`, samt tilordningen til `GF_`-miljøvariabler.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): oppbygningen av SMR-harddisker og deres oppførsel ved tilfeldige skriveoperasjoner.
