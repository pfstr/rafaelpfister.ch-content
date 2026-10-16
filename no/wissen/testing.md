---
title: "Testing og belastningstester: Test infrastrukturen under kontrollerte forhold"
blatt: "testing"
description: "Metodikk for funksjons- og belastningstester i meldings- og infrastrukturløsninger: måleparametere, testforløp med baseline, burst og soak, lastgeneratorer og sluk, merking av testkjøringer og evaluering med persentiler i stedet for gjennomsnitt."
fakten:
  - label: Grunnforløp
    wert: Funksjonstest → Baseline → Burst → Soak
    href: https://jmeter.apache.org/usermanual/best-practices.html
  - label: Fire måleparametere
    wert: Mottaksrate · Sesjonslatens · Ende-til-ende-latens · Køatferd
  - label: Evaluering
    wert: Persentilene p50 · p95 · p99, ikke gjennomsnitt
    href: https://sre.google/sre-book/monitoring-distributed-systems/
  - label: Lastgeneratorer
    wert: smtp-source · Postal · JMeter · egne skript
    href: https://www.postfix.org/smtp-source.1.html
  - label: Sluk
    wert: smtp-sink · bhm · aiosmtpd · discard-transport
    href: https://www.postfix.org/smtp-sink.1.html
  - label: Kontrollforsøk
    wert: uavhengig telling på mottakssiden
  - label: Merking
    wert: entydig header- eller emnemarkør per testkjøring
  - label: Generator-selvtest
    wert: Kjøring mot sluket dokumenterer oppnåelig målraten
  - label: Grunnregel
    wert: kun egne eller uttrykkelig godkjente systemer
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f6d1ef180221f41242e1d32b5fb8fb2e631894e3ccd32a3b3322a33b808da210
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:40:43.427Z
translationReview: automatic
---

# Testing og belastningstester: Test infrastrukturen under kontrollerte forhold

Om et miljø tåler topplasten, et regelsett oppfører seg likt etter en ombygging, eller en storage-mount holder under reelle tilgangsmønstre, viser seg ikke i databladet, men i testen. Denne siden samler metodikken som er felles for alle testtyper: hva som måles, hvordan et testforløp er bygget opp, hvilke verktøy som skaper og tar imot last, og hvordan rådata blir til pålitelige konklusjoner.

Den viktigste regelen kommer før enhver metodikk: Last- og forstyrrelsestester hører utelukkende hjemme på egne systemer eller i miljøer som uttrykkelig er godkjent for dette. En burst mot andres infrastruktur er et angrep; en e-posttest med oppdiktede avsendere mot produksjonsmål skaper backscatter og oppføringer på blokkeringslister.

## Skill mellom funksjonstest og belastningstest

En test besvarer alltid nøyaktig ett spørsmål, og de to grunnspørsmålene krever ulike oppsett. **Funksjonstesten** kontrollerer om en sti fungerer korrekt: én enkelt SMTP-transaksjon med full kontroll over hvert trinn, én enkelt filtilgang, én enkelt regelsti gjennom en gateway. **Belastningstesten** kontrollerer hvordan systemet oppfører seg under volum: gjennomstrømning, latensfordeling, køer, ressursbruk.

Rekkefølgen er obligatorisk: først funksjonstesten, deretter lasten. En burst over en feilaktig sti måler ingenting; den produserer bare titusen like feil. En feilteller på 100 prosent med responstider nær null betyr nesten alltid at feilen oppstår før nettverket, og at testen aldri har nådd målsystemet.

## De fire måleparameterne

Belastningstester i flertrinnsmiljøer måler fire forskjellige ting, og hver parameter krever sitt eget perspektiv:

1. **Mottaksrate:** Hvor mange operasjoner per sekund tar første hop imot? Dette er verdien lastgeneratorer leverer direkte.
2. **Sesjonslatens:** Hvor lenge varer én enkelt transaksjon? Under last øker denne verdien ofte lenge før mottaksraten faller.
3. **Ende-til-ende-latens:** Hvor lang tid bruker en operasjon gjennom alle mellomledd til målet? Dette er parameteren brukerne opplever.
4. **Køatferd:** Hvor mye vokser køen under toppen, og hvor raskt tømmes den etterpå? Et system som tar imot alt og behandler det i timevis, består mottakstesten og stryker likevel.

## Testforløp: Baseline, Burst, Soak

En pålitelig test kjøres i tre trinn, hver med sitt eget spørsmål. **Baseline** fastsetter referanseverdier for latens og ressursbruk med moderat, kjent last og avdekker konfigurasjonsfeil før de forsvinner i toppmålingen. **Burst** er selve målingen av topplast; flere kjøringer med økende parallellitet viser hvor mottaksraten flater ut og latensen øker kraftig. **Soak** kjører den forventede vedvarende lasten i timevis; først her viser minnelekkasjer, fulle spool-partisjoner og tilkoblingsgrenser seg, som en kort burst aldri når.

Tre praktiske regler hører med. For det første **generator-selvtesten**: Før første kjøring mot målsystemet kjører generatoren direkte mot sluket; dette dokumenterer at oppsettet i det hele tatt kan generere målraten, og gir øvre grense som senere målinger sammenlignes med. For det andre **merkingen**: Hver testoperasjon har en entydig markør (egen header, emnekonvensjon, kjøringsnummer), slik at kjøringer kan skilles rent i loggene og testdata kan ryddes helt opp etterpå. For det tredje **kontrollforsøket**: En uavhengig telling på mottakssiden bekrefter at det generatoren sendte, har kommet fram; hvis tallene ikke stemmer overens, har noe gått tapt underveis.

## Verktøy: Generator og sluk

En belastningstest består alltid av tre deler: lastgenerator, testsystem og kontrollert sluk. For SMTP-last er de vanlige generatorene [smtp-source](https://www.postfix.org/smtp-source.1.html) fra Postfix-pakken (maksimal rålast med minimal innsats), [Postal](https://doc.coker.com.au/projects/postal/) (vedvarende last med målrate og varierende meldinger), [Apache JMeter](https://jmeter.apache.org/) (lastprofiler, persentiler og HTML-rapport, førstevalget under Windows) og egne skript når testen trenger realistiske blandinger av operasjoner. [swaks](https://www.jetmore.org/john/code/swaks/) er ingen lastgenerator, men hører hjemme som funksjonstester før hver burst.

På mottakssiden tar et sluk imot alt og forkaster det: [smtp-sink](https://www.postfix.org/smtp-sink.1.html), `bhm` som hører til Postal, eller en liten [aiosmtpd](https://aiosmtpd.aio-libs.org/)-server i noen få linjer Python, som samtidig utfører kontrolltellingen. Utviklerverktøy med webgrensesnitt er uegnet som sluk for belastningstester; ved titusenvis av operasjoner blir ellers analyseverktøyet målt i stedet for testsystemet.

## Evaluering: Persentiler, pivot, forløp

Rådata blir først til konklusjoner gjennom tre evalueringssteg. **Persentiler i stedet for gjennomsnitt:** Et gjennomsnitt på to sekunder kan bety at alt tar to sekunder, eller at 95 prosent er fullført på et halvt sekund og resten trengte vesentlig lengre tid; p50, p95 og p99 skiller disse tilfellene. **Pivoter feil:** Fordelingen av feilkoder over tid viser når et system begynner å begrense, og hvilken beskyttelsesmekanisme som griper inn først. **Forløp i stedet for sluttstatus:** Gjennomstrømning og kødybde plottet over tid viser ramp-up, platå og utløp; for tolkningen teller platået, ikke gjennomsnittet over hele kjøringen, for det inkluderer opp- og nedkjøring og undervurderer den oppnådde vedvarende raten.

Til slutt hører en ærlig avgrensning med i evalueringen: Hver testrapport fastslår hva kjøringen dokumenterer og hva den ikke dokumenterer. En måling uten TLS sier ingenting om kostnadene ved TLS-handshake; et lokalt sluk sier ingenting om køatferden til det faktiske målet; en burst sier ingenting om minnelekkasjer ved kontinuerlig drift. Disse grensene angir de neste testkjøringene.

Artiklene nedenfor gjennomfører metodikken i konkrete tilfeller: verktøysammenligning og testplanlegging for e-post-belastningstester, en fullt gjennomført JMeter-kjøring med HTML-rapport, lagvis SMTP-diagnostikk med innebygde verktøy, en testplan for cloud-storage-mounts og den vitenskapelige metoden bak kontrollerte eksperimenter i e-postflyten.
