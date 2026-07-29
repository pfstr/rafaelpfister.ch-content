---
title: "Midea V2, V3 og Cloud API: Hva det faktisk betyr for PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "Lokalt enhetsprotokoll, private app-endepunkter og offisielt partner-API bruker lignende versjonsnavn. Kildeanalysen skiller disse nivåene og setter nedstengningsadvarselen i kontekst."
date: "2026-07-25"
kategorie: "Home Assistant og IoT"
timeToRead: "11 min lesetid"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "midea-portasplit-home-assistant-einrichten"
draft: false
slug: "midea-v2-v3-og-cloud-api-hva-det-faktisk-betyr-for-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
url: "https://rafaelpfister.ch/no/blog/midea-v2-v3-og-cloud-api-hva-det-faktisk-betyr-for-portasplit"
---

I Midea PortaSplit-sammenheng betegner «V2» flere ting som er uavhengige av hverandre. Det finnes en lokal V2-enhetsprotokoll, versjonsnumre i private app-endepunkter og en offisiell sky-til-sky-API V2 for partnere. De som likestiller disse nivåene, vil uunngåelig trekke feil konklusjoner om lokal styring.

Prosjektet `Midea AC LAN` advarer i sin [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) om at tidligere token-grensesnitt skulle bli stengt og erstattet av en skybasert V2-API. En gjennomgang av diskusjonene, den nåværende koden og Mideas offisielle dokumentasjon gir et mer nyansert bilde:

> En offisiell Midea sky-til-sky-API V2 finnes. Den er imidlertid ikke identisk med token-grensesnittet som brukes av Home Assistant, og heller ikke med den lokale V2- eller V3-enhetsprotokollen. En offisielt kunngjort nedstenging av lokal PortaSplit-styring med en konkret dato er ikke dokumentert. I juni 2026 ble det dessuten påvist at den angivelig nedstengte SmartHome-token-API-en fortsatt fungerte – den tidligere forespørselen fra fellesskapsbiblioteket var bare ufullstendig.

Denne artikkelen er oppdatert per 25. juli 2026.

## Hvorfor den tidligere vurderingen må korrigeres

I [den første artikkelen om sky-token-spørsmålet](/blog/midea-portasplit-home-assistant) gjenga jeg i hovedsak advarselen fra prosjektet `Midea AC LAN` som en varslet nedstenging av skygrensesnittene. Det samsvarte med ordlyden i prosjektets README, men var for sterkt formulert som en faktisk påstand.

Advarselen er fortsatt relevant som en risikovurdering. Den er imidlertid ikke en publisert Midea-veikart. Fremfor alt er nytt teknisk materiale nå tilgjengelig, og dette setter en vesentlig del av den tidligere tolkningen i tvil.

## Slik fungerer lokal PortaSplit-styring

Home Assistant-integrasjonen `Midea Smart AC` beskriver arkitekturen sin uttrykkelig som lokal styring. På nyere V3-enheter brukes Midea-skyen bare under oppsettet for å hente en enhetsspesifikk token og nøkkel. Deretter lagrer integrasjonen begge verdiene lokalt og trenger ingen ytterligere skyforbindelse for den faktiske styringen. Prosjektet dokumenterer dette under [«Note On Cloud Usage"](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

For manuelt konfigurerte V3-enheter krever `Midea Smart AC` enhets-ID, IP-adresse, port, token og nøkkel. Den dokumenterte standardporten er `6444/TCP`; token og nøkkel er oppgitt som henholdsvis 128 og 64 heksadesimale tegn. Denne informasjonen finnes i [dokumentasjonen for manuell konfigurasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

En PortaSplit ble for eksempel gjenkjent i sakssporeren til `Midea AC LAN` som enhetstype `0xAC`, modell `00000Q1D` og protokollversjon 3. Den samme brukeren kunne deretter legge den til i Home Assistant via NetHome Plus. Det konkrete forløpet er dokumentert i [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

Det avgjørende er skillet:

- Skytjenesten brukes for å hente de lokale tilgangsdataene.
- Senere styring skjer direkte i LAN-et.
- En feil i token-tjenesten hindrer derfor først og fremst nye oppsett.
- Den avslutter ikke automatisk en allerede konfigurert lokal forbindelse.

Sistnevnte samsvarer også med den uttrykkelige beskrivelsen fra [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Hvor nedstengningsadvarselen kommer fra

Advarselsteksten som er synlig i dag, ble lagt til i dokumentasjonen 19. mai 2025 med [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Begrunnelsen kan oppsummeres slik:

- Lokale tokener hadde ingen utløpsdato.
- Ulike Home Assistant-prosjekter brukte etterlignet eller uttrukket app-kryptering.
- Dette skapte et sikkerhetsproblem.
- Midea ville derfor gradvis stenge de tidligere token-tjenestene.
- På lang sikt skulle lokal V1-styring bli fortrengt av en skybasert V2-API.

I juli 2025 ble dokumentasjonen justert på nytt gjennom [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). I stedet for SmartHome-skyen ble NetHome Plus nå nevnt som en midlertidig token-kilde. Selve nedstengningsadvarselen ble stående.

Den underliggende diskusjonen er imidlertid formulert mer forsiktig enn README-filen.

I [kommentaren fra Midea-AC-LAN-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) står det i hovedsak at NetHome Plus muligens bare er en midlertidig løsning, og at Midea etter hans forståelse har en ny, fullt skybasert V2-tjeneste.

Vedlikeholderen av `midea-msmart` svarte at han også hadde antatt at det fantes en ny V2-API, men at han bare i begrenset grad kunne undersøke den fordi han ikke selv har Midea-enheter. Dette står i [det direkte svarkommentaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Kildesituasjonen er dermed klarere:

- Advarselen stammer fra erfarne utviklere i fellesskapet.
- Den bygger på observerte endringer og deres tekniske vurdering.
- Én av vedlikeholderne omtaler V2-migreringen uttrykkelig som sin forståelse.
- Den andre omtaler den som en antakelse.
- Verken pull request-en eller diskusjonen lenker til en offisiell Midea-kunngjøring om nedstenging eller en dato.

Dette gjør ikke advarselen verdiløs. Men det gjør den til en risikoanalyse, ikke et bekreftet produsentveikart.

## Det avgjørende nye funnet fra juni 2026

15. juni 2026 ble det tatt inn en rettelse i biblioteket `midea-local`, som vesentlig endrer den tidligere tolkningen.

Utgangspunktet var feilen:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Denne feilen oppsto ved forespørsel om token og nøkkel via SmartHome-skyen. Innlogging og enhetslisten fungerte fortsatt, men kallet til `/v1/iot/secure/getToken` ble avvist.

Først så dette ut som et nedstengt eller ubrukeliggjort grensesnitt. En analyse av forespørselen fra den offisielle SmartHome-appen viste imidlertid en annen årsak: Appen sendte feltet `applianceCodes` i tillegg til `udpid`. Fellesskapsbiblioteket hadde ikke sendt dette feltet.

Den korrigerte forespørselen inneholder nå:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Utvikleren testet endringen med en ekte SmartHome-konto og fire V3-klimaanlegg av typen `0xAC`:

- Uten `applianceCodes` svarte serveren med feil 3004.
- Med `applianceCodes` leverte den gyldige tokener og nøkler.
- De returnerte verdiene fungerte deretter for lokal V3-autentisering.

Den fullstendige undersøkelsen, testresultatene og kode-diffen er dokumentert i [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Den tilhørende uforanderlige commit-en er [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Også i den nåværende kildekoden brukes fortsatt nøyaktig dette endepunktet:

```text
/v1/iot/secure/getToken
```

I tillegg sendes nå `applianceCodes` med. Dette kan følges direkte i den nåværende [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Den nåværende versjonen av `Midea AC LAN` inkluderer `midea-local==6.11.0` og erklærer seg fortsatt som en `local_push`-integrasjon. Begge deler står i det nåværende [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

Den generelle påstanden om at SmartHome-token-API-en er blitt stengt, er dermed tilbakevist, i hvert fall for kontoene og enhetene som ble testet i juni 2026. Korrekt formulert ville det være:

> Den tidligere token-forespørselen fungerte ikke lenger etter en endring i det forventede forespørselsformatet. Etter tilpasning til formatet som brukes av den offisielle appen, leverte det samme V1-endepunktet igjen gyldige lokale tilgangsdata.

Regionale forskjeller, avvikende kontoer eller enhetstyper som ikke støttes, er dermed ikke utelukket. Men det var åpenbart ikke en global nedstenging.

## Hvorfor «V2" så lett misforstås her

I Midea-sammenheng brukes minst tre uavhengige versjonsbetegnelser.

| Begrep | Betydning |
| --- | --- |
| Lokal V2-/V3-protokoll | Generasjon av direkte kommunikasjon mellom integrasjon og enhet |
| V1-/V2-app-endepunkt | Versjonsnummer for ett enkelt HTTP-endepunkt i backend-en til Midea-appene |
| Sky-til-sky-API V2 | Offisiell partner-API for autoriserte tredjepartsselskaper |

### Lokal V2 og V3

I den lokale enhetsprotokollen betegner V2 og V3 enhetens kommunikasjonsgenerasjon. Nyere V3-enheter trenger token og nøkkel for lokal autentisering. `Midea Smart AC` dokumenterer dette kravet i sin [konfigurasjonsveiledning](https://github.com/mill1000/midea-ac-py#manual-configuration).

Denne protokollversjonen har ingenting med den offisielle sky-til-sky-API-en V2 å gjøre.

### V1 og V2 i app-URL-er

Også i samme app kan endepunkter med ulike versjonsnumre brukes samtidig. En `/v2/` i URL-stien betyr derfor ikke at hele plattformen er blitt lagt om til en ny arkitektur.

Den nåværende `midea-local`-koden bruker fortsatt [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) for token og nøkkel. Andre funksjoner kan likevel ligge under stier med andre versjoner.

### Offisiell sky-til-sky-API V2

Midea dokumenterer faktisk en [offisiell sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Denne bruker blant annet:

- OAuth 2.0
- `client_id` og `client_secret`
- kortvarige tilgangstokener og oppdateringstokener
- HMAC-SHA256-signaturer
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- skybaserte statusforespørsler og styringskommandoer

Dette er et kontrollert partnergrensesnitt. Den nødvendige `client_secret` tildeles en tredjepartsleverandør av Midea. En vanlig eier av en PortaSplit får den ikke bare via sin MSmartHome-konto. Kravene og signaturreglene er beskrevet i den [offisielle V2-dokumentasjonen](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Denne API-en oppsto dessuten ikke først i 2025. Dokumentasjonen inneholder forespørselseksempler med tidsstempler fra 2018 og en Java-kommentar fra 18. april 2019. V2-partnergrensesnittet fantes dermed lenge før advarselen i `Midea AC LAN`.

## Midea erstatter faktisk en V1-API – men en annen

Midea har også et eldre offisielt sky-til-sky-grensesnitt under `/v1/open/...`. Dokumentasjonen inneholder uttrykkelig en merknad om at det ikke lenger anbefales, kan bli stengt i fremtiden, og at den nye V2-dokumentasjonen bør brukes. Dette står i Mideas [dokumentasjon for den gamle sky-til-sky-API-en](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Denne merknaden er en reell, offisiell V1-til-V2-migrering. Den gjelder imidlertid partner-endepunktene:

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

Det ville derfor ikke være teknisk berettiget å likestille de tre grensesnittene bare på grunn av versjonsnummeret «V1".

## Finnes det allerede en fullstendig skybasert Home Assistant-integrasjon?

Med [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) finnes det nå en fellesskapsintegrasjon som styrer Midea-enheter via skyen i stedet for direkte via LAN.

Dette er imidlertid heller ikke bevis på at den offisielle partner-V2-API-en allerede har erstattet lokal styring. Den nåværende kildekoden til `Midea Auto Cloud` bruker blant annet:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Disse endepunktene kan sees i den nåværende [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Integrasjonen etterligner dermed private app- eller forbrukerskyfunksjoner. Den bruker ikke bare det dokumenterte `/v2/open/...`-partnergrensesnittet.

Det finnes altså allerede et skybasert alternativ. Men det medfører også de vanlige avhengighetene ved en skyintegrasjon: Internett-tilgang, en fungerende brukerkonto, tilgjengelige Midea-servere og fortsatt kompatible private endepunkter.

## Hva betyr dette konkret for PortaSplit-eiere?

### Allerede konfigurert lokal styring

For en PortaSplit som allerede er konfigurert, er situasjonen relativt avslappet. `Midea Smart AC` lagrer token og nøkkel lokalt etter oppsettet og trenger ifølge sin egen [skydokumentasjon](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) ingen skyforbindelse for videre styring.

En nedstenging av ren token-henting vil derfor ikke automatisk avslutte den eksisterende lokale forbindelsen.

### Nytt oppsett eller gjenoppretting

Risikoen er større ved:

- en ny Home Assistant-installasjon
- overgang til en annen integrasjon
- en tapt eller skadet sikkerhetskopi
- bytte av Wi-Fi-modul
- endringer i enhetstilknytningen
- ny paring dersom enhetens tilgangsdata endres i prosessen

I slike tilfeller må integrasjonen hente token og nøkkel på nytt, eller brukeren må oppgi dem manuelt. At `Midea Smart AC` støtter manuell konfigurasjon, er beskrevet i dens [konfigurasjonsdokumentasjon](https://github.com/mill1000/midea-ac-py#manual-configuration).

Det er ikke offisielt dokumentert om en fabrikktilbakestilling eller ny paring nødvendigvis genererer nye tilgangsdata for hver PortaSplit, og det bør derfor ikke hevdes generelt.

### En reell nedstenging av LAN-styring

For at en allerede konfigurert PortaSplit ikke lenger skal godta de lokalt lagrede tilgangsdataene, måtte også enhetens eller Wi-Fi-modulens oppførsel endres – for eksempel gjennom ny fastvare eller en endret autentiseringsmetode.

En ren nedstenging av skyendepunktet `/v1/iot/secure/getToken` fjerner ikke automatisk tilgangsdataene som allerede finnes i enheten og Home Assistant. Dette følger av skillet mellom engangs henting fra skyen og påfølgende LAN-styring, som er dokumentert av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

En slik fremtidig enhetsendring er teknisk mulig. Jeg har imidlertid ikke funnet noen konkret kunngjøring eller nedstengningsdato spesifikt for PortaSplit i Mideas offentlig tilgjengelige dokumentasjon.

## Hva jeg fortsatt vil anbefale

Til tross for de nyanserende funnene er en sikkerhetskopi fornuftig.

For V3-enheter anbefaler `Midea AC LAN` uttrykkelig å lagre den genererte JSON-konfigurasjonen utenfor HAOS. Den nåværende anbefalingen står direkte i [prosjektets README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Følgende gjelder:

- Behandle token og nøkkel som passord.
- Ikke last opp JSON-filen til et offentlig Git-repositorium.
- Ikke publiser usladdede feilsøkingslogger.
- Krypter sikkerhetskopien.
- Opprett i tillegg en fullstendig Home Assistant-sikkerhetskopi.
- Kontroller at alt fungerer før fastvare- og integrasjonsoppdateringer.
- Test den lokale styringen på nytt etter oppdateringer.

En sikkerhetskopi er en fornuftig sikring mot skyendringer, integrasjonsproblemer og egne feil. Den er imidlertid ikke et tegn på at en nedstenging er nært forestående. Hvordan en PortaSplit kan settes opp riktig og sikres i hjemmenettverket, står i [den praktiske delen om oppsett](/blog/midea-portasplit-home-assistant-einrichten).

## Vurdering basert på tilgjengelig dokumentasjon

Advarselen fra `Midea AC LAN` bør tas på alvor, men settes i riktig sammenheng.

Den dokumenterer en plausibel langsiktig risiko: Midea kan oppfatte lokale tokener uten utløp som et sikkerhetsproblem, begrense innhenting av slike tokener ytterligere eller knytte fremtidige enheter sterkere til skyen.

Det er derimot ikke dokumentert noen offisielt kunngjort, datofestet nedstenging av lokal PortaSplit-styring.

Den nåværende tekniske statusen viser til og med det motsatte av en allerede gjennomført nedstenging: I juni 2026 leverte det fortsatt brukte V1-token-endepunktet gyldige tilgangsdata etter at forespørselen var tilpasset formatet til den offisielle SmartHome-appen. Den aktuelle rettelsen er i dag en del av biblioteket som brukes av `Midea AC LAN`.

Den offisielle Midea sky-til-sky-API-en V2 finnes også. Den er imidlertid et eldre partnergrensesnitt med begrenset tilgang, og ikke automatisk etterfølgeren til den lokale PortaSplit-protokollen.

Den nøkterne konklusjonen er derfor:

> Opprett en sikkerhetskopi, følg med på integrasjonene og ha skyavhengigheter i bakhodet – men ikke avskriv lokal PortaSplit-styring for tidlig basert på en ubekreftet antakelse om nedstenging.

## Kilder

1.  [Midea AC LAN: nåværende README og nedstengningsadvarsel](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Ordlyden i advarselen, anbefaling om sikkerhetskopi og skillet mellom eldre V2- og nyere V3-enheter.

2.  [Midea AC LAN PR #578 fra 19. mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Innføring av advarselen om gradvis nedstenging av token-tjenestene og den påståtte migreringen til en skybasert V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Endring av den dokumenterte token-kilden til NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskusjon om den feilaktige SmartHome-token-forespørselen og midlertidig bruk av NetHome Plus.

5.  [Kommentar fra Midea-AC-LAN-vedlikeholderen om den antatte V2-migreringen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Markerer uttrykkelig uttalelsen om den nye V2-skyen som hans egen forståelse.

6.  [Svar fra midea-msmart-vedlikeholderen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beskriver eksistensen av en ny V2-API som en antakelse og peker på begrensede muligheter for reversering.

7.  [midea-local PR #470 fra 15. juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analyse av feil 3004, opptak av den offisielle app-forespørselen, tillegg av `applianceCodes` og vellykket test med fire V3-klimaanlegg.

8.  [Uforanderlig commit for SmartHome-getToken-rettelsen](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Nøyaktig kode-diff for den innlemmede rettelsen.

9.  [Nåværende midea-local-skytjenestekode](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Fortsatt brukt endepunkt `/v1/iot/secure/getToken` og gjeldende forespørselsfelt `applianceCodes`.

10.  [Nåværende manifest for Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Brukt versjon av `midea-local` og klassifisering som lokal push-integrasjon.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentasjon av lokal styring, engangs henting fra skyen for V3-enheter og manuell konfigurasjon med token og nøkkel.

12.  [Midea AC LAN Issue #607 om PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkret PortaSplit-eksempel med enhetstype `0xAC`, modell `00000Q1D`, protokollversjon 3 og vellykket oppsett via NetHome Plus.

13.  [Offisiell Midea sky-til-sky-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, klient-ID, klienthemmelighet, tilgangs- og oppdateringstokener, signaturmetode og `/v2/open/...`-endepunkter.

14.  [Offisiell Midea sky-til-sky-API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Offisiell merknad om at det gamle `/v1/open/...`-partnergrensesnittet ikke lenger anbefales og kan bli stengt i fremtiden.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) og [nåværende skytjenestekode](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Fellesskapsintegrasjon for fullstendig skystyring og de private V1-app-endepunktene som faktisk brukes.
