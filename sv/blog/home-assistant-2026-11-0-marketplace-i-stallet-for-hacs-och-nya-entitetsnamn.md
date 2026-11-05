---
title: "Home Assistant 2026.11.0: Marketplace i stället för HACS och nya entitetsnamn"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 integrerar HACS som en inbyggd Marketplace, bygger om entitetsnamn utifrån enhet och entitet och medför många bakåtinkompatibla ändringar. Kontrollera före uppdateringen dashboards, mallar och närvarodetektering via FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant och IoT"
timeToRead: "6 min läsning"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-i-stallet-for-hacs-och-nya-entitetsnamn"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/sv/blog/home-assistant-2026-11-0-marketplace-i-stallet-for-hacs-och-nya-entitetsnamn
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:25:18.390Z
translationReview: required
---

# Home Assistant 2026.11.0: Marketplace i stället för HACS och nya entitetsnamn

Home Assistant 2026.11.0 släpptes den 4 november 2026 och följer på 2026.10.4. Utgåvan integrerar den tidigare Home Assistant Community Store (HACS) som Marketplace direkt i Home Assistant, ändrar hur entitetsnamn skapas och innehåller en lång lista med bakåtinkompatibla ändringar, däribland ändringar av Device Trackers från FRITZ!Box Tools och inloggning via icke-lokala adresser. Enligt releaseblogginlägget tillkommer 38 nya integrationer. Release Notes nämner ingen säkerhetskorrigering med CVE. Utgåvan riktar sig till alla som driver Home Assistant, särskilt installationer med Custom Integrations från HACS, såsom de som används i artiklarna om [Midea PortaSplit](/blog/midea-portasplit-home-assistant) och [digitalSTROM](/blog/digitalstrom-home-assistant).

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Månadsrelease | HACS blir Marketplace, nya entitetsnamn, Device Trackers från nätverksskannrar alltid med enhet, autentiseringskontroll vid icke-lokala adresser, ingen `deps`-mapp längre, 38 nya integrationer |

## Marketplace: HACS blir en del av Home Assistant

Med 2026.11 blir HACS enligt releaseblogginlägget en del av Home Assistant och heter nu Marketplace. Administratörer hittar den under Inställningar > Marketplace och installerar där integrationer, dashboard-kort, teman och mallar; något GitHub-konto behövs inte. Efter installation av en integration behövs ingen omstart, men uppdateringar kräver fortfarande en omstart.

Home Assistant migrerar befintliga HACS-installationer vid första starten efter uppdateringen. Installerade projekt förblir installerade och fortsätter att få uppdateringar, Custom Repositories och uppdateringsentiteter tas över. Enligt tillverkaren krävs inget ingripande. HACS-installationsprogrammet upphör med Home Assistant 2026.11, en separat HACS-installation är inte längre möjlig i denna version och den tidigare HACS får inga nya utgåvor. Rapportera i fortsättningen fel i Home Assistants issue-tracker.

Vid första öppningen visar Marketplace ett meddelande om att communityprojekt varken testas, granskas eller stöds av Home Assistant-projektet. Du måste bekräfta meddelandet innan du installerar eller uppdaterar. Projekt utanför katalogen kan läggas till via ”Add from link”; det kräver en GitHub-anslutning via Device Sign-in, som endast läser offentlig information.

För Midea PortaSplit gäller detta Custom Integration `Midea AC LAN` och kortet `apexcharts-card`, om båda är installerade via HACS. Enligt informationen om migreringen förblir de installerade och fortsätter att få uppdateringar. Supply-chain-risken för Custom Integrations, som artikeln om [att säkra PortaSplit](/blog/midea-portasplit-home-assistant-absichern) beskriver, kvarstår med Marketplace, vilket tillverkarens riskmeddelande visar. Kontrollera efter uppdateringen under Inställningar > Marketplace att alla projekt och Custom Repositories som tidigare installerats via HACS finns kvar.

## Nya entitetsnamn

Från och med 2026.11 består entitetsnamn av entitetens namn och dess kontext, vanligtvis enhetsnamnet. Namn som du själv har angett migreras enligt samma regler, och det gamla namnet finns kvar som alias. Kort, badges och rader utan eget namn visar nu enhetsnamnet följt av entitetsnamnet i stället för Friendly Name. Entity-ID:n, automationer och skript påverkas enligt Release Notes inte.

För mallar rekommenderar tillverkaren funktionerna `entity_name()`, `device_name()` och `area_name()`; attributet `friendly_name` ska senare markeras som föråldrat. Kontrollera efter uppdateringen dina dashboards för ändrade etiketter och anpassa mallar som utvärderar `friendly_name`.

## FRITZ!Box Tools och Device Trackers

Trackers från nätverksskannrar – däribland uttryckligen FRITZ!Box Tools, UniFi Network, NETGEAR och Nmap Tracker – har nu alltid en enhet. Omdöpta tracker-entiteter får enhetsnamnet i början av sitt namn. Nya trackers från dessa integrationer skapas alltid inaktiverade; tidigare var de aktiva om en annan integration redan kände till enhetens MAC-adress. Befintliga trackers behåller sitt tillstånd (aktiverat eller inaktiverat).

För närvaroautomationer via FRITZ!Box, såsom det automatiska lämnandet som beskrivs i [digitalSTROM-artikeln](/blog/digitalstrom-home-assistant), innebär detta att befintliga trackers fortsätter att fungera. Trackern för en nytillagd smartphone måste du däremot först aktivera innan den levererar ett tillstånd i automationer.

Changeloggen listar dessutom flera korrigeringar för FRITZ!Box Tools, bland annat för skapandet av WLAN-brytarna, filtrering av FRITZ!Box-värdar utan IP-adress från Device Trackers, en omladdningsloop på enheter utan trafikdata och ett tydligare felmeddelande vid ogiltig inloggning i installationsdialogen. Integrationen FRITZ!Box Call Monitor rapporterar nu ett konfigurationsfel vid saknade behörigheter.

## Midea och digitalSTROM

Förbättringarna för Midea i releaseblogginlägget gäller den officiella integrationen `Midea`, som har ingått i Core sedan Home Assistant 2026.8. Custom Integration `Midea AC LAN` (domän `midea_ac_lan`) är separat från denna. Nytt är styrningar och en förloppssensor för tvättmaskiner och torktumlare, dörrsensorer för kylskåp, varmvattenberedarenheter, en på/av-brytare för luftrenare och en brytare för ljudsignalen på avfuktare. För luftkonditioneringsenheter nämner changeloggen en knapp `reset_filter`, med vilken filtret kan markeras som rengjort. Integrationen har nått kvalitetsnivån Platinum. Enligt integrationssidan fungerar styrning och statusfrågor lokalt; molnet behövs endast för token och key från V3-enheter.

Changeloggen för 2026.11.0 innehåller ingen post om digitalSTROM. Integrationen som beskrivs i [digitalSTROM-artikeln](/blog/digitalstrom-home-assistant) är en Custom Integration; om den är ansluten via HACS som Custom Repository gäller den beskrivna överföringen till Marketplace.

## Ytterligare bakåtinkompatibla ändringar

Den fullständiga listan finns i releaseblogginlägget. För drift är framför allt följande punkter relevanta:

- **Auth:** Vid åtkomst via en icke-lokal IP-adress måste schema, värd och port i webbläsaren överensstämma med adressen som Home Assistant ser, annars misslyckas inloggningen med ”Invalid client id”. Den tidigare kontrollen accepterade sådana adresser på grund av ett fel så snart de innehöll en port. Ange i så fall din Reverse Proxy under Trusted Proxies i HTTP-integrationen eller använd ett domännamn.
- **Python-beroenden:** Home Assistant installerar inte längre paket i mappen `deps` i konfigurationskatalogen. Endast Core-installationer utanför en virtuell miljö påverkas; OS-, Supervised- och Container-installationer påverkas inte. Efter uppdateringen kan mappen `deps` raderas.
- **SMTP:** En oläsbar bilaga leder nu till ett fel; tidigare skickades meddelandet utan bilagan.
- **Telegram:** Den gamla `notify`-plattformen tas bort. Använd Telegram-bottens Notify-entiteter och åtgärden `notify.send_message`.
- **Z-Wave:** zwave-js-server 3.11.0 eller senare (schema 51), Z-Wave JS App från 1.9.0 respektive Z-Wave JS UI från v11.24.2 krävs.
- **Uptime Kuma:** Versioner före 2.0.0 stöds inte längre.
- **Philips Hue:** Ljuseffekten `"None"` tas bort, i stället gäller `"off"`.
- **Matter och Electra Smart:** `climate.turn_on` fortsätter med det senast aktiva läget. För ett specifikt läge använder du `climate.set_hvac_mode`.
- **Pollingintervall:** För Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato och Risco ignoreras ett eget intervall och standardvärdet används. Control4 frågar alltid var femte sekund; för ett annat intervall inaktiverar du polling och anropar `homeassistant.update_entity` från en automation.

Integrationen Flipr har tagits bort eftersom företaget bakom den har likviderats.

## Nya funktioner

Utöver Marketplace nämner releaseblogginlägget bland annat att enskilda entiteter, enheter, områden, våningar eller etiketter kan exkluderas från ett mål, kopiering och inklistring av dashboard-avsnitt, ett datumkort och ett alertkort med visuell redigerare. Energidashboarden kan härleda en energisensor från en effektsensor. För felsökning finns filtret ”Triggered by” i aktiviteten, live-loggar per integration och radering av statistik inom en tidsperiod. Backuper och medier kan sparas på lokala datalagringsenheter. Nytt stöd finns för Matter 1.5-kameror med WebRTC samt LoRaWAN (ChirpStack, The Things Network, SenseCAP) och Zigbee Green Power.

För alertkortet heter alternativet `color` nu `alert_color`; du behöver bara anpassa dashboards som du själv har skapat i YAML. Den nya integrationen Debug console kör Python-kod inom Home Assistant, har full åtkomst och måste därför aktiveras i `configuration.yaml`; endast administratörer kan använda den.

## Rekommenderat tillvägagångssätt

Målversionen är 2026.11.0. Läs före uppdateringen de bakåtinkompatibla ändringarna i releaseblogginlägget för de integrationer du använder. Kontrollera efter uppdateringen i Marketplace om alla tidigare HACS-projekt har tagits över, granska dashboards och mallar för de nya entitetsnamnen och aktivera vid behov nyss skapade Device Trackers. Vid åtkomst via en Reverse Proxy ska du testa inloggningen. Säkerhetsanvisningarna i GitHub-repositoriet för Home Assistant Core innehåller per den 5 november 2026 ingen post som är kopplad till 2026.11.0; releaseblogginlägget innehåller inga uppgifter om Known Issues.

## Källor

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Releaseblogginlägg med Marketplace, entitetsnamn, nya integrationer och de bakåtinkompatibla ändringarna.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Fullständig changelog, däribland posterna om FRITZ!Box Tools, FRITZ!Box Call Monitor och Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Release-tagg med publicering den 4 november 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Dokumentation för den officiella Midea-integrationen, introducerad i 2026.8, lokal styrning.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Publicerade säkerhetsanvisningar, per den 5 november 2026 utan post om 2026.11.0.

6.  [Midea PortaSplit i Home Assistant: Installation och dashboard](/blog/midea-portasplit-home-assistant): Installation av Midea AC LAN och apexcharts-card via HACS.

7.  [Säkra Midea PortaSplit i Home Assistant](/blog/midea-portasplit-home-assistant-absichern): Supply-chain-risk för Custom Integrations.

8.  [Anslut digitalSTROM till Home Assistant](/blog/digitalstrom-home-assistant): Lokal digitalSTROM-integration och närvaro via FRITZ!Box.
