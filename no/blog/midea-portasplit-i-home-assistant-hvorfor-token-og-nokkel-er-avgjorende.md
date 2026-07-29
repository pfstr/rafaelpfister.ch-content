---
title: "Midea PortaSplit i Home Assistant: Hvorfor token og nøkkel er avgjørende"
navTitle: "PortaSplit og token"
description: "Lokal styring krever to verdier fra Midea-skyen. Slik hentes token og nøkkel, hvorfor det er problematisk å miste dem, og hvordan eiere kan sikre det eksisterende oppsettet."
date: "2026-07-24"
kategorie: "Home Assistant og IoT"
timeToRead: "9 min lesetid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-einrichten
  - serverloser-newsletter-cloudflare-workers-d1
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende"
translationOf: "midea-portasplit-home-assistant"
url: "https://rafaelpfister.ch/no/blog/midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: a02265cf4b8fde907361c3551326fd3283c83d660cf9fdfb40451a9e78ca690b
translatedAt: 2026-07-29T12:29:38.969Z
---

<aside class="article-update">
  <p class="article-update__label">Hva PortaSplit-eiere bør gjøre nå</p>
  <p>Via private skytjenestegrensesnitt henter Home Assistant den enhetsspesifikke tokenen og nøkkelen under oppsettet. Prosjektet Midea AC LAN har advart om mulige endringer siden 19. mai 2025. Det er imidlertid ikke dokumentert noen konkret dato for avvikling fra produsenten. For eiere betyr det:</p>
  <ol>
    <li><strong>Ikke fjern et eksisterende oppsett uten grunn.</strong> Bare innhentingen av tilgangsdataene krever Midea-skyen. Fremtidige endringer i det private endepunktet kan gjøre et nytt oppsett vanskeligere.</li>
    <li><strong>Sikkerhetskopier token, nøkkel og konfigurasjon kryptert.</strong> Hvis uthentingen senere ikke lenger fungerer, er sikkerhetskopien den mest pålitelige veien til gjenoppretting.</li>
    <li><strong>Ikke opphev koblingen uten behov.</strong> Fabrikkinnstillinger, fjerning fra Midea-kontoen eller bytte av WLAN-modul krever at token hentes på nytt, noe som kan mislykkes i fremtiden.</li>
  </ol>
  <p>Enheter som allerede er satt opp, styres lokalt. Endringer i skygrensesnittet påvirker derfor først og fremst tilføyelse og gjenoppretting, ikke hver løpende styringskommando. De konkrete trinnene finner du i <a href="/blog/midea-portasplit-home-assistant-einrichten">den praktiske artikkelen om integrasjon og sikring</a>.</p>
</aside>

![Eksempel på et Home Assistant-dashbord for en Midea PortaSplit med rom- og innstilt temperatur, luftfuktighet, effektforbruk, energiforbruk og kompressorens driftstid de siste 24 timene.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Den lokale styringen av Midea PortaSplit bygger på to enhetsspesifikke verdier: token og nøkkel. Home Assistant-integrasjonen henter begge via et privat Midea-skysluttpunkt under oppsettet. Deretter sender den styringskommandoer direkte i det lokale nettverket.

Prosjektet Midea AC LAN advarer om mulige endringer i disse skygrensesnittene. Nyere analyser viser imidlertid at det ikke kan utledes noen bekreftet produsentplan eller konkret dato for avvikling av dette. Denne artikkelen forklarer det tekniske avhengighetsforholdet; den [detaljerte API-analysen](/blog/midea-v2-cloud-api-portasplit-home-assistant) setter de ulike «V2»-betegnelsene og dagens status i sammenheng.

## Token-spørsmålet i detalj

### Hvorfor har Home Assistant i det hele tatt kunnet hente tokenen?

Det interessante er: Fellesskapet har aldri beregnet tokenen. De har i stedet analysert nettverkstrafikken til den offisielle appen og oppdaget at appen ikke genererer tokenen selv, men henter den fra skyen:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

Home Assistant-integrasjonen har reimplementert nettopp dette skyoppropet. Den logger inn i skyen med de samme endepunktene og den samme prosessen som appen, og mottar dermed den samme tokenen og nøkkelen. Det egentlige grunnlaget er altså ikke en smart beregning, men en gjenskapt uthenting. Forsvinner endepunktet, forsvinner også muligheten til å hente dem.

### Kan man lese ut tokenen fra den offisielle appen?

Teoretisk sett ja. Appen må på et tidspunkt kjenne tokenen, ellers kunne den ikke kommunisere lokalt med enheten. I prinsippet mulige veier er:

- reverse engineering av appen,
- avlytting av nettverkstrafikken, dersom den ikke er ekstra beskyttet,
- instrumentering av appen under kjøring, for eksempel med Frida eller Objection,
- hooking av funksjonene som behandler tokenen.

Det er nettopp dette utvikleren av Midea AC LAN sikter til når han sier at det nåværende designet er et sikkerhetsproblem sett fra Mideas ståsted: En langvarig hemmelighet som kan hentes ut av en bredt distribuert app med rimelig innsats, er vanskelig å kontrollere. For den enkelte brukeren er disse metodene imidlertid krevende og erstatter ikke den praktiske skyuthentingen.

### Kan man få tokenen direkte fra enheten?

Det ville vært den mest elegante løsningen. Hvis enheten utvekslet en offentlig nøkkel ved første lokale paring eller brukte en engangs paringskode via Bluetooth, ville ingen sky vært nødvendig. Mange moderne IoT-enheter gjør nettopp dette.

Midea har imidlertid utformet den opprinnelige LAN-protokollen annerledes: Enheten godtar lokale kommandoer først med riktige, skyrelaterte tilgangsdata. Det finnes ingen dokumentert lokal paringsmekanisme som kan gi ut tokenen uten å gå via skyen. Skyen er dermed ikke bare en bekvemmelighet, men arkitektonisk den eneste tiltenkte veien til tokenen.

### Kan fellesskapet omgå endringer i token-endepunktet?

Det ville bare være mulig dersom ett av følgende alternativer dukker opp:

- et nytt sky-API som fortsatt leverer tokens,
- en hittil ukjent lokal paringsmetode,
- en sårbarhet i enheten,
- eller at Midea en gang selv publiserer et offisielt lokalt API.

Å bare «regne ut» tokenen på nytt vil derimot svært sannsynlig ikke fungere. Hvis det var mulig, ville fellesskapet trolig ha implementert det for lenge siden og aldri vært avhengig av sky-API-et. At omveien via skyen i det hele tatt ble laget, er det sterkeste tegnet på at det ikke finnes noen enklere lokal vei.

## Advarselen fra Midea AC LAN

Repositoryet til `Midea AC LAN` inneholder en tydelig plassert «Important Notice». Ifølge utvikleren har Midea allerede stengt de serversidige token-API-ene i Meiju- og SmartHome-skyen. Integrasjonen bruker derfor for tiden token-grensesnitt fra NetHome Plus-skyen, og også disse skal stenges trinnvis. Konsekvensen ville være at enheter som allerede er satt opp, fortsatt fungerer lokalt, men at nye enheter ikke lenger kan legges til. Utvikleren går enda lenger og skriver at Midea på lang sikt vil gå over til et nytt Cloud Control API og dermed gjøre det hittil brukte V1-LAN-API-et ubrukelig.

Advarselen har en kort historie. Den tydelige «Important Notice» kom inn i README-filen 19. mai 2025 (Pull Request #578) og nevnte da SmartHome-skyen som reserve for å legge til nye enheter. Den 14. juli 2025 (#639) ble den oppdatert; siden da har den vist til NetHome Plus-skyen fordi Midea hadde stengt flere endepunkter. Kjernen var uendret i begge versjonene: Token-grensesnittene forsvinner gradvis, og bare den skyen som fortsatt kan brukes, endres.

Dette må vurderes nyansert. Det dreier seg om vurderingen fra et åpen kildekode-prosjekt, ikke en bindende plan fra Midea, og tidsplanen er ukjent. En fremtidig fastvareoppdatering kan endre lokale funksjoner, og en allerede lagret token kan fortsette å fungere, men trenger ikke å gjøre det for alltid. Fabrikkinnstillinger, bytte av WLAN-modul eller en ny enhet kan gjøre det nødvendig å hente tokenen på nytt.

Dette leder til de tre trinnene fra boksen i starten av artikkelen, hver med sin begrunnelse:

- **Ikke erstatt et fungerende oppsett uten grunn.** Innhenting av tokenen er det eneste trinnet som nødvendigvis går via Midea-skyen. Endringer i det private endepunktet kan særlig ramme et senere nytt oppsett.
- **Sikre tilgangsdataene.** Home Assistant lagrer token og nøkkel lokalt. Et defekt system, en mislykket gjenoppretting eller en integrasjon som ved et uhell blir slettet, kan likevel gjøre den lokale styringen ubrukelig dersom det ikke finnes en ekstern sikkerhetskopi.
- **Ikke opphev koblingen lettvint.** Det er ikke fullstendig dokumentert om en fabrikktilbakestilling eller fjerning fra Midea-kontoen krever nye tilgangsdata for hver modell. En sikkerhetskopi før slike endringer er derfor avgjørende.

Den løpende driften påvirkes foreløpig ikke av dette: Den lokale styringen bruker de allerede lagrede verdiene og trenger ikke lenger token-endepunktet. Det gjenstår en restrisiko dersom en senere fastvare endrer den lokale protokollen eller autentiseringen. Hvordan token, nøkkel og konfigurasjon sikkerhetskopieres, står i [den praktiske artikkelen om oppsett](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Hva dette betyr for sikkerheten

Advarselen har, i tillegg til tilgjengelighet, en sikkerhetsmessig kjerne. Ifølge `Midea AC LAN` bygger den eldre LAN-arkitekturen på en problematisk antakelse: Klientkommunikasjonen ble opprinnelig ansett som tilstrekkelig beskyttet, og derfor fikk tokenene som ble utstedt fra skyen ingen utløpstid.

En token uten utløpstid er ikke i seg selv en sårbarhet. Den blir problematisk dersom den havner i logger eller ubeskyttede sikkerhetskopier, kommer på avveie til tredjeparter eller verken kan tilbakekalles eller roteres. Utvikleren av `Midea AC LAN` antar at Midea reagerer på disse risikoene med endringer i token-tjenestene og en mer skybasert arkitektur. En tilsvarende produsentkunngjøring med tidsplan er imidlertid ikke dokumentert.

Språklig presisjon er viktig her. Fellesskapsintegrasjonen «hakker» ikke klimaanlegget. Den implementerer en proprietær protokoll som er kartlagt gjennom reverse engineering. Sikkerhetsproblemet oppstår fordi langvarige hemmeligheter kan brukes og lagres utenfor den opprinnelig tiltenkte appen.

For drift i eget nettverk er det først og fremst relevant hva token og nøkkel muliggjør. Begge autentiserer den lokale kommunikasjonen med enheten. Hvis de havner i feil hender, kan en angriper, avhengig av protokollen og sin nettverksposisjon, oppdage enheten, autentisere seg overfor den, lese statusinformasjon, endre innstillinger, slå klimaanlegget av eller på, bytte driftsmodus og endre den innstilte temperaturen. Angriperen må som regel likevel kunne opprette en nettverksforbindelse til enheten; besittelse av token og nøkkel alene muliggjør ikke et angrep fra hele internett. Token og nøkkel må derfor behandles som et passord. Hvordan enheten kan plasseres i nettverket slik at disse verdiene gjør liten skade selv ved en hendelse, er temaet for [andre del](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Hva som gjenstår i praksis

Den lokale styringen av PortaSplit avhenger helt av token og nøkkel, som for tiden bare kan hentes via Midea-skyen. Denne omveien er en del av protokolldesignet: Lokale kommandoer er bundet til skyrelaterte tilgangsdata. Fordi endepunktet er privat og udokumentert, er den langsiktige tilgjengeligheten til den uoffisielle integrasjonen usikker.

I praksis betyr det: Sikre tilgangsdata og konfigurasjon, ikke opphev en fungerende kobling uten behov, og følg med på endringer i integrasjon og fastvare. Enheter som allerede er satt opp, fortsetter å kjøre lokalt. Oppsett, sikkerhetskopiering og nettverksbeskyttelse beskrives i [den praktiske artikkelen om PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Kilder

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjon `Midea AC LAN` med «Important Notice» (siden 19. mai 2025, oppdatert 14. juli 2025), begrunnelsen om tokens uten utløpstid og rekonstruert klientkryptering, samt beskrivelsen av den skybaserte token-uthentingen.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjon `Midea Smart AC`: Beskrivelse av den skybaserte uthentingen av token og nøkkel for V3-enheter og lokal lagring av verdiene.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om SmartHome-økosystemet og de refererte sikkerhets- og personvernstandardene.
