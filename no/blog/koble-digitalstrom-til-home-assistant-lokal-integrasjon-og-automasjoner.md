---
title: "Koble digitalSTROM til Home Assistant: lokal integrasjon og automasjoner"
navTitle: "digitalSTROM og HA"
description: "En lokal Home Assistant-integrasjon for digitalSTROM-serveren, samt bevegelseslys, dusjmodus, musikk med 4× trykk og automatisk bortemodus via FRITZ!Box. Med problemene som oppstod i praksis."
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
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:24:36.133Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/koble-digitalstrom-til-home-assistant-lokal-integrasjon-og-automasjoner
---

I digitalSTROM-installasjoner er lys, persienner og andre forbrukere koblet til klemmer i sikringsskapet, styrt via trykknapper og en digitalSTROM-server (dSS20). Når flere systemer som Philips Hue, Sonos og en FRITZ!Box kommer til, er det naturlig å samle alt lokalt i Home Assistant, uten skykonto og uten å lagre passordet til dSS i Home Assistant. Til dette har jeg skrevet en liten integrasjon og publisert den sammen med passende automasjoner som [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under MIT-lisens.

Kort oppsummert: dSS tilbyr et brukbart JSON-API med hendelser. De som bruker det skånsomt, får trykknapper, scener og aktivitetene Gå og Komme til Home Assistant uten forsinkelse og kan dermed utløse automasjoner som digitalSTROM alene ikke kjenner til.

## Slik fungerer integrasjonen

DSS tilbyr API-et sitt på port 8080 via HTTPS, med et selvsignert sertifikat. For programmer har digitalSTROM app-tokener: Programmet ber om et token, brukeren godkjenner det i Configurator under System > Tilgangstillatelse, og deretter logger programmet inn med det. Passordet til dSS forblir hos brukeren.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-s` | ingen fremdriftsvisning |
| `-k` | godtar selvsignert sertifikat fra dSS |
| `requestApplicationToken` | oppretter et app-token som må godkjennes i Configurator |
| `loginApplication` | bytter det godkjente app-tokenet mot et økt-token |
| `event/subscribe` | abonnerer på en hendelse (`callScene`, `stateChange`, `buttonClick` …) under en fritt valgt ID |
| `event/get` med `timeout` | venter opptil 30 sekunder på nye hendelser (long poll) |

</details>

Integrasjonen holder en slik long-poll-forbindelse åpen kontinuerlig. Hver scene som en trykknapp, appen eller Configurator utløser, kommer inn som en hendelse. Tilstander som «lys på i rommet» eller tilstedeværelse ligger i hurtigbufferen til dSS under `/usr/states` og kan hentes uten å belaste klemmene.

Direkte spørringer av utgangsverdier (`device/getOutputValue`) går derimot over dS485-bussen helt til klemmen og tar et halvt til ett sekund per verdi. Mange slike spørringer med korte mellomrom kan overbelaste målerne. Integrasjonen leser derfor bare persienneposisjoner og Joker-utganger direkte, hvert 15. minutt og omtrent ett minutt etter en bevegelse.

| Plattform | Innhold |
|---|---|
| `light` | ett lys per rom, styrt via romscener som trykknappene; lysstyrke for dimmede klemmer, bare av/på for bryterstyrte |
| `cover` | persienner med opp, ned, stopp og posisjon |
| `scene` | stemninger med navn i Configurator |
| `button` | Gå (scene 72) og Komme (scene 71) |
| `binary_sensor` | tilstedeværelse, vindalarm, bevegelsessensorer, Joker-utganger |
| `sensor` | totalt forbruk |

I tillegg rapporterer integrasjonen hver romscene samt Gå og Komme som hendelsen `digitalstrom_local_event` til Home Assistant. Dermed kan 2× eller 4× trykk på en vanlig lysbryter tildeles fritt.

## Automatiseringer som Blueprints

Følgende automasjoner følger med i repositoryet som Blueprints og kan importeres til Home Assistant via importknappen i README-filen.

### Bevegelseslys som ikke slår av lys som er slått på manuelt

Ved bevegelse slås lyset på, og etter en justerbar tid uten bevegelse slås det av igjen. Valgfritt gjelder dette bare under en lysstyrketerskel, bare innenfor et tidsvindu eller med dempet lys om natten. Hvis noen slår på lyset med trykknappen, forblir det på. En hjelpekontakt (`input_boolean`) husker om automasjonen slo på lyset; bare da slår den det også av igjen.

En detalj blir først synlig i drift: Utløseren «sensoren har vært rolig i 2 minutter» er en teller som Home Assistant forkaster ved hver omstart. Hvis sensoren sist registrerte bevegelse før omstarten, kommer det ingen ny overgang til «rolig», og lyset blir værende på. Blueprintet kontrollerer derfor i tillegg hvert minutt om et lys det har slått på, fortsatt er på selv om sensoren har vært rolig lenge nok.

### Slå av lyset når rommet forlates

Utkoblingstiden for bevegelseslyset er et kompromiss: Er den for kort, slår lyset seg av mens noen står stille i rommet; er den for lang, fortsetter det å lyse i flere minutter etter at rommet er forlatt. Et annet Blueprint bruker derfor en andre sensor utenfor rommet, vanligvis i gangen. Hvis denne registrerer bevegelse, sensoren i rommet like før (innen 30 sekunder) fortsatt har registrert bevegelse, og det deretter forblir rolig der, slås lyset i rommet av umiddelbart. Med Hue-sensorer, som melder «rolig» omtrent 10 sekunder etter siste bevegelse, er dette rundt 10 til 15 sekunder etter at man går ut.

For at ingen skal sitte i mørket, gjelder regelen bare under bestemte vilkår: Lyset må ha blitt slått på av bevegelsesautomatiseringen, dusjmodus må ikke være aktiv, og det kan være høyst én person hjemme. Antall personer beregnes ut fra mobiltelefonene som Home Assistant kjenner som personer. 30-sekundersvilkåret beskytter dessuten mot tilfeller der noen har stått stille lenge i rommet, mens en annen person går gjennom gangen.

### Dusjmodus med 2× trykk

I dusjen registrerer en bevegelsessensor vanligvis ingen, og etter den angitte tiden blir det mørkt. 2× trykk på baderomsknappen utløser stemning 2 (scene 17) i digitalSTROM. Automatiseringen registrerer denne scenen i rommet og slår på en dusjmodus som blokkerer utkobling. Bevegelser de første 2 minuttene ignoreres (avkledning, innstigning). Den første bevegelsen etterpå avslutter dusjmodusen, og deretter gjelder den vanlige utkoblingstiden igjen. Etter en justerbar maksimal varighet (standard 20 minutter) avsluttes den av seg selv.

### 4× trykk starter Sonos

Når man trykker raskt flere ganger på en knapp, går digitalSTROM gjennom stemningene i rekkefølge: scene 5, 17, 18 og 19 ved fjerde trykk. Hvis scene 19 settes til «Ikke endre utgang» for alle lamper i Configurator, er den ledig for andre formål. Et Blueprint reagerer på denne scenen, slår av romlyset og starter Sonos-høyttaleren i rommet eller, i rom uten høyttaler, flere høyttalere samtidig.

Skriptet til dette tar hensyn til tre ting: Hvis en høyttaler allerede spiller, legges det nye rommet til gruppen, slik at avspillingen forblir synkron. Hvis ingenting kan fortsettes, spilles en radiostasjon fra Radio Browser-katalogen. Og før hver start settes samme startvolum, ellers spiller det ene rommet lavt og det andre like høyt som noen sist hørte på musikk.

### Gå og Komme via FRITZ!Box

Integrasjonen FRITZ!Box Tools rapporterer om en mobiltelefon er på WLAN; den evaluerer enhetslisten fra boksen og dekker dermed 2,4 GHz-nett, 5 GHz-nett og LAN samtidig. Hvis ingen lenger er hjemme og Gå ikke ble trykket, utløser et Blueprint Gå. Når noen kommer hjem, følger Komme og, hvis det er mørkt, et velkomstlys.

I tillegg kan en automatisering slå av lampene i andre systemer (for eksempel Hue) og sette Sonos på pause ved Gå. digitalSTROM-lampene slås av av dSS selv.

## Plantegning som dashboard

En oversiktlig visning i Home Assistant er en plantegning med det innebygde kortet `picture-elements`. For hvert rom ligger det en gjennomsiktig SVG over planen, som erstattes med en svakt gul når lyset er på; et trykk på rommet slår på lyset. Lamper, høyttalere, persienner og sensorer vises som symboler på sine plasseringer. Veiledningen med eksempelkonfigurasjon ligger i repositoryet under `docs/floor-plan-dashboard.md`.

## Problemer fra praksis

| Problem | Årsak | Løsning |
|---|---|---|
| Gå-knappen har ingen virkning | dSS er ikke på nettverket, aktiviteter over flere strømkretser går via serveren | nettverkstilkoblingen til dSS ble gjenopprettet |
| Persiennene reagerer ikke | vindalarm stod på «aktiv», selv om ingen vindsensor finnes | scene 87 («ingen vind») ble utløst for alle rom |
| Persiennene kjører ikke opp ved Gå | scene 72 står på «Ikke endre utgang» (`dontCare`) | `device/setSceneMode` med `dontCare=0`; verdien `false` ble godtatt, men ignorert |
| Lyset kan ikke dimmes | lysrør på en bryterstyrt klemme (utgangsmodus 35) | integrasjonen oppdager bryterstyrte klemmer og tilbyr der bare av/på |
| Lyset forblir på etter omstart | telleren for «rolig i X minutter» går tapt ved omstart | ekstra kontroll hvert minutt |
| Mobiltelefonen regnes som borte | iPhone bruker en skiftende privat WLAN-adresse og kobler kort fra WLAN i hvilemodus | Privat WLAN-adresse satt til «Fast», ventetid 10 minutter |
| DECT sender hele tiden | «DECT Eco» er ikke tilgjengelig så snart en FRITZ!-smarthjemenhet er registrert | unngå DECT-stikkontakter dersom DECT Eco er ønsket |

Et enhetssøk på Hue Bridge legger til alle enheter som akkurat nå er i paringsmodus. Kontroller deretter i enhetslisten at bare dine egne lamper er blitt lagt til.

Scenetabellene for klemmene kan leses ut og endres via API-et, for eksempel med `device/getSceneMode` og `device/saveScene`. Hvert av disse kallene går over bussen. Ta sikkerhetskopi av de gamle verdiene før endringer, for eksempel i en CSV-fil, slik at de kan gjenopprettes ved behov.

## Endringslogg

- **25. oktober 2026:** Kildelenken ble oppdatert til den nye adressen til FRITZ!-kunnskapsdatabasen.

## Kilder

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integrasjon og Blueprints fra denne artikkelen, MIT-lisens.

2.  [digitalSTROM: Håndbok for betjening og innstillinger](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktiviteten Gå (hold inne i 3 sekunder), innstillingen «Ikke endre utgang» i kapittel 3.5.2.

3.  [digitalSTROM: Dokumentasjon for klemmenes knapper](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Fabrikkinnstillinger for knappfunksjoner per klemmetype.

4.  [digitalSTROM: Bruksanvisninger](https://www.digitalstrom.com/bedienungsanleitungen/): Oversikt over håndbøker, planleggings- og installasjonsdokumenter.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Tilstedeværelsesregistrering, WLAN-brytere, alternativer for integrasjonen.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grunnlag for plantegningsdashboardet.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import og bruk av Blueprints.

8.  [FRITZ! Kunnskapsdatabase: FRITZ!-stikkontakt mister forbindelsen](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Informasjon om DECT-rekkevidde og sendeeffekt for smarthjem-stikkontakter.

9.  [Radio Browser](https://www.radio-browser.info/): Gratis katalog over radiostrømmer, integrert i Home Assistant som mediekilde.
