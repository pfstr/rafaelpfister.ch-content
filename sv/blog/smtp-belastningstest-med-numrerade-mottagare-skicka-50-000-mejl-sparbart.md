---
title: "SMTP-belastningstest med numrerade mottagare: skicka varje e-postmeddelande spårbart"
navTitle: "Numrerade belastningstester"
description: "Ett belastningstest är bara så bra som dess utvärdering. Med alternativet -N numrerar smtp-source varje e-postmeddelande via mottagaradressen utan att offra genomströmningen. Så bygger du körningen, hur många sessioner som är lämpliga och hur saknade nummer hittas automatiskt."
date: "2026-08-27"
kategorie: "SMTP och e-postflöde"
timeToRead: "8 min läsning"
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
translationSourceHash: 7145f2b49fb0b141d9c74d009d7c480ce4d119b4c97236e2ed7d92a39f65a1c5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:49:50.586Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/smtp-belastningstest-med-numrerade-mottagare-skicka-50-000-mejl-sparbart
---

# SMTP-belastningstest med numrerade mottagare: skicka varje e-postmeddelande spårbart

Den som kör ett belastningstest vill kunna besvara två frågor efteråt: Har alla e-postmeddelanden kommit fram, och om inte, vilka saknas? Med identiska testmeddelanden går det bara att räkna, och en räknarställning med 13 saknade meddelanden säger ingenting om när och var de gick förlorade. Om däremot varje e-postmeddelande har ett löpnummer blir räknandet en avstämning: Varje nummer kan hittas separat i målsystemets loggar, luckor visar tidpunkten för förlusten och leveransordningen kan kontrolleras.

Den vanliga reflexen är ett skript som räknar upp ämnesraden. Det fungerar, men kostar genomströmning, eftersom lastgeneratorn `smtp-source` från Postfix-paketet anger ämnesraden fast per anrop, och en loop med ett anrop per e-postmeddelande tvingar fram en egen anslutning för varje meddelande. Den bättre meddelandeidentifieringen är redan inbyggd: Alternativet `-N` numrerar mottagaradressen för varje meddelande, inom ett enda anrop med parallella sessioner. För utvärderingen är mottagaradressen lika användbar som ämnesraden, eftersom den finns i varje spårningslogg.

Den här testuppsättningen skickar, till skillnad från ett rent loopback-funktionstest, till ett annat system över nätverket. Om Postfix inte är installerat på källsystemet visar artikeln [smtp-source utan Postfix-installation](https://rafaelpfister.ch/blog/smtp-source-ohne-postfix-installation) hur du packar upp verktygen från RPM-paketet.

## De viktigaste alternativen för smtp-source

Som orientering följer här alternativen som förekommer i den här artikeln, fritt översatta från manualsidan:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Alternativ | Betydelse |
|---|---|
| `-s n` | Antal parallella SMTP-sessioner (standard: 1) |
| `-m n` | Totalt antal meddelanden som ska skickas (standard: 1) |
| `-l n` | Storlek på meddelandetexten i byte, exklusive header |
| `-f adresse` | Avsändaradress |
| `-t adresse` | Mottagaradress (standard: `foo@hostname`) |
| `-S text` | Ämnesrad, fast för alla meddelanden i anropet |
| `-F datei` | Skickar header och brödtext oförändrade från en fil; ersätter `-l` och `-S` |
| `-N` | Numrerar mottagaradressen för varje meddelande (räknare per process; position och startvärde beror på version, se nedan) |
| `-r n` | Antal mottagare per meddelande (standard: 1), adressbildning som med `-N` |
| `-d` | Koppla inte från efter ett meddelande, skicka nästa via samma anslutning |
| `-c` | Visa löpande räknare som ökar med varje slutförd `DATA` |
| `-w n` | Fast väntetid på n sekunder mellan meddelanden (per session) |
| `-v` | Utförlig utmatning för felsökning |
| `host:port` | Mål för inlämning via TCP; utan angiven port används standardporten smtp |

</details>

Den fullständiga listan, inklusive alternativ för TLS, LMTP och timing, finns i manualsidan `smtp-source(1)`; motsvarigheten för mottagarsidan är `smtp-sink(1)` och används i utvärderingen nedan.

## Hur -N numrerar mottagare

`-N` aktiverar en räknare per process som byggs in i mottagaradressen. Tre egenskaper avgör testuppsättningen, och alla tre går att läsa i källkoden för `smtp-source.c`:

För det första beror den exakta adressformen på Postfix-versionen. Postfix 3.5, som levereras med RHEL 8, placerar numret framför hela adressen (`RCPT TO:<%d%s>`): Av `-t test@example.com` blir `1test@example.com`, `2test@example.com` och så vidare, och räknaren börjar på 1. Aktuella Postfix-versioner lägger i stället till numret i slutet av local-part och börjar på 0 (`test0@` till `test49999@`); för denna variant rekommenderar manualsidan plusadressering (`-t 'test+@example.com'` blir `test+0@` och följande), så att ett målsystem med subadressering tilldelar allt till samma postlåda. Kontrollera formen före den stora körningen med en handfull e-postmeddelanden mot en `smtp-sink` eller i målets logg; den avgör referensmängden och sökmönstret vid utvärderingen.

För det andra är räknaren processomfattande och delas av alla parallella sessioner. Med `-s 8` tilldelar de åtta sessionerna numren gemensamt, varje nummer förekommer exakt en gång. Ordningen mellan sessionerna är inte deterministisk, men fullständigheten i mängden av nummer är garanterad.

För det tredje går startvärdet inte att konfigurera: 1 i Postfix 3.5, 0 i aktuella versioner. E-postmeddelandena har alltså numren 1 till det totala antalet från `-m` respektive 0 till totalt antal minus 1, och referensmängden för avstämningen måste passa detta.

## Testkörningen i ett anrop

Hur många e-postmeddelanden körningen omfattar spelar ingen roll för tillvägagångssättet; `-m` anger totalantalet, och exemplen i den här artikeln använder 50'000 som en godtycklig platshållare.

```bash
smtp-source -c -d -N -s 8 -m 50000 -l 5120 \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-c` | Löpande räknare över slutförda leveranser som en enradsförloppsindikator |
| `-d` | Anslutningarna hålls öppna för alla meddelanden; utan `-d` skapas en ny anslutning per meddelande |
| `-N` | Mottagarnumrering: lägger till räknaren per process i local-part |
| `-s 8` | Åtta parallella SMTP-sessioner |
| `-m 50000` | Totalt antal meddelanden, fördelat på sessionerna |
| `-l 5120` | Meddelandestorlek i byte (exklusive header), här 5 KB |
| `-f` | Avsändaradress |
| `-t` | Basadress för mottagaren; `-N` gör den till `1test@`, `2test@` och så vidare (Postfix 3.5) respektive `test0@`, `test1@` och så vidare (aktuella versioner) |
| `gateway.example.com:25` | Målserver och port |

</details>

`-d` är avgörande för lastprofilen: Utan detta alternativ kopplar `smtp-source` från efter varje meddelande och skapar en ny anslutning för nästa; med `-d` hålls de åtta anslutningarna öppna och levererar alla meddelanden i följd, som en massavsändare gör.

Det från funktionstester välkända `-v` utelämnas medvetet: Det loggar varje enskild SMTP-dialog från `HELO` till `QUIT` och skapar hundratusentals loggrader vid en stor körning utan mervärde för utvärderingen. `-c` ger i stället sammanfattningen som gör att körningens förlopp kan följas live. Den totala tiden för beräkning av takten fås med ett inledande `time`.

Förutsättning för hela metoden: Målsystemet accepterar de genererade adresserna. En `smtp-sink`, en catch-all-domän, en kassationsdomän hos leverantören eller en gateway som löser upp mottagare först efter godkännande uppfyller detta. Om målet däremot kontrollerar varje mottagare mot en katalog avvisar det de numrerade adresserna, och då återstår bara ämnesradsvarianten.

## Ange egna headers

Vissa belastningstester behöver en egen header, till exempel som markör för att gatewayen ska känna igen testmeddelandena eller för att en regel ska träffa. `smtp-source` har inget alternativ för detta, men `-F` läser ett fullständigt förformaterat meddelande från en fil, där valfria headers kan anges. Filen består av headerraderna, en tom rad och brödtexten, med alla rader avslutade med `\r\n`:

```bash
{ printf 'X-Lasttest: aktiv\r\n'
  printf 'Subject: Lasttest\r\n'
  printf '\r\n'
  head -c 5120 /dev/zero | tr '\0' 'x'
  printf '\r\n'; } > lasttest.eml
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `head -c 5120` | Skriver ut de första 5120 byten av inmatningen, här från `/dev/zero` |
| `tr '\0' 'x'` | Ersätter varje nollbyte med tecknet `x` och skapar därmed 5 KB fyllnadstext |
| `> lasttest.eml` | Skriver det sammansatta meddelandet till filen för `-F` |

</details>

```bash
smtp-source -c -d -N -s 8 -m 50000 -F lasttest.eml \
  -f lasttest@example.com \
  -t test@example.com \
  gateway.example.com:25
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-F datei` | Skickar header och brödtext oförändrade från filen; ersätter det genererade meddelandeinnehållet |

</details>

Det får två följder: `-F` ersätter `-l` och `-S`, eftersom storlek och ämnesrad nu kommer från filen (båda måste därför finnas där). `-N` fortsätter däremot att fungera och mottagarna numreras vidare; headern är identisk i alla meddelanden eftersom den kommer från den fasta filen.

## Hur många sessioner?

Det mest tillförlitliga sättet att fastställa lämpligt antal sessioner är genom mätning, med exakt samma alternativ som för den planerade huvudkörningen: samma meddelandekälla (samma `-F`-fil respektive samma `-l`), samma avsändare, samma mål. Endast mängden minskas till 2'000 per nivå, och `-s` varierar. En kort kalibreringskörning med ökande antal sessioner visar när ytterligare sessioner inte längre ger något:

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

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `date +%s%N` | Skriver ut Unix-sekunder direkt följda av nanosekundsdelen som ett tal |
| `-d` | Anslutningarna hålls öppna för alla meddelanden i nivån |
| `-N` | Mottagarnumrering via räknaren per process |
| `-s "$s"` | Antal parallella sessioner, 1 till 32 per loopvarv |
| `-m 2000` | 2'000 meddelanden per mätsteg |
| `-F lasttest.eml` | Samma meddelandefil som i den planerade huvudkörningen |
| `-f` | Avsändaradress |
| `-t '@blackhole.example.com'` | Basadress för mottagaren med tom local-part på en kassationsdomän |
| `gateway.example.com:25` | Målserver och port |

</details>

Två detaljer i anropet: `-c` utelämnas här medvetet så att inga löpande räknarutskrifter hamnar mellan mätraderna; loopen ger exakt en resultatrad per nivå. Och den tomma local-part i `-t` fungerar väl ihop med numreringen på en kassationsdomän: Med Postfix 3.5:s inledande räknare blir resultatet rent numeriska mottagaradresser (`1@blackhole.example.com`, `2@…`), vilket håller utvärderingen i loggarna överskådlig.

I detalj sker följande: Den yttre loopen går igenom sessionsantalet 1 till 32 i fördubblingssteg. Före och efter varje körning sparar `date +%s%N` den aktuella tiden som ett stort tal, nämligen Unix-sekunder direkt följda av nanosekundsdelen. Däremellan skickar `smtp-source` 2'000 meddelanden (innehåll, headers och storlek kommer från `-F`-filen) över respektive antal parallella anslutningar som hålls öppna tack vare `-d`; loopen väntar tills anropet är helt klart. Raden `echo` räknar om tidsskillnaden till en takt: 2'000 e-postmeddelanden delat med körtiden i sekunder, där körtiden anges i nanosekunder. Av 2'000 gånger 10⁹ blir därmed konstanten `2000000000000`. Bash-aritmetiken `$(( ))` beräknar heltal och kapar decimaler, vilket är tillräckligt exakt för denna mätning.

Tre praktiska anmärkningar: `%N` ger nanosekunder endast med GNU date (vilket gäller för RHEL och de flesta Linux-system; BusyBox och macOS saknar det). Den kompletta körningen skickar 6 × 2'000 = 12'000 e-postmeddelanden, och även de behöver en kontrollerad mottagaradress, medan numreringen med `-N` börjar om från startvärdet vid varje anrop. Och om ett anrop till `smtp-source` avbryts med ett felmeddelande saknar takten på den raden betydelse; åtgärda först orsaken och mät sedan igen.

Den förväntade utmatningen är en rad per nivå. Med påhittade men typiska exempelvärden ser den ut så här:

```text
1 Sessions: 11 Mails/s
2 Sessions: 21 Mails/s
4 Sessions: 40 Mails/s
8 Sessions: 71 Mails/s
16 Sessions: 79 Mails/s
32 Sessions: 80 Mails/s
```

Tolkning: Så länge takten ungefär fördubblas med antalet sessioner döljer de parallella sessionerna väntetiden på svar från målet; flaskhalsen är då sträckans latens, inte kapaciteten. När kurvan planar ut (i exemplet mellan 8 och 16 sessioner) är antingen målsystemet mättat eller källan vid sin gräns. Välj det minsta värde där takten inte längre ökar nämnvärt, i exemplet alltså 8 till 16; fler sessioner ökar då bara lasten genom parallellitet, inte genomströmningen. För huvudkörningen kan den förväntade tiden också uppskattas direkt från den uppmätta takten: det totala antalet från `-m` delat med takten.

## Utvärdering på mottagarsidan

Om det finns en egen testmottagare på målsystemet sköter `smtp-sink` loggningen direkt:

```bash
smtp-sink -c -d "mails/%Y%m%d-%H%M%S." 0.0.0.0:2525 200
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-c` | Löpande räknare i stället för hela SMTP-dialogen |
| `-d "mails/…"` | För sink: dump, inte anslutningshållning. Skriver varje accepterat meddelande till en egen fil (namnmönster via strftime), inklusive en `X-Rcpt-Args`-header med mottagaradressen |
| `0.0.0.0:2525` | Lyssnar på alla gränssnitt på port 2525 |
| `200` | Backlog: maximal längd på kön av väntande anslutningar enligt listen(2) |

</details>

Efter körningen extraherar du de mottagna numren och jämför dem med referensmängden. Eftersom numren inte har inledande nollor formateras båda listorna till ett fast antal siffror före jämförelsen, så att den alfabetiska sorteringen i `comm` motsvarar den numeriska. Sökmönstret passar adressformen i Postfix 3.5 (nummer före adressen); för aktuella versioner används i stället `test[0-9]+@` och `seq` från 0:

```bash
grep -rhoE '[0-9]+test@example\.com' mails/ | \
  grep -oE '^[0-9]+' | sort -u | \
  awk '{printf "%08d\n", $1}' | sort > empfangen.txt
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `grep -r` | Söker rekursivt i katalogen `mails/` |
| `grep -h` | Undertrycker filnamnen före träffarna |
| `grep -o` | Skriver endast ut den matchande adressdelen, inte hela raden |
| `grep -E` | Utökade reguljära uttryck, här för `[0-9]+` |
| `sort -u` | Sorterar och tar bort dubbletter (varje nummer en gång) |
| `awk '{printf "%08d\n", $1}'` | Fyller ut varje nummer med inledande nollor till åtta siffror |
| `sort` | Sorterar de utfyllda numren för jämförelse med `comm` |

</details>

```bash
seq 1 50000 | awk '{printf "%08d\n", $1}' | \
  comm -23 - empfangen.txt
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `seq 1 50000` | Skapar referensmängden av nummer; slutvärdet motsvarar det totala antalet skickade meddelanden från `-m` |
| `comm -23` | Undertrycker kolumn 2 (endast i fil 2) och kolumn 3 (i båda); kvar blir raderna som bara finns i referensmängden |
| `-` | Läser den första jämförelselistan från pipen i stället för från en fil |
| `empfangen.txt` | Andra jämförelselistan: de faktiskt mottagna numren |

</details>

`comm -23` skriver ut exakt de nummer som finns i referensmängden men inte i mottagningslistan: de saknade e-postmeddelandena. Tom utmatning betyder fullständig leverans. Om nummer förekommer dubbelt (vilket syns genom skillnaden mellan `sort` och `sort -u`) har ett system längs vägen duplicerat meddelandet, vilket också är ett fynd.

Om målet är ett system nära produktion i stället för en smtp-sink tar dess loggning rollen som dumpfilerna. På en Exchange-server ger till exempel `Get-MessageTrackingLog -Recipients` eller ett filter på mottagaradressen de mottagna numren, på ett Postfix-system ett `grep` på `to=` och basadressen via e-postloggen. Det är just fördelen med numret i adressen: Mottagaren finns i varje meddelandespårning, medan ämnesraden beroende på system saknas där eller måste aktiveras först.

## När numret måste stå i ämnesraden

Vissa utvärderingar bygger på ämnesraden, till exempel när målsystemet skriver om mottagaradresser eller loggarna endast visar mottagaren maskerad. Då återstår loopvarianten: ett anrop till `smtp-source` per e-postmeddelande med `-m 1` och en ämnesrad som skalet räknar upp, fördelat på flera parallella arbetare med sammanhängande nummerintervall.

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

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-s 1` | En session per anrop; de fyra arbetarna ger parallelliteten |
| `-m 1` | Exakt ett meddelande per anrop, så att ämnesraden kan anges per e-postmeddelande |
| `-l 5120` | Meddelandestorlek i byte (exklusive header), här 5 KB |
| `-S "$(printf 'Lasttest %05d' "$i")"` | Ämnesrad med det löpande numret utfyllt till fem siffror |
| `-f` / `-t` | Avsändar- och mottagaradress |
| `gateway.example.com:25` | Målserver och port |

</details>

Priset är en fullständig anslutningsetablering per e-postmeddelande: TCP-handshake, banner, `HELO`, sändning, `QUIT`. Denna körning mäter alltså inte målsystemets maximala genomströmning, utan ett avsiktligt anslutningsintensivt fall. Antalet arbetare bestäms på samma sätt som i kalibreringskörningen ovan, men med arbetar-loopen i stället för `-s`. De inledande nollorna i ämnesraden gör att omformateringen som `-N`-varianten behöver slipper användas vid avstämningen.

## Regler för tester mot andra system

Så snart testet lämnar det egna systemet gäller tre villkor. För det första: Operatören av målsystemet känner till testet och har godkänt tidsfönstret; ett belastningstest ser för varje övervakningssystem ut som en attack eller en spamvåg. För det andra: Mottagaradressen avslutas kontrollerat, i en dedikerad testpostlåda, en kasseringsregel på målet eller en kassationsdomän avsedd för detta från leverantören; produktiva adresser hör inte hemma i ett belastningstest. För det tredje: Ett avbrottskriterium fastställs före start, till exempel en växande kö på målet eller en felfrekvens över ett tröskelvärde, och någon övervakar dessa värden under körningen.

Med dessa tre punkter och numreringen ger körningen i slutet inte bara ett genomströmningsvärde, utan ett beläggbart resultat: vilka e-postmeddelanden som kom fram, vilka som saknas och var längs vägen de senast sågs.

## Källor

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): Manualsida för lastgeneratorn; beskriver `-N`-beteendet i den aktuella versionen (räknare vid local-part, plusadressering).

2.  [Postfix-källkod 3.5.8: smtp-source.c](https://github.com/vdukhovni/postfix/blob/v3.5.8/postfix/src/smtpstone/smtp-source.c): Visar för RHEL-8-versionen att numret placeras före (`RCPT TO:<%d%s>`) med startvärdet 1; i [den aktuella versionen](https://github.com/vdukhovni/postfix/blob/master/postfix/src/smtpstone/smtp-source.c) läggs numret i stället till vid local-part, från 0.

3.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): Manualsida för testmottagaren med dumpalternativen och de registrerade X-headerna.

4.  [GNU Coreutils: comm](https://www.gnu.org/software/coreutils/manual/html_node/comm-invocation.html): Mängdjämförelse mellan två sorterade listor, här för avstämning av referens- och mottagningsnummer.
