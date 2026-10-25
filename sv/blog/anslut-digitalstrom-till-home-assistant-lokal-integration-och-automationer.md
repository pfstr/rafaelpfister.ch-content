---
title: "Anslut digitalSTROM till Home Assistant: lokal integration och automationer"
navTitle: "digitalSTROM och HA"
description: "En lokal Home Assistant-integration för digitalSTROM-servern, med rörelsestyrt ljus, duschläge, musik med 4× tryckningar och automatisk bortavaro via FRITZ!Box. Med de problem som uppstått i praktiken."
date: "2026-10-06"
kategorie: "Home Assistant och IoT"
timeToRead: "13 min läsning"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant
slug: "anslut-digitalstrom-till-home-assistant-lokal-integration-och-automationer"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
translationSourceHash: 36ff960cb636859ab9f7ed1cccb3152c61e6657a074546cef23685dbde2ec8fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:49:11.496Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/anslut-digitalstrom-till-home-assistant-lokal-integration-och-automationer
---

I digitalSTROM-installationer är lampor, persienner och andra förbrukare anslutna till klämmor i säkringsskåpet och styrs via tryckknappar och en digitalSTROM-server (dSS20). När ytterligare system som Philips Hue, Sonos och en FRITZ!Box tillkommer är det naturligt att samla allt lokalt i Home Assistant, utan molnkonto och utan att spara dSS-lösenordet i Home Assistant. Därför har jag skrivit en liten integration och publicerat den tillsammans med passande automationer som [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under MIT-licens.

Kort slutsats: dSS erbjuder ett användbart JSON-API med händelser. Den som använder det varsamt får tryckknappar, scener och aktiviteterna Gå och Komma till Home Assistant utan fördröjning och kan därmed utlösa automationer som digitalSTROM inte känner till på egen hand.

## Så fungerar integrationen

dSS tillhandahåller sitt API på port 8080 via HTTPS, med ett självsignerat certifikat. För program erbjuder digitalSTROM app-token: Programmet begär en token, användaren godkänner den i Configurator under System > Access authorization, och programmet loggar sedan in med den. dSS-lösenordet stannar hos användaren.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-s` | ingen förloppsindikator |
| `-k` | acceptera dSS:s självsignerade certifikat |
| `requestApplicationToken` | skapar en app-token som måste godkännas i Configurator |
| `loginApplication` | byter den godkända app-token mot en session-token |
| `event/subscribe` | prenumererar på en händelse (`callScene`, `stateChange`, `buttonClick` …) under ett fritt valt ID |
| `event/get` med `timeout` | väntar upp till 30 sekunder på nya händelser (long poll) |

</details>

Integrationen håller en sådan long-poll-anslutning öppen permanent. Varje scen som utlöses av en tryckknapp, appen eller Configurator kommer in som en händelse. Tillstånd som ”ljuset i rummet på” eller närvaro finns i dSS:s cache under `/usr/states` och kan hämtas utan att belasta klämmorna.

Direkta avfrågningar av utgångsvärden (`device/getOutputValue`) går däremot via dS485-bussen till klämman och tar en halv till en hel sekund per värde. Många sådana avfrågningar med korta intervall kan överbelasta mätarna. Integrationen läser därför endast persiennpositioner och Joker-utgångar direkt, var 15:e minut och ungefär en minut efter en körning.

| Plattform | Innehåll |
|---|---|
| `light` | ett ljus per rum, styrt via rumsscener som tryckknapparna; ljusstyrka för dimrade klämmor, endast på/av för switchade |
| `cover` | persienner med upp, ned, stopp och position |
| `scene` | stämningar namngivna i Configurator |
| `button` | Gå (scen 72) och Komma (scen 71) |
| `binary_sensor` | närvaro, vindlarm, rörelsedetektorer, Joker-utgångar |
| `sensor` | total förbrukning |

Dessutom rapporterar integrationen varje rumsscen samt Gå och Komma som händelsen `digitalstrom_local_event` till Home Assistant. Därmed kan 2× eller 4× tryckningar på en vanlig ljusströmbrytare tilldelas fritt.

## Automationer som Blueprints

Följande automationer finns som Blueprints i repositoryt och kan importeras till Home Assistant via importknappen i README.

### Rörelsestyrt ljus som inte släcker manuellt tänd belysning

Vid rörelse tänds ljuset och släcks igen efter en inställbar tid utan rörelse. Valfritt gäller detta bara under en ljusstyrketröskel, endast inom ett tidsfönster eller dimrat på natten. Om någon tänder ljuset med tryckknappen förblir det tänt. En hjälpbrytare (`input_boolean`) minns om automationen tände ljuset; endast då släcker den det igen.

En detalj märks först i drift: Utlösaren ”detektorn har varit stilla i 2 minuter” är en räknare som Home Assistant förkastar vid varje omstart. Om detektorn senast registrerade rörelse före omstarten sker ingen ny övergång till ”stilla”, och ljuset förblir tänt. Blueprinten kontrollerar därför dessutom varje minut om ett ljus som den tänt lyser trots att detektorn varit stilla tillräckligt länge.

### Släck ljuset när rummet lämnas

Avstängningstiden för det rörelsestyrda ljuset är en kompromiss: Är den för kort släcks ljuset medan någon står stilla i rummet; är den för lång fortsätter det lysa i flera minuter efter att personen lämnat rummet. Därför använder en annan Blueprint en andra detektor utanför rummet, vanligtvis i hallen. Om den registrerar rörelse, detektorn i rummet nyligen (inom 30 sekunder) också har registrerat rörelse och det sedan förblir stilla där, släcks ljuset i rummet direkt. Med Hue-detektorer, som rapporterar ”stilla” omkring 10 sekunder efter den senaste rörelsen, blir det cirka 10 till 15 sekunder efter att man gått ut.

För att ingen ska sitta i mörkret gäller regeln endast under vissa villkor: Ljuset måste ha tänts av rörelseautomation, ett duschläge får inte vara aktivt och högst en person får vara hemma. Antalet personer beräknas utifrån mobiltelefonerna som Home Assistant känner till som personer. Villkoret på 30 sekunder skyddar dessutom mot fallet där någon är stilla i rummet under en längre tid medan en annan person går genom hallen.

### Duschläge med 2× tryckningar

I duschen upptäcker en rörelsedetektor oftast ingen, och efter den inställda tiden blir det mörkt. 2× tryckningar på badrummets tryckknapp utlöser stämning 2 (scen 17) i digitalSTROM. Automationerna känner igen denna scen i rummet och aktiverar ett duschläge som blockerar avstängning. Rörelser under de första 2 minuterna ignoreras (klä av sig, kliva in). Den första rörelsen därefter avslutar duschläget, och från då gäller den vanliga avstängningstiden igen. Efter en inställbar maximal tid (standard 20 minuter) avslutas det automatiskt.

### 4× tryckningar startar Sonos

När man trycker flera gånger snabbt på en tryckknapp växlar digitalSTROM igenom stämningarna i tur och ordning: scen 5, 17, 18 och vid den fjärde tryckningen 19. Om scen 19 ställs in på ”Ändra inte utgång” i Configurator för alla lampor, är den fri för andra ändamål. En Blueprint reagerar på denna scen, släcker rumsljuset och startar Sonos-högtalaren i rummet eller, i rum utan högtalare, flera tillsammans.

Skriptet för detta tar hänsyn till tre saker: Om en högtalare redan spelar läggs det nya rummet till i dess grupp så att uppspelningen förblir synkroniserad. Om inget kan återupptas spelas en radiostation från Radio Browser-katalogen. Och före varje start ställs samma startvolym in, annars spelar det ena rummet tyst och det andra lika högt som någon senast lyssnade på musik.

### Gå och Komma via FRITZ!Box

Integrationen FRITZ!Box Tools rapporterar om en mobiltelefon finns i WLAN; den utvärderar boxens enhetslista och täcker därmed 2,4 GHz-nät, 5 GHz-nät och LAN samtidigt. När ingen längre är hemma och Gå inte har tryckts utlöser en Blueprint Gå. När man kommer hem följer Komma och, om det är mörkt, ett välkomstljus.

Som komplement kan en automation släcka lampor från andra system (till exempel Hue) och pausa Sonos vid Gå. digitalSTROM-lamporna släcks av dSS självt.

## Planritning som dashboard

En överskådlig visning i Home Assistant är en planritning med det inbyggda kortet `picture-elements`. För varje rum ligger en genomskinlig SVG över planen, som ersätts av en lätt gul när ljuset är tänt; ett tryck på rummet växlar ljuset. Lampor, högtalare, persienner och detektorer placeras som symboler på sina platser. Instruktionerna med exempelkonstruktion finns i repositoryt under `docs/floor-plan-dashboard.md`.

## Problem från praktiken

| Problem | Orsak | Lösning |
|---|---|---|
| Gå-knappen saknar effekt | dSS är inte i nätverket, aktiviteter över flera strömkretsar går via servern | nätverksanslutningen till dSS återställdes |
| Persienner reagerar inte | vindlarmet var ”aktivt”, trots att ingen vindsensor fanns | scen 87 (”ingen vind”) utlöstes för alla rum |
| Persienner kör inte upp vid Gå | scen 72 står på ”Ändra inte utgång” (`dontCare`) | `device/setSceneMode` med `dontCare=0`; värdet `false` accepterades men ignorerades |
| Ljuset går inte att dimra | lysrör anslutet till en switchad klämma (utgångsläge 35) | integrationen känner igen switchade klämmor och erbjuder där endast på/av |
| Ljuset förblir tänt efter omstart | räknaren för ”stilla sedan X minuter” försvinner vid omstart | ytterligare kontroll varje minut |
| Mobiltelefonen räknas som frånvarande | iPhone använder en växlande privat WLAN-adress och kopplar kort bort WLAN i viloläge | privat WLAN-adress satt till ”Fast”, respittid 10 minuter |
| DECT sänder hela tiden | ”DECT Eco” är inte tillgängligt så snart en FRITZ! Smart Home-enhet är registrerad | avstå från DECT-uttag om DECT Eco önskas |

En enhetssökning på Hue-bryggan tar med alla enheter som just då är i parkopplingsläge. Kontrollera sedan i enhetslistan att endast dina egna lampor har lagts till.

Klämmornas scener-tabeller kan läsas ut och ändras via API:t, till exempel med `device/getSceneMode` och `device/saveScene`. Vart och ett av dessa anrop går via bussen. Säkerhetskopiera de gamla värdena före ändringar, till exempel i en CSV-fil, så att de kan återställas vid behov.

## Källor

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration och Blueprints från denna artikel, MIT-licens.

2.  [digitalSTROM: Handbok för användning och inställning](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktiviteten Gå (håll in i 3 sekunder), inställningen ”Ändra inte utgång” i kapitel 3.5.2.

3.  [digitalSTROM: Dokumentation för klämmornas tryckknappar](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Fabriksinställningar för tryckknappsfunktioner per klämtyp.

4.  [digitalSTROM: Bruksanvisningar](https://www.digitalstrom.com/bedienungsanleitungen/): Översikt över handböcker, planerings- och installationsunderlag.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Närvarodetektering, WLAN-brytare, integrationsalternativ.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grund för planritnings-dashboarden.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import och användning av Blueprints.

8.  [FRITZ! Kunskapsdatabas: FRITZ!-uttag förlorar anslutningen](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Information om DECT-räckvidd och radiosändningseffekt för Smart Home-uttag.

9.  [Radio Browser](https://www.radio-browser.info/): Fritt katalog över radioströmmar, integrerad i Home Assistant som mediekälla.
