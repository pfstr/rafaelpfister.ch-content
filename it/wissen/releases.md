---
title: "Release e aggiornamenti: artefatti, rollout e prova operativa"
blatt: "releases"
description: "Gestione di release e aggiornamenti per sistemi di messaggistica e infrastruttura: versioni, canali, ciclo di vita, integrità degli artefatti, prioritizzazione delle vulnerabilità, dipendenze, rollout graduale, migrazione dello stato, rollback, verifica ed evoluzione tecnica."
fakten:
  - label: Processo
    wert: identificare · prioritizzare · acquisire · installare · verificare l'installazione
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Unità di release
    wert: artefatto + versione + piattaforma di destinazione + metadati + prova di integrità + dipendenze
    href: https://theupdateframework.io/specification/latest/
  - label: Versione
    wert: interpretare solo nel modello di versionamento documentato; SemVer non è universale
    href: https://semver.org/
  - label: Prioritizzazione
    wert: sfruttamento · esposizione · impatto · misura disponibile · capacità di recovery
    href: https://www.cisa.gov/known-exploited-vulnerabilities-catalog
  - label: CVSS
    wert: gravità tecnica e metriche di contesto; non è da solo una scadenza per l'applicazione di patch
    href: https://www.first.org/cvss/v4.0/specification-document
  - label: EPSS
    wert: probabilità statistica di sfruttamento osservato; non è una valutazione del danno
    href: https://www.first.org/epss/how-it-works.html
  - label: Integrità
    wert: verificare provenienza, hash, firma ed eventualmente build provenance
    href: https://slsa.dev/spec/v1.2/
  - label: Rollout
    wert: pilot/Canary → piccole ondate → ondate ampie; ogni ondata ha criteri di interruzione
    href: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
  - label: Rollback
    wert: pianificare separatamente il ripristino di codice, dati e configurazione
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Collaudo della messaggistica
    wert: inbound · outbound · submission · autenticazione · coda · policy · store · monitoraggio
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Limite di supporto
    wert: valutare insieme prodotto, branch, livello di patch, piattaforma e dipendenze
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq
  - label: Prova operativa
    wert: versione prevista/effettiva, ID artefatto, change, ora, risultato, deviazione e responsabile
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - releases
  - patch-management
  - messaging
translationSourceHash: f5adb284b43cb4bb0db285f310beabb3b7e8b6ac825ea0abf603e08062c0cbbb
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:43:16.436Z
translationReview: automatic
---

# Release e aggiornamenti: artefatti, rollout e prova operativa

Una release è un'unità denominata e pubblicata di software o firmware; un aggiornamento è il passaggio controllato di un sistema in esercizio da uno stato approvato a un altro. Per gli amministratori, ciò comprende molto più del file di installazione: piattaforma di destinazione, relazione tra versione e supporto, dipendenze, migrazioni di configurazione e dati, firma o altra prova di provenienza, problemi noti, percorso di installazione, criteri di interruzione e verifica funzionale. NIST definisce di conseguenza l'Enterprise Patch Management come identificazione, prioritizzazione, acquisizione, installazione e verifica di patch, aggiornamenti e upgrade ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

I sistemi di messaggistica rendono questo compito più complesso. I listener Internet elaborano input esterni, le code portano responsabilità non ancora concluse, i cluster distribuiscono lo stato e gli utenti si aspettano una consegna continua. Un programma di installazione completato con successo non dimostra quindi né un flusso di posta funzionante né uno stack complessivo supportato. La gestione delle release è una modifica dello stato di un'architettura di dipendenze, con una base di partenza misurabile, una propagazione controllata e uno stato finale verificabile.

La spiegazione parte dall'artefatto concreto e dalla sua provenienza. Seguono poi tipo di release, valutazione del rischio, test e rollout graduale; interruzione, rollback e verifica concludono il percorso di modifica.

Un aggiornamento è il passaggio controllato da una versione software nota a un nuovo artefatto. Il processo sicuro inizia con provenienza e firma, passa per qualificazione e rollout graduale e termina solo con l'accettazione tecnica e funzionale.

## Oggetto della release e catena di fiducia

Un'approvazione affidabile descrive l'**oggetto di release esatto**. Il nome commerciale «Versione 16» non basta se edizione, build, livello di hotfix, piattaforma, architettura o digest dell'immagine determinano il binario realmente eseguito. Neppure un nome file è un'identità: in un modello di versionamento pulito, un artefatto già pubblicato non deve essere sostituito silenziosamente. Semantic Versioning richiede, ad esempio, che il contenuto di una versione pubblicata rimanga invariato; questa regola vale però solo se il produttore garantisce effettivamente SemVer ([Semantic Versioning 2.0.0](https://semver.org/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-releases.svg?v=20260813" title="Interaktive Infografik: Releaseobjekt, Vertrauenskette, Risikopriorisierung, Betriebsmodelle, gestufter Rollout, Abbruch, Rollback und Verifikation" loading="lazy">
  <a href="/images/kb-interaktiv-releases.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

Un oggetto di release comprende almeno:

| Componente | Domanda dell'amministratore | Prova |
|---|---|---|
| Artefatto | Quali byte vengono installati o avviati? | Hash, ID pacchetto, digest dell'immagine o identificativo del firmware |
| Produttore e provenienza | Da quale canale approvato proviene? | Repository HTTPS, metadati firmati, portale del produttore, catena di trasferimento offline |
| Versione e destinazione | Per quale prodotto, edizione, sistema operativo, architettura e release di partenza? | Note di rilascio, matrice di compatibilità, percorso di upgrade |
| Integrità e autenticità | L'artefatto è invariato e autorizzato dall'editore previsto? | Firma, verifica del certificato/della chiave, hash da fonte autenticata |
| Contenuto e provenienza | Quali componenti e quale build lo hanno prodotto? | SBOM, build provenance, attestazione, identità del sistema di build |
| Modifica dello stato | Quali schemi, indici, code, configurazioni o chiavi vengono modificati? | Piano di migrazione, passaggi irreversibili, spazio richiesto e durata |
| Contratto operativo | Quali servizi, porte, API, valori predefiniti e dipendenze modificano il proprio comportamento? | Breaking changes, deprecazioni, problemi noti, nuovi livelli minimi |

The Update Framework (TUF) modella una distribuzione sicura degli aggiornamenti mediante ruoli firmati per metadati Root, Targets, Snapshot e Timestamp. L'obiettivo è limitare, tra l'altro, chiavi di repository compromesse, rollback a stati precedenti e metadati congelati. SLSA descrive la build provenance come informazione verificabile su come, dove e da quali input è nato un artefatto. Entrambi gli approcci mostrano che un hash conferma soltanto l'uguaglianza con un valore di riferimento; solo la provenienza affidabile di tale valore di riferimento risponde alla questione dell'autenticità ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

## Stack tecnologico del percorso di release

La gestione delle release è essa stessa uno stack tecnologico. Collega sistema sorgente e di build, artefatto o formato di pacchetto, procedura di firma e provenienza, repository o registry, canale di distribuzione, programma di installazione oppure orchestratore, migrazione di configurazione e schema, nonché telemetria e controllo del rollback. TUF descrive i ruoli dei metadati firmati di un repository; SLSA descrive la provenance tra stato sorgente, piattaforma di build e artefatto ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

Per un prodotto di destinazione si aggiunge il suo stack: sistema operativo, architettura CPU, runtime, database, directory, proxy, estensioni e client. Una release è qualificata solo quando questi due stack sono compatibili. Un'immagine container firmata validamente può essere inutilizzabile sull'architettura sbagliata; un gestore di pacchetti completato con successo non può comunque annullare una migrazione di database non supportata.

## Versioni, canali e limiti di supporto

I numeri di versione sono nomi all'interno di un contratto del produttore, non una scala universale del rischio. Con SemVer, `MAJOR.MINOR.PATCH` indica una modifica API incompatibile, una funzionalità retrocompatibile e una correzione di bug retrocompatibile. I produttori possono però utilizzare modelli basati su calendario, build, firmware, aggiornamenti cumulativi, funzionalità o abbonamento. Da `3.9.1` non segue, senza documentazione, né che `3.10.0` sia compatibile né che una patch sia piccola o a basso rischio ([Semantic Versioning 2.0.0](https://semver.org/)).

Anche «stable», «LTS», «extended», «preview», «early access» e «general availability» sono canali di release specifici del produttore. Un inventario amministrativo non memorizza quindi solo la versione in esecuzione, ma anche il canale, la fine del supporto, i percorsi supportati per predecessori/successori e la matrice di piattaforma. Decisivo è l'**insieme di supporto** composto da branch del prodotto, livello di patch, sistema operativo, runtime, database, directory, browser/client, estensioni e, se necessario, modello hardware.

Il modello di manutenzione Exchange di Microsoft illustra questa multidimensionalità: Cumulative Updates, Security Updates e Hotfix Updates hanno prerequisiti e relazioni differenti; i Security Updates sono specifici della CU e possono inoltre essere necessarie misure manuali. La regola amministrativa non è quindi «Exchange viene sottoposto a patch ogni mese», ma: per ogni server vengono verificati congiuntamente lo stato CU/SU supportato, lo stato di Windows, i Management Tools e le azioni aggiuntive associate allo specifico advisory ([Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)).

Quando artefatto, canale e stato di supporto sono definiti, viene valutato il rischio della modifica. Un numero di versione elevato da solo non indica né urgenza né compatibilità.

## Prioritizzazione: rischio invece del numero di versione

Non ogni nuova versione è urgente e non ogni vulnerabilità critica può attendere la prossima finestra ordinaria. La prioritizzazione combina più segnali indipendenti:

1. **Prova dello sfruttamento:** CISA gestisce il Known Exploited Vulnerabilities Catalog quale elenco autorevole delle vulnerabilità sfruttate attivamente e ne raccomanda l'uso come input per la prioritizzazione ([CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)).
2. **Gravità tecnica:** CVSS v4.0 descrive metriche Base, Threat, Environmental e Supplemental. Il punteggio non è un'affermazione completa sull'asset proprio o su una data di applicazione della patch ([FIRST – CVSS v4.0](https://www.first.org/cvss/v4.0/specification-document)).
3. **Probabilità di sfruttamento:** EPSS stima la probabilità che una CVE pubblicata venga effettivamente sfruttata entro un intervallo di tempo. EPSS non valuta il possibile danno e non sostituisce la conoscenza degli asset ([FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)).
4. **Esposizione propria:** listener Internet, interfaccia di gestione raggiungibile, autenticazione presente, funzione attiva, versione, controlli compensativi e valore dei dati determinano se il percorso descritto esiste nel proprio sistema.
5. **Impatto operativo:** indisponibilità, crescita della coda, migrazione dei dati, perdita del cluster o dipendenze non testate possono essere causati dall'aggiornamento stesso.
6. **Capacità di recovery:** un rollback testato, nodi sostitutivi e un [modello di backup/DR](/kb/backup-dr) aggiornato consentono una velocità diversa rispetto a un upgrade irreversibile dello schema senza prova di ripristino.

Il risultato è una scadenza motivata con un responsabile, non soltanto «CVSS 9.8». Una vulnerabilità sfruttata attivamente su un gateway Internet può richiedere misure immediate, riduzione dell'esposizione e una finestra di emergenza. Un componente locale non utilizzato su un sistema isolato può essere pianificato in un'ondata regolare con una compensazione documentata. Le eccezioni ricevono una data di scadenza; altrimenti diventano deviazioni permanenti invisibili.

## Approcci architetturali per gli aggiornamenti

Il rollback e l'ambito dell'indisponibilità dipendono dall'approccio architetturale:

| Modello operativo | Modifica dello stato | Modello di rollout | Limite centrale |
|---|---|---|---|
| Server o appliance in-place | File binari, pacchetti e spesso configurazione sullo stesso nodo | Modalità manutenzione, aggiornare il nodo, verificare, riprendere | un aggiornamento parziale o un programma di installazione non riuscito può lasciare il nodo in uno stato incerto |
| Cluster con ruoli stateful | Nodo più stato di replica/quorum | Spostare i ruoli, isolare un nodo, aggiornare, risincronizzare, quindi ondata successiva | funzionamento con versioni miste, quorum, carico di rebuild e ordine delle versioni |
| Immagine immutabile / container | una nuova immagine sostituisce la vecchia istanza; configurazione e dati risiedono all'esterno | Canary, RollingUpdate, Blue/Green | il rollback dell'immagine non aiuta contro una migrazione dati irreversibile |
| SaaS / servizio gestito | il produttore distribuisce il servizio; il cliente controlla configurazione e integrazioni | ondate annunciate, Feature Flags, compatibilità tenant/API | nessun rollback binario da parte del cliente; contratto, telemetria e test delle integrazioni sono centrali |
| Firmware / appliance hardware | firmware, partizioni di boot, driver ed eventualmente formato dati | prima nodo standby/HA o doppio banco, poi lato attivo | avviabilità, modello hardware e percorso di downgrade sono specifici del produttore |

I Deployment Kubernetes mostrano esplicitamente il modello Rolling: `maxUnavailable` e `maxSurge` limitano rispettivamente i Pod mancanti e aggiuntivi durante la sostituzione; il rollout può essere osservato, sospeso e riportato a una revisione memorizzata. Ciò garantisce soltanto lo stato del controller. Una migrazione del database, uno stato di coda esterno o client incompatibili restano al di fuori di questo rollback ([Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)).

Per i cluster di messaggistica, anche «prima lo standby» è corretto solo se il produttore supporta questo percorso. Cluster di configurazione peer-to-peer, pool di load balancer, replica di database e alta disponibilità delle code sono meccanismi diversi. Prima dell'aggiornamento deve essere chiaro quali stati vengono replicati, quali versioni possono collaborare temporaneamente e se un nodo aggiornato scrive dati in un formato che il vecchio nodo può ancora leggere.

## Le note di rilascio come contratto tecnico

Le note di rilascio non vengono sfogliate cercando parole chiave, ma tradotte in una matrice locale delle modifiche:

- **Sicurezza:** CVE, componente/funzione interessata, vettore di attacco, privilegio, sfruttamento noto, mitigazione e azione aggiuntiva.
- **Prerequisiti:** versioni iniziali consentite, release intermedie, spazio libero, runtime, sistema operativo, schema, licenza e modello hardware.
- **Comportamento:** nuovi valori predefiniti, porte modificate, cipher/protocolli, campi API/CLI, autenticazione, certificati, routing della posta e limiti.
- **Rimozione:** deprecazioni, funzionalità rimosse, algoritmi obsoleti, piattaforme o interfacce di gestione ritirate.
- **Migrazione:** durata, blocchi, creazione di code/indici, formato dei dati, retrocompatibilità e attività successive necessarie.
- **Problemi noti:** trigger, rilevamento, workaround, obiettivo della correzione ed effetto su rollback o HA.
- **Percorso di installazione:** sequenza, riavvii, ruoli dei nodi, health check e passaggi manuali successivi.

La documentazione del produttore è la fonte primaria; forum e resoconti di esperienza possono fornire ipotesi, ma non sostituiscono una relazione di upgrade supportata. NIST SSDF richiede di archiviare il software approvato e i relativi componenti, proteggerne l'integrità e fornire informazioni per la correzione delle vulnerabilità. Per gli operatori ne consegue che note di rilascio, artefatti, informazioni su hash/firme, decisione di approvazione e risultati dei test locali siano conservati insieme in modo referenziabile ([NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

Le note di rilascio descrivono le modifiche attese; la qualificazione le verifica rispetto alla topologia, al volume di dati, alle integrazioni e alle possibilità di ripristino proprie.

## Qualificazione prima della finestra di manutenzione

L'ambiente di test non deve avere le stesse dimensioni, ma deve essere rappresentativo nelle **caratteristiche rilevanti per il rischio**: stessa edizione del prodotto, autenticazione, catene di certificati, versione di directory e database, regole di flusso della posta, estensioni, dimensioni dei messaggi, protocolli e percorso di upgrade. Un laboratorio appena installato non testa un errore di upgrade che si manifesta solo con una configurazione cresciuta nel corso degli anni.

Prima dell'approvazione vengono verificati almeno:

1. download da fonte approvata e prova di integrità/firma;
2. prerequisiti e percorso intermedio documentato;
3. backup di configurazione e dati con percorso di ripristino noto;
4. upgrade di una copia rappresentativa o di un nodo pilota;
5. transazioni funzionali, inclusi casi negativi e decisioni di policy;
6. prestazioni, coda, replica, indice/rebuild e crescita dello spazio;
7. effetto di monitoraggio e allarmi durante indisponibilità, degrado e ritorno;
8. rollback o ripristino fino al punto decisionale definito;
9. durata con riserva per interruzione, recovery e verifica.

Uno snapshot non è un rollback universale. Nei sistemi distribuiti, database, coda, blob store, directory, certificati e peer esterni possono continuare a operare dopo lo snapshot. Il ripristino di un singolo nodo può generare uno stato incoerente o già rielaborato. Il [modello di backup e DR](/kb/backup-dr) deve indicare il punto di consistenza comune e i limiti di responsabilità.

## Rollout graduale e criteri di interruzione

Un rollout controllato limita il raggio del danno attraverso ondate:

1. **Pilota/Canary:** unità piccola, osservabile e rappresentativa senza funzione esclusiva.
2. **Prima ondata:** più sistemi o utenti, affinché emergano dipendenze più rare.
3. **Ondate ampie:** capacità, regioni, ruoli o tenant in batch definiti.
4. **Osservazione successiva:** almeno per i cicli rilevanti di retry, cache, coda, certificati e carico giornaliero.

Ogni ondata ha **criteri Go, Hold e Abort** definiti in anticipo: tasso di errore, latenza, età/profondità della coda, distribuzione SMTP 4xx/5xx, errori di autenticazione, ritardo della replica, risorse, crash loop, integrità dei dati e segnalazioni al supporto. «Nessun ticket» non è un health check sufficiente. L'ondata successiva inizia solo quando periodo di misurazione e criteri sono soddisfatti.

In caso di abort non si esegue automaticamente il rollback. Se un rollback è più pericoloso del mantenimento, ad esempio dopo una modifica dello schema orientata in avanti, la reazione sicura può essere: sospendere il rollout, rimuovere il traffico dal nodo interessato, proteggere i dati e seguire il percorso del produttore. La matrice decisionale viene preparata prima della finestra, non improvvisata sotto pressione temporale.

Dopo una semplice modifica binaria, un rollback può essere facile. Non appena vengono modificati schema, indice o dati memorizzati, deve essere chiarito in anticipo se il ritorno sia ancora tecnicamente possibile.

## Rollback, roll-forward e migrazione dei dati

«Tornare alla vecchia versione» si scompone in almeno quattro questioni indipendenti di ritorno:

- **Codice:** il vecchio pacchetto, immagine o firmware è disponibile e firmato? Il produttore supporta il downgrade?
- **Configurazione:** il formato è stato modificato e la vecchia versione comprende la nuova configurazione?
- **Dati:** uno schema o un indice è stato migrato in avanti? Il vecchio software può leggere tali dati o è necessario un ripristino?
- **Effetto esterno:** sono stati accettati messaggi, emessi token, modificato DNS, ruotati certificati o utilizzate API da altri sistemi?

Con frontend stateless, il rollback di un'immagine è spesso rapido. Per una piattaforma di posta, lo stesso passaggio può elaborare i messaggi due volte o perdere stati accettati di recente. Esiste quindi un **Point of No Return**: fino a quel punto viene testato un rollback tecnico; in seguito sono previsti roll-forward, ripristino o una modalità di compatibilità definita.

Il rollback non sostituisce inoltre l'analisi della causa. Dopo un ritorno rimangono ipotesi diverse: artefatto difettoso, prerequisito incompleto, deviazione locale, limite di capacità e errore di prodotto sconosciuto. Il riscontro della release registra versione esatta, host, fase, codice di errore, ora e log protetti e segue una [metodologia di troubleshooting basata sulle evidenze](/kb/troubleshooting).

## Strumenti amministrativi per inventario e verifica

I comandi forniscono elementi per una prova della modifica; non sostituiscono gli health checker del produttore né le istruzioni di upgrade specifiche del prodotto. L'output viene salvato prima e dopo l'aggiornamento con ora, host e ID della modifica. Segreti o configurazioni complete non devono essere inseriti senza protezione nei ticket.

### Rilevare l'inventario di piattaforma e patch

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ComputerInfo -Property WindowsProductName,WindowsVersion,OsBuildNumber,OsArchitecture
Get-HotFix | Sort-Object InstalledOn,HotFixID</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">uname -a
cat /etc/os-release
rpm -qa --qf '%{NAME}\t%{EPOCHNUM}:%{VERSION}-%{RELEASE}\t%{ARCH}\n' 2&gt;/dev/null \
  || dpkg-query -W -f='${binary:Package}\t${Version}\n'</code></pre>
  </div>
</div>

[`Get-ComputerInfo`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo) e [`Get-HotFix`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix) rilevano le informazioni di Windows e degli aggiornamenti registrati. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html), [`os-release`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html), [`rpm`](https://rpm.org/docs/6.0.x/man/rpm.8) e [`dpkg-query`](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html) forniscono dati sul sistema operativo e sui pacchetti in Linux. Build del prodotto, firmware dell'appliance, digest del container ed estensioni devono essere rilevati in aggiunta con gli strumenti dei rispettivi produttori.

### Verificare hash e firma dell'artefatto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash C:\\Staging\\update.msi -Algorithm SHA256
Get-AuthenticodeSignature C:\\Staging\\update.msi |
  Format-List Status,StatusMessage,SignerCertificate,TimeStamperCertificate</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum update.tar.gz
gpgv release-signature.asc release-manifest.txt</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) confrontano i byte con un valore di riferimento autenticato. [`Get-AuthenticodeSignature`](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature) e [`gpgv`](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html) verificano le firme rispetto all'ancora di fiducia configurata. Uno stato «valido» risponde alla questione dell'editore solo se certificato, chiave, nome, periodo di validità e percorso di approvvigionamento sono quelli attesi.

### Confrontare servizi e listener prima e dopo il rollout

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Dienst- und Listenerprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Service | Sort-Object Name | Select-Object Name,Status,StartType
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort | Select-Object LocalAddress,LocalPort,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">systemctl --failed --no-pager
systemctl list-units --type=service --state=running --no-pager
ss -lntup</code></pre>
  </div>
</div>

[`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service), [`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection), [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano lo stato locale di servizi e listener. Un processo in esecuzione non è ancora un health check funzionale; indirizzo di bind, porta, processo, TLS, risposta del protocollo e dipendenze successive vengono verificati separatamente.

### Verificare la raggiungibilità di un endpoint rappresentativo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Smoke-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection admin.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
nc -vz -w 5 admin.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) dimostrano la connessione TCP dall'origine selezionata. Seguono test di [TLS](/kb/tls)-, [SMTP](/kb/smtp)-, autenticazione e flusso di posta funzionale; una porta aperta non conferma un servizio funzionante.

### Salvare errori e avvisi nella finestra temporale della modifica

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T18:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since; Level=1,2,3} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 18:00:00 UTC' \
  --priority warning --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) e [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) delimitano il riscontro alla finestra di manutenzione. I log dell'installer, del prodotto, del cluster e del database vengono inoltre salvati con fonte esatta e base temporale. La verifica non cerca solo nuovi errori, ma anche eventi attesi assenti, riavvii ripetuti e retry in crescita.

Uno stato del processo privo di anomalie non è sufficiente per la messaggistica. Accettazione, coda, policy, consegna in uscita, accesso client e tracciamento devono essere verificati lungo il percorso reale dei messaggi.

## Collaudo specifico della messaggistica

Dopo l'aggiornamento non viene inviata solo un'email di test. Il collaudo riproduce i ruoli della piattaforma:

- **Relay inbound:** selezione DNS/MX esterna, TCP/TLS, verifica del destinatario, accettazione dopo DATA, policy di contenuto e dominio, coda e consegna finale.
- **Relay outbound:** selezione del connector/smarthost, IP sorgente, policy TLS, [SPF/DKIM/DMARC](/kb/mail-auth), retry della coda e accettazione da parte di un destinatario controllato.
- **Submission e accesso:** autenticazione, autorizzazione, IMAP/HTTP/API, Send-as/Delegate e client mobili/legacy nella misura supportata.
- **Identità:** [LDAP](/kb/ldap)-bind e ricerca, risoluzione di gruppi/attributi, [Kerberos](/kb/kerberos), token e certificati.
- **Policy e crittografia:** malware/DLP, quarantena, disclaimer, routing, [S/MIME/OpenPGP/Portal](/kb/verschluesselung) e accesso alle chiavi.
- **Stato e HA:** coda, database, object store, replica, indice, agente di backup, failover e ritorno del nodo aggiornato.
- **Amministrazione:** GUI, API, CLI, ruoli, audit log, esportazione della configurazione e monitoraggio.

Ogni test dispone di risposta attesa e reale, ID di correlazione, ora, origine e destinazione. Un Synthetic Check verde e un piccolo pilota reale si completano a vicenda: il Synthetic Check è riproducibile; il pilota individua combinazioni impreviste di client, policy e carico. La separazione tra trasporto, autenticazione del dominio e meccanismi di contenuto segue l'architettura tecnica dell'email definita in NIST SP 800-177 Rev. 1 ([NIST – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)).

## Evoluzione tecnica della gestione delle release

In passato, gli stati dei sistemi venivano spesso distribuiti come supporti completi o patch del produttore installate manualmente. Gestori di pacchetti, repository firmati e modelli di manutenzione cumulativi hanno automatizzato dipendenze e distribuzione; macchine virtuali, immagini e container hanno spostato l'attenzione sulla sostituzione riproducibile delle istanze. CI/CD ha ridotto il tempo tra build e produzione, mentre TUF, SLSA e SSDF modellano esplicitamente la catena di fiducia tra codice sorgente, sistema di build, metadati e artefatto ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/), [NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

La conseguenza operativa non è «installare tutto automaticamente», bensì **identificare in modo leggibile dalle macchine ogni modifica dello stato, qualificarla in modo riproducibile, distribuirla in modo limitato e verificarla funzionalmente**. L'automazione riduce le deviazioni manuali; senza ondate e logica di interruzione aumenta allo stesso tempo la velocità di un errore. Le moderne piattaforme di release richiedono quindi gli stessi controlli classici — inventario, approvazione, finestra di manutenzione e recovery — integrati da provenance dell'artefatto, stato desiderato dichiarativo, telemetria e distribuzione progressiva.

## Fonti

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [NIST SP 800-177 Rev. 1 – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [The Update Framework – Specification](https://theupdateframework.io/specification/latest/)
- [SLSA – Specification 1.2](https://slsa.dev/spec/v1.2/)
- [Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)
- [CISA – Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [FIRST – CVSS v4.0 Specification](https://www.first.org/cvss/v4.0/specification-document)
- [FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)
- [Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [NIST – SP 800-218, Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [Microsoft Learn – Get-ComputerInfo](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo)
- [Microsoft Learn – Get-HotFix](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix)
- [GNU Coreutils – uname](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html)
- [systemd – os-release](https://www.freedesktop.org/software/systemd/man/latest/os-release.html)
- [RPM – rpm manpage](https://rpm.org/docs/6.0.x/man/rpm.8)
- [Debian – dpkg-query](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-AuthenticodeSignature](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [GnuPG – gpgv](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
