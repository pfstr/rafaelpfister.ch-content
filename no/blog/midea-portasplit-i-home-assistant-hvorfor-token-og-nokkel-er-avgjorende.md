---
title: "Midea PortaSplit i Home Assistant: Hvorfor token og nøkkel er avgjørende"
navTitle: "PortaSplit og token"
description: "Lokal styring krever to verdier fra Midea-skyen. Slik hentes token og nøkkel, hvorfor det er problematisk å miste dem, og hvordan eiere sikrer det eksisterende oppsettet sitt."
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
translationSourceHash: e2e3c42704dc7a3f4618f688790356c5a0ccfa18e0796789bd48cf9841bed1a8
translatedAt: 2026-08-08T14:17:25.775Z
url: https://rafaelpfister.ch/no/blog/midea-portasplit-i-home-assistant-hvorfor-token-og-nokkel-er-avgjorende
translationModel: gpt-5.6-terra
---

<aside class="article-update">
  <p class="article-update__label">Hva PortaSplit-eiere bør gjøre nå</p>
  <p>Via private skygrensesnitt henter Home Assistant enhetsspesifikk token og nøkkel under oppsettet. Prosjektet Midea AC LAN har advart mot mulige endringer siden 19. mai 2025. Produsenten har imidlertid ikke dokumentert noen konkret dato for avvikling. For eiere betyr dette:</p>
  <ol>
    <li><strong>Ikke fjern et eksisterende oppsett unødvendig.</strong> Bare innhenting av tilgangsdata krever Midea-skyen. Fremtidige endringer i det private endepunktet kan gjøre et nytt oppsett vanskeligere.</li>
    <li><strong>Sikkerhetskopier token, nøkkel og konfigurasjon kryptert.</strong> Hvis hentingen senere ikke lenger fungerer, er sikkerhetskopien den mest pålitelige måten å gjenopprette på.</li>
    <li><strong>Ikke opphev sammenkoblingen uten grunn.</strong> Fabrikkinnstillinger, fjerning fra Midea-kontoen eller bytte av Wi-Fi-modul tvinger frem ny innhenting av token, noe som i fremtiden kan mislykkes.</li>
  </ol>
  <p>Enheter som allerede er satt opp, styres lokalt. Endringer i skygrensesnittet påvirker derfor først og fremst tilføyelse og gjenoppretting, ikke hver enkelt løpende styrekommando. De konkrete trinnene finner du i <a href="/blog/midea-portasplit-home-assistant-einrichten">den praktiske artikkelen om integrering og sikring</a>.</p>
</aside>

![Eksempel på Home Assistant-dashbord for en Midea PortaSplit med rom- og innstilt temperatur, luftfuktighet, effektforbruk, energiforbruk og kompressorens driftstid de siste 24 timene.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Den lokale styringen av Midea PortaSplit bygger på to enhetsspesifikke verdier: token og nøkkel. Home Assistant-integrasjonen henter begge via et privat skyendepunkt fra Midea under oppsettet. Deretter sender den styrekommandoer direkte i det lokale nettverket.

Prosjektet Midea AC LAN advarer mot mulige endringer i disse skygrensesnittene. Nyere analyser viser imidlertid at det ikke kan utledes noen bekreftet produsentplan eller konkret dato for avvikling av dette. Denne artikkelen forklarer det tekniske avhengighetsforholdet; den [detaljerte API-analysen](/blog/midea-v2-cloud-api-portasplit-home-assistant) setter de ulike «V2»-betegnelsene og dagens situasjon i sammenheng.

## Token-spørsmålet i detalj

### Hvorfor kunne Home Assistant i det hele tatt hente tokenet tidligere?

Det interessante er at fellesskapet aldri har beregnet tokenet. I stedet analyserte det nettverkstrafikken til den offisielle appen og fant ut at appen ikke genererer tokenet selv, men henter det fra skyen:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

Home Assistant-integrasjonen har implementert nettopp dette skyoppropet på nytt. Den logger inn i skyen med de samme endepunktene og den samme prosessen som appen og mottar dermed samme token og nøkkel. Det egentlige grunnlaget er altså ikke en smart beregning, men en gjenskapt henting. Forsvinner endepunktet, forsvinner også muligheten til å hente dataene.

### Kunne man lese ut tokenet fra den offisielle appen?

Teoretisk sett, ja. Appen må kjenne tokenet på et tidspunkt, ellers kunne den ikke kommunisere lokalt med enheten. I prinsippet mulige veier ville være:

- reverse engineering av appen,
- avlytting av nettverkstrafikken, dersom den ikke er ytterligere beskyttet,
- instrumentering av appen under kjøring, for eksempel med Frida eller Objection,
- hooking av funksjonene som behandler tokenet.

Det er nettopp dette uttalelsen fra utvikleren av Midea AC LAN sikter til når han sier at det hittilværende designet er et sikkerhetsproblem sett fra Mideas ståsted: En langvarig hemmelighet som med overkommelig innsats kan hentes ut fra en bredt distribuert app, er vanskelig å kontrollere. For den enkelte brukeren er disse metodene imidlertid krevende og erstatter ikke den enkle hentingen fra skyen.

### Kunne man få tokenet direkte fra enheten?

Det ville vært den mest elegante løsningen. Hvis enheten ved første lokale sammenkobling utvekslet en offentlig nøkkel eller brukte en engangskode for sammenkobling via Bluetooth, ville ingen sky vært nødvendig. Mange moderne IoT-enheter gjør nettopp det.

Midea har imidlertid utformet den opprinnelige LAN-protokollen annerledes: Enheten aksepterer først lokale kommandoer med riktige, skyrelaterte tilgangsdata. Det finnes ingen dokumentert mekanisme for lokal sammenkobling som ville gi ut tokenet uten omveien via skyen. Skyen er dermed ikke bare en bekvemmelighet, men arkitektonisk den eneste tiltenkte veien til tokenet.

### Kan fellesskapet omgå endringer i token-endepunktet?

Det ville bare være mulig dersom ett av følgende alternativer dukker opp:

- et nytt sky-API som fortsatt leverer token,
- en hittil ukjent lokal sammenkoblingsmetode,
- en sårbarhet i enheten,
- eller at Midea selv en gang publiserer et offisielt lokalt API.

Å bare «beregne på nytt» tokenet vil derimot svært sannsynlig ikke fungere. Hvis det var mulig, ville fellesskapet trolig ha implementert det for lenge siden og aldri vært avhengig av sky-API-et. At omveien via skyen i det hele tatt ble laget, er den sterkeste indikasjonen på at det ikke finnes noen enklere lokal vei.

## Advarselen fra Midea AC LAN

Repositoryet til `Midea AC LAN` inneholder en tydelig plassert «Important Notice». Ifølge utvikleren har Midea allerede stengt de serverbaserte token-API-ene i Meiju- og SmartHome-skyen. Integrasjonen bruker derfor for tiden token-grensesnittene i NetHome Plus-skyen, og også disse skal stenges gradvis. Konsekvensen ville være at enheter som allerede er satt opp, fortsatt fungerer lokalt, men at nye enheter ikke lenger kan legges til. Utvikleren går enda lenger og skriver at Midea på sikt vil gå over til et nytt Cloud-Control-API og dermed gjøre det hittilværende V1-LAN-API-et ubrukelig.

Advarselen har en kort historie. Den fremtredende «Important Notice» kom inn i README 19. mai 2025 (Pull Request #578) og nevnte den gang SmartHome-skyen som reservealternativ for å legge til nye enheter. Den ble oppdatert 14. juli 2025 (#639); siden da viser den til NetHome Plus-skyen, fordi Midea hadde stengt flere endepunkter. Kjernen forble uendret i begge versjonene: Token-grensesnittene forsvinner gradvis, og bare skyen som fortsatt kan brukes, endres.

Dette må ses med nyanser. Det er vurderingen til et open source-prosjekt, ikke en bindende plan fra Midea, og tidsplanen er ukjent. En fremtidig fastvareoppdatering kan endre lokale funksjoner, og et token som allerede er lagret, kan fortsatt fungere, men trenger ikke gjøre det for alltid. Fabrikkinnstillinger, bytte av Wi-Fi-modul eller en ny enhet kan gjøre det nødvendig å hente tokenet på nytt.

Dette leder til de tre trinnene fra boksen i begynnelsen av artikkelen, med begrunnelsen for hvert av dem:

- **Ikke erstatt et fungerende oppsett uten grunn.** Innhenting av token er det eneste trinnet som nødvendigvis går via Midea-skyen. Endringer i det private endepunktet kan først og fremst ramme et senere nytt oppsett.
- **Sikkerhetskopier tilgangsdataene.** Home Assistant lagrer token og nøkkel lokalt. Et defekt system, en mislykket gjenoppretting eller en integrasjon som blir slettet ved en feil, kan likevel gjøre den lokale styringen ubrukelig dersom det ikke finnes en ekstern sikkerhetskopi.
- **Ikke opphev sammenkoblingen lettvint.** Om en fabrikktilbakestilling eller fjerning fra Midea-kontoen tvinger frem nye tilgangsdata for hver modell, er ikke fullt dokumentert. En sikkerhetskopi før slike endringer er derfor avgjørende.

Den løpende driften påvirkes foreløpig ikke av dette: Den lokale styringen bruker verdiene som allerede er lagret, og trenger ikke lenger token-endepunktet. Det gjenstår en restrisiko dersom en senere fastvare endrer den lokale protokollen eller autentiseringen. Hvordan token, nøkkel og konfigurasjon sikkerhetskopieres, står i [den praktiske artikkelen om oppsett](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Hva dette betyr for sikkerheten

Advarselen har, i tillegg til tilgjengelighet, en sikkerhetsmessig kjerne. Ifølge `Midea AC LAN` bygger den eldre LAN-arkitekturen på en problematisk antakelse: Klientkommunikasjonen ble opprinnelig ansett som tilstrekkelig beskyttet, og derfor fikk tokenene som ble utstedt av skyen, ingen utløpstid.

Et token som ikke utløper, er i seg selv ennå ingen sårbarhet. Det blir problematisk hvis det havner i protokoller eller ubeskyttede sikkerhetskopier, kommer på avveie til tredjeparter, eller verken kan tilbakekalles eller roteres. Utvikleren av `Midea AC LAN` antar at Midea reagerer på disse risikoene med endringer i token-tjenestene og en mer skybasert arkitektur. En tilsvarende produsentkunngjøring med tidsplan er imidlertid ikke dokumentert.

Språklig presisjon er viktig her. Fellesskapsintegrasjonen «hacker» ikke klimaanlegget. Den implementerer en proprietær protokoll som er rekonstruert gjennom reverse engineering. Sikkerhetsproblemet oppstår fordi langvarige hemmeligheter kan brukes og lagres utenfor den opprinnelig tiltenkte appen.

For drift i eget nettverk er det først og fremst relevant hva token og nøkkel muliggjør. Begge autentiserer den lokale kommunikasjonen med enheten. Dersom de havner i feil hender, kan en angriper, avhengig av protokollen og sin nettverksposisjon, oppdage enheten, autentisere seg overfor den, lese ut statusinformasjon, endre innstillinger, slå klimaanlegget av eller på, bytte driftsmodus og endre innstilt temperatur. Angriperen må som regel likevel kunne opprette en nettverksforbindelse til enheten; det å ha token og nøkkel alene muliggjør ikke et angrep fra hele internett. Token og nøkkel må derfor behandles som et passord. Hvordan enheten kan integreres nettverksmessig slik at disse verdiene gjør liten skade selv ved en hendelse, er temaet i [del to](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Hva som praktisk gjenstår

Den lokale styringen av PortaSplit står og faller med token og nøkkel, som for tiden bare kan hentes via Midea-skyen. Denne omveien er en del av protokolldesignet: Lokale kommandoer er knyttet til skyrelaterte tilgangsdata. Fordi endepunktet er privat og udokumentert, forblir den langsiktige tilgjengeligheten til den uoffisielle integrasjonen usikker.

I praksis betyr dette: Sikkerhetskopier tilgangsdata og konfigurasjon, ikke opphev en fungerende sammenkobling unødvendig, og følg med på endringer i integrasjonen og fastvaren. Enheter som allerede er satt opp, fortsetter å kjøre lokalt. Oppsett, sikkerhetskopi og nettverksbeskyttelse beskrives i [den praktiske artikkelen om PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Kilder

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjonen `Midea AC LAN` med «Important Notice» (siden 19. mai 2025, oppdatert 14. juli 2025), begrunnelsen om token som ikke utløper og rekonstruert klientkryptering, samt beskrivelsen av skybasert innhenting av token.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjonen `Midea Smart AC`: Beskrivelse av skybasert innhenting av token og nøkkel for V3-enheter og lokal lagring av verdiene.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om SmartHome-økosystemet og de refererte standardene for sikkerhet og personvern.
