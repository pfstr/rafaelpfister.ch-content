---
title: "Midea V2, V3 og sky-API: Hva dette faktisk betyr for PortaSplit"
navTitle: "Midea V2 sky-API"
description: "Lokalt enhetsprotokoll, private app-endepunkter og offisielt partner-API bruker lignende versjonsnavn. Kildeanalysen skiller disse nivåene og setter nedstengningsadvarselen i sammenheng."
date: "2026-07-25"
kategorie: "Home Assistant og IoT"
timeToRead: "11 min lesetid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
slug: "midea-v2-v3-og-cloud-api-hva-det-faktisk-betyr-for-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:36:46.612Z
translationReview: automatic
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/midea-v2-v3-og-cloud-api-hva-det-faktisk-betyr-for-portasplit
---

I Midea PortaSplit-sammenheng betegner «V2» flere innbyrdes uavhengige ting. Det finnes en lokal V2-enhetsprotokoll, versjonsnumre i private app-endepunkter og et offisielt sky-til-sky-API V2 for partnere. De som likestiller disse nivåene, trekker uunngåelig feil slutninger om lokal styring.

Prosjektet `Midea AC LAN` advarer i sin [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) om at tidligere token-grensesnitt skulle stenges og erstattes av et skybasert V2-API. En gjennomgang av diskusjonene, gjeldende kode og den offisielle Midea-dokumentasjonen gir et mer nyansert bilde:

> Et offisielt Midea sky-til-sky-API V2 finnes. Det er imidlertid ikke identisk med token-grensesnittet som brukes av Home Assistant, og heller ikke med den lokale V2- eller V3-enhetsprotokollen. En offisielt varslet nedstenging av den lokale PortaSplit-styringen med en konkret dato er ikke dokumentert. I juni 2026 ble det dessuten påvist at det angivelig nedstengte SmartHome-token-API-et fortsatt fungerte – den tidligere forespørselen fra community-biblioteket var bare ufullstendig.

Dette er del 3 av serien; [del 1](/blog/midea-portasplit-home-assistant) beskriver oppsettet frem til dashbordet, og [del 2](/blog/midea-portasplit-home-assistant-absichern) sikring av token, nøkkel og hjemmenettverk. Denne artikkelen er oppdatert per 25. oktober 2026.

![Home Assistant-dashbord for Midea PortaSplit i kjøledrift: nøkkeltall øverst, termostat på 22 °C, historikk for romtemperatur, effektforbruk, daglig energi, kompressorfrekvens, kompressordrift og viftehastighet, med tekniske verdier og status nedenfor.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Hvorfor den tidligere vurderingen må korrigeres

I en tidligere versjon av [artikkelen om token og nøkkel](/blog/midea-portasplit-home-assistant-absichern) gjenga jeg advarselen fra prosjektet `Midea AC LAN` omtrent som en varslet nedstenging av skygrensesnittene. Det tilsvarte ordlyden i prosjektets README, men var formulert for sterkt som en påstand om fakta.

Advarselen er fortsatt relevant som en risikoindikasjon. Den er imidlertid ikke en offentliggjort Midea-veikart. Fremfor alt er nytt teknisk materiale nå tilgjengelig som setter en vesentlig del av den tidligere tolkningen i tvil.

## Slik fungerer lokal PortaSplit-styring

Home Assistant-integrasjonen `Midea Smart AC` beskriver arkitekturen uttrykkelig som lokal styring. For nyere V3-enheter brukes Midea-skyen bare under oppsettet for å hente en enhetsspesifikk token og nøkkel. Deretter lagrer integrasjonen begge verdiene lokalt og trenger ingen ytterligere skyforbindelse for selve styringen. Prosjektet dokumenterer dette under [«Note On Cloud Usage»](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Forenklet ser forløpet slik ut:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

For manuelt konfigurerte V3-enheter krever `Midea Smart AC` enhets-ID, IP-adresse, port, token og nøkkel. Den dokumenterte standardporten er `6444/TCP`; token og nøkkel oppgis med henholdsvis 128 og 64 heksadesimale tegn. Denne informasjonen finnes i [dokumentasjonen for manuell konfigurasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

En PortaSplit ble for eksempel registrert i sakssporeren til `Midea AC LAN` som enhetstype `0xAC`, modell `00000Q1D` og protokollversjon 3. Den samme brukeren kunne deretter legge den til i Home Assistant via NetHome Plus. Det konkrete forløpet er dokumentert i [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

Det avgjørende er skillet:

- Skytjenesten brukes til å hente lokale tilgangsdata.
- Senere styring skjer direkte i LAN-et.
- En feil i token-tjenesten hindrer derfor først og fremst nye oppsett.
- Den avslutter ikke automatisk en allerede konfigurert lokal forbindelse.

Sistnevnte samsvarer også med den uttrykkelige beskrivelsen av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Hvor nedstengningsadvarselen kommer fra

Advarselsteksten som er synlig i dag, ble lagt til i dokumentasjonen 19. mai 2025 med [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Begrunnelsen kan oppsummeres slik:

- De lokale tokenene skulle ikke ha noen utløpsdato.
- Ulike Home Assistant-prosjekter brukte etterlignet eller ekstrahert app-kryptering.
- Dette medførte et sikkerhetsproblem.
- Midea ville derfor gradvis stenge de tidligere token-tjenestene.
- På lang sikt skulle lokal V1-styring fortrenges av et skybasert V2-API.

I juli 2025 ble dokumentasjonen justert på nytt gjennom [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). I stedet for SmartHome-skyen ble NetHome Plus nå oppgitt som midlertidig brukt token-kilde. Selve nedstengningsadvarselen ble stående.

Den underliggende diskusjonen er imidlertid formulert mer forsiktig enn README-en.

I [kommentaren fra Midea-AC-LAN-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) står det omtrent at NetHome Plus muligens bare er en midlertidig løsning, og at Midea etter hans forståelse har en ny, fullt skybasert V2-tjeneste.

Vedlikeholderen av `midea-msmart` svarte at han også hadde antatt at et nytt V2-API eksisterte, men på grunn av manglende egne Midea-enheter bare kunne undersøke det i begrenset grad. Dette står i [den direkte svarkommentaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Kildesituasjonen er dermed klarere:

- Advarselen kommer fra erfarne community-utviklere.
- Den bygger på observerte endringer og deres tekniske vurdering.
- Én av vedlikeholderne omtaler V2-migreringen uttrykkelig som sin forståelse.
- Den andre omtaler det som en antakelse.
- Verken Pull Request-en eller diskusjonen lenker til en offisiell Midea-kunngjøring om nedstenging eller en dato.

Det gjør ikke advarselen verdiløs. Men det gjør den til en risikoanalyse og ikke et bekreftet produsent-veikart.

## Det avgjørende nye funnet fra juni 2026

15. juni 2026 ble en rettelse tatt inn i biblioteket `midea-local`, som vesentlig endrer den tidligere tolkningen.

Utgangspunktet var feilen:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Denne feilen oppsto ved henting av token og nøkkel via SmartHome-skyen. Innlogging og enhetslisten fortsatte å fungere, men kall til `/v1/iot/secure/getToken` ble avvist.

Først så dette ut som et grensesnitt som var stengt eller gjort ubrukelig. En analyse av forespørselen fra den offisielle SmartHome-appen viste imidlertid en annen årsak: I tillegg til `udpid` sendte appen feltet `applianceCodes`. Community-biblioteket sendte ikke med dette feltet.

Den korrigerte forespørselen inneholder nå:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Utvikleren testet endringen med en ekte SmartHome-konto og fire V3-klimaanlegg av typen `0xAC`:

- Uten `applianceCodes` svarte serveren med feil 3004.
- Med `applianceCodes` leverte den gyldige token og nøkler.
- De returnerte verdiene fungerte deretter for lokal V3-autentisering.

Den fullstendige undersøkelsen, testresultatene og kode-diffen er dokumentert i [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Den tilhørende uforanderlige commit-en er [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Også i gjeldende kildekode brukes fortsatt nøyaktig dette endepunktet:

```text
/v1/iot/secure/getToken
```

I tillegg sendes nå `applianceCodes` med. Dette kan spores direkte i den gjeldende [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Siden 12. august 2026 bruker `Midea AC LAN` sitt eget bibliotek `midea-lan` i stedet for `midea-local`; den gjeldende versjonen inkluderer `midea-lan==2026.10.0` og erklærer seg fortsatt som en `local_push`-integrasjon. Begge deler står i gjeldende [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json). `midea-lan` henter også token og nøkkel fra SmartHome-skyen via `/v1/iot/secure/getToken` med `applianceCodes`, slik det fremgår av [`midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

Den generelle påstanden om at SmartHome-token-API-et ble stengt, er dermed tilbakevist, i hvert fall for kontoene og enhetene som ble testet i juni 2026. Korrekt formulert ville det være:

> Den tidligere token-forespørselen fungerte ikke lenger etter en endring i det forventede forespørselsformatet. Etter tilpasning til formatet som brukes av den offisielle appen, leverte det samme V1-endepunktet igjen gyldige lokale tilgangsdata.

Regionale forskjeller, avvikende kontoer eller enhetstyper som ikke støttes, er dermed ikke utelukket. Men det var åpenbart ikke en global nedstenging.

For andre Midea-skyer har situasjonen endret seg siden den gang. Den kinesiske Meiju-skyen har svart på `/v1/iot/secure/getToken` med feilen `40404` siden august 2026; token og nøkkel leveres der kun av app-endepunktet `/v2/iot/secure/getToken`, som i tillegg krever en `homegroupId`. Bibliotekene ble tilpasset tilsvarende i slutten av august 2026, dokumentert i [`midea-local` Pull Request #717](https://github.com/midea-lan/midea-local/pull/717). Via NetHome-Plus-kontoer kan det ifølge [README-en til `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice) for tiden ikke lenger hentes token; bare personlige Meiju- og SmartHome-kontoer som enheten er knyttet til, kan brukes. Også denne endringen gjelder et privat app-endepunkt, ikke det offisielle sky-til-sky-API-et V2.

## Hvorfor «V2» så lett misforstås her

I Midea-sammenheng brukes minst tre innbyrdes uavhengige versjonsbetegnelser.

| Begrep | Betydning |
| --- | --- |
| Lokalt V2-/V3-protokoll | Generasjon for direkte kommunikasjon mellom integrasjon og enhet |
| V1-/V2-app-endepunkt | Versjonsnummeret til ett enkelt HTTP-endepunkt i backend-en til Midea-appene |
| Sky-til-sky-API V2 | Offisielt partner-API for autoriserte tredjepartsbedrifter |

### Lokalt V2 og V3

I den lokale enhetsprotokollen betegner V2 og V3 enhetens kommunikasjonsgenerasjon. Nyere V3-enheter trenger token og nøkkel for lokal autentisering. `Midea Smart AC` dokumenterer denne forutsetningen i sin [konfigurasjonsveiledning](https://github.com/mill1000/midea-ac-py#manual-configuration).

Denne protokollversjonen har ingenting med det offisielle sky-til-sky-API-et V2 å gjøre.

### V1 og V2 i app-URL-er

Også i samme app kan endepunkter med ulike versjonsnumre brukes samtidig. Et `/v2/` i URL-stien betyr derfor ikke at hele plattformen er lagt om til en ny arkitektur.

Den gjeldende `midea-local`-koden bruker fortsatt [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for token og nøkkel. Andre funksjoner kan likevel ligge under stier med andre versjoner.

### Offisielt sky-til-sky-API V2

Midea dokumenterer faktisk et [offisielt sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Dette bruker blant annet:

- OAuth 2.0
- `client_id` og `client_secret`
- kortvarige access-token og refresh-token
- HMAC-SHA256-signaturer
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- skybaserte statusforespørsler og styrekommandoer

Dette er et kontrollert partnergrensesnitt. Den nødvendige `client_secret` tildeles en tredjepartsleverandør av Midea. En vanlig eier av en PortaSplit får den ikke bare via sin MSmartHome-konto. Kravene og signaturreglene er beskrevet i den [offisielle V2-dokumentasjonen](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Dette API-et ble dessuten ikke opprettet først i 2025. Dokumentasjonen inneholder forespørselseksempler med tidsstempler fra 2018 og en Java-kommentar fra 18. april 2019. V2-partnergrensesnittet eksisterte dermed lenge før advarselen i `Midea AC LAN`.

## Midea erstatter faktisk et V1-API – men et annet

Midea har også et eldre offisielt sky-til-sky-grensesnitt under `/v1/open/...`. Dokumentasjonen har uttrykkelig en merknad om at det ikke lenger anbefales, kan bli stengt i fremtiden, og at den nye V2-dokumentasjonen bør brukes. Dette står i Mideas [dokumentasjon for det gamle sky-til-sky-API-et](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Denne merknaden er en reell offisiell V1-til-V2-migrering. Den gjelder imidlertid partnerendepunktene:

```text
/v1/open/...
           ↓
/v2/open/...
```

Token-forespørselen som brukes av Home Assistant-bibliotekene, er derimot:

```text
/v1/iot/secure/getToken
```

Og den lokale PortaSplit-forbindelsen går deretter ikke lenger via en slik sky-URL, men direkte i hjemmenettverket.

Det ville derfor ikke være teknisk begrunnet å likestille de tre grensesnittene bare på grunn av versjonsnummeret «V1».

## Finnes det allerede en fullt skybasert Home Assistant-integrasjon?

Med [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) finnes det nå en community-integrasjon som styrer Midea-enheter via skyen i stedet for direkte over LAN-et.

Dette er imidlertid heller ikke bevis på at det offisielle partner-V2-API-et allerede har erstattet lokal styring. Den gjeldende kildekoden til `Midea Auto Cloud` bruker blant annet:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Disse endepunktene kan ses i den gjeldende [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Integrasjonen etterligner dermed private app- eller forbrukerskyfunksjoner. Den bruker ikke bare det dokumenterte `/v2/open/...`-partnergrensesnittet.

En skybasert alternativ løsning finnes altså allerede. Men den har også de vanlige avhengighetene til en skyintegrasjon: Internett-tilgang, en fungerende brukerkonto, tilgjengelige Midea-servere og fortsatt kompatible private endepunkter.

## Hva betyr dette konkret for PortaSplit-eiere?

### Allerede konfigurert lokal styring

For en allerede konfigurert PortaSplit er situasjonen relativt ukritisk. `Midea Smart AC` lagrer token og nøkkel lokalt etter oppsettet og trenger ifølge sin egen [skydokumentasjon](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) ingen skyforbindelse for videre styring.

En nedstenging av ren token-henting ville derfor ikke automatisk avslutte den eksisterende lokale forbindelsen.

### Nytt oppsett eller gjenoppretting

Risikoen er større ved:

- en ny Home Assistant-installasjon
- bytte til en annen integrasjon
- en tapt eller skadet sikkerhetskopi
- utskifting av WLAN-modulen
- endringer i enhetstilknytningen
- en ny sammenkobling dersom enhetens tilgangsdata endres samtidig

I slike tilfeller må integrasjonen hente token og nøkkel på nytt, eller brukeren må angi dem manuelt. At `Midea Smart AC` støtter manuell konfigurasjon, er beskrevet i dens [konfigurasjonsdokumentasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

Om en fabrikktilbakestilling eller ny sammenkobling nødvendigvis genererer nye tilgangsdata for hver PortaSplit, er ikke offisielt dokumentert og bør derfor ikke påstås generelt.

### En reell nedstenging av LAN-styringen

For at en allerede konfigurert PortaSplit ikke lenger skal godta sine lokalt lagrede tilgangsdata, måtte også oppførselen til enheten eller WLAN-modulen endres, for eksempel gjennom ny fastvare eller en endret autentiseringsmekanisme.

En ren nedstenging av skyendepunktet `/v1/iot/secure/getToken` fjerner ikke automatisk tilgangsdataene som allerede finnes i enheten og i Home Assistant. Dette følger av skillet mellom engangsinnhenting fra skyen og påfølgende LAN-styring, som er dokumentert av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

En slik fremtidig enhetsendring er teknisk mulig. Jeg har imidlertid ikke funnet noen konkret kunngjøring eller nedstengningsdato spesielt for PortaSplit i offentlig tilgjengelig Midea-dokumentasjon.

## Hva jeg fortsatt ville anbefalt

Til tross for de nyanserende funnene er det fortsatt fornuftig med en sikkerhetskopi.

For V3-enheter anbefaler `Midea AC LAN` uttrykkelig å sikre den genererte JSON-konfigurasjonen utenfor HAOS. Den gjeldende anbefalingen står direkte i [prosjektets README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

En sikkerhetskopi er en fornuftig sikring mot skyendringer, integrasjonsproblemer og egne feil, men ikke et tegn på at en nedstenging er nært forestående. [Del 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) beskriver hvordan token, nøkkel og konfigurasjon sikres.

## Vurdering basert på tilgjengelig dokumentasjon

Advarselen fra `Midea AC LAN` bør tas alvorlig, men settes inn i riktig sammenheng.

Den dokumenterer en plausibel langsiktig risiko: Midea kan vurdere lokale token uten utløp som et sikkerhetsproblem, ytterligere begrense anskaffelsen av slike token eller knytte fremtidige enheter tettere til skyen.

Det som derimot ikke er dokumentert, er en offisielt varslet og datofestet nedstenging av lokal PortaSplit-styring.

Den gjeldende tekniske situasjonen viser til og med det motsatte av en allerede gjennomført nedstenging: I juni 2026 leverte det fortsatt brukte V1-token-endepunktet gyldige tilgangsdata etter at forespørselen var tilpasset formatet til den offisielle SmartHome-appen. Den tilsvarende rettelsen er nå en del av biblioteket som brukes av `Midea AC LAN`.

Det offisielle Midea sky-til-sky-API-et V2 finnes også. Men det er et eldre partnergrensesnitt med begrenset tilgang, og ikke automatisk etterfølgeren til den lokale PortaSplit-protokollen.

Den nøkterne konklusjonen er derfor:

> Opprett en sikkerhetskopi, følg med på integrasjonene og ha skyavhengigheter i bakhodet – men ikke avskriv lokal PortaSplit-styring for tidlig på grunnlag av en ubekreftet antakelse om nedstenging.

## Endringslogg

- **25. oktober 2026:** Lagt til: Midea AC LAN bruker siden august 2026 biblioteket midea-lan (for tiden 2026.10.0), og Meiju-skyen leverer nå bare token og nøkkel via endepunktet /v2/iot/secure/getToken med homegroupId. Artikkelens status er oppdatert til 25. oktober 2026, og kildene er supplert.

## Kilder

1.  [Midea AC LAN: gjeldende README og nedstengningsadvarsel](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Advarselens ordlyd, anbefaling om sikkerhetskopi og skillet mellom eldre V2- og nyere V3-enheter.

2.  [Midea AC LAN PR #578 fra 19. mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Innføring av advarselen om gradvis nedstenging av token-tjenestene og den påståtte migreringen til et skybasert V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Endring av den dokumenterte token-kilden til NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskusjon om den feilaktige SmartHome-token-forespørselen og den midlertidige bruken av NetHome Plus.

5.  [Kommentar fra Midea-AC-LAN-vedlikeholderen om den antatte V2-migreringen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Markerer uttrykkelig utsagnet om den nye V2-skyen som egen forståelse.

6.  [Svar fra midea-msmart-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beskriver eksistensen av et nytt V2-API som en antakelse og peker på begrensede muligheter for reverse engineering.

7.  [midea-local PR #470 fra 15. juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analyse av feil 3004, opptak av den offisielle app-forespørselen, tillegg av `applianceCodes` og vellykket test med fire V3-klimaanlegg.

8.  [Uforanderlig commit for SmartHome-getToken-rettelsen](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Nøyaktig kode-diff for den innførte rettelsen.

9.  [Gjeldende midea-local-skykode](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Fortsatt brukt endepunkt `/v1/iot/secure/getToken` og gjeldende forespørselsfelt `applianceCodes`.

10.  [Gjeldende manifest fra Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Brukt versjon av `midea-lan` og klassifisering som lokal push-integrasjon.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentasjon av lokal styring, engangsinnhenting fra skyen for V3-enheter og manuell konfigurasjon med token og nøkkel.

12.  [Midea AC LAN Issue #607 om PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkret PortaSplit-eksempel med enhetstype `0xAC`, modell `00000Q1D`, protokollversjon 3 og vellykket oppsett via NetHome Plus.

13.  [Offisielt Midea sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, klient-ID, klienthemmelighet, access- og refresh-token, signaturmetode og `/v2/open/...`-endepunkter.

14.  [Offisielt Midea sky-til-sky-API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Offisiell merknad om at det gamle `/v1/open/...`-partnergrensesnittet ikke lenger anbefales og kan bli stengt i fremtiden.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) og [gjeldende skykode](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Community-integrasjon for fullstendig skystyring og de faktisk brukte private V1-app-endepunktene.

16.  [midea-local PR #717](https://github.com/midea-lan/midea-local/pull/717): Bytte i Meiju-skyen fra `/v1/iot/secure/getToken` (feil 40404) til `/v2/iot/secure/getToken` med `homegroupId`.

17.  [Gjeldende midea-lan-skykode](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py): Bibliotek brukt av Midea AC LAN siden august 2026, SmartHome-token-forespørsel via `/v1/iot/secure/getToken` med `applianceCodes`.
