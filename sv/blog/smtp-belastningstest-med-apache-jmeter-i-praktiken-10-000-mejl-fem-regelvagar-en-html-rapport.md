---
title: "SMTP-belastningstest med Apache JMeter i praktiken: 10 000 mejl, fem regelvägar, en HTML-rapport"
navTitle: "JMeter-belastningstest"
description: "Ett genomfört belastningstest från A till Ö: testplan med meddelandemix längs regelvägarna i en krypteringsgateway, portabel installation utan installation, 10 000 mejl i en burst och utvärdering med JMeter HTML-rapporten, inklusive de problem som faktiskt uppstod."
date: "2026-08-24"
kategorie: "SMTP och e-postflöde"
timeToRead: "11 min lästid"
themen:
  - smtp-mailflow
  - testing
  - totemomail
produkte:
  - "uebergreifend"
  - "totemomail"
  - "apache-james"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - mail-lasttest-tools-linux-windows-vergleich
image: "../images/jmeter-report-dashboard.png"
slug: "smtp-belastningstest-med-apache-jmeter-i-praktiken-10-000-mejl-fem-regelvagar-en-html-rapport"
translationId: "article-fc3f25272e051f92"
aiPrompt: |
  Du bist mein Assistent für Mail-Lasttests mit Apache JMeter. Hilf mir Schritt für Schritt, einen SMTP-Lasttest aufzubauen: portables Setup (JRE + JMeter ohne Installation), lokale SMTP-Senke mit aiosmtpd, Testplan mit Thread Group, Throughput Controllern für den Nachrichtenmix und SMTP Samplern, Lauf im CLI-Modus mit HTML-Report und Auswertung der Perzentile pro Nachrichtenklasse. Frage zuerst nach Zielsystem, Nachrichtenklassen und gewünschtem Volumen.
translationOf: jmeter-smtp-lasttest-html-report
translationSourceHash: 26c09e391d2252b6203dceb5dc45edd23beba797820fe0b95273bf48a9afc181
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:24:19.424Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/smtp-belastningstest-med-apache-jmeter-i-praktiken-10-000-mejl-fem-regelvagar-en-html-rapport
---

# SMTP-belastningstest med Apache JMeter i praktiken: 10 000 mejl, fem regelvägar, en HTML-rapport

[Översiktsartikeln om e-postbelastningstester](/blog/mail-lasttest-tools-linux-windows-vergleich) jämförde verktygen och skisserade testplanen. Här följer det praktiska genomförandet: ett komplett JMeter-belastningstest med 10 000 mejl, en meddelandemix längs verkliga gateway-regelvägar och HTML-rapporten som utvärdering. Alla värden som visas kommer från den faktiska körningen, inklusive de fel som uppstod på vägen.

Scenariot är modellerat efter ett verkligt projekt: En e-postkrypteringsgateway baserad på Apache James (Totemomail) är ansluten som en smarthost-slinga bakom Exchange Online och avgör för varje meddelande om kryptering, signering och specialroutning. Mailet-regeluppsättningen har flera vägar för detta: ämnestriggers som (sec), (sign) och (unsec), nyckelord som VERTRAULICH för routning till en branschgateway samt standardvägen med certifikatkontroll och reservväg till klartext. Ett belastningstest som bara levererar en enda meddelandetyp skulle alltid mäta samma väg genom detta regelverk; testplanen modellerar därför fem klasser vars blandningsförhållande motsvarar den förväntade trafiken.

Viktigt för tolkningen: Den här testplanen skapar belastningsbilden av många oberoende avsändare, eftersom JMeter öppnar en egen anslutning för varje meddelande (bakgrunden beskrivs i avgränsningen i slutet). För att visa att ett regelverk fungerar korrekt och tillräckligt snabbt under parallell blandad trafik är detta rätt mönster. Planen modellerar däremot inte topplasten från en enskild massavsändare med öppna sessioner; för denna belastningsbild är `smtp-source` från [översiktsartikeln](/blog/mail-lasttest-tools-linux-windows-vergleich) rätt verktyg.

## De viktigaste alternativen för jmeter

Som orientering först de kommandoradsalternativ som förekommer i denna artikel, fritt översatta från dokumentationen:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Alternativ | Betydelse |
|---|---|
| `-n` | CLI-läge (non-GUI): kör testplanen utan grafiskt gränssnitt |
| `-t datei` | Sökväg till JMX-filen med testplanen |
| `-l datei` | Sökväg till JTL-resultatfilen där mätvärdena skrivs |
| `-e` | Skapar HTML-dashboardrapporten direkt efter körningen |
| `-o verzeichnis` | Målmapp för rapporten; måste vara tom eller ännu inte finnas |
| `-g datei` | Skapar rapporten i efterhand från en befintlig JTL-fil, utan ny körning |
| `-J<property>=<wert>` | Anger en JMeter-property endast för detta anrop |

</details>

Den fullständiga listan visas av `jmeter -?`; alternativen beskrivs i kapitlet om non-GUI-drift i [JMeter User's Manual](https://jmeter.apache.org/usermanual/get-started.html).

## Uppbyggnaden: inget behöver installeras

Testet kördes på en Windows-dator utan Java och utan JMeter. Båda kan användas portabelt, vilket är avgörande på administratörsarbetsplatser med begränsade installationsrättigheter: Temurin-JRE som ZIP från Adoptium, JMeter som ZIP från apache.org, packa upp båda, ange `JAVA_HOME` till JRE-mappen, klart.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `export JAVA_HOME=…` | Pekar på den uppackade JRE-mappen; JMeter hittar Java-körmiljön via den utan installation |
| `export PATH=…` | Lägger JRE-binärfilerna först i sökvägen |
| `-n` | CLI-läge utan grafiskt gränssnitt |
| `-t gateway-lasttest.jmx` | Testplanen som ska köras |
| `-l lauf.jtl` | Resultatfil med mätvärdena för varje sampler |
| `-e` | Skapar HTML-rapporten direkt efter körningen |
| `-o report` | Målmapp för rapporten |

</details>

Som sänka användes en lokal SMTP-blackbox baserad på aiosmtpd, drygt 40 rader Python: Den tar emot varje meddelande med `250`, kastar innehållet, räknar och tilldelar varje mejl en klass utifrån ämnesraden. Denna oberoende räkning på mottagarsidan är testets kontrollförsök; om generatorns och sänkans antal inte stämmer överens har något gått förlorat på vägen.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Hämta ämnesraden för klassstatistiken från headern,
        # Innehållet kastas
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Viktigt för tolkningen: Generatorn och sänkan kördes på samma dator, utan TLS och utan något nätverk emellan. De uppmätta siffrorna är därför inget uttalande om en gateway, utan generatorns självtest från översiktsartikeln: belägget för att belastningsuppbyggnaden över huvud taget kan skapa målhastigheten, samt den övre gräns som senare mätningar mot det verkliga testsystemet jämförs med.

## Testplanen: fem meddelandeklasser, ett blandningsförhållande

Planens kärna är en Thread Group med 20 trådar, 10 sekunders ramp-up och 500 loopar, alltså 10 000 iterationer. Under den finns fem Throughput Controllers i läget "Percent Executions", var och en med exakt en SMTP Sampler:

| Klass (sampler-etikett) | Andel | Regelväg i gatewayen |
|---|---|---|
| 01 Standard utan trigger | 60 % | AutoGenerated-kontroll, certifikatkontroll, reservväg till klartext |
| 02 Trigger (sec) | 15 % | TRE-envelope för mottagare utan certifikat |
| 03 Trigger (sign) | 10 % | Certificate Exchange: signera, skicka med nyckel |
| 04 Nyckelord VERTRAULICH | 10 % | Specialroutning till branschgatewayen |
| 05 Trigger (unsec) | 5 % | Klartext tvingas |

Uppdelningen i fem separata samplers i stället för en sampler med variabel ämnesrad har ett konkret skäl: HTML-rapporten grupperar alla nyckeltal efter sampler-etiketten. Fem etiketter ger fem rader i statistiken med egna percentiler per klass; en enda sampler med CSV-matad ämnesrad skulle ge en enda samlingsrad, och skillnaden mellan regelvägarna skulle vara osynlig i utvärderingen.

Varje sampler fyller i de vanliga fälten: målhost och port som användardefinierade variabler (`${zielhost}`, `${zielport}`), så att samma plan kan köras mot sänka, testmiljö eller PreProd utan ändringar, samt avsändare, mottagare, ämnesrad med en tydlig markering (här ordet LOADTEST i ämnesraden) och en textkropp på cirka 1 till 2 KB. Alternativet "Include timestamp in subject" lägger till leveranstidpunkten i millisekunder; vid en senare körning mot ett verkligt system i flera steg kan end-to-end-latensen per meddelande beräknas från detta tillsammans med sänkans mottagningstidpunkter.

Ett fel från denna körning som kan generaliseras: Det första försöket misslyckades med 10 000 fel på 10 sekunder, alla med `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` i stället för ett SMTP-svar. Orsaken var en handbyggd JMX-fil där samplerns header-lista saknades; samplern kräver denna property, även om den är tom. Lärdomen är mindre den specifika propertyn än mönstret: Klicka ihop och spara testplaner i GUI:t, skriv inte XML för hand, och gör en mycket liten körning före varje burst och kontrollera på sänkan att ämnesrad och innehåll verkligen kommer fram. En felräknare på 100 procent vid 0 ms svarstid betyder nästan alltid att felet uppstår före nätverket, alltså att testet aldrig nådde målsystemet.

## Körningen

Själva mätningen körs i CLI-läge; GUI:t används endast som editor. Ett enda anrop skapar körning, rådata och rapport:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-n` | CLI-läge: testplanen körs utan GUI, endast Summariser skriver till konsolen |
| `-t gateway-lasttest.jmx` | Testplanen som skapats i GUI:t |
| `-l lauf-10k.jtl` | Körningens rådata; rapporten kan senare genereras på nytt från denna fil |
| `-e` | Genererar rapporten direkt efter körningen |
| `-o report-10k` | Målmapp för HTML-rapporten |

</details>

Summariser på konsolen visar förloppet live, och körningens slutresultat:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10 000 meddelanden på 12,8 sekunder, i genomsnitt 782 meddelanden per sekund, inga fel. Sänkan bekräftade oberoende exakt 10 000 mottagna mejl med mixen 6000 / 1500 / 1000 / 1000 / 500; blandningsförhållandet hos Throughput Controllers stämde alltså exakt på meddelandenivå.

## HTML-rapporten

Argumentet för JMeter jämfört med smalare generatorer som smtp-source är utvärderingen, och Dashboard-rapporten levererar den utan extra arbete:

![JMeter-dashboard för körningen: APDEX 1.000 för alla fem klasser, Requests Summary 100 procent PASS, statistiktabell med percentiler per meddelandeklass](../images/jmeter-report-dashboard.png)

Statistiktabellen är rapportens viktigaste del. Per sampler-etikett, alltså per meddelandeklass, visas antal, felkvot, genomsnitt, median, 90:e, 95:e och 99:e percentilen, maximum och genomströmning. I den konkreta körningen: median 7 ms, p95 på 11 ms, p99 på 12 ms, maximum 27 ms, och i praktiken identiskt för alla fem klasser. För en lokal sänka som behandlar varje meddelande likadant är detta precis den förväntade bilden och samtidigt referensvärdet: Om samma plan senare körs mot den verkliga gatewayen och (sec)-klassen plötsligt visar multiplar av standardmedianen, är det krypteringsvägens extra arbete, rent isolerat per regelgren.

APDEX-blocket ovanför sammanfattar samma sak i ett tal per klass (här överallt 1.000, eftersom alla svar låg långt under toleranströskeln på 500 ms); trösklarna kan anpassas till egna tjänstemål i rapportens properties. Errors-blocket är tomt i denna körning, men är den första platsen att titta på vid test mot verkliga system: Det grupperar fel efter svarstext, så att strypning av målsystemet med `421` direkt kan skiljas från anslutningsavbrott.

Även här finns ett typiskt utvärderingsfel, och det berör varje kort burst: Rapportens tidsserie-diagram arbetar som standard med en granularitet på en minut. En körning på 13 sekunder kollapsar därmed till en enda datapunkt, och kurvorna under "Charts" ser ut som ett mätfel. Rapporten kan återskapas från den befintliga JTL-filen utan ny körning med finare upplösning:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-g lauf-10k.jtl` | Skapar rapporten från den befintliga JTL-filen utan att köra testet igen |
| `-o report-fein` | Ny målmapp; den befintliga rapportmappen förblir oförändrad |
| `-Jjmeter.reportgenerator.overall_granularity=1000` | Sätter diagramgranulariteten för detta anrop till 1 000 ms i stället för standardminuten |

</details>

Med sekundgranularitet blir den enskilda punkten den faktiska belastningskurvan:

![Hits per Second med 1-sekundsgranularitet: ökning under 10-sekunders ramp-up till en platå runt 840 meddelanden per sekund, sedan ett brant fall vid testets slut](../images/jmeter-report-hits-per-second.png)

Kurvan visar 10 sekunders ramp-up, en platå runt 840 meddelanden per sekund och fallet i slutet när de första trådarna har avslutat sina 500 loopar. För tolkningen är platån viktig, inte genomsnittet över hela körningen: Snittet på 782/s inkluderar ramp-up och utfasning och underskattar den uppnådda varaktiga hastigheten.

## Vad denna körning visar och inte visar

Efter denna körning är följande belagt: Testplanen är funktionellt korrekt (mycket liten körning med innehållskontroll på sänkan), blandningsförhållandet stämmer exakt och generatorn klarar minst 840 meddelanden per sekund utan TLS på denna dator. Den som vill testa en gateway dimensionerad för 100 mejl per sekund har därmed en reservfaktor på åtta och kan med gott samvete tillskriva flaskhalsar målsystemet.

Allt annat är inte belagt, och denna avgränsning hör till varje testrapport: inget uttalande om kostnader för TLS-handshakes (den verkliga vägen använder STARTTLS), inget om gatewayens köbeteende, inget om regelvägarnas bearbetningstid. För detta pekar samma plan, med ändrade variabler `zielhost`/`zielport`, mot gatewayens testmiljö; utvärderingen sker sedan identiskt, kompletterad med gateway-loggarna och köobservationen från översiktsartikeln. Just denna återanvändbarhet – en plan för sänka, testmiljö och PreProd med identisk utvärdering – är det egentliga skälet att en gång lägga arbetet på en ren JMeter-plan.

En begränsning i själva verktyget hör också till avgränsningen: JMeter kan inte hålla SMTP-sessioner öppna. SMTP Sampler öppnar en ny anslutning för varje meddelande, genomgår EHLO, eventuellt STARTTLS och AUTH och avslutar den efter exakt en transaktion med QUIT. De 840 meddelandena per sekund omfattar alltså en fullständig anslutningsuppbyggnad per meddelande. En massavsändare som skickar hundratals meddelanden över en öppen session skapar en annan belastningsbild i gatewayen med färre anslutningar och fler transaktioner per anslutning, och anslutningsgränser slår därför till tidigare med JMeter-belastning. Orsaken är ramverkets arkitektur: JMeter mäter varje sampler som en fristående, oberoende enhet så att timers, assertions och percentiler fungerar likadant för alla protokoll som stöds, och SMTP Sampler är ett lager ovanpå JavaMail-biblioteket, som som klient-API ansluter och kopplar ned för varje sändning. Återanvändning av anslutningar som med HTTP Sampler och Keep-Alive finns inte för SMTP. För belastningsbilden av en bulkavsändare med öppen session lämpar sig `smtp-source` eller ett eget skript bättre; verktygsjämförelsen i översiktsartikeln placerar detta i sitt sammanhang.

## Källor

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Referens för samplerfälten, inklusive header, timestamp-alternativ och EML-sändning.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): Skapande av HTML-rapporten från körningen eller i efterhand från JTL-filen, inklusive properties för granularitet och APDEX.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): Så fungerar Throughput Controller i läget Percent Executions för meddelandemixen.

4.  [aiosmtpd, dokumentation](https://aiosmtpd.aio-libs.org/): Den asyncio-baserade SMTP-servern som gör att sänkan kan skapas på några rader Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): Portabla JRE-arkiv för att köra JMeter utan Java-installation.

6.  [Apache JMeter: Getting Started, Non-GUI Mode](https://jmeter.apache.org/usermanual/get-started.html): Översikt över kommandoradsalternativen för CLI-drift, inklusive `-n`, `-t`, `-l`, `-e`, `-o`, `-g` och `-J`.
