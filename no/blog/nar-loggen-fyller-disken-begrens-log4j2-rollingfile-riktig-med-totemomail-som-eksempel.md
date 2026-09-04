---
title: "Når loggen fyller disken: Begrens log4j2 RollingFile riktig, med totemomail som eksempel"
navTitle: "log4j2 diskplass"
description: "Et loggvolum som fylles opp, kan i verste fall lamme hele gatewayen. Hvorfor kombinasjonen av tids- og størrelsesrotasjon uten %i skaper én enkelt enorm fil, hvordan strategy.max begrenser oppbevaringen, hvilken rolle loggnivået spiller, og hvor totemomail skjuler disse verdiene."
date: "2026-09-04"
kategorie: "Totemomail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "troubleshooting"
  - "storage"
slug: "nar-loggen-fyller-disken-begrens-log4j2-rollingfile-riktig-med-totemomail-som-eksempel"
translationId: "article-c400eee99d90052d"
translationOf: log4j2-rollingfile-plattenplatz-totemomail
url: https://rafaelpfister.ch/no/blog/nar-loggen-fyller-disken-begrens-log4j2-rollingfile-riktig-med-totemomail-som-eksempel
translationSourceHash: 39952348654f81231356634fc8b434cbfecdea73118db7ff1add02720283792b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:18:34.856Z
translationReview: automatic
---

# Når loggen fyller disken: Begrens log4j2 RollingFile riktig, med totemomail som eksempel

En Java-basert e-postgateway skriver overraskende store mengder i DEBUG-modus. Én enkelt dag med høy belastning kan generere flere gigabyte med sporingslogger, og dersom loggvolumet er lite dimensjonert, fylles det opp. Konsekvensen er ubehagelig: Java-prosessen kan ikke lenger skrive til loggen sin, loggrammeverket havner i en feiltilstand, og selv etter at det igjen er ledig plass, begynner den først å skrive igjen etter en omstart. For en e-postgateway kan en full disk dessuten forstyrre spoolingen og leveringen. Utløseren er nesten alltid en loggrotasjon som riktignok er konfigurert, men som ikke fungerer slik man antar.

Denne artikkelen forklarer rotasjonen i log4j2 nettopp på dette punktet, først generelt og deretter konkret for totemomail (som bygger på Apache James og log4j2). Kjernen er én enkelt opplysning i filmønsteret som er lett å overse.

## Slik roterer log4j2

`RollingFileAppender` i log4j2 kombinerer to byggeklosser: én eller flere **TriggeringPolicies** avgjør *når* det roteres, og en **RolloverStrategy** avgjør *hvordan* arkivfilene navngis og hvor mange som beholdes. To policyer samtidig er typisk:

- `TimeBasedTriggeringPolicy`: roterer etter tid, som regel daglig.
- `SizeBasedTriggeringPolicy`: roterer så snart den aktive filen når en bestemt størrelse, for eksempel 100 MB.

Ved rollover blir den aktive filen omdøpt og arkivert. Navnet på arkivfilen bestemmes av `filePattern`, og den inneholder to plassholdere der samspillet utgjør den avgjørende forskjellen.

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Plassholder | Betydning |
|---|---|
| `%d{...}` | Dato/tid for rollover etter det angitte mønsteret, f.eks. `%d{yyyy-MM-dd}` for dagen |
| `%i` | Den beregnede indeksen for arkivfilen, en teller som øker ved hver rollover |
| `%03i` | Den samme indeksen, nullutfylt til tre posisjoner |
| `.gz` / `.zip` på slutten av mønsteret | Arkivet komprimeres ved rollover |

</details>

Den fullstendige referansen finnes i log4j2-dokumentasjonen for Rolling File Appender; tabellen over nevner bare elementene som er vesentlige for størrelses- og tidsrotasjon.

## %i-fellen

Det er nettopp her feilen som fyller disken ligger. Den som bare navngir etter dato, altså `filePattern = trace.log.%d{yyyy-MM-dd}`, og samtidig setter en størrelsespolicy på 100 MB, får ikke mange 100 MB-filer per dag, men én enkelt fil som fortsetter å vokse ukontrollert. Størrelsesrotasjonen har ikke noe eget mål den kan skrive neste del til, fordi mønsteret ikke inneholder noen teller. Log4j2-dokumentasjonen er tydelig på dette punktet:

> When combined with a time-based triggering policy, the filePattern attribute of the Appender should contain an `%i` conversion pattern. Otherwise, the target file will be overwritten on each rollover.

Uten `%i` er kombinasjonen av tids- og størrelsesrotasjon altså feilaktig; avhengig av strategien blir filen enten overskrevet eller vokser utover den angitte størrelsen. I praksis betyr det: Grensen på 100 MB slår aldri inn, én dag med belastning skriver alt til én fil, og den blir flere gigabyte stor. Løsningen er å utvide mønsteret:

```text
filePattern = trace.log.%d{yyyy-MM-dd}.%i
```

Dermed oppretter hver rollover på 100 MB en egen indeksert fil (`trace.log.2026-09-04.1`, `.2`, `.3`), og størrelsesbegrensningen fungerer som tiltenkt.

## Oppbevaring via strategy.max

Indeksen er samtidig forutsetningen for at oppbevaringen fungerer. `DefaultRolloverStrategy` har et attributt `max`, som angir maksimalt antall arkivfiler som beholdes; over denne grensen slettes de eldste. Uten `%i` finnes det ingen indeks som `max` kan telle, og dermed slettes heller ingenting, slik at gamle daterte filer hoper seg opp.

<details class="options-details">
<summary>Alternativer forklart</summary>

| Attributt | Virkning |
|---|---|
| `max` | Maksimalt antall arkivfiler som beholdes; de eldste fjernes over denne grensen |
| `min` | Laveste indeksverdi (standard 1) |
| `fileIndex="min"` | Nyeste fil får indeks `min`, eldste `max` |
| `fileIndex="max"` (standard) | Eldste fil får indeks `min`, nyeste `max` |
| `fileIndex="nomax"` | Det slettes aldri; nye arkiver får fortløpende stigende indekser |

</details>

Størrelse og antall gir den samlede øvre grensen: 100 MB per fil ganger `max=10` begrenser loggen til rundt én gigabyte, uavhengig av hvor mye som skrives. Den som trenger finere kontroll over alder i stedet for antall, legger til en `Delete`-handling i strategien med `IfLastModified` (alder) eller `IfAccumulatedFileSize` (totalstørrelse); for de fleste tilfeller er kombinasjonen av størrelse per fil og `max` tilstrekkelig.

## Loggnivået som den egentlige mengdedriveren

Rotasjon og oppbevaring begrenser plassforbruket, men endrer ikke hvor mye som faktisk skrives. Den største påvirkningsfaktoren er loggnivået. En gateway som kjører DEBUG i produksjon, logger hvert behandlingstrinn for hver melding, og under belastning summerer dette seg til gigabyte per dag. Ved normal drift bør nivået være INFO eller høyere; DEBUG er et verktøy for en avgrenset analyse, ikke for kontinuerlig drift. Når nivået står på INFO og størrelsesrotasjonen med `%i` i tillegg er riktig satt, virker begge sammen: INFO holder dagsmengden liten, og rotasjonen begrenser selv et DEBUG-avvik.

## Hvor totemomail lagrer disse verdiene

I totemomail finnes disse innstillingene ikke i en lokal `log4j2.xml`, og det kan lett villede under feilsøking. Konfigurasjonen opprettes ved kjøretid fra Properties med prefikset `totemo.log4j2.*`, og disse Properties administreres sentralt via Management Console (området Logging + Tracking). Et søk etter `log4j2.xml` i filsystemet er derfor resultatløst; en `log4j.xml` i konfigurasjonskatalogen tilhører en medlevert komponent (openjms) og har ingenting med sporingsloggen å gjøre.

De relevante Properties og deres betydning:

<details class="options-details">
<summary>Alternativer forklart</summary>

| Property | Betydning |
|---|---|
| `totemo.log4j2.appender.a1.filePattern` | Filmønsteret; `%i` skal inn her |
| `totemo.log4j2.appender.a1.policies.size.size` | Størrelse per fil for SizeBasedTriggeringPolicy, f.eks. `100MB` |
| `totemo.log4j2.appender.a1.strategy.max` | Antall arkivfiler som beholdes |
| `totemo.log4j2.rootLogger.level` | Nivået for log4j2s root-logger |
| `totemo.log.priority` | Overordnet loggprioritet for applikasjonen, den egentlige DEBUG-bryteren |
| `totemo.tracking` | Detaljnivå for meldingssporing; `debug` genererer linjene per Mailet |

</details>

Det viktige er den doble funksjonen: log4j2-loggerne kan være satt til `warn` eller `error` og likevel skape en DEBUG-flom i sporingsloggen, fordi `totemo.log.priority` og `totemo.tracking` fungerer som egne, overordnede brytere. Den som vil redusere volumet, setter `totemo.log.priority` til INFO og `totemo.tracking` fra `debug` til `on`; det fjerner de detaljerte behandlingslinjene. Siden verdiene administreres via Console, gjelder de for hele klyngen, og noen krever omstart av instansen for å tre i kraft (dette er angitt ved den enkelte Property).

## Omstart etter at disken er fylt opp

En detalj som lett overses: Etter at disken har vært full én gang, kommer ikke loggingen tilbake av seg selv, selv om man frigjør plass. Fil-appenderen forblir i feiltilstanden til Java-prosessen starter på nytt. Dette ser man ved at gatewayen fortsatt mottar og behandler e-post (SMTP-banneret viser riktig klokkeslett), men sporingsloggen stopper på tidspunktet da disken ble full. En kontrollert omstart av instansen gjenoppretter loggingen og aktiverer samtidig endrede appender-innstillinger som det nye `filePattern`.

## Diagnose med noen få kommandoer

Den fulle partisjonen og årsaken kan raskt avgrenses. Først vises hvilket filsystem som er berørt:

```bash
df -h
```

Hvis loggvolumet er på 100 prosent, identifiserer en oppføring sortert etter størrelse hovedårsaken:

```bash
du -sh /pfad/zu/logs/* | sort -rh | head
```

Hvis det finnes én enkelt dagsfil på mange gigabyte i stedet for mange små indekserte arkiver, er det `%i`-fellen. Etter løsningen og en omstart bekrefter fillisten at rotasjonen fungerer:

```bash
ls -laht /pfad/zu/logs/trace.log*
```

Forventet er `trace.log` pluss indekserte arkiver `trace.log.<datum>.1`, `.2` og så videre, hver omtrent i den angitte maksimale størrelsen.

## Oppsummering

Den som bruker log4j2 med tids- og størrelsesrotasjon, må ha `%i` i `filePattern`, ellers vokser én enkelt fil ukontrollert og størrelsesgrensen blir virkningsløs. Via `strategy.max` (sammen med indeksen) begrenser antallet arkiver plassforbruket, og loggnivået avgjør mengden ved kilden. I totemomail ligger disse verdiene i Management Console under `totemo.log4j2.*` samt i de overordnede bryterne `totemo.log.priority` og `totemo.tracking`; etter at disken er fylt opp, må instansen startes på nytt for at loggingen skal skrive igjen.

## Kilder

1.  [Apache Logging Services: Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html): Referanse for filePattern, TriggeringPolicies og DefaultRolloverStrategy, inkludert uttalelsen om `%i` ved tidsbasert rotasjon.

2.  [Apache Logging Services: Log4j Architecture](https://logging.apache.org/log4j/2.x/manual/architecture.html): Plassering av Appender, Layout og logger-hierarki, for å forstå root-loggeren og loggnivået.
