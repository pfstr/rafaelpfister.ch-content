---
title: "Anslut digitalSTROM till Home Assistant: lokal integration och automationer"
navTitle: "digitalSTROM och HA"
description: "En lokal Home Assistant-integration för digitalSTROM-servern, med rörelsebelysning, duschläge, musik med 4× tryckningar och automatisk bortgång via FRITZ!Box. Med de problem som uppstod i praktiken."
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
url: https://rafaelpfister.ch/sv/blog/anslut-digitalstrom-till-home-assistant-lokal-integration-och-automationer
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:25:27.027Z
translationReview: automatic
---

I digitalSTROM-installationer är lampor, persienner och andra förbrukare anslutna till klämmor i elcentralen och styrs via tryckknappar och en digitalSTROM-server (dSS20). När ytterligare system som Philips Hue, Sonos och en FRITZ!Box tillkommer är det naturligt att samla allt lokalt i Home Assistant, utan molnkonto och utan att lagra dSS-lösenordet i Home Assistant. För detta har jag skrivit en liten integration och publicerat den tillsammans med passande automationer som [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under MIT-licens.

Kort sammanfattat: dSS erbjuder ett användbart JSON-API med händelser. Den som använder det varsamt får tryckknappar, scener samt aktiviteterna Gå och Komma till Home Assistant utan fördröjning och kan därmed utlösa automationer som digitalSTROM inte känner till på egen hand.

## Så fungerar integrationen

dSS tillhandahåller sitt API på port 8080 via HTTPS, med ett självsignerat certifikat. För program erbjuder digitalSTROM app-token: Programmet begär en token, användaren godkänner den i Configurator under System > Åtkomstbehörighet och programmet loggar sedan in med den. dSS-lösenordet stannar hos användaren.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Förklaring av alternativen</summary>

| Alternativ | Funktion |
|---|---|
| `-s` | ingen förloppsindikator |
| `-k` | acceptera dSS:s självsignerade certifikat |
| `requestApplicationToken` | skapar en app-token som måste godkännas i Configurator |
| `loginApplication` | byter den godkända app-token mot en session-token |
| `event/subscribe` | prenumererar på en händelse (`callScene`, `stateChange`, `buttonClick` …) under ett valfritt ID |
| `event/get` med `timeout` | väntar upp till 30 sekunder på nya händelser (long poll) |

</details>

Integrationen håller en sådan long poll-anslutning öppen permanent. Varje scen som utlöses av en tryckknapp, appen eller Configurator kommer in som en händelse. Tillstånd som ”ljus på i rummet” eller närvaro finns i dSS:s cache under `/usr/states` och kan hämtas utan att belasta klämmorna.

Direkta frågor om utgångsvärden (`device/getOutputValue`) går däremot via dS485-bussen hela vägen till klämman och tar en halv till en sekund per värde. Många sådana frågor med korta intervall kan överbelasta mätarna. Integrationen läser därför endast persiennpositioner och Joker-utgångar direkt, var 15:e minut och ungefär en minut efter en körning.

| Plattform | Innehåll |
|---|---|
| `light` | en lampa per rum, styrd via rumsscener som tryckknapparna; ljusstyrka för dimrade klämmor, endast på/av för switchade |
| `cover` | persienner med upp, ned, stopp och position |
| `scene` | stämningar namngivna i Configurator |
| `button` | Gå (scen 72) och Komma (scen 71) |
| `binary_sensor` | närvaro, vindlarm, rörelsedetektorer, Joker-utgångar |
| `sensor` | total förbrukning |

Dessutom rapporterar integrationen varje rumsscen samt Gå och Komma som händelsen `digitalstrom_local_event` till Home Assistant. Därmed kan 2× eller 4× tryckningar på en vanlig ljusknapp tilldelas fritt.

## Automationer som Blueprints

Följande automationer ingår som Blueprints i repositoriet och kan läggas till i Home Assistant via importknappen i README.

### Rörelsebelysning som inte släcker manuellt tänd belysning

Vid rörelse tänds ljuset och släcks igen efter en inställbar tid utan rörelse. Valfritt gäller detta endast under en ljusstyrketröskel, endast inom ett tidsfönster eller dämpat på natten. Om någon tänder ljuset med tryckknappen förblir det tänt. En hjälpbrytare (`input_boolean`) minns om automationen tände ljuset; endast då släcker den det också igen.

En detalj märks först under drift: Utlösaren ”detektorn har varit lugn i 2 minuter” är en räknare som Home Assistant förkastar vid varje omstart. Om detektorn senast registrerade rörelse före omstarten sker ingen ny övergång till ”lugn”, och ljuset förblir tänt. Blueprinten kontrollerar därför dessutom varje minut om ett ljus som den har tänt fortfarande lyser trots att detektorn har varit lugn tillräckligt länge.

### Släck ljuset när rummet lämnas

Avstängningstiden för rörelsebelysningen är en kompromiss: Är den för kort släcks ljuset medan någon står stilla i rummet; är den för lång fortsätter det lysa i flera minuter efter att rummet lämnats. En annan Blueprint använder därför en andra detektor utanför rummet, vanligtvis i hallen. Om denna registrerar rörelse, detektorn i rummet nyligen (inom 30 sekunder) också har registrerat rörelse och det sedan förblir lugnt där, släcks ljuset i rummet omedelbart. Med Hue-detektorer, som rapporterar ”lugn” ungefär 10 sekunder efter den senaste rörelsen, blir det cirka 10 till 15 sekunder efter att man gått ut.

För att ingen ska sitta i mörkret gäller regeln endast under vissa villkor: Ljuset måste ha tänts av rörelseautomation, ett duschläge får inte vara aktivt och högst en person får vara hemma. Antalet personer avgörs av mobilerna som Home Assistant känner till som personer. 30-sekundersvillkoret skyddar dessutom mot fallet att någon står stilla i rummet under längre tid medan en annan person går genom hallen.

### Duschläge med 2× tryckningar

I duschen registrerar en rörelsedetektor oftast ingen, och efter den inställda tiden blir det mörkt. 2× tryckningar på badrumsknappen utlöser stämning 2 (scen 17) i digitalSTROM. Automation känner igen denna scen i rummet och aktiverar ett duschläge som blockerar avstängning. Rörelser under de första 2 minuterna ignoreras (avklädning, att stiga in). Den första rörelsen därefter avslutar duschläget, och från då gäller åter den normala avstängningstiden. Efter en inställbar maximal tid (standard 20 minuter) avslutas det automatiskt.

### 4× tryckningar startar Sonos

När man trycker snabbt flera gånger på en knapp växlar digitalSTROM genom stämningarna i tur och ordning: scen 5, 17, 18 och 19 vid den fjärde tryckningen. Om scen 19 i Configurator ställs in på ”Ändra inte utgång” för alla lampor är den ledig för andra ändamål. En Blueprint reagerar på denna scen, släcker rummets ljus och startar Sonos-högtalaren i rummet eller, i rum utan högtalare, flera högtalare tillsammans.

Skriptet tar hänsyn till tre saker: Om en högtalare redan spelar läggs det nya rummet till i dess grupp så att uppspelningen förblir synkroniserad. Om inget kan återupptas spelas en radiostation från Radio Browser-katalogen. Och före varje start ställs samma startvolym in, annars spelar det ena rummet tyst och det andra lika högt som någon senast lyssnade på musik.

### Gå och Komma via FRITZ!Box

Integrationen FRITZ!Box Tools rapporterar om en mobil är ansluten till WLAN; den utvärderar boxens enhetslista och täcker därmed 2,4 GHz-nät, 5 GHz-nät och LAN samtidigt. Om ingen längre är hemma och Gå inte har aktiverats utlöser en Blueprint Gå. När någon kommer hem följer Komma och, om det är mörkt, en välkomstbelysning.

Som komplement kan en automation släcka lampor i andra system (till exempel Hue) och pausa Sonos vid Gå. digitalSTROM-lamporna släcks av dSS självt.

## Planritning som dashboard

En tydlig visning i Home Assistant är en planritning med det inbyggda kortet `picture-elements`. I varje rum ligger en transparent SVG över planen, som ersätts av en svagt gul när ljuset är tänt; ett tryck på rummet växlar ljuset. Lampor, högtalare, persienner och detektorer placeras som symboler på sina respektive platser. Instruktionerna med exempelkonfiguration finns i repositoriet under `docs/floor-plan-dashboard.md`.

## Problem från praktiken

| Problem | Orsak | Lösning |
|---|---|---|
| Gå-knappen har ingen effekt | dSS är inte ansluten till nätverket, aktiviteter över flera strömkretsar går via servern | dSS:s nätverksanslutning återställdes |
| Persienner reagerar inte | Vindlarm var inställt på ”aktivt” trots att ingen vindsensor finns | Scen 87 (”ingen vind”) utlöstes för alla rum |
| Persienner körs inte upp vid Gå | Scen 72 är inställd på ”Ändra inte utgång” (`dontCare`) | `device/setSceneMode` med `dontCare=0`; värdet `false` accepterades men ignorerades |
| Ljuset kan inte dimras | Lysrör anslutet till en switchad klämma (utgångsläge 35) | Integrationen identifierar switchade klämmor och erbjuder där endast på/av |
| Ljuset förblir tänt efter omstart | Räknaren för ”lugn sedan X minuter” försvinner vid omstart | ytterligare kontroll varje minut |
| Mobilen räknas som frånvarande | iPhone använder en växlande privat WLAN-adress och kopplar kort bort WLAN i viloläge | Privat WLAN-adress inställd på ”Fast”, respittid 10 minuter |
| DECT sänder hela tiden | ”DECT Eco” är inte tillgängligt så snart en FRITZ!-smart-home-enhet är registrerad | avstå från DECT-uttag om DECT Eco önskas |

En enhetssökning på Hue-bryggan lägger till alla enheter som just då är i parkopplingsläge. Kontrollera därefter i enhetslistan att endast dina egna lampor har lagts till.

Klämmornas scentabeller kan läsas ut och ändras via API:t, till exempel med `device/getSceneMode` och `device/saveScene`. Vart och ett av dessa anrop går via bussen. Säkerhetskopiera de gamla värdena före ändringar, exempelvis i en CSV-fil, så att de kan återställas vid behov.

## Källor

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration och Blueprints från denna artikel, MIT-licens.

2.  [digitalSTROM: Handbok för användning och inställning](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktiviteten Gå (håll inne i 3 sekunder), inställningen ”Ändra inte utgång” i kapitel 3.5.2.

3.  [digitalSTROM: Dokumentation för klämmornas knappar](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Fabriksinställningar för knappfunktioner per klämmtyp.

4.  [digitalSTROM: Bruksanvisningar](https://www.digitalstrom.com/bedienungsanleitungen/): Översikt över handböcker, planerings- och installationsunderlag.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Närvarodetektering, WLAN-brytare, integrationsalternativ.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grund för planritningsdashboarden.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import och användning av Blueprints.

8.  [FRITZ! kunskapsdatabas: FRITZ!-uttag tappar anslutningen](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Information om DECT-räckvidd och radiosändareffekt för smart-home-uttag.

9.  [Radio Browser](https://www.radio-browser.info/): Fri katalog över radioströmmar, integrerad som mediekälla i Home Assistant.
