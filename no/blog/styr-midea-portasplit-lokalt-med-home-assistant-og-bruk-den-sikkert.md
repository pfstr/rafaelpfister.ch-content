---
title: "Styr Midea PortaSplit lokalt med Home Assistant og bruk den sikkert"
navTitle: "Sett opp PortaSplit"
description: "Fra riktig fellesskapsintegrasjon til IoT-VLAN: Slik setter du opp PortaSplit, sikrer token og nøkkel og begrenser sky- og nettverkstilgang."
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
translationSourceHash: 859c24ec38af3b4b931702c7be50cf2224580d30045559ba089224d0de25339c
translatedAt: 2026-08-08T14:23:08.247Z
url: https://rafaelpfister.ch/no/blog/styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert
translationModel: gpt-5.6-terra
---

Midea PortaSplit kan etter oppsett styres direkte på det lokale nettverket via Home Assistant. Til dette trenger fellesskapsintegrasjonen to enhetsspesifikke tilgangsverdier fra Midea-skyen: token og nøkkel.

Denne artikkelen tar for seg valg, oppsett og sikring av integrasjonen. Løsningene som beskrives, kommer fra fellesskapet og støttes ikke offisielt av verken Midea eller Home Assistant. Endringer i fastvare eller skyen kan derfor når som helst påvirke hvordan de fungerer. Bakgrunnen for token-grensesnittet og den tvetydige advarselen om avvikling finnes i [analysen av Midea-sky-API-ene](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Slik fungerer lokal styring

De faktiske styrekommandoene går etter oppsettet direkte fra Home Assistant til PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

En bryterkommando trenger ikke gå via en ekstern Midea-server, responstiden er kort, en feil i Midea-skyen setter ikke nødvendigvis den allerede konfigurerte lokale styringen ut av spill, og enheten kan i utgangspunktet fortsatt styres uten internettilgang.

På nyere enheter med den såkalte V3-protokollen godtar PortaSplit imidlertid ikke lokale kommandoer uten beskyttelse. Home Assistant trenger to enhetsspesifikke verdier, et token og en nøkkel, som brukes til autentisering og kryptering av den lokale forbindelsen. Integrasjonen henter dem én gang via et Midea-skygrensesnitt under førstegangsoppsettet og lagrer dem deretter lokalt; ingen skyforbindelse er nødvendig for videre styring.

Forenklet ser prosessen slik ut:

1. PortaSplit kobles til MSmartHome.
2. Home Assistant logger inn på en Midea-sky.
3. Home Assistant mottar enhets-ID, token og nøkkel.
4. Token og nøkkel lagres lokalt.
5. Home Assistant styrer PortaSplit direkte i LAN-et.

## Hvilken integrasjon passer

### Midea Smart AC

Repositoryet <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> fokuserer på Midea-klimaanlegg og relaterte OEM-modeller og støtter enhetstypene `0xAC` og `0xCC`. Det tilbyr lokal styring, grafisk oppsett, automatisk oppdagelse, manuelt oppsett med token og nøkkel samt automatisk henting av enhetens funksjoner. PortaSplits «Out Silent Mode» støttes eksplisitt.

Som indikasjon på kompatibilitet nevner prosjektet blant annet appene Artic King, Midea Air, NetHome Plus, SmartHome eller MSmartHome, Toshiba AC NA og 美的美居. PortaSplit bruker vanligvis MSmartHome i Europa og passer dermed inn i dette økosystemet.

### Midea AC LAN

Repositoryet <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> støtter ikke bare klimaanlegg, men en rekke andre Midea-enhetsklasser: avfuktere, vifter, luftrensere, vaskemaskiner, tørketromler, oppvaskmaskiner, varmtvannsberedere, varmepumper, kjøleskap og mer, delvis også under andre merker som Carrier eller Electrolux. Det tilbyr også lokal kommunikasjon, automatisk enhetsoppdagelse og ekstra sensorer, og holder ifølge prosjektbeskrivelsen en lengre TCP-forbindelse åpen til enheten for å synkronisere statusendringer raskt. Home Assistant 2024.4.1 eller nyere kreves.

Den største ulempen er for tiden utviklerens advarsel: Sky-token-API-ene som brukes til å legge til nye enheter, avvikles gradvis. Dette kan gjøre det umulig å legge til nye enheter senere.

### Anbefaling

For en ren PortaSplit-installasjon ville jeg startet med `Midea Smart AC` og hatt `Midea AC LAN` i bakhodet som et alternativ. `Midea Smart AC` er mer spesialisert på klimaanlegg og dokumenterer de nåværende PortaSplit-funksjonene eksplisitt.

Det er ikke hensiktsmessig å bruke begge integrasjonene samtidig og permanent med samme enhet. Flere parallelle forbindelser fører til statusproblemer, unødvendig nettverkstrafikk og vanskelig forståelig oppførsel.

## Hva integrasjonen gir

Etter oppsettet vises PortaSplit som en `climate`-entitet i Home Assistant. Avhengig av fastvare og integrasjon er blant annet følgende funksjoner tilgjengelige:

- Slå av og på
- Angi ønsket temperatur
- Lese av gjeldende romtemperatur
- Kjøling, avfukting og ren viftedrift
- Angi viftehastighet
- Styre swing-funksjonen
- Eco- og Boost-modus
- Lese av luftfuktighet
- Vise feilkoder
- Lese av energi- og effektverdier
- Vise kompressorverdier
- Aktivere stillegående modus for utedelen

Hvilke entiteter som faktisk vises, avhenger av modellen, fastvaren, protokollen som brukes og den aktuelle integrasjonen. `Midea Smart AC` henter funksjonene som rapporteres av enheten, og skjuler funksjoner modellen ikke støtter. `Midea AC LAN` dokumenterer også omfattende klimaentiteter, inkludert temperatur, luftfuktighet, aktuell effekt, total energi, kompressorfrekvens, pumpestatus og ulike driftsmoduser, og angir egne metoder for dekoding av energidata for bestemte PortaSplit-undertyper.

Ikke alle viste målinger trenger å være korrekte. Særlig energiforbruk og effekt overføres i ulike formater på forskjellige Midea-modeller. Hvis Home Assistant viser åpenbart feil verdier, må vanligvis dekodingsmetoden som brukes tilpasses, og det betyr ikke at enheten er defekt.

## Forutsetninger

Du trenger en Midea PortaSplit med Wi-Fi-funksjon, et 2,4 GHz Wi-Fi-nettverk, MSmartHome-appen, en Midea-brukerkonto, Home Assistant, HACS og nettverkstilgang mellom Home Assistant og PortaSplit. PortaSplit bør først kobles til MSmartHome-appen på vanlig måte, og først deretter legges til i Home Assistant.

## Trinn 1: Koble PortaSplit til MSmartHome

1. Installer MSmartHome-appen.
2. Opprett eller logg inn på Midea-kontoen.
3. Sett PortaSplit i Wi-Fi-paringsmodus.
4. Koble enheten til 2,4 GHz Wi-Fi-nettverket.
5. Kontroller at PortaSplit kan styres via appen.

Mange IoT-enheter støtter fortsatt bare 2,4 GHz. Dersom ruteren bruker samme SSID for 2,4 og 5 GHz, fungerer oppsettet vanligvis likevel. Ved problemer kan det hjelpe å midlertidig opprette et separat 2,4 GHz Wi-Fi-nettverk.

## Trinn 2: Installer HACS

HACS er Home Assistant Community Store. Den gjør det mulig å installere fellesskapsintegrasjoner som ikke er en del av Home Assistant Core. Etter at HACS er installert, åpner du HACS, går til integrasjoner, søker etter `Midea Smart AC`, laster ned integrasjonen og starter Home Assistant på nytt. Alternativt kan du søke etter `Midea AC LAN`.

HACS forenkler installasjon og oppdateringer. Det gjør imidlertid ikke en Custom Integration til en offisielt kontrollert Home Assistant-komponent. Dette skillet er vesentlig fra et sikkerhetsperspektiv og behandles nedenfor.

## Trinn 3: Legg til Midea Smart AC

Etter omstart går du via Innstillinger, Enheter og tjenester og Legg til integrasjon til søket etter `Midea Smart AC`, og deretter til `Discover devices`. Integrasjonen kan enten søke gjennom hele lokalnettet eller spørre målrettet etter PortaSplits IP-adresse.

Når enheten blir funnet, trenger integrasjonen for nyere V3-enheter region, Midea-konto, passord og enhets-ID samt tokenet og nøkkelen som avledes av disse. Skyregionen må passe med kontoen som brukes. Ved problemer anbefaler prosjektet også å prøve de andre tilgjengelige regionene.

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

For V3-enheter angir dokumentasjonen tokenet som en heksadesimal tegnstreng på 128 tegn og nøkkelen som en på 64 tegn. Begge verdiene er hemmeligheter og må behandles deretter. De som ikke ønsker å hente tilgangsdataene via Discovery, kan hente dem med sin egen konto via CLI-en `msmart-ng`.

## Bruk PortaSplit sikkert

Når du styrer PortaSplit lokalt, flyttes en del av kontrollen tilbake fra produsentens sky, men ansvaret flyttes samtidig til ditt eget nettverk. Punktene nedenfor bidrar til at token og nøkkel gjør begrenset skade selv ved en hendelse, og at enheten forblir godt isolert.

### Token og nøkkel er hemmeligheter

Token og nøkkel autentiserer den lokale kommunikasjonen med enheten og må behandles som et passord. Det viktigste for driften er: De skal ikke havne i logger, ukrypterte sikkerhetskopier eller et repository.

### Ingen portvideresending til PortaSplit

Den vanligste unngåelige feilen ville være å gjøre den lokale enhetsporten direkte tilgjengelig fra internett. En regel som denne ville være farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finnes ingen god grunn til å gjøre PortaSplit direkte tilgjengelig fra internett. Home Assistant befinner seg allerede i det lokale nettverket og fungerer som den kontrollerende instansen. Ruteren bør ikke ha portvideresending til PortaSplit, UPnP bør begrenses eller deaktiveres der det er mulig, innkommende forbindelser bør blokkeres som standard, og ingen DMZ-frigivelse bør brukes for enheten.

### Eget IoT-VLAN

Den beste nettverksarkitekturen er et separat IoT-nettverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit befinner seg i IoT-VLAN-et. Home Assistant kan få målrettet tilgang til enheten, men PortaSplit skal ikke fritt kunne få tilgang til PC-er, NAS-er og andre interne systemer. En mulig brannmurlogikk:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under førstegangsoppsettet trenger enheten internettilgang til Midea-skyen. Etter vellykket lokalt oppsett kan du teste om utgående internettilgang kan blokkeres. Det bør ikke settes en endelig blokkering umiddelbart. Først må det kontrolleres om lokal styring fortsatt fungerer, om enheten fortsatt er tilgjengelig etter omstart, om den tåler omstart av ruteren, om den fortsatt reagerer etter flere dager, om MSmartHome-appen fortsatt trengs, og om fastvareoppdateringer fortsatt tilbys. De som vil fortsette å bruke skyen og fastvareoppdateringer, kan tillate utgående internettilgang midlertidig og blokkere den igjen etterpå.

### Nettverkssegmentering kan hindre Discovery

Automatisk enhetssøk er ofte basert på broadcast- eller multicast-trafikk, som vanligvis ikke rutes over VLAN-grenser. Home Assistant finner derfor kanskje ikke PortaSplit automatisk, selv om en vanlig IP-forbindelse ville vært tillatt.

Da kan det hjelpe å sette opp PortaSplit midlertidig i samme VLAN som Home Assistant, angi enhetens IP-adresse manuelt, bruke en egnet broadcast-reléfunksjon eller definere målrettede brannmurregler etter oppsettet. Manuell konfigurasjon er ofte til og med det bedre alternativet fra et sikkerhetsperspektiv, fordi det ikke er nødvendig å tillate ekstra broadcast-trafikk mellom nettverkene.

### Statisk DHCP-tilordning

PortaSplit bør få en fast DHCP-tilordning i ruteren:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservasjon er vanligvis å foretrekke fremfor en statisk IP-adresse satt på enheten. Home Assistant finner enheten pålitelig, brannmurregler kan begrenses til en fast adresse, feilsøking blir enklere, og tilordningen forblir stabil etter omstart av ruter eller enhet. En brannmurregel kan dermed formuleres svært strengt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den faktisk nødvendige porten må verifiseres ut fra integrasjonen og din egen enhet.

### Home Assistant som sentralt tillitsanker

Når du styrer PortaSplit lokalt, flyttes tilliten delvis fra Midea-skyen til Home Assistant. Hvis Home Assistant kompromitteres, kan en angriper potensielt kontrollere ikke bare klimaanlegget, men hele smarthjemmet.

Home Assistant bør derfor oppdateres regelmessig, ikke publiseres via ubeskyttet portvideresending, beskyttes med et sterkt og unikt passord, bruke flerfaktorautentisering, lage krypterte sikkerhetskopier, bare inneholde nødvendige tillegg og ikke tillate unødvendig SSH-tilgang fra internett. For fjerntilgang er VPN, Home Assistant Cloud eller en korrekt konfigurert reverse proxy bedre alternativer enn enkel portvideresending til port 8123.

### HACS og risikoen i forsyningskjeden

`Midea Smart AC` og `Midea AC LAN` er Custom Integrations. De kjører i Home Assistant og får dermed omfattende tilgang til kjøretidsmiljøet. En ondsinnet eller kompromittert integrasjon kan teoretisk lese konfigurasjonsdata, hente ut hemmeligheter, opprette nettverkstilkoblinger, skanne enheter i lokalnettet, lese tilstander for andre entiteter, overføre data til eksterne systemer og påvirke tilgjengeligheten til Home Assistant.

Dette betyr ikke at de nevnte integrasjonene er ondsinnede. Begge prosjektene er offentlig tilgjengelige, utvikles aktivt og har et synlig fellesskap. Open source er imidlertid ingen automatisk sikkerhetsgaranti. Før installasjon er det minst verdt å se på om repositoryet vedlikeholdes aktivt, om det finnes regelmessige utgivelser, hvor mange som bidrar til koden, om det finnes åpne sikkerhetssaker, om vedlikeholdere eller repository-eiere nylig har blitt byttet, om HACS peker til forventet repository og om en oppdatering inneholder uvanlig store eller uforklarlige endringer.

Oppdateringer bør ikke installeres blindt umiddelbart etter publisering. Særlig for sikkerhetskritiske smarthjemsystemer er det fornuftig å vente noen dager og kontrollere utgivelsesnotater og rapporterte problemer.

### Sikre skykontoen

Så lenge Midea-skyen brukes til oppsett eller appstyring, forblir også Midea-kontoen en del av sikkerhetsmodellen. Den bør ha et unikt passord som ikke deles med andre tjenester, en passordbehandler, flerfaktorautentisering dersom det tilbys, fjerning av gamle smarttelefoner og økter, ingen delte kontoer og regelmessig kontroll av hvilke enheter som er registrert på kontoen.

Hvis Home Assistant-integrasjonen ber om brukernavn og passord under oppsettet, må du kontrollere om tilgangsdataene bare brukes til engangshenting av token eller lagres permanent. Utviklerne av `Midea Smart AC` skriver at enheter etter oppsett ikke kobles til innebygde integrasjonskontoer, og at token og nøkkel også kan hentes manuelt med egen konto via CLI. Der det er mulig, bør egen konto foretrekkes fremfor fremmede eller integrerte felleskontoer.

### Blokkere skyen eller ikke?

Etter vellykket oppsett oppstår spørsmålet om PortaSplits internettilgang bør blokkeres helt. Argumenter for blokkering er mindre telemetri, mindre avhengighet av eksterne tjenester, en mindre angrepsvei via produsentens sky, at enheten ikke kan kontakte vilkårlige eksterne mål, og mindre påvirkning fra endringer på skysiden.

Mot dette taler at MSmartHome-appen kanskje ikke lenger fungerer utenfor hjemmenettverket, at fastvareoppdateringer ikke lastes ned, at tids- eller skyfunksjoner kan falle bort, at ny innlogging eller gjenoppretting blir vanskeligere, og at enkelte enheter reagerer uventet etter lang tid frakoblet.

En pragmatisk rekkefølge: Sett opp enheten normalt, test Home Assistant og appen, sikkerhetskopier token og konfigurasjon, blokker internettilgang, start enheten og Home Assistant på nytt, observer i flere dager og åpne eventuelt internettilgangen bare midlertidig igjen ved behov.

### Fastvareoppdateringer: sikkerhetsgevinst eller integrasjonsrisiko?

Fastvareoppdateringer er et dilemma for IoT-enheter. De kan lukke kjente sårbarheter, forbedre stabiliteten, modernisere sikkerhetsmekanismer og gi nye funksjoner. Men de kan også endre lokale grensesnitt, ødelegge reverse-engineering-integrasjoner, gjøre token ugyldige, deaktivere det lokale API-et og innføre nye skyavhengigheter.

PortaSplit-fastvaren som ble levert i januar 2026, fikk for eksempel en ny stillegående modus for utedelen som reduserer støynivået med rundt 6 desibel. Fellesskapsintegrasjonene måtte først kartlegge og implementere denne, dokumentert i en egen GitHub-sak for PortaSplit.

Dette innebærer: Ikke hindre fastvareoppdateringer generelt; før en oppdatering bør du kontrollere om andre Home Assistant-brukere rapporterer problemer, sikkerhetskopiere konfigurasjon og token på forhånd, opprette en Home Assistant-sikkerhetskopi og teste den lokale styringen grundig etter oppdateringen. Sikkerhet betyr ikke «aldri oppdatere». Utdatert fastvare kan være farligere enn en midlertidig inkompatibel integrasjon.

### Debug-logger inneholder sensitive data

Ved problemer ber open source-prosjekter ofte om debug-logger. Dokumentasjonen for `Midea AC LAN` viser hvordan logging aktiveres for de to relevante komponentene:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Deretter kan loggene lastes ned via Innstillinger, System og Logger. Slike logger kan, avhengig av integrasjonen og feilsituasjonen, inneholde lokale IP-adresser, enhets-ID, serienummer, modellidentifikator, skyresponser, kontoinformasjon, token eller deler av dem, nettverkspakker samt tidsstempler og bruksmønstre. De må derfor gjennomgås og sensitive verdier sladdes før de lastes opp til en offentlig GitHub-sak.

Etter at feilsøkingen er ferdig, bør debug-logging fjernes igjen. Permanent aktivert debug-logging øker ikke bare lagringsforbruket, men også mengden sensitiv informasjon i sikkerhetskopiene.

### Hva Midea selv sier om sikkerhet

Midea markedsfører SmartHome-økosystemet sitt med orientering mot flere sikkerhets- og personvernstandarder, blant annet EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling og kravene i EUs Radio Equipment Directive. Dette er positive signaler, men ingen uttalelse om hvordan hver enkelt PortaSplit-fastvare, hvert skyendepunkt og hvert lokale API faktisk er implementert. Sertifiserings- og markedsføringspåstander erstatter ikke teknisk kontroll av den konkrete enheten.

På samme måte ville det være feil å utlede fra advarselen fra en fellesskapsintegrasjon at PortaSplit generelt er usikker. Det beskrevne problemet gjelder arkitekturen til langvarige token og deres bruk av uoffisielle klienter.

### Risiko etter scenario

| Scenario | Risiko | Begrunnelse |
| --- | --- | --- |
| Vanlig hjemmenettverk uten portvideresending | oversiktlig | En angriper må først få tilgang til Wi-Fi, Home Assistant eller en sikkerhetskopi. |
| Flatt hjemmenettverk med mange usikre IoT-enheter | middels | En kompromittert annen IoT-enhet kan nå PortaSplit eller Home Assistant i samme nettverk. |
| PortaSplit direkte tilgjengelig fra internett | høy | Enheten skal aldri publiseres via portvideresending. |
| Token og nøkkel offentlig på GitHub | høy | Hemmelighetene må anses som kompromitterte; om de kan tilbakekalles, er ikke garantert. |
| Separat IoT-VLAN, restriktiv brannmur, lokal styring | forholdsvis lav | Selv ved en sårbarhet i enheten er bevegelsesfriheten i nettverket sterkt begrenset. |

## Sikkerhetskopi av konfigurasjonen

Sikring av token, nøkkel og konfigurasjon er det viktigste engangstrinnet: Når sky-token-grensesnittene først er stengt, er en sikkerhetskopi den eneste måten å foreta et nytt oppsett på. `Midea AC LAN` lagrer en JSON-konfigurasjonsfil for V3-enheter etter vellykket oppsett. Den dokumenterte banen er:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID-en som filnavn:

```text
<device-id>.json
```

Denne filen er ikke et vanlig tekstdokument. Den kan inneholde enhets-ID, serienummer, IP-adresse, token, nøkkel, protokollinformasjon samt sky- og enhetsparametere. Derfor gjelder følgende:

- Ikke last den opp til et offentlig GitHub-repository.
- Ikke publiser den i forum.
- Ikke del den som et usladdet skjermbilde.
- Ikke send den via ukryptert e-post.

Selv et privat Git-repository er ikke automatisk riktig lagringssted, fordi hemmeligheter blir værende i Git-historikken selv om de senere slettes fra den aktuelle filen. Bedre alternativer er en kryptert sikkerhetskopi, en passordbehandler med filvedlegg, en kryptert NAS-sikkerhetskopi, et kryptert frakoblet medium eller et kryptert arkiv med separat lagret passord.

For sikkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Vis filen:

```bash
cat <device-id>.json
```

Filen bør ikke overføres via en offentlig nettjeneste ved kopiering. Et kryptert arkiv, som deretter overføres til en kryptert sikkerhetskopi, er bedre:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filene i `.storage` bør ikke redigeres manuelt. Utvikleren anbefaler uttrykkelig at JSON-filen verken slettes eller endres direkte ved problemer, men at den gis nytt navn og sikkerhetskopieres før endringer.

En fullstendig Home Assistant-sikkerhetskopi inneholder også disse filene. En separat kopi er likevel fornuftig, fordi Home Assistant-sikkerhetskopier kan bli skadet, en gjenoppretting kan overskrive integrasjonen, filen kan være nødvendig spesifikt ved et senere nytt oppsett, og en sikkerhetskopi aldri bare bør ligge på samme system.

## Fjern hemmeligheter fra et publisert Git-repository

Hvis en JSON-fil ved en feil er publisert på GitHub, er det ikke nok å slette den normalt og gjøre en ny commit. Filen forblir tilgjengelig i Git-historikken. Minst disse trinnene er nødvendige:

1. Sett repositoryet til privat umiddelbart, dersom mulig.
2. Fjern filen fra hele Git-historikken.
3. Ta hensyn til GitHub-cacher og forker.
4. Behandle tokenet som kompromittert.
5. Fjern enheten fra Midea-kontoen og koble den til på nytt dersom dette genererer nye nøkler.
6. Sett opp Home Assistant-integrasjonen på nytt.
7. Endre Midea-kontopassordet dersom tilgangsdataene også var berørt.

Om ny paring faktisk genererer et nytt token, varierer etter enhet og skyarkitektur. Du bør ikke stole på at endring av kontopassordet automatisk gjør det lokale enhetstokenet ugyldig.

## Nyttige automasjoner

Etter vellykket integrasjon kan PortaSplit brukes betydelig smartere. Entity-ID-ene må tilpasses din egen installasjon.

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

Ønsket kommunikasjonsretning ser dermed slik ut:

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

## Anbefalt driftsmodus

Midea PortaSplit lar seg integrere overraskende godt i Home Assistant. Etter vellykket oppsett kan den styres lokalt og inngå i automasjoner, slik at en stor del av skyavhengigheten faller bort i daglig bruk.

Fra et sikkerhetsperspektiv er integrasjonen forsvarlig dersom noen grunnregler følges: ingen portvideresending, hold token og nøkkel hemmelige, krypter sikkerhetskopier, kontroller debug-logger før publisering, sikre Home Assistant, segmenter IoT-enheter, begrens utgående internettilgang til det nødvendige, og ikke installer fastvare- og HACS-oppdateringer blindt. Drevet slik forblir PortaSplit et kraftig klimaanlegg og blir samtidig en fornuftig integrerbar del av et lokalt styrt smarthjem.

## Kilder

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjonen `Midea Smart AC`: støttede enhetstyper `0xAC` og `0xCC`, PortaSplit med «Out Silent Mode», skybruk for henting av token og nøkkel på V3-enheter, manuelt oppsett og standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjonen `Midea AC LAN`: støttede enhetsklasser, lengre TCP-forbindelse for statussynkronisering og minimumsversjon Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentasjon av klimaentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): Entiteter og attributter for klimaanlegg, inkludert effekt, total energi, kompressorfrekvens og dekodingsmetodene for energiverdier for enkelte undertyper.

4.  [midea_ac_lan: Veiledning om debug og konfigurasjon](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): Lagring av enhetskonfigurasjonen under `/config/.storage/midea_ac_lan/`, anbefaling om å sikkerhetskopiere fremfor å slette JSON-filen og loggerkonfigurasjonen for debug-logger.

5.  [Issue 779: PortaSplits Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): Forespørsel om støtte for den stillegående modusen for utedelen som ble innført med fastvareoppdateringen i januar 2026, og som reduserer støynivået med rundt 6 desibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): Produsentopplysninger om sikkerhets- og personvernstandardene EN 303 645, PSTI, NIST, GDPR og RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installasjon og administrasjon av Custom Integrations som ikke er en del av Home Assistant Core.
