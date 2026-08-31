---
title: "Viktigaste kontrollerna för TotemoMail-administratörer: stoppa servern, kontrollera köer och rensa kontrollerat"
navTitle: "TotemoMail-kontroller"
description: "De viktigaste kontrollerna för drift av en totemomail-gateway: stoppa tjänsten via systemd och Tanuki-kontrollskriptet, räkna köinnehållet per repository, inspektera enskilda meddelanden, rensa kontrollerat och starta tjänsten igen."
date: "2026-08-28"
kategorie: "TotemoMail"
timeToRead: "9 min lästid"
themen:
  - totemomail
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "viktigaste-kontrollerna-for-totemomail-administratorer-stoppa-servern-kontrollera-koer-och"
translationId: "article-3a0a526ab6e38a06"
translationOf: totemomail-server-stoppen-queues-bereinigen
url: https://rafaelpfister.ch/sv/blog/viktigaste-kontrollerna-for-totemomail-administratorer-stoppa-servern-kontrollera-koer-och
translationSourceHash: bc887dcd4aa82db7e020247f75b86528f0fa331e1643c28a215a1638587197a6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:41:41.818Z
translationReview: automatic
---

# Viktigaste kontrollerna för TotemoMail-administratörer: stoppa servern, kontrollera köer och rensa kontrollerat

För driften av en totemomail-gateway (numera Kiteworks Email Protection Gateway) hör fyra arbetsmoment till grundverktygen: att stoppa tjänsten korrekt, inventera köinnehållet, inspektera enskilda meddelanden och rensa köerna kontrollerat innan tjänsten startas igen.

Dessa steg behövs både vid planerat underhåll och vid störningar, till exempel när en felaktig regel, ett ouppnåeligt mål eller ett belastningstest har fyllt köerna. Den här artikeln visar varje steg med de konkreta kommandona, inklusive frågan om hur tjänsten egentligen stoppas korrekt. Bearbetningsmodellen bakom detta (processors, repositories, filformat) beskrivs i artikeln [Förstå e-postroutning mellan totemomail och Exchange Online](/blog/totemomail-m365).

Alla sökvägar avser en installation under `/opt/totemomail` med tjänstanvändaren `totemo`. Anpassa sökvägarna till din miljö.

## Hur totemomail startas och stoppas

Innan du stoppar en tjänst bör du veta hur den körs. För totemomail är tre lager involverade:

- En **systemd-unit** `totemomail.service` som yttersta kontrollnivå.
- **Kontrollskriptet** `/opt/totemomail/bin/totemomail`, som anropas av uniten vid start och stopp.
- **Tanuki Java Service Wrapper**: en inbyggd `wrapper`-process som startar och övervakar den egentliga Java-processen och kan starta om den vid en krasch.

Du kan kontrollera uppbyggnaden på ditt system utan att behöva läsa unit-filen. `systemctl show` frågar systemd direkt efter egenskaperna och fungerar även om filen under `/etc/systemd/system/` bara kan läsas av root:

```bash
systemctl show totemomail.service -p Type -p User -p ExecStart -p ExecStop \
  -p KillMode -p TimeoutStopUSec --no-pager
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `show totemomail.service` | Visar unitens körningsegenskaper så som systemd har läst in dem |
| `-p <Property>` | Begränsar utdata till den angivna egenskapen; kan anges flera gånger |
| `--no-pager` | Skriver direkt till konsolen i stället för att öppna en sidvisare som `less` |

</details>

En typisk utdata ser ut så här:

```text
Type=oneshot
TimeoutStopUSec=1min 30s
ExecStart={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail start ; ... }
ExecStop={ path=/opt/totemomail/bin/totemomail ; argv[]=/opt/totemomail/bin/totemomail stop ; ... }
User=totemo
KillMode=control-group
```

Därifrån kan de viktiga egenskaperna utläsas: `systemctl stop totemomail` anropar kontrollskriptet med argumentet `stop`, väntar upp till 90 sekunder på ett korrekt avslut och avslutar därefter alla återstående processer i uniten via `KillMode=control-group`. Att stoppa via systemd är därmed likvärdigt med att anropa skriptet direkt, men städar dessutom upp om skriptet hänger sig.

Statusen `active (exited)` hos `systemctl status totemomail` är normal i denna uppbyggnad och inget fel: uniten är `Type=oneshot`, startskriptet avslutas efter starten och wrappen fortsätter att köra som en fristående daemon som systemd bara hanterar indirekt. Om tjänsten verkligen körs visas därför inte av unit-statusen utan av processlistan:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-e` | Visar alla processer, inte bara de för den egna sessionen |
| `-f` | Fullständigt utdataformat med komplett kommandorad |
| `grep -E 'wrapper\|TotemoBootStrapper'` | Filtrerar efter wrapper-processen och Java-huvudklassen |
| `grep -v grep` | Tar bort grep-processerna själva från träfflistan |

</details>

Vid normal drift visas två processer: den inbyggda `wrapper` (startad med `../conf/wrapper.conf` och PID-filen `totemomail.pid`) och Java-processen med huvudklassen `ch.totemo.bootstrapper.TotemoBootStrapper`. Om någon av de två saknas har tjänsten inte startat fullständigt.

## Steg 1: Stoppa tjänsten

Stoppa först tjänsten för alla arbeten med köerna. Så länge totemomail körs tar den emot meddelanden, bearbetar köerna och levererar; först stoppet fryser innehållet för analys.

```bash
sudo systemctl stop totemomail
```

Kontrollera sedan att wrapper- och Java-processen har avslutats:

```bash
ps -ef | grep -E 'wrapper|TotemoBootStrapper' | grep -v grep
```

Utdata måste vara tom. Dessutom försvinner PID-filen `/opt/totemomail/bin/totemomail.pid`. Om en process fortfarande kör efter att stopptidsgränsen har löpt ut avslutar systemd den via kontrollgruppen; kontrollera i så fall `journalctl -u totemomail` innan du fortsätter.

Glöm inte det föregående lagret: Under stoppet köas nya inkommande meddelanden hos det levererande systemet, till exempel i Exchange-kön eller hos det föregående reläet. Det är avsiktligt. Seriösa avsändare levererar automatiskt på nytt efter återstarten.

## Steg 2: Inventera köinnehållet

Totemomails köer är filbaserade e-postrepositories från underliggande Apache James. De finns under James-programkatalogen, här `/opt/totemomail/mailer/apps/james/var/mail/`. Varje underkatalog är ett repository och varje meddelande består av två filer: `*.FileStreamStore` innehåller det kompletta MIME-meddelandet, `*.FileObjectStore` det serialiserade statusobjektet med metadata.

En översikt över innehållet får du genom att räkna `FileObjectStore`-filerna per katalog:

```bash
for d in /opt/totemomail/mailer/apps/james/var/mail/*/; do \
  printf '%-22s %s\n' "$(basename "$d")" \
  "$(find "$d" -maxdepth 1 -name '*.FileObjectStore' | wc -l)"; \
done
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `for d in .../*/` | Itererar över alla repository-kataloger (det avslutande `/` begränsar till kataloger) |
| `printf '%-22s %s\n'` | Formaterar utdata i två kolumner; `%-22s` fyller ut namnet vänsterjusterat till 22 tecken |
| `basename "$d"` | Reducerar hela sökvägen till katalognamnet |
| `find "$d" -maxdepth 1` | Söker endast direkt i katalogen, utan underkataloger |
| `-name '*.FileObjectStore'` | Räknar en fil per meddelande; stream-motsvarigheten skulle dubblera antalet |
| `wc -l` | Räknar de hittade filerna |

</details>

Resultatet är en rad per kö med antalet meddelanden, till exempel:

```text
DBUnavailable          0
error                  12
incoming               121
outgoing               0
spool                  0
```

Standard-repositories innebär: `spool` innehåller mottagna men ännu obearbetade meddelanden, `incoming` meddelanden för intern leverans, `outgoing` utgående meddelanden, `error` misslyckade meddelanden och `DBUnavailable` meddelanden som parkerats på grund av en otillgänglig backend. Beroende på konfigurationen finns ytterligare repositories för särskilda rutter; de följer samma filschema.

Om `find` körs från en katalog som tjänstanvändaren inte har åtkomst till, exempelvis en annan användares hemkatalog efter `sudo -u totemo`, visas varningen `Failed to restore initial working directory` för varje anrop. Den är ofarlig och försvinner efter ett `cd ~`.

## Steg 3: Titta i meddelandena

Siffror räcker inte som beslutsunderlag. Innan du raderar något bör du veta vad som finns i köerna: oönskade meddelanden från en störning eller legitima e-postmeddelanden som ska levereras efter återstarten?

Filerna `FileStreamStore` är oförändrade RFC-822-meddelanden. De viktigaste rubrikerna kan därför läsas direkt:

```bash
for f in /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore; do \
  awk 'BEGIN{IGNORECASE=1} /^(From|To|Subject|Date):/{print} /^\r?$/{exit}' "$f"; \
  echo ---; \
done | less
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `BEGIN{IGNORECASE=1}` | Jämför rubriknamn utan hänsyn till versaler/gemener (GNU awk) |
| `/^(From\|To\|Subject\|Date):/{print}` | Skriver endast ut de fyra relevanta rubrikraderna |
| `/^\r?$/{exit}` | Avslutar vid den tomma raden mellan rubrik och brödtext; meddelandeinnehållet läses inte |
| `echo ---` | Avgränsningslinje mellan meddelandena |
| `less` | Bläddra i stället för att skrolla vid många meddelanden |

</details>

Vid stora mängder är fördelningen mer informativ än enskild visning. De vanligaste avsändarna visas med:

```bash
grep -him1 '^From:' /opt/totemomail/mailer/apps/james/var/mail/incoming/*.FileStreamStore \
  | sort | uniq -c | sort -rn | head
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-h` | Undertrycker filnamnen i utdata så att identiska avsändare slås ihop |
| `-i` | Ignorerar versaler/gemener |
| `-m1` | Endast första träffen per fil (rubriken, inte citerade `From:`-rader i brödtexten) |
| `sort \| uniq -c` | Grupperar identiska avsändarrader och räknar dem |
| `sort -rn \| head` | Sorterar fallande efter frekvens och visar de tio vanligaste |

</details>

Om en enskild avsändare eller ett enskilt ämne dominerar med hundratals kopior tyder det på en slinga eller ett felriktat massutskick; dessa meddelanden är kandidater för rensning. En titt på filernas tidsstämplar (`ls -lt`) avgränsar dessutom tidsperioden och visar om äldre, legitima meddelanden finns däremellan.

## Steg 4: Rensa kontrollerat

Först nu raderas något, och även nu med ett mellansteg: Flytta först innehållet till en säkerhetskatalog i stället för att radera det direkt. Resultatet för e-postdriften är detsamma (kön är tom), men steget går att återställa och enskilda legitima meddelanden kan senare läggas tillbaka från säkerhetskopian eller användas vidare som `.eml`.

```bash
mkdir -p /opt/totemomail/queue-backup-$(date +%F)
mv /opt/totemomail/mailer/apps/james/var/mail/incoming/* \
   /opt/totemomail/queue-backup-$(date +%F)/
```

Viktigt: Repository-katalogerna själva blir kvar, endast deras innehåll flyttas. Stream- och object-filen för ett meddelande hör ihop; den som bara tar bort en av dem lämnar kvar föräldralösa filer som orsakar fel i loggen vid nästa start.

När säkerhetskopian har kontrollerats eller innehållet utan tvekan saknar värde, exempelvis rena belastningstestmeddelanden, raderar du hela köinnehållet i alla repositories:

```bash
find /opt/totemomail/mailer/apps/james/var/mail/ -mindepth 2 -maxdepth 2 -type f \
  \( -name '*.FileStreamStore' -o -name '*.FileObjectStore' \) -delete
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-mindepth 2 -maxdepth 2` | Träffar endast filer direkt i repository-katalogerna, inte `var/mail` självt och inga djupare nivåer |
| `-type f` | Endast vanliga filer; katalogerna behålls |
| `\( -name ... -o -name ... \)` | Båda filtyperna för ett meddelande, stream och statusobjekt |
| `-delete` | Raderar träffarna direkt; kör först utan detta alternativ för att kontrollera träfflistan |

</details>

Kör därefter samma räkning som i steg 2: Alla repositories måste visa 0.

## Steg 5: Starta tjänsten igen

```bash
sudo systemctl start totemomail
```

Starten anropar kontrollskriptet med `start`, som daemoniserar wrappen; wrappen startar sedan Java-processen. Kontrollera båda via processlistan från första avsnittet och titta i loggfilerna under `/opt/totemomail/bin/`: `wrapper.log` loggar starten av wrappen och JVM, medan `console.log` och `console.err` innehåller utdata från själva programmet.

Avsluta med ett funktionstest med ett enskilt testmeddelande genom gatewayen innan det vanliga e-postflödet släpps på igen. Och om en regel eller en e-postslinga hade fyllt köerna: åtgärda först orsaken och tillåt sedan trafiken igen. Annars börjar inventeringen av köinnehållet om från början.

## Sammanfattning

| Steg | Kommando | Kontroll |
|---|---|---|
| Stoppa | `sudo systemctl stop totemomail` | `ps`-filtret tomt, PID-filen borta |
| Räkna innehåll | `find`-slinga över `var/mail/*/` | Antal per repository |
| Inspektera | `awk`-utdrag av rubriker, `grep`-avsändarstatistik | Separera oönskade från legitima meddelanden |
| Rensa | `mv` till säkerhetskopia, sedan `find ... -delete` | Räkningen visar 0 överallt |
| Starta | `sudo systemctl start totemomail` | Processer, `wrapper.log`, testmeddelande |

## Källor

1.  [Apache James Server 2: Provided Mailets](https://james.apache.org/server/2/provided_mailets.html): Dokumentation av mailets och repositories som totemomails köstruktur bygger på.

2.  [Tanuki Software: Java Service Wrapper](https://wrapper.tanukisoftware.com/doc/english/introduction.html): Hur wrappen fungerar, som startar och övervakar totemomails Java-process, inklusive PID-fil och `wrapper.conf`.

3.  [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html): Betydelsen av `Type=oneshot`, `ExecStop` och `TimeoutStopSec` för units som anropar ett externt kontrollskript.

4.  [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html): `KillMode=control-group` som skyddsåtgärd som avslutar återstående processer i uniten efter stoppskriptet.

5.  [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322): Strukturen för meddelanderubrikerna som läses ut när `FileStreamStore`-filerna inspekteras.
