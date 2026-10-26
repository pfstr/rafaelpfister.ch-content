---
title: "Anslut digitalSTROM till Home Assistant: lokal integration och automationer"
navTitle: "digitalSTROM och HA"
description: "En lokal Home Assistant-integration för digitalSTROM-servern, med rörelsebelysning, duschläge, musik med 4× tryckningar och automatiskt läge för lämna hemmet via FRITZ!Box. Med problemen som uppstod i praktiken."
date: "2026-10-06"
kategorie: "Home Assistant och IoT"
timeToRead: "13 min lästid"
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
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:24:05.019Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/anslut-digitalstrom-till-home-assistant-lokal-integration-och-automationer
---

I digitalSTROM-installationer är lampor, persienner och andra förbrukare anslutna till klämmor i elcentralen och styrs via tryckknappar och en digitalSTROM-server (dSS20). När ytterligare system som Philips Hue, Sonos och en FRITZ!Box tillkommer är det naturligt att samla allt lokalt i Home Assistant, utan molnkonto och utan att lagra dSS-lösenordet i Home Assistant. För detta har jag skrivit en liten integration och publicerat den tillsammans med lämpliga automationer som [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) under MIT-licens.

Kort slutsats: dSS erbjuder ett användbart JSON-API med händelser. Den som använder det varsamt får tryckknappar, scener samt aktiviteterna Lämna hemmet och Komma hem till Home Assistant utan fördröjning och kan därmed utlösa automationer som digitalSTROM inte känner till på egen hand.

## Så fungerar integrationen

dSS tillhandahåller sitt API på port 8080 via HTTPS, med ett självsignerat certifikat. digitalSTROM har app-token för program: Programmet begär en token, användaren godkänner den i Configurator under System > Åtkomstbehörighet och därefter loggar programmet in med den. dSS-lösenordet stannar hos användaren.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-s` | ingen förloppsindikator |
| `-k` | accepterar dSS:s självsignerade certifikat |
| `requestApplicationToken` | skapar en app-token som måste godkännas i Configurator |
| `loginApplication` | byter den godkända app-token mot en session-token |
| `event/subscribe` | prenumererar på en händelse (`callScene`, `stateChange`, `buttonClick` …) under ett fritt valt ID |
| `event/get` med `timeout` | väntar upp till 30 sekunder på nya händelser (long poll) |

</details>

Integrationen håller en sådan long-poll-anslutning öppen permanent. Varje scen som utlöses av en tryckknapp, appen eller Configurator kommer in som en händelse. Tillstånd som ”ljuset i rummet är på” eller närvaro finns i dSS:s cache under `/usr/states` och kan hämtas utan att belasta klämmorna.

Direkta förfrågningar om utgångsvärden (`device/getOutputValue`) går däremot via dS485-bussen till klämman och tar en halv till en hel sekund per värde. Många sådana förfrågningar med korta intervall kan överbelasta mätarna. Integrationen läser därför endast persiennpositioner och Joker-utgångar direkt, var 15:e minut och ungefär en minut efter en körning.

| Plattform | Innehåll |
|---|---|
| `light` | ett ljus per rum, styrt via rumsscener som tryckknapparna; ljusstyrka för dimrade klämmor, endast på/av för switchade |
| `cover` | persienner med upp, ned, stopp och position |
| `scene` | stämningar namngivna i Configurator |
| `button` | Lämna hemmet (scen 72) och Komma hem (scen 71) |
| `binary_sensor` | närvaro, vindlarm, rörelsedetektor, Joker-utgångar |
| `sensor` | total förbrukning |

Dessutom rapporterar integrationen varje rumsscen samt Lämna hemmet och Komma hem som händelsen `digitalstrom_local_event` till Home Assistant. Därmed kan 2× eller 4× tryckningar på en vanlig ljusströmbrytare tilldelas fritt.

## Automationer som Blueprints

Följande automationer ingår som Blueprints i repositoriet och kan importeras till Home Assistant via importknappen i README.

### Rörelsebelysning som inte släcker ljus som tänds manuellt

Vid rörelse tänds ljuset och släcks igen efter en inställbar tid utan rörelse. Alternativt gäller detta endast under en ljusstyrketröskel, endast inom ett tidsfönster eller dimmat på natten. Om någon tänder ljuset med tryckknappen förblir det tänt. En hjälpbrytare (`input_boolean`) kommer ihåg om automationen tände ljuset; endast då släcker den det igen.

En detalj märks först under drift: Utlösaren ”detektorn har varit lugn i 2 minuter” är en räknare som Home Assistant förkastar vid varje omstart. Om detektorn senast registrerade rörelse före omstarten sker ingen ny ändring till ”lugn”, och ljuset förblir tänt. Blueprinten kontrollerar därför dessutom varje minut om ett ljus som den tänt lyser trots att detektorn varit lugn tillräckligt länge.

### Släck ljuset när rummet lämnas

Rörelsebelysningens släcktid är en kompromiss: Är den för kort släcks ljuset medan någon står stilla i rummet; är den för lång lyser det i flera minuter efter att personen gått. Därför använder en annan Blueprint en andra detektor utanför rummet, vanligen i hallen. Om den registrerar rörelse, detektorn i rummet kort dessförinnan (inom 30 sekunder) också registrerade rörelse och det sedan förblir lugnt där, släcks ljuset i rummet omedelbart. Med Hue-detektorer, som rapporterar ”lugn” cirka 10 sekunder efter den senaste rörelsen, blir det omkring 10 till 15 sekunder efter att man gått ut.

För att ingen ska sitta i mörkret tillämpas regeln bara under vissa förutsättningar: Ljuset måste ha tänts av rörelseautomationen, ett duschläge får inte vara aktivt och högst en person får vara hemma. Antalet personer bestäms utifrån de telefoner som Home Assistant känner som personer. Villkoret på 30 sekunder skyddar dessutom mot fallet där någon står stilla länge i rummet medan en annan person går genom hallen.

### Duschläge med 2× tryckningar

I duschen registrerar en rörelsedetektor oftast ingen, och efter den inställda tiden blir det mörkt. 2× tryckningar på badrummets knapp utlöser stämning 2 (scen 17) i digitalSTROM. Automationen känner igen denna scen i rummet och aktiverar ett duschläge som blockerar avstängningen. Rörelser under de första 2 minuterna ignoreras (klä av sig, gå in). Den första rörelsen därefter avslutar duschläget och från då gäller den vanliga släcktiden igen. Efter en inställbar maximal tid (standard 20 minuter) avslutas det automatiskt.

### 4× tryckningar startar Sonos

Om man trycker flera gånger snabbt på en knapp växlar digitalSTROM genom stämningarna i tur och ordning: scen 5, 17, 18 och scen 19 vid den fjärde tryckningen. Om scen 19 ställs in på ”Ändra inte utgången” för alla lampor i Configurator är den fri för andra ändamål. En Blueprint reagerar på denna scen, släcker rummets ljus och startar Sonos-högtalaren i rummet eller, i rum utan högtalare, flera tillsammans.

Skriptet tar hänsyn till tre saker: Om en högtalare redan spelar läggs det nya rummet till i dess grupp så att uppspelningen förblir synkroniserad. Om inget kan återupptas spelas en radiostation från Radio Browser-katalogen. Och före varje start ställs samma startvolym in, annars spelar det ena rummet svagt och det andra lika högt som någon senast lyssnade på musik.

### Lämna hemmet och Komma hem via FRITZ!Box

Integrationen FRITZ!Box Tools rapporterar om en telefon finns i WLAN; den utvärderar enhetens lista i routern och omfattar därmed 2,4 GHz-nät, 5 GHz-nät och LAN samtidigt. Om ingen längre är hemma och Lämna hemmet inte har aktiverats, utlöser en Blueprint Lämna hemmet. När någon kommer hem följer Komma hem och, om det är mörkt, ett välkomstljus.

Som komplement kan en automation släcka lamporna i andra system (till exempel Hue) och pausa Sonos vid Lämna hemmet. digitalSTROM-lamporna släcker dSS själv.

## Planritning som dashboard

En överskådlig visning i Home Assistant är en planritning med det inbyggda kortet `picture-elements`. För varje rum ligger en transparent SVG över planen som ersätts av en lätt gul när ljuset är tänt; ett tryck på rummet växlar ljuset. Lampor, högtalare, persienner och detektorer finns som symboler på sina platser. Anvisningarna med exempelkonfiguration finns i repositoriet under `docs/floor-plan-dashboard.md`.

## Problem från praktiken

| Problem | Orsak | Lösning |
|---|---|---|
| Knappen Lämna hemmet saknar effekt | dSS finns inte i nätverket, aktiviteter över flera strömkretsar går via servern | Återställde dSS:s nätverksanslutning |
| Persienner reagerar inte | Vindlarmet var ”aktivt” trots att ingen vindsensor finns | Utlöste scen 87 (”ingen vind”) för alla rum |
| Persienner körs inte upp vid Lämna hemmet | Scen 72 på ”Ändra inte utgången” (`dontCare`) | `device/setSceneMode` med `dontCare=0`; värdet `false` accepterades men ignorerades |
| Ljuset kan inte dimras | Lysrör på en switchad klämma (utgångsläge 35) | Integrationen känner igen switchade klämmor och erbjuder där endast på/av |
| Ljuset förblir tänt efter omstart | Räknaren för ”lugn i X minuter” försvinner vid omstart | Ytterligare kontroll varje minut |
| Telefonen räknas som frånvarande | iPhone använder en växlande privat WLAN-adress och kopplar kort bort WLAN i viloläge | Privat WLAN-adress till ”Fast”, respittid 10 minuter |
| DECT sänder hela tiden | ”DECT Eco” är inte tillgängligt när en FRITZ!-smart-home-enhet är registrerad | Avstå från DECT-uttag om DECT Eco önskas |

En enhetssökning på Hue-bryggan lägger till alla enheter som just nu är i parkopplingsläge. Kontrollera därefter i enhetslistan att endast dina egna lampor har lagts till.

Klämmornas scentabeller kan läsas ut och ändras via API:et, till exempel med `device/getSceneMode` och `device/saveScene`. Vart och ett av dessa anrop går via bussen. Säkerhetskopiera de gamla värdena före ändringar, till exempel i en CSV-fil, så att de kan återställas vid behov.

## Ändringshistorik

- **25 oktober 2026:** Uppdaterade källänken till den nya adressen för FRITZ!-kunskapsdatabasen.

## Källor

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration och Blueprints från denna artikel, MIT-licens.

2.  [digitalSTROM: Handbok för användning och inställning](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktiviteten Lämna hemmet (håll in i 3 sekunder), inställningen ”Ändra inte utgången” i kapitel 3.5.2.

3.  [digitalSTROM: Dokumentation om klämmornas knappar](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Fabriksinställningar för knappfunktioner per klämmtyp.

4.  [digitalSTROM: Bruksanvisningar](https://www.digitalstrom.com/bedienungsanleitungen/): Översikt över handböcker, planerings- och installationsunderlag.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Närvarodetektering, WLAN-brytare, integrationsalternativ.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grund för planritningsdashboarden.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import och användning av Blueprints.

8.  [FRITZ! Kunskapsdatabas: FRITZ!-uttag tappar anslutningen](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Information om DECT-räckvidd och radiosändningseffekt för smart-home-uttag.

9.  [Radio Browser](https://www.radio-browser.info/): Fri katalog över radioströmmar, integrerad i Home Assistant som mediekälla.
