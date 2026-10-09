---
title: "Midea V2, V3 og sky-API: Hva dette faktisk betyr for PortaSplit"
navTitle: "Midea V2 sky-API"
description: "Lokalt enhetsprotokoll, private app-endepunkter og offisiell partner-API bruker lignende versjonsnavn. Kildeanalysen skiller disse nivåene og setter avviklingsadvarselen i sammenheng."
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
translatedAt: 2026-10-09T10:49:17.410Z
translationReview: automatic
translationSourceHash: fb685f86fac63fa4efa6770539bc7b23d1376fc51c6cca5993d68a517872975f
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/no/blog/midea-v2-v3-og-cloud-api-hva-det-faktisk-betyr-for-portasplit
---

I Midea PortaSplit-sammenheng betegner «V2» flere innbyrdes uavhengige ting. Det finnes en lokal V2-enhetsprotokoll, versjonsnumre i private app-endepunkter og en offisiell sky-til-sky-API V2 for partnere. Den som likestiller disse nivåene, vil uunngåelig trekke feil konklusjoner om lokal styring.

Prosjektet `Midea AC LAN` advarer i sin [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) om at tidligere token-grensesnitt ville bli stengt og erstattet av en skybasert V2-API. En gjennomgang av diskusjonene, den nåværende koden og den offisielle Midea-dokumentasjonen gir et mer nyansert bilde:

> En offisiell Midea sky-til-sky-API V2 finnes. Den er imidlertid ikke identisk med token-grensesnittet som brukes av Home Assistant, og heller ikke med den lokale V2- eller V3-enhetsprotokollen. En offisielt kunngjort avvikling av lokal PortaSplit-styring med en konkret dato er ikke dokumentert. I juni 2026 ble det dessuten påvist at den angivelig avviklede SmartHome-token-API-en fortsatt fungerte – den tidligere forespørselen fra fellesskapsbiblioteket var bare ufullstendig.

Dette er del 3 av serien; [del 1](/blog/midea-portasplit-home-assistant) beskriver oppsettet frem til dashbordet, [del 2](/blog/midea-portasplit-home-assistant-absichern) sikringen av token, nøkkel og hjemmenettverk. Denne artikkelen er oppdatert per 25. juli 2026.

![Home Assistant-dashbord for Midea PortaSplit i kjøledrift: nøkkeltall øverst, termostat på 22 °C, kurver for romtemperatur, effektforbruk, dagsenergi, kompressorfrekvens, kompressordrift og viftehastighet, med tekniske verdier og status nedenfor.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Hvorfor den tidligere vurderingen må korrigeres

I en tidligere versjon av [artikkelen om token og nøkkel](/blog/midea-portasplit-home-assistant-absichern) gjenga jeg advarselen fra prosjektet `Midea AC LAN` omtrent som en varslet avvikling av skygrensesnittene. Det samsvarte med ordlyden i prosjektets README, men var formulert for sterkt som en faktisk påstand.

Advarselen er fortsatt relevant som en risikohenvisning. Den er imidlertid ikke en publisert Midea-veikart. Fremfor alt er nytt teknisk materiale nå tilgjengelig, som stiller en vesentlig del av den tidligere tolkningen i tvil.

## Slik fungerer lokal PortaSplit-styring

Home Assistant-integrasjonen `Midea Smart AC` beskriver arkitekturen sin uttrykkelig som lokal styring. På nyere V3-enheter brukes Midea-skyen bare under oppsettet, for å hente en enhetsspesifikk token og nøkkel. Deretter lagrer integrasjonen begge verdiene lokalt og trenger ingen ytterligere skyforbindelse for selve styringen. Prosjektet dokumenterer dette under [«Note On Cloud Usage»](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Forenklet ser prosessen slik ut:

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

For manuelt konfigurerte V3-enheter krever `Midea Smart AC` enhets-ID, IP-adresse, port, token og nøkkel. Den dokumenterte standardporten er `6444/TCP`; token og nøkkel oppgis som henholdsvis 128 og 64 heksadesimale tegn. Denne informasjonen finnes i [dokumentasjonen for manuell konfigurasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

En PortaSplit ble for eksempel gjenkjent i sakssporeren til `Midea AC LAN` som enhetstype `0xAC`, modell `00000Q1D` og protokollversjon 3. Den samme brukeren kunne deretter legge den til i Home Assistant via NetHome Plus. Det konkrete forløpet er dokumentert i [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

Det avgjørende er skillet:

- Skytjenesten brukes til å hente de lokale tilgangsdataene.
- Senere styring skjer direkte i LAN-et.
- En feil i token-tjenesten hindrer derfor først og fremst nye oppsett.
- Den avslutter ikke automatisk en allerede konfigurert lokal forbindelse.

Sistnevnte samsvarer også med den uttrykkelige beskrivelsen av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Hvor avviklingsadvarselen stammer fra

Advarselsteksten som er synlig i dag, ble lagt til i dokumentasjonen 19. mai 2025 med [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Begrunnelsen kan oppsummeres slik:

- De lokale tokenene hadde ingen utløpsdato.
- Ulike Home Assistant-prosjekter brukte emulert eller ekstrahert app-kryptering.
- Dette medførte et sikkerhetsproblem.
- Midea ville derfor gradvis stenge de tidligere token-tjenestene.
- På lang sikt skulle lokal V1-styring bli fortrengt av en skybasert V2-API.

I juli 2025 ble dokumentasjonen justert igjen gjennom [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). I stedet for SmartHome-skyen ble NetHome Plus nå angitt som en midlertidig brukt token-kilde. Selve avviklingsadvarselen ble stående.

Den underliggende diskusjonen er imidlertid formulert mer forsiktig enn README-en.

I [kommentaren fra Midea-AC-LAN-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) står det omtrent at NetHome Plus muligens bare er en midlertidig løsning, og at Midea etter hans forståelse har en ny, fullt skybasert V2-tjeneste.

Vedlikeholderen av `midea-msmart` svarte at han også hadde antatt at det fantes en ny V2-API, men at han bare i begrenset grad kunne undersøke den fordi han ikke hadde egne Midea-enheter. Dette står i [det direkte svarinnlegget](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Kildesituasjonen er dermed klarere:

- Advarselen stammer fra erfarne fellesskapsutviklere.
- Den bygger på observerte endringer og deres tekniske vurdering av disse.
- Én av vedlikeholderne omtaler V2-migreringen uttrykkelig som sin forståelse.
- Den andre snakker om en antakelse.
- Verken Pull Request-en eller diskusjonen lenker til en offisiell Midea-kunngjøring om avvikling eller en dato.

Det gjør ikke advarselen verdiløs. Men det gjør den til en risikoanalyse, ikke et bekreftet produsentveikart.

## Det avgjørende nye funnet fra juni 2026

15. juni 2026 ble en rettelse tatt inn i biblioteket `midea-local`, som vesentlig endrer den tidligere tolkningen.

Utgangspunktet var feilen:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Denne feilen oppsto ved henting av token og nøkkel via SmartHome-skyen. Innlogging og enhetslisten fungerte fortsatt, men kallet til `/v1/iot/secure/getToken` ble avvist.

Først så dette ut som et avviklet eller ubrukeliggjort grensesnitt. En analyse av forespørselen fra den offisielle SmartHome-appen viste imidlertid en annen årsak: Appen sendte, i tillegg til `udpid`, feltet `applianceCodes`. Fellesskapsbiblioteket hadde ikke sendt med dette feltet.

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

Også i den nåværende kildekoden brukes fortsatt nøyaktig dette endepunktet:

```text
/v1/iot/secure/getToken
```

I tillegg sendes nå `applianceCodes` med. Dette kan følges direkte i den nåværende [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Den nåværende versjonen av `Midea AC LAN` inkluderer `midea-local==6.11.0` og erklærer fortsatt seg selv som en `local_push`-integrasjon. Begge deler står i den nåværende [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

Den generelle påstanden om at SmartHome-token-API-en var blitt stengt, er dermed motbevist, i det minste for kontoene og enhetene som ble testet i juni 2026. Korrekt formulert ville det være:

> Den tidligere token-forespørselen fungerte ikke lenger etter en endring i det forventede forespørselsformatet. Etter tilpasning til formatet som brukes av den offisielle appen, leverte det samme V1-endepunktet igjen gyldige lokale tilgangsdata.

Regionale forskjeller, avvikende kontoer eller enhetstyper som ikke støttes, er dermed ikke utelukket. Men det var åpenbart ikke en global avvikling.

## Hvorfor «V2» så lett misforstås her

I Midea-sammenheng brukes minst tre innbyrdes uavhengige versjonsbetegnelser.

| Begrep | Betydning |
| --- | --- |
| Lokal V2-/V3-protokoll | Generasjon av den direkte kommunikasjonen mellom integrasjon og enhet |
| V1-/V2-app-endepunkt | Versjonsnummer for ett enkelt HTTP-endepunkt i backend-en til Midea-appene |
| Sky-til-sky-API V2 | Offisiell partner-API for autoriserte tredjepartsselskaper |

### Lokal V2 og V3

I den lokale enhetsprotokollen betegner V2 eller V3 enhetens kommunikasjonsgenerasjon. Nyere V3-enheter trenger token og nøkkel for lokal autentisering. `Midea Smart AC` dokumenterer denne forutsetningen i sin [konfigurasjonsveiledning](https://github.com/mill1000/midea-ac-py#manual-configuration).

Denne protokollversjonen har ingenting med den offisielle sky-til-sky-API V2 å gjøre.

### V1 og V2 i app-URL-er

Også i samme app kan endepunkter med ulike versjonsnumre brukes samtidig. Et `/v2/` i URL-stien betyr derfor ikke at hele plattformen er lagt om til en ny arkitektur.

Den nåværende `midea-local`-koden bruker fortsatt [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for token og nøkkel. Andre funksjoner kan likevel ligge under stier med andre versjoner.

### Offisiell sky-til-sky-API V2

Midea dokumenterer faktisk en [offisiell sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Denne bruker blant annet:

- OAuth 2.0
- `client_id` og `client_secret`
- kortlivede access-token og refresh-token
- HMAC-SHA256-signaturer
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- skybaserte statusforespørsler og styringskommandoer

Dette er et kontrollert partnergrensesnitt. Den nødvendige `client_secret` tildeles en tredjepartsleverandør av Midea. En vanlig eier av en PortaSplit får den ikke bare via sin MSmartHome-konto. Kravene og signaturreglene er beskrevet i den [offisielle V2-dokumentasjonen](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Denne API-en oppsto dessuten ikke først i 2025. Dokumentasjonen inneholder forespørselseksempler med tidsstempler fra 2018 og en Java-kommentar fra 18. april 2019. Partnergrensesnittet V2 eksisterte dermed lenge før advarselen i `Midea AC LAN`.

## Midea erstatter faktisk en V1-API – men en annen

Midea har også et eldre offisielt sky-til-sky-grensesnitt under `/v1/open/...`. Dokumentasjonen har uttrykkelig en merknad om at det ikke lenger anbefales, kan bli avviklet i fremtiden, og at den nye V2-dokumentasjonen bør brukes. Dette står i Mideas [dokumentasjon for den gamle sky-til-sky-API-en](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

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

Å likestille de tre grensesnittene bare på grunn av versjonsnummeret «V1» ville derfor ikke være teknisk berettiget.

## Finnes det allerede en fullt skybasert Home Assistant-integrasjon?

Med [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) finnes det nå en fellesskapsintegrasjon som styrer Midea-enheter via skyen i stedet for direkte over LAN-et.

Dette er imidlertid heller ikke bevis på at den offisielle partner-V2-API-en allerede har erstattet lokal styring. Den nåværende kildekoden til `Midea Auto Cloud` bruker blant annet:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Disse endepunktene kan sees i den nåværende [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Integrasjonen emulerer dermed private app- eller forbrukerskyfunksjoner. Den bruker ikke bare det dokumenterte partnergrensesnittet `/v2/open/...`.

Det finnes altså allerede et skybasert alternativ. Det medfører imidlertid også de vanlige avhengighetene ved en skyintegrasjon: internettilgang, en fungerende brukerkonto, tilgjengelige Midea-servere og fortsatt kompatible private endepunkter.

## Hva betyr dette konkret for PortaSplit-eiere?

### Allerede konfigurert lokal styring

For en allerede konfigurert PortaSplit er situasjonen forholdsvis ukritisk. `Midea Smart AC` lagrer token og nøkkel lokalt etter oppsettet og trenger ifølge sin egen [skydokumentasjon](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) ingen skyforbindelse for videre styring.

En avvikling av bare token-hentingen ville derfor ikke automatisk avslutte den eksisterende lokale forbindelsen.

### Nytt oppsett eller gjenoppretting

Risikoen er større ved:

- en ny Home Assistant-installasjon
- bytte til en annen integrasjon
- en tapt eller skadet sikkerhetskopi
- utskifting av WLAN-modulen
- endringer i enhetstilknytningen
- ny paring, dersom enhetens tilgangsdata endres som følge av dette

I slike tilfeller må integrasjonen hente token og nøkkel på nytt, eller brukeren må angi dem manuelt. At `Midea Smart AC` støtter manuell konfigurasjon, er beskrevet i dens [konfigurasjonsdokumentasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

Om en fabrikktilbakestilling eller ny paring nødvendigvis genererer nye tilgangsdata for hver PortaSplit, er ikke offisielt dokumentert og bør derfor ikke hevdes generelt.

### En reell avvikling av LAN-styring

For at en allerede konfigurert PortaSplit ikke lenger skal akseptere lokalt lagrede tilgangsdata, måtte også oppførselen til enheten eller WLAN-modulen endres, for eksempel gjennom ny fastvare eller en endret autentiseringsmetode.

En ren avvikling av skyendepunktet `/v1/iot/secure/getToken` fjerner ikke automatisk tilgangsdataene som allerede finnes i enheten og i Home Assistant. Dette følger av skillet mellom engangs skyhenting og påfølgende LAN-styring som er dokumentert av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

En slik fremtidig enhetsendring er teknisk mulig. Jeg har imidlertid ikke funnet noen konkret kunngjøring eller avviklingsdato spesifikt for PortaSplit i offentlig tilgjengelig Midea-dokumentasjon.

## Hva jeg fortsatt vil anbefale

Til tross for de relativiserende funnene er en sikkerhetskopi fornuftig.

For V3-enheter anbefaler `Midea AC LAN` uttrykkelig å sikre den genererte JSON-konfigurasjonen utenfor HAOS. Den gjeldende anbefalingen står direkte i [prosjektets README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

En sikkerhetskopi er en fornuftig beskyttelse mot skyendringer, integrasjonsproblemer og egne feil, men ikke et tegn på at en avvikling er nært forestående. [Del 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) beskriver hvordan token, nøkkel og konfigurasjon sikres.

## Vurdering basert på tilgjengelig dokumentasjon

Advarselen fra `Midea AC LAN` bør tas på alvor, men settes i riktig sammenheng.

Den dokumenterer en plausibel langsiktig risiko: Midea kan betrakte lokale token uten utløp som et sikkerhetsproblem, ytterligere begrense innhentingen av slike token eller knytte fremtidige enheter sterkere til skyen.

Det som derimot ikke er dokumentert, er en offisielt kunngjort og datofestet avvikling av lokal PortaSplit-styring.

Den nåværende tekniske situasjonen viser til og med det motsatte av en allerede gjennomført avvikling: I juni 2026 leverte det fortsatt brukte V1-token-endepunktet gyldige tilgangsdata etter at forespørselen var tilpasset formatet til den offisielle SmartHome-appen. Den relevante rettelsen er i dag en del av biblioteket som brukes av `Midea AC LAN`.

Den offisielle Midea sky-til-sky-API V2 finnes også. Den er imidlertid et eldre partnergrensesnitt med begrenset tilgang, og ikke automatisk etterfølgeren til den lokale PortaSplit-protokollen.

Den nøkterne konklusjonen er derfor:

> Lag en sikkerhetskopi, følg med på integrasjonene og ha skyavhengigheter i bakhodet – men ikke avskriv lokal PortaSplit-styring forhastet basert på en ubekreftet antakelse om avvikling.

## Kilder

1.  [Midea AC LAN: nåværende README og avviklingsadvarsel](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Advarselens ordlyd, anbefaling om sikkerhetskopi og skille mellom eldre V2- og nyere V3-enheter.

2.  [Midea AC LAN PR #578 fra 19. mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Innføring av advarselen om gradvis avvikling av token-tjenestene og den påståtte migreringen til en skybasert V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Endring av den dokumenterte token-kilden til NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskusjon om den feilaktige SmartHome-token-forespørselen og den midlertidige bruken av NetHome Plus.

5.  [Kommentar fra Midea-AC-LAN-vedlikeholderen om den antatte V2-migreringen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Markerer utsagnet om den nye V2-skyen uttrykkelig som vedkommendes egen forståelse.

6.  [Svar fra midea-msmart-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beskriver eksistensen av en ny V2-API som en antakelse og peker på de begrensede mulighetene for reverse engineering.

7.  [midea-local PR #470 fra 15. juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analyse av feil 3004, opptak av den offisielle app-forespørselen, tilføyelse av `applianceCodes` og vellykket test med fire V3-klimaanlegg.

8.  [Uforanderlig commit for SmartHome-getToken-rettelsen](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Nøyaktig kode-diff for den innarbeidede rettelsen.

9.  [Nåværende midea-local-skytjenestekode](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Fortsatt brukt endepunkt `/v1/iot/secure/getToken` og gjeldende forespørselsfelt `applianceCodes`.

10.  [Nåværende manifest for Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Brukt versjon av `midea-local` og klassifisering som lokal push-integrasjon.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentasjon av lokal styring, engangs skyhenting for V3-enheter og manuell konfigurasjon med token og nøkkel.

12.  [Midea AC LAN Issue #607 om PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkret PortaSplit-eksempel med enhetstype `0xAC`, modell `00000Q1D`, protokollversjon 3 og vellykket oppsett via NetHome Plus.

13.  [Offisiell Midea sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, Client-ID, Client-Secret, access- og refresh-token, signaturmetode og `/v2/open/...`-endepunkter.

14.  [Offisiell Midea sky-til-sky-API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Offisiell merknad om at det gamle partnergrensesnittet `/v1/open/...` ikke lenger anbefales og kan bli avviklet i fremtiden.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) og [nåværende skytjenestekode](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Fellesskapsintegrasjon for full skybasert styring og de private V1-app-endepunktene som faktisk brukes.
