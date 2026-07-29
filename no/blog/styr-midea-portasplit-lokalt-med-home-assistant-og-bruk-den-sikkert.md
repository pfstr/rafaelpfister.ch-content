---
title: "Styr Midea PortaSplit lokalt med Home Assistant og bruk den sikkert"
navTitle: "Sett opp PortaSplit"
description: "Fra riktig fellesskapsintegrasjon til IoT-VLAN: Slik setter du opp PortaSplit, sikrer token og nøkkel og begrenser sky- og nettverkstilgang."
date: "2026-07-24"
kategorie: "Smarthus og IoT"
timeToRead: "14 min lesetid"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert"
translationOf: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/no/blog/styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert"
---

Midea PortaSplit kan styres direkte i det lokale nettverket via Home Assistant etter oppsettet. Til dette trenger fellesskapsintegrasjonen to enhetsspesifikke tilgangsverdier fra Midea-skyen: token og nøkkel.

Denne artikkelen går gjennom valg, oppsett og sikring av integrasjonen. Løsningene som beskrives, kommer fra fellesskapet og støttes ikke offisielt av verken Midea eller Home Assistant. Endringer i fastvare eller skyen kan derfor når som helst påvirke hvordan de fungerer. Bakgrunnen for token-grensesnittet og den tvetydige advarselen om avvikling finnes i [analysen av Midea-sky-API-ene](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Slik fungerer lokal styring

De faktiske styringskommandoene går direkte fra Home Assistant til PortaSplit etter oppsettet:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

En bryterkommando trenger ikke å gå via en ekstern Midea-server, responstiden er kort, en feil i Midea-skyen lammer ikke nødvendigvis den allerede oppsatte lokale styringen, og enheten kan i utgangspunktet fortsatt styres uten internettilgang.

På nyere enheter med den såkalte V3-protokollen godtar PortaSplit imidlertid ikke lokale kommandoer uten beskyttelse. Home Assistant trenger to enhetsspesifikke verdier, et token og en nøkkel, som brukes til autentisering og kryptering av den lokale forbindelsen. Integrasjonen henter dem én gang via et Midea-skygrensesnitt under det første oppsettet og lagrer dem deretter lokalt; det kreves ingen skytilkobling for videre styring.

Forenklet ser prosessen slik ut:

1. PortaSplit kobles til MSmartHome.
2. Home Assistant logger inn i en Midea-sky.
3. Home Assistant mottar enhets-ID, token og nøkkel.
4. Token og nøkkel lagres lokalt.
5. Home Assistant styrer PortaSplit direkte i LAN-et.

## Hvilken integrasjon passer

### Midea Smart AC

Repositoryet <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> fokuserer på Midea-klimaanlegg og relaterte OEM-modeller og støtter enhetstypene `0xAC` og `0xCC`. Det tilbyr lokal styring, grafisk oppsett, automatisk oppdagelse, manuelt oppsett med token og nøkkel samt automatisk henting av enhetskapasiteter. PortaSplit sin «Out Silent Mode» støttes eksplisitt.

Prosjektet nevner blant annet appene Artic King, Midea Air, NetHome Plus, SmartHome eller MSmartHome, Toshiba AC NA og 美的美居 som indikasjon på kompatibilitet. PortaSplit bruker vanligvis MSmartHome i Europa og passer dermed inn i dette økosystemet.

### Midea AC LAN

Repositoryet <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> støtter ikke bare klimaanlegg, men en rekke andre Midea-enhetsklasser: avfuktere, vifter, luftrensere, vaskemaskiner, tørketromler, oppvaskmaskiner, varmtvannsenheter, varmepumper, kjøleskap og mer, delvis også under andre merker som Carrier eller Electrolux. Det tilbyr også lokal kommunikasjon, automatisk enhetsoppdagelse og ekstra sensorer, og holder ifølge prosjektbeskrivelsen en lengre TCP-forbindelse åpen til enheten for å synkronisere statusendringer raskt. Home Assistant 2024.4.1 eller nyere kreves.

Den største ulempen er for tiden utviklerens advarsel: Sky-token-API-ene som brukes til å legge til nye enheter, avvikles gradvis. Det kan dermed bli umulig å legge til nye enheter senere.

### Anbefaling

For en ren PortaSplit-installasjon ville jeg startet med `Midea Smart AC` og hatt `Midea AC LAN` i bakhodet som alternativ. `Midea Smart AC` er mer spesialisert for klimaanlegg og dokumenterer de nåværende PortaSplit-funksjonene eksplisitt.

Det er ikke fornuftig å bruke begge integrasjonene samtidig og permanent med samme enhet. Flere parallelle forbindelser fører til statusproblemer, unødvendig nettverkstrafikk og atferd som er vanskelig å forstå.

## Hva integrasjonen gir

Etter oppsettet vises PortaSplit som en `climate`-entitet i Home Assistant. Avhengig av fastvare og integrasjon er blant annet følgende funksjoner tilgjengelige:

- Slå på og av
- Stille inn ønsket temperatur
- Lese av aktuell romtemperatur
- Kjøling, avfukting og ren viftedrift
- Stille inn viftehastighet
- Styre swing-funksjonen
- Eco- og boost-modus
- Lese av luftfuktighet
- Vise feilkoder
- Lese av energi- og effektverdier
- Vise kompressorverdier
- Aktivere stillemodus for utedelen

Hvilke entiteter som faktisk vises, avhenger av modellen, fastvaren, protokollen som brukes, og den aktuelle integrasjonen. `Midea Smart AC` henter funksjonene enheten rapporterer og skjuler funksjoner som modellen ikke støtter. `Midea AC LAN` dokumenterer også omfattende klimaentiteter, blant annet temperatur, luftfuktighet, aktuell effekt, total energi, kompressorfrekvens, pumpestatus og ulike driftsmoduser, og nevner egne metoder for dekoding av energidata for bestemte PortaSplit-undertyper.

Ikke alle viste målinger trenger å være korrekte. Særlig energiforbruk og effekt overføres i ulike formater på forskjellige Midea-modeller. Dersom Home Assistant viser åpenbart feil verdier, må den brukte dekodingsmetoden som regel tilpasses, og enheten er ikke nødvendigvis defekt.

## Forutsetninger

Du trenger en Midea PortaSplit med Wi-Fi-funksjon, et 2,4 GHz-Wi-Fi, MSmartHome-appen, en Midea-brukerkonto, Home Assistant, HACS og nettverkstilgang mellom Home Assistant og PortaSplit. PortaSplit bør først kobles til MSmartHome-appen på vanlig måte, og deretter legges til i Home Assistant.

## Trinn 1: Koble PortaSplit til MSmartHome

1. Installer MSmartHome-appen.
2. Opprett en Midea-konto eller logg inn.
3. Sett PortaSplit i Wi-Fi-paringsmodus.
4. Koble enheten til 2,4 GHz-Wi-Fi.
5. Kontroller at PortaSplit kan styres via appen.

Mange IoT-enheter støtter fortsatt bare 2,4 GHz. Dersom ruteren bruker samme SSID for 2,4 og 5 GHz, fungerer oppsettet som regel likevel. Ved problemer kan det hjelpe å midlertidig tilby et separat 2,4 GHz-Wi-Fi.

## Trinn 2: Installer HACS

HACS er Home Assistant Community Store. Med det kan du installere fellesskapsintegrasjoner som ikke er en del av Home Assistant Core. Etter HACS-installasjonen åpner du HACS, går til integrasjoner, søker etter `Midea Smart AC`, laster ned integrasjonen og starter Home Assistant på nytt. Alternativt kan du søke etter `Midea AC LAN`.

HACS forenkler installasjon og oppdateringer. Det gjør imidlertid ikke en Custom Integration til en offisielt kontrollert Home Assistant-komponent. Denne forskjellen er vesentlig fra et sikkerhetsperspektiv og omtales nedenfor.

## Trinn 3: Legg til Midea Smart AC

Etter omstart går du via Innstillinger, Enheter og tjenester og Legg til integrasjon til søket etter `Midea Smart AC`, og deretter til `Discover devices`. Integrasjonen kan enten søke gjennom hele det lokale nettverket eller spørre målrettet etter IP-adressen til PortaSplit.

Hvis enheten blir funnet, trenger integrasjonen region, Midea-konto, passord og enhets-ID samt tokenet og nøkkelen som avledes fra disse for nyere V3-enheter. Skyregionen må passe til kontoen som brukes. Ved problemer anbefaler prosjektet å prøve de andre tilgjengelige regionene også.

### Manuelt oppsett

Hvis automatisk oppsett mislykkes, kan enheten konfigureres manuelt. For `Midea Smart AC` kreves følgende opplysninger:

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

For V3-enheter oppgir dokumentasjonen tokenet som en heksadesimal streng på 128 tegn og nøkkelen som en heksadesimal streng på 64 tegn. Begge verdiene er hemmeligheter og må behandles deretter. De som ikke vil hente tilgangsdataene via oppdagelse, kan hente dem med sin egen konto via CLI-en `msmart-ng`.

## Bruk PortaSplit sikkert

Når du styrer PortaSplit lokalt, henter du en del av kontrollen tilbake fra produsentens sky, men flytter samtidig ansvaret til ditt eget nettverk. Følgende punkter sørger for at token og nøkkel gjør liten skade selv ved en hendelse, og at enheten holdes godt isolert.

### Token og nøkkel er hemmeligheter

Token og nøkkel autentiserer den lokale kommunikasjonen med enheten og må behandles som et passord. For drift er det viktigste: De hører ikke hjemme i logger, ukrypterte sikkerhetskopier eller et repository.

### Ingen portvideresending til PortaSplit

Den vanligste feilen som kan unngås, ville være å gjøre den lokale enhetsporten direkte tilgjengelig fra internett. En regel som denne ville vært farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finnes ingen god grunn til å gjøre PortaSplit direkte tilgjengelig fra internett. Home Assistant befinner seg allerede i det lokale nettverket og fungerer som den kontrollerende instansen. Ruteren bør ikke ha portvideresending til PortaSplit, UPnP bør begrenses eller deaktiveres der det er mulig, innkommende forbindelser bør blokkeres som standard, og det bør ikke brukes DMZ-frigivelse for enheten.

### Eget IoT-VLAN

Den beste nettverksarkitekturen er et separat IoT-nettverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit befinner seg i IoT-VLAN-et. Home Assistant kan få målrettet tilgang til enheten, men PortaSplit skal ikke fritt kunne få tilgang til PC-er, NAS og andre interne systemer. En mulig brannmurlogikk:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under første oppsett trenger enheten internettilgang til Midea-skyen. Etter vellykket lokalt oppsett kan du teste om utgående internettilgang kan blokkeres. Du bør ikke sette en permanent blokkering umiddelbart. Kontroller først om lokal styring fortsatt fungerer, om enheten er tilgjengelig etter omstart, om den tåler omstart av ruteren, om den fortsatt reagerer etter flere dager, om MSmartHome-appen fortsatt er nødvendig og om fastvareoppdateringer fortsatt tilbys. Hvis du vil fortsette å bruke skyen og fastvareoppdateringer, kan du midlertidig tillate utgående internettilgang og deretter blokkere den igjen.

### Nettverkssegmentering kan hindre oppdagelse

Automatisk enhetsoppdagelse baserer seg ofte på broadcast- eller multicast-trafikk, og denne rutes normalt ikke over VLAN-grenser. Home Assistant kan derfor unnlate å finne PortaSplit automatisk, selv om en vanlig IP-forbindelse ville vært tillatt.

Da kan det hjelpe å sette opp PortaSplit midlertidig i samme VLAN som Home Assistant, oppgi enhetens IP-adresse manuelt, bruke en egnet broadcast-reléfunksjon eller definere målrettede brannmurregler etter oppsettet. Manuell konfigurasjon er ofte til og med det bedre alternativet fra et sikkerhetsperspektiv, fordi det ikke krever at ekstra broadcast-trafikk tillates mellom nettverkene.

### Statisk DHCP-tilordning

PortaSplit bør få en fast DHCP-tilordning i ruteren:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservasjon er som regel å foretrekke fremfor en statisk IP satt på enheten. Home Assistant finner enheten pålitelig, brannmurregler kan begrenses til en fast adresse, feilsøking blir enklere, og tilordningen forblir stabil etter omstart av ruter eller enhet. En brannmurregel kan dermed formuleres svært strengt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den faktiske nødvendige porten må verifiseres ut fra integrasjonen og din egen enhet.

### Home Assistant som sentralt tillitsanker

Når du styrer PortaSplit lokalt, flytter du deler av tilliten fra Midea-skyen til Home Assistant. Hvis Home Assistant kompromitteres, kan en angriper i enkelte tilfeller ikke bare kontrollere klimaanlegget, men hele smarthuset.

Home Assistant bør derfor oppdateres regelmessig, ikke publiseres via ubeskyttet portvideresending, beskyttes med et sterkt, unikt passord, bruke flerfaktorautentisering, lage krypterte sikkerhetskopier, bare inneholde nødvendige tillegg og ikke tillate unødvendig SSH-tilgang fra internett. For fjerntilgang er VPN, Home Assistant Cloud eller en korrekt konfigurert reverse proxy bedre alternativer enn enkel portvideresending på port 8123.

### HACS og risikoen i forsyningskjeden

`Midea Smart AC` og `Midea AC LAN` er Custom Integrations. De kjører i Home Assistant og får dermed omfattende tilgang til kjøretidsmiljøet. En ondsinnet eller kompromittert integrasjon kan teoretisk lese konfigurasjonsdata, hente ut hemmeligheter, opprette nettverksforbindelser, skanne enheter i det lokale nettverket, lese tilstander til andre entiteter, overføre data til eksterne systemer og påvirke tilgjengeligheten til Home Assistant.

Det betyr ikke at de nevnte integrasjonene er ondsinnede. Begge prosjektene er offentlig tilgjengelige, utvikles aktivt og har et synlig fellesskap. Open source er imidlertid ingen automatisk sikkerhetsgaranti. Før installasjon er det verdt å se minst på om repositoryet vedlikeholdes aktivt, om det kommer regelmessige utgivelser, hvor mange personer som bidrar til koden, om det finnes åpne sikkerhetssaker, om maintainere eller repository-eiere nylig har byttet, om HACS peker til forventet repository, og om en oppdatering inneholder uvanlig store eller uforklarlige endringer.

Oppdateringer bør ikke installeres blindt umiddelbart etter publisering. Særlig for sikkerhetskritiske smarthussystemer er det fornuftig å vente noen dager og kontrollere utgivelsesnotater og rapporterte problemer.

### Sikre skykontoen

Så lenge Midea-skyen brukes til oppsett eller appstyring, forblir Midea-kontoen en del av sikkerhetsmodellen. Den bør ha et unikt passord som ikke deles med andre tjenester, en passordbehandler, flerfaktorautentisering der det tilbys, fjerning av gamle smarttelefoner og økter, ingen delte kontoer og regelmessig kontroll av hvilke enheter som er registrert på kontoen.

Hvis Home Assistant-integrasjonen ber om brukernavn og passord under oppsettet, bør du kontrollere om tilgangsdataene bare brukes til engangshenting av tokenet eller lagres permanent. Utviklerne av `Midea Smart AC` skriver at enheter etter oppsettet ikke kobles til innebygde integrasjonskontoer, og at token og nøkkel også kan hentes manuelt via CLI med egen konto. Der det er mulig, bør egen konto foretrekkes fremfor fremmede eller integrerte samlekontoer.

### Blokkere skyen eller ikke?

Etter vellykket oppsett oppstår spørsmålet om internettilgangen til PortaSplit bør blokkeres fullstendig. Argumenter for blokkering er mindre telemetri, mindre avhengighet av eksterne tjenester, en mindre angrepsflate via produsentens sky, at enheten ikke kan kontakte vilkårlige eksterne mål, og mindre påvirkning fra endringer på skysiden.

Imot taler at MSmartHome-appen kanskje ikke lenger fungerer utenfor hjemmenettverket, at fastvareoppdateringer ikke lastes ned, at klokke- eller skyfunksjoner kan slutte å fungere, at ny innlogging eller gjenoppretting blir vanskeligere, og at enkelte enheter reagerer uventet etter lang tid uten nett.

En pragmatisk rekkefølge: Sett opp enheten normalt, test Home Assistant og appen, sikkerhetskopier token og konfigurasjon, blokker internettilgang, start enheten og Home Assistant på nytt, observer i flere dager og gi om nødvendig bare midlertidig tilgang til internett igjen.

### Fastvareoppdateringer: sikkerhetsgevinst eller integrasjonsrisiko?

Fastvareoppdateringer er et dilemma for IoT-enheter. De kan lukke kjente sårbarheter, forbedre stabiliteten, modernisere sikkerhetsmekanismer og gi nye funksjoner. Men de kan også endre lokale grensesnitt, bryte integrasjoner basert på reverse engineering, gjøre token ugyldige, deaktivere det lokale API-et og innføre nye skyavhengigheter.

PortaSplit-fastvaren som ble levert i januar 2026, fikk for eksempel en ny stillemodus for utedelen som reduserer støynivået med rundt 6 desibel. Denne måtte først analyseres og implementeres av fellesskapsintegrasjonene, dokumentert i en egen GitHub-sak for PortaSplit.

Følgende gjelder derfor: Ikke hindre fastvareoppdateringer generelt, men kontroller før en oppdatering om andre Home Assistant-brukere rapporterer problemer, sikkerhetskopier konfigurasjon og token på forhånd, lag en Home Assistant-sikkerhetskopi og test den lokale styringen fullstendig etter oppdateringen. Sikkerhet betyr ikke «aldri oppdatere». Utdatert fastvare kan være farligere enn en midlertidig inkompatibel integrasjon.

### Debug-logger inneholder sensitive data

Ved problemer ber open-source-prosjekter ofte om debug-logger. Dokumentasjonen for `Midea AC LAN` viser hvordan logging aktiveres for de to relevante komponentene:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Deretter kan loggene lastes ned via Innstillinger, System og Logger. Slike logger kan, avhengig av integrasjon og feiltilfelle, inneholde lokale IP-adresser, enhets-ID, serienummer, modellidentifikator, skyresponser, kontoinformasjon, token eller deler av det, nettverkspakker samt tidsstempler og bruksmønster. Før opplasting til en offentlig GitHub-sak bør de derfor gjennomgås og sensitive verdier sladdes.

Når feilsøkingen er avsluttet, bør debug-loggingen fjernes igjen. Permanent aktivert debug-logging øker ikke bare lagringsforbruket, men også mengden sensitiv informasjon i sikkerhetskopiene.

### Hva Midea selv sier om sikkerhet

Midea markedsfører SmartHome-økosystemet sitt med orientering mot flere sikkerhets- og personvernstandarder, blant annet EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling og kravene i EU Radio Equipment Directive. Dette er positive signaler, men sier ikke noe om hvordan hver enkelt PortaSplit-fastvare, hvert skyendepunkt og hvert lokale API faktisk er implementert. Sertifiserings- og markedsføringspåstander erstatter ikke teknisk kontroll av den konkrete enheten.

På samme måte ville det være feil å utlede av advarselen fra en fellesskapsintegrasjon at PortaSplit generelt er usikker. Problemet som beskrives, gjelder arkitekturen for langvarige token og deres bruk av uoffisielle klienter.

### Risiko etter scenario

| Scenario | Risiko | Begrunnelse |
| --- | --- | --- |
| Normalt hjemmenettverk uten portvideresending | oversiktlig | En angriper må først få tilgang til Wi-Fi, Home Assistant eller en sikkerhetskopi. |
| Flatt hjemmenettverk med mange usikre IoT-enheter | middels | En kompromittert annen IoT-enhet kan nå PortaSplit eller Home Assistant i samme nettverk. |
| PortaSplit direkte tilgjengelig fra internett | høy | Enheten bør aldri publiseres med portvideresending. |
| Token og nøkkel offentlig på GitHub | høy | Hemmelighetene regnes som kompromittert; det er ikke garantert at de kan tilbakekalles. |
| Separat IoT-VLAN, restriktiv brannmur, lokal styring | forholdsvis lav | Selv ved en sårbarhet i enheten er bevegelsesfriheten i nettverket sterkt begrenset. |

## Sikkerhetskopi av konfigurasjonen

Sikkerhetskopiering av token, nøkkel og konfigurasjon er det viktigste engangstrinnet: Når sky-token-grensesnittene først er stengt, er en sikkerhetskopi den eneste veien til et nytt oppsett. `Midea AC LAN` lagrer en JSON-konfigurasjonsfil for V3-enheter etter vellykket oppsett. Den dokumenterte banen er:

```text
/config/.storage/midea_ac_lan/
```

Filen bruker enhets-ID-en som filnavn:

```text
<device-id>.json
```

Denne filen er ikke et vanlig tekstnotat. Den kan inneholde enhets-ID, serienummer, IP-adresse, token, nøkkel, protokollinformasjon samt sky- og enhetsparametere. Følgende gjelder derfor:

- Ikke last den opp til et offentlig GitHub-repository.
- Ikke legg den ut på forum.
- Ikke del den som et usladdet skjermbilde.
- Ikke send den via ukryptert e-post.

Selv et privat Git-repository er ikke automatisk riktig lagringssted, fordi hemmeligheter blir værende i Git-historikken selv om de senere slettes fra den aktuelle filen. Bedre alternativer er en kryptert sikkerhetskopi, en passordbehandler med filvedlegg, en kryptert NAS-sikkerhetskopi, et kryptert frakoblet medium eller et kryptert arkiv med passordet lagret separat.

For sikkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Vis fil:

```bash
cat <enhets-id>.json
```

Ved kopiering bør filen ikke overføres via en offentlig nettjeneste. Et kryptert arkiv som deretter legges inn i en kryptert sikkerhetskopi, er bedre:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filene i `.storage` bør ikke redigeres manuelt. Utvikleren anbefaler uttrykkelig å verken slette eller endre JSON-filen direkte ved problemer, men å gi den nytt navn og sikkerhetskopiere den før endringer.

En fullstendig Home Assistant-sikkerhetskopi inneholder også disse filene. En separat kopi er likevel fornuftig, fordi Home Assistant-sikkerhetskopier kan bli skadet, en gjenoppretting kan overskrive integrasjonen, filen kan være nødvendig spesifikt for et senere nytt oppsett, og en sikkerhetskopi aldri bare bør ligge på samme system.

## Fjern hemmeligheter fra et publisert Git-repository

Hvis en JSON-fil ved et uhell er publisert på GitHub, er det ikke nok å slette den normalt og lage en ny commit. Filen forblir tilgjengelig i Git-historikken. Minst disse trinnene er nødvendige:

1. Gjør repositoryet privat umiddelbart, dersom mulig.
2. Fjern filen fra hele Git-historikken.
3. Ta hensyn til GitHub-cacher og forks.
4. Behandle tokenet som kompromittert.
5. Fjern enheten fra Midea-kontoen og koble den til på nytt dersom det genererer nye nøkler.
6. Sett opp Home Assistant-integrasjonen på nytt.
7. Endre passordet for Midea-kontoen hvis tilgangsdataene også var berørt.

Om ny paring faktisk oppretter et nytt token, varierer avhengig av enhet og skyarkitektur. Du bør ikke stole på at endring av kontopassordet automatisk gjør det lokale enhetstokenet ugyldig.

## Nyttige automatiseringer

Etter vellykket integrasjon kan PortaSplit drives langt smartere. Entity-ID-ene må tilpasses din egen installasjon.

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

Midea PortaSplit kan integreres overraskende godt med Home Assistant. Etter vellykket oppsett kan den styres lokalt og brukes i automatiseringer, slik at en stor del av skyavhengigheten faller bort i den daglige driften.

Fra et sikkerhetsperspektiv er integrasjonen forsvarlig hvis noen grunnregler følges: ingen portvideresending, hold token og nøkkel hemmelige, krypter sikkerhetskopier, kontroller debug-logger før publisering, sikre Home Assistant, segmenter IoT-enheter, begrens utgående internettilgang til det nødvendige og ikke installer fastvare- eller HACS-oppdateringer blindt. Brukt slik forblir PortaSplit et kraftig klimaanlegg og blir samtidig en fornuftig integrerbar del av et lokalt styrt smarthus.

## Kilder

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjon `Midea Smart AC`: støttede enhetstyper `0xAC` og `0xCC`, PortaSplit med «Out Silent Mode», skybruk for innhenting av token og nøkkel på V3-enheter, manuell konfigurasjon og standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjon `Midea AC LAN`: støttede enhetsklasser, lengre TCP-forbindelse for statussynkronisering og minimumsversjon Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentasjon av klimaentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entiteter og attributter for klimaanlegg, blant annet effekt, total energi, kompressorfrekvens og dekodingsmetoder for energiverdier for enkelte undertyper.

4.  [midea_ac_lan: Feilsøkings- og konfigurasjonsmerknader](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): lagring av enhetskonfigurasjon under `/config/.storage/midea_ac_lan/`, anbefaling om å sikkerhetskopiere i stedet for å slette JSON-filen og loggerkonfigurasjonen for debug-logger.

5.  [Issue 779: PortaSplit sin Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): forespørsel om støtte for stillemodus for utedelen som ble innført med fastvareoppdateringen fra januar 2026, og som reduserer støynivået med rundt 6 desibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): produsentopplysninger om sikkerhets- og personvernstandardene EN 303 645, PSTI, NIST, GDPR og RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installasjon og administrasjon av Custom Integrations som ikke er en del av Home Assistant Core.
