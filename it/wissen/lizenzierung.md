---
title: "Licenze: diritti d’uso, metriche e punti di conteggio tecnici"
blatt: "lizenzierung"
description: "Licenze tecniche per amministratori di infrastrutture e messaggistica: entitlement, metriche per utenti, dispositivi, istanze, core e capacità, edizioni, abbonamenti, attivazione, server delle licenze e API cloud, fonti di identità, HA/DR, limiti tecnici di enforcement, dati di audit, Open Source e verifica."
fakten:
  - label: Determinanti
    wert: Contratto, Product Terms e ordine; la visualizzazione tecnica è una prova, non il contratto legale
    href: https://www.iso.org/standard/52293.html
  - label: Tre quantità
    wert: diritti acquisiti · assegnati tecnicamente · utilizzati effettivamente
    href: https://www.iso.org/standard/52293.html
  - label: Metriche
    wert: utente · dispositivo · istanza · core/vCPU · capacità · transazione · funzionalità
    href: https://www.iso.org/standard/52293.html
  - label: Punto di conteggio
    wert: scope + ancoraggio dell’oggetto + filtro di stato + finestra temporale + regola di aggregazione
    href: https://www.iso.org/standard/68531.html
  - label: ID software
    wert: I tag SWID standardizzano l’identificazione del prodotto, non automaticamente il diritto d’uso
    href: https://www.iso.org/standard/65666.html
  - label: Licenze basate sull’identità
    wert: valutare separatamente assegnazione diretta e basata su gruppi, SKU e piano di servizio
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA e DR
    wert: valutare nodi passivi, cold standby, istanze di test e di ripristino solo in base ai Terms concreti
    href: https://www.iso.org/standard/52293.html
  - label: Enforcement
    wert: avviso, limite di funzionalità, nessuna nuova assegnazione, Grace Period o interruzione del servizio dipendono dal prodotto
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Funzionamento offline
    wert: documentare durata di token/lease, Grace Period, orologio, Trust Store e percorso di ripristino
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Open Source
    wert: utilizzabile gratuitamente non significa senza obblighi; verificare il testo della licenza e lo scenario di distribuzione
    href: https://opensource.org/osd
  - label: Leggibile dalle macchine
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
translationSourceHash: fb38ff7080a593642081c7fa017dd2d33735c4fd96cdf1ae256bebe4be37eb3d
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:41:28.650Z
translationReview: automatic
---

# Licenze: diritti d’uso, metriche e punti di conteggio tecnici

La concessione di licenze software collega un contratto legale di utilizzo a oggetti tecnicamente misurabili. Per gli amministratori è fondamentale non confondere questi livelli. Una chiave di licenza, una visualizzazione cloud o un contatore utenti interno possono attivare funzionalità e misurare l’utilizzo; tuttavia, non definiscono da soli ciò che un’organizzazione è legalmente autorizzata a utilizzare. ISO/IEC 19770-3 tratta esplicitamente i dati digitali di entitlement come rappresentazione dei diritti d’uso e chiarisce che le condizioni di licenza originali prevalgono ai fini legali ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Il compito operativo è quindi: **confrontare in modo riproducibile i diritti acquisiti, i diritti tecnicamente installati o assegnati e l’utilizzo effettivo**. Le discrepanze possono indicare un utilizzo eccessivo, costi inutilizzati, filtri di directory errati, account orfani, nodi di cluster passivi, token scaduti o semplicemente definizioni differenti della stessa parola «utente». L’articolo descrive la prospettiva tecnica; l’interpretazione contrattuale e legale resta di competenza degli acquisti, della gestione delle licenze e della consulenza legale.

La spiegazione parte dal diritto d’uso contrattuale e segue il modo in cui prodotto, metrica e punto di conteggio tecnico ne ricavano un consumo misurato. Seguono assegnazione, enforcement, audit, casi particolari e ripristino.

Una licenza è anzitutto un diritto d’uso; diventa tecnicamente visibile solo tramite assegnazione, utilizzo misurato ed eventualmente enforcement. L’articolo separa questi quattro livelli prima di trattare metriche di prodotto e audit.

## Entitlement, assegnazione, utilizzo ed enforcement

Un bilancio delle licenze corretto mantiene quattro stati indipendenti:

1. **Entitlement:** quali diritti d’uso sono stati acquisiti con quale contratto, prodotto, metrica, scope, periodo e diritto speciale?
2. **Distribuzione/assegnazione:** su quali dispositivi, istanze, utenti, tenant o funzionalità il software è installato, attivato o assegnato?
3. **Utilizzo:** quali oggetti o funzionalità rilevanti ai fini della licenza sono stati effettivamente utilizzati nel periodo di misurazione concordato?
4. **Enforcement:** quale limite verifica tecnicamente il prodotto e come reagisce in caso di connessione assente, scadenza o superamento?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

Un entitlement esistente non prova un’assegnazione corretta; un’assegnazione non prova l’utilizzo; un utilizzo ridotto non revoca automaticamente una licenza per utente nominativo. Viceversa, un prodotto può continuare tecnicamente a funzionare benché un diritto di abbonamento o supporto sia scaduto. Conformità e disponibilità tecnica sono quindi due obiettivi di controllo distinti.

ISO/IEC 19770-1 specifica i requisiti per un sistema di IT asset management. ISO/IEC 19770-2 standardizza i Software Identification Tag (SWID), che rendono identificabile il software, ma secondo la norma non prescrivono una riconciliazione degli entitlement. ISO/IEC 19770-3 definisce termini e un formato di trasporto per gli entitlement e le metriche associate. Nel loro insieme forniscono i livelli dati **inventario**, **identità software** e **diritto d’uso**, non un calcolo universale delle licenze ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Metriche di licenza: che cosa viene contato?

Una metrica è interpretabile solo con il contratto completo:

| Metrica | Possibile ancoraggio del conteggio | Domande per gli amministratori |
|---|---|---|
| Utente nominativo | ID immutabile della persona/del tenant | vengono contati account disabilitati, condivisi, esterni, di servizio o di test; può essere riassegnata? |
| Utente/sessione concorrente | sessione attiva o lease di checkout | quale sessione inizia/termina, come vengono trattati timeout, dispositivi multipli e lease offline? |
| Dispositivo | ID hardware, dispositivo registrato, istanza client | VDI, dispositivo sostitutivo, dispositivo condiviso e agenti reinstallati vengono contati separatamente? |
| Server/istanza/nodo | VM, host, appliance, nodo di cluster, istanza container | vengono contate istanze passive, temporanee, con scalabilità automatica, di test o di ripristino? |
| Processore/core/vCPU | socket/core fisico, vCPU assegnata, numero minimo | come vengono calcolati Hyperthreading, affinità, spostamento del cluster, dimensioni cloud e pacchetti minimi? |
| Capacità | caselle di posta, storage, domini, messaggi, throughput o record di dati | vale il picco, la media, il massimo mensile, il provisioned o l’used; quale finestra temporale? |
| Funzionalità/edizione | piano di servizio, modulo, limite di database o API attivati | sono sufficienti installazione, attivazione, configurazione o utilizzo effettivo? |
| Abbonamento/consumo | assegnazione SKU, crediti, richieste, GB-mese | quando viene riservato, consumato, ricalcolato o restituito? |

L’unità tecnica non deve essere dedotta dal nome del prodotto. «Per core» può significare core fisici dell’host, vCPU della VM o una metrica dei core normalizzata. «Utente» può significare persona fisica, account attivo, casella di posta, identità con licenza o mittente che invia. «Istanza» può essere conteggiata come in esecuzione, installata, registrata o per nodo di cluster. ISO/IEC 19770-3 standardizza una struttura di entitlement, ma non sostituisce la definizione concreta nei Product Terms e nell’ordine ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La sola metrica non indica ancora dove avviene il conteggio. Solo il punto di conteggio tecnico spiega perché portale del produttore, inventario locale e valore in fattura possano differire.

## Il punto di conteggio tecnico

Ogni metrica viene documentata come funzione di conteggio riproducibile:

**Contatore = Scope × ancoraggio dell’oggetto × filtro di stato × finestra temporale × regola di aggregazione × eccezioni**

- **Scope:** organizzazione, tenant, dominio, cluster, abbonamento, sede o contratto.
- **Ancoraggio dell’oggetto:** ID utente immutabile, ID dispositivo, ID VM, seriale host, ID SKU o hash, non solo nome visualizzato.
- **Filtro di stato:** enabled, assigned, provisioned, active, seen, mounted, running o consumed.
- **Finestra temporale:** data di riferimento, mese di calendario, picco, media, finestra mobile o anno contrattuale.
- **Aggregazione:** distinct, somma, massimo, 95° percentile, pacchetto minimo o scaglione.
- **Eccezioni:** utenti esterni, account di sistema, istanza DR passiva, trial, NFR o caso d’uso contrattualmente gratuito.

Il punto di conteggio è spesso un **punto di passaggio**. Un’importazione LDAP può contare tutti gli account corrispondenti, anche se non utilizzano mai una funzione di posta. Un gateway può apprendere mittenti dal traffico e quindi mantenere identità eliminate o tecniche. Una SKU cloud può essere assegnata direttamente o indirettamente tramite un gruppo. L’API `licenseDetails` di Microsoft Graph fornisce licenze dirette ed ereditate tramite appartenenza a gruppi, nonché singoli piani di servizio e stati di provisioning; ciò mostra perché un booleano «ha licenza» non sia sufficiente per l’analisi ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Architettura di un sistema di licenze

I prodotti commerciali utilizzano implementazioni diverse, ma possono essere tecnicamente scomposti in superfici di controllo. Questo modello è una sintesi operativa dei livelli dati ISO/IEC 19770 e dei meccanismi documentati dai produttori, non un’architettura normativa universale:

| Superficie di controllo | Funzione | Area di guasto |
|---|---|---|
| Archivio degli entitlement | contratto/SKU, quantità, periodo, funzionalità, diritti speciali | ordine errato, scaduto, tenant/Smart Account errato |
| Fonte di inventario/identità | utenti, dispositivi, istanze, core, cluster, ID software | duplicato, oggetto obsoleto, filtro o scope errato |
| Assegnazione | assegna il diritto a un oggetto o piano di servizio | assegnazione diretta vs. basata su gruppi, errore di provisioning |
| Misuratore | raccoglie utilizzo, sessioni, capacità o heartbeat | tempo, buffer offline, campionamento, telemetria mancante |
| Valutatore | applica metrica, pool, eccezioni e periodo | la logica contrattuale non corrisponde alla policy tecnica |
| Servizio licenze | server locale, API cloud, token, lease, certificato o chiave | DNS/TLS/proxy/orologio/trust, guasto o rate limit |
| Enforcement | attiva edizione/funzionalità o limita il comportamento | blocco rigido, Grace Period, avviso o fail-open/fail-closed |
| Esportazione delle evidenze | dati di audit, utilizzo e assegnazione | rotazione, protezione dei dati, cronologia mancante o esportazione non riproducibile |

Cisco Smart Licensing descrive una gestione centralizzata di account e licenze; Microsoft Graph espone tramite API le SKU acquisite, le assegnazioni e i piani di servizio. Tali sistemi rendono la concessione di licenze una dipendenza distribuita da identità, servizio cloud, rete, TLS e tempo. Un prodotto può continuare a elaborare dati, ma non riuscire a ottenere una nuova licenza; un altro può bloccare funzionalità dopo un periodo offline. Il comportamento concreto deve essere desunto dalla documentazione del produttore e dal contratto ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licenze basate sull’identità

Per gli utenti nominativi, la directory fa parte del sistema di licenze. Un filtro corretto non risponde soltanto a «persone nella OU X», ma anche a:

- quale attributo costituisce l’ancoraggio immutabile dell’oggetto;
- se vengono contati account disabilitati, bloccati, eliminati o non ancora sottoposti a provisioning;
- come vengono trattate Shared/Resource Mailboxes, account di servizio, ospiti, partner esterni e account di test;
- se le assegnazioni dirette e basate su gruppi vengono consolidate;
- quando una licenza rimossa torna disponibile;
- se è rilevante l’utilizzo storico o solo lo stato alla data di riferimento;
- quali attributi il prodotto memorizza localmente nella cache e quando li elimina.

LDAP fornisce voci e attributi, ma non una definizione universale di «persona con licenza». [LDAP](/kb/ldap) può implementare tecnicamente scope e filtro; la metrica deriva dal contratto. Nelle directory cloud vale lo stesso: SKU, piano di servizio, `assignedLicenses`, stato di provisioning e stato effettivo del workload sono dati distinti. Microsoft Graph documenta `subscribedSku` come abbonamento commerciale acquisito e `licenseDetails` per ogni utente ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

La pulizia degli account orfani non deve essere guidata direttamente dalla pressione sulle licenze. Prima vengono verificati proprietari, conservazione, routing della posta, Legal Hold, dipendenze di servizio e ripristino; poi l’identità viene disabilitata o eliminata in modo controllato. Un report sulle licenze è un input nel ciclo di vita, non un ordine di eliminazione.

## Modelli per istanze, core e capacità

La virtualizzazione e i cluster rendono insufficiente il nome del server fisico. Un inventario richiede host, VM/container, vCPU assegnate, CPU/core fisici, cluster hypervisor, regole di mobilità, edizione e ruolo. Se una VM può spostarsi tra host, a seconda del contratto può essere rilevante l’intero scope degli host possibili; un’affinità CPU rigida può essere dimostrabile tecnicamente, ma non è automaticamente riconosciuta contrattualmente.

Le edizioni collegano il diritto d’uso a limiti tecnici. Microsoft documenta, ad esempio, per Exchange edizioni che differiscono tra l’altro nel numero di database montati contemporaneamente; anche le copie passive dei database possono essere conteggiate come database montati. Il Product Key imposta l’edizione del server. Questo è un esempio di come enforcement e architettura della capacità coincidano, non un calcolo generale delle licenze Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Le metriche di capacità necessitano di un andamento temporale. Un valore istantaneo non mostra né il picco mensile né un superamento di breve durata. Per la messaggistica, contatori tecnici diffusi sono in particolare caselle di posta attive, mittenti interni, domini, numero giornaliero di messaggi, throughput, storage e utenti con crittografia; se siano rilevanti ai fini della licenza lo stabilisce solo il diritto sul prodotto. I dashboard memorizzano quindi valore grezzo, tempo, scope, fonte e regola di calcolo invece di un semplice semaforo.

Non appena vengono conteggiate istanze o identità, anche alta disponibilità, test e migrazioni incidono sulla quantità di licenze. I sistemi passivi non sono automaticamente gratuiti; è determinante il contratto applicabile.

## HA, Disaster Recovery, test e migrazione

Nodi passivi, cold standby, istanze di ripristino, laboratori, test, formazione e stati paralleli temporanei durante una [migrazione](/kb/migration) sono casi particolari tipici. Tecnicamente, «passivo» può comunque significare che il software è installato, l’elaborazione della replica è in corso, i database sono montati o una licenza viene estratta dal server. I termini contrattuali devono essere mappati su stati osservabili.

Per ogni caso particolare vengono documentati:

- numero consentito e definizione delle istanze passive/fredde;
- se sono inclusi test, qualificazione delle patch, ripristino da backup o test DR;
- per quanto tempo è consentito il funzionamento simultaneo vecchio/nuovo durante la migrazione;
- se una licenza è mobile e quali condizioni di riassegnazione/attesa si applicano;
- se i diritti cloud e on-premises sono collegati;
- quale evidenza distingue un caso DR reale da un carico produttivo permanente;
- come funziona il servizio licenze in una rete di ripristino isolata.

Il [concetto di backup e DR](/kb/backup-dr) include pertanto anche file di licenza, dati di attivazione, server delle licenze, token offline, certificati, ora, DNS/proxy e contatti del produttore. Un ripristino tecnicamente perfetto che non possa attivare la propria edizione o le funzionalità necessarie non soddisfa l’obiettivo di recovery.

## Scadenza, Grace Period e indisponibilità del servizio licenze

Nei modelli di abbonamento, lease o cloud, sono rilevanti almeno i seguenti timer: fine dell’entitlement, scadenza del token, heartbeat, durata di borrow/checkout, termine offline, Grace Period e validità del certificato. La prospettiva dell’amministratore tiene conto di **ora assoluta, fuso orario, fonte di sincronizzazione e ultimo rinnovo riuscito**. Un orologio errato può causare una scadenza apparente della licenza o impedire la convalida di un token firmato.

Il comportamento alla scadenza è specifico del prodotto:

- solo avviso o evento di conformità;
- nessuna nuova assegnazione, l’utilizzo esistente rimane;
- funzionalità premium disabilitata o ritorno a un’edizione inferiore;
- numero limitato di nuove sessioni/utenti;
- funzionamento in sola lettura;
- interruzione completa del servizio;
- Grace Period locale se il servizio cloud non è raggiungibile.

Questa reazione viene determinata in un ambiente di test o sulla base di una fonte esplicita del produttore, non provata nel funzionamento di produzione. Il monitoraggio avvisa del primo timer operativo, non soltanto della fine del contratto. Cisco documenta per Smart Licensing propri meccanismi online/offline e di account; altri produttori usano server delle licenze locali, file firmati, dongle USB, Product Key o entitlement SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Open Source: diritto d’uso senza contatore tecnico

Open Source non significa soltanto codice sorgente visibile. La Open Source Definition richiede tra l’altro libera ridistribuzione, accesso al codice sorgente, opere derivate e diritti tecnologicamente neutrali. Le singole licenze prevedono condizioni diverse per modifica, ridistribuzione, notices, fornitura del codice sorgente o licenze di brevetto ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

La Apache License 2.0 concede licenze su copyright e brevetti a determinate condizioni e richiede, in caso di ridistribuzione, tra l’altro una copia della licenza, indicazioni delle modifiche e il mantenimento di determinate notices. Un prezzo di download pari a zero non elimina tali obblighi ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). Nei complessi di componenti possono applicarsi più licenze contemporaneamente o in alternativa.

Le SPDX License Expressions modellano tali casi in modo leggibile dalle macchine: `AND` per obblighi cumulativi, `OR` per una scelta di licenza e `WITH` per un’eccezione. Un’espressione SPDX identifica la situazione di licenza dichiarata, ma non effettua una verifica della compatibilità legale ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Per gli amministratori, file di licenza e notice, SBOM/elenco dei componenti, canale di distribuzione e modifiche proprie fanno parte dell’evidenza di rilascio e archiviazione.

## Strumenti amministrativi per contatori riproducibili

Gli esempi mostrano query tecniche di inventario e di evidenza. Se un campo sia rilevante ai fini della licenza deve essere dedotto dai Terms. Le esportazioni possono contenere dati personali e richiedono controllo degli accessi, conservazione e limitazione della finalità.

### Rilevare l’inventario di CPU, core e virtualizzazione

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-CimInstance Win32_ComputerSystem |
  Select-Object Name,Manufacturer,Model,NumberOfProcessors,NumberOfLogicalProcessors,HypervisorPresent
Get-CimInstance Win32_Processor |
  Select-Object DeviceID,Name,SocketDesignation,NumberOfCores,NumberOfLogicalProcessors</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">lscpu --extended=CPU,NODE,SOCKET,CORE,ONLINE
lscpu</code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) legge dati di inventario CIM/WMI; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) raccoglie dati su CPU, core, thread, socket e NUMA. In una VM mostrano principalmente la prospettiva dell’ospite. Scope dell’host, spostamento del cluster, tipo di istanza cloud e fattori contrattuali dei core vengono documentati separatamente.

### Contare oggetti di directory con scope esplicito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -SearchBase 'OU=Licensed,DC=example,DC=ch' -Filter * \
  -Properties ObjectGUID,Enabled,mail,employeeType |
  Select-Object ObjectGUID,SamAccountName,Enabled,mail,employeeType |
  Export-Csv .\\licensed-users.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'ou=Licensed,dc=example,dc=ch' \
  '(&(objectClass=person)(mail=*))' entryUUID uid mail employeeType</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) e [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) forniscono ancoraggi degli oggetti e attributi. Vengono documentati Search Base, filtro, paging, attributi multivalore e oggetti disabilitati. L’esportazione non conta «persone soggette a licenza» finché la regola contrattuale non è mappata esattamente su questi campi.

### Leggere SKU cloud e assegnazioni utente

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-MgSubscribedSku -All |
  Select-Object SkuId,SkuPartNumber,ConsumedUnits,PrepaidUnits,CapabilityStatus
Get-MgUserLicenseDetail -UserId 'admin@example.ch' |
  Select-Object SkuId,SkuPartNumber,ServicePlans</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) e [`curl`](https://curl.se/docs/manpage.html) leggono dati di Microsoft Graph. I token non vengono salvati in script, ticket o cronologia della shell. `ConsumedUnits`, assegnazione utente, provisioning del piano di servizio e workload effettivamente utilizzato vengono valutati separatamente.

### Verificare il server delle licenze o l’endpoint cloud dalla rete del prodotto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection license.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 license.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano la connessione DNS/TCP dall’origine selezionata. Un esito positivo non prova né proxy, [TLS](/kb/tls), token, associazione dell’account né la transazione di licenza. I log del prodotto e lo stato del produttore forniscono la transizione di stato successiva.

### Proteggere l’esportazione di audit dalle modifiche

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\evidence\\license-inventory.csv -Algorithm SHA256
Get-FileHash .\\evidence\\entitlements.pdf -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./evidence/license-inventory.csv
sha256sum ./evidence/entitlements.pdf</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) rilevano successive modifiche dei byte. Vengono inoltre documentati ora di creazione, scope della query, versione dello strumento/API, query, fuso orario, esportatore e archiviazione sicura. Un hash non conferma la completezza sostanziale.

### Verificare eventi di licenza e attivazione nel periodo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-01T00:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=$since} |
  Where-Object Message -Match 'licen[cs]e|activation|entitlement' |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-01 00:00:00 UTC' \
  --output short-iso-precise --no-pager \
  | grep -Ei 'licen[cs]e|activation|entitlement'</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) e [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrano gli eventi locali. I log del produttore possono usare codici strutturati anziché messaggi testuali in inglese; sono preferibili provider, ID evento o campi definiti. Ora, rotazione e copia centrale dei log fanno parte del riscontro.

Da entitlement, punto di conteggio e stato operativo nasce l’evidenza di audit. Deve mostrare in modo riproducibile ciò che è stato acquistato, assegnato, installato e utilizzato effettivamente.

## Evidenza di audit e operativa

Un’evidenza di licenza riproducibile contiene:

- riferimento a contratto/ordine, prodotto/SKU, metrica, quantità, scope, durata e diritti speciali;
- inventario software e hardware con ID immutabili, edizione, versione e relazione di cluster;
- assegnazioni dirette, basate su gruppi e automatiche con fonte e stato di provisioning;
- valori di misurazione grezzi e regola di calcolo per ogni finestra temporale;
- eccezioni con motivazione, owner, data di scadenza e controllo tecnico;
- HA/DR/test/migrazione come popolazione distinta;
- visualizzazione del prodotto, esportazione API/CLI e controcalcolo indipendente;
- ora, fuso orario, versione della query/dello strumento, hash e archiviazione protetta.

Le discrepanze vengono classificate: **errore nei dati** (duplicati, vecchi account), **errore di modello** (metrica errata), **errore di processo** (licenza non rimossa dopo l’uscita), **errore tecnico** (sync/token/server) o **entitlement mancante**. Solo questa separazione mostra se la risposta sia pulizia, configurazione, chiarimento contrattuale, approvvigionamento o incident response.

## Evoluzione tecnica della concessione di licenze

Le chiavi prodotto locali e i dongle collegavano inizialmente il diritto d’uso strettamente a un computer. I server delle licenze di rete hanno introdotto pool condivisi e lease concorrenti; la virtualizzazione ha richiesto nuove regole per host, core e mobilità. Abbonamenti e SaaS hanno spostato gli entitlement in account cloud, gruppi di identità e piani di servizio. Cisco Smart Licensing e Microsoft Graph illustrano modelli centrali di account/API, mentre ISO/IEC 19770 standardizza dati SWID e di entitlement ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallelamente, Open Source ha reso superflua l’attivazione tecnica per molti componenti, ma non le condizioni di licenza. SPDX ha creato identificatori abbreviati ed espressioni leggibili dalle macchine per le catene di fornitura software. Il compito dell’amministratore è così passato da «inserire una chiave» a un problema di dati relativo a contratto, identità, asset, durata, telemetria e composizione software. La contromisura è la stessa del monitoraggio: ancoraggi degli oggetti chiari, finestre temporali esplicite, dati grezzi e calcoli riproducibili.

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
