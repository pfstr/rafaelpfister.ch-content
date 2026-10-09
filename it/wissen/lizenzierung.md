---
title: "Licenze: diritti d’uso, metriche e punti di conteggio tecnici"
blatt: "lizenzierung"
description: "Licenze tecniche per amministratori di infrastrutture e messaggistica: entitlement, metriche per utenti, dispositivi, istanze, core e capacità, edizioni, abbonamenti, attivazione, server di licenza e API cloud, fonti di identità, HA/DR, limiti tecnici di enforcement, dati di audit, Open Source e verifica."
fakten:
  - label: Determinante
    wert: Contratto, Product Terms e ordine; l’indicazione tecnica è una prova, non il contratto legale
    href: https://www.iso.org/standard/52293.html
  - label: Tre quantità
    wert: diritti acquisiti · assegnati tecnicamente · effettivamente utilizzati
    href: https://www.iso.org/standard/52293.html
  - label: Metriche
    wert: utente · dispositivo · istanza · core/vCPU · capacità · transazione · funzionalità
    href: https://www.iso.org/standard/52293.html
  - label: Punto di conteggio
    wert: scope + ancoraggio dell’oggetto + filtro di stato + finestra temporale + regola di aggregazione
    href: https://www.iso.org/standard/68531.html
  - label: ID software
    wert: I tag SWID standardizzano l’identificazione del prodotto, non automaticamente l’autorizzazione
    href: https://www.iso.org/standard/65666.html
  - label: Licenze basate sull’identità
    wert: valutare separatamente assegnazione diretta e basata su gruppi, SKU e piano di servizio
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA e DR
    wert: valutare nodi passivi, Cold Standby, istanze di test e di ripristino solo in base ai Terms specifici
    href: https://www.iso.org/standard/52293.html
  - label: Enforcement
    wert: avviso, limite di funzionalità, nessuna nuova assegnazione, Grace Period o interruzione del servizio dipendono dal prodotto
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Funzionamento offline
    wert: documentare durata di token/lease, Grace Period, orologio, Trust Store e percorso di ripristino
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Open Source
    wert: utilizzabile gratuitamente non significa privo di obblighi; verificare il testo della licenza e lo scenario di distribuzione
    href: https://opensource.org/osd
  - label: Leggibile dalla macchina
    wert: le espressioni SPDX modellano licenze singole, alternative e combinate
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Evidenza amministrativa
    wert: esportare in modo riproducibile entitlement · inventario · assegnazione · utilizzo · eccezione · tempo · fonte
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: 4abc9d3d627b9f38adb1f2fcb541662b430f07694209251464f0a4b0f9c4351f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:23:14.881Z
translationReview: required
---

# Licenze: diritti d’uso, metriche e punti di conteggio tecnici

La licenza software collega un contratto legale di utilizzo a oggetti misurabili tecnicamente. Per gli amministratori è essenziale non confondere questi livelli. Una chiave di licenza, un indicatore cloud o un contatore interno di utenti può attivare funzioni e misurare l’utilizzo; tuttavia non definisce da solo ciò che un’organizzazione può utilizzare legalmente. ISO/IEC 19770-3 tratta esplicitamente i dati di entitlement digitali come rappresentazione dei diritti d’uso e chiarisce che le condizioni di licenza originali prevalgono a fini legali ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Il compito operativo consiste quindi nel **confrontare in modo riproducibile i diritti acquisiti, i diritti installati o assegnati tecnicamente e l’utilizzo effettivo**. Le discrepanze possono indicare sovrautilizzo, costi inutilizzati, filtri di directory errati, account orfani, nodi passivi di cluster, token scaduti o semplicemente definizioni diverse della stessa parola «utente». L’articolo descrive la prospettiva tecnica; l’interpretazione contrattuale e legale resta di competenza di acquisti, gestione delle licenze e consulenza legale.

La spiegazione parte dal diritto d’uso contrattuale e segue il modo in cui prodotto, metrica e punto di conteggio tecnico ne ricavano un consumo misurato. Seguono assegnazione, enforcement, audit, casi speciali e ripristino.

Una licenza è anzitutto un diritto d’uso; diventa tecnicamente visibile solo tramite assegnazione, utilizzo misurato ed eventualmente enforcement. L’articolo separa questi quattro livelli prima di trattare metriche di prodotto e audit.

## Entitlement, assegnazione, utilizzo ed enforcement

Un bilancio di licenze corretto considera quattro stati indipendenti:

1. **Entitlement:** quali diritti d’uso sono stati acquisiti con quale contratto, prodotto, metrica, scope, periodo e diritto speciale?
2. **Deployment/assegnazione:** su quali dispositivi, istanze, utenti, tenant o funzionalità il software è installato, attivato o assegnato?
3. **Utilizzo:** quali oggetti o funzioni rilevanti ai fini della licenza sono stati effettivamente utilizzati nel periodo di misurazione concordato?
4. **Enforcement:** quale limite viene verificato tecnicamente dal prodotto e come reagisce in caso di assenza di connessione, scadenza o superamento?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Aprire direttamente il grafico interattivo</a>.
</iframe>

Un entitlement esistente non dimostra un’assegnazione corretta; un’assegnazione non dimostra l’utilizzo; un utilizzo ridotto non revoca automaticamente una licenza utente nominativa. Viceversa, un prodotto può continuare a funzionare tecnicamente anche se è scaduto un diritto di abbonamento o di supporto. Conformità e disponibilità tecnica sono quindi due obiettivi di controllo distinti.

ISO/IEC 19770-1 specifica i requisiti di un sistema di gestione degli asset IT. ISO/IEC 19770-2 standardizza i Software Identification Tags (SWID), che rendono identificabile il software, ma secondo la norma non prescrivono alcuna riconciliazione degli entitlement. ISO/IEC 19770-3 definisce termini e un formato di trasporto per entitlement e metriche associate. Nel loro insieme forniscono i livelli dati **inventario**, **identità software** e **diritto d’uso**, non un calcolo universale delle licenze ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Metriche di licenza: che cosa viene conteggiato?

Una metrica è interpretabile solo insieme al contratto completo:

| Metrica | Possibile ancoraggio di conteggio | Domande per gli amministratori |
|---|---|---|
| Named User | ID immutabile di persona/tenant | vengono conteggiati account disattivati, condivisi, esterni, di servizio o di test; è consentita la riassegnazione? |
| Concurrent User/Session | sessione attiva o lease di checkout | quale sessione inizia/termina, come vengono trattati timeout, dispositivi multipli e lease offline? |
| Device | ID hardware, dispositivo registrato, istanza client | VDI, dispositivo sostitutivo, dispositivo condiviso e agenti reinstallati vengono conteggiati separatamente? |
| Server/Instance/Node | VM, host, appliance, nodo di cluster, istanza container | vengono conteggiate istanze passive, temporanee, autoscalate, di test o di ripristino? |
| Processor/Core/vCPU | socket/core fisico, vCPU assegnata, numero minimo | come vengono calcolati Hyperthreading, affinità, spostamento del cluster, dimensioni cloud e pacchetti minimi? |
| Capacity | caselle di posta, storage, domini, messaggi, throughput o record di dati | vale il picco, la media, il massimo mensile, il provisioned o l’used; quale finestra temporale? |
| Feature/Edition | piano di servizio attivato, modulo, limite del database, API | sono sufficienti installazione, attivazione, configurazione o utilizzo effettivo? |
| Subscription/Consumption | assegnazione SKU, crediti, richieste, GB-mesi | quando viene riservato, consumato, ricalcolato o restituito? |

L’unità tecnica non deve essere dedotta dal nome del prodotto. «Per core» può significare core fisici dell’host, vCPU della VM o una metrica core normalizzata. «Utente» può indicare una persona fisica, un account attivo, una casella di posta, un’identità con licenza o un mittente. «Istanza» può essere conteggiata in esecuzione, installata, registrata o per nodo di cluster. ISO/IEC 19770-3 standardizza una struttura di entitlement, ma non sostituisce la definizione concreta nei Product Terms e nell’ordine ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La sola metrica non indica ancora dove viene effettuato il conteggio. Solo il punto di conteggio tecnico spiega perché portale del produttore, inventario locale e valore di fatturazione possono differire.

## Il punto di conteggio tecnico

Ogni metrica viene documentata come funzione di conteggio riproducibile:

**Contatore = scope × ancoraggio dell’oggetto × filtro di stato × finestra temporale × regola di aggregazione × eccezioni**

- **Scope:** organizzazione, tenant, dominio, cluster, subscription, sede o contratto.
- **Ancoraggio dell’oggetto:** ID utente immutabile, ID dispositivo, ID VM, seriale host, ID SKU o hash – non solo nome visualizzato.
- **Filtro di stato:** enabled, assigned, provisioned, active, seen, mounted, running o consumed.
- **Finestra temporale:** data di riferimento, mese di calendario, picco, media, finestra mobile o anno contrattuale.
- **Aggregazione:** distinct, somma, massimo, 95° percentile, pacchetto minimo o scaglione.
- **Eccezioni:** utenti esterni, account di sistema, istanza DR passiva, trial, NFR o caso d’uso contrattualmente gratuito.

Il punto di conteggio è spesso un **punto di consegna**. Un’importazione LDAP può conteggiare tutti gli account corrispondenti, anche se non utilizzano mai una funzione di posta. Un gateway può apprendere i mittenti dal traffico e quindi mantenere identità eliminate o tecniche. Una SKU cloud può essere assegnata direttamente o indirettamente tramite un gruppo. L’API `licenseDetails` di Microsoft Graph fornisce licenze dirette e ereditate dall’appartenenza a gruppi, nonché singoli piani di servizio e stati di provisioning; ciò mostra perché un booleano «ha licenza» non è sufficiente per l’analisi ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Architettura di un sistema di licenze

I prodotti commerciali utilizzano implementazioni diverse, ma tecnicamente possono essere scomposti in superfici di controllo. Questo modello è una sintesi operativa dei livelli dati ISO/IEC 19770 e dei meccanismi documentati dai produttori, non un’architettura normativa universale:

| Superficie di controllo | Funzione | Area di guasto |
|---|---|---|
| Entitlement Store | contratto/SKU, quantità, periodo, funzionalità, diritti speciali | ordine errato, scaduto, tenant/Smart Account errato |
| Inventory/Identity Source | utenti, dispositivi, istanze, core, cluster, ID software | duplicato, oggetto obsoleto, filtro o scope errato |
| Assignment | assegna un diritto a un oggetto o piano di servizio | assegnazione diretta vs. basata su gruppi, errore di provisioning |
| Meter | raccoglie utilizzo, sessioni, capacità o heartbeat | tempo, buffer offline, campionamento, telemetria mancante |
| Evaluator | applica metrica, pool, eccezioni e periodo | la logica contrattuale non corrisponde alla policy tecnica |
| License Service | server locale, API cloud, token, lease, certificato o chiave | DNS/TLS/proxy/orologio/trust, guasto o rate limit |
| Enforcement | attiva edizione/funzionalità o limita il comportamento | blocco rigido, Grace Period, avviso o fail-open/fail-closed |
| Evidence Export | dati di audit, utilizzo e assegnazione | rotazione, protezione dei dati, cronologia mancante o esportazione non riproducibile |

Cisco Smart Licensing descrive una gestione centralizzata di account e licenze; Microsoft Graph espone SKU acquisite, assegnazioni e piani di servizio tramite API. Tali sistemi rendono le licenze una dipendenza distribuita da identità, servizio cloud, rete, TLS e tempo. Un prodotto può continuare a elaborare dati ma non riuscire a ottenere una nuova licenza; un altro può bloccare funzionalità dopo un periodo offline. Il comportamento concreto deve essere ricavato dalla documentazione del produttore e dal contratto ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licenze basate sull’identità

Per gli utenti nominativi, la directory fa parte del sistema di licenze. Un filtro corretto non risponde solo a «persone nell’OU X», ma anche a:

- quale attributo costituisce l’ancoraggio immutabile dell’oggetto;
- se vengono conteggiati account disattivati, bloccati, eliminati o non ancora sottoposti a provisioning;
- come vengono trattate caselle Shared/Resource, account di servizio, ospiti, partner esterni e account di test;
- se le assegnazioni dirette e basate su gruppi vengono consolidate;
- quando una licenza rimossa torna disponibile;
- se conta l’utilizzo storico o solo lo stato alla data di riferimento;
- quali attributi il prodotto memorizza localmente nella cache e quando li elimina.

LDAP fornisce voci e attributi, ma non una definizione universale di «persona con licenza». [LDAP](/kb/ldap) può implementare tecnicamente scope e filtro; la metrica deriva dal contratto. Lo stesso vale nelle directory cloud: SKU, piano di servizio, `assignedLicenses`, stato di provisioning e stato effettivo del workload sono dati distinti. Microsoft Graph documenta `subscribedSku` come subscription commerciale acquisita e `licenseDetails` per utente ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

La pulizia degli account orfani non deve essere guidata direttamente dalla pressione sulle licenze. Prima si verificano proprietari, conservazione, routing della posta, Legal Hold, dipendenze di servizio e ripristino; poi l’identità viene disattivata o eliminata in modo controllato. Un report di licenze è un input del ciclo di vita, non un ordine di eliminazione.

## Modelli per istanze, core e capacità

Virtualizzazione e cluster rendono insufficiente il nome del server fisico. Un inventario richiede host, VM/container, vCPU assegnate, CPU/core fisici, cluster hypervisor, regole di mobilità, edizione e ruolo. Se una VM può spostarsi tra host, a seconda del contratto può essere rilevante l’intero scope possibile degli host; una rigida affinità CPU può essere dimostrabile tecnicamente, ma non è automaticamente riconosciuta dal contratto.

Le edizioni collegano il diritto d’uso ai limiti tecnici. Microsoft documenta per Exchange Server, ad esempio, edizioni che differiscono tra l’altro nel numero di database montati contemporaneamente; anche le copie passive del database possono essere conteggiate come database montati. Il Product Key imposta l’edizione del server. Questo è un esempio di come enforcement e architettura di capacità coincidano, non un calcolo generale delle licenze Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Le metriche di capacità richiedono una serie temporale. Un valore istantaneo non mostra né il picco mensile né un superamento di breve durata. Per la messaggistica, contatori tecnici diffusi sono in particolare caselle di posta attive, mittenti interni, domini, numero giornaliero di messaggi, throughput, storage e utenti cifrati; se siano rilevanti ai fini della licenza lo stabilisce esclusivamente il diritto di prodotto. I dashboard memorizzano pertanto valore grezzo, ora, scope, fonte e regola di calcolo anziché solo un semaforo.

Non appena vengono conteggiate istanze o identità, alta disponibilità, test e migrazioni incidono anche sulla quantità di licenze. I sistemi passivi non sono automaticamente gratuiti; fa fede il contratto specifico.

## HA, Disaster Recovery, test e migrazione

Nodi passivi, Cold Standby, istanze di ripristino, laboratorio, test, formazione e funzionamento parallelo temporaneo durante una [migrazione](/kb/migration) sono casi speciali tipici. Dal punto di vista tecnico «passivo» può comunque significare che il software è installato, viene elaborata la replica, sono montati database o viene effettuato il checkout di una licenza sul server. I termini contrattuali devono essere mappati su stati osservabili.

Per ogni caso speciale vengono documentati:

- numero consentito e definizione delle istanze passive/fredde;
- se sono inclusi test, qualificazione delle patch, ripristino di backup o test DR;
- per quanto tempo è consentito il funzionamento simultaneo vecchio/nuovo durante la migrazione;
- se una licenza è mobile e quali condizioni di riassegnazione/attesa si applicano;
- se i diritti cloud e on-premises sono collegati;
- quale evidenza distingue un vero caso DR da un carico produttivo continuativo;
- come funziona il servizio di licenza in una rete di ripristino isolata.

Il [concetto di backup e DR](/kb/backup-dr) include quindi anche file di licenza, dati di attivazione, server di licenza, token offline, certificati, ora, DNS/proxy e contatti del produttore. Un ripristino tecnicamente perfetto che non riesce ad attivare la propria edizione o le funzionalità necessarie non soddisfa l’obiettivo di recovery.

## Scadenza, Grace Period e guasto del servizio di licenza

Nei modelli subscription, lease o cloud sono rilevanti almeno i seguenti timer: fine dell’entitlement, scadenza del token, heartbeat, durata di borrow/checkout, periodo offline, Grace Period e validità del certificato. La prospettiva amministrativa mantiene **ora assoluta, fuso orario, fonte di sincronizzazione e ultimo rinnovo riuscito**. Un orologio errato può causare una presunta scadenza della licenza o impedire la validazione di un token firmato.

Il comportamento alla scadenza dipende dal prodotto:

- solo avviso o evento di conformità;
- nessuna nuova assegnazione, l’utilizzo esistente resta;
- funzionalità premium disattivata o ritorno a un’edizione inferiore;
- numero limitato di nuove sessioni/utenti;
- funzionamento in sola lettura;
- interruzione completa del servizio;
- Grace Period locale se il servizio cloud non è raggiungibile.

Questa reazione viene determinata in un ambiente di test o tramite una fonte esplicita del produttore, non sperimentata nel flusso di produzione. Il monitoraggio avvisa prima del primo timer operativo, non solo della fine del contratto. Cisco documenta per Smart Licensing propri meccanismi online/offline e di account; altri produttori utilizzano server di licenza locali, file firmati, dongle USB, Product Key o entitlement SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Open Source: diritto d’uso senza contatore tecnico

Open Source non significa semplicemente codice sorgente visibile. La Open Source Definition richiede, tra l’altro, libera ridistribuzione, accesso al codice sorgente, opere derivate e diritti tecnologicamente neutrali. Le singole licenze prevedono condizioni differenti per modifica, ridistribuzione, notices, messa a disposizione del codice sorgente o licenze di brevetto ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

La Apache License 2.0 concede licenze di copyright e brevetto a determinate condizioni e, in caso di ridistribuzione, richiede tra l’altro una copia della licenza, avvisi di modifica e il mantenimento di specifici notices. Un prezzo di download pari a zero non elimina tali obblighi ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). Per combinazioni di componenti possono applicarsi più licenze contemporaneamente o in alternativa.

Le SPDX License Expressions modellano tali casi in modo leggibile dalla macchina: `AND` per obblighi cumulativi, `OR` per una scelta di licenza e `WITH` per un’eccezione. Un’espressione SPDX identifica la situazione di licenza dichiarata, ma non esegue alcuna verifica di compatibilità legale ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Per gli amministratori, file di licenza e notice, SBOM/elenco dei componenti, canale di distribuzione e modifiche proprie devono rientrare nell’evidenza di rilascio e archiviazione.

## Strumenti amministrativi per contatori riproducibili

Gli esempi mostrano interrogazioni tecniche di inventario e raccolta di evidenze. Se un campo sia rilevante ai fini della licenza deve essere dedotto dai Terms. Le esportazioni possono contenere dati personali e richiedono controllo degli accessi, conservazione e limitazione delle finalità.

### Rilevare l’inventario di CPU, core e virtualizzazione

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) legge dati di inventario CIM/WMI; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) raccoglie dati di CPU, core, thread, socket e NUMA. In una VM mostrano principalmente la vista del guest. Scope dell’host, spostamento del cluster, tipo di istanza cloud e fattori core contrattuali vengono documentati separatamente.

### Conteggiare oggetti della directory con scope esplicito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) e [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) forniscono ancoraggi degli oggetti e attributi. Vengono documentati Search Base, filtro, paging, attributi multivalore e oggetti disattivati. L’esportazione non conta «persone soggette a licenza» finché la regola contrattuale non è mappata esattamente su questi campi.

### Leggere SKU cloud e assegnazioni utenti

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) e [`curl`](https://curl.se/docs/manpage.html) leggono dati Microsoft Graph. I token non vengono salvati in script, ticket o cronologia della shell. `ConsumedUnits`, assegnazione utenti, provisioning del piano di servizio e workload effettivamente utilizzato vengono valutati separatamente.

### Verificare il server di licenza o endpoint cloud dalla rete del prodotto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano la connessione DNS/TCP dall’origine scelta. Un esito positivo non dimostra né proxy, [TLS](/kb/tls), token, associazione dell’account né la transazione di licenza. I log del prodotto e lo stato del produttore forniscono la transizione di stato successiva.

### Proteggere l’esportazione di audit dalle modifiche

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) rilevano successive modifiche a livello di byte. Vengono inoltre documentati ora di creazione, scope della query, versione dello strumento/API, query, fuso orario, esportatore e archiviazione sicura. Un hash non conferma la completezza sostanziale.

### Verificare gli eventi di licenza e attivazione nel periodo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) e [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrano eventi locali. I log del produttore possono utilizzare codici strutturati anziché messaggi testuali in inglese; sono preferibili provider, Event ID o campi definiti. Ora, rotazione e copia centralizzata dei log fanno parte del riscontro.

Da entitlement, punto di conteggio e stato operativo nasce l’evidenza di audit. Deve mostrare in modo riproducibile cosa è stato acquistato, assegnato, installato ed effettivamente utilizzato.

## Evidenza di audit e operativa

Un’evidenza di licenza riproducibile contiene:

- riferimento del contratto/ordine, prodotto/SKU, metrica, quantità, scope, durata e diritti speciali;
- inventario software e hardware con ID immutabili, edizione, versione e relazione di cluster;
- assegnazioni dirette, basate su gruppi e automatiche con fonte e stato di provisioning;
- valori di misurazione grezzi e regola di calcolo per ciascuna finestra temporale;
- eccezioni con motivazione, owner, data di scadenza e controllo tecnico;
- HA/DR/test/migrazione come popolazione distinta;
- indicazione del prodotto, esportazione API/CLI e ricalcolo indipendente;
- ora, fuso orario, versione query/strumento, hash e archiviazione protetta.

Le discrepanze vengono classificate: **errore dei dati** (duplicati, vecchi account), **errore del modello** (metrica errata), **errore di processo** (licenza non rimossa dopo l’uscita), **errore tecnico** (sync/token/server) o **entitlement mancante**. Solo questa separazione mostra se la risposta sia pulizia, configurazione, chiarimento contrattuale, approvvigionamento o Incident Response.

## Evoluzione tecnica delle licenze

Le chiavi prodotto locali e i dongle collegavano inizialmente strettamente il diritto d’uso a un computer. I server di licenza di rete hanno introdotto pool condivisi e lease concorrenti; la virtualizzazione ha richiesto nuove regole per host, core e mobilità. Abbonamenti e SaaS hanno spostato gli entitlement in account cloud, gruppi di identità e piani di servizio. Cisco Smart Licensing e Microsoft Graph illustrano modelli centralizzati basati su account/API, mentre ISO/IEC 19770 standardizza dati SWID e di entitlement ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallelamente, Open Source ha reso superflua l’attivazione tecnica per molti componenti, ma non le condizioni di licenza. SPDX ha creato identificatori brevi ed espressioni leggibili dalla macchina per le catene di fornitura software. Il compito dell’amministratore si è quindi spostato da «inserire una chiave» a un problema di dati relativo a contratto, identità, asset, durata, telemetria e composizione del software. L’approccio opposto è lo stesso del monitoraggio: ancoraggi chiari degli oggetti, finestre temporali esplicite, dati grezzi e calcoli riproducibili.

## Fonti

- [ISO/IEC 19770-3:2016 – Entitlement Schema](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – IT Asset Management Systems](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Software Identification Tag](https://www.iso.org/standard/65666.html)
- [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)
- [Cisco – Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)
- [Microsoft Graph PowerShell – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)
- [Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions)
- [Microsoft – Enter Exchange Server Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)
- [Open Source Initiative – Open Source Definition](https://opensource.org/osd)
- [Apache Software Foundation – Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [SPDX Specification 3.0.1 – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)
- [Microsoft Learn – Get-CimInstance](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance)
- [Linux man-pages – lscpu](https://man7.org/linux/man-pages/man1/lscpu.1.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Graph PowerShell – Get-MgUserLicenseDetail](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [GNU Grep – Manual](https://www.gnu.org/software/grep/manual/grep.html)
