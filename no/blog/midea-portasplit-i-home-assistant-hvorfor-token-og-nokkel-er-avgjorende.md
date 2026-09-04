---
title: "Midea PortaSplit i Home Assistant: Hvorfor token og nøkkel er avgjørende"
navTitle: "PortaSplit og token"
description: "Lokal styring krever to verdier fra Midea-skyen. Slik skaffer du token og nøkkel, hvorfor det er problematisk å miste dem, og hvordan eiere kan sikre det eksisterende oppsettet sitt."
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
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: 93933b82cdbb4151fe6dc6ac73a356fc752f120f41461268af1c8e484b62652c
translatedAt: 2026-09-04T08:30:24.142Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/no/blog/midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende
---

<aside class="article-update">
  <p class="article-update__label">Hva PortaSplit-eiere bør gjøre nå</p>
  <p>Via private skygrensesnitt henter Home Assistant den enhetsspesifikke tokenen og nøkkelen under oppsettet. Prosjektet Midea AC LAN har advart mot mulige endringer siden 19. mai 2025. Det finnes imidlertid ingen dokumentert konkret dato for produsentens utfasing. For eiere betyr dette:</p>
  <ol>
    <li><strong>Ikke fjern et eksisterende oppsett uten grunn.</strong> Bare anskaffelsen av tilgangsdata krever Midea-skyen. Fremtidige endringer i det private endepunktet kan gjøre et nytt oppsett vanskeligere.</li>
    <li><strong>Sikkerhetskopier token, nøkkel og konfigurasjon kryptert.</strong> Dersom hentingen senere ikke lenger fungerer, er sikkerhetskopien den mest pålitelige måten å gjenopprette på.</li>
    <li><strong>Ikke opphev sammenkoblingen uten nød.</strong> Fabrikkinnstillinger, fjerning fra Midea-kontoen eller bytte av WLAN-modul tvinger frem ny anskaffelse av token, noe som kan mislykkes i fremtiden.</li>
  </ol>
  <p>Enheter som allerede er satt opp, styres lokalt. Endringer i skygrensesnittet påvirker derfor først og fremst tilføyelse og gjenoppretting, ikke hver enkelt løpende styrekommando. De konkrete trinnene finnes i <a href="/blog/midea-portasplit-home-assistant-einrichten">den praktiske artikkelen om integrasjon og sikring</a>.</p>
</aside>

![Eksempel på et Home Assistant-dashboard for en Midea PortaSplit med rom- og settpunktstemperatur, luftfuktighet, effektforbruk, energiforbruk og kompressorens driftstider de siste 24 timene.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Den lokale styringen av Midea PortaSplit bygger på to enhetsspesifikke verdier: token og nøkkel. Home Assistant-integrasjonen henter begge under oppsettet via et privat Midea-skyendepunkt. Deretter sender den styrekommandoer direkte i det lokale nettverket.

Prosjektet Midea AC LAN advarer mot mulige endringer i disse skygrensesnittene. Nyere analyser viser imidlertid at det ikke kan utledes noen bekreftet produsentplan eller konkret dato for utfasing fra dette. Denne artikkelen forklarer det tekniske avhengighetsforholdet; den [detaljerte API-analysen](/blog/midea-v2-cloud-api-portasplit-home-assistant) setter de ulike «V2»-betegnelsene og dagens status i sammenheng.

## Token-spørsmålet i detalj

### Hvorfor har Home Assistant i det hele tatt kunnet hente tokenen så langt?

Fellesskapet har aldri beregnet tokenen. I stedet analyserte det nettverkstrafikken til den offisielle appen og fant ut at appen ikke genererer tokenen selv, men henter den fra skyen:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

Home Assistant-integrasjonen har reimplementert nettopp dette skyopprinnkallet. Den logger seg på skyen med de samme endepunktene og den samme prosessen som appen, og får dermed samme token og nøkkel. Det egentlige grunnlaget er altså en rekonstruert henting, ikke en beregning. Forsvinner endepunktet, forsvinner også muligheten til å hente dem.

### Kunne man lese ut tokenen fra den offisielle appen?

Teoretisk sett ja. Appen må kjenne tokenen på et tidspunkt, ellers kunne den ikke kommunisere lokalt med enheten. I prinsippet kunne følgende veier tenkes:

- Reverse engineering av appen,
- avlytting av nettverkstrafikken, dersom den ikke er ekstra beskyttet,
- instrumentering av appen under kjøring, for eksempel med Frida eller Objection,
- hooking av funksjonene som behandler tokenen.

Det er nettopp dette uttalelsen fra utvikleren av Midea AC LAN sikter til når den beskriver det tidligere designet som et sikkerhetsproblem fra Mideas ståsted: En langvarig hemmelighet som kan hentes ut fra en bredt distribuert app med rimelig innsats, er vanskelig å kontrollere. For den enkelte brukeren er disse metodene imidlertid krevende og erstatter ikke den praktiske skyhentingen.

### Kunne man få tokenen direkte fra enheten?

Det ville vært den mest elegante løsningen. Hvis enheten ved første lokale sammenkobling utvekslet en offentlig nøkkel eller brukte en engangskode for sammenkobling via Bluetooth, ville ingen sky vært nødvendig. Mange moderne IoT-enheter gjør nettopp dette.

Midea har imidlertid utformet den opprinnelige LAN-protokollen annerledes: Enheten aksepterer først lokale kommandoer med riktige, skyrelaterte tilgangsdata. Det finnes ingen dokumentert lokal sammenkoblingsmekanisme som ville gitt ut tokenen uten omveien via skyen. Skyen er dermed ikke bare en bekvemmelighet, men arkitektonisk den eneste tiltenkte veien til tokenen.

### Kan fellesskapet omgå endringer i token-endepunktet?

Det ville bare være mulig dersom ett av følgende alternativer finnes:

- et nytt sky-API som fortsatt leverer token,
- en hittil ukjent lokal sammenkoblingsmetode,
- en sårbarhet i enheten,
- eller at Midea en gang selv publiserer et offisielt lokalt API.

Det er derimot svært lite sannsynlig at man bare kan «regne ut» tokenen på nytt. Hvis det var mulig, ville fellesskapet trolig ha implementert det for lenge siden og aldri vært avhengig av sky-API-et. At omveien via skyen i det hele tatt ble laget, er det sterkeste tegnet på at det ikke finnes en enklere lokal løsning.

## Advarselen fra Midea AC LAN

Repositoryet til `Midea AC LAN` inneholder en fremtredende plassert «Important Notice». Ifølge utvikleren har Midea allerede stengt de serverbaserte token-API-ene i Meiju- og SmartHome-skyen. Integrasjonen bruker derfor for tiden token-grensesnittene til NetHome Plus-skyen, og også disse skal stenges gradvis. Følgen ville være at enheter som allerede er satt opp fortsatt fungerer lokalt, men at nye enheter ikke lenger kan legges til. Utvikleren går enda lenger og skriver at Midea på sikt vil gå over til et nytt Cloud-Control-API og dermed gjøre det tidligere V1-LAN-API-et ubrukelig.

Advarselen har en kort historie. Den fremtredende «Important Notice» kom inn i README 19. mai 2025 (Pull Request #578) og nevnte da SmartHome-skyen som reserveløsning for å legge til nye enheter. Den ble oppdatert 14. juli 2025 (#639); siden da viser den til NetHome Plus-skyen, fordi Midea hadde stengt flere endepunkter. Kjernen forble uendret i begge versjonene: Token-grensesnittene forsvinner litt etter litt, bare skyen som fortsatt kan brukes, endres.

Dette må vurderes nyansert. Det er vurderingen til et åpen kildekode-prosjekt, ikke en bindende plan fra Midea, og tidsplanen er ukjent. En fremtidig fastvareoppdatering kan endre lokale funksjoner, og en allerede lagret token kan fortsatt fungere, men ikke nødvendigvis for alltid. Tilbakestilling til fabrikkinnstillinger, bytte av WLAN-modul eller en ny enhet kan gjøre det nødvendig å hente tokenen på nytt.

Dette gir de tre trinnene fra boksen i begynnelsen av artikkelen, med begrunnelsen for hvert av dem:

- **Ikke erstatt et fungerende oppsett uten grunn.** Henting av token er det eneste trinnet som nødvendigvis går via Midea-skyen. Endringer i det private endepunktet kan særlig ramme et senere nytt oppsett.
- **Sikre tilgangsdataene.** Home Assistant lagrer token og nøkkel lokalt. Et defekt system, en mislykket gjenoppretting eller en integrasjon som slettes ved et uhell, kan likevel gjøre lokal styring ubrukelig dersom det ikke finnes en ekstern sikkerhetskopi.
- **Ikke opphev sammenkoblingen lettvint.** Det er ikke fullstendig dokumentert om tilbakestilling til fabrikkinnstillinger eller fjerning fra Midea-kontoen tvinger frem nye tilgangsdata for hver modell. En sikkerhetskopi før slike endringer er derfor nødvendig.

Den løpende driften påvirkes foreløpig ikke av dette: Den lokale styringen bruker de allerede lagrede verdiene og trenger ikke lenger token-endepunktet. Det gjenstår en restrisiko dersom en senere fastvare endrer den lokale protokollen eller autentiseringen. Hvordan token, nøkkel og konfigurasjon sikres, står i [den praktiske artikkelen om oppsett](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Hva dette betyr for sikkerheten

Advarselen har, i tillegg til tilgjengelighet, en kjerne knyttet til sikkerhet. Ifølge `Midea AC LAN` bygger den eldre LAN-arkitekturen på en problematisk antakelse: Klientkommunikasjonen ble opprinnelig ansett som tilstrekkelig beskyttet, og derfor fikk tokenene som ble utstedt av skyen ingen utløpstid.

En token som ikke utløper, er ikke i seg selv en sårbarhet. Det blir problematisk hvis den havner i logger eller ubeskyttede sikkerhetskopier, kommer på avveie til tredjeparter eller verken kan tilbakekalles eller roteres. Utvikleren av `Midea AC LAN` antar at Midea reagerer på disse risikoene med endringer i token-tjenestene og en mer skybasert arkitektur. Det finnes imidlertid ingen dokumentert produsentkunngjøring med tidsplan.

Språklig presisjon er viktig her. Fellesskapsintegrasjonen «hacker» ikke klimaanlegget. Den implementerer en proprietær protokoll som er rekonstruert gjennom reverse engineering. Sikkerhetsproblemet oppstår fordi langvarige hemmeligheter kan brukes og lagres utenfor den opprinnelig tiltenkte appen.

For drift i eget nettverk er det først og fremst relevant hva token og nøkkel muliggjør. Begge autentiserer den lokale kommunikasjonen med enheten. Hvis de havner i feil hender, kan en angriper, avhengig av protokollen og sin nettverksposisjon, oppdage enheten, autentisere seg overfor den, lese statusinformasjon, endre innstillinger, slå klimaanlegget av eller på, bytte driftsmodus og endre settpunktstemperaturen. Angriperen må som regel likevel kunne opprette en nettverksforbindelse til enheten; det å bare ha token og nøkkel muliggjør ikke et angrep fra hele internett. Token og nøkkel må derfor behandles som et passord. Hvordan enheten kan integreres nettverksmessig slik at disse verdiene gjør liten skade selv ved en feil, er temaet i [del to](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Hva som står igjen i praksis

Den lokale styringen av PortaSplit avhenger helt av token og nøkkel, som for tiden bare kan hentes via Midea-skyen. Denne omveien er en del av protokolldesignet: Lokale kommandoer er knyttet til skyrelaterte tilgangsdata. Fordi endepunktet er privat og udokumentert, er den langsiktige tilgjengeligheten til den uoffisielle integrasjonen usikker.

I praksis betyr det: Sikre tilgangsdata og konfigurasjon, ikke opphev en fungerende sammenkobling unødvendig, og følg med på endringer i integrasjon og fastvare. Enheter som allerede er satt opp, fortsetter å fungere lokalt. Oppsett, sikkerhetskopiering og nettverksbeskyttelse beskrives i [den praktiske artikkelen om PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Kilder

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjonen `Midea AC LAN` med «Important Notice» (siden 19. mai 2025, oppdatert 14. juli 2025), begrunnelsen knyttet til token som ikke utløper og rekonstruert klientkryptering, samt beskrivelsen av skybasert henting av token.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjonen `Midea Smart AC`: Beskrivelse av skybasert henting av token og nøkkel for V3-enheter og lokal lagring av verdiene.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om SmartHome-økosystemet og de refererte sikkerhets- og personvernstandardene.
