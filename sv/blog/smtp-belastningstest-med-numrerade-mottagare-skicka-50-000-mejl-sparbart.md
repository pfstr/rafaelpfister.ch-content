---
title: "SMTP-belastningstest med numrerade mottagare: skicka 50 000 mejl spårbart"
navTitle: "Numrerade belastningstester"
description: "Ett belastningstest är bara så bra som dess utvärdering. Med alternativet -N numrerar smtp-source varje mejl via mottagaradressen utan att offra genomströmningen. Så byggs körningen med 50 000 mejl upp, hur många sessioner som är rimliga och hur saknade nummer hittas automatiskt."
date: "2026-08-27"
kategorie: "SMTP och mejlflöde"
timeToRead: "8 min lästid"
themen:
  - smtp-lasttests
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
slug: "smtp-belastningstest-med-numrerade-mottagare-skicka-50-000-mejl-sparbart"
translationId: "article-57f09c758baf6e1e"
translationOf: smtp-lasttest-nummerierte-empfaenger
url: https://rafaelpfister.ch/sv/blog/smtp-belastningstest-med-numrerade-mottagare-skicka-50-000-mejl-sparbart
translationSourceHash: a2ec75884c06a6d736ea9b5895211ddc4cbba252c7ddf491752e1bec5ab1a24d
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:22:24.880Z
translationReview: automatic
---

# SMTP-belastningstest med numrerade mottagare: skicka 50'000 mejl spårbart

Den som kör ett belastningstest med 50'000 mejl vill efteråt kunna besvara två frågor: Kom alla fram, och om inte, vilka saknas? Med identiska testmejl går det bara att räkna, och en skillnad mellan 49'987 och 50'000 säger inget om när och var de 13 saknade meddelandena försvann. Om varje mejl däremot har ett löpnummer blir räknandet en avstämning: Varje nummer går att hitta separat i målsystemets loggar, luckor visar tidpunkten för förlusten och leveransordningen kan kontrolleras.

Den vanliga reflexen är ett skript som räknar upp ämnesraden. Det fungerar, men kostar genomströmning, eftersom lastgeneratorn `smtp-source` från Postfix-paketet anger ämnesraden fast per anrop, och en slinga med ett anrop per mejl tvingar fram en egen anslutningsetablering för varje meddelande. Den bättre meddelandeidentifieringen är redan inbyggd: Alternativet `-N` numrerar mottagaradressen för varje meddelande, inom ett enda anrop med parallella sessioner. För utvärderingen är mottagaradressen lika användbar som ämnesraden, eftersom den finns i varje spårningslogg.

Den här testuppsättningen skickar, till skillnad från ett rent loopback-funktionstest, till ett annat system över nätverket. Om Postfix inte är installerat på källsystemet visar artikeln [smtp-source utan Postfix-installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation), hur du packar upp verktygen från RPM-paketet.

## De viktigaste alternativen för smtp-source

Som orientering följer här de alternativ som förekommer i artikeln, fritt översatta från man-sidan:

| Alternativ | Betydelse |
|---|---|
| `-s n` | Antal parallella SMTP-sessioner (standard: 1) |
| `-m n` | Totalt antal meddelanden som ska skickas (standard: 1) |
| `-l n` | Meddelandetextens storlek i byte, utan header |
| `-f adresse` | Avsändaradress |
| `-t adresse` | Mottagaradress (standard: `foo@hostname`) |
| `-S text` | Ämnesrad, fast för alla meddelanden i anropet |
| `-F datei` | Skickar header och brödtext oförändrade från en fil; ersätter `-l` och `-S` |
| `-N` | Numrerar mottagaradressen per meddelande (räknare per process; placering och startvärde beror på version, se nedan) |
| `-r n` | Antal mottagare per meddelande (standard: 1), adressbildning som med `-N` |
| `-d` | Koppla inte från efter ett meddelande, utan skicka nästa via samma anslutning |
| `-c` | Visa löpande räknare som ökar med varje slutfört `DATA` |
| `-w n` | Fast väntetid på n sekunder mellan meddelanden (per session) |
| `-v` | Utförlig utmatning för felsökning |
| `host:port` | Mål för inlämning via TCP; utan portangivelse används standardporten smtp |

Den fullständiga listan, inklusive alternativ för TLS, LMTP och tidsmätning, finns i man-sidan för `smtp-source(1)`; motsvarigheten på mottagarsidan är `smtp-sink(1)` och används vid utvärderingen längre ned.

## Så numrerar -N mottagarna

`-N` aktiverar en räknare per process som byggs in i mottagaradressen. Tre egenskaper avgör testuppsättningen, och alla tre kan läsas i källkoden för `smtp-source.c`:

För det första beror den exakta adressformen på Postfix-versionen. Postfix 3.5, som levereras med RHEL 8, placerar numret framför hela adressen (`RCPT TO:<%d%s>`): Av `-t test@example.com` blir `1test@example.com`, `2test@example.com` och så vidare, och räknaren börjar på 1. Aktuella Postfix-versioner lägger i stället till numret i slutet av local-part och börjar på 0 (`test0@` till `test49999@`); för denna variant rekommenderar man-sidan plusadressering (`-t 'test+@example.com'` blir `test+0@` och följande), så att ett målsystem med underadressering tilldelar allt till samma brevlåda. Kontrollera formen före den stora körningen med en handfull mejl mot en `smtp-sink` eller i målets logg; både referensmängden och utvärderingens sökmönster beror på detta.

För det andra gäller räknaren för hela processen och delas av alla parallella sessioner. Med `-s 8` tilldelar de åtta sessionerna numren gemensamt, och varje nummer förekommer exakt en gång. Ordningen mellan sessionerna är inte deterministisk, men numermängdens fullständighet garanteras.

För det tredje går startvärdet inte att konfigurera: 1 i Postfix 3.5, 0 i aktuella versioner. De 50'000 mejlen får alltså numren 1 till 50'000 respektive 0 till 49'999, och referensmängden för avstämningen måste anpassas till detta.

## Testkörningen i ett anrop

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Alternativ | Effekt |
|---|---|
| `-c` | Löpande räknare över slutförda leveranser som enradig förloppsindikator |
| `-d` | Anslutningarna hålls öppna för alla meddelanden; utan `-d` skapas en ny anslutning per meddelande |
| `-N` | Mottagarnumrering: lägger till räknaren per process i local-part |
| `-s 8` | Åtta parallella SMTP-sessioner |
| `-m 50000` | Totalt antal meddelanden, fördelat över sessionerna |
| `-l 5120` | Meddelandestorlek i byte (utan header), här 5 KB |
| `-f` | Avsändaradress |
| `-t` | Basadress för mottagare; `-N` gör den till `1test@` till `50000test@` (Postfix 3.5) respektive `test0@` till `test49999@` (aktuella versioner) |
| `gateway.example.com:25` | Målhost och port |

`-d` är avgörande för lastbilden: Utan detta alternativ kopplar `smtp-source` från efter varje meddelande och skapar en ny anslutning för nästa; med `-d` hålls de åtta anslutningarna öppna och levererar alla meddelanden i följd, som en massavsändare gör.

Det från funktionstester välkända `-v` utelämnas medvetet: Det loggar varje enskild SMTP-dialog från `HELO` till `QUIT` och skapar hundratusentals loggrader vid 50'000 mejl utan något mervärde för utvärderingen. `-c` ger i stället den sammanfattning som gör det möjligt att följa körningens förlopp live. Den totala tiden för hastighetsberäkningen fås med ett inledande `time`.

Förutsättningen för hela tillvägagångssättet är att målsystemet accepterar de genererade adresserna. En `smtp-sink`, en catch-all-domän, en kasseringsdomän från leverantören eller en gateway som löser upp mottagare först efter godkännandet uppfyller detta. Om målet däremot kontrollerar varje mottagare mot en katalog, avvisar det de numrerade adresserna och endast ämnesvarianten återstår.

## Ange egna headers

Vissa belastningstester behöver en egen header, till exempel som markör som gatewayen identifierar testmejlen med eller för att en regel ska träffa. `smtp-source` har inget alternativ för detta, men `-F` läser ett helt förformaterat meddelande från en fil, där varje önskad header kan anges. Filen består av headerraderna, en tom rad och brödtexten, där alla rader avslutas med `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

| Alternativ | Effekt |
|---|---|
| `-F datei` | Skickar header och brödtext oförändrade från filen; ersätter det genererade meddelandeinnehållet |

Detta får två följder: `-F` ersätter `-l` och `-S`, eftersom storlek och ämnesrad nu kommer från filen (båda måste därför finnas där). `-N` fortsätter däremot att fungera och mottagarna numreras vidare; headern är identisk i alla meddelanden eftersom den kommer från den fasta filen.

## Hur många sessioner?

Det mest tillförlitliga sättet att fastställa rätt antal sessioner är att mäta, med exakt samma alternativ som i den planerade huvudkörningen: samma meddelandekälla (samma `-F`-fil respektive samma `-l`), samma avsändare och samma mål. Endast mängden minskas till 2'000 per steg, och `-s` varieras. En kort kalibreringskörning med stigande antal sessioner visar när fler sessioner inte längre ger något:

```bash
for s in 1 2 4 8 16 32; do
  t0=$(date +%s%N)
  smtp-source -d -N -s "$s" -m 2000 -F lasttest.eml \
    -f lasttest@example.com -t '@blackhole.example.com' \
    gateway.example.com:25
  t1=$(date +%s%N)
  echo "$s Sessions: $(( 2000000000000 / (t1 - t0) )) Mails/s"
done
```

Två detaljer i anropet: `-c` utelämnas medvetet här, så att inga löpande räknarutskrifter visas mellan mätraderna; slingan ger exakt en resultatrad per steg. Och den tomma local-part i `-t` fungerar väl ihop med numreringen för en kasseringsdomän: Med den inledande räknaren i Postfix 3.5 skapas rent numeriska mottagaradresser (`1@blackhole.example.com`, `2@…`), vilket gör utvärderingen i loggarna överskådlig.

I detalj sker följande: Den yttre slingan går igenom sessionsantalet 1 till 32 i fördubblingssteg. Före och efter varje körning sparar `date +%s%N` den aktuella tiden som ett stort tal, det vill säga Unix-sekunder direkt följda av nanosekundsdelen. Däremellan skickar `smtp-source` 2'000 meddelanden (innehåll, header och storlek kommer från `-F`-filen) över respektive antal parallella anslutningar som hålls öppna tack vare `-d`; slingan väntar tills anropet är helt klart. Raden med `echo` räknar om tidsskillnaden till en hastighet: 2'000 mejl dividerat med körtiden i sekunder, där körtiden anges i nanosekunder. Ur 2'000 gånger 10⁹ fås därmed konstanten `2000000000000`. Bash-aritmetiken `$(( ))` räknar med heltal och kapar decimaler, vilket är tillräckligt noggrant för denna mätning.

Tre praktiska anmärkningar: `%N` ger nanosekunder endast med GNU date (vilket gäller RHEL och de flesta Linux-system; BusyBox och macOS stöder det inte). Hela genomgången skickar 6 × 2'000 = 12'000 mejl, och även dessa behöver en kontrollerad mottagaradress, medan numreringen med `-N` börjar om från startvärdet i varje anrop. Om ett anrop av `smtp-source` avbryts med ett felmeddelande är hastigheten på den raden meningslös; åtgärda först orsaken och mät sedan igen.

Den förväntade utmatningen är en rad per steg. Med påhittade men typiska exempelvärden ser det ut så här:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

Tolkningen är följande: Så länge hastigheten ungefär fördubblas med antalet sessioner döljer de parallella sessionerna väntetiden på svaren från målet; flaskhalsen är då förbindelsens latens, inte kapaciteten. Från den punkt där kurvan planar ut (i exemplet mellan 8 och 16 sessioner) är antingen målsystemet mättat eller källan vid sin gräns. Välj det minsta värde där hastigheten inte längre ökar nämnvärt, alltså 8 till 16 i exemplet; fler sessioner ökar då bara lasten genom parallellitet, inte genomströmningen. För huvudkörningen med 50'000 mejl går det också att uppskatta den förväntade tiden direkt från den uppmätta hastigheten: vid 71 mejl/s cirka 12 minuter.

## Utvärdering på mottagarsidan

Om en egen testmottagare finns på målsystemet sköter `smtp-sink` loggningen direkt:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

| Alternativ | Effekt |
|---|---|
| `-c` | Löpande räknare i stället för hela SMTP-dialogen |
| `-d "mails/…"` | För sink: dumpning, inte anslutningshållning. Skriver varje accepterat meddelande till en egen fil (namnmönster via strftime), inklusive en `X-Rcpt-Args`-header med mottagaradressen |
| `0.0.0.0:2525` | Lyssnar på alla gränssnitt på port 2525 |
| `200` | Backlog: maximal längd på kön med väntande anslutningar enligt listen(2) |

Efter körningen extraherar du de mottagna numren och jämför dem med referensmängden. Eftersom numren saknar inledande nollor fylls båda listorna ut till en fast sifferlängd före jämförelsen, så att den alfabetiska sorteringen i `comm` motsvarar den numeriska. Sökmönstret passar adressformen i Postfix 3.5 (nummer före adressen); för aktuella versioner används i stället `test[0-9]+@` och `seq 0 49999`:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

`comm -23` ger exakt de nummer som finns i referensmängden men inte i mottagarlistan: de saknade mejlen. Tom utmatning innebär fullständig leverans. Om nummer förekommer dubbelt (synligt genom skillnaden mellan `sort` och `sort -u`) har ett system på vägen duplicerat meddelandet, vilket också är ett resultat.

Om målet är ett produktionsliknande system i stället för en smtp-sink tar dess loggning över dumpfilernas roll. På en Exchange-server ger till exempel `Get-MessageTrackingLog -Recipients` eller ett filter på mottagaradressen de ankomna numren, och på ett Postfix-system ger en `grep` på `to=` och basadressen i mailloggen samma resultat. Detta är just fördelen med numret i adressen: Mottagaren finns i varje meddelandespårning, medan ämnesraden beroende på system saknas där eller först måste aktiveras.

## När numret måste stå i ämnesraden

Vissa utvärderingar är beroende av ämnesraden, exempelvis om målsystemet skriver om mottagaradresser eller om loggarna endast visar mottagaren maskerad. Då återstår slingvarianten: ett anrop av `smtp-source` per mejl med `-m 1` och en ämnesrad som shell räknar upp, fördelad på flera parallella workers med sammanhängande nummerintervall.

```bash
worker() {
  local i
  for ((i = $1; i <= $2; i++)); do
    smtp-source -s 1 -m 1 -l 5120 \
      -S "$(printf 'Lasttest %05d' "$i")" \
      -f lasttest@example.com -t test@example.com \
      gateway.example.com:25 || echo "$i" >> fehlend.log
  done
}
for w in 0 1 2 3; do
  worker $(( w * 12500 + 1 )) $(( (w + 1) * 12500 )) &
done
wait
```

Priset är en fullständig anslutningsetablering per mejl: TCP-handshake, banderoll, `HELO`, sändning, `QUIT`. Körningen mäter därmed inte målsystemets maximala genomströmning, utan ett medvetet anslutningsintensivt fall. Antalet workers fastställs analogt med kalibreringskörningen ovan, men med worker-slingan i stället för `-s`. De inledande nollorna i ämnesraden gör att omformateringen som `-N`-varianten behöver inte krävs vid avstämningen.

## Regler för tester mot andra system

Så snart testet lämnar det egna systemet gäller tre villkor. För det första: Operatören av målsystemet känner till testet och har godkänt tidsfönstret; 50'000 mejl ser för all övervakning ut som en attack eller en spamvåg. För det andra: Mottagaradressen avslutas kontrollerat, i en dedikerad testbrevlåda, en kasseringsregel på målet eller en kasseringsdomän som leverantören avsett för detta; produktiva adresser hör inte hemma i ett belastningstest. För det tredje: Ett avbrottskriterium fastställs före start, till exempel en växande kö på målet eller en felfrekvens över ett tröskelvärde, och någon övervakar dessa värden under körningen.

Med dessa tre punkter och numreringen ger körningen i slutändan inte bara en genomströmningssiffra, utan ett beläggbart påstående: vilka av de 50'000 mejlen som kom fram, vilka som saknas och var på sträckan de senast sågs.

## Källor

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Man-sida för lastgeneratorn; beskriver hur `-N` fungerar i den aktuella versionen (räknare i local-part, plusadressering).

2.  [Postfix-källkod 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): visar för RHEL-8-versionen att numret placeras före (`RCPT TO:<%d%s>`) med startvärdet 1; i [aktuell version](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) läggs numret i stället till i local-part, från 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Man-sida för testmottagaren med dumpalternativen och de registrerade X-headerarna.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Mängdjämförelse mellan två sorterade listor, här för avstämning av referens- och mottagna nummer.
