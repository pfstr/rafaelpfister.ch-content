---
title: "Backup e disaster recovery: stati, obiettivi e ripristino operativo"
blatt: "backup-dr"
description: "Backup e disaster recovery per amministratori di messaggistica: distinzione da snapshot, replica e alta disponibilità, RPO, RTO e MTD, backup coerente di caselle di posta, code, configurazione, chiavi e indici, recovery isolato, ordine di ripristino, test di restore e diagnostica."
fakten:
  - label: Obiettivo
    wert: Riportare servizi e dati a uno stato noto e affidabile
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Base di pianificazione
    wert: Business Impact Analysis e inventario delle dipendenze
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
  - label: MTD
    wert: interruzione massima tollerabile del processo aziendale
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RTO
    wert: indisponibilità massima di una risorsa prima di un impatto inaccettabile
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RPO
    wert: momento fino al quale i dati devono essere ripristinati dopo l'evento
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Backup
    wert: copia ripristinabile con marca temporale; integra la replica
    href: https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup
  - label: Coerenza
    wert: applicazione, software di backup e storage devono coordinare il punto di backup
    href: https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service
  - label: Livelli dei dati
    wert: casella di posta/blob · metadati · coda · indice · configurazione · chiavi
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Cyber-recovery
    wert: copia isolata e ripristino in uno stato affidabile
    href: https://cas8.docs.cisecurity.org/en/latest/source/Controls11/
  - label: Immutabilità
    wert: la retention impedisce l'eliminazione o la sovrascrittura di determinate versioni di oggetti
    href: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
  - label: Regola per le chiavi
    wert: pianificare il recovery in base al tipo di chiave e allo scopo d'uso
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
  - label: Criterio di accettazione
    wert: ripristino riuscito e tempestivo di un servizio aziendale
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - backup
  - disaster-recovery
  - messaging
translationSourceHash: dec2b9d3915258979e801cb9d3ea573f054ae89136cd0631af06b681f36c3ca9
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:37:11.546Z
translationReview: required
---

# Backup e disaster recovery: stati, obiettivi e ripristino operativo

Un job di backup completato con successo dimostra inizialmente soltanto che uno strumento ha scritto dei dati. Non indica ancora se il punto di backup sia completo, coerente con l'applicazione, protetto dallo stesso guasto e ripristinabile entro il tempo concordato come servizio utilizzabile. Il **backup** è la copia ripristinabile di uno stato precedente; il **disaster recovery** comprende inoltre persone, priorità, infrastruttura di destinazione, dipendenze, convalida e ritorno controllato in esercizio. NIST distingue quindi backup, strategia di ripristino, procedure di recovery, test e manutenzione continua del piano ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)).

Per le piattaforme di messaggistica, «il database» non è un oggetto di protezione completo. Uno stato utilizzabile può essere distribuito tra storage di mailbox o blob, metadati relazionali, code di trasporto, indici di ricerca, configurazioni, regole di routing, riferimenti alla directory, certificati, chiavi private, DNS, licenze e automazione. Alcune parti sono autorevoli, altre sono solo proiezioni e altre ancora stati transitori. Il piano di recovery deve specificare per ogni parte se viene **ripristinata, ricostruita, riemessa o deliberatamente scartata**. Oltre ai dati utente, NIST richiede anche stato del sistema, software, inventario, licenze e documentazione rilevante per la sicurezza; PostgreSQL, ad esempio, evidenzia espressamente che l'archiviazione WAL non include il backup dei suoi file di configurazione ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [PostgreSQL: archiviazione continua e PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Il criterio operativo di accettazione non è quindi «backup montato», ma ad esempio: un mittente esterno può consegnare un messaggio, viene applicata la policy corretta, il messaggio appare nella mailbox prevista, è ricercabile e può ricevere una risposta, mentre monitoraggio e audit registrano l'operazione. NIST cita come risultati di recovery misurabili restore riusciti e tempestivi, obiettivi di recovery raggiunti e utenti o sistemi nuovamente disponibili ([NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery), [NIST SP 800-184, metriche di recovery](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)).

L'analisi parte dal processo aziendale che deve tornare a funzionare dopo un guasto. Da questo derivano RTO e RPO, quindi la catena di backup adeguata; il risultato finale non è il job di backup, ma un test di restore misurato.

## Il backup non è alta disponibilità

Backup, snapshot, replica e alta disponibilità risolvono classi di errore diverse. Microsoft descrive esplicitamente backup e replica come complementari: la replica mantiene una copia aggiornata per l'esercizio corrente, ma replica anche eliminazioni logiche o corruzioni; un backup con marca temporale consente di tornare a uno stato precedente ([Microsoft Azure Reliability: ridondanza, replica e backup](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)).

| Meccanismo | Vantaggio principale | Cosa non dimostra da solo |
|---|---|---|
| Backup | stato storico e ripristinabile con retention | breve tempo di commutazione o piattaforma di destinazione immediatamente operativa |
| Snapshot di storage | stato point-in-time rapido di un volume | coerenza dell'applicazione, dominio di guasto separato o conservazione a lungo termine |
| Replica | stato dei dati aggiornato presso una seconda destinazione | protezione da eliminazione, cifratura o corruzione silenziosa replicate |
| Alta disponibilità | continuità del servizio in caso di guasti definiti dei componenti | ritorno storico o ricostruzione dopo compromissione amministrativa |
| Archivio | conservazione di dati selezionati per lunghi periodi | ricostruzione completa del servizio e delle sue dipendenze |
| Disaster recovery | ripristino coordinato dopo eventi relativi a sede, piattaforma o sicurezza | recupero dei dati senza backup adeguati e testati |

Uno snapshot può essere un componente del backup. Tuttavia, diventa una fonte di recovery affidabile solo grazie a coerenza, esportazione o replica verso uno storage gestito in modo indipendente, retention, catalogo e procedura di restore. L'API snapshot di Kubernetes, ad esempio, non garantisce di per sé la coerenza dell'applicazione; le applicazioni devono essere preparate adeguatamente prima dello snapshot. In Windows, VSS coordina questi aspetti solo se requester, writer e provider interagiscono correttamente ([Kubernetes: snapshot dei volumi e coerenza dell'applicazione](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/), [Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## MTD, RTO e RPO appartengono alle funzioni aziendali e di sistema

Il **Maximum Tolerable Downtime (MTD)** è l'interruzione più lunga tollerata dall'intero processo aziendale. Il **Recovery Time Objective (RTO)** descrive per quanto tempo una risorsa di sistema concreta può essere indisponibile prima di compromettere in modo inaccettabile altre risorse, il processo supportato o il suo MTD. Il **Recovery Point Objective (RPO)** indica il momento precedente all'evento fino al quale i dati devono essere ripristinati. In genere l'RTO deve essere più breve dell'MTD, poiché dopo il ripristino tecnico occorre ancora rielaborare i dati e verificare il servizio dal punto di vista operativo ([NIST SP 800-34 Rev. 1, sezione 3.2](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

Un unico «RTO della posta» nasconde differenze rilevanti. Una piattaforma può accettare SMTP anche se l'accesso utente o la ricerca non sono ancora disponibili. Un gateway può bufferizzare messaggi anche se il servizio mailbox a valle è indisponibile. Al contrario, un'interfaccia web può essere raggiungibile mentre mancano chiavi, lookup della directory o connettori in uscita. Gli obiettivi dovrebbero pertanto essere definiti per ogni **funzione aziendale e dipendenza**.

| Funzione | Stato da misurare | Tipica domanda RPO | L'RTO termina solo quando |
|---|---|---|---|
| accettazione esterna | MX, TLS, listener SMTP, policy e coda | Quali messaggi accettati possono mancare? | l'accettazione è controllata e l'elaborazione della coda è dimostrabile |
| consegna in uscita | routing, DNS, policy TLS, retry e DSN | Quali voci della coda possono andare perse? | la consegna riesce oppure viene ritardata in modo conforme agli standard |
| accesso alla mailbox | identità, metadati, blob e protocollo | Quale ultimo stato della mailbox è necessario? | funzionano autenticazione, lettura, scrittura e operazioni sulle cartelle |
| ricerca | indice e proiezioni | L'indice deve essere sottoposto a backup o rigenerato? | l'insieme di dati definito è nuovamente reperibile |
| cifratura | policy, certificati, chiavi e trust | Quali contenuti precedenti devono rimanere decifrabili? | un messaggio di test definito può essere cifrato e decifrato |
| amministrazione | control plane, ruoli, audit e monitoraggio | Quale modifica di configurazione può mancare? | modifiche autorizzate, allarmi e audit sono tracciabili |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-backup-dr.svg?v=20260813" title="Interaktive Infografik: Backup- und Disaster-Recovery-Kette für Messaging-Plattformen von produktiven Zuständen über konsistente, isolierte Sicherungen bis zur geprüften Wiederherstellung" loading="lazy">
  <a href="/images/kb-interaktiv-backup-dr.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

Una volta definiti gli obiettivi, occorre inventariare lo stato effettivo della piattaforma. Il solo database delle mailbox non ripristina né routing, né identità, né chiavi, né indici di ricerca.

## Inventario degli stati di una piattaforma di messaggistica

Una policy di backup inizia con un inventario di stati e dipendenze, non con il catalogo dei prodotti del produttore di backup. Per ogni stato vengono documentati **fonte autorevole, meccanismo di coerenza, RPO, retention, dominio di protezione, metodo di ripristino, ordine e fase di verifica**. La Business Impact Analysis di NIST identifica processi critici, risorse e relativa priorità di recovery; NIST SP 800-184 aggiunge scenari realistici e dipendenze individuate durante il ripristino ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final), [NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)).

| Stato | Carattere | Decisione di recovery | Fase di verifica funzionale |
|---|---|---|---|
| Mailbox e blob dei messaggi | dati utente autorevoli | ripristinare in modo coerente o ricostruire da una fonte immutabile | leggere un messaggio noto inclusi gli allegati MIME |
| Database dei metadati | transazioni, assegnazioni, ACL, UID | utilizzare backup di base più replay dei log o restore specifico dell'applicazione | cartelle, permessi e riferimenti ai messaggi corrispondono |
| Coda di trasporto | stato di consegna transitorio ma rilevante per l'attività | salvare, acquisire ordinatamente o ritrasmettere deliberatamente | nessuna lacuna silenziosa e gestione controllata dei duplicati |
| Indice di ricerca e proiezioni | perlopiù derivati e potenzialmente coerenti | salvare solo se la ricostruzione viola l'RTO; altrimenti reindicizzare | il campione definito è completamente reperibile |
| Configurazione e policy | dichiarativa, esportata o basata su database | salvare esportazione versionata più versione di schema/prodotto | routing, filtri, limiti e separazione dei tenant sono efficaci |
| Identità e riferimenti alla directory | spesso autorevoli esternamente | ripristinare separatamente la directory; mantenere binding, ID e claim | funzionano autenticazione di servizio e utenti |
| Certificati, chiavi e segreti | altamente sensibili, talvolta non esportabili | in base al tipo di chiave: salvare, riemettere o ricostruire tramite HSM/KMS | TLS, firma, decifrazione e rotazione verificati |
| DNS, ora, rete e load balancer | livello esterno di controllo e denominazione | documentare come codice/esportazione e presso il provider | nomi, porte, nomi dei certificati e ora corrispondono |
| Software, immagini, IaC e licenze | base di esecuzione riproducibile | mantenere artefatti affidabili, versioni e dipendenze | si avvia una build identica o compatibile approvata |
| Log, audit, catalogo di backup e runbook | evidenza e gestione | mantenere disponibili al di fuori del dominio amministrativo interessato | incidente, punto di restore e approvazioni sono tracciabili |

Il recovery delle code è un caso particolare. SMTP richiede di eseguire in modo affidabile la responsabilità accettata, ma in caso di interruzione della connessione ammette situazioni in cui mittente e destinatario valutano diversamente il completamento. Uno stato della coda ripristinato può quindi consegnare nuovamente i messaggi. I runbook richiedono una strategia definita per i duplicati, ID delle code, finestre temporali e comunicazione ai destinatari; la semplice copia di una directory spool non è un restore conforme agli standard ([RFC 5321, accodamento e messaggi duplicati](https://datatracker.ietf.org/doc/html/rfc5321)).

## La coerenza nasce al livello dell'applicazione

Un punto di backup **coerente con un crash** contiene lo stato che un sistema vedrebbe dopo un'improvvisa interruzione di corrente. Il file system e singoli blocchi possono essere coerenti internamente, mentre database, blob e code correlati rappresentano momenti diversi. Un punto di backup **coerente con l'applicazione** coordina buffer di scrittura, log delle transazioni, checkpoint ed eventualmente più volumi affinché l'applicazione disponga di un percorso di recovery definito.

VSS mostra esplicitamente questa architettura: il requester di backup richiede il salvataggio, il writer specifico dell'applicazione fornisce un set di dati coerente e il provider crea la Shadow Copy. Exchange fornisce a questo scopo un proprio VSS Writer; un backup compatibile con Exchange è quindi più di uno snapshot dei relativi file di database ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Microsoft: Windows Server Backup per Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)).

PostgreSQL utilizza un modello di recovery diverso ma comparabile. Un backup di base fornisce il punto di partenza, una sequenza senza lacune di segmenti Write-Ahead Log archiviati lo porta fino al momento desiderato. Un `pg_dump` è un'esportazione logica e non sostituisce la catena di backup di base/WAL richiesta per PITR. Anche file di configurazione quali `postgresql.conf` e `pg_hba.conf` sono esterni a questo recovery WAL e richiedono un percorso di backup separato ([PostgreSQL: archiviazione continua e PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Per i prodotti distribuiti, la documentazione del prodotto deve chiarire se i backend vengono salvati indipendentemente, come gruppo di coerenza o tramite funzioni di esportazione proprie dell'applicazione. Uno snapshot simultaneo dello storage di più volumi non costituisce automaticamente un taglio coerente di database, object storage, coda e indice di ricerca. L'amministratore deve conoscere la **fonte di verità** e il percorso di ricostruzione ammesso per ogni proiezione.

### Inventariare capacità e artefatti di backup

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kapazitäts- und Sicherungsinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-Volume | Sort-Object DriveLetter |
  Select-Object DriveLetter, FileSystemLabel, Size, SizeRemaining
Get-ChildItem \\backup.example.ch\mail -File -Recurse |
  Select-Object FullName, Length, LastWriteTime
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
df -hT
find /backup/mail -type f -printf '%TY-%Tm-%TdT%TH:%TM:%TS %s %p\n'
```

  </div>
</div>

[`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) e [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) mostrano la capacità occupata e libera. [`Get-ChildItem`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem) e [`find`](https://www.gnu.org/software/findutils/find) inventariano artefatti e marche temporali. Nessuno dei due dimostra né la coerenza dell'applicazione né la ripristinabilità; a tale scopo sono necessarie evidenze di catalogo, log e restore.

## Architettura di protezione: separata, isolata e verificabile

Una catena di backup affidabile comprende almeno quattro ruoli distinguibili:

1. **Acquisizione:** l'applicazione o la funzione di esportazione genera uno stato definito.
2. **Catalogo e manifest:** ID di backup, fonte, momento, versione software, log necessari, riferimenti alle chiavi e checksum rendono il set reperibile e verificabile.
3. **Storage di recovery:** copie versionate risiedono al di fuori del dominio di errore primario e, per quanto possibile, anche del dominio amministrativo.
4. **Control plane di recovery:** identità separate, runbook, infrastruttura di destinazione e approvazioni consentono il ripristino quando la produzione non è affidabile.

Il CIS Control 11 richiede dati di recovery protetti in modo equivalente, un'istanza isolata ad esempio offline, nel cloud o fuori sede, nonché test di restore regolari. Per gli scenari ransomware, CISA raccomanda backup offline o altrimenti isolati, cifrati e testati regolarmente, nonché immagini pulite e un ambiente di recovery separato ([CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

**Immutabilità** e **isolamento** non sono la stessa cosa. S3 Object Lock può proteggere specifiche versioni di oggetti nel modello WORM dall'eliminazione e dalla sovrascrittura durante una retention o un Legal Hold. Le modalità Governance e Compliance offrono diverse possibilità di aggiramento. Ciò protegge le versioni memorizzate, ma non dimostra né credenziali separate né un endpoint di restore pulito, una catena applicativa completa o chiavi di decifrazione raggiungibili ([Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

### Verificare checksum e manifest

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Prüfsummen von Sicherungsartefakten">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-backup-2026-08-08.tar.zst -Algorithm SHA256
Get-FileHash .\mail-backup-2026-08-08.manifest.json -Algorithm SHA256
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-backup-2026-08-08.tar.zst
sha256sum mail-backup-2026-08-08.manifest.json
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) rilevano modifiche a un artefatto se l'hash previsto proviene da una fonte affidabile. Un checksum non sostituisce l'autenticazione del manifest né una prova di ripristino. NIST cita hash crittografici e firme digitali quali meccanismi per proteggere l'integrità delle informazioni di backup ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

## Chiavi e segreti costituiscono un piano di recovery separato

«Salvare tutte le chiavi private» è errato tanto quanto «i certificati possono essere riemessi». È decisivo lo scopo d'uso:

- Una **chiave server TLS** persa può di norma essere sostituita con una nuova coppia di chiavi e un nuovo certificato; la commutazione deve comunque rientrare nell'RTO e occorre verificare le dipendenze correlate di trust o pinning.
- Una **chiave di decifrazione** per dati S/MIME, OpenPGP o di backup memorizzati deve rimanere disponibile finché il testo cifrato protetto deve rimanere leggibile.
- Secondo NIST, il backup di una **chiave privata di firma** non è in genere auspicabile, poiché il riutilizzo può compromettere il valore probatorio della firma; eccezioni motivate richiedono un recovery particolarmente sicuro e una rapida sostituzione.
- Una **chiave HSM/KMS non esportabile** richiede il percorso di ridondanza, backup o reprovisioning previsto dal sistema. L'esportazione di un certificato senza chiave privata non costituisce un backup della chiave.
- La **chiave che cifra il backup** non deve trovarsi esclusivamente all'interno del backup cifrato o del dominio di produzione compromesso.

NIST richiede una decisione in base al tipo di chiave, metadati associati, una policy di key recovery nonché controlli di riservatezza, integrità, disponibilità e audit per il materiale di recovery. Se si perde una chiave di decifrazione, il testo cifrato non può più essere riconvertito in testo in chiaro ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final), [NIST SP 800-57 Part 1 Rev. 5, Key Recovery](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)).

### Verificare ora e risoluzione dei nomi prima del restore

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeit- und DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
w32tm /query /status
Resolve-DnsName -Type MX example.ch
Resolve-DnsName backup.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
timedatectl status
dig +short MX example.ch
dig +short A backup.example.ch
```

  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) e [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) verificano la base temporale per certificati, Kerberos, log e punti di recovery. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano se i nomi MX, di servizio e di repository vengono risolti come previsto nella zona di recovery. La fonte [DNS](/kb/dns) e la relativa autorizzazione alle modifiche fanno anch'esse parte dell'inventario delle dipendenze.

Un backup coerente è solo metà del piano. Durante il restore, identità, DNS, database, coda, chiavi e applicazioni devono tornare in una sequenza motivata.

## Il ripristino operativo segue il grafo delle dipendenze

Un ordine fisso dei prodotti sarebbe inventato. L'ordine affidabile deriva dalla BIA, dall'inventario delle risorse e dalle dipendenze effettive. NIST richiede un elenco prioritario delle risorse di sistema e scenari di test realistici; NIST SP 800-184 richiede che le dipendenze scoperte durante il restore vengano riportate nella documentazione ([NIST SP 800-34 Rev. 1, Recovery Priorities](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [NIST SP 800-184, Recovery Execution](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). Per una tipica piattaforma di messaggistica ne deriva spesso la seguente catena, da convalidare localmente:

1. **Delimitare l'evento:** distinguere guasto e compromissione, preservare le evidenze, definire un punto di recovery noto e pulito e l'approvazione.
2. **Ripristinare il control plane di recovery:** mettere a disposizione identità amministrative separate, MFA, runbook, catalogo di backup e accesso alla decifrazione.
3. **Convalidare i servizi di base:** verificare nella zona di destinazione rete, routing, [DNS](/kb/dns), ora, [LDAP](/kb/ldap) o [Kerberos](/kb/kerberos), PKI/KMS e load balancer.
4. **Ripristinare la persistenza:** ricostruire object storage/mailbox, database e log delle transazioni necessari in un taglio coerente.
5. **Avviare applicazione e policy:** applicare immagini approvate, configurazione, segreti, connettori e ruoli; non consentire ancora un flusso di posta esterno non controllato.
6. **Attivare coda e routing in modo controllato:** valutare età, destinatari, stato dei retry e possibili duplicati; abilitare separatamente entrata e uscita.
7. **Ricreare le proiezioni:** generare indici di ricerca, cache e reporting dalle fonti autorevoli e monitorare il ritardo di ricostruzione.
8. **Accettare la transazione aziendale:** testare consegna, accesso alla mailbox, ricerca, [TLS](/kb/tls), cifratura, monitoraggio e audit rispetto ai criteri definiti.

Per un evento di sicurezza, «il sistema si avvia» non è espressamente sufficiente. NIST descrive la ricostituzione in uno stato noto e sicuro con parametri sicuri, patch, configurazione, software affidabile, backup noto e pulito e test completo. CIS formula lo stesso obiettivo come ripristino in uno «stato pre-incidente e affidabile» ([NIST SP 800-34 Rev. 1, CP-10](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)).

### Raggiungere endpoint di recovery e TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Recovery-Endpunkt- und TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection backup.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose https://backup.example.ch/health
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz backup.example.ch 443
openssl s_client -connect backup.example.ch:443 \
  -servername backup.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano il percorso TCP. [`curl`](https://curl.se/docs/manpage.html) e [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostrano rispettivamente il comportamento HTTP e TLS. Un endpoint Health raggiungibile dimostra soltanto il control plane, non la leggibilità di tutti i set di backup.

Il ripristino operativo è concluso soltanto quando un utente o un sistema remoto può effettivamente utilizzare il servizio. Un supporto di backup letto con successo non ne è una prova sufficiente.

## I test di restore misurano il servizio, non il supporto

Un test completo si svolge in un ambiente di destinazione isolato, con punto di partenza documentato, misurazione dei tempi e criteri di accettazione. Verifica almeno:

- che catalogo, credenziali, chiavi di decifrazione e artefatti siano raggiungibili senza la produzione;
- che sia possibile fornire un sistema di destinazione compatibile a partire da immagini affidabili;
- che backup di base, log delle transazioni, blob storage e configurazione producano lo stesso stato funzionale;
- che le code siano elaborate in modo controllato e i duplicati vengano rilevati;
- che identità, DNS, TLS, flusso di posta, accesso alla mailbox, ricerca e monitoraggio funzionino;
- che perdita di dati e tempo di ripristino operativo misurati rispettino RPO e RTO;
- che il servizio possa essere approvato come affidabile dopo eventi di sicurezza.

CIS Control 11.5 valuta un campione di backup ripristinati e poi effettivamente funzionanti. NIST SP 800-184 misura restore riusciti e tempestivi e richiede scenari realistici, riesame e miglioramento del piano ([CIS Control 11: Test Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [NIST SP 800-184](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). La frequenza dei test dipende da rischio, tasso di cambiamento e requisiti; un test completo annuale può essere integrato da campionamenti automatizzati più frequenti e restore relativi ai componenti, ma non può essere sostituito da statistiche positive dei job.

### Listener e stato dello storage dopo il restore

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Speicherprüfung nach Restore">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
df -hT
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) e [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) mostrano i listener locali e i processi associati. [`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) e [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) mostrano lo spazio libero. La verifica deve poi proseguire a livello di protocollo: un listener sulla porta 25 non equivale ancora a una transazione [SMTP](/kb/smtp) funzionante.

## Scenari di errore e ambito di recovery adeguato

Un guasto determina fino a che punto deve estendersi un ripristino. La tabella collega quindi l'evento osservato al più piccolo ambito di recovery sensato e all'errata supposizione che in quel caso ricorre più spesso.

| Evento | Rischio primario | Ambito di recovery adeguato | Errata supposizione frequente |
|---|---|---|---|
| eliminazione accidentale | danno logico limitato | ripristinare selettivamente oggetto, mailbox, policy o point-in-time | riportare indietro l'intera piattaforma e perdere dati corretti più recenti |
| singolo nodo o supporto dati | errore di infrastruttura locale | failover HA, replica o restore relativo al componente | confondere il failover con un backup storico |
| perdita di sede o provider | dominio di guasto fisico o amministrativo condiviso | zona/regione/sito alternativo più copie esterne e commutazione DNS/rete | considerare DR una copia dei dati senza capacità di destinazione raggiungibile |
| ransomware o compromissione amministrativa | dati, identità, software e backup non affidabili | control plane di recovery isolato, build pulita, punto di restore noto | continuare a usare un'identità compromessa per sbloccare tutti i backup |
| perdita della chiave | testo cifrato permanentemente illeggibile o identità inutilizzabile | recovery specifico per tipo di chiave, riemissione o procedura HSM/KMS | confondere un certificato pubblico con una chiave privata |
| modifica errata della configurazione | dati corretti, comportamento errato | annullare la configurazione versionata e convalidare selettivamente | scegliere come prima misura il restore di database o mailbox |

In caso di compromissione, il momento noto e pulito non deve essere per forza il punto di backup più recente. I backup più nuovi possono contenere lo stato dell'attaccante; quelli più vecchi possono introdurre vulnerabilità note o versioni software incompatibili. Il recovery unisce pertanto analisi forense, livello di patch, baseline di configurazione, rotazione delle chiavi e ripristino funzionale dei dati. CISA raccomanda tra l'altro «golden images» pulite, definizioni dell'infrastruttura mantenute offline e una zona di rete di recovery affinché i sistemi non vengano nuovamente infettati durante la ricostruzione ([CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

## Evoluzione tecnica

Il nastro magnetico fu introdotto all'inizio degli anni 1950 come mezzo veloce di memorizzazione dei dati per i computer e resta un supporto di backup grazie a costi, capacità e separabilità fisica ([IBM: nastro magnetico](https://www.ibm.com/history/magnetic-tape)). Le architetture di backup successive hanno separato sempre più il punto di backup logico dal supporto di destinazione: i database hanno combinato backup di base con log delle transazioni e Point-in-Time Recovery; i sistemi di storage hanno consentito snapshot rapidi; deduplicazione e object storage hanno modificato trasferimento e retention.

Per le applicazioni Windows in esecuzione, Microsoft ha introdotto con VSS un modello coordinato composto da requester, writer e provider; la tecnologia è comparsa con Windows XP e Windows Server 2003. Nelle piattaforme distribuite e containerizzate, le API di snapshot e orchestrazione sono state standardizzate senza risolvere automaticamente la coerenza dell'applicazione ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Kubernetes: Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)).

Il cyber-recovery ha nuovamente spostato l'attenzione. Versioning e copie offsite non sono sufficienti se identità altamente privilegiate possono eliminare tutte le destinazioni o se ritornano immagini compromesse. Istanze di recovery isolate, identità separate, versioni di oggetti immutabili, infrastruttura dichiarativa e zone di ripristino pulite completano i backup classici completi, incrementali e basati su log. Amazon S3 Object Lock è stato introdotto nel 2018 come protezione WORM per le versioni di oggetti; la funzione illustra questa transizione, ma non sostituisce tuttora né la coerenza dell'applicazione né i test di restore ([AWS: introduzione di S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/), [Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

## Checklist per amministratori

La pianificazione è affidabile solo quando obiettivi, copie, accessi e test sono documentati insieme. La checklist riassume queste dipendenze per la revisione e l'esercitazione di restore.

- [ ] Processi aziendali, MTD nonché RTO e RPO per ogni funzione di sistema sono approvati.
- [ ] Tutti gli stati autorevoli e derivati della piattaforma di messaggistica sono inventariati.
- [ ] Coerenza dell'applicazione, catena dei log e gruppi di coerenza sono documentati in modo specifico per il prodotto.
- [ ] Restore della coda, possibili duplicati e riabilitazione di entrata e uscita sono regolamentati.
- [ ] Configurazione, policy, DNS, certificati, chiavi, segreti, licenze e runbook rientrano nell'ambito.
- [ ] Almeno una copia di recovery è separata dalla produzione e dagli account amministrativi primari.
- [ ] Immutabilità, isolamento, cifratura e accesso alle chiavi sono valutati separatamente.
- [ ] Il control plane di recovery e la capacità di destinazione funzionano senza la produzione compromessa.
- [ ] L'ordine di ripristino operativo segue un grafo delle dipendenze mantenuto aggiornato.
- [ ] I test ripristinano un processo aziendale completo di posta e misurano RPO/RTO.
- [ ] Risultati, nuove dipendenze e deviazioni confluiscono nuovamente in runbook e architettura.

## Fonti

- [NIST – SP 800-34 Rev. 1, guida alla pianificazione di emergenza](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [NIST – SP 800-34 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)
- [PostgreSQL – archiviazione continua e Point-in-Time Recovery](https://www.postgresql.org/docs/17/continuous-archiving.html)
- [NIST – SP 800-184, guida al recovery dagli eventi di cybersecurity](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)
- [NIST – SP 800-184, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)
- [Microsoft Azure Reliability – ridondanza, replica e backup](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)
- [Kubernetes – snapshot dei volumi e coerenza dell'applicazione](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/)
- [Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Windows Server Backup per Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/powershell/module/storage/get-volume)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [Microsoft Learn – Get-ChildItem](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem)
- [GNU Findutils – find](https://www.gnu.org/software/findutils/find)
- [CIS – Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)
- [CISA – StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)
- [Amazon S3 – Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
- [Microsoft Learn – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [systemd – timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – pagina man di dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – pagina man di nc](https://man.openbsd.org/nc)
- [curl – pagina man della riga di comando](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [IBM – nastro magnetico](https://www.ibm.com/history/magnetic-tape)
- [Kubernetes – Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [AWS – introduzione di S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/)
