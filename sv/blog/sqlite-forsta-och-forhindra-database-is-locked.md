---
title: "SQLite: Förstå och förhindra «database is locked»"
navTitle: "SQLite-låsningar"
description: "Varför SQLite svarar med SQLITE_BUSY och «database is locked», hur WAL-läge, busy_timeout, korta transaktioner och återförsök undviker låsningar och varför långsamma lagringsenheter och I/O-belastning på en NAS kan utlösa problemet. Med ett praktiskt fall från Grafana."
date: "2026-11-12"
kategorie: "Databaser"
timeToRead: "11 min lästid"
themen:
  - datenbanken
produkte:
  - "sqlite"
protokolle:
  - "troubleshooting"
  - "container"
slug: "sqlite-forsta-och-forhindra-database-is-locked"
translationId: "article-2a328dabf9164dae"
translationOf: sqlite-database-locked-wal-io-last
url: https://rafaelpfister.ch/sv/blog/sqlite-forsta-och-forhindra-database-is-locked
translationSourceHash: 121b1c0f6dc58cb2249e9e0276937397b52725621cd1553b455a37069f5958de
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T13:52:59.808Z
translationReview: automatic
---

SQLite används i många tjänster som drivs på egen hand: Grafana, Home Assistant, Vaultwarden, Uptime Kuma, Jellyfin och många andra program lagrar som standard sina data i en enda fil. Det fungerar tills meddelandet `database is locked` visas i loggen. Det anger att en åtkomst inte fick något lås inom den tillåtna väntetiden; själva databasfilen är intakt. Om detta sker sällan eller hela tiden beror på tre saker: journalläget, programmets beteende vid konflikter och lagringsenhetens latens.

## Praktiskt fall: Grafana på en NAS

Grafana kördes som en Docker-container på en Synology-NAS, och SQLite-filen låg i en Named Volume. På samma NAS kördes Prometheus, Loki, Paperless-ngx och flera restic-säkerhetskopior. Lagringsenheterna var hårddiskar med Shingled Magnetic Recording (SMR).

Först rapporterade Grafanas health-endpoint `"database": "failing"`. I loggen fanns med korta intervall poster av följande slag:

```text
level=error msg="Failed to save history batch"
  error="database is locked (5) (SQLITE_BUSY)"
```

Meddelandet kommer från alert-state-historiken: Grafana skriver som standard tillståndsändringar för alert-regler som annotationer i sin egen databas. Med många regler uppstår därför fortlöpande små skrivtransaktioner. Efter en omstart av containern kunde Grafana inte längre starta och startade om i en loop:

```text
Failed to stop all modules ... kv get:
  [sqlstore.max-retries-reached] retry 1: database is locked
```

Fallet löstes med två inställningar i `docker-compose.yml`: aktivera WAL-läge och låta Grafana försöka igen flera gånger vid låsningar. Varför just dessa två värden fungerar framgår av hur SQLite-låsningar fungerar.

## Hur SQLite låser

SQLite har ingen serverprocess som samordnar åtkomst. Varje anslutning öppnar filen själv och reglerar den gemensamma åtkomsten via operativsystemets fillås. Därav följer den viktigaste egenskapen: **Endast en anslutning får skriva åt gången.** Detta gäller i alla journallägen.

I det klassiska rollback-journalläget (standard, `journal_mode=DELETE`) går en skrivtransaktion igenom flera låsnivåer:

| Lås | Betydelse |
|---|---|
| `SHARED` | Anslutningen läser. Godtyckligt många läsare samtidigt är tillåtna. |
| `RESERVED` | Anslutningen vill skriva och förbereder ändringar. Läsare får fortsätta läsa, men inte en andra skrivare. |
| `PENDING` | Anslutningen väntar på att pågående läsåtkomster ska avslutas. Nya läsare avvisas. |
| `EXCLUSIVE` | Anslutningen skriver till databasfilen. Ingen annan får läsa eller skriva. |

För commit behöver skrivaren alltså det exklusiva låset, och det får den först när ingen läsare längre är aktiv. En långvarig läsåtkomst håller därmed upp varje skrivare, och under commit blockeras alla läsare.

Om en anslutning inte får det lås som behövs returnerar SQLite felkoden `SQLITE_BUSY` (5) med texten `database is locked`. Detta ska skiljas från `SQLITE_LOCKED` (6, `database table is locked`): Denna kod gäller konflikter inom samma anslutning eller i shared-cache-läge och har andra orsaker.

Avgörande är hur länge SQLite väntar före detta meddelande. Utan konfiguration väntar SQLite inte alls: Standardvärdet för `busy_timeout` är 0, och meddelandet kommer direkt så snart en annan anslutning håller låset. Många bibliotek anger ett eget värde; Pythons modul `sqlite3` väntar exempelvis 5 sekunder.

## WAL-läge: separera läsare och skrivare

I Write-Ahead-Log-läge (`journal_mode=WAL`) lägger SQLite till ändringar i en separat fil `<name>-wal` i stället för att skriva dem direkt till databasfilen. Läsare ser läget vid början av sin lästransaktion och läser då från databasfilen och WAL. Detta ger två fördelar:

- Läsare blockerar inte skrivaren, och skrivaren blockerar inte läsarna.
- En commit är en tilläggsskrivning till en fil i stället för flera spridda skrivoperationer med journal. Per transaktion uppstår mindre I/O och skrivlåset hålls under kortare tid.

WAL-läget ändrar inte att endast en skrivare kan vara aktiv åt gången. Men det förkortar tiden som låset hålls och eliminerar konflikter mellan läsning och skrivning. För ett program som Grafana, som samtidigt levererar dashboards och lagrar alert-tillstånd, är detta den mest effektiva åtgärden.

Läget ställs in en gång och lagras i filen:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `journal_mode = WAL` | Växlar till Write-Ahead-Log. Inställningen är beständig; nya anslutningar öppnar filen automatiskt i WAL-läge. |
| `busy_timeout = 5000` | Väntetid i millisekunder under vilken SQLite upprepade gånger försöker få ett upptaget lås. Gäller bara den aktuella anslutningen och måste anges på nytt vid varje öppning. |
| `synchronous = NORMAL` | I WAL-läge inget `fsync` per commit, utan endast vid checkpoints. Databasen förblir konsekvent vid strömavbrott; de senast bekräftade transaktionerna kan gå förlorade. Gäller per anslutning. |

</details>

WAL-läget har förutsättningar. Alla processer som kommer åt filen måste köras på samma dator, eftersom de samordnar sig via en shared-memory-fil `<name>-shm`. Katalogen behöver skrivrättigheter för de två tilläggsfilerna, även för processer som bara läser. Vid säkerhetskopiering måste filerna dessutom hanteras tillsammans (mer om det längre ned).

## Checkpoints: när WAL växer

WAL växer tills en checkpoint överför ändringarna till den egentliga databasfilen. SQLite utlöser den automatiskt så snart WAL omfattar cirka 1000 sidor (`wal_autocheckpoint`, vid 4-KiB-sidor ungefär 4 MB). Checkpointen skriver sidorna till sina positioner i databasfilen och anropar därefter `fsync`. På en långsam lagringsenhet är just detta steg det som tar längst tid.

En checkpoint kan bara överföra sidor så länge ingen läsare behöver ett äldre tillstånd. Om minst en lästransaktion alltid pågår förblir checkpointen ofullständig och WAL fortsätter att växa. Ett WAL på flera hundra megabyte är en indikation på långvariga lästransaktioner eller checkpoints som inte blir klara på grund av I/O-belastning. Det kan kontrolleras så här:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

Utdata består av tre tal: om checkpointen blockerades (`1` = ja), hur många sidor som finns i WAL och hur många av dem som överfördes. Med `TRUNCATE` kortas WAL därefter ned till noll byte, förutsatt att checkpointen var fullständig.

## Busy-timeout och återförsök

Med en busy-timeout väntar SQLite vid en konflikt i stället för att avbryta direkt. Felet visas först när låset hålls längre än den inställda tiden. För program med få, korta skrivoperationer räcker 5 till 10 sekunder.

Det finns ett fall där busy-timeout inte gäller. Om en transaktion börjar med läsning (`BEGIN` utan tillägg, alltså `DEFERRED`) och senare vill skriva medan en annan skrivare under tiden har slutfört en commit, returnerar SQLite direkt `SQLITE_BUSY`. Transaktionen arbetar på ett föråldrat tillstånd och kan inte bli en skrivtransaktion; väntan skulle inte ändra detta. I rollback-läge finns en jämförbar situation när två anslutningar samtidigt vill växla från `SHARED` till ett skrivlås.

Lösningen är att markera skrivtransaktioner som sådana från början:

```sql
BEGIN IMMEDIATE;
UPDATE alert_state SET state = 'firing' WHERE id = 42;
COMMIT;
```

`BEGIN IMMEDIATE` begär skrivlåset vid starten. Om det är upptaget fungerar busy-timeout normalt. I Python ser det ut så här:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `"app.db"` | Sökväg till databasfilen. |
| `timeout=10.0` | Busy-timeout i sekunder; motsvarar `PRAGMA busy_timeout = 10000`. |
| `isolation_level=None` | Stänger av modulens automatiska transaktionsstyrning så att `BEGIN IMMEDIATE` och `COMMIT` kan anges explicit. |

</details>

Återförsök på programnivå kompletterar busy-timeout. De fångar `SQLITE_BUSY`, väntar kort och kör om hela transaktionen. Det täcker även fallet med föråldrat lästillstånd, eftersom omkörningen börjar med ett nytt tillstånd. Grafana erbjuder egna inställningar för detta (se avsnittet om lösningen).

## Korta transaktioner och en skrivare

Hur länge ett skrivlås hålls bestäms av programmet. Tre regler håller tiden kort:

- **Inget arbete inom transaktionen som inte har med databasen att göra.** HTTP-anrop, filåtkomster eller beräkningar hör hemma före `BEGIN` eller efter `COMMIT`.
- **Samla många små skrivoperationer.** 500 enskilda commits kostar 500 låscykler och i rollback-läge 500 gånger flera `fsync`-anrop. En transaktion med 500 rader håller låset en gång och något längre.
- **Stäng läscursorer.** Ett frågeresultat som inte har hämtats helt håller lästransaktionen öppen och blockerar checkpointen i WAL-läge.

Där flera trådar eller processer skriver är en enda skrivare med kö den mest tillförlitliga lösningen: En tråd eller process äger den enda skrivande anslutningen, och alla andra lämnar över uppdrag till den. Låskonflikter mellan skrivare försvinner då helt, medan läsare fortsätter att arbeta parallellt i WAL-läge.

Flera containrar som öppnar samma SQLite-fil via en delad volym är däremot en vanlig orsak till låsningar. Varje container är en egen skrivare, och ingen känner till den andra. Om ett program skalas horisontellt är en databasserver som PostgreSQL eller MySQL rätt val.

## Inga nätverksfilsystem

SQLite förlitar sig på att fillås fungerar korrekt och atomärt och att ett `fsync` faktiskt skriver data till lagringsenheten. Med NFS, SMB/CIFS och många FUSE-filsystem garanteras inget av detta. SQLite-utvecklarna avråder uttryckligen från att använda en databas via ett nätverksfilsystem från flera datorer. Följderna sträcker sig från permanenta låsmeddelanden till en skadad databas, och WAL-läget fungerar där i grunden inte eftersom shared-memory-samordningen endast är möjlig lokalt.

För Docker innebär det: En Named Volume eller en bind-mount på en lokal lagringsenhet är lämplig. En bind-mount på en monterad SMB- eller NFS-resurs är det inte, även om den verkar fungera vid normal drift.

## I/O-belastning: varför låset plötsligt tar för lång tid

Låset är bara ett symtom. Det blir ett problem när det hålls längre än busy-timeout tillåter, och det beror nästan alltid på lagringsenhetens skrivlatens.

Varje commit avslutas med ett `fsync`. Först när lagringsenheten bekräftar att data har skrivits permanent frigör SQLite låset. På en SSD tar det under en millisekund, på en lugn hårddisk några millisekunder. Om lagringsenheten samtidigt måste hantera andra uppdrag väntar `fsync` i kön, och låset kvarstår så länge. Alla andra skrivare väntar också tills deras busy-timeout löper ut.

Flera faktorer driver upp denna latens på en NAS:

- **SMR-hårddiskar.** Vid Shingled Magnetic Recording överlappar spåren varandra. Slumpmässiga skrivoperationer hamnar först i ett cacheområde och sorteras senare om till de överlappande zonerna. Om cacheområdet fylls vid kontinuerlig belastning ökar skrivlatensen från millisekunder till flera sekunder. Databaser med många små, spridda skrivoperationer är den mest ogynnsamma belastningsprofilen för detta.
- **Parallella säkerhetskopior.** restic, Hyper Backup eller rsync läser stora mängder data och konkurrerar med varje `fsync` om samma diskar.
- **Scrubbing och RAID-konsistenskontroller.** En lagringsenhetsscrub läser hela datamängden och körs schemalagt på många NAS-system, ofta i flera timmar.
- **Andra databaser på samma volym.** Prometheus och Loki skriver kontinuerligt block och komprimerar dem periodiskt. Dessa compactions skapar belastningstoppar.
- **Checkpoints i WAL-läge.** Checkpointen skriver många sidor på en gång och avslutas med `fsync`. Om den hamnar i en period med hög belastning förlängs den därefter.

I det praktiska fallet sammanföll flera av dessa faktorer: SMR-hårddiskar, säkerhetskopior och en övervakningsmiljö med flera skrivande tjänster. Grafana kördes i rollback-läge utan återförsök. Varje commit i alert-state-historiken krävde flera `fsync`-anrop, och vart och ett väntade bakom den övriga belastningen. Vid start läser Grafana bland annat värden från sin nyckel-värde-lagring i databasen. Utan återförsök avbröts starten vid den första låskonflikten, och containern startades om.

### Mät latens

Om lagringsenheterna är problemet kan mätas direkt. Belastningen per lagringsenhet visas av `iostat` från paketet sysstat:

```bash
iostat -x 5
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-x` | Utökad statistik per enhet, bland annat `r_await`/`w_await` (genomsnittlig väntetid i ms) och `%util` (belastning). |
| `5` | Intervall i sekunder; den första utdata visar genomsnittsvärden sedan systemstarten och är inte representativ för det aktuella läget. |

</details>

Värden för `w_await` i tresiffriga millisekundintervall och `%util` nära 100 visar att lagringsenheten måste köa uppdrag. Latensen för enskilda synkrona skrivoperationer kan kontrolleras med `dd` i databasens katalog:

```bash
dd if=/dev/zero of=./fsync-test bs=4k count=500 \
   oflag=dsync
rm ./fsync-test
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `if=/dev/zero` | Källa: nollbyte, endast skrivprestandan mäts. |
| `of=./fsync-test` | Målfil i aktuell katalog; den måste ligga på databasens volym. |
| `bs=4k` | Blockstorlek 4 KiB, motsvarar den vanliga SQLite-sidstorleken. |
| `count=500` | Antal block, totalt 2 MB. |
| `oflag=dsync` | Varje block skrivs synkront, jämförbart med ett `fsync` per skrivoperation. |

</details>

`dd` rapporterar i slutet tid och genomströmning. 500 block på 2 till 5 sekunder motsvarar 4 till 10 ms per synkron skrivoperation och är normalt för hårddiskar. Om samma test under en säkerhetskopia tar 60 sekunder eller längre ligger orsaken till låsningarna hos lagringsenheten.

## Lösning i det praktiska fallet

Grafana stöder båda åtgärderna via konfigurationsparametrar i avsnittet `[database]`, som kan anges som miljövariabler:

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
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `GF_DATABASE_WAL` | Motsvarar `wal = true`: Grafana öppnar SQLite-filen i WAL-läge. Standard är `false`. |
| `GF_DATABASE_QUERY_RETRIES` | Motsvarar `query_retries`: Antal återförsök för en fråga när SQLite rapporterar `database is locked`. Standard är `0`. |
| `grafana-data:/var/lib/grafana` | Named Volume på NAS:ens lokala lagringsenhet; där finns `grafana.db` samt efter ändringen `grafana.db-wal` och `grafana.db-shm`. |

</details>

För transaktioner finns dessutom `transaction_retries` (standard `5`), som inte behövde anpassas i det praktiska fallet. Efter omstarten med båda inställningarna startade Grafana utan fel, health-endpointen rapporterade åter `"database": "ok"`, och SQLITE_BUSY-meddelandena från alert-state-historiken uteblev. Att ändringen har fått effekt visas av de nya filerna i volymen:

```bash
docker exec grafana ls -l /var/lib/grafana
```

Om `grafana.db-wal` och `grafana.db-shm` visas arbetar databasen i WAL-läge.

På system med hög belastning kan ytterligare två åtgärder övervägas. Alert-state-historiken kan skrivas till Loki i stället för databasen (`[unified_alerting.state_history]` med `backend = loki`), om Loki redan körs. För installationer med många användare eller flera Grafana-instanser rekommenderar Grafana också en extern databas (MySQL eller PostgreSQL) i stället för SQLite.

## Säkerhetskopiera utan låskonflikter

En filbaserad säkerhetskopia av `grafana.db` under drift fångar ett godtyckligt mellanläge. I WAL-läge saknas dessutom alla ännu ej överförda ändringar utan `-wal`-filen. SQLite kan själv skapa en konsekvent kopia:

```bash
sqlite3 /var/lib/grafana/grafana.db \
  ".backup /backup/grafana-$(date +%F).db"
```

Online-Backup-API:t kopierar databasen sida för sida och tar samtidigt hänsyn till pågående skrivoperationer. Alternativt skapar `VACUUM INTO '/backup/grafana.db'` en kompakt kopia. Båda varianterna kan köras före den egentliga filbaserade säkerhetskopieringen, som sedan säkerhetskopierar kopian i stället för live-filen. Planera säkerhetskopior och scrubs så att de inte överlappar varandra och inte sammanfaller med kända belastningstoppar från andra tjänster.

## Checklista

- Aktivera WAL-läge, förutsatt att all åtkomst kommer från samma dator.
- Ange busy-timeout för varje anslutning, med 5 till 10 sekunder som startvärde.
- Börja skrivtransaktioner med `BEGIN IMMEDIATE` och håll dem korta.
- Förse programnivån med återförsök vid `SQLITE_BUSY`, i Grafana via `query_retries`.
- Kör endast en skrivande process per databasfil.
- Lagra endast databasfiler på lokala filsystem, aldrig på SMB- eller NFS-resurser.
- Mät skrivlatensen när låsningar uppträder ofta; undvik SMR-hårddiskar för databaser eller flytta databaser till en SSD.
- Skapa säkerhetskopior via `.backup` eller `VACUUM INTO` och separera dem tidsmässigt från scrubs.

## Källor

1.  [SQLite: Result and Error Codes](https://www.sqlite.org/rescode.html): betydelsen av `SQLITE_BUSY` (5), `SQLITE_LOCKED` (6) och utökade koder som `SQLITE_BUSY_SNAPSHOT`.

2.  [SQLite: File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html): låsnivåerna SHARED, RESERVED, PENDING och EXCLUSIVE i rollback-läge.

3.  [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html): funktion, checkpoints, `wal_autocheckpoint` och begränsningen till lokala filsystem.

4.  [SQLite: PRAGMA Statements](https://www.sqlite.org/pragma.html): `journal_mode`, `busy_timeout`, `synchronous` och `wal_checkpoint`.

5.  [SQLite: BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html): skillnaden mellan DEFERRED, IMMEDIATE och EXCLUSIVE.

6.  [SQLite: SQLite Over a Network, Caveats and Considerations](https://www.sqlite.org/useovernet.html): varför nätverksfilsystem är olämpliga för SQLite.

7.  [SQLite: How To Corrupt An SQLite Database File](https://www.sqlite.org/howtocorrupt.html): felaktiga fillås och `fsync` som orsaker till skadade databaser.

8.  [SQLite: SQLite Backup API](https://www.sqlite.org/backup.html): konsekventa kopior av en databas under drift.

9.  [Python: sqlite3 DB-API 2.0 interface](https://docs.python.org/3/library/sqlite3.html): parametrarna `timeout` och `isolation_level` för `sqlite3.connect()`.

10.  [Grafana: Configure Grafana, avsnittet database](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#database): parametrarna `wal`, `query_retries` och `transaction_retries` samt mappningen till `GF_`-miljövariabler.

11.  [Wikipedia: Shingled magnetic recording](https://en.wikipedia.org/wiki/Shingled_magnetic_recording): uppbyggnaden av SMR-hårddiskar och deras beteende vid slumpmässiga skrivoperationer.
