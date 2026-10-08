---
title: "Koble digitalSTROM til Home Assistant: lokal integrasjon og automasjoner"
navTitle: "digitalSTROM og HA"
description: "En lokal Home Assistant-integrasjon for digitalSTROM-serveren, med bevegelseslys, dusjmodus, musikk med 4× trykk og automatisk Borte via FRITZ!Box. Med problemene som oppstod i praksis."
date: "2026-10-06"
kategorie: "Home Assistant og IoT"
timeToRead: "13 min lesetid"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant
slug: "koble-digitalstrom-til-home-assistant-lokal-integrasjon-og-automasjoner"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
url: https://rafaelpfister.ch/no/blog/koble-digitalstrom-til-home-assistant-lokal-integrasjon-og-automasjoner
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:25:59.754Z
translationReview: automatic
---

I digitalSTROM-installasjoner er lys, persienner og andre forbrukere koblet til klemmer i sikringsskapet, styrt via brytere og en digitalSTROM-server (dSS20). Når flere systemer som Philips Hue, Sonos og en FRITZ!Box kommer til, er det nærliggende å samle alt lokalt i Home Assistant, uten skykonto og uten å lagre passordet til dSS i Home Assistant. Derfor har jeg skrevet en liten integrasjon og publisert den sammen med passende automasjoner som [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under MIT-lisens.

Kort oppsummert: dSS tilbyr et brukbart JSON-API med hendelser. De som bruker det skånsomt, får brytere, scener og aktivitetene Borte og Hjem uten forsinkelse inn i Home Assistant og kan dermed utløse automasjoner som digitalSTROM alene ikke kjenner.

## Slik fungerer integrasjonen

DSS tilbyr API-et sitt på port 8080 via HTTPS, med et selvsignert sertifikat. digitalSTROM har app-tokener for programmer: Programmet ber om et token, brukeren godkjenner det i Configurator under System > Access authorization, og deretter logger programmet inn med det. Passordet til dSS forblir hos brukeren.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-s` | ingen fremdriftsvisning |
| `-k` | godta selvsignert sertifikat fra dSS |
| `requestApplicationToken` | oppretter et app-token som må godkjennes i Configurator |
| `loginApplication` | bytter det godkjente app-tokenet mot et sesjonstoken |
| `event/subscribe` | abonnerer på en hendelse (`callScene`, `stateChange`, `buttonClick` …) under en valgfri ID |
| `event/get` med `timeout` | venter i opptil 30 sekunder på nye hendelser (long poll) |

</details>

Integrasjonen holder en slik long-poll-forbindelse åpen permanent. Hver scene som utløses av en bryter, appen eller Configurator, kommer inn som en hendelse. Tilstander som «lys på i rommet» eller tilstedeværelse ligger i hurtigbufferen til dSS under `/usr/states` og kan hentes uten å belaste klemmene.

Direkte spørringer av utgangsverdier (`device/getOutputValue`) går derimot via dS485-bussen helt til klemmen og tar et halvt til ett sekund per verdi. Mange slike spørringer med korte intervaller kan overbelaste målerne. Integrasjonen leser derfor bare persienneposisjoner og Joker-utganger direkte, hvert 15. minutt og omtrent ett minutt etter en bevegelse.

| Plattform | Innhold |
|---|---|
| `light` | ett lys per rom, styrt via romscener som bryterne; lysstyrke for dimmede klemmer, bare på/av for brytede |
| `cover` | persienner med opp, ned, stopp og posisjon |
| `scene` | stemninger med navn i Configurator |
| `button` | Borte (scene 72) og Hjem (scene 71) |
| `binary_sensor` | tilstedeværelse, vindalarm, bevegelsessensorer, Joker-utganger |
| `sensor` | totalt forbruk |

I tillegg rapporterer integrasjonen hver romscene samt Borte og Hjem som hendelsen `digitalstrom_local_event` til Home Assistant. Dermed kan 2× eller 4× trykk på en vanlig lysbryter tildeles fritt.

## Automatiseringer som blueprints

De følgende automatiseringene er inkludert som blueprints i repositoriet og kan importeres til Home Assistant via importknappen i README.

### Bevegelseslys som ikke slår av lys som er slått på manuelt

Ved bevegelse slås lyset på, og etter en justerbar tid uten bevegelse slås det av igjen. Valgfritt gjelder dette bare under en terskel for lysstyrke, bare i et tidsvindu eller dimmet om natten. Hvis noen slår på lyset med bryteren, forblir det på. En hjelpekontakt (`input_boolean`) husker om automasjonen slo på lyset; bare da slår den det også av igjen.

En detalj blir først tydelig i drift: Utløseren «sensoren har vært rolig i 2 minutter» er en teller som Home Assistant forkaster ved hver omstart. Dersom sensoren sist registrerte bevegelse før omstarten, kommer det ingen ny overgang til «rolig», og lyset forblir på. Blueprintet kontrollerer derfor også hvert minutt om et lys det har slått på, fortsatt lyser selv om sensoren har vært rolig lenge nok.

### Slå av lys når du forlater rommet

Tidspunktet for å slå av bevegelseslyset er et kompromiss: Er det for kort, slås lyset av mens noen står stille i rommet; er det for langt, fortsetter det å lyse i flere minutter etter at rommet er forlatt. Et annet blueprint bruker derfor en andre sensor utenfor rommet, typisk i gangen. Hvis denne registrerer bevegelse, sensoren i rommet har registrert bevegelse kort tid før (innen 30 sekunder), og det deretter forblir rolig der inne, slås lyset i rommet av umiddelbart. Med Hue-sensorer, som rapporterer «rolig» rundt 10 sekunder etter siste bevegelse, skjer dette omtrent 10 til 15 sekunder etter at man går ut.

For at ingen skal sitte i mørket, gjelder regelen bare under visse betingelser: Lyset må ha blitt slått på av bevegelsesautomasjonen, en dusjmodus må ikke være aktiv, og det kan maksimalt være én person hjemme. Antall personer beregnes ut fra mobiltelefonene som Home Assistant kjenner som personer. 30-sekundersbetingelsen beskytter dessuten mot tilfeller der noen står stille lenge i rommet mens en annen person går gjennom gangen.

### Dusjmodus med 2× trykk

I dusjen registrerer en bevegelsessensor som regel ingen, og etter den angitte tiden blir det mørkt. 2× trykk på baderomsbryteren utløser stemning 2 (scene 17) i digitalSTROM. Automatiseringen gjenkjenner denne scenen i rommet og slår på en dusjmodus som blokkerer avslåing. Bevegelser i de første 2 minuttene ignoreres (avkledning, innstigning). Den første bevegelsen etterpå avslutter dusjmodusen, og fra da av gjelder normal avslåingstid igjen. Etter en justerbar maksimal varighet (standard 20 minutter) avsluttes den av seg selv.

### 4× trykk starter Sonos

Når man trykker raskt flere ganger på en bryter, går digitalSTROM gjennom stemningene etter tur: scene 5, 17, 18 og 19 ved fjerde trykk. Hvis scene 19 settes til «ikke endre utgang» for alle lamper i Configurator, er den ledig for andre formål. Et blueprint reagerer på denne scenen, slår av romlyset og starter Sonos-høyttaleren i rommet eller, i rom uten høyttaler, flere høyttalere samlet.

Skriptet tar hensyn til tre punkter: Hvis en høyttaler allerede spiller, legges det nye rommet til gruppen, slik at avspillingen forblir synkron. Hvis ingenting kan fortsettes, spilles en radiostasjon fra Radio Browser-katalogen. Og før hver start settes den samme startvolumet, ellers spiller det ene rommet lavt og det andre like høyt som sist noen hørte på musikk.

### Borte og Hjem via FRITZ!Box

Integrasjonen FRITZ!Box Tools rapporterer om en mobiltelefon er på WLAN; den evaluerer enhetslisten til boksen og dekker dermed 2,4 GHz-nett, 5 GHz-nett og LAN samtidig. Hvis ingen lenger er hjemme og Borte ikke er trykket, utløser et blueprint Borte. Når noen kommer hjem, følger Hjem og, hvis det er mørkt, et velkomstlys.

I tillegg kan en automatisering slå av lampene i andre systemer (for eksempel Hue) og sette Sonos på pause ved Borte. digitalSTROM-lampene slår dSS av selv.

## Plantegning som dashboard

En oversiktlig fremstilling i Home Assistant er en plantegning med det innebygde kortet `picture-elements`. For hvert rom ligger en gjennomsiktig SVG over planen, som erstattes med en lett gul når lyset er slått på; et trykk på rommet slår på lyset. Lamper, høyttalere, persienner og sensorer er plassert som symboler der de befinner seg. Veiledningen med eksempelkonfigurasjon ligger i repositoriet under `docs/floor-plan-dashboard.md`.

## Problemer fra praksis

| Problem | Årsak | Løsning |
|---|---|---|
| Borte-bryter uten effekt | dSS er ikke på nettverket, aktiviteter på tvers av flere strømkretser går via serveren | nettverksforbindelsen til dSS ble gjenopprettet |
| Persienner reagerer ikke | vindalarm var satt til «aktiv», selv om ingen vindsensor finnes | scene 87 («ingen vind») ble utløst for alle rom |
| Persienner går ikke opp ved Borte | scene 72 er satt til «ikke endre utgang» (`dontCare`) | `device/setSceneMode` med `dontCare=0`; verdien `false` ble godtatt, men ignorert |
| Lyset kan ikke dimmes | lysrør på en brytet klemme (utgangsmodus 35) | integrasjonen oppdager brytede klemmer og tilbyr der bare på/av |
| Lyset forblir på etter omstart | teller for «rolig i X minutter» går tapt ved omstart | ekstra kontroll hvert minutt |
| Mobiltelefonen anses som fraværende | iPhone bruker en skiftende privat WLAN-adresse og kobler kort fra WLAN i hvilemodus | privat WLAN-adresse satt til «Fast», grace-periode på 10 minutter |
| DECT sender kontinuerlig | «DECT Eco» er ikke tilgjengelig så snart en FRITZ!-smarthjem-enhet er registrert | unngå DECT-stikkontakter hvis DECT Eco er ønsket |

Et enhetssøk på Hue-broen legger til alle enheter som for øyeblikket er i paringsmodus. Kontroller deretter i enhetslisten at bare dine egne lamper er lagt til.

Scenetabellene til klemmene kan leses ut og endres via API-et, for eksempel med `device/getSceneMode` og `device/saveScene`. Hvert av disse kallene går via bussen. Sikkerhetskopier de gamle verdiene før endringer, for eksempel i en CSV-fil, slik at de kan gjenopprettes ved behov.

## Kilder

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integrasjon og blueprints fra denne artikkelen, MIT-lisens.

2.  [digitalSTROM: Håndbok for betjening og innstillinger](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktiviteten Borte (hold inne i 3 sekunder), innstillingen «ikke endre utgang» i kapittel 3.5.2.

3.  [digitalSTROM: Knappdokumentasjon for klemmene](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Fabrikkinnstillinger for knappfunksjonene per klemmetype.

4.  [digitalSTROM: Bruksanvisninger](https://www.digitalstrom.com/bedienungsanleitungen/): Oversikt over håndbøker, planleggings- og installasjonsdokumenter.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Tilstedeværelsesregistrering, WLAN-bryter, integrasjonens alternativer.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grunnlag for plantegningsdashboardet.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import og bruk av blueprints.

8.  [FRITZ! Kunnskapsbase: FRITZ!-stikkontakt mister forbindelsen](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Merknader om DECT-rekkevidde og sendeeffekt for smarthjem-stikkontakter.

9.  [Radio Browser](https://www.radio-browser.info/): Fritt katalog over radiostrømmer, integrert i Home Assistant som mediekilde.
