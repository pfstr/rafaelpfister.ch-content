---
title: "När loggen fyller disken: begränsa log4j2 RollingFile korrekt med totemomail som exempel"
navTitle: "log4j2 diskutrymme"
description: "En loggvolym som fylls kan i värsta fall slå ut hela gatewayen. Varför kombinationen av tids- och storleksrotation utan %i skapar en enda enorm fil, hur strategy.max begränsar lagringen, vilken roll loggnivån spelar och var totemomail döljer dessa värden."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 min läsning"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "nar-loggen-fyller-disken-begransa-log4j2-rollingfile-korrekt-med-totemomail-som-exempel"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/sv/blog/nar-loggen-fyller-disken-begransa-log4j2-rollingfile-korrekt-med-totemomail-som-exempel
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:17:57.256Z
translationReview: automatic
---

# När loggen fyller disken: begränsa log4j2 RollingFile korrekt med totemomail som exempel

En Java-baserad e-postgateway skriver förvånansvärt stora mängder i DEBUG-läge. En enda dag med hög belastning kan skapa flera gigabyte trace-loggar, och om loggvolymen är liten fylls den. Följden är obehaglig: Java-processen kan inte längre skriva till sin logg, loggramverket hamnar i ett feltillstånd och börjar inte skriva igen förrän efter en omstart, även om utrymme har frigjorts. För en e-postgateway kan en full disk dessutom störa spoolning och leverans. Orsaken är nästan alltid en loggrotation som visserligen är konfigurerad, men inte fungerar som man antar.

Följande artikel förklarar log4j2-rotation just på denna punkt, först generellt och sedan konkret för totemomail (som bygger på Apache James och log4j2). Kärnan är en enda, lättförbisedd uppgift i filmönstret.

## Hur log4j2 roterar

`RollingFileAppender` i log4j2 kombinerar två byggstenar: en eller flera **TriggeringPolicies** avgör *när* rotation sker, och en **RolloverStrategy** avgör *hur* arkivfilerna namnges och hur många som behålls. Två policies används vanligtvis samtidigt:

- `TimeBasedTriggeringPolicy`: roterar efter tid, oftast dagligen.
- `SizeBasedTriggeringPolicy`: roterar när den aktiva filen når en viss storlek, till exempel 100 MB.

Vid rollover byter den aktiva filen namn och arkiveras. Arkivfilens namn bestäms av `filePattern`, och det innehåller två platshållare vars samspel gör den avgörande skillnaden.

<details class="options-details">
<summary>Alternativ i översikt</summary>

| Platshållare | Betydelse |
|---|---|
| `%d{...}` | Datum/tid för rollover enligt det angivna mönstret, t.ex. `%d{yyyy-MM-dd}` för dagen |
| `%i` | Det beräknade indexet för arkivfilen, en räknare som ökar vid varje rollover |
| `%03i` | Samma index, nollutfyllt till tre positioner |
| `.gz` / `.zip` i slutet av mönstret | Arkivet komprimeras vid rollover |

</details>

Den fullständiga referensen finns i log4j2-dokumentationen för Rolling File Appender; tabellen ovan nämner endast de element som är viktiga för storleks- och tidsrotation.

## %i-fällan

Här ligger just felet som fyller diskar. Den som bara namnger efter datum, alltså `filePattern = trace.log.%d{yyyy-MM-dd}`, och samtidigt sätter en storlekspolicy på 100 MB, får inte många 100 MB-filer per dag utan en enda fil som fortsätter växa obegränsat. Storleksrotationen har inget eget mål där den kan skriva nästa del, eftersom mönstret saknar en räknare. Log4j2-dokumentationen är tydlig på denna punkt:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Utan `%i` är kombinationen av tids- och storleksrotation alltså felaktig; beroende på strategi skrivs filen antingen över eller växer förbi den inställda storleken. I praktiken innebär det att gränsen på 100 MB aldrig träder i kraft, en dag med hög belastning skriver allt i en fil och den blir flera gigabyte stor. Lösningen är att komplettera mönstret:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

Därmed skapar varje rollover vid 100 MB en egen indexerad fil (`trace.log.2026-09-04.1`, `.2`, `.3`), och storleksbegränsningen fungerar som avsett.

## Lagring via strategy.max

Indexet är samtidigt förutsättningen för att lagringen ska fungera. `DefaultRolloverStrategy` har attributet `max`, som anger det maximala antalet arkivfiler som behålls; över denna gräns raderas de äldsta. Utan `%i` finns inget index som `max` kan räkna, alltså raderas ingenting och gamla daterade filer samlas på hög.

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Attribut | Effekt |
|---|---|
| `max` | Högsta antal arkivfiler som behålls; de äldsta tas bort över denna gräns |
| `min` | Lägsta indexvärde (standard är 1) |
| `fileIndex="min"` | Den nyaste filen får index `min`, den äldsta `max` |
| `fileIndex="max"` (standard) | Den äldsta filen får index `min`, den nyaste `max` |
| `fileIndex="nomax"` | Inget raderas någonsin, nya arkiv får kontinuerligt stigande index |

</details>

Storlek och antal ger tillsammans den totala övre gränsen: 100 MB per fil gånger `max=10` begränsar loggen till omkring en gigabyte, oavsett hur mycket som skrivs. Den som behöver mer detaljerad kontroll över ålder i stället för antal lägger till en `Delete`-åtgärd med `IfLastModified` (ålder) eller `IfAccumulatedFileSize` (total storlek) i strategin; i de flesta fall räcker kombinationen av storlek per fil och `max`.

## Loggnivån som den egentliga mängddrivaren

Rotation och lagring begränsar utrymmesförbrukningen, men de ändrar inte hur mycket som faktiskt skrivs. Den största hävstången är loggnivån. En gateway som körs i DEBUG i produktion loggar varje bearbetningssteg för varje meddelande, och under belastning summeras det till gigabyte per dag. För normal drift bör nivån vara INFO eller högre; DEBUG är ett verktyg för begränsad analys, inte för kontinuerlig drift. Om nivån är INFO och storleksrotationen med `%i` dessutom är korrekt inställd samverkar båda: INFO håller den dagliga mängden liten, och rotationen begränsar även en DEBUG-avvikelse.

## Var totemomail har dessa värden

I totemomail finns dessa inställningar inte i en lokal `log4j2.xml`, vilket lätt leder felsökningen fel. Konfigurationen skapas vid körning från properties med prefixet `totemo.log4j2.*`, och dessa properties hanteras centralt via Management Console (området Logging + Tracking). En sökning efter `log4j2.xml` i filsystemet ger därför inget resultat; en `log4j.xml` i konfigurationskatalogen hör till en medföljande komponent (openjms) och har inget med trace-loggen att göra.

De relevanta properties och deras betydelse:

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Property | Betydelse |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | Filmönstret; här ska `%i` finnas med |
| `totemo.log4j2.appender.a1.policies.size.size` | Storlek per fil för SizeBasedTriggeringPolicy, t.ex. `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Antal arkivfiler som behålls |
| `totemo.log4j2.rootLogger.level` | Nivån för log4j2:s root-logger |
| `totemo.log.priority` | Programmets övergripande loggprioritet, den egentliga DEBUG-omkopplaren |
| `totemo.tracking` | Detaljnivå för meddelandespårning; `debug` skapar raderna per Mailet |

</details>

Det viktiga är dubbelheten: log4j2-loggrarna kan vara inställda på `warn` eller `error` och ändå skapa en DEBUG-flod i trace-loggen, eftersom `totemo.log.priority` och `totemo.tracking` fungerar som egna, överordnade brytare. Den som vill minska volymen sätter `totemo.log.priority` till INFO och `totemo.tracking` från `debug` till `on`; det tar bort de utförliga bearbetningsraderna. Eftersom värdena hanteras via Console gäller de för hela klustret, och vissa kräver en omstart av instansen för att träda i kraft (det anges vid respektive property).

## Omstarten efter att disken fyllts

En detalj som är lätt att missa: När disken väl har varit full återkommer loggningen inte av sig själv, även om man frigör utrymme. Fil-appendaren förblir i sitt feltillstånd tills Java-processen startas om. Det märks genom att gatewayen fortfarande tar emot och bearbetar e-post (SMTP-bannern visar korrekt tid), men trace-loggen stannar vid tidpunkten då disken fylldes. En kontrollerad omstart av instansen återställer loggningen och aktiverar samtidigt ändrade appender-inställningar som det nya `filePattern`.

## Diagnos med några få kommandon

Den fulla partitionen och orsaken kan snabbt avgränsas. Först ser du vilket filsystem som är påverkat:

```bash
df -h
```

Om loggvolymen är på 100 procent visar en lista sorterad efter storlek huvudorsaken:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Om det finns en enskild daglig fil på många gigabyte i stället för många små indexerade arkiv är det `%i`-fällan. Efter korrigeringen och en omstart bekräftar fillistan att rotationen fungerar:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Du förväntar dig `trace.log` plus indexerade arkiv `trace.log.<datum>.1`, `.2` och så vidare, vart och ett ungefär i den inställda maximala storleken.

## Sammanfattning

Den som kör log4j2 med tids- och storleksrotation måste ha `%i` i `filePattern`, annars växer en enskild fil obegränsat och storleksgränsen förblir verkningslös. Med `strategy.max` (tillsammans med indexet) begränsar antalet arkiv utrymmesförbrukningen, och loggnivån avgör mängden vid källan. I totemomail finns dessa värden i Management Console under `totemo.log4j2.*` samt i de överordnade brytarna `totemo.log.priority` och `totemo.tracking`; efter att disken fyllts krävs en omstart av instansen för att loggningen ska börja skriva igen.

## Källor

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): Referens för filePattern, TriggeringPolicies och DefaultRolloverStrategy, inklusive uttalandet om `%i` vid tidsbaserad rotation.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): Klassificering av Appender, Layout och logger-hierarki, för förståelsen av root-logger och loggnivå.
