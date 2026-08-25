---
title: "SMTP-belastningstest med Apache JMeter i praktiken: 10 000 mejl, fem regelvägar, en HTML-rapport"
navTitle: "JMeter-belastningstest"
description: "Ett genomfört belastningstest från A till Ö: testplan med en meddelandemix längs regelvägarna i en krypteringsgateway, portabel installation utan installation, 10 000 mejl i en burst och utvärdering via JMeter HTML-rapporten, inklusive de fallgropar som faktiskt uppstod."
date: "2026-08-24"
kategorie: "SMTP och e-postflöde"
timeToRead: "11 min läsning"
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
url: https://rafaelpfister.ch/sv/blog/smtp-belastningstest-med-apache-jmeter-i-praktiken-10-000-mejl-fem-regelvagar-en-html-rapport
translationSourceHash: a41d58b7a4a717db179b3fec1ef8fac7961ff3ee12069f65627ddb48338aef0a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:09:47.485Z
translationReview: required
---

# SMTP-belastningstest med Apache JMeter i praktiken: 10 000 mejl, fem regelvägar, en HTML-rapport

[Översiktsartikeln om e-postbelastningstester](/blog/mail-lasttest-tools-linux-windows-vergleich) jämförde verktygen och skisserade testplanen. Den här artikeln sätter teorin på prov: ett helt genomfört JMeter-belastningstest med 10 000 mejl, en meddelandemix längs verkliga gateway-regelvägar och HTML-rapporten som utvärdering. Alla värden som visas kommer från den faktiska körningen, inklusive de fel som uppstod på vägen.

Scenariot är baserat på ett verkligt projekt: En e-postkrypteringsgateway baserad på Apache James (Totemomail) ligger som en smarthost-slinga bakom Exchange Online och avgör för varje meddelande kryptering, signering och specialroutning. Mailet-rulesetet har flera vägar för detta: ämnestriggers som (sec), (sign) och (unsec), nyckelord som VERTRAULICH för routning till en branschgateway samt standardvägen med certifikatkontroll och klartextfallback. Ett belastningstest som bara levererar en enda typ av meddelande skulle alltid mäta samma väg genom regelverket; testplanen omfattar därför fem klasser vars blandningsförhållande motsvarar den förväntade trafiken.

## Upplägget: inget behöver installeras

Testet kördes på en Windows-dator utan Java och utan JMeter. Båda kan köras portabelt, vilket är avgörande på administratörsarbetsplatser med begränsade installationsrättigheter: Temurin-JRE som ZIP från Adoptium, JMeter som ZIP från apache.org, packa upp båda, sätt `JAVA_HOME` till JRE-katalogen, klart.

```bash
export JAVA_HOME="$PWD/jdk-21-jre"
export PATH="$JAVA_HOME/bin:$PATH"
./apache-jmeter-5.6.3/bin/jmeter -n -t gateway-lasttest.jmx -l lauf.jtl -e -o report
```

Som mottagare användes en lokal SMTP-blackbox baserad på aiosmtpd, drygt 40 rader Python: Den accepterar varje meddelande med `250`, kastar innehållet, räknar och tilldelar varje mejl en klass utifrån ämnesraden. Denna oberoende räkning på mottagarsidan är testets kontrollförsök; om generatorns och mottagarens siffror inte stämmer överens har något gått förlorat på vägen.

```python
from aiosmtpd.controller import Controller

class SinkHandler:
    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        # Hämta ämnesraden för klassstatistiken från headern,
        # Innehållet kastas bort
        return "250 Message accepted for delivery"

controller = Controller(SinkHandler(), hostname="127.0.0.1", port=2525)
controller.start()
```

Viktigt för tolkningen: Generatorn och mottagaren kördes på samma dator, utan TLS och utan något nätverk däremellan. De uppmätta siffrorna säger därför inget om en gateway, utan är självtestet av generatorn från översiktsartikeln: beviset på att belastningsupplägget över huvud taget kan generera målraten, och den övre gräns som senare mätningar mot det verkliga testsystemet jämförs med.

## Testplanen: fem meddelandeklasser, ett blandningsförhållande

Kärnan i planen är en Thread Group med 20 trådar, 10 sekunders ramp-up och 500 loopar, alltså 10 000 iterationer. Under den finns fem Throughput Controllers i läget "Percent Executions", var och en med exakt en SMTP Sampler:

| Klass (Sampler-etikett) | Andel | Regelväg i gatewayen |
|---|---|---|
| 01 Standard utan trigger | 60 % | AutoGenerated-kontroll, certifikatkontroll, klartextfallback |
| 02 Trigger (sec) | 15 % | TRE-kuvert för mottagare utan certifikat |
| 03 Trigger (sign) | 10 % | Certificate Exchange: signera, skicka med nyckeln |
| 04 Nyckelord VERTRAULICH | 10 % | Specialroutning till branschgatewayen |
| 05 Trigger (unsec) | 5 % | Klartext tvingas fram |

Fördelningen på fem separata Samplers i stället för en Sampler med variabel ämnesrad har ett konkret skäl: HTML-rapporten grupperar alla nyckeltal efter Sampler-etikett. Fem etiketter ger fem rader i statistiken med egna percentiler per klass; en enda Sampler med CSV-matad ämnesrad skulle ge en enda sammanfattande rad, och skillnaden mellan regelvägarna vore osynlig i utvärderingen.

Varje Sampler fyller i de vanliga fälten: målhost och port som användardefinierade variabler (`${zielhost}`, `${zielport}`), så att samma plan utan ändringar kan köras mot mottagare, testmiljö eller PreProd, samt avsändare, mottagare, ämnesrad med en tydlig markör (här ordet LOADTEST i ämnesraden) och en textkropp på cirka 1 till 2 KB. Alternativet "Include timestamp in subject" lägger till leveranstidpunkten i millisekunder; vid en senare körning mot ett verkligt system med flera steg kan end-to-end-latensen per meddelande beräknas utifrån detta tillsammans med mottagarens mottagningstidpunkter.

En fallgrop från denna körning som kan generaliseras: Det första försöket misslyckades med 10 000 fel på 10 sekunder, alla med `java.lang.ClassCastException ... NullProperty cannot be cast to ... CollectionProperty` i stället för ett SMTP-svar. Orsaken var en handbyggd JMX-fil där Samplerns headerlista saknades; Samplern kräver egenskapen, även när den är tom. Lärdomen handlar mindre om den specifika egenskapen än om mönstret: klicka ihop och spara testplaner i GUI:t, skriv inte XML för hand, och gör en minimal testkörning före varje burst och kontrollera hos mottagaren att ämnesrad och innehåll verkligen kommer fram. En felräknare på 100 procent vid 0 ms svarstid betyder nästan alltid att felet inträffar före nätverket och att testet alltså aldrig nådde målsystemet.

## Körningen

Själva mätningen körs i CLI-läge; GUI:t är bara redigeraren. Ett enda anrop skapar körning, rådata och rapport:

```bash
jmeter -n -t gateway-lasttest.jmx -l lauf-10k.jtl -e -o report-10k
```

Summarisern i konsolen visar förloppet live, slutresultatet för körningen:

```text
summary =  10000 in 00:00:13 =  781.8/s Avg: 6 Min: 1 Max: 27 Err: 0 (0.00%)
```

10 000 meddelanden på 12,8 sekunder, 782 meddelanden per sekund i genomsnitt, inga fel. Mottagaren bekräftade oberoende av detta exakt 10 000 accepterade mejl med mixen 6000 / 1500 / 1000 / 1000 / 500; Throughput Controllers blandningsförhållande stämde alltså exakt ner på meddelandenivå.

## HTML-rapporten

Argumentet för JMeter jämfört med slankare generatorer som smtp-source är utvärderingen, och dashboard-rapporten levererar den utan extra arbete:

![JMeter-dashboard för körningen: APDEX 1.000 för alla fem klasser, Requests Summary 100 procent PASS, statistiktabell med percentiler per meddelandeklass](../images/jmeter-report-dashboard.png)

Statistiktabellen är den viktigaste delen av rapporten. Per Sampler-etikett, alltså per meddelandeklass, visas antal, felfrekvens, genomsnitt, median, 90:e, 95:e och 99:e percentilen, maximum och genomströmning. I den konkreta körningen: median 7 ms, p95 på 11 ms, p99 på 12 ms, maximum 27 ms, i praktiken identiskt för alla fem klasser. För en lokal mottagare som behandlar varje meddelande likadant är det precis den förväntade bilden och samtidigt referensvärdet: Om samma plan senare körs mot den verkliga gatewayen och (sec)-klassen plötsligt visar flera gånger standardmedianen, är det krypteringsvägens merarbete, tydligt isolerat per regelgren.

APDEX-blocket ovanför komprimerar samma sak till ett tal per klass (här överallt 1.000, eftersom alla svar låg långt under toleranströskeln på 500 ms); trösklarna kan anpassas till egna tjänstemål i report-properties. Errors-blocket förblir tomt i denna körning, men är den första platsen att titta på vid tester mot verkliga system: Det grupperar fel efter svarstext, så att en `421`-begränsning från målsystemet omedelbart kan skiljas från anslutningsavbrott.

Även här finns en fallgrop, och den berör varje kort burst: Rapportens tidsseriediagram arbetar som standard med en granularitet på en minut. En körning på 13 sekunder kollapsar därmed till en enda datapunkt, och kurvorna under "Charts" ser ut som ett mätfel. Rapporten kan genereras på nytt från den befintliga JTL-filen utan en ny körning, med finare upplösning:

```bash
jmeter -g lauf-10k.jtl -o report-fein -Jjmeter.reportgenerator.overall_granularity=1000
```

Med sekundgranularitet blir den enskilda punkten den faktiska belastningskurvan:

![Hits per Second med 1-sekundsgranularitet: ökning under 10-sekunders-ramp-up till en platå runt 840 meddelanden per sekund, därefter kraftigt fall när testet avslutas](../images/jmeter-report-hits-per-second.png)

Kurvan visar 10-sekunders-ramp-upen, en platå runt 840 meddelanden per sekund och fallet i slutet när de första trådarna har avslutat sina 500 loopar. För tolkningen är platån viktig, inte genomsnittet över hela körningen: Genomsnittet på 782/s inkluderar ramp-up och nedvarvning och underskattar den uppnådda hållbara hastigheten.

## Vad denna körning visar och vad den inte visar

Efter denna körning är följande belagt: Testplanen är funktionellt korrekt (minimal testkörning med innehållskontroll hos mottagaren), blandningsförhållandet stämmer exakt och generatorn klarar minst 840 meddelanden per sekund på denna dator utan TLS. Den som vill testa en gateway dimensionerad för 100 mejl per sekund har därmed en reservfaktor på åtta och kan med gott samvete tillskriva flaskhalsar målsystemet.

Allt annat är inte belagt, och denna avgränsning hör hemma i varje testrapport: inget uttalande om kostnaden för TLS-handshake (den verkliga vägen använder STARTTLS), inget om gatewayens köbeteende, inget om regelvägarnas bearbetningstid. Samma plan pekar däremot med omställda variabler `zielhost`/`zielport` mot gatewayens testmiljö; utvärderingen körs då identiskt, kompletterad med gateway-loggarna och köövervakningen från översiktsartikeln. Just denna återanvändbarhet, en plan för mottagare, testmiljö och PreProd med identisk utvärdering, är det egentliga skälet att lägga ner arbetet på en välgjord JMeter-plan en gång.

## Källor

1.  [Apache JMeter User's Manual: Component Reference, SMTP Sampler](https://jmeter.apache.org/usermanual/component_reference.html): Referens för Sampler-fälten inklusive headers, tidsstämpelalternativ och EML-sändning.

2.  [Apache JMeter: Generating Dashboard Report](https://jmeter.apache.org/usermanual/generating-dashboard.html): Skapande av HTML-rapporten från körningen eller i efterhand från JTL-filen, inklusive granularitets- och APDEX-egenskaperna.

3.  [Apache JMeter: Test Plan, Logic Controllers](https://jmeter.apache.org/usermanual/component_reference.html#Throughput_Controller): Hur Throughput Controller fungerar i läget Percent Executions för meddelandemixen.

4.  [aiosmtpd, dokumentation](https://aiosmtpd.aio-libs.org/): Den asyncio-baserade SMTP-servern som gör det möjligt att skapa mottagaren med några rader Python.

5.  [Eclipse Temurin, Adoptium](https://adoptium.net/temurin/releases/): Portabla JRE-arkiv för att köra JMeter utan Java-installation.
