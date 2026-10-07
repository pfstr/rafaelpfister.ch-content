---
title: "Home Assistant: arkitektur, datamodell och drift"
blatt: "home-assistant"
description: "Home Assistant för plattforms-, nätverks- och IoT-administratörer: händelsedriven Python-kärna, integrationer och register, Home Assistant OS och containrar, protokoll- och radiobryggor, automationskörning, Recorder, API:er, autentisering, observerbarhet, backup och återställning."
fakten:
  - label: Systemroll
    wert: central, händelsedriven plattform för styrning och automatisering av lokala enheter, radionät och externa tjänster
    href: https://developers.home-assistant.io/docs/architecture_index/
  - label: Kärna
    wert: Event Bus, State Machine, Service Registry och Timer utgör körningskärnan
    href: https://developers.home-assistant.io/docs/architecture/core/
  - label: Teknikstack
    wert: Home Assistant Core och dess integrationer är implementerade i Python; asyncio hanterar samtidig I/O-bearbetning
    href: https://github.com/home-assistant/core
  - label: Utökningsmodell
    wert: Integrationer består av domänlogik och plattformar; Config Entries styr deras beständiga livscykel
    href: https://developers.home-assistant.io/docs/architecture_components/
  - label: Objektmodell
    wert: Config Entry → enhet → entitet → tillstånd; register stabiliserar identiteter, namn och tilldelningar
    href: https://developers.home-assistant.io/docs/architecture/devices-and-services/
  - label: Installation som stöds
    wert: Home Assistant OS som hanterad appliance eller Home Assistant Container på en egenhanterad värd
    href: https://www.home-assistant.io/faq/ha-vs-hassio/
  - label: HAOS-stack
    wert: Buildroot, Linux, systemd, Docker, Supervisor, Core och appar; RAUC uppdaterar operativsystemet
    href: https://developers.home-assistant.io/docs/operating-system/
  - label: Gränssnitt
    wert: REST via /api och WebSocket via /api/websocket på samma HTTP-slutpunkt som frontend
    href: https://developers.home-assistant.io/docs/api/rest/
  - label: Standardport
    wert: TCP 8123 för frontend, REST och WebSocket; TLS eller reverse proxy ändrar den externa åtkomstvägen
    href: https://www.home-assistant.io/integrations/http/
  - label: Historik
    wert: Recorder skriver tillstånd och utvalda händelser via SQLAlchemy som standard till SQLite; MariaDB, MySQL och PostgreSQL stöds
    href: https://www.home-assistant.io/integrations/recorder/
  - label: Automationer
    wert: Triggers startar en körning, Conditions avgör och Actions använder samma sekvenssemantik som Scripts
    href: https://www.home-assistant.io/docs/automation/basics/
  - label: Återställning
    wert: krypterade backuper kan återställa konfiguration, Core och appar; nycklar, radiokontroller och externa databaser förblir egna beroenden
    href: https://www.home-assistant.io/common-tasks/general/
werbung:
  - newsletter
ctaThemen:
  - smart-home-iot
translationSourceHash: 204801ccce2af55eaa473c7a7599bd0744a8e9db9b0e1dbdb99efe8949a6fff1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:55:56.806Z
translationReview: automatic
---

# Home Assistant: arkitektur, datamodell och drift

Home Assistant är en central styr- och automationsplattform för enheter, radionät, IP-tjänster och användargränssnitt. Instansen samlar tillstånd via integrationer, normaliserar dem till entiteter, distribuerar ändringar via en Event Bus och utför åtgärder utifrån dem. ”Lokalt” betecknar här en arkitekturpreferens, inte en generell egenskap hos varje integration: en Zigbee-ljuskälla kan vara helt lokalt nåbar, medan en tillverkarintegration hämtar sina tillstånd uteslutande från ett moln-API. Den officiella [arkitekturöversikten](https://developers.home-assistant.io/docs/architecture_index/) skiljer mellan operativsystem, Supervisor och Core; [integrationsarkitekturen](https://developers.home-assistant.io/docs/architecture_components/) beskriver utökningen av Core med Python-komponenter.

För administratörer är Home Assistant därför varken bara en dashboard eller en universell protokollkonverterare. Det är en tillståndsbärande orkestrerare med flera möjliga felpunkter: Python-körmiljö, integrationer, register, databas, autentisering, lokala nätverk, radiokontroller, broker, tillverkarens molntjänster och i förekommande fall Supervisor-appar. Ett grönt gränssnitt bevisar bara att frontendvägen fungerar. Det bevisar inte att händelser kommer i tid, att enheter är nåbara, att automationer körs deterministiskt eller att en backup inklusive externa beroenden går att återställa.

Förklaringen följer en enhetshändelse via integration, Event Bus och State Machine till automation och åtgärd. Därefter sätts persistens, tillägg, säkerhet, övervakning och återställning i sitt sammanhang.

## Arkitekturprincip: central händelse- och tillståndsnod

Home Assistant Core är händelsedriven. Fyra dokumenterade byggstenar utgör kärnan ([Core architecture](https://developers.home-assistant.io/docs/architecture/core/)):

1. **Event Bus** distribuerar händelser till registrerade lyssnare.
2. **State Machine** håller det senast kända tillståndet för varje laddad entitet och publicerar `state_changed`.
3. **Service Registry** hanterar anropbara åtgärder och behandlar tjänsteanrop.
4. **Timer** skapar tidshändelser för tidsberoende bearbetning.

Integrationer översätter enhets- eller tjänstetillstånd till denna modell. En integration kan polla, ta emot pushhändelser, använda lokala bibliotek eller anropa ett fjärr-API. Home Assistant enhetliggör det resulterande tillståndet, inte transporten. Det är den viktigaste driftsgränsen: två entiteter med samma domäntyp, exempelvis `light`, kan ha helt olika vägar för latens, autentisering och återställning.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-home-assistant.svg?v=20260813" title="Interaktive Infografik: Home Assistant von Geräten und Protokollbrücken über Integrationen, Registries, Event Bus, State Machine, Automationen und Recorder bis zu APIs, Supervisor, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-home-assistant.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## Körningslager och installationsmodeller

Home Assistant erbjuder två installationsmodeller som stöds. **Home Assistant OS** är en hanterad appliance. **Home Assistant Container** kör Home Assistant Core som en container på en värd som driftansvarig ansvarar för. Den officiella jämförelsen anger HAOS som rekommendation för nästan alla installationer och beskriver Container som en fristående Core-installation utan Supervisor-appar ([HAOS eller Container](https://www.home-assistant.io/faq/ha-vs-hassio/)).

### Home Assistant OS

HAOS byggs med Buildroot och består av Linux, GNU C Library, systemd och Docker. SquashFS bär de skrivskyddade systemdelarna, ZRAM temporära filsystem och swap, AppArmor begränsar processer och RAUC uppdaterar operativsystemet ([Home Assistant Operating System](https://developers.home-assistant.io/docs/operating-system/)). Ovanpå detta hanterar **Supervisor** Core, appar, DNS, ljud, mDNS, backuper och uppdateringar ([Supervisor](https://developers.home-assistant.io/docs/supervisor/)).

Appliance-modellen minskar variationerna men ger Supervisor ett omfattande ansvar. Ett fel kan finnas på minst fem nivåer: start-/OS-slot, Docker Engine, Supervisor, Core-container eller en enskild app. Supervisor kan återställa en misslyckad uppdateringsväg för Core; den mekanismen upptäcker inte automatiskt felaktigt enhets- eller databasbeteende.

### Home Assistant Container

Container tillhandahåller endast Core. Värdoperativsystem, container engine, nätverk, volymer, databas, broker, radioserver, reverse proxy, backup och uppdateringar tillhör driftansvarig. Supervisor-appar är separat paketerade tjänster. I en containerdesign körs Mosquitto, Matter Server, Zigbee2MQTT, Z-Wave JS UI, PostgreSQL eller en reverse proxy som egna arbetslaster med egna volymer, versioner och hälsokontroller.

Fördelen är en uttrycklig plattformsarkitektur; priset är en större driftsyta. En backup av Core-volymen innehåller exempelvis varken den externa Recorder-databasen eller brokertillstånd, radio-NVM eller reverse-proxy-nycklar om dessa ligger utanför.

### Historiska installationsformer

Den tidigare **Core**-installationen i en Python-miljö och den **Supervised**-installationen på ett självhanterat Linux avvecklades 2025. Sedan release 2025.12 stöds de inte; 32-bitarsarkitekturerna `i386`, `armhf` och `armv7` förlorade samtidigt releasevägen. Projektmeddelandet anger HAOS och Container som återstående modeller ([Avveckling av Core och Supervised](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)). Ett historiskt installationsnamn får därför inte förväxlas med programvarukomponenten **Home Assistant Core**, som fortsätter att köras även i HAOS och Container.

## Teknikstack

Home Assistant Core är en Python-applikation under Apache-2.0-licens. Det officiella [Core-repositoriet](https://github.com/home-assistant/core) visar Python, asyncio och den modulära integrationsstrukturen. I/O-intensiva integrationer ska inte blockera: kvalitetsreglerna föredrar asynkrona beroenden så att nätverks- och enhetsanrop inte stoppar den gemensamma eventloopen ([async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)). Blockerande bibliotekskod flyttas till executor-trådar; CPU-intensivt eller dåligt begränsat arbete förblir dock en kapacitets- och latensrisk.

Den synliga stacken omfattar mer än Python:

| Lager | Typisk teknik | Driftsrelevans |
|---|---|---|
| Frontend | Webbläsarapplikation, HTTP och WebSocket | Användar- och realtidsväg |
| Core | Python, asyncio, integrationer | Tillstånd, händelser, åtgärder, autentisering |
| Persistens | JSON-baserade konfigurationslager, YAML, SQLAlchemy/SQL | Konfiguration, register, historik |
| HAOS | Buildroot, Linux, systemd, Docker, AppArmor, RAUC | Appliance-livscykel och isolering |
| Tjänster | Supervisor-appar eller externa containrar/värdar | MQTT, Matter, databas, proxy, fildelningar |
| Edge | Radiokontroller, protokollbryggor, enhets- och moln-API:er | Fysisk nåbarhet och dataursprung |

[Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/) bedömer integrationer efter konfigurationsflöde, tester, typning, diagnostik, effektiv dataanvändning och asynkront beteende. En hög nivå förbättrar den förväntade underhållbarheten, men är inget tillgänglighets-SLA för den bakomliggande enheten eller molnleverantören.

Efter installationsmodell och körmiljö följer datamodellen. Först när Config Entry, enhet, entitet och tillstånd hålls isär går det att förklara dubbla entiteter, saknade enheter och felaktiga automationer på ett korrekt sätt.

## Objektmodell: Config Entry, enhet, entitet och tillstånd

Den operativa inventarienyckeln är inte den synliga rutan utan kedjan av konfiguration, enhetsidentitet och entitet.

### Config Entries

En **Config Entry** sparar den beständiga konfigurationen för en integrationsinstans. Ett UI-konfigurationsflöde skapar den; alternativ, omkonfigurering, omladdning, avlastning, borttagning och migrering är definierade livscykeloperationer. Integrationer får inte mutera Entry-data direkt utan måste använda Config-Entry-hanteraren ([Config entries](https://developers.home-assistant.io/docs/config_entries_index/)). Ett autentiseringsfel, en oladdad Entry och en onåbar motpart är därför olika tillstånd.

### Enheter och register

**Device Registry** grupperar tekniska ändpunkter till enheter. Identifiers eller Connections, exempelvis serienummer och MAC-adress, används för matchning; `via_device` kan avbilda en brygg- eller föräldrarelation ([Device registry](https://developers.home-assistant.io/docs/device_registry_index/)). En Zigbee-sensor kan därmed visas som en ansluten enhet via en Coordinator utan att Coordinators applikationstillstånd är dess eget.

**Entity Registry** ger entiteter med `unique_id` en beständig identitet och förhindrar kolliderande Entity IDs. IP-adress, värdnamn, URL, användarnamn eller e-postadress räknas uttryckligen inte som stabila Unique IDs ([Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)). Det förklarar varför manuell namnändring av en värd inte får ersätta enhetsidentiteten och varför integrationsmigreringar behöver stabila tillverkaridentifierare.

### Entitet och tillstånd

En **entitet** representerar en funktion eller mätstorhet: `sensor`, `switch`, `light`, `climate`, `binary_sensor` eller en annan domän. Dess tillstånd består av ett primärt State, attribut, ändringstider och Context. State Machine håller endast det senast kända tillståndet. `unavailable` betyder att entiteten för närvarande inte försörjs av ett aktivt Entity-objekt; `unknown` betyder att inget användbart värde finns. ”Senaste värde” betyder därmed inte automatiskt ”färskt mätvärde”.

Den dokumenterade [interaktionen mellan enheter och tjänster](https://developers.home-assistant.io/docs/architecture/devices-and-services/) skiljer mellan Entity Integration, Entity Component, Entity Platform och tillverkarspecifik integration. För diagnostik frågar man därför alltid:

- Vilken Config Entry äger entiteten?
- Via vilken integration och plattform skapas den?
- Vilket stabilt enhets- och Entity ID kopplar samman historik och konfiguration?
- Pollas eller pushas den?
- Vilken tids- och tillgänglighetsmodell har källvärdet?
- Vilken brygga, vilket bibliotek, moln-API eller radiolänk ligger framför den?

## Integrationer och felisolering

En integration definierar en domän och kan tillhandahålla plattformar som `sensor`, `light` eller `switch`. Plattformen abstraherar entitetstypen; enhetsintegrationen kommunicerar med det konkreta protokollet. Inbyggda integrationer levereras med Core och testas genom dess releaseprocess. **Custom Integrations** körs dock i samma Python-process och kan påverka importer, eventloop, starttid eller minnesförbrukning. Katalogen `/config/custom_components` är därför en del av inventarie, change management och återställning.

Den officiella [integrationsöversikten](https://www.home-assistant.io/integrations/) skiljer bland annat mellan IoT-klasser som Local Push, Local Polling, Cloud Push och Cloud Polling. Denna klassificering är mer användbar för driftsmodeller än en lång tillverkarlista:

| Klass | Dataväg | Typiskt felområde |
|---|---|---|
| Local Push | Enhet eller brygga skickar i LAN | Multicast, brandvägg, brygga, subnät |
| Local Polling | Core frågar lokal enhet | Latens, timeout, frågeintervall, enhetskapacitet |
| Cloud Push | Molnet skickar eller strömmar händelser | Internet, konto, token, leverantörsström |
| Cloud Polling | Core frågar leverantörens API | Rate Limit, token, internet, API-ändring |
| Calculated/Internal | Core beräknar tillstånd | Indata, mallar, tid, omstartstillstånd |

Integrationen är ingen processisolator. En ren felavgränsning inaktiverar eller laddar om den berörda Config Entry riktat innan hela Core startas om. En omstart förstör flyktig evidens och kan återställa tidsberoende automationstimers.

## Protokoll- och nätverksmodell

För Home Assistant är en **beroendegraf** mer användbar än en allmän OSI-tabell. Plattformen ligger på applikationsnivån, men dess datavägar förgrenar sig:

- Frontend, REST och WebSocket körs över HTTP på TCP, som standard port 8123.
- DNS löser upp värd- och molntjänster; mDNS och SSDP upptäcker enheter i det lokala nätet.
- MQTT använder en separat broker och en publish/subscribe-modell över TCP eller WebSocket.
- Zigbee, Z-Wave, Thread och Bluetooth kräver radiokontroller eller nätverksproxyer.
- Matter använder IP-kommunikation men kräver för provisionering och Fabric-drift en Matter Server och eventuellt en Thread Border Router.
- Tillverkarintegrationer kan använda HTTPS, proprietära lokala protokoll eller molnströmmar.

De inbyggda discovery-integrationerna dokumenterar [mDNS/Zeroconf](https://www.home-assistant.io/integrations/zeroconf/) och [SSDP](https://www.home-assistant.io/integrations/ssdp/). Båda är beroende av segment och multicast. En reverse proxy för frontend reparerar inte discovery över VLAN-gränser. Multicast-relä, IGMP-snooping, WLAN-klientisolering, IPv6-RA, DNS-suffix och brandväggsregler kontrolleras för varje faktisk enhetsväg.

### MQTT som eget tillståndsrum

MQTT är inte den interna Event Bus. Det är en extern brokertjänst som en integration kommunicerar med. Den officiella [MQTT-integrationen](https://www.home-assistant.io/integrations/mqtt/) beskriver Discovery Topics, retained messages, Birth/Last Will, Availability, TLS och MQTT 5. Retained Discovery kan återskapa enheter efter en omstart, men kan också bevara föråldrade Ghost Entities. Tillgänglighet kräver egen semantik; ett befintligt retained State bevisar inte att publiceraren fortfarande lever.

Robust MQTT-drift inventerar broker, Client IDs, autentisering, CA, Topics, QoS, Retain, Expiry, Birth/Will och Discovery-Origin. Brokerbackup och Corebackup är separata skyddsobjekt.

### Radio- och bryggvägar

[ZHA](https://www.home-assistant.io/integrations/zha/) integrerar en Zigbee-Coordinator, [Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/) använder en separat Z-Wave-JS-server och [Matter](https://www.home-assistant.io/integrations/matter/) ansluter en Matter Server. [Thread](https://www.home-assistant.io/integrations/thread/) hanterar relationer till Border Routers och nätverk, men är inte identiskt med Matter. Radioenheter, kontrollerns firmware, nätverksdata, nyckelmaterial och enhetskonfiguration utgör var och en en återställningshelhet. Att flytta ett USB-minne eller ersätta en Coordinator är ingen vanlig IP-adressändring.

Integrationer levererar tillstånd och händelser; automationer reagerar på dem. Deras flöde från Trigger, Conditions och Actions måste därför diagnostiseras separat från enhetskonfigurationen.

## Automationskörning: Trigger, Condition, Action

En automation är en reaktiv körningsdefinition. [Automationsgrunderna](https://www.home-assistant.io/docs/automation/basics/) skiljer mellan Trigger, valfria Conditions och Actions. Trigger skapar en körning, Conditions kontrollerar tillståndet vid inträdet och Actions använder Scriptens sekvenssemantik ([Actions](https://www.home-assistant.io/docs/automation/action/)).

Tidpunkten är viktig: State, attribut och mallvärden kan ändras mellan Trigger och en senare Action. En Delay håller ingen transaktion öppen. Flera körningar av samma automation behöver därför ett läge som Single, Restart, Queued eller Parallel samt en medveten konfliktmodell. Fysiska aktorer är sällan transaktionella; en delvis utförd körning kan behöva kompensationsåtgärder.

[Trigger-dokumentationen](https://www.home-assistant.io/docs/automation/trigger/) påpekar att `for`-väntetider inte överlever en omstart eller omladdning av automationen. Den som måste bevara en tidsfrist över omstarter lagrar en tidpunkt, exempelvis i `input_datetime`, och utlöser mot den. Conditions är bara kontroller i den aktuella körningen; [Condition-semantiken](https://www.home-assistant.io/docs/scripts/conditions/) gör dem inte till ett lås mot parallella ändringar.

Mallar utvärderas i Home Assistant med Jinja-uttryck. Indata- och typfel, `unknown`, `unavailable`, tidszoner och implicit strängkonvertering ska ingå i tester. [Templating-dokumentationen](https://www.home-assistant.io/docs/automation/templating/) beskriver triggerberoende variabler. En administratör testar inte bara happy path utan även omstart, saknad entitet, försenad händelse, dubbel Trigger och aktorfel.

## Konfiguration, register och Source of Truth

Home Assistant kombinerar UI-styrda Config Entries, registerdata och YAML. `configuration.yaml` är roten för manuell konfiguration, men inte den fullständiga Source of Truth. Den officiella [konfigurationsöversikten](https://www.home-assistant.io/docs/configuration/) skiljer mellan UI och YAML; Packages kan strukturera sammanhängande YAML-block ([Packages](https://www.home-assistant.io/docs/configuration/packages/)).

För Git och review lämpar sig endast den textuella delen utan hemligheter. `secrets.yaml` separerar värden från YAML men krypterar dem inte; [härdningsguiden](https://www.home-assistant.io/docs/configuration/securing/) påpekar uttryckligen detta. UI-tillstånd, register, tokens och Config Entries ligger i konfigurationslagret och ändras via stödda UI-/API-vägar. Direkt redigering av interna storage-filer medan Core körs kringgår schema-, livscykel- och konsistenslogik.

En konfigurationsinventering omfattar:

- YAML, Packages, Blueprints och Custom Components,
- Config Entries med ursprung, ägare och autentisering,
- Device-, Entity- och Area-tilldelningar,
- automationer, Scripts, scener och dashboards,
- användare, tokens, MFA och externa Identity Providers,
- Supervisor-appar eller externa tjänster,
- radiokontroller, broker, databas och proxy,
- Secrets, certifikat och återställningsnycklar.

## Recorder, historik och långtidsstatistik

State Machine håller det aktuella tillståndet i minnet. Historik uppstår först genom **Recorder**. Den skriver tillståndsändringar och utvalda händelser via SQLAlchemy till en databas; History, Activity, diagram och långtidsstatistik läser därifrån. Den officiella [Recorder-dokumentationen](https://www.home-assistant.io/integrations/recorder/) anger SQLite som standard och rekommendation samt MariaDB, MySQL och PostgreSQL som alternativ som stöds.

Recorderdata är ingen händelsekälla för realtidsstyrning. En databas som har fallerat kan påverka historik och statistik medan aktuella tillstånd och automationer delvis fortsätter att fungera. Omvänt bevisar en komplett historik inte att en Action lyckades på den fysiska enheten.

De viktigaste driftparametrarna är:

- `purge_keep_days` för råhistorik,
- Include-/Exclude-filter för entiteter och händelser,
- `commit_interval` som förhållande mellan I/O och förlustfönster,
- databasstorlek, ledigt utrymme och skrivlatens,
- Purge och Repack,
- startordning och nåbarhet för externa databaser,
- långtidsstatistik och metadatakonsistens.

Ett byte av Recorder-databas migrerar inte den befintliga historiken på ett sätt som stöds. Externa databaser behöver egna konsistenta backuper och återställningstester. För SQLite anger dokumentationen ledigt utrymme på minst 2,5 gånger databasens storlek för korruptionshantering. Storage och Recorder är därmed en egen kapacitets- och återställningsväg, inte bara en valfri cache.

## API, WebSocket och autentisering

Frontend och API:er delar som standard samma HTTP-listener. [REST API](https://developers.home-assistant.io/docs/api/rest/) använder JSON och Bearer Tokens; bassökvägen är `/api/`. [WebSocket API](https://developers.home-assistant.io/docs/api/websocket/) ligger under `/api/websocket`, går igenom `auth_required`, `auth` och `auth_ok` och korrelerar kommandon via numeriska ID:n. WebSocket levererar händelseströmmar och register effektivare än upprepad REST-pollning.

Långlivade tokens är användaruppgifter. [Authentication API](https://developers.home-assistant.io/docs/auth_api/) beskriver OAuth/IndieAuth, Refresh Tokens, Long-Lived Access Tokens och kortlivade Signed Paths. En token ärver sin användares kontext; en Long-Lived Token som gäller i tio år hör hemma i en Secrets-hanterare, inte i YAML, shellhistorik, URL eller dashboard-JavaScript.

En API-monitor kontrollerar minst autentisering, `/api/config`, förväntade entiteter, `last_updated`, WebSocket-prenumeration och en ofarlig läs-/åtgärdsväg. HTTP 200 på `/` kontrollerar endast frontendens nåbarhet.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-API-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:HA_TOKEN" }
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/config"
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/states/sensor.uptime" |
  ConvertTo-Json -Depth 8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/config | jq .
curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/states/sensor.uptime | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) och [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) bearbetar Windows-frågan; [`curl`](https://curl.se/docs/manpage.html) och [`jq`](https://jqlang.org/manual/) gör samma sak i Unix. Token visas endast som processmiljövariabel; i produktion kommer den från en kontrollerad Secrets-hanterare.

## HTTP, TLS och reverse proxy

HTTP-slutpunkten lyssnar som standard på TCP 8123. Direkt TLS, reverse proxy och Home Assistant Cloud är olika åtkomstmodeller. Vid en traditionell reverse proxy måste `use_x_forwarded_for` och `trusted_proxies` ställas in korrekt; annars blir klient-IP fel eller begäran avvisas ([HTTP integration](https://www.home-assistant.io/integrations/http/)). En bred lista över betrodda proxyer möjliggör förfalskning av Forwarded-For-information.

[Säkerhetsguiden](https://www.home-assistant.io/docs/configuration/securing/) rekommenderar unika lösenord, MFA, minsta möjliga administratörsbehörigheter och säker fjärråtkomst i stället för direkt internetexponering. TLS skyddar bara transporten. Tokenbehörigheter, proxyheaders, WebSocket-uppgraderingar, Rate Limits, DNS, certifikatförnyelse och säkerheten hos den framförliggande IdP:n förblir egna kontroller.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName ha.example.net
Test-NetConnection ha.example.net -Port 443
curl.exe -sS -D - -o NUL https://ha.example.net/api/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short ha.example.net A ha.example.net AAAA
curl -sS -D - -o /dev/null https://ha.example.net/api/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect ha.example.net:443 -servername ha.example.net -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) och [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) testar Windows; [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) testar Unix. Det oautentiserade anropet `/api/` får returnera 401; avgörande är namnupplösning, TLS-identitet, proxyväg och förväntad autentiseringsgräns.

## MQTT-diagnostik

Brokertillståndet kontrolleras utanför Home Assistant. En Subscriber observerar Discovery, Availability och State utan att ändra Topics. Ett Publish-test använder en särskilt reserverad testsökväg; produktiva Command Topics beskrivs inte i förbifarten.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MQTT-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">mosquitto_sub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-observer -P $env:MQTT_PASSWORD -v -t "homeassistant/#"
mosquitto_pub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-probe -P $env:MQTT_PASSWORD -t "ops/probe" -m "online" -q 1</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mosquitto_sub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-observer -P "$MQTT_PASSWORD" -v -t 'homeassistant/#'
mosquitto_pub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-probe -P "$MQTT_PASSWORD" -t 'ops/probe' -m 'online' -q 1</code></pre>
  </div>
</div>

[`mosquitto_sub`](https://mosquitto.org/man/mosquitto_sub-1.html) och [`mosquitto_pub`](https://mosquitto.org/man/mosquitto_pub-1.html) är de officiella brokerklienterna. Lösenord på kommandoraden kan synas i processlistor eller historik; exemplen illustrerar vägen, medan produktionsanropet använder lösenordsfil, operativsystemets Secrets-hanterare eller kortlivade inloggningsuppgifter.

## Drift av HAOS och Container

HAOS tillhandahåller kommandot `ha` via terminal-/SSH-åtkomst. Containerinstallationer drivs med verktygen för den valda runtime-miljön. Ett diagnospaket samlar systeminformation, Core-logg, integrationsdiagnostik, containerstatus, ledigt utrymme och tidpunkt.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-Laufzeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker inspect homeassistant | ConvertFrom-Json
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 |
  Select-String -Pattern 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker inspect homeassistant | jq '.[0].State, .[0].Mounts, .[0].NetworkSettings.Networks'
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 | grep -E 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant
df -h /path/to/config &amp;&amp; du -sh /path/to/config</code></pre>
  </div>
</div>

[`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/), [`docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/) och [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) visar containertillstånd. [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) och [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) bearbetar Windows-utdata; [`grep`](https://www.gnu.org/software/grep/manual/grep.html), [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) och [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html) kompletterar Unix. En körande container är bara den första kontrollen; därefter följer integrations-, register-, händelse- och enhetsväg.

För felsökning läses signalvägen baklänges: åtgärd, Automation Trace, tillståndsändring, integration, nätverksprotokoll och fysisk enhet.

## Observerbarhet och systematisk diagnostik

**System Health** samlar installationstyp, arkitektur, Python-, Core- och frontendinformation och erbjuder diagnostikfunktioner via Inställningar > System > Reparationer ([System Health](https://www.home-assistant.io/integrations/system_health/)). [Logger-integrationen](https://www.home-assistant.io/integrations/logger/) styr globala och komponentspecifika loggnivåer. Debuggloggning begränsas tidsmässigt och till berörda namnrymder; radio- eller händelsestormar kan annars dominera minne och I/O.

En robust diagnostikkedja är:

1. **Symtom och önskat tillstånd:** Vilken entitet, Action, automation eller yta är berörd?
2. **Tid och omfattning:** Sedan när, för vilka enheter, användare, nätverk och integrationsinstanser?
3. **Objektidentitet:** Säkerställ Config Entry, Device ID, Entity ID, Unique ID och bryggrelation.
4. **Körmiljö:** Kontrollera Core, eventloop, minne, CPU, filsystem och databas.
5. **Integration:** Kontrollera Entry-tillstånd, autentisering, Coordinator-/pollningsstatus och diagnostiknedladdning.
6. **Transport:** Kontrollera discovery, DNS, TCP, TLS, broker, radiokontroller eller tillverkar-API.
7. **Automation:** Kontrollera Trace, triggerdata, Conditions, Run Mode och åtgärdsresultat.
8. **Persistens:** Bedöm Recorder-fördröjning och historik separat från live-tillståndet.
9. **Kontrollerat test:** Använd en skrivskyddad eller ofarlig testentitet.
10. **Återställning:** Reload före restart, restart före restore; säkra evidens först.

En entitet `unavailable` kan bero på en avlastad Config Entry, saknad brygga, radioförlust eller timeout i källan. En synlig gammal siffra är farligare eftersom den verkar trovärdig. Övervakning behöver därför färskhetsgränser, inte bara värdegränser.

## Uppdateringar, releaser och Custom Integrations

Home Assistant publicerar frekventa Core-releaser och dokumenterar bakåtinkompatibla ändringar. En statisk referensartikel låser medvetet inte fast någon aktuell versionsnivå. Vid underhållstidpunkten kontrollerar utrullningen i stället release notes, integrationsändringar och målberoenden.

En kontrollerad uppdateringsväg omfattar:

1. Bekräfta backup och oberoende nedladdning respektive extern lagringsplats.
2. Kontrollera ledigt utrymme, databastillstånd och System Health.
3. Bedöm release notes samt berörda integrationer och Custom Components.
4. Inventera radio-, broker-, databas- och proxyberoenden.
5. Uppdatera Core respektive HAOS och appar i definierad ordning.
6. Kontrollera startlogg, reparationer och registermigreringar.
7. Testa kritiska sensor-, aktor-, automations-, API- och fjärråtkomstvägar.
8. Fastställ felgränsen och utlös först därefter rollback eller restore.

HAOS använder RAUC med två operativsystemslots; `ha os info` och `rauc status` visar slottillståndet ([HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)). Den mekanismen skyddar OS-uppdateringsvägen, men inte automatiskt Core-konfiguration, Recorderdata eller radionätverkstillstånd.

När körmiljön och datavägen är kända kan backupomfattningen fastställas. Konfiguration, register, Secrets, databas och add-on-tillstånd måste tillsammans passa den valda installationsmodellen.

## Backup och återställning

Home Assistant kan skriva automatiska och manuella, krypterade backuper till lokala eller externa mål. Den officiella [guiden för backup och restore](https://www.home-assistant.io/common-tasks/general/) beskriver backupplatser, Emergency Kit, nedladdning, restore vid onboarding och migrering till annan maskinvara. Sedan 2026 har backupernas kryptomodell moderniserats; [meddelandet om backupkryptering](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/) dokumenterar formatändringen och kompatibilitetsgränserna.

En backup är bara komplett i förhållande till installationsmodellen:

| Objekt | HAOS-backup | Ansvar för Container/externt |
|---|---|---|
| Core-konfiguration och register | kan inkluderas | säkra Config-volymen |
| Supervisor-appar | appdata kan inkluderas | separata containrar och volymer |
| Recorder SQLite | i Config-området | konsistent DB-backup vid extern DB |
| MQTT Broker | endast med lämpligt appval | brokerkonfiguration och persistens separat |
| Zigbee/Z-Wave/Matter | integrationsdata delvis | kontrollera controller-/serverbackup och nycklar separat |
| TLS/proxy/DNS | endast om de ligger inom valda data | extern infrastruktur separat |
| Backupnyckel | inte tillräckligt i den krypterade backupen själv | förvara Emergency Kit separat |

Ett återställningstest slutar inte vid inloggning. Acceptanskriterier är: Config Entries laddade, register konsistenta, användaråtkomst möjlig, databas utan fel, broker och bryggor anslutna, radioenheter styrbara, kritiska automationer testade och fjärråtkomst med korrekt certifikat tillgänglig. Batterienheter kan först sova efter en migrering; ett saknat omedelbart värde ska inte förhastat tolkas som dataförlust.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Backupinventar und Prüfsummen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\ha-backups -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\ha-backups-manifest.csv -NoTypeInformation
Get-Content .\ha-backups-manifest.csv -First 5</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./ha-backups -type f -print0 | sort -z | xargs -0 sha256sum &gt; ha-backups-manifest.sha256
head -n 5 ha-backups-manifest.sha256
tar -tf ./ha-backups/example-backup.tar | head</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv) och [`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content) skapar respektive läser Windows-manifestet. [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html), [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) och [`tar`](https://www.gnu.org/software/tar/manual/html_node/index.html) hanterar Unix. En kontrollsumma bevisar att arkivet är oförändrat; endast återställningstestet bevisar dekrypterbarhet och funktionell återställning.

## RPO, RTO och hög tillgänglighet

Home Assistant är i normal drift en tillståndsbärande enskild instans. Två aktiva Core-instanser mot samma enheter, register eller broker-Commands ger ingen automatiskt samordnad hög tillgänglighet. Dubbla automationer kan slå om aktorer flera gånger; radiokontroller och lokala enheter tillåter ofta bara ett aktivt ägarskap.

En realistisk resiliensmodell kombinerar:

- tillförlitlig enskild nod eller VM med övervakade resurser,
- UPS och lämplig lagring i stället för känsliga flashmedier,
- separata, automatiska och krypterade backuper,
- dokumenterad reservmaskinvara eller VM-målplattform,
- exporterbara radiokontrollertillstånd och nycklar,
- reproducerbara externa tjänster,
- kontrollerad restore med entydigt övertagande av enheter och nätverk.

**RPO** beror på den senast säkrade konfigurationen, registret, app- och externa tjänstekopian. Recorderhistorik kan ha ett annat RPO än automationskonfigurationen. **RTO** omfattar inte bara Core-start utan även DNS, proxy, databas, broker, radiokontroller, enhetsåteranslutning, sovande sensorer och acceptanstester.

## Säkerhet och förtroendegränser

Home Assistant kan styra dörrar, värme, larmsystem och energiflöden. Påverkansområdet är därmed fysiskt. Säkerhetsdesign skiljer mellan:

- användare och administratörer,
- sessioner för webbläsare, Companion App och API,
- Long-Lived Tokens och webhooks,
- Core och Custom Integrations,
- Supervisor-appar eller externa containrar,
- IoT-, management- och användarsegment,
- lokala enheter och tillverkarmoln,
- radionät och deras nycklar,
- backupmål och Emergency Kit.

MFA skyddar interaktiva konton, men inte en stulen Long-Lived Token. Nätverkssegmentering begränsar lateral förflyttning, men får inte okontrollerat blockera nödvändig discovery och returkanaler. Custom Integrations har processnärhet till Core och behandlas som koddistributioner. Secrets förekommer varken i Git, diagnosfiler eller supportinlägg. Allmänna kontroller finns under [härdning](/kb/haertung), grundläggande transportinformation under [TLS](/kb/tls) och API-avtalsmodeller under [API:er](/kb/apis).

## Teknisk historia

Home Assistant började 2013 som ett Python-projekt av Paulus Schoutsen. Återblicken över tioårsjubileet beskriver utvecklingen från en liten lokal automationsapplikation till ett stort open-source-projekt ([10 år med Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)). Python-Core och integrationsmodellen förblev den funktionella kärnan, medan frontend, mobila klienter, Supervisor, HAOS, enhetshårdvara och molnalternativ utvecklades runt dem.

Med Hass.io, senare Home Assistant respektive Home Assistant OS och Supervisor, uppstod en appliance-stack med operativsystem, containerhantering, Core och tilläggstjänster. Uppdelningen har språkligt förtydligats flera gånger: ”Add-ons” kallas i dag **Apps**, medan ”integrationer” fortfarande är Python-utökningar av Core. Dessa termer beskriver olika körnings- och säkerhetsgränser.

2024 övergick Home Assistant till den ideella Open Home Foundation; Nabu Casa förblev kommersiell partner. Projektmeddelandet om [Open Home-ekosystemet](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/) beskriver ägarskap och styrning. 2025 minskade projektet de installationsvarianter som stöds till HAOS och Container. Den historiska trenden går därmed inte mot ett distribuerat kluster utan mot en stabilare central Core med tydligt stödda körmiljöpaket och fristående protokollservrar.

## Checklista för administratörer

En grön dashboard räcker inte som driftsbevis. Checklistan sammanför installation, enhetsvägar, automationer, datahållning och återställning till en verifierbar helhetsbild.

- **Installation:** Dokumentera HAOS eller Container, arkitektur, värd, lagring, nätverk och ägarskap.
- **Stack:** Håll isär Core, Supervisor, appar/externa containrar, databas, broker, proxy och radioserver.
- **Inventering:** Registrera Config Entry, Device ID, Entity ID, Unique ID, Area och `via_device`.
- **Dataursprung:** Markera Local/Cloud samt Push/Polling för varje kritisk integration.
- **Tillstånd:** Skilj mellan `unknown`, `unavailable`, föråldrat värde och bekräftad enhetsframgång.
- **Automation:** Testa Trigger, Context, Condition, Run Mode, omstartsbeteeende och kompensation.
- **API:er:** Kontrollera användarkontext, tokenlagring, WebSocket, reverse proxy och TLS.
- **Recorder:** Övervaka databas, filter, commitintervall, purge, I/O, tillväxt och backup.
- **IoT-nät:** Kontrollera uttryckligen mDNS, SSDP, MQTT, VLAN, IPv6 och radio-/bryggvägar.
- **Uppdateringar:** Sammanför release notes, Custom Integrations, backup, utrullning och godkännande.
- **Återställning:** Testa Core, externa tjänster, radiotillstånd, nycklar och Emergency Kit tillsammans.
- **Bevis:** Verifiera inte bara UI och container utan minst en sensor-, aktor-, automations- och API-väg från början till slut.

## Källor

- [Home Assistant Developer Docs – arkitekturöversikt](https://developers.home-assistant.io/docs/architecture_index/)
- [Home Assistant Developer Docs – integrationsarkitektur](https://developers.home-assistant.io/docs/architecture_components/)
- [Home Assistant Developer Docs – Core architecture](https://developers.home-assistant.io/docs/architecture/core/)
- [Home Assistant – HAOS eller Container](https://www.home-assistant.io/faq/ha-vs-hassio/)
- [Home Assistant Developer Docs – Operating System](https://developers.home-assistant.io/docs/operating-system/)
- [Home Assistant Developer Docs – Supervisor](https://developers.home-assistant.io/docs/supervisor/)
- [Home Assistant – avveckling av Core, Supervised och 32-bit](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
- [GitHub – Home Assistant Core](https://github.com/home-assistant/core)
- [Home Assistant Developer Docs – Async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)
- [Home Assistant Developer Docs – Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/)
- [Home Assistant Developer Docs – Config entries](https://developers.home-assistant.io/docs/config_entries_index/)
- [Home Assistant Developer Docs – Device registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Home Assistant Developer Docs – Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)
- [Home Assistant Developer Docs – enheter och tjänster](https://developers.home-assistant.io/docs/architecture/devices-and-services/)
- [Home Assistant – integrationer](https://www.home-assistant.io/integrations/)
- [Home Assistant – Zeroconf](https://www.home-assistant.io/integrations/zeroconf/)
- [Home Assistant – SSDP](https://www.home-assistant.io/integrations/ssdp/)
- [Home Assistant – MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Home Assistant – ZHA](https://www.home-assistant.io/integrations/zha/)
- [Home Assistant – Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/)
- [Home Assistant – Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant – Thread](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant – automationsgrunder](https://www.home-assistant.io/docs/automation/basics/)
- [Home Assistant – Automation actions](https://www.home-assistant.io/docs/automation/action/)
- [Home Assistant – Automation triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Home Assistant – Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Home Assistant – Automation templating](https://www.home-assistant.io/docs/automation/templating/)
- [Home Assistant – konfiguration](https://www.home-assistant.io/docs/configuration/)
- [Home Assistant – Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [Home Assistant – säkra Home Assistant](https://www.home-assistant.io/docs/configuration/securing/)
- [Home Assistant – Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant Developer Docs – REST API](https://developers.home-assistant.io/docs/api/rest/)
- [Home Assistant Developer Docs – WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)
- [Home Assistant Developer Docs – Authentication API](https://developers.home-assistant.io/docs/auth_api/)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – handbok](https://curl.se/docs/manpage.html)
- [jq – handbok](https://jqlang.org/manual/)
- [Home Assistant – HTTP integration](https://www.home-assistant.io/integrations/http/)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Eclipse Mosquitto – mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)
- [Eclipse Mosquitto – mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Docker – logs](https://docs.docker.com/reference/cli/docker/container/logs/)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [Microsoft – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [GNU Grep – handbok](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [GNU Coreutils – du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Home Assistant – System Health](https://www.home-assistant.io/integrations/system_health/)
- [Home Assistant – Logger](https://www.home-assistant.io/integrations/logger/)
- [Home Assistant Developer Docs – HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)
- [Home Assistant – backup och restore](https://www.home-assistant.io/common-tasks/general/)
- [Home Assistant – moderniserad backupkryptering](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [GNU Tar – handbok](https://www.gnu.org/software/tar/manual/html_node/index.html)
- [Home Assistant – 10 år med Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)
- [Home Assistant – Open Home Foundation och styrning](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/)
