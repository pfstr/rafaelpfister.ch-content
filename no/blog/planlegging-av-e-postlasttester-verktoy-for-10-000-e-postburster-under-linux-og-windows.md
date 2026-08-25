---
title: "Planlegging av e-postlasttester: Verktøy for 10'000-e-postburster under Linux og Windows sammenlignet"
navTitle: "E-postlasttester"
description: "Alle som migrerer en gateway eller dimensjonerer et e-postmiljø, trenger pålitelige tall fremfor magefølelse. Hvilke verktøy som genererer burster på flere titusen e-poster, hvordan en ryddig testplan ser ut, og hvordan du evaluerer resultatene fra loggene."
date: "2026-08-24"
kategorie: "SMTP og e-postflyt"
timeToRead: "12 min. lesetid"
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
url: https://rafaelpfister.ch/no/blog/planlegging-av-e-postlasttester-verktoy-for-10-000-e-postburster-under-linux-og-windows
translationSourceHash: c9b76f3c9887117756e07c71a3dc30d1ee99aeb8a322c50dee994a07df46cb97
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:13:35.547Z
translationReview: automatic
---

# Planlegging av e-postlasttester: Verktøy for 10'000-e-postburster under Linux og Windows sammenlignet

Om en ny e-postgateway tåler toppbelastningen på en natt med fakturakjøringer, viser seg ikke i databladet, men i testen. Alle som erstatter en appliance, dimensjonerer et Exchange-miljø eller planlegger utsendelse av nyhetsbrev via egen infrastruktur, trenger på forhånd pålitelige tall: Hvor mange meldinger per sekund godtar systemet, hvordan oppfører køen seg under press, og fra hvilket punkt begynner utsettelser? Denne artikkelen sammenligner vanlige lastgeneratorer under Linux og Windows og viser hvordan en test med burster på flere titusen e-poster planlegges, gjennomføres og evalueres.

Først den viktigste regelen: Lasttester hører kun hjemme i egen infrastruktur eller i et testmiljø som uttrykkelig er godkjent for dette. En burst mot eksterne systemer er et angrep, og en test med oppdiktede avsenderadresser mot produksjonsmål skaper tilbakespredning som fører til blokkeringslister. Et ryddig oppsett består av en lastgenerator, systemet som skal testes, og en kontrollert sink som til slutt godtar og forkaster e-postene.

## Hva en e-postlasttest skal måle

Før et verktøy kommer på tale, er det verdt å spørre hvilken størrelse som egentlig er interessant. I praksis er det fire ulike, og de krever ulike testoppsett:

1. **Mottaksrate:** Hvor mange meldinger per sekund mottar første hop via SMTP? Dette er den klassiske gjennomstrømningsmålingen og verdien lastgeneratorer leverer direkte.
2. **Sesjonslatens:** Hvor lang tid tar én SMTP-transaksjon fra tilkoblingsopprettelse til `250` etter `DATA`? Under last øker denne verdien ofte lenge før mottaksraten faller.
3. **Ende-til-ende-latens:** Hvor lang tid bruker en melding fra innlevering til levering til sinken, gjennom alle mellomliggende ledd? Dette er størrelsen brukerne merker.
4. **Køatferd:** Hvor mye vokser køen under burstet, og hvor raskt tømmes den etterpå? En gateway som godtar 50'000 e-poster og deretter arbeider i tre timer, består mottakstesten, men stryker likevel.

En test som bare måler mottaksraten, sier lite om et flertrinnsmiljø med gateway, krypteringsnivå og målserver. Særlig der er ende-til-ende-perspektivet avgjørende.

## Verktøy under Linux

**smtp-source og smtp-sink** fra Postfix-pakken er standarden for rå SMTP-last og tilgjengelig på praktisk talt alle systemer der Postfix er installert. `smtp-source` genererer meldinger med justerbar størrelse, parallellitet og antall, `smtp-sink` er motstykket: en SMTP-server som godtar og forkaster alt. En burst på 10'000 e-poster med 50 parallelle sesjoner og meldinger på 5 KB er en énlinskommando:

```bash
time smtp-source -s 50 -m 10000 -l 5120 -c -f last@test.example -t senke@test.example gateway.test.example:25
```

Alternativet `-c` teller sendte meldinger fortløpende, `time` gir total varighet og dermed raten. Viktige begrensninger: `smtp-source` måler ingen latenspersentiler, og meldingene er syntetisk ensartede. For spørsmålet «hvor mye godtar systemet maksimalt» er det likevel førstevalget, fordi det selv på svak maskinvare genererer titusenvis av meldinger per minutt og generatoren praktisk talt aldri blir flaskehalsen.

**Postal** er den klassiske dedikerte e-postserverbenchmarken under Linux. Den varierer avsender, mottaker og meldingsstørrelse automatisk, holder en målraten over lange tidsperioder og skriver statistikk per minutt. Dermed egner den seg bedre enn `smtp-source` for soak-tester, altså vedvarende last over timer. Den tilhørende `bhm` (Black Hole Mailer) overtar rollen som sink. Postal er gammelt, men bygget nettopp for dette og finnes i pakkekildene til de fleste distribusjoner.

**swaks** er ingen lastgenerator, men hører hjemme i enhver testplan. Den utfører én enkelt SMTP-transaksjon med full kontroll over hvert trinn: autentisering, STARTTLS, vilkårlige headere, vedlegg. Før hver lasttest bør det kjøres en swaks-runde som funksjonstest, slik at burstet ikke feiler på grunn av feil mottaker eller et TLS-problem og gjør målingen verdiløs. I en løkke med `xargs -P` kan swaks også misbrukes som en liten lastgenerator, men for titusenvis av e-poster er prosessoverheaden for stor.

**Egne skript** i Python (smtplib, aiosmtplib) eller Go er veien å gå når testen trenger realistiske meldingsmikser: ulike størrelser, ekte vedlegg, varierende antall mottakere per transaksjon og målrettede feiltilfeller. Innsatsen er større, men skriptet måler nøyaktig det eget miljø senere vil se, og kan skrive tidsstempler per melding for latensevalueringen.

## Verktøy under Windows

**Apache JMeter** er første anbefaling under Windows. Den innebygde SMTP Sampler støtter Auth, STARTTLS, vedlegg og EML-filer som meldingskilde, og JMeter-mekanismen leverer det Postfix-verktøyene mangler: trådgrupper for trinnvise lastprofiler, responstidspersentiler, feilsatser og rapporter. For burster utover noen få tusen e-poster per minutt gjelder den vanlige JMeter-regelen: Bruk GUI kun til å opprette testplanen, og kjør selve målingen i CLI-modus, ellers måler man brukergrensesnittet.

**PowerShell med MailKit** er veien med innebygde midler. Det tidligere vanlige `Send-MailMessage` er av Microsoft selv merket som foreldet, og de anbefaler overgang; MailKit kan lastes via NuGet og parallelliseres med Runspaces fra PowerShell 7. Realistisk er noen hundre til noen få tusen e-poster per minutt, nok for funksjons- og regresjonstester, men for lite til måling av maksimal last. Fordelen: Skriptet kjører uten ekstra installasjon på enhver administratorarbeidsstasjon og kan skrive resultater direkte som CSV for evaluering.

**Microsoft Exchange Load Generator (LoadGen)** var i årevis det offisielle verktøyet for å belaste Exchange-miljøer med simulerte brukerprofiler (Outlook, ActiveSync, OWA). Microsoft har ikke videreutviklet det etter Exchange 2013 og har fjernet nedlastingen. For ren SMTP-last var LoadGen uansett feil verktøy; den som i dag vil simulere Exchange-postboksbelastning, står uten et offisielt verktøy og tester SMTP-veien bedre direkte.

**WSL** fortjener et eget punkt: Den som sitter på en Windows-maskin, men trenger Linux-verktøy, kan installere `smtp-source` og Postal i en WSL-distribusjon og får dermed hele Linux-verktøykassen uten en separat test-VM. For lastene som diskuteres her, er WSL-nettverksveien ingen relevant flaskehals.

## Sammenligning

| Verktøy | Plattform | Styrke | Begrensning |
|---|---|---|---|
| smtp-source / smtp-sink | Linux (Postfix) | Maksimal rålast med minimal innsats, generator og sink fra samme leverandør | Ingen latenspersentiler, ensartede meldinger |
| Postal / bhm | Linux | Vedvarende last med målrate, varierende meldinger, minuttsstatistikk | Utdatert verktøysett, bygg evalueringen selv |
| swaks | Linux, Windows (Perl) | Fullt kontrollerbar enkelttest, ideell som funksjonssjekk før burstet | Ingen lastgenerator |
| JMeter (SMTP Sampler) | Windows, Linux (Java) | Lastprofiler, persentiler, rapporter, EML-meldingskilder | Java-overhead, GUI-felle ved høye rater |
| PowerShell + MailKit | Windows | Uten ekstra installasjon på enhver administrator-PC, CSV-utdata | Begrenset gjennomstrømning, parallellisering må bygges selv |
| Eget skript (Python/Go) | begge | Realistisk meldingsmiks, egne målepunkter | Utviklingsarbeid, valider generatoren selv |

## Sinken: hvor skal e-postene?

Den undervurderte halvdelen av testoppsettet er målet. Tre varianter har vist seg å fungere godt:

- **smtp-sink** eller `bhm` som svart hull: godtar alt, forkaster alt og måler den rene transportkjeden. `smtp-sink` kan ved behov generere kunstige svarforsinkelser og feilkoder, og dermed også teste hvordan testsystemet oppfører seg mot et langsomt eller vrangt mål.
- **Postfix med discard-transport** som mer realistisk sink når målet selv skal være en fullverdig SMTP-server med køhåndtering.
- **Noen få ekte seed-postbokser** i tillegg til sinken, for stikkprøvevis å kontrollere at meldinger ankommer innholdsmessig intakte, inkludert krypterings- eller signaturnivå.

Verktøy med webgrensesnitt, som Mailpit, er beregnet på utvikling og blir raskt selv flaskehalsen ved titusenvis av e-poster. De er uegnet som sink for en lasttest; målingen ville måle analyseverktøyet i stedet for testsystemet.

## Planlegg testen

En pålitelig test kjører i tre trinn, hvert med sitt eget spørsmål:

1. **Baseline:** En moderat, kjent last (omtrent 10 prosent av forventet topp) i noen minutter. Den gir referanseverdier for latens og ressursbruk og avdekker konfigurasjonsfeil før de forsvinner i burstmålingen.
2. **Burst:** Selve målingen av toppbelastning, for eksempel 10'000 til 50'000 e-poster så raskt som mulig eller med definert målrate. Flere kjøringer med økende parallellitet viser hvor mottaksraten flater ut og latensen tipper.
3. **Soak:** Den forventede dagslasten over flere timer. Først her viser minnelekkasjer, fulle spool-partisjoner, loggrotasjon under last og tilkoblingsgrenser seg – forhold som et kort burst aldri når.

For meldingsmiksen gjelder: så realistisk som nødvendig. En måling med utelukkende tekst-e-poster på 5 KB overvurderer gjennomstrømningen i et miljø der hverdagen består av PDF-vedlegg, med en betydelig faktor. En blanding fra egen bestand er fornuftig, for eksempel 70 prosent små, 25 prosent med typisk vedlegg og 5 prosent store. TLS må også inngå i testen dersom produksjonen bruker TLS: Handshaken koster betydelig mer per forbindelse enn selve meldingsoverføringen, og generatorer som åpner en ny forbindelse for hver e-post, måler ellers primært TLS-termineringen.

For senere evaluering får hver testmelding en entydig markør, enklest en egen header som `X-Loadtest-Id` med løpenummer og tidsstempel, samt en gjenkjennelig emnekonvensjon. Da kan testkjøringer enkelt skilles fra hverandre og fra øvrig trafikk i loggene, og test-e-postene kan fjernes målrettet fra karantener og journaler etter kjøringen.

Tre punkter som regelmessig blir glemt i planleggingen: For det første ratebegrensninger og tarpitting på testveien; en gateway som bremser etter 100 e-poster per minutt per kilde-IP, tester ellers bare sin egen begrensning (unnta den målrettet for måling av maksimal last, og la den bevisst være aktiv for realitetssjekken). For det andre DNS: Dersom testsystemet slår opp mottakerdomener eller utfører DNSBL-oppslag for hver melding, må en resolver inngå i testmiljøet, ellers måler testen oppstrøms DNS. For det tredje generatoren selv: Før første kjøring mot målsystemet bør generatoren kjøres direkte mot sinken for å dokumentere at den i det hele tatt kan generere målraten.

## Evaluering

Måleverdiene fra lastgeneratoren er bare halve sannheten, fordi de viser innlevererens perspektiv. Den andre halvparten står i loggene til testsystemet.

Under Postfix leverer e-postloggen feltene `delay` og `delays`, sistnevnte delt opp etter tid i køen, tilkoblingsopprettelse og overføring. En evaluering over en testkjøring er mulig med innebygde verktøy:

```bash
grep "status=sent" /var/log/mail.log |
  grep -o "delay=[0-9.]*" | cut -d= -f2 | sort -n |
  awk '{a[NR]=$1} END {print "n="NR, "p50="a[int(NR*0.5)], "p95="a[int(NR*0.95)], "p99="a[int(NR*0.99)], "max="a[NR]}'
```

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

Forskjellen mellom tidsstemplene for RECEIVE- og DELIVER-hendelsen til samme MessageId gir ende-til-ende-latensen per melding; eksportert som CSV kan persentilfordelingen beregnes ut fra dette.

Ved tolkning teller tre grunnprinsipper. For det første: Persentiler i stedet for gjennomsnitt. Et gjennomsnitt på to sekunder kan bety at alt tar to sekunder, eller at 95 prosent er ferdige på et halvt sekund mens resten henger i køen; p50, p95 og p99 skiller disse tilfellene. For det andre: Pivotér SMTP-svarkoder. Fordelingen av 4xx-svar over tid viser når systemet begynner å bremse, og hvilke koder det er (tilkoblingsgrense, købeskyttelse, greylisting), viser hvilken mekanisme som griper inn først. For det tredje: Plott kødybden over tid, under Postfix med `qshape` eller `postqueue -j`, på Exchange med `Get-Queue` med minuttsintervall. Arealet under denne kurven, ikke mottaksraten, avgjør om miljøet håndterer et burst eller bare lagrer det.

Parallelt med e-postloggene må systemmetrikkene til testsystemet inngå i evalueringen: CPU, I/O-ventetider på spool-partisjonen, filbeskrivere og tilkoblingstellere. Det vanligste funnet i flertrinnsmiljøer er at det ikke er e-postprosessen som begrenser, men et innholdsinspeksjonstrinn (virusskanner, krypteringsmodul, DLP) med fast antall workere. Nettopp slike funn er testens egentlige verdi: De peker på justeringsmuligheten før produksjonen finner den.

## Konklusjon

For en rask måling av maksimal last under Linux kommer man ikke utenom `smtp-source` med `smtp-sink`; Postal utfyller tilfellet med vedvarende last. Under Windows leverer JMeter den mest komplette målingen, PowerShell med MailKit dekker funksjons- og regresjonstester, og WSL henter ved behov Linux-verktøyene til administratorarbeidsstasjonen. Viktigere enn verktøyet er planen: separat måling av mottak, latens og køatferd, en realistisk meldingsmiks, en merket testkjøring og en evaluering som inkluderer persentiler og loggene til målsystemet i stedet for kun generatorens teller.

## Kilder

1.  [smtp-source(1), Postfix-manual](https://www.postfix.org/smtp-source.1.html): Referanse for lastgeneratoren med alle alternativer for parallellitet, meldingsstørrelse og TLS.

2.  [smtp-sink(1), Postfix-manual](https://www.postfix.org/smtp-sink.1.html): Referanse for e-postsinken, inkludert kunstige forsinkelser og feilsvar.

3.  [Postal-dokumentasjon, Russell Coker](https://doc.coker.com.au/projects/postal/): Beskrivelse av e-postserverbenchmarken med målrate, meldingsvariasjon og bhm-sink.

4.  [swaks, John Jetmore](https://www.jetmore.org/john/code/swaks/): SMTP-funksjonstesteren for forhåndssjekk av hver testvei.

5.  [Apache JMeter Component Reference: SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Funksjonsomfanget til SMTP Sampler, inkludert Auth, TLS og EML-kilder.

6.  [Send-MailMessage, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Offisiell merknad fra Microsoft om at cmdleten er foreldet, med henvisning til alternativer som MailKit.

7.  [MailKit, Jeffrey Stedfast](https://github.com/jstedfast/MailKit): .NET-e-postbiblioteket for egne utsendelsesskript under PowerShell 7.

8.  [Get-MessageTrackingLog, Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog): Referanse for evaluering av Exchange Message Tracking Log etter en testkjøring.

9.  [qshape(1), Postfix-manual](https://www.postfix.org/qshape.1.html): Verktøy for analyse av køfordelingen under og etter burstet.
