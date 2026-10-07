---
title: "Home Assistant: arkitektur, datamodell og drift"
blatt: "home-assistant"
description: "Home Assistant for plattform-, nettverks- og IoT-administratorer: hendelsesdrevet Python-kjerne, integrasjoner og registre, Home Assistant OS og containere, protokoll- og radiobroer, automatiseringskjøretid, Recorder, API-er, autentisering, observability, sikkerhetskopiering og gjenoppretting."
fakten:
  - label: Systemrolle
    wert: sentral, hendelsesdrevet styrings- og automatiseringsplattform for lokale enheter, radionettverk og eksterne tjenester
    href: https://developers.home-assistant.io/docs/architecture_index/
  - label: Kjerne
    wert: Event Bus, State Machine, Service Registry og Timer utgjør kjøretidskjernen
    href: https://developers.home-assistant.io/docs/architecture/core/
  - label: Teknologistack
    wert: Home Assistant Core og integrasjonene er implementert i Python; asyncio håndterer samtidig I/O-behandling
    href: https://github.com/home-assistant/core
  - label: Utvidelsesmodell
    wert: Integrasjoner består av domenelogikk og plattformer; Config Entries styrer den persistente livssyklusen
    href: https://developers.home-assistant.io/docs/architecture_components/
  - label: Objektmodell
    wert: Config Entry → enhet → entitet → tilstand; registre stabiliserer identiteter, navn og tilordninger
    href: https://developers.home-assistant.io/docs/architecture/devices-and-services/
  - label: Støttet installasjon
    wert: Home Assistant OS som administrert appliance eller Home Assistant Container på en vert som driftes selv
    href: https://www.home-assistant.io/faq/ha-vs-hassio/
  - label: HAOS-stack
    wert: Buildroot, Linux, systemd, Docker, Supervisor, Core og apper; RAUC oppdaterer operativsystemet
    href: https://developers.home-assistant.io/docs/operating-system/
  - label: Grensesnitt
    wert: REST via /api og WebSocket via /api/websocket på samme HTTP-endepunkt som frontend
    href: https://developers.home-assistant.io/docs/api/rest/
  - label: Standardport
    wert: TCP 8123 for frontend, REST og WebSocket; TLS eller reverse proxy endrer den eksterne tilgangsveien
    href: https://www.home-assistant.io/integrations/http/
  - label: Historikk
    wert: Recorder skriver tilstander og utvalgte hendelser via SQLAlchemy som standard til SQLite; MariaDB, MySQL og PostgreSQL støttes
    href: https://www.home-assistant.io/integrations/recorder/
  - label: Automatiseringer
    wert: Triggere starter en kjøring, Conditions avgjør, Actions bruker samme sekvenssemantikk som Scripts
    href: https://www.home-assistant.io/docs/automation/basics/
  - label: Gjenoppretting
    wert: krypterte sikkerhetskopier kan gjenopprette konfigurasjon, Core og apper; nøkler, radiokontrollere og eksterne databaser forblir egne avhengigheter
    href: https://www.home-assistant.io/common-tasks/general/
werbung:
  - newsletter
ctaThemen:
  - smart-home-iot
translationSourceHash: 204801ccce2af55eaa473c7a7599bd0744a8e9db9b0e1dbdb99efe8949a6fff1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:00:54.115Z
translationReview: automatic
---

# Home Assistant: arkitektur, datamodell og drift

Home Assistant er en sentral styrings- og automatiseringsplattform for enheter, radionettverk, IP-tjenester og brukergrensesnitt. Instansen samler tilstander via integrasjoner, normaliserer dem til entiteter, distribuerer endringer via en Event Bus og utfører handlinger basert på dette. «Lokal» betegner her en arkitekturpreferanse, ikke en generell egenskap ved hver integrasjon: En Zigbee-lyskilde kan være fullt tilgjengelig lokalt, mens en produsentintegrasjon utelukkende henter tilstandene sine fra en sky-API. Den offisielle [arkitekturoversikten](https://developers.home-assistant.io/docs/architecture_index/) skiller mellom operativsystem, Supervisor og Core; [integrasjonsarkitekturen](https://developers.home-assistant.io/docs/architecture_components/) beskriver utvidelsen av Core med Python-komponenter.

For administratorer er Home Assistant derfor verken bare et dashboard eller en universell protokollkonverter. Det er en tilstandsfull orkestrator med flere mulige feilpunkter: Python-kjøretid, integrasjoner, registre, database, autentisering, lokale nettverk, radiokontrollere, meglere, produsentskyer og eventuelt Supervisor-apper. Et grønt grensesnitt beviser bare at frontendbanen fungerer. Det beviser ikke at hendelser kommer frem i tide, at enheter er tilgjengelige, at automatiseringer kjører deterministisk eller at en sikkerhetskopi inkludert eksterne avhengigheter kan gjenopprettes.

Forklaringen følger en enhetshendelse via integrasjon, Event Bus og State Machine til automatisering og handling. Deretter settes persistens, tilleggsapper, sikkerhet, overvåking og gjenoppretting i sammenheng.

## Arkitekturtilnærming: sentral hendelses- og tilstandsknute

Home Assistant Core er hendelsesdrevet. Fire dokumenterte byggeblokker utgjør kjernen ([Core architecture](https://developers.home-assistant.io/docs/architecture/core/)):

1. **Event Bus** distribuerer hendelser til registrerte lyttere.
2. **State Machine** holder den sist kjente tilstanden for hver innlastede entitet og publiserer `state_changed`.
3. **Service Registry** administrerer kallbare handlinger og behandler tjenestekall.
4. **Timer** genererer tidshendelser for tidsavhengig behandling.

Integrasjoner oversetter enhets- eller tjenestetilstander til denne modellen. En integrasjon kan polle, motta push-hendelser, bruke lokale biblioteker eller kontakte en ekstern API. Home Assistant standardiserer den resulterende tilstanden, ikke transporten. Dette er den viktigste driftsgrensen: To entiteter med identisk domene, for eksempel `light`, kan ha helt ulike veier for latens, autentisering og gjenoppretting.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-home-assistant.svg?v=20260813" title="Interaktive Infografik: Home Assistant von Geräten und Protokollbrücken über Integrationen, Registries, Event Bus, State Machine, Automationen und Recorder bis zu APIs, Supervisor, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-home-assistant.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Kjøretidslag og installasjonsmodeller

Home Assistant tilbyr to støttede installasjonsmodeller. **Home Assistant OS** er en administrert appliance. **Home Assistant Container** kjører Home Assistant Core som en container på en vert operatøren har ansvar for. Den offisielle sammenligningen anbefaler HAOS for nesten alle installasjoner og beskriver Container som en selvstendig Core-installasjon uten Supervisor-apper ([HAOS eller Container](https://www.home-assistant.io/faq/ha-vs-hassio/)).

### Home Assistant OS

HAOS bygges med Buildroot og består av Linux, GNU C Library, systemd og Docker. SquashFS bærer de skrivebeskyttede systemområdene, ZRAM midlertidige filsystemer og swap, AppArmor begrenser prosesser, og RAUC oppdaterer operativsystemet ([Home Assistant Operating System](https://developers.home-assistant.io/docs/operating-system/)). Over dette administrerer **Supervisor** Core, apper, DNS, lyd, mDNS, sikkerhetskopier og oppdateringer ([Supervisor](https://developers.home-assistant.io/docs/supervisor/)).

Appliance-modellen reduserer variasjoner, men gir Supervisor omfattende ansvar. En feil kan ligge på minst fem nivåer: oppstarts-/OS-spor, Docker Engine, Supervisor, Core-container eller enkeltapp. Supervisor kan rulle tilbake en mislykket Core-oppdateringsbane; den oppdager ikke automatisk faglig feilaktig enhets- eller databaseatferd.

### Home Assistant Container

Container leverer bare Core. Vertens operativsystem, container-engine, nettverk, volumer, database, megler, radioserver, reverse proxy, sikkerhetskopiering og oppdateringer er operatørens ansvar. Supervisor-apper er separat pakkede tjenester. I et containerdesign drives Mosquitto, Matter Server, Zigbee2MQTT, Z-Wave JS UI, PostgreSQL eller en reverse proxy som egne arbeidslaster med egne volumer, versjoner og helsesjekker.

Fordelen er en eksplisitt plattformarkitektur; prisen er en større driftsflate. En sikkerhetskopi av Core-volumet inneholder for eksempel verken den eksterne Recorder-databasen, meglerstatus, radio-NVM eller reverse-proxy-nøkler dersom disse ligger utenfor.

### Historiske installasjonsformer

Den tidligere **Core**-installasjonen i et Python-miljø og **Supervised**-installasjonen på selvadministrert Linux ble avviklet i 2025. Siden utgave 2025.12 regnes de som ikke støttet; 32-bitsarkitekturene `i386`, `armhf` og `armv7` mistet samtidig utgivelsesbanen. Prosjektkunngjøringen oppgir HAOS og Container som gjenværende modeller ([avvikling av Core og Supervised](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)). Et historisk installasjonsnavn må derfor ikke forveksles med programvarekomponenten **Home Assistant Core**, som fortsatt kjører både i HAOS og Container.

## Teknologistack

Home Assistant Core er en Python-applikasjon under Apache-2.0-lisensen. Det offisielle [Core-repositoriet](https://github.com/home-assistant/core) viser Python, asyncio og den modulære integrasjonsstrukturen. I/O-tunge integrasjoner skal ikke blokkere: Kvalitetsreglene foretrekker asynkrone avhengigheter, slik at nettverks- og enhetskall ikke stopper den felles event-loopen ([async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)). Blokkerende bibliotekskode flyttes til executor-tråder; CPU-intensivt eller dårlig begrenset arbeid er likevel en risiko for kapasitet og latens.

Den synlige stacken omfatter mer enn Python:

| Lag | Typisk teknologi | Driftsrelevans |
|---|---|---|
| Frontend | Nettleserapplikasjon, HTTP og WebSocket | Bruker- og sanntidsbane |
| Core | Python, asyncio, integrasjoner | Tilstander, hendelser, handlinger, autentisering |
| Persistens | JSON-baserte konfigurasjonslagre, YAML, SQLAlchemy/SQL | Konfigurasjon, registre, historikk |
| HAOS | Buildroot, Linux, systemd, Docker, AppArmor, RAUC | Appliance-livssyklus og isolasjon |
| Tjenester | Supervisor-apper eller eksterne containere/verter | MQTT, Matter, database, proxy, fildelinger |
| Edge | Radiokontrollere, protokollbroer, enhets- og sky-API-er | Fysisk tilgjengelighet og dataopprinnelse |

[Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/) vurderer integrasjoner etter konfigurasjonsflyt, tester, typing, diagnostikk, effektiv databruk og asynkron atferd. Et høyt nivå forbedrer forventet vedlikeholdbarhet, men er ingen tilgjengelighets-SLA for den underliggende enheten eller skyleverandøren.

Etter installasjonsmodell og kjøretid følger datamodellen. Bare når Config Entry, enhet, entitet og tilstand holdes fra hverandre, kan doble entiteter, manglende enheter og feilaktige automatiseringer forklares rent.

## Objektmodell: Config Entry, enhet, entitet og tilstand

Den driftsmessige inventarnøkkelen er ikke den synlige flisen, men kjeden av konfigurasjon, enhetsidentitet og entitet.

### Config Entries

En **Config Entry** lagrer den persistente konfigurasjonen for en integrasjonsinstans. En UI-konfigurasjonsflyt oppretter den; alternativer, rekonfigurering, reload, unload, fjerning og migrering er definerte livssyklusoperasjoner. Integrasjoner må ikke mutere Entry-data direkte, men må bruke Config-Entry Manager ([Config entries](https://developers.home-assistant.io/docs/config_entries_index/)). En autentiseringsfeil, en ikke-lastet Entry og en utilgjengelig motpart er derfor forskjellige tilstander.

### Enheter og registre

**Device Registry** grupperer tekniske endepunkter i enheter. Identifikatorer eller Connections, for eksempel serienummer og MAC-adresse, brukes til matching; `via_device` kan representere en bro- eller foreldrerelasjon ([Device registry](https://developers.home-assistant.io/docs/device_registry_index/)). En Zigbee-sensor kan dermed vises som en tilkoblet enhet via en Coordinator, uten at Coordinator er applikasjonstilstanden dens.

**Entity Registry** gir entiteter med `unique_id` en varig identitet og hindrer kolliderende Entity IDs. IP-adresse, vertsnavn, URL, brukernavn eller e-postadresse anses uttrykkelig ikke som stabile Unique IDs ([Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)). Dette forklarer hvorfor manuell endring av navnet på en vert ikke kan erstatte enhetsidentiteten, og hvorfor integrasjonsmigreringer krever stabile produsentidentifikatorer.

### Entitet og tilstand

En **entitet** representerer en funksjon eller måleverdi: `sensor`, `switch`, `light`, `climate`, `binary_sensor` eller et annet domene. Tilstanden består av en primær State, attributter, endringstidspunkter og Context. State Machine holder bare den sist kjente tilstanden. `unavailable` betyr at entiteten for øyeblikket ikke forsynes av et aktivt Entity-objekt; `unknown` betyr at det ikke finnes en brukbar verdi. «Siste verdi» er dermed ikke automatisk en «fersk måleverdi».

Den dokumenterte [samhandlingen mellom enheter og tjenester](https://developers.home-assistant.io/docs/architecture/devices-and-services/) skiller mellom Entity Integration, Entity Component, Entity Platform og produsentspesifikk integrasjon. For diagnostikk spørres det derfor alltid:

- Hvilken Config Entry eier entiteten?
- Hvilken integrasjon og plattform oppretter den?
- Hvilken stabil enhets- og Entity ID knytter sammen historikk og konfigurasjon?
- Polles det eller brukes push?
- Hvilken tids- og tilgjengelighetsmodell har kildeverdien?
- Hvilken bro, hvilket bibliotek, sky-API eller radiostrekning ligger foran?

## Integrasjoner og feilisolering

En integrasjon definerer et domene og kan tilby plattformer som `sensor`, `light` eller `switch`. Plattformen abstraherer entitetstypen; enhetsintegrasjonen snakker den konkrete protokollen. Innebygde integrasjoner leveres med Core og testes gjennom utgivelsesprosessen. **Custom Integrations** kjører imidlertid i samme Python-prosess og kan påvirke importer, event-loop, oppstartstid eller minnebruk. Katalogen `/config/custom_components` er derfor del av inventar, endringshåndtering og gjenoppretting.

Den offisielle [integrasjonsoversikten](https://www.home-assistant.io/integrations/) skiller blant annet mellom IoT-klasser som Local Push, Local Polling, Cloud Push og Cloud Polling. Denne klassifiseringen er mer nyttig for driftsmodeller enn en lang produsentliste:

| Klasse | Databane | Typisk feilområde |
|---|---|---|
| Local Push | Enhet eller bro sender til LAN | Multicast, brannmur, bro, subnett |
| Local Polling | Core spør lokal enhet | Latens, timeout, spørreintervall, enhetskapasitet |
| Cloud Push | Skyen sender eller strømmer hendelser | Internett, konto, token, leverandørstrøm |
| Cloud Polling | Core spør leverandør-API | Rate limit, token, Internett, API-endring |
| Calculated/Internal | Core beregner tilstand | Inndata, maler, tid, omstartstilstand |

Integrasjonen er ingen prosessisolator. En ryddig feilavgrensning deaktiverer eller laster målrettet inn den berørte Config Entry på nytt før hele Core startes på nytt. En omstart ødelegger flyktig bevismateriale og kan tilbakestille tidsavhengige automatiseringstimere.

## Protokoll- og nettverksmodell

For Home Assistant er en **avhengighetsgraf** mer nyttig enn en generell OSI-tabell. Plattformen ligger på applikasjonsnivå, men databanene forgrener seg:

- Frontend, REST og WebSocket kjører over HTTP på TCP, som standard på port 8123.
- DNS løser opp verts- og skytjenester; mDNS og SSDP oppdager enheter i det lokale nettet.
- MQTT bruker en separat megler og en publish/subscribe-modell over TCP eller WebSocket.
- Zigbee, Z-Wave, Thread og Bluetooth krever radiokontrollere eller nettverksproxyer.
- Matter bruker IP-kommunikasjon, men krever en Matter-server og eventuelt en Thread Border Router for provisjonering og Fabric-drift.
- Produsentintegrasjoner kan bruke HTTPS, proprietære lokale protokoller eller skystrømmer.

De innebygde oppdagelsesintegrasjonene dokumenterer [mDNS/Zeroconf](https://www.home-assistant.io/integrations/zeroconf/) og [SSDP](https://www.home-assistant.io/integrations/ssdp/). Begge er avhengige av segmenter og multicast. En reverse proxy for frontend reparerer ikke oppdagelse på tvers av VLAN-grenser. Multicast-relé, IGMP-snooping, WLAN-klientisolasjon, IPv6-RA, DNS-suffikser og brannmurregler kontrolleres per faktisk enhetsbane.

### MQTT som eget tilstandsrom

MQTT er ikke den interne Event Bus. Det er en ekstern meglertjeneste som en integrasjon kommuniserer med. Den offisielle [MQTT-integrasjonen](https://www.home-assistant.io/integrations/mqtt/) beskriver Discovery Topics, retained messages, Birth/Last Will, Availability, TLS og MQTT 5. Retained Discovery kan gjenopprette enheter etter omstart, men kan også bevare foreldede Ghost Entities. Tilgjengelighet krever egen semantikk; en eksisterende retained State beviser ikke at publisheren fortsatt lever.

Robust MQTT-drift inventariserer megler, Client IDs, autentisering, CA, Topics, QoS, Retain, Expiry, Birth/Will og Discovery Origin. Meglersikkerhetskopi og Core-sikkerhetskopi er separate beskyttelsesobjekter.

### Radio- og brobaner

[ZHA](https://www.home-assistant.io/integrations/zha/) integrerer en Zigbee-Coordinator, [Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/) bruker en separat Z-Wave-JS-server, og [Matter](https://www.home-assistant.io/integrations/matter/) kobler til en Matter-server. [Thread](https://www.home-assistant.io/integrations/thread/) administrerer relasjoner til Border Router og nettverk, men er ikke identisk med Matter. Radioenheter, kontrollerfastvare, nettverksdata, nøkkelmateriale og enhetskonfigurasjon utgjør hver sin gjenopprettingsgruppe. Å flytte en USB-pinne eller bytte ut en Coordinator er ingen vanlig IP-adresseendring.

Integrasjoner leverer tilstander og hendelser; automatiseringer reagerer på dem. Forløpet deres fra Trigger, Condition og Action må derfor diagnostiseres separat fra enhetskonfigurasjonen.

## Automatiseringskjøretid: Trigger, Condition, Action

En automatisering er en reaktiv kjøringsdefinisjon. [Grunnleggende om automatiseringer](https://www.home-assistant.io/docs/automation/basics/) skiller mellom Trigger, valgfrie Conditions og Actions. Trigger oppretter en kjøring, Conditions kontrollerer inngangstilstanden, og Actions bruker skriptenes sekvenssemantikk ([Actions](https://www.home-assistant.io/docs/automation/action/)).

Tidspunktet er viktig: State, attributter og malverdier kan endres mellom Trigger og en senere Action. En forsinkelse holder ikke en transaksjon åpen. Flere kjøringer av samme automatisering krever derfor en modus som Single, Restart, Queued eller Parallel og en bevisst konfliktmodell. Fysiske aktuatorer er sjelden transaksjonelle; en delvis utført kjøring trenger eventuelt kompenserende handlinger.

[Trigger-dokumentasjonen](https://www.home-assistant.io/docs/automation/trigger/) påpeker at `for`-ventetider ikke overlever en omstart eller reload av automatiseringen. Den som må bevare en frist over omstarter, persisterer et tidspunkt, for eksempel i `input_datetime`, og utløser mot dette. Conditions er bare kontroller i gjeldende kjøring; [Condition-semantikken](https://www.home-assistant.io/docs/scripts/conditions/) gjør dem ikke til en lås mot parallelle endringer.

Maler evalueres i Home Assistant med Jinja-uttrykk. Inndata- og typefeil, `unknown`, `unavailable`, tidssoner og implisitt strengkonvertering skal inngå i tester. [Templating-dokumentasjonen](https://www.home-assistant.io/docs/automation/templating/) beskriver triggeravhengige variabler. En administrator tester ikke bare happy path, men også omstart, manglende entitet, forsinket hendelse, dobbel Trigger og aktuatorfeil.

## Konfigurasjon, registre og Source of Truth

Home Assistant kombinerer UI-styrte Config Entries, registerdata og YAML. `configuration.yaml` er roten til manuell konfigurasjon, men ikke hele Source of Truth. Den offisielle [konfigurasjonsoversikten](https://www.home-assistant.io/docs/configuration/) skiller mellom UI og YAML; Packages kan strukturere tilhørende YAML-blokker ([Packages](https://www.home-assistant.io/docs/configuration/packages/)).

Bare den tekstlige delen som er renset for hemmeligheter, egner seg for Git og review. `secrets.yaml` skiller verdier fra YAML, men krypterer dem ikke; [herdeveiledningen](https://www.home-assistant.io/docs/configuration/securing/) påpeker dette uttrykkelig. UI-tilstand, registre, tokens og Config Entries ligger i konfigurasjonslageret og endres via støttede UI-/API-veier. Direkte redigering av interne storage-filer mens Core kjører, omgår skjema-, livssyklus- og konsistenslogikk.

Et konfigurasjonsinventar omfatter:

- YAML, Packages, Blueprints og Custom Components,
- Config Entries inkludert opphav, Owner og autentisering,
- Device-, Entity- og Area-tilordninger,
- automatiseringer, Scripts, scener og dashboards,
- brukere, tokens, MFA og eksterne Identity Providers,
- Supervisor-apper eller eksterne tjenester,
- radiokontrollere, megler, database og proxy,
- Secrets, sertifikater og gjenopprettingsnøkler.

## Recorder, historikk og langtidsstatistikk

State Machine holder gjeldende tilstand i minnet. Historikk oppstår først gjennom **Recorder**. Den skriver tilstandsendringer og utvalgte hendelser via SQLAlchemy til en database; History, Activity, diagrammer og langtidsstatistikk leser derfra. Den offisielle [Recorder-dokumentasjonen](https://www.home-assistant.io/integrations/recorder/) oppgir SQLite som standard og anbefaling, samt MariaDB, MySQL og PostgreSQL som støttede alternativer.

Recorder-data er ingen hendelseskilde for sanntidsstyring. En databasefeil kan påvirke historikk og statistikk mens gjeldende tilstander og automatiseringer delvis fortsetter å kjøre. Omvendt beviser ikke en komplett historikk at en Action på den fysiske enheten lyktes.

De viktigste driftsparameterne er:

- `purge_keep_days` for råhistorikk,
- Include-/Exclude-filtre for entiteter og hendelser,
- `commit_interval` som forhold mellom I/O og tapsvindu,
- databasestørrelse, ledig lagringsplass og skrivelatens,
- Purge og Repack,
- oppstartsrekkefølge og tilgjengelighet for eksterne databaser,
- langtidsstatistikk og konsistens i metadata.

Bytte av Recorder-database migrerer ikke eksisterende historikk på en støttet måte. Eksterne databaser trenger egne konsistente sikkerhetskopier og gjenopprettingstester. For SQLite oppgir dokumentasjonen minst 2,5 ganger databasestørrelsen i ledig plass for korrupsjonshåndtering. Lagring og Recorder er dermed en egen kapasitet- og gjenopprettingsbane, ikke bare en valgfri cache.

## API, WebSocket og autentisering

Frontend og API-er deler som standard samme HTTP-listener. [REST API](https://developers.home-assistant.io/docs/api/rest/) bruker JSON og Bearer Tokens; basisbanen er `/api/`. [WebSocket API](https://developers.home-assistant.io/docs/api/websocket/) ligger under `/api/websocket`, går gjennom `auth_required`, `auth` og `auth_ok` og korrelerer kommandoer via numeriske ID-er. WebSocket leverer hendelsesstrømmer og registre mer effektivt enn gjentatt REST-polling.

Langvarige tokens er brukerlegitimasjon. [Authentication API](https://developers.home-assistant.io/docs/auth_api/) beskriver OAuth/IndieAuth, Refresh Tokens, Long-Lived Access Tokens og kortvarige Signed Paths. Et token arver konteksten til brukeren; et Long-Lived Token som er gyldig i ti år hører hjemme i en Secrets-lagring, ikke i YAML, shell-historikk, URL eller dashboard-JavaScript.

En API-monitor kontrollerer minst autentisering, `/api/config`, forventede entiteter, `last_updated`, WebSocket-subscription og en ufarlig lese-/handlingsbane. En HTTP-200 på `/` kontrollerer kun frontend-tilgjengelighet.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) og [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) behandler Windows-spørringen; [`curl`](https://curl.se/docs/manpage.html) og [`jq`](https://jqlang.org/manual/) gjør det samme på Unix. Tokenet vises bare som prosessmiljøvariabel; i produksjon kommer det fra en kontrollert Secrets-lagring.

## HTTP, TLS og reverse proxy

HTTP-endepunktet lytter som standard på TCP 8123. Direkte TLS, reverse proxy og Home Assistant Cloud er forskjellige tilgangsmodeller. Med en tradisjonell reverse proxy må `use_x_forwarded_for` og `trusted_proxies` settes korrekt; ellers er klient-IP-en feil eller forespørselen avvises ([HTTP integration](https://www.home-assistant.io/integrations/http/)). En bred liste over klarerte proxyer gjør det mulig å forfalske Forwarded-For-informasjon.

[Sikkerhetsveiledningen](https://www.home-assistant.io/docs/configuration/securing/) anbefaler entydige passord, MFA, minimale administratorrettigheter og sikret fjerntilgang i stedet for direkte eksponering mot Internett. TLS avslutter bare transporten. Tokenrettigheter, proxy-headere, WebSocket-oppgraderinger, rate limits, DNS, sertifikatfornyelse og sikkerheten til den foranliggende IdP-en forblir egne kontroller.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) og [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) kontrollerer Windows; [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerer Unix. Det uautentiserte `/api/`-kallet kan gi 401; avgjørende er navneoppløsning, TLS-identitet, proxy-rute og forventet autentiseringsgrense.

## MQTT-diagnostikk

Meglerstatus kontrolleres utenfor Home Assistant. En subscriber observerer Discovery, Availability og State uten å endre Topics. En publish-test bruker en særskilt reservert testbane; produktive Command Topics beskrives ikke tilfeldig.

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

[`mosquitto_sub`](https://mosquitto.org/man/mosquitto_sub-1.html) og [`mosquitto_pub`](https://mosquitto.org/man/mosquitto_pub-1.html) er de offisielle meglerklientene. Passord på kommandolinjen kan være synlige i prosesslister eller historikk; eksemplene illustrerer banen, mens produksjonskallet bruker passordfil, operativsystemets Secrets-lagring eller kortvarige legitimasjoner.

## Drift av HAOS og Container

HAOS tilbyr kommandoen `ha` via terminal-/SSH-tilgang. Containerinstallasjoner driftes med verktøyene til den valgte runtime-en. En diagnosepakke samler systeminformasjon, Core-logg, integrasjonsdiagnostikk, containerstatus, ledig plass og tidspunkt.

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

[`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/), [`docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/) og [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) gir containerstatus. [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) og [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) behandler Windows-utdata; [`grep`](https://www.gnu.org/software/grep/manual/grep.html), [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) og [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html) kompletterer Unix. En kjørende container er bare første kontroll; deretter følger integrasjons-, register-, hendelses- og enhetsbane.

Ved feilsøking leses signalveien baklengs: handling, Automation Trace, tilstandsendring, integrasjon, nettverksprotokoll og fysisk enhet.

## Observability og systematisk diagnostikk

**System Health** samler installasjonstype, arkitektur, Python-, Core- og frontendinformasjon og tilbyr diagnosefunksjoner via Innstillinger > System > Reparasjoner ([System Health](https://www.home-assistant.io/integrations/system_health/)). [Logger-integrasjonen](https://www.home-assistant.io/integrations/logger/) styrer globale og komponentspesifikke loggnivåer. Debug-logging begrenses i tid og til berørte namespaces; radio- eller hendelsesstormer kan ellers dominere minne og I/O.

En robust diagnosekjede er:

1. **Symptom og ønsket tilstand:** Hvilken entitet, Action, automatisering eller overflate er berørt?
2. **Tid og omfang:** Siden når, for hvilke enheter, brukere, nettverk og integrasjonsinstanser?
3. **Objektidentitet:** Sikre Config Entry, Device ID, Entity ID, Unique ID og broreferanse.
4. **Kjøretid:** Kontroller Core, event-loop, minne, CPU, filsystem og database.
5. **Integrasjon:** Kontroller Entry-status, autentisering, Coordinator-/pollingstatus og diagnostikknedlasting.
6. **Transport:** Kontroller Discovery, DNS, TCP, TLS, megler, radiokontroller eller produsent-API.
7. **Automatisering:** Kontroller Trace, triggerdata, Conditions, Run Mode og handlingsresultat.
8. **Persistens:** Vurder Recorder-forsinkelse og historikk separat fra live-tilstanden.
9. **Kontrollert test:** Bruk read-only eller en ufarlig testentitet.
10. **Gjenoppretting:** Reload før restart, restart før restore; sikre bevismateriale på forhånd.

En entitet `unavailable` kan skyldes en utlastet Config Entry, manglende bro, radiotap eller kilde-timeout. Et synlig gammelt tall er farligere fordi det virker plausibelt. Overvåking trenger derfor friskhetsgrenser, ikke bare verdigrenser.

## Oppdateringer, utgivelser og Custom Integrations

Home Assistant publiserer hyppige Core-utgivelser og dokumenterer bakoverinkompatible endringer. En statisk referanseartikkel fryser bevisst ikke en gjeldende versjonsstatus. Utrullingen kontrollerer i stedet Release Notes, integrasjonsendringer og mål-avhengigheter på vedlikeholdstidspunktet.

En kontrollert oppdateringsbane omfatter:

1. Bekreft sikkerhetskopi og uavhengig nedlasting eller ekstern lagringsplass.
2. Kontroller ledig plass, databasestatus og System Health.
3. Vurder Release Notes samt berørte integrasjoner og Custom Components.
4. Inventariser radio-, megler-, database- og proxyavhengigheter.
5. Oppdater Core eller HAOS og apper i definert rekkefølge.
6. Kontroller oppstartslogg, reparasjoner og registermigreringer.
7. Test kritiske sensor-, aktuator-, automatiserings-, API- og fjernaksessbaner.
8. Bestem feilgrensen og utløs først deretter rollback eller restore.

HAOS bruker RAUC med to operativsystemspor; `ha os info` og `rauc status` viser sporstatusen ([HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)). Denne mekanismen beskytter OS-oppdateringsbanen, ikke automatisk Core-konfigurasjon, Recorder-data eller radionettverkstilstand.

Når kjøretid og databane er kjent, kan omfanget av sikkerhetskopieringen bestemmes. Konfigurasjon, registre, Secrets, database og Add-on-tilstander må passe sammen med den valgte installasjonsmodellen.

## Sikkerhetskopiering og gjenoppretting

Home Assistant kan skrive automatiske og manuelle, krypterte sikkerhetskopier til lokale eller eksterne mål. Den offisielle [veiledningen for backup og restore](https://www.home-assistant.io/common-tasks/general/) beskriver sikkerhetskopieringssteder, Emergency Kit, nedlasting, restore ved onboarding og migrering til annen maskinvare. Siden 2026 er krypteringsmodellen for sikkerhetskopier modernisert; [kunngjøringen om backupkryptering](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/) dokumenterer formatendringen og kompatibilitetsgrensene.

En sikkerhetskopi er bare komplett relativt til installasjonsmodellen:

| Objekt | HAOS-sikkerhetskopi | Ansvar for Container/eksternt |
|---|---|---|
| Core-konfigurasjon og registre | kan inkluderes | sikre Config-volum |
| Supervisor-apper | appdata kan inkluderes | separate containere og volumer |
| Recorder SQLite | i Config-området | konsistent DB-sikkerhetskopi ved ekstern DB |
| MQTT-megler | bare ved passende appvalg | meglerkonfigurasjon og persistens separat |
| Zigbee/Z-Wave/Matter | integrasjonsdata delvis | kontroller-/serversikkerhetskopi og nøkler kontrolleres separat |
| TLS/proxy/DNS | bare hvis innenfor valgte data | ekstern infrastruktur separat |
| Backupnøkler | ikke tilstrekkelig i den krypterte sikkerhetskopien selv | Emergency Kit oppbevares separat |

En gjenopprettingstest avsluttes ikke ved innlogging. Akseptansekriteriene er: Config Entries lastet, registre konsistente, brukertilgang mulig, database uten feil, megler og broer tilkoblet, radioenheter kontrollerbare, kritiske automatiseringer testet og fjerntilgang med riktig sertifikat tilgjengelig. Batterienheter kan først sove etter en migrering; en manglende umiddelbar verdi må ikke forhastet vurderes som datatap.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv) og [`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content) oppretter eller leser Windows-manifestet. [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html), [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) og [`tar`](https://www.gnu.org/software/tar/manual/html_node/index.html) håndterer Unix. En sjekksum beviser at arkivet er uendret; bare gjenopprettingstesten beviser at det kan dekrypteres og gjenopprettes faglig korrekt.

## RPO, RTO og høy tilgjengelighet

Home Assistant er i vanlig drift en tilstandsfull enkeltinstans. To aktive Core-instanser mot de samme enhetene, registrene eller meglerkommandoene gir ikke automatisk koordinert høy tilgjengelighet. Doble automatiseringer kan slå aktuatorer flere ganger; radiokontrollere og lokale enheter tillater ofte bare ett aktivt eierskap.

En realistisk resiliensmodell kombinerer:

- pålitelig enkeltnode eller VM med overvåkede ressurser,
- UPS og egnet lagring i stedet for sårbare flashmedier,
- separate, automatiske og krypterte sikkerhetskopier,
- dokumentert reserve-maskinvare eller VM-målplattform,
- eksporterbare radiokontrollertilstander og nøkler,
- reproduserbare eksterne tjenester,
- kontrollert restore med entydig overtakelse av enhet og nettverk.

**RPO** avhenger av sist sikrede konfigurasjon, register, app- og eksterne tjenestekopi. Recorder-historikk kan ha et annet RPO enn automatiseringskonfigurasjon. **RTO** omfatter ikke bare Core-oppstart, men også DNS, proxy, database, megler, radiokontroller, enhetsreconnect, sovende sensorer og akseptansetester.

## Sikkerhet og tillitsgrenser

Home Assistant kan styre dører, oppvarming, alarmsystemer og energistrømmer. Virkeområdet er dermed fysisk. Sikkerhetsdesign skiller mellom:

- brukere og administratorer,
- nettleser-, Companion App- og API-sesjoner,
- Long-Lived Tokens og webhooks,
- Core og Custom Integrations,
- Supervisor-apper eller eksterne containere,
- IoT-, administrasjons- og brukersegmenter,
- lokale enheter og produsentskyer,
- radionettverk og nøklene deres,
- sikkerhetskopieringsmål og Emergency Kit.

MFA beskytter interaktive kontoer, men ikke et stjålet Long-Lived Token. Nettverkssegmentering begrenser lateral bevegelse, men må ikke ukontrollert blokkere nødvendige Discovery- og returkanaler. Custom Integrations får prosessnærhet til Core og behandles som kodeutrullinger. Secrets skal verken vises i Git, diagnosefiler eller supportinnlegg. Generelle kontroller finnes under [herding](/kb/haertung), grunnleggende om transport under [TLS](/kb/tls) og API-kontraktsmodeller under [API-er](/kb/apis).

## Teknisk historie

Home Assistant startet i 2013 som et Python-prosjekt av Paulus Schoutsen. Tilbakeblikket på tiårsjubileet beskriver utviklingen fra en liten lokal automatiseringsapplikasjon til et stort Open Source-prosjekt ([10 år med Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)). Python-kjernen og integrasjonsmodellen forble den faglige kjernen, mens frontend, mobile klienter, Supervisor, HAOS, enhetsmaskinvare og skyvalg utviklet seg rundt den.

Med Hass.io, senere Home Assistant og deretter Home Assistant OS og Supervisor, oppsto en appliance-stack med operativsystem, containeradministrasjon, Core og tilleggstjenester. Skillet er språklig ryddet opp flere ganger: «Add-ons» heter nå **Apps**, mens «Integrations» fortsatt er Python-utvidelser av Core. Disse begrepene beskriver ulike kjørings- og sikkerhetsgrenser.

I 2024 gikk Home Assistant inn i den ideelle Open Home Foundation; Nabu Casa forble kommersiell partner. Prosjektkunngjøringen om [Open Home-økosystemet](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/) beskriver eierskap og styring. I 2025 reduserte prosjektet de støttede installasjonsvariantene til HAOS og Container. Den historiske trenden går dermed ikke mot en distribuert klynge, men mot en mer stabil sentral Core med tydeligere støttede kjøretidspakker og selvstendige protokollservere.

## Admin-sjekkliste

Et grønt dashboard er ikke tilstrekkelig driftsbevis. Sjekklisten knytter installasjon, enhetsbaner, automatiseringer, datalagring og gjenoppretting til en kontrollerbar helhetsoversikt.

- **Installasjon:** Dokumenter HAOS eller Container, arkitektur, vert, lagring, nettverk og eierskap.
- **Stack:** Skill mellom Core, Supervisor, apper/eksterne containere, database, megler, proxy og radioserver.
- **Inventar:** Registrer Config Entry, Device ID, Entity ID, Unique ID, Area og `via_device`.
- **Dataopprinnelse:** Merk Local/Cloud og Push/Polling for hver kritiske integrasjon.
- **Tilstand:** Skill mellom `unknown`, `unavailable`, foreldet verdi og bekreftet enhetssuksess.
- **Automatisering:** Test Trigger, Context, Condition, Run Mode, omstartsatferd og kompensasjon.
- **API-er:** Kontroller brukerkontekst, tokenlagring, WebSocket, reverse proxy og TLS.
- **Recorder:** Overvåk database, filtre, commit-intervall, Purge, I/O, vekst og sikkerhetskopiering.
- **IoT-nett:** Kontroller mDNS, SSDP, MQTT, VLAN, IPv6 og radio-/brobaner eksplisitt.
- **Oppdateringer:** Samle Release Notes, Custom Integrations, sikkerhetskopi, utrulling og akseptanse.
- **Gjenoppretting:** Test Core, eksterne tjenester, radiotilstand, nøkler og Emergency Kit samlet.
- **Bevis:** Verifiser ikke bare UI og container, men minst én sensor-, aktuator-, automatiserings- og API-bane ende til ende.

## Kilder

- [Home Assistant Developer Docs – arkitekturoversikt](https://developers.home-assistant.io/docs/architecture_index/)
- [Home Assistant Developer Docs – integrasjonsarkitektur](https://developers.home-assistant.io/docs/architecture_components/)
- [Home Assistant Developer Docs – Core architecture](https://developers.home-assistant.io/docs/architecture/core/)
- [Home Assistant – HAOS eller Container](https://www.home-assistant.io/faq/ha-vs-hassio/)
- [Home Assistant Developer Docs – Operating System](https://developers.home-assistant.io/docs/operating-system/)
- [Home Assistant Developer Docs – Supervisor](https://developers.home-assistant.io/docs/supervisor/)
- [Home Assistant – avvikling av Core, Supervised og 32-bit](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
- [GitHub – Home Assistant Core](https://github.com/home-assistant/core)
- [Home Assistant Developer Docs – Async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)
- [Home Assistant Developer Docs – Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/)
- [Home Assistant Developer Docs – Config entries](https://developers.home-assistant.io/docs/config_entries_index/)
- [Home Assistant Developer Docs – Device registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Home Assistant Developer Docs – Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)
- [Home Assistant Developer Docs – enheter og tjenester](https://developers.home-assistant.io/docs/architecture/devices-and-services/)
- [Home Assistant – integrasjoner](https://www.home-assistant.io/integrations/)
- [Home Assistant – Zeroconf](https://www.home-assistant.io/integrations/zeroconf/)
- [Home Assistant – SSDP](https://www.home-assistant.io/integrations/ssdp/)
- [Home Assistant – MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Home Assistant – ZHA](https://www.home-assistant.io/integrations/zha/)
- [Home Assistant – Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/)
- [Home Assistant – Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant – Thread](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant – grunnleggende om automatiseringer](https://www.home-assistant.io/docs/automation/basics/)
- [Home Assistant – Automation actions](https://www.home-assistant.io/docs/automation/action/)
- [Home Assistant – Automation triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Home Assistant – Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Home Assistant – Automation templating](https://www.home-assistant.io/docs/automation/templating/)
- [Home Assistant – konfigurasjon](https://www.home-assistant.io/docs/configuration/)
- [Home Assistant – Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [Home Assistant – sikre Home Assistant](https://www.home-assistant.io/docs/configuration/securing/)
- [Home Assistant – Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant Developer Docs – REST API](https://developers.home-assistant.io/docs/api/rest/)
- [Home Assistant Developer Docs – WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)
- [Home Assistant Developer Docs – Authentication API](https://developers.home-assistant.io/docs/auth_api/)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – håndbok](https://curl.se/docs/manpage.html)
- [jq – håndbok](https://jqlang.org/manual/)
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
- [GNU Grep – håndbok](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [GNU Coreutils – du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Home Assistant – System Health](https://www.home-assistant.io/integrations/system_health/)
- [Home Assistant – Logger](https://www.home-assistant.io/integrations/logger/)
- [Home Assistant Developer Docs – HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)
- [Home Assistant – backup og restore](https://www.home-assistant.io/common-tasks/general/)
- [Home Assistant – modernisert backupkryptering](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [GNU Tar – håndbok](https://www.gnu.org/software/tar/manual/html_node/index.html)
- [Home Assistant – 10 år med Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)
- [Home Assistant – Open Home Foundation og styring](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/)
