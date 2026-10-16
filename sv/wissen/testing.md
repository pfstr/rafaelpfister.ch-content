---
title: "Tester och belastningstester: kontrollera infrastrukturen under kontrollerade förhållanden"
blatt: "testing"
description: "Metodik för funktions- och belastningstester i meddelande- och infrastruktursmiljöer: mätvärden, testförlopp med baslinje, burst och soak, lastgeneratorer och sänkor, märkning av testkörningar och utvärdering med percentiler i stället för medelvärden."
fakten:
  - label: Grundförlopp
    wert: Funktionstest → Baslinje → Burst → Soak
    href: https://jmeter.apache.org/usermanual/best-practices.html
  - label: Fyra mätvärden
    wert: Acceptanshastighet · Sessionslatens · End-to-end-latens · Köbeteende
  - label: Utvärdering
    wert: Percentiler p50 · p95 · p99, inte medelvärden
    href: https://sre.google/sre-book/monitoring-distributed-systems/
  - label: Lastgeneratorer
    wert: smtp-source · Postal · JMeter · egna skript
    href: https://www.postfix.org/smtp-source.1.html
  - label: Sänkor
    wert: smtp-sink · bhm · aiosmtpd · discard-transport
    href: https://www.postfix.org/smtp-sink.1.html
  - label: Kontrollförsök
    wert: oberoende räkning på mottagarsidan
  - label: Märkning
    wert: unik header- eller ämnesmarkör per testkörning
  - label: Generatorns självtest
    wert: Körning mot sänkan bekräftar den uppnåeliga målhastigheten
  - label: Grundregel
    wert: endast egna eller uttryckligen godkända system
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f6d1ef180221f41242e1d32b5fb8fb2e631894e3ccd32a3b3322a33b808da210
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:40:23.800Z
translationReview: automatic
---

# Tester och belastningstester: kontrollera infrastrukturen under kontrollerade förhållanden

Om en miljö klarar sin topplast, om en regeluppsättning beter sig likadant efter en ombyggnad eller om en lagringsmontering håller under verkliga åtkomstmönster syns inte i databladet utan i testet. Den här sidan sammanfattar den metodik som är gemensam för alla testtyper: vad som mäts, hur ett testförlopp byggs upp, vilka verktyg som skapar och tar emot last samt hur rådata blir till tillförlitliga slutsatser.

Den viktigaste regeln kommer före all metodik: belastnings- och störningstester hör uteslutande hemma på egna system eller i miljöer som uttryckligen har godkänts för detta. En burst mot främmande infrastruktur är en attack; ett e-posttest med påhittade avsändare mot produktiva mål skapar backscatter och poster på blockeringslistor.

## Separera funktionstest och belastningstest

Ett test besvarar alltid exakt en fråga, och de två grundfrågorna kräver olika upplägg. **Funktionstestet** kontrollerar om en väg fungerar korrekt: en enskild SMTP-transaktion med full kontroll över varje steg, en enskild filåtkomst, en enskild regelväg genom en gateway. **Belastningstestet** kontrollerar hur systemet beter sig under volym: genomströmning, latensfördelning, köer, resursförbrukning.

Ordningen är obligatorisk: först funktionstestet, sedan lasten. En burst över en felaktig väg mäter ingenting; den producerar bara tiotusen likadana fel. En felräknare på 100 procent vid svarstider nära noll innebär nästan alltid att felet uppstår före nätverket och att testet aldrig når målsystemet.

## De fyra mätvärdena

Belastningstester i miljöer med flera steg mäter fyra olika saker, och varje värde kräver sitt eget perspektiv:

1. **Acceptanshastighet:** Hur många transaktioner per sekund tar det första hoppet emot? Det är värdet som lastgeneratorer levererar direkt.
2. **Sessionslatens:** Hur lång tid tar en enskild transaktion? Under last ökar detta värde ofta långt innan acceptanshastigheten sjunker.
3. **End-to-end-latens:** Hur lång tid behöver en transaktion genom alla mellanliggande steg fram till målet? Det är det värde användarna uppfattar.
4. **Köbeteende:** Hur mycket växer kön under toppen och hur snabbt töms den efteråt? Ett system som accepterar allt och arbetar av det i timmar klarar acceptanstestet men misslyckas ändå.

## Testförlopp: baslinje, burst, soak

Ett tillförlitligt test genomförs i tre steg, vart och ett med sin egen frågeställning. **Baslinjen** fastställer referensvärden för latens och resursförbrukning med måttlig, känd last och upptäcker konfigurationsfel innan de försvinner i toppmätningen. **Bursten** är den egentliga mätningen av topplast; flera körningar med ökande parallellitet visar var acceptanshastigheten planar ut och latensen ökar kraftigt. **Soak-testet** kör den förväntade kontinuerliga lasten under timmar; först här visar sig minnesläckor, fulla spool-partitioner och anslutningsgränser som en kort burst aldrig når.

Därtill hör tre hantverksregler. För det första **generatorns självtest**: Före den första körningen mot målsystemet körs generatorn direkt mot sänkan; det visar att uppställningen över huvud taget kan skapa målhastigheten och ger den övre gräns som senare mätningar jämförs med. För det andra **märkningen**: Varje testtransaktion får en unik markör (egen header, ämneskonvention, körningsnummer), så att körningar kan skiljas åt i loggarna och testdata därefter kan rensas bort fullständigt. För det tredje **kontrollförsöket**: En oberoende räkning på mottagarsidan bekräftar att det som generatorn skickade har kommit fram; om siffrorna inte stämmer överens har något gått förlorat på vägen.

## Verktyg: generator och sänka

Ett belastningstest består alltid av tre delar: lastgenerator, testsystem och kontrollerad sänka. För SMTP-last är vanliga generatorer [smtp-source](https://www.postfix.org/smtp-source.1.html) från Postfix-paketet (maximal rålast med minimal insats), [Postal](https://doc.coker.com.au/projects/postal/) (kontinuerlig last med målhastighet och varierande meddelanden), [Apache JMeter](https://jmeter.apache.org/) (lastprofiler, percentiler och HTML-rapport, förstahandsvalet under Windows) samt egna skript när testet behöver realistiska transaktionsmixar. [swaks](https://www.jetmore.org/john/code/swaks/) är ingen lastgenerator, men hör hemma som funktionstestare före varje burst.

På mottagarsidan tar en sänka emot allt och kastar bort det: [smtp-sink](https://www.postfix.org/smtp-sink.1.html), `bhm` som hör till Postal eller en liten [aiosmtpd](https://aiosmtpd.aio-libs.org/)-server i några rader Python, som samtidigt sköter kontrollräkningen. Utvecklarverktyg med webbgränssnitt är olämpliga som sänka för belastningstest; vid tiotusentals transaktioner mäts annars analysverktyget i stället för testsystemet.

## Utvärdering: percentiler, pivot, förlopp

Rådata blir till slutsatser först genom tre utvärderingssteg. **Percentiler i stället för medelvärden:** Ett genomsnitt på två sekunder kan betyda att allt tar två sekunder, eller att 95 procent slutförs på en halv sekund och att resten behövde betydligt längre tid; p50, p95 och p99 skiljer dessa fall åt. **Pivota fel:** Fördelningen av felkoder över tid visar när ett system börjar begränsa och vilken skyddsmekanism som träder i kraft först. **Förlopp i stället för slutläge:** Genomströmning och ködjup plottade över tid visar ramp-up, platå och avklingning; för tolkningen räknas platån, inte genomsnittet över hela körningen, eftersom det omfattar upp- och nedvarvning och underskattar den uppnådda kontinuerliga hastigheten.

Till utvärderingen hör slutligen den ärliga avgränsningen: Varje testrapport fastslår vad körningen belägger och vad den inte belägger. En mätning utan TLS säger inget om kostnaden för TLS-handshake; en lokal sänka inget om köbeteendet hos det verkliga målet; en burst inget om minnesläckor i kontinuerlig drift. Dessa gränser anger nästa testkörning.

Artiklarna nedan tillämpar metodiken på konkreta fall: verktygsjämförelse och testplanering för e-postbelastningstester, en fullständigt genomförd JMeter-körning med HTML-rapport, skiktvis SMTP-diagnostik med inbyggda verktyg, en testplan för molnlagringsmonteringar och den vetenskapliga metoden bakom kontrollerade experiment i e-postflödet.
