---
title: "Styr Midea PortaSplit lokalt med Home Assistant og bruk den på en sikker måte"
navTitle: "Konfigurere PortaSplit"
description: "Fra riktig community-integrasjon til IoT-VLAN: Slik konfigurerer du PortaSplit, sikrer token og nøkkel og begrenser sky- og nettverkstilgang."
date: "2026-07-24"
kategorie: "Home Assistant og IoT"
timeToRead: "14 min lesetid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - serverloser-newsletter-cloudflare-workers-d1
slug: "styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert"
translationOf: "midea-portasplit-home-assistant-einrichten"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: bbe70b67dd255184cf0db69f7308c756937dc961c3c83e152268ee668f93dd07
translatedAt: 2026-09-04T08:37:13.699Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/no/blog/styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert
---

Midea PortaSplit kan etter oppsett styres direkte i det lokale nettverket via Home Assistant. Til dette trenger community-integrasjonen to enhetsspesifikke tilgangsverdier fra Midea-skyen: token og nøkkel.

Denne artikkelen veileder deg gjennom valg, oppsett og sikring av integrasjonen. Løsningene som beskrives, kommer fra communityet og støttes ikke offisielt av verken Midea eller Home Assistant. Firmware- eller skyendringer kan derfor når som helst påvirke hvordan de fungerer. Bakgrunnen for token-grensesnittet og den tvetydige advarselen om nedstengning finnes i [analysen av Midea-sky-API-ene](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Slik fungerer lokal styring

De faktiske styringskommandoene går etter oppsettet direkte fra Home Assistant til PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

En bryterkommando trenger ikke å gå via en ekstern Midea-server, responstiden er kort, en feil i Midea-skyen avbryter ikke nødvendigvis den allerede konfigurerte lokale styringen, og enheten kan i utgangspunktet styres også uten internettilgang.

På nyere enheter med den såkalte V3-protokollen godtar PortaSplit imidlertid ikke lokale kommandoer uten beskyttelse. Home Assistant trenger to enhetsspesifikke verdier, en token og en nøkkel, som brukes til autentisering og kryptering av den lokale forbindelsen. Integrasjonen henter dem én gang via et Midea-skygrensesnitt under førstegangsoppsettet og lagrer dem deretter lokalt; det er ikke nødvendig med noen skyforbindelse for videre styring.

Forenklet ser prosessen slik ut:

1. PortaSplit kobles til MSmartHome.
2. Home Assistant logger inn i en Midea-sky.
3. Home Assistant mottar enhets-ID, token og nøkkel.
4. Token og nøkkel lagres lokalt.
5. Home Assistant styrer PortaSplit direkte i LAN-et.

## Hvilken integrasjon passer

### Midea Smart AC

Repositoryet <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> fokuserer på Midea-klimaanlegg og relaterte OEM-modeller og støtter enhetstypene `0xAC` og `0xCC`. Det tilbyr lokal styring, grafisk oppsett, automatisk oppdagelse, manuelt oppsett med token og nøkkel samt automatisk forespørsel om enhetens funksjoner. PortaSplit sin «Out Silent Mode» støttes eksplisitt.

Som indikator på kompatibilitet nevner prosjektet blant annet appene Artic King, Midea Air, NetHome Plus, SmartHome eller MSmartHome, Toshiba AC NA og 美的美居. PortaSplit bruker vanligvis MSmartHome i Europa og passer derfor inn i dette økosystemet.

### Midea AC LAN

Repositoryet <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> støtter ikke bare klimaanlegg, men en rekke andre Midea-enhetsklasser: avfuktere, vifter, luftrensere, vaskemaskiner, tørketromler, oppvaskmaskiner, varmtvannsapparater, varmepumper, kjøleskap og mer, delvis også under andre merker som Carrier eller Electrolux. Det tilbyr også lokal kommunikasjon, automatisk enhetsoppdagelse og ekstra sensorer, og ifølge prosjektbeskrivelsen holder det en lengre TCP-forbindelse åpen mot enheten for å synkronisere statusendringer raskt. Det kreves minst Home Assistant 2024.4.1.

Den største ulempen er for tiden utviklerens advarsel: Sky-token-API-ene som brukes til å legge til nye enheter, avvikles gradvis. Det kan dermed bli umulig å legge til nye enheter senere.

### Anbefaling

For en ren PortaSplit-installasjon ville jeg startet med `Midea Smart AC` og kjent til `Midea AC LAN` som alternativ. `Midea Smart AC` er mer spesialisert for klimaanlegg og dokumenterer de nåværende PortaSplit-funksjonene eksplisitt.

Det er ikke fornuftig å bruke begge integrasjonene samtidig og permanent med samme enhet. Flere parallelle forbindelser fører til statusproblemer, unødvendig nettverkstrafikk og atferd som er vanskelig å spore.

## Hva integrasjonen gir

Etter oppsettet vises PortaSplit som en `climate`-entitet i Home Assistant. Avhengig av firmware og integrasjon er blant annet følgende funksjoner tilgjengelige:

- Slå av og på
- Stille inn ønsket temperatur
- Lese av aktuell romtemperatur
- Kjøling, avfukting og ren viftedrift
- Stille inn viftehastighet
- Styre swing-funksjonen
- Eco- og Boost-modus
- Lese av luftfuktighet
- Vise feilkoder
- Lese av energi- og effektverdier
- Vise kompressorverdier
- Aktivere stillemodus for utedelen

Hvilke entiteter som faktisk vises, avhenger av modellen, firmwaren, protokollen som brukes og den aktuelle integrasjonen. `Midea Smart AC` spør etter funksjonene som enheten rapporterer, og skjuler funksjoner modellen ikke støtter. `Midea AC LAN` dokumenterer også omfattende klimaentiteter, deriblant temperatur, luftfuktighet, aktuell effekt, total energi, kompressorfrekvens, pumpestatus og ulike driftsmoduser, og nevner egne metoder for dekoding av energidata for bestemte PortaSplit-undertyper.

Ikke alle viste målinger må være korrekte. Særlig energiforbruk og effekt overføres i ulike formater på forskjellige Midea-modeller. Hvis Home Assistant viser åpenbart feil verdier, må som regel dekodingsmetoden som brukes, justeres – enheten er ikke nødvendigvis defekt.

## Forutsetninger

Du trenger en Midea PortaSplit med Wi-Fi-funksjon, et 2,4 GHz-Wi-Fi-nettverk, MSmartHome-appen, en Midea-brukerkonto, Home Assistant, HACS og nettverkstilgang mellom Home Assistant og PortaSplit. PortaSplit bør først kobles til MSmartHome-appen på vanlig måte, og først deretter legges til i Home Assistant.

## Trinn 1: Koble PortaSplit til MSmartHome

1. Installer MSmartHome-appen.
2. Opprett eller logg inn på en Midea-konto.
3. Sett PortaSplit i Wi-Fi-tilkoblingsmodus.
4. Koble enheten til 2,4 GHz-Wi-Fi-nettverket.
5. Kontroller at PortaSplit kan styres via appen.

Mange IoT-enheter støtter fortsatt bare 2,4 GHz. Hvis ruteren bruker samme SSID for 2,4 og 5 GHz, fungerer oppsettet som regel likevel. Ved problemer kan det hjelpe å midlertidig sette opp et eget 2,4 GHz-Wi-Fi-nettverk.

## Trinn 2: Installer HACS

HACS er Home Assistant Community Store. Med den kan du installere community-integrasjoner som ikke er en del av Home Assistant Core. Etter HACS-installasjonen åpner du HACS, går til integrasjoner, søker etter `Midea Smart AC`, laster ned integrasjonen og starter Home Assistant på nytt. Alternativt kan du søke etter `Midea AC LAN`.

HACS forenkler installasjon og oppdateringer. Det gjør imidlertid ikke en Custom Integration til en offisielt kontrollert Home Assistant-komponent. Denne forskjellen er viktig fra et sikkerhetsperspektiv og omtales nedenfor.

## Trinn 3: Legg til Midea Smart AC

Etter omstart går du via Innstillinger, Enheter og tjenester og Legg til integrasjon til søket etter `Midea Smart AC`, og deretter til `Discover devices`. Integrasjonen kan enten søke gjennom hele det lokale nettverket eller spørre målrettet etter IP-adressen til PortaSplit.

Når enheten blir funnet, trenger integrasjonen for nyere V3-enheter region, Midea-konto, passord og enhets-ID, samt token og nøkkel avledet fra disse. Skyregionen må passe til kontoen som brukes. Ved problemer anbefaler prosjektet å prøve de andre tilgjengelige regionene også.

### Manuelt oppsett

Hvis det automatiske oppsettet mislykkes, kan enheten konfigureres manuelt. For `Midea Smart AC` kreves følgende opplysninger:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

Den dokumenterte standardporten er:

```text
6444/TCP
```

For V3-enheter oppgir dokumentasjonen token som en heksadesimal tegnstreng på 128 tegn og nøkkelen som en på 64 tegn. Begge verdiene er hemmeligheter og må behandles deretter. Hvis du ikke ønsker å hente tilgangsdataene via Discovery, kan du hente dem med din egen konto via CLI-en `msmart-ng`.

## Bruk PortaSplit på en sikker måte

Når du styrer PortaSplit lokalt, tar du tilbake deler av kontrollen fra produsentens sky, men flytter samtidig ansvaret til ditt eget nettverk. Punktene nedenfor sørger for at token og nøkkel gjør begrenset skade selv ved en feil, og at enheten forblir rent isolert.

### Token og nøkkel er hemmeligheter

Token og nøkkel autentiserer den lokale kommunikasjonen med enheten og må behandles som et passord. Det viktigste for drift er: De hører ikke hjemme i logger, ukrypterte sikkerhetskopier eller et repository.

### Ingen portvideresending til PortaSplit

Den vanligste feilen som kan unngås, ville være å gjøre den lokale enhetsporten direkte tilgjengelig fra internett. En regel som denne ville være farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finnes ingen god grunn til å gjøre PortaSplit direkte tilgjengelig fra internett. Home Assistant befinner seg allerede i det lokale nettverket og fungerer som kontrollerende instans. Ruteren bør ikke ha portvideresending til PortaSplit, UPnP bør om mulig begrenses eller deaktiveres, innkommende forbindelser bør blokkeres som standard, og ingen DMZ-frigivelse bør brukes for enheten.

### Eget IoT-VLAN

Den beste nettverksarkitekturen er et separat IoT-nettverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit befinner seg i IoT-VLAN-et. Home Assistant kan få målrettet tilgang til enheten, men PortaSplit skal ikke kunne få vilkårlig tilgang til PC-er, NAS og andre interne systemer. En mulig brannmurlogikk:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under førstegangsoppsettet trenger enheten internettilgang til Midea-skyen. Etter vellykket lokalt oppsett kan du teste om utgående internettilgang kan blokkeres. Ikke legg inn en endelig sperre med én gang. Kontroller først om den lokale styringen fortsatt fungerer, om enheten forblir tilgjengelig etter omstart, om den tåler en ruteromstart, om den fortsatt reagerer etter flere dager, om MSmartHome-appen fortsatt er nødvendig og om firmwareoppdateringer fortsatt tilbys. Hvis du vil fortsette å bruke skyen og firmwareoppdateringer, kan du tillate utgående internettilgang midlertidig og deretter blokkere den igjen.

### Nettverkssegmentering kan hindre Discovery

Automatisk enhetssøk er ofte basert på broadcast- eller multicast-trafikk, og dette rutes vanligvis ikke over VLAN-grenser. Home Assistant finner derfor kanskje ikke PortaSplit automatisk, selv om en vanlig IP-forbindelse ville vært tillatt.

Da kan du midlertidig sette opp PortaSplit i samme VLAN som Home Assistant, oppgi enhetens IP-adresse manuelt, bruke en egnet broadcast-reléfunksjon eller definere målrettede brannmurregler etter oppsettet. Manuell konfigurasjon er ofte til og med den bedre varianten fra et sikkerhetsperspektiv, fordi det ikke trenger å tillates ekstra broadcast-trafikk mellom nettverkene.

### Statisk DHCP-tilordning

PortaSplit bør få en fast DHCP-tilordning i ruteren:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservasjon er vanligvis å foretrekke fremfor en statisk IP-adresse satt på enheten. Home Assistant finner enheten pålitelig, brannmurregler kan begrenses til en fast adresse, feilsøking blir enklere, og tilordningen forblir stabil etter omstart av ruter eller enhet. En brannmurregel kan dermed formuleres svært presist:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den faktisk nødvendige porten må verifiseres ut fra integrasjonen og din egen enhet.

### Home Assistant som sentralt tillitsanker

Når du styrer PortaSplit lokalt, flytter du delvis tilliten fra Midea-skyen til Home Assistant. Hvis Home Assistant kompromitteres, kan en angriper potensielt ikke bare kontrollere klimaanlegget, men hele smarthjemmet.

Home Assistant bør derfor oppdateres regelmessig, ikke publiseres via ubeskyttet portvideresending, beskyttes med et sterkt og unikt passord, bruke flerfaktorautentisering, opprette krypterte sikkerhetskopier, bare inneholde nødvendige tillegg og ikke tillate unødvendig SSH-tilgang fra internett. For ekstern tilgang er VPN, Home Assistant Cloud eller en korrekt konfigurert reverse proxy bedre alternativer enn enkel portvideresending på port 8123.

### HACS og forsyningskjederisikoen

`Midea Smart AC` og `Midea AC LAN` er Custom Integrations. De kjører i Home Assistant og får dermed omfattende tilgang til kjøretidsmiljøet. En ondsinnet eller kompromittert integrasjon kan teoretisk lese konfigurasjonsdata, hente ut hemmeligheter, opprette nettverksforbindelser, skanne enheter i det lokale nettverket, lese tilstander for andre entiteter, overføre data til eksterne systemer og påvirke tilgjengeligheten til Home Assistant.

Dette betyr ikke at de nevnte integrasjonene er ondsinnede. Begge prosjektene er offentlig tilgjengelige, utvikles aktivt og har et synlig community. Open source er imidlertid ingen automatisk sikkerhetsgaranti. Før installasjon bør du i det minste se på om repositoryet vedlikeholdes aktivt, om det finnes regelmessige utgivelser, hvor mange personer som bidrar til koden, om det finnes åpne sikkerhetssaker, om maintainere eller repository-eiere nylig har blitt byttet, om HACS peker til det forventede repositoryet og om en oppdatering inneholder uvanlig store eller uforklarlige endringer.

Oppdateringer bør ikke installeres blindt umiddelbart etter publisering. Særlig for sikkerhetskritiske smarthjemsystemer er det fornuftig å vente noen dager og kontrollere utgivelsesnotater og rapporterte problemer.

### Sikre skykontoen

Så lenge Midea-skyen brukes til oppsett eller appstyring, forblir også Midea-kontoen en del av sikkerhetsmodellen. Den bør ha et unikt passord som ikke deles med andre tjenester, en passordbehandler, flerfaktorautentisering dersom det tilbys, fjerning av gamle smarttelefoner og økter, ingen delte kontoer og regelmessig kontroll av hvilke enheter som er registrert på kontoen.

Hvis Home Assistant-integrasjonen ber om brukernavn og passord under oppsettet, bør du undersøke om tilgangsdataene bare brukes til engangshenting av token eller lagres permanent. Utviklerne av `Midea Smart AC` skriver at enheter etter oppsett ikke knyttes til innebygde integrasjonskontoer, og at token og nøkkel også kan hentes manuelt via din egen konto med CLI. Der det er mulig, bør din egen konto foretrekkes fremfor fremmede eller integrerte felleskontoer.

### Blokkere skyen eller ikke?

Etter vellykket oppsett oppstår spørsmålet om internettilgangen til PortaSplit bør blokkeres fullstendig. Argumenter for blokkering er mindre telemetri, mindre avhengighet av eksterne tjenester, en mindre angrepsflate via produsentens sky, at enheten ikke kan kontakte vilkårlige eksterne mål og redusert effekt av endringer på skysiden.

Imot taler at MSmartHome-appen kanskje ikke lenger fungerer utenfor hjemmenettverket, at firmwareoppdateringer ikke lastes ned, at klokke- eller skyfunksjoner kan falle bort, at ny innlogging eller gjenoppretting blir vanskeligere og at enkelte enheter reagerer uventet etter lengre tid uten nett.

En pragmatisk rekkefølge: Sett opp enheten normalt, test Home Assistant og appen, sikkerhetskopier token og konfigurasjon, blokker internettilgang, start enheten og Home Assistant på nytt, observer i flere dager og gi ved behov bare midlertidig internettilgang igjen.

### Firmwareoppdateringer: sikkerhetsgevinst eller integrasjonsrisiko?

Firmwareoppdateringer er et dilemma for IoT-enheter. De kan lukke kjente sårbarheter, forbedre stabiliteten, modernisere sikkerhetsmekanismer og gi nye funksjoner. Men de kan også endre lokale grensesnitt, ødelegge reverse-engineering-integrasjoner, gjøre token ugyldige, deaktivere lokal API og innføre nye skyavhengigheter.

PortaSplit-firmwaren som ble levert i januar 2026, ga for eksempel en ny stillemodus for utedelen som reduserer støynivået med rundt 6 desibel. Community-integrasjonene måtte først forstå og implementere den, dokumentert i en egen GitHub-sak for PortaSplit.

Det følger av dette: Ikke hindre firmwareoppdateringer generelt, men kontroller før en oppdatering om andre Home Assistant-brukere rapporterer problemer, sikkerhetskopier konfigurasjon og token på forhånd, lag en Home Assistant-sikkerhetskopi og test den lokale styringen fullstendig etter oppdateringen. Sikkerhet betyr ikke «aldri oppdatere». Utdatert firmware kan være farligere enn en midlertidig inkompatibel integrasjon.

### Debug-logger inneholder sensitive data

Ved problemer ber open-source-prosjekter ofte om debug-logger. Dokumentasjonen for `Midea AC LAN` viser hvordan logging for de to relevante komponentene aktiveres:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Deretter kan du laste ned loggene via Innstillinger, System og Logger. Slike logger kan avhengig av integrasjon og feilsituasjon inneholde lokale IP-adresser, enhets-ID, serienummer, modellidentifikasjon, skysvar, kontoinformasjon, token eller deler av dem, nettverkspakker samt tidsstempler og bruksmønster. Før opplasting til en offentlig GitHub-sak må de derfor kontrolleres og sensitive verdier sladdes.

Etter at feilsøkingen er avsluttet, bør debug-logging fjernes igjen. Permanently aktivert debug-logging øker ikke bare lagringsforbruket, men også mengden sensitiv informasjon i sikkerhetskopiene.

### Hva Midea selv sier om sikkerhet

Midea markedsfører SmartHome-økosystemet sitt med orientering mot flere standarder for sikkerhet og personvern, blant annet EN 303 645, UK PSTI, NIST, DSGVO-kompatibel databehandling og kravene i EU Radio Equipment Directive. Dette er positive signaler, men ikke en uttalelse om hvordan hver enkelt PortaSplit-firmware, hvert skyendepunkt og hvert lokale API faktisk er implementert. Sertifiserings- og markedsføringspåstander erstatter ikke teknisk kontroll av den konkrete enheten.

Det ville på samme måte være feil å utlede fra advarselen fra en community-integrasjon at PortaSplit generelt er usikker. Problemet som beskrives, gjelder arkitekturen til langvarige token og bruken av dem av uoffisielle klienter.

### Risiko etter scenario

| Scenario | Risiko | Begrunnelse |
| --- | --- | --- |
| Vanlig hjemmenettverk uten portvideresending | oversiktlig | En angriper må først få tilgang til Wi-Fi, Home Assistant eller en sikkerhetskopi. |
| Flatt hjemmenettverk med mange usikre IoT-enheter | middels | En kompromittert annen IoT-enhet kan nå PortaSplit eller Home Assistant i samme nettverk. |
| PortaSplit direkte tilgjengelig fra internett | høy | Enheten må aldri publiseres via portvideresending. |
| Token og nøkkel offentlig på GitHub | høy | Hemmelighetene anses som kompromittert; det er ikke garantert at de kan tilbakekalles. |
| Separat IoT-VLAN, restriktiv brannmur, lokal styring | forholdsvis lav | Selv ved en sårbarhet i enheten er bevegelsesfriheten i nettverket sterkt begrenset. |

## Sikkerhetskopi av konfigurasjonen

Sikkerhetskopiering av token, nøkkel og konfigurasjon er det viktigste engangstrinnet: Når sky-token-grensesnittene først er stengt, er en sikkerhetskopi den eneste veien til et nytt oppsett. `Midea AC LAN` lagrer en JSON-konfigurasjonsfil for V3-enheter etter vellykket oppsett. Den dokumenterte stien er:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID-en som filnavn:

```text
<device-id>.json
```

Denne filen er ikke et vanlig tekstdokument. Den kan inneholde enhets-ID, serienummer, IP-adresse, token, nøkkel, protokollinformasjon samt sky- og enhetsparametere. Følgende gjelder derfor:

- Ikke last den opp til et offentlig GitHub-repository.
- Ikke publiser den i forum.
- Ikke del den som et usladdet skjermbilde.
- Ikke send den via ukryptert e-post.

Selv et privat Git-repository er ikke automatisk riktig lagringssted, fordi hemmeligheter forblir i Git-historikken selv om de senere slettes fra den gjeldende filen. Bedre alternativer er en kryptert sikkerhetskopi, en passordbehandler med filvedlegg, en kryptert NAS-sikkerhetskopi, et kryptert frakoblet medium eller et kryptert arkiv med passordet lagret separat.

For sikkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Vis filen:

```bash
cat <device-id>.json
```

Ved kopiering bør filen ikke overføres via en offentlig nettjeneste. Et kryptert arkiv som deretter overføres til en kryptert sikkerhetskopi, er bedre:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filene i `.storage` bør ikke redigeres manuelt. Utvikleren anbefaler uttrykkelig at JSON-filen verken slettes eller endres direkte ved problemer, men at den gis nytt navn og sikkerhetskopieres før endringer.

En fullstendig Home Assistant-sikkerhetskopi inneholder også disse filene. En separat kopi er likevel fornuftig, fordi Home Assistant-sikkerhetskopier kan bli skadet, en gjenoppretting kan overskrive integrasjonen, filen kan være nødvendig spesifikt for et senere nytt oppsett, og en sikkerhetskopi aldri bare bør ligge på samme system.

## Fjern hemmeligheter fra et publisert Git-repository

Hvis en JSON-fil ved et uhell ble publisert på GitHub, er det ikke nok med vanlig sletting og en ny commit. Filen forblir tilgjengelig i Git-historikken. Minst disse trinnene er nødvendige:

1. Gjør repositoryet privat umiddelbart, hvis mulig.
2. Fjern filen fra hele Git-historikken.
3. Ta hensyn til GitHub-cacher og forks.
4. Behandle token som kompromittert.
5. Fjern enheten fra Midea-kontoen og koble den til på nytt dersom dette genererer nye nøkler.
6. Sett opp Home Assistant-integrasjonen på nytt.
7. Endre Midea-kontopassordet dersom tilgangsdataene også var berørt.

Om ny paring faktisk genererer en ny token, varierer etter enhet og skyarkitektur. Du bør ikke stole på at endring av kontopassord automatisk gjør den lokale enhetstokenen ugyldig.

## Nyttige automasjoner

Etter vellykket integrasjon kan PortaSplit drives betydelig smartere. Entity-ID-ene må tilpasses din egen installasjon.

Kjøl bare når vinduene er lukket:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Slå på ved høy romtemperatur:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Forkjøl før leggetid:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Slå av når ingen er hjemme:

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Anbefalt konfigurasjon i oversikt

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

Den ønskede kommunikasjonsretningen ser dermed slik ut:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

## Anbefalt driftstilstand

Midea PortaSplit kan integreres godt i Home Assistant. Etter vellykket oppsett kan den styres lokalt og brukes i automasjoner, noe som fjerner en stor del av skyavhengigheten i den daglige driften.

Fra et sikkerhetsperspektiv er integrasjonen forsvarlig når noen grunnregler følges: ingen portvideresending, hold token og nøkkel hemmelige, krypter sikkerhetskopier, kontroller debug-logger før publisering, sikre Home Assistant, segmenter IoT-enheter, begrens utgående internettilgang til det nødvendige og ikke installer firmware- eller HACS-oppdateringer blindt. Med denne driften forblir PortaSplit et kraftig klimaanlegg og blir samtidig en fornuftig integrerbar del av et lokalt styrt smarthjem.

## Kilder

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjonen `Midea Smart AC`: støttede enhetstyper `0xAC` og `0xCC`, PortaSplit med «Out Silent Mode», skybruk for å hente token og nøkkel på V3-enheter, manuell konfigurasjon og standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjonen `Midea AC LAN`: støttede enhetsklasser, lengre TCP-forbindelse for statussynkronisering og minimumsversjon Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentasjon av klimaentitetene](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): Entiteter og attributter for klimaanlegg, blant annet effekt, total energi, kompressorfrekvens og dekodingsmetoder for energiverdier for enkelte undertyper.

4.  [midea_ac_lan: Merknader om debug og konfigurasjon](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): Lagring av enhetskonfigurasjonen under `/config/.storage/midea_ac_lan/`, anbefaling om å sikkerhetskopiere JSON-filen i stedet for å slette den, og loggerkonfigurasjonen for debug-logger.

5.  [Issue 779: Out Silent Mode for PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): Forespørsel om støtte for stillemodusen for utedelen som ble innført med firmwareoppdateringen i januar 2026, og som reduserer støynivået med rundt 6 desibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om standardene for sikkerhet og personvern EN 303 645, PSTI, NIST, DSGVO og RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installasjon og administrasjon av Custom Integrations som ikke er en del av Home Assistant Core.
