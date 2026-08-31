---
title: "Planlegging av belastningstester for e-post: Verktøy for e-postburster på 10 000 under Linux og Windows sammenlignet"
navTitle: "Belastningstester for e-post"
description: "De som migrerer en gateway eller dimensjonerer et e-postmiljø, trenger pålitelige tall fremfor magefølelse. Hvilke verktøy som genererer burster på flere titusen e-poster, hvordan en ryddig testplan ser ut, og hvordan du evaluerer resultatene fra loggene."
date: "2026-08-24"
kategorie: "SMTP og e-postflyt"
timeToRead: "12 min lesetid"
themen:
  - smtp-mailflow
  - testing
produkte:
  - "uebergreifend"
protokolle:
  - "testing"
  - "smtp"
  - "tcp"
  - "tls"
  - "troubleshooting"
slug: "planlegging-av-e-postlasttester-verktoy-for-10-000-e-postburster-under-linux-og-windows"
translationId: "article-14a98de0cef45565"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests. Hilf mir Schritt für Schritt, einen Lasttest gegen mein eigenes Test-Mailgateway zu planen: Zieldefinition (Durchsatz, Latenz, Queue-Verhalten), Wahl des Lastgenerators (smtp-source, Postal, JMeter oder Skript), Aufbau einer Mail-Senke, Testablauf (Baseline, Burst, Soak) und Auswertung der Logs mit Perzentilen. Frage zuerst nach Plattform, Zielsystem und erwartetem Mailvolumen.
translationOf: mail-lasttest-tools-linux-windows-vergleich
translationSourceHash: 2fd0b1bd0748b9fb44be85907a946bbf85604b5eb7c85107170fa7443068efd7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:28:54.112Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/planlegging-av-e-postlasttester-verktoy-for-10-000-e-postburster-under-linux-og-windows
---

# Planlegging av belastningstester for e-post: Verktøy for e-postburster på 10 000 under Linux og Windows sammenlignet

Om en ny e-postgateway tåler toppbelastningen fra en nattlig fakturakjøring, kan bare kontrolleres med en belastningstest. De som erstatter en appliance, dimensjonerer et Exchange-miljø eller planlegger utsendelse av nyhetsbrev via egen infrastruktur, trenger på forhånd pålitelige tall: Hvor mange meldinger per sekund tar systemet imot, hvordan oppfører køen seg under press, og fra hvilket punkt begynner utsettelser? Denne artikkelen sammenligner vanlige belastningsgeneratorer under Linux og Windows og viser hvordan en test med burster på flere titusen e-poster planlegges, gjennomføres og evalueres.

Først den viktigste regelen: Belastningstester hører utelukkende hjemme i egen infrastruktur eller i et testmiljø som uttrykkelig er godkjent for dette. En burst mot fremmede systemer er et angrep, og en test med oppdiktede avsenderadresser mot produksjonsmål genererer backscatter som fører til blokkeringslister. Et ryddig oppsett består av en belastningsgenerator, systemet som skal testes og en kontrollert sluk som til slutt tar imot og forkaster e-postene.

## Hva en belastningstest for e-post skal måle

Før et verktøy kommer på tale, er det verdt å spørre hvilken størrelse som egentlig er interessant. I praksis er det fire ulike, og de krever ulike testoppsett:

1. **Mottaksrate:** Hvor mange meldinger per sekund tar første hopp imot via SMTP? Dette er den klassiske gjennomstrømningsmålingen og verdien belastningsgeneratorer leverer direkte.
2. **Øktlatens:** Hvor lenge varer én enkelt SMTP-transaksjon fra forbindelsen opprettes til `250` etter `DATA`? Under belastning øker denne verdien ofte lenge før mottaksraten faller.
3. **Ende-til-ende-latens:** Hvor lang tid bruker en melding fra innlevering til levering til sluket, gjennom alle mellomstasjoner? Dette er størrelsen brukerne merker.
4. **Køatferd:** Hvor dypt vokser køen under bursten, og hvor raskt tømmes den etterpå? En gateway som tar imot 50'000 e-poster og deretter bruker tre timer på å behandle dem, består mottakstesten, men stryker likevel.

En test som bare måler mottaksraten, sier lite om et flertrinnsmiljø med gateway, krypteringslag og målserver. Det er særlig der ende-til-ende-perspektivet avgjør.

## Belastningsbildet bestemmer verktøyet

Ved siden av måleverdien avgjør et annet spørsmål verktøyvalget, og det blir ofte hoppet over: Hvilken forbindelsesatferd har belastningen som skal simuleres? Det må skilles mellom to belastningsbilder.

En **bulk-avsender med åpne økter** er belastningsbildet for fakturakjøringer, lønnsavregninger og nyhetsbrevsystemer: Ett enkelt system oppretter få forbindelser og sender hundrevis til tusenvis av meldinger etter hverandre via dem. Forbindelsesoverheaden påløper én gang per økt, ikke én gang per melding, og gatewayen ser få forbindelser med mange transaksjoner.

**Mange uavhengige innleverere** er belastningsbildet for applikasjonslandskap og brukertrafikk: Tallrike systemer leverer hver sin enkeltmelding over hver sin forbindelse. Her hører forbindelsesopprettelsen, inkludert TLS og AUTH, til hver melding.

For dimensjonering av masseutsendelser er det første belastningsbildet relevant, og da må belastningsgeneratoren kunne holde økter åpne: `smtp-source` gjør dette (mange meldinger fordelt på få økter), det samme gjør Postal og egne skript med vedvarende forbindelse. JMeter kan ikke det; bakgrunnen beskrives i Windows-delen. For toppbelastningen i en fakturakjøring er derfor dette øktkriteriet avgjørende, ikke plattformen; under Windows går veien da via WSL.

## Verktøy under Linux

**smtp-source og smtp-sink** fra Postfix-pakken er standarden for rå SMTP-belastning og er tilgjengelige på praktisk talt alle systemer med Postfix installert. `smtp-source` genererer meldinger med justerbar størrelse, parallellitet og antall, mens `smtp-sink` er motstykket: en SMTP-server som tar imot og forkaster alt. En burst på 10'000 e-poster med 50 parallelle økter og meldinger på 5 KB er en énlinjekommando:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `time` | Måler den totale kjøretiden for kallet; dette gir raten i e-poster per sekund |
| `-s 50` | 50 parallelle SMTP-økter |
| `-m 10000` | Totalt antall meldinger, fordelt på øktene |
| `-l 5120` | Størrelsen på meldingsteksten i byte (uten headere), her 5 KB |
| `-c` | Løpende teller for sendte meldinger som fremdriftsindikator |
| `-f last@test.example` | Avsenderadresse |
| `-t senke@test.example` | Mottakeradresse |
| `gateway.test.example:25` | Målvert og port for innlevering |

</details>

Viktige begrensninger: `smtp-source` måler ingen latenspercentiler, og meldingene er syntetiske og ensartede. For spørsmålet «hvor mye tar systemet maksimalt imot» er det likevel førstevalget, fordi det selv på svak maskinvare genererer titusenvis av meldinger per minutt og generatoren praktisk talt aldri blir flaskehalsen.

**Postal** er den klassiske dedikerte e-postserverbenchmarken under Linux. Den varierer avsender, mottaker og meldingsstørrelse automatisk, holder en målrate over lange tidsrom og skriver statistikk per minutt. Dermed egner den seg bedre enn `smtp-source` for soak-tester, altså vedvarende belastning over flere timer. Den tilhørende `bhm` (Black Hole Mailer) fyller rollen som sluk. Postal er gammelt, men er bygget nettopp for dette og finnes i pakkekildene til de fleste distribusjoner.

**swaks** er ingen belastningsgenerator, men hører hjemme i enhver testplan. Det utfører én enkelt SMTP-transaksjon med full kontroll over hvert trinn: autentisering, STARTTLS, vilkårlige headere og vedlegg. Før hver belastningstest bør du kjøre swaks som en funksjonstest, slik at bursten ikke mislykkes på grunn av feil mottaker eller et TLS-problem og gjør målingen verdiløs. I en løkke med `xargs -P` kan swaks også misbrukes som en liten belastningsgenerator, men for titusenvis av e-poster er prosessoverheaden for stor.

**Egne skript** i Python (smtplib, aiosmtplib) eller Go er veien å gå når testen trenger realistiske meldingsmikser: ulike størrelser, faktiske vedlegg, varierende antall mottakere per transaksjon og målrettede feilsituasjoner. Innsatsen er større, men skriptet måler nøyaktig det eget miljø senere vil se, og kan skrive tidsstempler per melding for latensevalueringen.

## Verktøy under Windows

**Apache JMeter** er det riktige verktøyet under Windows når belastningsbildet består av mange uavhengige innleverere, eller når percentiler, meldingsmiks og rapporter står i sentrum. Den innebygde SMTP Sampler støtter Auth, STARTTLS, vedlegg og EML-filer som meldingskilde, og JMeter-mekanismen leverer det Postfix-verktøyene mangler: trådgrupper for trinnvise belastningsprofiler, responstidspercentiler, feilsatser og rapporter. For burster utover noen få tusen e-poster per minutt gjelder den vanlige JMeter-regelen: Bruk GUI kun til å opprette testplanen, og kjør selve målingen i CLI-modus, ellers måler du brukergrensesnittet samtidig.

En begrensning ved SMTP Sampler må være kjent: JMeter kan ikke holde SMTP-økter åpne. Hver sample-kjøring åpner en ny forbindelse, går gjennom hele dialogen med TCP-handshake, EHLO, eventuelt STARTTLS og AUTH, sender nøyaktig én melding og avslutter forbindelsen med QUIT. Flere meldinger over samme åpne forbindelse, slik masseavsendere med gjenbruk av økter gjør, kan ikke modelleres; `smtp-source` fordeler derimot mange meldinger på få åpne økter. Årsaken ligger i arkitekturen: JMeter er et protokolluavhengig rammeverk for belastningstesting, ikke et SMTP-verktøy. Utførelsesmodellen behandler hver sampler som en selvstendig, uavhengig målt enhet, for bare slik kan timere, assertions og percentilutregning fungere enhetlig for alle støttede protokoller. SMTP Sampler er derfor et tynt lag oppå JavaMail-biblioteket, som som klient-API oppretter og lukker en forbindelse for hver sending; gjenbruk av forbindelser på tvers av samples, slik HTTP Sampler tilbyr med Keep-Alive, har aldri blitt implementert for SMTP. For målingen betyr dette: JMeter genererer belastningsbildet med mange enkeltstående innleverere, ikke med en bulk-avsender med åpen økt. Den målte gjennomstrømningen inkluderer full forbindelses- og TLS-overhead per melding, og forbindelsesgrenser på gatewayen slår derfor inn tidligere enn ved gjenbruk av økter. For bulk-avsenderens belastningsbilde ved en fakturakjøring er JMeter dermed ikke riktig verktøy; under Windows er WSL-veien med `smtp-source` et bedre valg.

**PowerShell med MailKit** er veien med innebygde virkemidler. Det tidligere vanlige `Send-MailMessage` er av Microsoft selv markert som foreldet, og selskapet anbefaler overgang; MailKit kan lastes inn via NuGet og parallelliseres fra PowerShell 7 med Runspaces. Realistisk oppnår du dermed noen hundre til noen få tusen e-poster per minutt, nok for funksjons- og regresjonstester, men for lite for måling av maksimal belastning. Fordelen er at skriptet kjører uten ekstra installasjon på enhver administratorarbeidsstasjon og kan skrive resultater direkte som CSV for evaluering.

**Microsoft Exchange Load Generator (LoadGen)** var i mange år det offisielle verktøyet for å belaste Exchange-miljøer med simulerte brukerprofiler (Outlook, ActiveSync, OWA). Microsoft videreutviklet det ikke etter Exchange 2013 og fjernet nedlastingen. For ren SMTP-belastning var LoadGen uansett feil verktøy; de som i dag vil simulere belastning på Exchange-postbokser, står uten et offisielt verktøy og tester SMTP-veien bedre direkte.

**WSL** fortjener et eget punkt: De som sitter på en Windows-maskin, men trenger Linux-verktøy, kan installere `smtp-source` og Postal i en WSL-distribusjon og får dermed alle Linux-verktøyene uten en separat test-VM. For belastningene som diskuteres her, er WSL-nettverksveien ingen relevant flaskehals.

## Sammenligning

| Verktøy | Plattform | Styrke | Begrensning |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maksimal råbelastning med minimal innsats, generator og sluk fra samme sted | Ingen latenspercentiler, ensartede meldinger |
| Postal / bhm | Linux | Vedvarende belastning med målrate, varierende meldinger, minuttsstatistikk | Utdatert verktøykjede, må bygge evalueringen selv |
| swaks | Linux, Windows (Perl) | Fullt kontrollerbar enkelttest, ideell som funksjonssjekk før bursten | Ingen belastningsgenerator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Belastningsprofiler, percentiler, rapporter, EML-meldingskilder | Java-overhead, GUI-modus uegnet for høye rater, én forbindelse per melding (ingen gjenbruk av økter) |
| PowerShell + MailKit | Windows | Uten ekstra installasjon på enhver administrator-PC, CSV-utskrift | Begrenset gjennomstrømning, må bygge parallelliseringen selv |
| Eget skript (Python/Go) | begge | Realistisk meldingsmiks, egne målepunkter | Utviklingsinnsats, må validere generatoren selv |

## Sluket: hvor skal e-postene gå?

Den undervurderte halvdelen av testoppsettet er målet. Tre varianter har vist seg å fungere godt:

- **smtp-sink** eller `bhm` som svart hull: tar imot alt, forkaster alt og måler den rene transportkjeden. `smtp-sink` kan ved behov generere kunstige svarforsinkelser og feilkoder, og dermed også teste testssystemets atferd ved et langsomt eller feilende mål.
- **Postfix med discard-transport** som et mer realistisk sluk når målet selv skal være en fullverdig SMTP-server med køhåndtering.
- **Noen få faktiske seed-postbokser** i tillegg til sluket, for stikkprøvevis å kontrollere at meldinger ankommer innholdsmessig intakte, inkludert krypterings- eller signeringslag.

Verktøy med webgrensesnitt som Mailpit er ment for utvikling og blir raskt selv flaskehalsen ved titusenvis av e-poster. De er uegnet som sluk for en belastningstest; målingen ville målt analyseverktøyet i stedet for testsystemet.

## Planlegg testen

En pålitelig test går i tre trinn, hvert med sitt eget spørsmål:

1. **Baseline:** En moderat, kjent belastning (omtrent 10 prosent av forventet topp) over noen minutter. Den gir referanseverdier for latens og ressursforbruk og avdekker konfigurasjonsfeil før de drukner i burst-målingen.
2. **Burst:** Selve målingen av toppbelastning, for eksempel 10'000 til 50'000 e-poster så raskt som mulig eller med definert målrate. Flere kjøringer med økende parallellitet viser hvor mottaksraten flater ut og latensen tipper.
3. **Soak:** Forventet dagsbelastning over flere timer. Først her viser minnelekkasjer, fulle spool-partisjoner, loggrotering under belastning og forbindelsesgrenser seg – forhold som en kort burst aldri når.

For meldingsmiksen gjelder: så realistisk som nødvendig. En måling med utelukkende tekstmeldinger på 5 KB overvurderer gjennomstrømningen i et miljø der hverdagen består av PDF-vedlegg, med en mangedobling. En blanding fra egen bestand er fornuftig, for eksempel 70 prosent små, 25 prosent med typisk vedlegg og 5 prosent store. TLS hører også med i testen dersom produksjonen bruker TLS: Handshaken koster betydelig mer per forbindelse enn selve meldingsoverføringen, og generatorer som åpner en ny forbindelse per e-post, måler ellers primært TLS-termineringen.

For senere evaluering får hver testmelding en entydig markør, enklest i form av en egen header som `X-Loadtest-Id` med løpenummer og tidsstempel samt en gjenkjennelig emnekonvensjon. Dermed kan testkjøringer skilles ryddig fra hverandre og fra øvrig trafikk i loggene, og testmeldingene kan ryddes målrettet ut av karantener og journaler etter kjøringen.

Tre punkter som regelmessig glemmes i planleggingen: For det første rategrenser og tarpitting på testveien; en gateway som struper etter 100 e-poster per minutt per kilde-IP, tester ellers bare sin egen struping (ta det målrettet ut ved måling av maksimal belastning, behold det bevisst for realitetssjekken). For det andre DNS: Hvis testsystemet slår opp mottakerdomener eller utfører DNSBL-oppslag for hver melding, må en resolver inngå i testmiljøet, ellers måler testen oppstrøms DNS. For det tredje selve generatoren: Før første kjøring mot målsystemet bør generatoren kjøres direkte mot sluket for å dokumentere at den overhodet kan generere målraten.

## Evaluering

Måleverdiene fra belastningsgeneratoren er bare halve sannheten, for de viser innlevererens perspektiv. Den andre halvdelen står i loggene til testsystemet.

I Postfix gir e-postloggen feltene `delay` og `delays` per melding, der sistnevnte er delt opp etter tid i køen, forbindelsesopprettelse og overføring. En evaluering av en testkjøring kan gjøres med innebygde verktøy:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `grep "status=sent" /var/log/mail.log` | Filtrerer e-postloggen på meldinger som er levert uten feil |
| `grep -o "delay=[0-9.]*"` | `-o` skriver bare ut selve treffet, her feltet `delay` med verdien sin |
| `cut -d= -f2` | Deler på `=` (`-d`) og beholder det andre feltet (`-f2`), altså tallverdien |
| `sort -n` | Sorterer numerisk i stedet for alfabetisk; forutsetning for percentilberegning |
| `awk '…'` | Samler de sorterte verdiene i et array og skriver ut antall, p50, p95, p99 og maksimum |

</details>

På Exchange-siden er Message Tracking Log den sentrale kilden. For en testkjøring med emnekonvensjon:

```powershell
$p = @{
    Start          = "24.08.2026 14:00"
    End            = "24.08.2026 15:00"
    MessageSubject = "LOADTEST"
    ResultSize     = "Unlimited"
}
Get-MessageTrackingLog @p | Group-Object EventId |
    Sort-Object Count -Descending | Format-Table Name, Count
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Start` / `End` | Tidsvindu for loggsøket; overføres her via splatting (`@p`) |
| `MessageSubject "LOADTEST"` | Filtrerer på meldinger der emnet inneholder markøren |
| `ResultSize Unlimited` | Opphever standardgrensen på 1000 returnerte oppføringer |
| `Group-Object EventId` | Grupperer sporingshendelsene etter type (RECEIVE, DELIVER, DEFER, …) |
| `Sort-Object Count -Descending` | Sorterer hendelsesgruppene synkende etter hyppighet |
| `Format-Table Name, Count` | Viser antallet per hendelsestype |

</details>

Differansen mellom tidsstemplene for RECEIVE- og DELIVER-hendelsen for samme MessageId gir ende-til-ende-latensen per melding; eksportert som CSV kan percentilfordelingen beregnes ut fra dette.

Ved tolkningen teller tre grunnprinsipper. For det første: Percentiler i stedet for gjennomsnitt. Et gjennomsnitt på to sekunder kan bety at alt tar to sekunder, eller at 95 prosent er gjennom på et halvt sekund mens resten hang i køen; p50, p95 og p99 skiller disse tilfellene. For det andre: Pivotér SMTP-svarkoder. Fordelingen av 4xx-svar over tid viser når systemet begynner å strupe, og hvilke koder det er (forbindelsesgrense, købeskyttelse, greylisting) viser hvilken mekanisme som griper inn først. For det tredje: Plott kødybden over tid, under Postfix med `qshape` eller `postqueue -j`, på Exchange med `Get-Queue` hvert minutt. Arealet under denne kurven, ikke mottaksraten, avgjør om miljøet håndterer en burst eller bare lagrer den.

Parallelt med e-postloggene hører systemmålingene fra testsystemet med i evalueringen: CPU, I/O-ventetider på spool-partisjonen, fildeskriptorer, forbindelsestellere. Det vanligste funnet i flertrinnsmiljøer er at det ikke er e-postprosessen som begrenser, men et content-inspection-lag (virusskanner, krypteringsmodul, DLP) med et fast antall arbeidere. Nettopp slike funn er den egentlige verdien av testen: De peker ut justeringsmuligheten før produksjonen finner den.

## Konklusjon

For rask måling av maksimal belastning under Linux kommer du ikke utenom `smtp-source` med `smtp-sink`; Postal kompletterer vedvarende belastning. Under Windows leverer JMeter den mest komplette målingen, PowerShell med MailKit dekker funksjons- og regresjonstester, og WSL henter ved behov Linux-verktøyene til administratorarbeidsstasjonen. Viktigere enn verktøyet er planen: separat måling av mottak, latens og køatferd, realistisk meldingsmiks, en merket testkjøring og en evaluering som inkluderer percentiler og loggene til målsystemet i stedet for bare generatorens teller.

## Kilder

1.  [smtp-source(1), Postfix-manual](https://www.postfix.org/smtp-source.1.html): Referanse for belastningsgeneratoren med alle alternativer for parallellitet, meldingsstørrelse og TLS.

2.  [smtp-sink(1), Postfix-manual](https://www.postfix.org/smtp-sink.1.html): Referanse for e-postsluket, inkludert kunstige forsinkelser og feilsvar.

3.  [Postal-dokumentasjon, Russell Coker](https://doc.coker.com.au/projects/postal/): Beskrivelse av e-postserverbenchmarken med målrate, meldingsvariasjon og bhm-sluk.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): SMTP-funksjonstesteren for forhåndssjekk av enhver testvei.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funksjonsomfanget til SMTP Sampler, inkludert Auth, TLS og EML-kilder.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Microsofts offisielle informasjon om at cmdleten er foreldet, med henvisning til alternativer som MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): .NET-e-postbiblioteket for egne utsendelsesskript under PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referanse for evaluering av Exchange Message Tracking Log etter en testkjøring.

9.  [qshape(1), Postfix-manual](https://www.postfix.org/qshape.1.html): Verktøy for analyse av køfordelingen under og etter bursten.
