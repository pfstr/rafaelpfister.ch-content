---
title: "Home Assistant: architettura, modello dati e operatività"
blatt: "home-assistant"
description: "Home Assistant per amministratori di piattaforme, reti e IoT: core Python basato sugli eventi, integrazioni e registri, Home Assistant OS e Container, bridge di protocollo e radio, runtime delle automazioni, Recorder, API, autenticazione, osservabilità, backup e ripristino."
fakten:
  - label: Ruolo del sistema
    wert: piattaforma centrale di controllo e automazione basata sugli eventi per dispositivi locali, reti radio e servizi esterni
    href: https://developers.home-assistant.io/docs/architecture_index/
  - label: Core
    wert: Event Bus, State Machine, Service Registry e Timer costituiscono il nucleo di runtime
    href: https://developers.home-assistant.io/docs/architecture/core/
  - label: Stack tecnologico
    wert: Home Assistant Core e le relative integrazioni sono implementati in Python; asyncio supporta l'elaborazione I/O concorrente
    href: https://github.com/home-assistant/core
  - label: Modello di estensione
    wert: le integrazioni sono composte da logica di dominio e piattaforme; i Config Entry ne controllano il ciclo di vita persistente
    href: https://developers.home-assistant.io/docs/architecture_components/
  - label: Modello a oggetti
    wert: Config Entry → dispositivo → entità → stato; i registri stabilizzano identità, nomi e assegnazioni
    href: https://developers.home-assistant.io/docs/architecture/devices-and-services/
  - label: Installazione supportata
    wert: Home Assistant OS come appliance gestita oppure Home Assistant Container su un host autogestito
    href: https://www.home-assistant.io/faq/ha-vs-hassio/
  - label: Stack HAOS
    wert: Buildroot, Linux, systemd, Docker, Supervisor, Core e Apps; RAUC aggiorna il sistema operativo
    href: https://developers.home-assistant.io/docs/operating-system/
  - label: Interfacce
    wert: REST tramite /api e WebSocket tramite /api/websocket sullo stesso endpoint HTTP del frontend
    href: https://developers.home-assistant.io/docs/api/rest/
  - label: Porta standard
    wert: TCP 8123 per frontend, REST e WebSocket; TLS o reverse proxy modificano il percorso di accesso esterno
    href: https://www.home-assistant.io/integrations/http/
  - label: Cronologia
    wert: Recorder scrive stati ed eventi selezionati tramite SQLAlchemy in SQLite per impostazione predefinita; MariaDB, MySQL e PostgreSQL sono supportati
    href: https://www.home-assistant.io/integrations/recorder/
  - label: Automazioni
    wert: i trigger avviano un'esecuzione, le condizioni decidono, le azioni usano la stessa semantica sequenziale degli script
    href: https://www.home-assistant.io/docs/automation/basics/
  - label: Ripristino
    wert: i backup crittografati possono ripristinare configurazione, Core e Apps; chiavi, controller radio e database esterni restano dipendenze separate
    href: https://www.home-assistant.io/common-tasks/general/
werbung:
  - newsletter
ctaThemen:
  - smart-home-iot
translationSourceHash: 204801ccce2af55eaa473c7a7599bd0744a8e9db9b0e1dbdb99efe8949a6fff1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:52:51.842Z
translationReview: automatic
---

# Home Assistant: architettura, modello dati e operatività

Home Assistant è una piattaforma centrale di controllo e automazione per dispositivi, reti radio, servizi IP e interfacce utente. L'istanza raccoglie gli stati tramite integrazioni, li normalizza in entità, distribuisce le modifiche tramite un Event Bus ed esegue azioni di conseguenza. In questo contesto, «locale» indica una preferenza architetturale, non una caratteristica generalizzata di ogni integrazione: una lampadina Zigbee può essere raggiungibile interamente in locale, mentre un'integrazione del produttore può ottenere i propri stati esclusivamente da un'API cloud. La [panoramica dell'architettura](https://developers.home-assistant.io/docs/architecture_index/) ufficiale separa sistema operativo, Supervisor e Core; l'[architettura delle integrazioni](https://developers.home-assistant.io/docs/architecture_components/) descrive l'estensione del Core tramite componenti Python.

Per gli amministratori, Home Assistant non è quindi né soltanto una dashboard né un convertitore universale di protocolli. È un orchestratore con stato e molteplici possibili punti di guasto: runtime Python, integrazioni, registri, database, autenticazione, reti locali, controller radio, broker, cloud dei produttori e, se presenti, app del Supervisor. Un'interfaccia verde dimostra solo che il percorso del frontend funziona. Non dimostra che gli eventi arrivino in tempo, che i dispositivi siano raggiungibili, che le automazioni vengano eseguite in modo deterministico o che un backup, incluse le dipendenze esterne, sia ripristinabile.

La spiegazione segue un evento di dispositivo attraverso integrazione, Event Bus e State Machine fino ad automazione e azione. Successivamente inquadra persistenza, add-on, sicurezza, monitoraggio e ripristino.

## Approccio architetturale: nodo centrale di eventi e stati

Home Assistant Core è basato sugli eventi. Quattro componenti documentati costituiscono il nucleo ([Core architecture](https://developers.home-assistant.io/docs/architecture/core/)):

1. L'**Event Bus** distribuisce gli eventi ai listener registrati.
2. La **State Machine** mantiene l'ultimo stato noto di ogni entità caricata e pubblica `state_changed`.
3. Il **Service Registry** gestisce le azioni invocabili ed elabora le chiamate di servizio.
4. Il **Timer** genera eventi temporali per l'elaborazione dipendente dal tempo.

Le integrazioni traducono gli stati di dispositivi o servizi in questo modello. Un'integrazione può eseguire polling, ricevere eventi push, usare librerie locali o interrogare un'API remota. Home Assistant uniforma lo stato risultante, non il trasporto. Questo è il confine operativo più importante: due entità dello stesso tipo di dominio, ad esempio `light`, possono avere percorsi di latenza, autenticazione e ripristino completamente diversi.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-home-assistant.svg?v=20260813" title="Interaktive Infografik: Home Assistant von Geräten und Protokollbrücken über Integrationen, Registries, Event Bus, State Machine, Automationen und Recorder bis zu APIs, Supervisor, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-home-assistant.svg?v=20260813">Apri direttamente la grafica interattiva</a>.
</iframe>

## Livelli di runtime e modelli di installazione

Home Assistant offre due modelli di installazione supportati. **Home Assistant OS** è un'appliance gestita. **Home Assistant Container** esegue Home Assistant Core come container su un host di responsabilità dell'operatore. Il confronto ufficiale indica HAOS come raccomandazione per quasi tutte le installazioni e descrive Container come installazione Core autonoma senza app del Supervisor ([HAOS o Container](https://www.home-assistant.io/faq/ha-vs-hassio/)).

### Home Assistant OS

HAOS è creato con Buildroot ed è composto da Linux, GNU C Library, systemd e Docker. SquashFS ospita le aree di sistema in sola lettura, ZRAM i file system temporanei e lo swap, AppArmor limita i processi e RAUC aggiorna il sistema operativo ([Home Assistant Operating System](https://developers.home-assistant.io/docs/operating-system/)). Al di sopra, il **Supervisor** gestisce Core, app, DNS, audio, mDNS, backup e aggiornamenti ([Supervisor](https://developers.home-assistant.io/docs/supervisor/)).

Il modello appliance riduce le varianti, ma attribuisce al Supervisor una responsabilità molto ampia. Un errore può trovarsi ad almeno cinque livelli: slot di boot/OS, Docker Engine, Supervisor, container Core o singola app. Il Supervisor può eseguire il rollback di un percorso di aggiornamento Core non riuscito; non riconosce automaticamente un comportamento funzionalmente errato del dispositivo o del database.

### Home Assistant Container

Container fornisce solo il Core. Sistema operativo host, container engine, rete, volumi, database, broker, server radio, reverse proxy, backup e aggiornamenti sono di responsabilità dell'operatore. Le app del Supervisor sono servizi pacchettizzati separatamente. In un'architettura Container, Mosquitto, Matter Server, Zigbee2MQTT, Z-Wave JS UI, PostgreSQL o un reverse proxy vengono eseguiti come workload distinti con propri volumi, versioni e health check.

Il vantaggio è un'architettura di piattaforma esplicita; il prezzo è una superficie operativa maggiore. Un backup del volume Core, per esempio, non contiene né il database Recorder esterno né lo stato del broker, la NVM radio o le chiavi del reverse proxy, se questi risiedono all'esterno.

### Forme di installazione storiche

La precedente installazione **Core** in un ambiente Python e l'installazione **Supervised** su Linux autogestito sono state dismesse nel 2025. Dalla release 2025.12 sono considerate non supportate; le architetture a 32 bit `i386`, `armhf` e `armv7` hanno contemporaneamente perso il percorso di release. L'annuncio del progetto indica HAOS e Container come modelli rimanenti ([disattivazione di Core e Supervised](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)). Un nome storico di installazione non deve quindi essere confuso con il componente software **Home Assistant Core**, che continua a essere eseguito anche all'interno di HAOS e Container.

## Stack tecnologico

Home Assistant Core è un'applicazione Python con licenza Apache-2.0. Il [repository Core](https://github.com/home-assistant/core) ufficiale mostra Python, asyncio e la struttura modulare delle integrazioni. Le integrazioni a elevato I/O non devono bloccare: le regole di qualità privilegiano dipendenze asincrone affinché chiamate di rete e dispositivi non blocchino l'Event Loop condiviso ([async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)). Il codice di libreria bloccante viene spostato in thread Executor; il lavoro a elevato impiego di CPU o mal limitato rimane comunque un rischio per capacità e latenza.

Lo stack visibile comprende più di Python:

| Livello | Tecnologia tipica | Rilevanza operativa |
|---|---|---|
| Frontend | Applicazione browser, HTTP e WebSocket | Percorso utente e tempo reale |
| Core | Python, asyncio, integrazioni | Stati, eventi, azioni, autenticazione |
| Persistenza | Archivi di configurazione basati su JSON, YAML, SQLAlchemy/SQL | Configurazione, registri, cronologia |
| HAOS | Buildroot, Linux, systemd, Docker, AppArmor, RAUC | Ciclo di vita dell'appliance e isolamento |
| Servizi | App del Supervisor o container/host esterni | MQTT, Matter, database, proxy, condivisioni file |
| Edge | Controller radio, bridge di protocollo, API di dispositivi e cloud | Raggiungibilità fisica e origine dei dati |

La [Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/) valuta le integrazioni in base a flusso di configurazione, test, tipizzazione, diagnostica, uso efficiente dei dati e comportamento asincrono. Un livello elevato migliora la manutenibilità attesa, ma non costituisce uno SLA di disponibilità per il dispositivo o il fornitore cloud sottostante.

Dopo modello di installazione e runtime segue il modello dati. Solo distinguendo Config Entry, dispositivo, entità e stato è possibile spiegare chiaramente entità duplicate, dispositivi mancanti e automazioni errate.

## Modello a oggetti: Config Entry, dispositivo, entità e stato

La chiave dell'inventario operativo non è il riquadro visibile, bensì la catena composta da configurazione, identità del dispositivo ed entità.

### Config Entries

Un **Config Entry** memorizza la configurazione persistente di un'istanza di integrazione. Un flusso di configurazione dell'interfaccia lo crea; opzioni, riconfigurazione, reload, unload, rimozione e migrazione sono operazioni del ciclo di vita definite. Le integrazioni non devono modificare direttamente i dati dell'entry, ma devono usare il Config Entry Manager ([Config entries](https://developers.home-assistant.io/docs/config_entries_index/)). Un errore di autenticazione, un entry non caricato e una controparte non raggiungibile sono quindi stati diversi.

### Dispositivi e registri

Il **Device Registry** raggruppa endpoint tecnici in dispositivi. Identificatori o connessioni, per esempio numero di serie e indirizzo MAC, servono per il matching; `via_device` può rappresentare una relazione di bridge o genitore ([Device registry](https://developers.home-assistant.io/docs/device_registry_index/)). Un sensore Zigbee può quindi apparire come dispositivo connesso tramite un coordinator, senza che il coordinator costituisca il suo stato applicativo.

L'**Entity Registry** attribuisce alle entità un'identità permanente con `unique_id` e impedisce collisioni tra Entity ID. Indirizzo IP, hostname, URL, nome utente o indirizzo e-mail non sono espressamente considerati Unique ID stabili ([Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)). Ciò spiega perché rinominare manualmente un host non può sostituire l'identità del dispositivo e perché le migrazioni delle integrazioni richiedono identificatori stabili del produttore.

### Entità e stato

Un'**entità** rappresenta una funzione o una grandezza di misura: `sensor`, `switch`, `light`, `climate`, `binary_sensor` oppure un altro dominio. Il suo stato consiste in uno State primario, attributi, orari di modifica e Context. La State Machine conserva soltanto l'ultimo stato noto. `unavailable` significa che l'entità non è attualmente alimentata da un oggetto Entity attivo; `unknown` significa che non è disponibile alcun valore utilizzabile. L'«ultimo valore» non equivale quindi automaticamente a una «misurazione recente».

L'[interazione documentata tra dispositivi e servizi](https://developers.home-assistant.io/docs/architecture/devices-and-services/) separa Entity Integration, Entity Component, Entity Platform e integrazione specifica del produttore. Per la diagnosi, si pone quindi sempre la domanda:

- Quale Config Entry possiede l'entità?
- Tramite quale integrazione e piattaforma viene creata?
- Quale ID stabile di dispositivo e di entità collega cronologia e configurazione?
- Viene eseguito polling o push?
- Quale modello di tempo e disponibilità possiede il valore sorgente?
- Quale bridge, libreria, API cloud o collegamento radio si trova a monte?

## Integrazioni e isolamento dei guasti

Un'integrazione definisce un dominio e può fornire piattaforme quali `sensor`, `light` o `switch`. La piattaforma astrae il tipo di entità; l'integrazione del dispositivo comunica con il protocollo concreto. Le integrazioni integrate sono distribuite con il Core e testate dal relativo processo di release. Le **Custom Integrations**, tuttavia, vengono eseguite nello stesso processo Python e possono influire su import, Event Loop, tempo di avvio o consumo di memoria. La directory `/config/custom_components` è quindi parte di inventario, change management e ripristino.

La [panoramica delle integrazioni](https://www.home-assistant.io/integrations/) ufficiale distingue, tra l'altro, classi IoT quali Local Push, Local Polling, Cloud Push e Cloud Polling. Questa classificazione è più utile per i modelli operativi rispetto a un lungo elenco di produttori:

| Classe | Percorso dati | Tipica area di guasto |
|---|---|---|
| Local Push | Il dispositivo o bridge invia nella LAN | Multicast, firewall, bridge, subnet |
| Local Polling | Il Core interroga il dispositivo locale | Latenza, timeout, intervallo di interrogazione, capacità del dispositivo |
| Cloud Push | Il cloud invia o trasmette in streaming eventi | Internet, account, token, stream del fornitore |
| Cloud Polling | Il Core interroga l'API del fornitore | Rate limit, token, Internet, modifica API |
| Calculated/Internal | Il Core calcola lo stato | Dati di input, template, tempo, stato dopo riavvio |

L'integrazione non è un isolatore di processo. Una corretta delimitazione del guasto disabilita o ricarica selettivamente il Config Entry interessato prima di riavviare l'intero Core. Un riavvio distrugge evidenze volatili e può ripristinare timer di automazione dipendenti dal tempo.

## Modello di protocollo e rete

Per Home Assistant, un **grafo delle dipendenze** è più utile di una tabella OSI generalizzata. La piattaforma si trova a livello applicativo, ma i relativi percorsi dati si diramano:

- Frontend, REST e WebSocket usano HTTP su TCP, in genere sulla porta 8123.
- DNS risolve host e servizi cloud; mDNS e SSDP rilevano dispositivi nella rete locale.
- MQTT utilizza un broker separato e un modello publish/subscribe tramite TCP o WebSocket.
- Zigbee, Z-Wave, Thread e Bluetooth richiedono controller radio o proxy di rete.
- Matter utilizza comunicazione IP, ma per provisioning e funzionamento Fabric richiede un Matter Server e, se necessario, un Thread Border Router.
- Le integrazioni dei produttori possono usare HTTPS, protocolli locali proprietari o stream cloud.

Le integrazioni Discovery integrate documentano [mDNS/Zeroconf](https://www.home-assistant.io/integrations/zeroconf/) e [SSDP](https://www.home-assistant.io/integrations/ssdp/). Entrambi dipendono dal segmento e dal multicast. Un reverse proxy per il frontend non ripara la Discovery oltre i confini VLAN. Multicast relay, IGMP snooping, isolamento dei client WLAN, IPv6 RA, suffissi DNS e regole firewall devono essere verificati per ogni percorso effettivo del dispositivo.

### MQTT come spazio di stato separato

MQTT non è l'Event Bus interno. È un servizio broker esterno con il quale comunica un'integrazione. L'[integrazione MQTT](https://www.home-assistant.io/integrations/mqtt/) ufficiale descrive Discovery Topics, retained messages, Birth/Last Will, Availability, TLS e MQTT 5. La Discovery retained può ricreare dispositivi dopo un riavvio, ma può anche conservare Ghost Entities obsolete. La disponibilità richiede una semantica propria; la presenza di uno State retained non dimostra che il publisher sia ancora attivo.

Un funzionamento MQTT affidabile inventaria broker, Client ID, autenticazione, CA, topic, QoS, Retain, Expiry, Birth/Will e origine Discovery. Il backup del broker e quello del Core sono oggetti di protezione separati.

### Percorsi radio e bridge

[ZHA](https://www.home-assistant.io/integrations/zha/) integra un coordinator Zigbee, [Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/) usa un server Z-Wave JS separato e [Matter](https://www.home-assistant.io/integrations/matter/) collega un Matter Server. [Thread](https://www.home-assistant.io/integrations/thread/) gestisce riferimenti a Border Router e reti, ma non è identico a Matter. Dispositivi radio, firmware dei controller, dati di rete, materiale crittografico e configurazione dei dispositivi formano ciascuno un insieme di ripristino. Spostare una chiavetta USB o sostituire un coordinator non è una normale modifica dell'indirizzo IP.

Le integrazioni forniscono stati ed eventi; le automazioni reagiscono a essi. Il loro flusso, da trigger, condizioni e azioni, deve pertanto essere diagnosticato separatamente dalla configurazione dei dispositivi.

## Runtime delle automazioni: trigger, condition, action

Un'automazione è una definizione di esecuzione reattiva. I [fondamenti delle automazioni](https://www.home-assistant.io/docs/automation/basics/) separano trigger, Conditions opzionali e Actions. Il trigger crea un'esecuzione, le Conditions verificano lo stato di ingresso e le Actions usano la semantica sequenziale degli Scripts ([Actions](https://www.home-assistant.io/docs/automation/action/)).

È importante il momento: State, attributi e valori dei template possono cambiare tra il trigger e un'Action successiva. Un Delay non mantiene aperta una transazione. Esecuzioni multiple della stessa automazione richiedono quindi una modalità come Single, Restart, Queued o Parallel e un modello di conflitto consapevole. Gli attuatori fisici sono raramente transazionali; un'esecuzione parzialmente completata può richiedere azioni compensative.

La [documentazione sui trigger](https://www.home-assistant.io/docs/automation/trigger/) segnala che i tempi di attesa `for` non sopravvivono a un riavvio o a un reload dell'automazione. Chi deve preservare una scadenza oltre i riavvii persiste un orario, per esempio in `input_datetime`, e attiva rispetto a esso. Le Conditions sono solo verifiche nell'esecuzione corrente; la [semantica delle Conditions](https://www.home-assistant.io/docs/scripts/conditions/) non le trasforma in un blocco contro modifiche parallele.

In Home Assistant i template vengono valutati con espressioni Jinja. Errori di input e di tipo, `unknown`, `unavailable`, fusi orari e conversione implicita delle stringhe devono far parte dei test. La [documentazione sul templating](https://www.home-assistant.io/docs/automation/templating/) descrive variabili dipendenti dai trigger. Un amministratore non testa soltanto l'happy path, ma anche riavvio, entità mancante, evento ritardato, trigger duplicato ed errore dell'attuatore.

## Configurazione, registri e source of truth

Home Assistant combina Config Entries guidati dall'interfaccia, dati dei registri e YAML. `configuration.yaml` è la radice della configurazione manuale, ma non è la source of truth completa. La [panoramica della configurazione](https://www.home-assistant.io/docs/configuration/) ufficiale distingue UI e YAML; i Packages possono strutturare blocchi YAML correlati ([Packages](https://www.home-assistant.io/docs/configuration/packages/)).

Per Git e review è adatta soltanto la parte testuale priva di segreti. `secrets.yaml` separa i valori dallo YAML, ma non li cifra; la [guida all'hardening](https://www.home-assistant.io/docs/configuration/securing/) lo evidenzia espressamente. Stato UI, registri, token e Config Entries risiedono nell'archivio di configurazione e vengono modificati attraverso percorsi UI/API supportati. La modifica diretta dei file Storage interni mentre il Core è in esecuzione aggira la logica di schema, ciclo di vita e coerenza.

Un inventario di configurazione comprende:

- YAML, Packages, Blueprints e Custom Components,
- Config Entries con origine, owner e autenticazione,
- assegnazioni Device, Entity e Area,
- automazioni, Scripts, scene e dashboard,
- utenti, token, MFA e Identity Provider esterni,
- app del Supervisor o servizi esterni,
- controller radio, broker, database e proxy,
- Secrets, certificati e chiavi di ripristino.

## Recorder, cronologia e statistiche a lungo termine

La State Machine mantiene lo stato attuale in memoria. La cronologia nasce soltanto tramite il **Recorder**. Esso scrive modifiche di stato ed eventi selezionati in un database tramite SQLAlchemy; History, Activity, grafici e statistiche a lungo termine leggono da lì. La [documentazione Recorder](https://www.home-assistant.io/integrations/recorder/) ufficiale indica SQLite come impostazione predefinita e raccomandazione, nonché MariaDB, MySQL e PostgreSQL come alternative supportate.

I dati Recorder non sono una sorgente di eventi per il controllo in tempo reale. Un database non disponibile può compromettere cronologia e statistiche mentre gli stati correnti e le automazioni continuano parzialmente a funzionare. Viceversa, una cronologia completa non dimostra che un'Action sul dispositivo fisico sia riuscita.

I parametri operativi più importanti sono:

- `purge_keep_days` per la cronologia grezza,
- filtri Include/Exclude per entità ed eventi,
- `commit_interval` come rapporto tra I/O e finestra di perdita,
- dimensione del database, spazio libero e latenza di scrittura,
- Purge e Repack,
- sequenza di avvio e raggiungibilità dei database esterni,
- statistiche a lungo termine e coerenza dei metadati.

Un cambio del database Recorder non migra la cronologia esistente in modo supportato. I database esterni richiedono backup coerenti e test di ripristino propri. Per SQLite, la documentazione richiede spazio libero pari ad almeno 2,5 volte la dimensione del database per il trattamento della corruzione. Storage e Recorder sono quindi un percorso separato di capacità e ripristino, non soltanto una cache opzionale.

## API, WebSocket e autenticazione

Frontend e API condividono per impostazione predefinita lo stesso listener HTTP. La [REST API](https://developers.home-assistant.io/docs/api/rest/) usa JSON e Bearer Tokens; il percorso base è `/api/`. La [WebSocket API](https://developers.home-assistant.io/docs/api/websocket/) si trova in `/api/websocket`, passa attraverso `auth_required`, `auth` e `auth_ok` e correla i comandi tramite ID numerici. WebSocket fornisce flussi di eventi e registri in modo più efficiente rispetto al polling REST ripetuto.

I token a lunga durata sono credenziali utente. L'[Authentication API](https://developers.home-assistant.io/docs/auth_api/) descrive OAuth/IndieAuth, Refresh Tokens, Long-Lived Access Tokens e Signed Paths di breve durata. Un token eredita il contesto del proprio utente; un Long-Lived Token valido dieci anni appartiene a un archivio di Secrets, non a YAML, Shell History, URL o JavaScript della dashboard.

Un monitor API verifica almeno autenticazione, `/api/config`, entità attese, `last_updated`, sottoscrizione WebSocket e un percorso Read/Action non pericoloso. Un HTTP 200 su `/` verifica soltanto la raggiungibilità del frontend.

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

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) e [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) elaborano la query Windows; [`curl`](https://curl.se/docs/manpage.html) e [`jq`](https://jqlang.org/manual/) fanno lo stesso su Unix. Il token è mostrato solo come variabile d'ambiente del processo; in produzione proviene da un archivio di Secrets controllato.

## HTTP, TLS e reverse proxy

L'endpoint HTTP ascolta per impostazione predefinita su TCP 8123. TLS diretto, reverse proxy e Home Assistant Cloud sono modelli di accesso diversi. Con un reverse proxy tradizionale, `use_x_forwarded_for` e `trusted_proxies` devono essere configurati correttamente; altrimenti l'IP del client è errato o la richiesta viene rifiutata ([HTTP integration](https://www.home-assistant.io/integrations/http/)). Un elenco ampio di proxy attendibili consente la falsificazione delle informazioni Forwarded-For.

La [guida alla sicurezza](https://www.home-assistant.io/docs/configuration/securing/) raccomanda password univoche, MFA, privilegi amministrativi minimi e accesso remoto protetto anziché esposizione diretta a Internet. TLS protegge solo il trasporto. Diritti dei token, header del proxy, upgrade WebSocket, rate limit, DNS, rinnovo dei certificati e sicurezza dell'IdP a monte restano controlli separati.

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

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) e [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) verificano Windows; [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) verificano Unix. La chiamata non autenticata a `/api/` può restituire 401; sono determinanti risoluzione dei nomi, identità TLS, percorso proxy e limite di autenticazione atteso.

## Diagnostica MQTT

Lo stato del broker viene verificato al di fuori di Home Assistant. Un subscriber osserva Discovery, Availability e State senza modificare i topic. Un test di Publish usa un percorso di test riservato appositamente; i Command Topics produttivi non vengono descritti incidentalmente.

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

[`mosquitto_sub`](https://mosquitto.org/man/mosquitto_sub-1.html) e [`mosquitto_pub`](https://mosquitto.org/man/mosquitto_pub-1.html) sono i client ufficiali del broker. Le password sulla riga di comando possono essere visibili negli elenchi dei processi o nella cronologia; gli esempi illustrano il percorso, mentre in produzione la chiamata usa un file password, l'archivio di Secrets del sistema operativo o credenziali di breve durata.

## Funzionamento di HAOS e Container

HAOS mette a disposizione il comando `ha` tramite accesso Terminal/SSH. Le installazioni Container vengono gestite con gli strumenti del runtime scelto. Un pacchetto diagnostico mantiene insieme informazioni di sistema, log Core, diagnostica dell'integrazione, stato del container, spazio libero e orario.

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

[`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/), [`docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/) e [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) forniscono lo stato del container. [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) e [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) elaborano gli output Windows; [`grep`](https://www.gnu.org/software/grep/manual/grep.html), [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) e [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html) integrano Unix. Un container in esecuzione è soltanto il primo controllo; seguono percorsi di integrazione, registry, eventi e dispositivi.

Per la ricerca guasti, il percorso del segnale viene letto a ritroso: azione, Automation Trace, modifica di stato, integrazione, protocollo di rete e dispositivo fisico.

## Observability e diagnostica sistematica

**System Health** raccoglie tipo di installazione, architettura, informazioni su Python, Core e frontend e offre funzioni diagnostiche tramite Impostazioni > Sistema > Riparazioni ([System Health](https://www.home-assistant.io/integrations/system_health/)). L'[integrazione Logger](https://www.home-assistant.io/integrations/logger/) controlla i livelli di log globali e specifici dei componenti. Il debug logging è limitato temporalmente e ai namespace interessati; tempeste radio o di eventi possono altrimenti dominare memoria e I/O.

Una catena diagnostica affidabile è la seguente:

1. **Sintomo e stato atteso:** quale entità, Action, automazione o interfaccia è interessata?
2. **Tempo e scope:** da quando, per quali dispositivi, utenti, reti e istanze di integrazione?
3. **Identità dell'oggetto:** assicurare Config Entry, Device ID, Entity ID, Unique ID e riferimento al bridge.
4. **Runtime:** verificare Core, Event Loop, memoria, CPU, file system e database.
5. **Integrazione:** verificare stato dell'Entry, autenticazione, stato Coordinator/Polling e download diagnostico.
6. **Trasporto:** verificare Discovery, DNS, TCP, TLS, broker, controller radio o API del produttore.
7. **Automazione:** verificare Trace, dati del trigger, Conditions, Run Mode e risultato dell'Action.
8. **Persistenza:** valutare Recorder Lag e cronologia separatamente dallo stato live.
9. **Test controllato:** usare un'entità di test read-only o non pericolosa.
10. **Ripristino:** Reload prima di Restart, Restart prima di Restore; salvare prima le evidenze.

Un'entità `unavailable` può derivare da un Config Entry scaricato, un bridge mancante, una perdita radio o un timeout della sorgente. Un vecchio valore visibile è più pericoloso perché appare plausibile. Il monitoraggio richiede quindi limiti di freschezza, non solo limiti di valore.

## Aggiornamenti, release e Custom Integrations

Home Assistant pubblica frequenti release del Core e documenta modifiche incompatibili con le versioni precedenti. Un articolo di riferimento statico non fissa deliberatamente una versione momentanea. Al momento della manutenzione, il rollout verifica invece Release Notes, modifiche delle integrazioni e dipendenze target.

Un percorso di aggiornamento controllato comprende:

1. Confermare backup e download indipendente oppure posizione di archiviazione esterna.
2. Verificare spazio libero, stato del database e System Health.
3. Valutare Release Notes, integrazioni interessate e Custom Components.
4. Inventariare dipendenze radio, broker, database e proxy.
5. Aggiornare Core oppure HAOS e app nell'ordine definito.
6. Verificare log di avvio, riparazioni e migrazioni dei registry.
7. Testare i percorsi critici di sensori, attuatori, automazioni, API e accesso remoto.
8. Definire la soglia di errore e solo successivamente avviare rollback o restore.

HAOS usa RAUC con due slot di sistema operativo; `ha os info` e `rauc status` rendono visibile lo stato degli slot ([HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)). Questo meccanismo protegge il percorso di aggiornamento dell'OS, non automaticamente la configurazione Core, i dati Recorder o lo stato della rete radio.

Una volta noti runtime e percorso dati, è possibile definire l'ambito del backup. Configurazione, registri, Secrets, database e stati degli add-on devono essere coerenti con il modello di installazione scelto.

## Backup e ripristino

Home Assistant può scrivere backup automatici e manuali, crittografati, in destinazioni locali o esterne. La [guida a backup e restore](https://www.home-assistant.io/common-tasks/general/) ufficiale descrive sedi di backup, Emergency Kit, download, restore durante l'onboarding e migrazione verso altro hardware. Dal 2026, il modello crittografico dei backup è stato modernizzato; l'[annuncio sulla crittografia dei backup](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/) documenta il cambio di formato e i limiti di compatibilità.

Un backup è completo solo in relazione al modello di installazione:

| Oggetto | Backup HAOS | Responsabilità Container/esterna |
|---|---|---|
| Configurazione Core e registri | includibili | proteggere il volume Config |
| App del Supervisor | dati delle app includibili | container e volumi separati |
| Recorder SQLite | nell'area Config | backup DB coerente per DB esterno |
| Broker MQTT | solo con selezione app adeguata | configurazione e persistenza broker separate |
| Zigbee/Z-Wave/Matter | dati di integrazione parziali | verificare separatamente backup controller/server e chiavi |
| TLS/proxy/DNS | solo se nei dati selezionati | infrastruttura esterna separata |
| Chiavi di backup | non sufficientemente nel backup crittografato stesso | conservare Emergency Kit separatamente |

Un test di restore non termina al login. I criteri di accettazione sono: Config Entries caricati, registri coerenti, accesso utente possibile, database senza errori, broker e bridge connessi, dispositivi radio controllabili, automazioni critiche testate e accesso remoto disponibile con certificato corretto. I dispositivi a batteria possono inizialmente dormire dopo una migrazione; un valore immediato mancante non va interpretato prematuramente come perdita di dati.

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

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv) e [`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content) creano o leggono il manifest Windows. [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html), [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) e [`tar`](https://www.gnu.org/software/tar/manual/html_node/index.html) si occupano di Unix. Un checksum prova l'immutabilità dell'archivio; decifrabilità e ripristino funzionale sono provati solo dal test di restore.

## RPO, RTO e alta disponibilità

Nell'operatività usuale, Home Assistant è una singola istanza con stato. Due istanze Core attive contro gli stessi dispositivi, registri o comandi broker non generano alta disponibilità coordinata automaticamente. Automazioni duplicate possono commutare più volte gli attuatori; controller radio e dispositivi locali consentono spesso una sola proprietà attiva.

Un modello di resilienza realistico combina:

- nodo singolo affidabile o VM con risorse monitorate,
- UPS e storage adeguato anziché supporti flash sensibili,
- backup separati, automatici e crittografati,
- hardware sostitutivo documentato o piattaforma VM di destinazione,
- stati e chiavi dei controller radio esportabili,
- servizi esterni riproducibili,
- restore controllato con acquisizione univoca di dispositivi e rete.

L'**RPO** dipende dall'ultima copia protetta di configurazione, registry, app e servizi esterni. La cronologia Recorder può avere un RPO diverso dalla configurazione delle automazioni. L'**RTO** comprende non solo l'avvio del Core, ma anche DNS, proxy, database, broker, controller radio, riconnessione dei dispositivi, sensori dormienti e test di accettazione.

## Sicurezza e confini di fiducia

Home Assistant può controllare porte, riscaldamento, sistemi di allarme e flussi energetici. Il suo ambito d'influenza è quindi fisico. Il design della sicurezza separa:

- utenti e amministratori,
- sessioni browser, Companion App e API,
- Long-Lived Tokens e webhook,
- Core e Custom Integrations,
- app del Supervisor o container esterni,
- segmenti IoT, di gestione e degli utenti,
- dispositivi locali e cloud dei produttori,
- reti radio e rispettive chiavi,
- destinazioni di backup ed Emergency Kit.

MFA protegge gli account interattivi, ma non un Long-Lived Token rubato. La segmentazione di rete limita il movimento laterale, ma non deve bloccare senza controllo i necessari canali Discovery e di ritorno. Le Custom Integrations ottengono vicinanza di processo al Core e vengono trattate come deployment di codice. I Secrets non compaiono né in Git né in file diagnostici o post di supporto. I controlli generali sono disponibili in [hardening](/kb/haertung), i fondamenti del trasporto in [TLS](/kb/tls) e i modelli contrattuali API in [API](/kb/apis).

## Storia tecnica

Home Assistant è iniziato nel 2013 come progetto Python di Paulus Schoutsen. La retrospettiva per il decimo anniversario descrive lo sviluppo da piccola applicazione locale di automazione a grande progetto open source ([10 anni di Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)). Il Core Python e il modello delle integrazioni sono rimasti il centro funzionale, mentre attorno a essi si sono sviluppati frontend, client mobili, Supervisor, HAOS, hardware per dispositivi e opzioni cloud.

Con Hass.io, poi Home Assistant ovvero Home Assistant OS e Supervisor, è nato uno stack appliance composto da sistema operativo, gestione dei container, Core e servizi aggiuntivi. La distinzione è stata più volte chiarita a livello linguistico: gli «Add-ons» oggi si chiamano **Apps**, mentre le «integrazioni» restano estensioni Python del Core. Questi termini indicano confini di esecuzione e sicurezza differenti.

Nel 2024 Home Assistant è passato alla fondazione non-profit Open Home Foundation; Nabu Casa è rimasta partner commerciale. L'annuncio del progetto sull'[ecosistema Open Home](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/) descrive proprietà e governance. Nel 2025 il progetto ha ridotto le varianti di installazione supportate a HAOS e Container. La tendenza storica non va quindi verso un cluster distribuito, ma verso un Core centrale più stabile con pacchetti di runtime chiaramente supportati e server di protocollo autonomi.

## Checklist per amministratori

Una dashboard verde non basta come prova operativa. La checklist unisce installazione, percorsi dei dispositivi, automazioni, conservazione dei dati e ripristino in una visione complessiva verificabile.

- **Installazione:** documentare HAOS o Container, architettura, host, storage, rete e ownership.
- **Stack:** separare Core, Supervisor, app/container esterni, database, broker, proxy e server radio.
- **Inventario:** rilevare Config Entry, Device ID, Entity ID, Unique ID, Area e `via_device`.
- **Origine dei dati:** contrassegnare Local/Cloud e Push/Polling per ogni integrazione critica.
- **Stato:** distinguere `unknown`, `unavailable`, valore obsoleto e successo confermato sul dispositivo.
- **Automazione:** testare trigger, Context, Condition, Run Mode, comportamento al riavvio e compensazione.
- **API:** controllare contesto utente, archiviazione token, WebSocket, reverse proxy e TLS.
- **Recorder:** monitorare database, filtri, intervallo di commit, Purge, I/O, crescita e backup.
- **Rete IoT:** verificare esplicitamente mDNS, SSDP, MQTT, VLAN, IPv6 e percorsi radio/bridge.
- **Aggiornamenti:** riunire Release Notes, Custom Integrations, backup, rollout e accettazione.
- **Ripristino:** testare insieme Core, servizi esterni, stato radio, chiavi ed Emergency Kit.
- **Verifica:** non verificare soltanto UI e container, ma almeno un percorso completo di sensore, attuatore, automazione e API.

## Fonti

- [Home Assistant Developer Docs – panoramica dell'architettura](https://developers.home-assistant.io/docs/architecture_index/)
- [Home Assistant Developer Docs – architettura delle integrazioni](https://developers.home-assistant.io/docs/architecture_components/)
- [Home Assistant Developer Docs – Core architecture](https://developers.home-assistant.io/docs/architecture/core/)
- [Home Assistant – HAOS o Container](https://www.home-assistant.io/faq/ha-vs-hassio/)
- [Home Assistant Developer Docs – Operating System](https://developers.home-assistant.io/docs/operating-system/)
- [Home Assistant Developer Docs – Supervisor](https://developers.home-assistant.io/docs/supervisor/)
- [Home Assistant – disattivazione di Core, Supervised e 32 bit](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
- [GitHub – Home Assistant Core](https://github.com/home-assistant/core)
- [Home Assistant Developer Docs – Async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)
- [Home Assistant Developer Docs – Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/)
- [Home Assistant Developer Docs – Config entries](https://developers.home-assistant.io/docs/config_entries_index/)
- [Home Assistant Developer Docs – Device registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Home Assistant Developer Docs – Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)
- [Home Assistant Developer Docs – dispositivi e servizi](https://developers.home-assistant.io/docs/architecture/devices-and-services/)
- [Home Assistant – integrazioni](https://www.home-assistant.io/integrations/)
- [Home Assistant – Zeroconf](https://www.home-assistant.io/integrations/zeroconf/)
- [Home Assistant – SSDP](https://www.home-assistant.io/integrations/ssdp/)
- [Home Assistant – MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Home Assistant – ZHA](https://www.home-assistant.io/integrations/zha/)
- [Home Assistant – Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/)
- [Home Assistant – Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant – Thread](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant – fondamenti delle automazioni](https://www.home-assistant.io/docs/automation/basics/)
- [Home Assistant – Automation actions](https://www.home-assistant.io/docs/automation/action/)
- [Home Assistant – Automation triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Home Assistant – Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Home Assistant – Automation templating](https://www.home-assistant.io/docs/automation/templating/)
- [Home Assistant – configurazione](https://www.home-assistant.io/docs/configuration/)
- [Home Assistant – Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [Home Assistant – proteggere Home Assistant](https://www.home-assistant.io/docs/configuration/securing/)
- [Home Assistant – Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant Developer Docs – REST API](https://developers.home-assistant.io/docs/api/rest/)
- [Home Assistant Developer Docs – WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)
- [Home Assistant Developer Docs – Authentication API](https://developers.home-assistant.io/docs/auth_api/)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – manuale](https://curl.se/docs/manpage.html)
- [jq – manuale](https://jqlang.org/manual/)
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
- [GNU Grep – manuale](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [GNU Coreutils – du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Home Assistant – System Health](https://www.home-assistant.io/integrations/system_health/)
- [Home Assistant – Logger](https://www.home-assistant.io/integrations/logger/)
- [Home Assistant Developer Docs – HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)
- [Home Assistant – backup e restore](https://www.home-assistant.io/common-tasks/general/)
- [Home Assistant – crittografia dei backup modernizzata](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [GNU Coreutils – sha2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [GNU Tar – manuale](https://www.gnu.org/software/tar/manual/html_node/index.html)
- [Home Assistant – 10 anni di Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)
- [Home Assistant – Open Home Foundation e governance](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/)
