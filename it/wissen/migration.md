---
title: "Migrazione: architettura di destinazione, coesistenza e cutover controllato"
blatt: "migration"
description: "Migrazione tecnica per amministratori di messaggistica e infrastruttura: architettura di partenza e di destinazione, dipendenze, mappatura di identità e dati, da rehost a refactor, coesistenza, sincronizzazione seed e delta, pilota, cutover, confini di responsabilità, rollback, accettazione e dismissione."
fakten:
  - label: Punto di partenza
    wert: transazione aziendale + architettura di partenza + stati + proprietari
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Obiettivo
    wert: architettura di destinazione misurabile con interfacce, sovranità dei dati, confini di fiducia e di errore
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Strategie
    wert: Retain · Retire · Rehost/Relocate · Repurchase · Replatform · Refactor
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Modelli
    wert: Cutover · Pilota/ondate · Coesistenza · sostituzione progressiva
    href: https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html
  - label: Identità
    wert: mappare separatamente ID sorgente, ID destinazione, login, indirizzi SMTP, gruppi, ruoli e ciclo di vita
    href: https://datatracker.ietf.org/doc/html/rfc7643
  - label: Flusso dei dati
    wert: Seed → sincronizzazione delta → Freeze/Final Delta → riconciliazione
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration
  - label: Coesistenza
    wert: rubrica · instradamento della posta · discovery · autenticazione · autorizzazioni · calendario · supporto
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Responsabilità della posta
    wert: 250 dopo End-of-Data significa accettazione da parte dell'hop SMTP che risponde
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Cutover DNS
    wert: il TTL limita la durata della cache; verificare separatamente cache negative e override locali
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Limite di rollback
    wert: definirlo esplicitamente prima della prima scrittura esclusiva nella destinazione
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Accettazione
    wert: completezza · integrità · autorizzazione · transazione · prestazioni · monitoraggio
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Dismissione
    wert: gestire in modo controllato routing, identità, chiavi, backup, log, contratti e supporti dati
    href: https://csrc.nist.gov/pubs/sp/800/88/r1/final
werbung:
  - stargate
  - newsletter
ctaThemen:
  - migration
  - messaging
  - exchange-hybrid
translationSourceHash: bdb2fea86e1f811e78f2f210afd26b082f808325322fb675a8ec446e47b38ac3
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:19:47.082Z
translationReview: automatic
---

# Migrazione: architettura di destinazione, coesistenza e cutover controllato

Una migrazione tecnica trasferisce processi aziendali, interfacce e stati da un'architettura di partenza a un'architettura di destinazione definita. Il cambio di prodotto visibile è soltanto una parte. Identità, indirizzi, autorizzazioni, code, dati, chiavi, DNS, certificati, client, automazioni, monitoraggio, backup e processi di supporto devono continuare a soddisfare le medesime garanzie funzionali oppure essere trasferiti consapevolmente a un nuovo contratto.

Per i sistemi di messaggistica, la garanzia più importante è la responsabilità controllata di ogni messaggio. Durante una migrazione, piattaforme vecchie e nuove possono conoscere contemporaneamente gli indirizzi, accettare connessioni e modificare dati. Senza una sovranità dei dati esplicita, si producono loop, rubriche in split-brain, consegne duplicate, autorizzazioni perse e un percorso di ritorno che esiste solo nella pianificazione. Una migrazione solida inizia quindi dall'architettura e dagli stati, non dalla data di commutazione.

La spiegazione parte dallo stato attuale comprovato e conduce, attraverso l'immagine di destinazione, le dipendenze e il trasferimento dei dati, alla coesistenza. Pilota, cutover, percorso di ritorno e accettazione seguono soltanto quando è chiaro quali dati e funzioni vengono spostati.

Una migrazione non è una procedura di copia, bensì il passaggio controllato da un'architettura di partenza funzionante a un'architettura di destinazione verificata. Prima si rilevano stati e dipendenze, poi si pianificano coesistenza, movimento dei dati, cutover e percorso di ritorno.

## Dall'architettura di partenza a quella di destinazione

Prima della scelta del prodotto, il servizio esistente viene documentato come grafo. Ogni nodo ha un ruolo; ogni arco ha iniziatore, destinazione, protocollo, identità, classe di dati, comportamento in caso di errore e proprietario. Un elenco CMDB di nomi di server non basta se rimane ignoto quale scanner richiama un hop SMTP, quale servizio espande gruppi o quale chiave decritta messaggi precedenti.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-migration.svg?v=20260813" title="Interaktive Infografik: Migration von Ausgangs- zu Zielarchitektur mit Strategie, Zustandsklassen, Koexistenz, Seed und Delta, Pilotwellen, Cutover, Rollbackgrenze, Abnahme und Stilllegung" loading="lazy">
  <a href="/images/kb-interaktiv-migration.svg?v=20260813">Aprire direttamente il grafico interattivo</a>.
</iframe>

La documentazione di partenza rileva almeno:

| Dominio | Stato attuale da rilevare | Tipico contratto nascosto |
|---|---|---|
| Instradamento posta | MX, listener relay/submission, connettori, smarthost, domini accettati, IP sorgente, policy TLS | Un partner inserisce in allowlist un vecchio IP o si aspetta un nome di certificato |
| Identità | Fonte della directory, ancoraggio dell'oggetto, login, gruppi, ruoli, account di servizio, provisioning | L'applicazione usa l'indirizzo e-mail come ID immutabile |
| Dati | Cassetta postale, archivi, journal, coda, quarantena, blob, indice, configurazione, audit | L'indice di ricerca è ricostruibile, il journal invece è soggetto a conservazione obbligatoria |
| Crittografia | Certificati TLS, chiavi S/MIME/OpenPGP, HSM/KMS, passphrase, recovery | Un vecchio messaggio cifrato richiede una chiave privata storica |
| Client e API | Outlook/IMAP/SMTP/API, agenti, dispositivi mobili, webhook, script | URL, cifrario, porta, header o campo API codificati rigidamente |
| Esercizio | Monitoraggio, backup, ripristino, patch, logging, supporto, licenze | L'allarme controlla solo lo stato del processo del vecchio sistema |
| Organizzazione | Proprietario, reperibilità, approvazione, protezione dei dati, conservazione, contratto | Un team esterno controlla DNS o firewall con lunghi tempi di preavviso |

L'architettura di destinazione descrive gli stessi campi come **contratto target**. Inoltre vengono indicati confini di fiducia, sovranità dei dati, aree di guasto, obiettivi di ripristino, scalabilità e stato finale pianificato del vecchio sistema. NIST SP 800-34 collega lo sviluppo del sistema al contingency planning e richiede di derivare strategie e test di recovery dai requisiti aziendali. Per le migrazioni ne consegue che architettura di destinazione ed esercizio di fallback vengono progettati insieme ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)).

## Strategia di migrazione: cosa viene effettivamente modificato?

Le «7 R» documentate da AWS distinguono Retire, Retain, Rehost, Relocate, Repurchase, Replatform e Refactor/Re-architect. Sono una tassonomia per la migrazione cloud, ma possono essere applicate come domanda precisa a ogni migrazione di piattaforma: la funzione rimane, viene terminata, spostata senza modifiche, sostituita da un prodotto, adattata a una nuova piattaforma o ricostruita funzionalmente? AWS rileva che il refactoring nelle grandi migrazioni è la strategia più complessa e che una modernizzazione separata può ridurre i rischi ([AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)).

| Strategia | Cosa rimane uguale? | Cosa cambia? | Esempio di messaggistica |
|---|---|---|---|
| Retain | Piattaforma e responsabilità | solo eccezione/scadenza documentata | L'archivio legacy resta in sola lettura fino alla fine della retention |
| Retire | Nulla di produttivo | Funzione e accessi vengono eliminati | Un endpoint POP3 inutilizzato viene disattivato |
| Rehost / Relocate | Applicazione e modello dati in larga misura | Posizione del compute/della virtualizzazione | Una VM MTA viene spostata su un'altra infrastruttura |
| Repurchase | Requisito aziendale | Prodotto, modello dati, esercizio e contratto | Un gateway appliance viene sostituito da un gateway SaaS |
| Replatform | Funzione aziendale e parti dell'architettura | Runtime, database o modello operativo | Il servizio di posta usa una nuova piattaforma database/container |
| Refactor | Solo l'intento aziendale | Interfacce, componenti e stati | Una pipeline di posta monolitica viene ricostruita orientata agli eventi |

La strategia viene scelta **per componente e classe di dati**. Un progetto può replatformare l'instradamento della posta, mantenere i dati di archivio, ritirare uno strumento di reporting e riacquistare l'interfaccia utente. Definire genericamente tutto come «Lift and Shift» nasconde i rischi effettivamente diversi.

## Cutover, ondate, coesistenza e sostituzione progressiva

Quattro modelli di commutazione sono tecnicamente diversi e devono essere derivati da capacità del prodotto documentate:

- **Cutover unico:** tutti i percorsi di scrittura cambiano in una finestra. Adatto quando la coesistenza non è tecnicamente possibile, lo stato è piccolo e l'interruzione è accettata. Il percorso di ritorno deve essere chiaro prima della prima scrittura nella destinazione.
- **Pilota e ondate:** una popolazione rappresentativa cambia per prima; ulteriori batch seguono dopo l'accettazione. I batch di migrazione Microsoft sono un esempio concreto di trasferimento graduale delle cassette postali ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).
- **Coesistenza:** vecchio e nuovo condividono temporaneamente namespace, rubrica, instradamento o popolazione di utenti. Exchange Hybrid documenta, tra l'altro, flusso di posta sicuro, namespace di dominio comune, rubrica e condivisione del calendario come funzioni di coesistenza ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).
- **Sostituzione progressiva:** un livello di instradamento o integrazione intercetta le funzioni e le sposta gradualmente verso la destinazione. Martin Fowler ha descritto questo modello nel 2004 come «Strangler Fig Application», per ridurre il rischio di un unico grande cutover di riscrittura ([Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Nessuno di questi modelli è sicuro in assoluto. La coesistenza riduce l'ampiezza del cutover, ma crea stati aggiuntivi, regole di instradamento e casi di supporto. Il dual write può aumentare la disponibilità, ma crea problemi di risoluzione dei conflitti e ordinamento. Un cutover unico è più facile da comprendere, ma aumenta il raggio del danno simultaneo. La decisione indica quindi evidenza del prodotto, durata consentita, sovranità dei dati e criteri di uscita.

Dopo aver scelto la strategia, per ogni classe di dati deve essere stabilito quale sistema può scrivere durante ogni fase. Due fonti della verità non coordinate generano conflitti che uno strumento di copia non può risolvere.

## Classi di stato e sovranità dei dati

Prima di ogni sincronizzazione si stabilisce quali dati sono **fonte autorevole**, **copia replicata**, **proiezione derivata** oppure **solo archivio storico**:

| Classe di stato | Autorità durante la coesistenza | Questione di conflitto/recovery |
|---|---|---|
| Dati master dell'identità | Esattamente una directory principale o sistema HR | Quale ancoraggio dell'oggetto sopravvive a rinomina e cambio tenant? |
| Indirizzi SMTP e oggetti di routing | Una fonte di indirizzi approvata | Come vengono evitati duplicati e regole catch-all/alias? |
| Contenuto della cassetta postale | Per ogni cassetta migrata, origine o destinazione in modo univoco | Dopo il cutover è ancora consentito scrivere nella cassetta sorgente? |
| Coda | L'MTA che accetta di volta in volta ne porta la responsabilità | Come vengono svuotate le vecchie code senza reimmettere due volte i messaggi? |
| Policy e configurazione | Target dichiarativo o master designato | Quali regole sono semanticamente equivalenti, non soltanto copiate sintatticamente? |
| Chiavi e certificati | KMS/HSM o key owner definito | Quali chiavi private storiche devono rimanere leggibili? |
| Audit, journal e retention | Fonte di conservazione immutabile | Sono dimostrabili tempo, completezza, legal hold ed esportazione? |
| Indice/cache | Proiezione ricostruibile | Quanto dura il rebuild e quando la ricerca è considerata completa? |

«Sincronizzato bidirezionalmente» senza una regola di conflitto non è un modello dati. Per ogni campo vengono documentati direzione, frequenza, trasformazione, deposito degli errori e comportamento in caso di modifica simultanea. Se lo strumento non può unire le modifiche senza perdita, un lato viene reso in sola lettura durante la fase pertinente oppure viene modificato soltanto tramite un percorso di scrittura centrale.

## Identità, indirizzi e autorizzazioni

Nome utente, indirizzo e-mail e ID tecnico dell'oggetto sono cose diverse. RFC 7643 definisce per SCIM, tra l'altro, `id` come identificatore assegnato dal service provider, immutabile e univoco all'interno del tenant; `externalId` può contenere una correlazione gestita dal client. Ciò illustra una regola centrale di migrazione: un indirizzo modificabile non è un ancoraggio sicuro dell'oggetto ([RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643)).

Una mappatura dell'identità contiene almeno ID sorgente, ID destinazione, `externalId` o altro ancoraggio di correlazione, login/UPN, indirizzo SMTP primario, alias, gruppi, ruoli, attributi di manager/organizzazione, account di servizio, metodo di autenticazione e stato di provisioning. RFC 7644 descrive le operazioni SCIM per creazione, modifica, ricerca ed elaborazione bulk; la piattaforma di destinazione concreta può comunque avere campi obbligatori, regole di univocità e limiti di velocità diversi ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)).

Le autorizzazioni vengono migrate come relazioni, non come elenco di testo. Delega, Send As, Send on Behalf, ACL delle cartelle, Shared Mailbox, prenotazione di risorse, Application Access e ruolo amministrativo possono fare riferimento a tipi di oggetto differenti. Ogni relazione richiede un ancoraggio di destinazione risolto; i riferimenti orfani o ciclici vengono segnalati prima dell'ondata. Microsoft documenta, per esempio, che la migrazione di cassette postali cross-tenant richiede oggetti `MailUser` di destinazione e determinati attributi, e trasferisce solo contenuti/autorizzazioni definiti ([Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)).

## Percorso dati: seed, delta, freeze e riconciliazione

Gli stati di grandi dimensioni vengono in genere precaricati prima del cutover. Il processo solido è:

1. **Seed:** inventario iniziale completo con conteggio oggetti, volume in byte, elenco errori e intervallo temporale.
2. **Sincronizzazione delta:** trasferire ripetutamente le modifiche dal seed; osservare lag e tasso di errore.
3. **Freeze o reindirizzamento della scrittura:** arrestare percorsi di scrittura della sorgente definiti con precisione o indirizzarli alla destinazione.
4. **Final Delta:** recuperare le modifiche rimanenti e salvaguardare un livello dell'acqua riproducibile.
5. **Cutover:** commutare instradamento, discovery e accesso utente alla destinazione.
6. **Riconciliazione:** confrontare numero, dimensione, attributi chiave, autorizzazioni, campioni e transazioni aziendali.

La migrazione della posta elettronica a fasi di Microsoft documenta batch, stato di sincronizzazione e la necessità di attendere almeno un'ulteriore sincronizzazione dopo la commutazione dell'instradamento prima di eliminare un batch. Questa è un'evidenza specifica del prodotto per le fasi seed/delta/finalizzazione, non un comando universale per altri sistemi ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).

I checksum dimostrano l'uguaglianza dei byte solo con serializzazione identica. L'esportazione/importazione può normalizzare i messaggi, trasformare metadati o strutturare diversamente i contenitori. L'accettazione combina pertanto somme tecniche con invarianti aziendali: conteggio oggetti per classe, timestamp più vecchio/più recente, dimensione totale, oggetti mancanti/più grandi, relazioni di autorizzazione, campione casuale e casi volutamente difficili come messaggi MIME di grandi dimensioni, ricorrenze del calendario, Unicode, crittografia e legal hold.

Una volta definite identità e sovranità dei dati, il traffico dei messaggi in corso può essere instradato tra sistema vecchio e sistema di destinazione. La coesistenza è un'architettura di produzione temporanea con propri casi di routing e di errore.

## Architettura di coesistenza per la messaggistica

Un dominio SMTP comune su due piattaforme richiede una decisione univoca su destinatario e instradamento. Microsoft descrive per l'esercizio ibrido un namespace di dominio comune, flusso di posta sicuro, rubrica comune e funzioni di calendario; per la coesistenza con fornitori terzi vengono evidenziate esplicitamente le decisioni di identità, sincronizzazione e autenticazione ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid), [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)).

La matrice di coesistenza locale risponde a:

- Quale piattaforma conosce ogni destinatario e con quale tipo?
- Chi è responsabile dei destinatari sconosciuti e come vengono impediti i loop?
- Quale hop firma DKIM, applica DLP/archivio/disclaimer o cifra?
- Come riconoscono vecchio e nuovo l'altro come hop interno, autenticato o esterno?
- Dove risiedono Free/Busy, delega, espansione dei gruppi e Autodiscover/Service Discovery?
- Quale piattaforma scrive quali attributi di identità e indirizzo?
- Come vengono correlati bounce, quarantena, journal e message trace sui due lati?
- Quale punto di supporto ha una visione end-to-end?

Ogni transizione viene resa osservabile con risposta [SMTP](/kb/smtp)-risposta, ID coda e ID trace/messaggio. RFC 5321 stabilisce che il server ricevente assume la responsabilità dopo il completamento corretto di DATA. Un rollback dopo una risposta `250` non deve dimenticare questo messaggio accettato né reimmetterlo in modo incontrollato ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

## DNS, certificati e contratti esterni

Il DNS è un meccanismo di selezione distribuito, non un interruttore globale. RFC 1034 descrive il caching come componente fondamentale e lascia alla fonte dei dati il compromesso TTL tra attualità e impegno. Prima di una commutazione, i TTL rilevanti vengono pertanto ridotti per tempo; dopo il cutover, i percorsi vecchi restano controllati almeno finché resolver, cache negative, file hosts locali, proxy e gateway esterni possono utilizzare dati vecchi ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).

Vanno verificati MX, A/AAAA, CNAME, SRV e TXT, nonché viste interne/esterne. SPF deve autorizzare nuovi IP mittenti senza rimuovere troppo presto le fonti precedenti; selector DKIM e allineamento DMARC vengono verificati con messaggi reali. I certificati richiedono nomi, catena, trust, chiave privata, Extended Key Usage e capacità di esportazione/HSM. Le chiavi di decrittazione storiche S/MIME/OpenPGP seguono il proprio ciclo di vita. NIST SP 800-57 tratta stati delle chiavi, protezione e archiviazione; un cambio di piattaforma non annulla tali requisiti ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

I contratti esterni comprendono allowlist dei partner, credenziali smarthost, URI di redirect API, webhook, fonti SIEM, agenti di backup, probe di monitoraggio, firewall, NAT, DNS inverso e accessi di supporto. Ogni contratto possiede un owner tecnico e un test prima del cutover.

## Pilota, ondate e gate decisionali

Un pilota è rappresentativo, non scelto solo perché favorevole. Include cassette postali grandi e piccole, deleghe, Shared/Resource Mailbox, client mobili e meno recenti ma supportati, partner esterni, crittografia, archivi, power user, sedi diverse e almeno un caso negativo controllato. La popolazione pilota non deve al contempo contenere una funzione esclusiva irreversibile.

Ogni ondata ha dei gate:

- **Ready:** mappatura completa, oggetti di destinazione presenti, seed entro l'obiettivo di lag, client/rete/policy pronti, ripristino e supporto coperti.
- **Go:** ultima verifica delta e di integrità riuscita, nessuna classe bloccante aperta, dipendenze e finestra temporale confermate.
- **Hold:** tasso di errore, lag, coda, replica o carico di supporto fuori soglia; non avviare altri oggetti.
- **Abort:** integrità dei dati, sicurezza, perdita di posta non controllabile o altri criteri rossi definiti in precedenza.
- **Accept:** accettazione aziendale e periodo di osservazione successiva soddisfatti; l'oggetto appartiene ora in modo univoco alla destinazione.

Le ondate vengono suddivise in modo da limitare un guasto: per esempio in base a sede, tenant, dominio, database, tipo di client, gruppo funzionale o dipendenza. Le quantità casuali possono essere rappresentative, ma sono difficili da ricondurre operativamente. Ogni ondata riceve un elenco immutabile di oggetti e un set di dati dei risultati.

Pilota e ondate forniscono le evidenze per il cutover effettivo. Il ritorno rimane realistico solo finché le modifiche di stato possono ancora essere riportate completamente nel vecchio sistema.

## Cutover e limite di rollback

Il runbook di cutover non contiene soltanto passaggi, ma **precondizioni, owner, riscontro atteso, durata massima, criterio di interruzione e percorso di ritorno**. Una possibile sequenza è:

1. Congelare le modifiche, aprire il canale incident/comunicazione e salvaguardare le metriche iniziali.
2. Verificare Final Delta ed errori di mappatura; bloccare i percorsi di scrittura della sorgente secondo il modello.
3. Testare nuovamente listener di destinazione, certificati, identità, policy e dipendenze downstream.
4. Commutare i controlli di routing/discovery interni, poi i percorsi DNS/partner esterni.
5. Eseguire transazioni controllate inbound, outbound, submission, access, policy e crittografia.
6. Osservare coda, codici SMTP, autenticazione, replica, indice, errori client e monitoraggio.
7. Decidere Go/Hold/Abort e documentare l'ora.

Il **Rollback Point** si colloca prima del momento in cui la destinazione assume l'esclusiva responsabilità di scrittura non riconducibile senza perdita. Dopo questo punto, una commutazione cieca alla sorgente non è un rollback, ma una seconda migrazione. Nuovi messaggi nella destinazione, modifiche di calendario, autorizzazioni, chiavi e stati delle code devono allora essere risincronizzati, esportati o preservati tramite roll-forward. Il modello di [Backup e DR](/kb/backup-dr) definisce punto di consistenza, RPO/RTO e sequenza di ripristino.

## Strumenti amministrativi per evidenze prima/dopo

Gli esempi verificano singoli contratti. Vengono eseguiti dall'origine effettivamente interessata e con dati di test approvati. Credenziali, contenuti personali e chiavi private non devono essere inseriti senza protezione in ticket o cronologia della shell.

### Confrontare il DNS prima e dopo il cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName autodiscover.example.ch -Type CNAME -DnsOnly -Server 192.0.2.53
Resolve-DnsName example.ch -Type TXT -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer example.ch MX
dig @192.0.2.53 +noall +answer autodiscover.example.ch CNAME
dig @192.0.2.53 +noall +answer example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) rendono espliciti resolver e tipo di record. I resolver interni ed esterni vengono verificati separatamente; una risposta corretta di un resolver pubblico non dimostra che il servizio produttivo utilizzi la stessa vista.

### Raggiungere nuovi endpoint dall'origine reale

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection new-mx.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection new-api.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 new-mx.example.ch 25
nc -vz -w 5 new-api.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) dimostrano solo la raggiungibilità DNS/TCP dall'origine selezionata. Firewall, NAT, IP sorgente, TLS, autenticazione e protocollo applicativo vengono verificati separatamente in seguito.

### Verificare identità TLS e risposta del protocollo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) e [`curl`](https://curl.se/docs/manpage.html) verificano TLS e un dialogo controllato. Entrambi utilizzano la propria configurazione di trust/proxy/build; il client produttivo deve essere testato ulteriormente con la propria policy effettiva.

### Esportare e confrontare la mappatura delle identità

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Identitätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -Filter * -Properties ObjectGUID,UserPrincipalName,mail,proxyAddresses,Enabled |
  Select-Object ObjectGUID,SamAccountName,UserPrincipalName,mail,proxyAddresses,Enabled |
  Export-Csv .\\identity-source.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'dc=example,dc=ch' '(objectClass=person)' \
  entryUUID uid userPrincipalName mail proxyAddresses</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) e [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) forniscono attributi di identità dalle rispettive directory. Esportazione e query devono tenere conto di scope, paging, oggetti eliminati/disabilitati, attributi multivalore e diritti di accesso. Il confronto con la destinazione utilizza l'ancoraggio dell'oggetto concordato, non solo il nome visualizzato.

### Verificare gli artefatti di esportazione byte per byte

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Hashprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\export\\mailbox-001.pst -Algorithm SHA256
Get-FileHash .\\export\\mapping.csv -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./export/mailbox-001.mbox
sha256sum ./export/mapping.csv</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) e [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) rilevano modifiche a file identici. Non dimostrano che la cassetta postale sorgente e quella di destinazione siano semanticamente uguali; a questo scopo sono necessari conteggio degli oggetti, attributi, elenchi degli errori e campioni aziendali.

### Salvaguardare i log nella finestra temporale di cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T20:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 20:00:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) e [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) forniscono eventi di sistema con riferimento temporale. Si aggiungono log di prodotto, batch di migrazione, SMTP, directory, database e API con ID coda/oggetto/trace. Prima di riavvio o rotazione, le evidenze volatili vengono salvaguardate.

## Accettazione e periodo successivo

L'accettazione non verifica soltanto che «l'utente può accedere», ma sei dimensioni:

1. **Completezza:** oggetti previsti, intervalli temporali, cartelle, attributi, regole, chiavi e autorizzazioni presenti.
2. **Integrità:** nessuna modifica non consentita, duplicato, relazione persa o sovranità dei dati incoerente.
3. **Autorizzazione:** utenti, servizi e amministratori possono eseguire esattamente le operazioni previste, compresi i test negativi.
4. **Transazione:** inbound/outbound, submission/access, routing, policy, crittografia, quarantena, archivio e API funzionano end-to-end.
5. **Esercizio:** prestazioni, coda, replica, indice, backup, ripristino, patch, supporto e capacità soddisfano i valori target.
6. **Osservabilità:** log, metriche, trace, message tracking, allarmi e runbook vedono il percorso di destinazione.

La fase successiva dura oltre i cicli rilevanti di TTL, cache, retry, delta, retention e carico aziendale. Le piattaforme precedenti rimangono attive soltanto nel ruolo definito in precedenza: per esempio drenare una coda o leggere dati storici, non accettare inosservatamente nuovi dati. Forwarder temporanei e domini di coesistenza ricevono una data di scadenza e monitoraggio.

La dismissione inizia solo dopo l'accettazione aziendale e il periodo successivo. Log, chiavi, record DNS, account di servizio e integrazioni nascoste devono essere rimossi o archiviati in modo controllato.

## Dismissione senza dipendenze perse

Il decommissioning è una modifica di stato controllata a sé stante. Prima dello spegnimento vengono verificati ultimi accessi, code residue, sessioni attive, job, hit DNS/firewall, account di servizio, backup, obblighi di ripristino, legal hold, licenze e contratti di supporto. Quindi vengono rimossi routing e discovery, bloccati gli accessi, revocati o archiviati segreti e certificati, adeguato il monitoraggio e aggiornata la CMDB.

Supporti dati e risorse cloud vengono eliminati o resi crittograficamente inutilizzabili in base alla classe di dati e al contratto. NIST SP 800-88 descrive Clear, Purge e Destroy come metodi di sanitizzazione e richiede una decisione adeguata al supporto e al rischio ([NIST SP 800-88 Rev. 1](https://csrc.nist.gov/pubs/sp/800/88/r1/final)). Backup e chiavi possono vivere più a lungo dell'istanza di prodotto; la loro conservazione, leggibilità e distruzione definitiva rimangono documentate.

## Evoluzione tecnica delle migrazioni

Le prime migrazioni infrastrutturali erano spesso una sostituzione una tantum di hardware o software con una grande finestra di commutazione. Virtualizzazione e copie di immagini hanno semplificato Rehost e Relocate; protocolli standardizzati per directory, API e provisioning hanno consentito una coesistenza più fine. Le piattaforme cloud e SaaS hanno reso confini di tenant, identità e contratti oggetti di migrazione tanto importanti quanto server e database. La tassonomia delle 7 R descrive questa diversa profondità di intervento; il modello Strangler Fig descrive la sostituzione funzionale progressiva ([AWS – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Il progresso tecnico non elimina il cutover. Lo sposta su unità più piccole: singole identità, domini, API, classi di dati o decisioni di instradamento. Ciò riduce il raggio del danno simultaneo, mentre coesistenza e sovranità dei dati diventano più impegnative. L'architettura di migrazione moderna ottimizza quindi non solo la velocità di copia, ma **l'univocità della responsabilità, l'osservabilità di ogni transizione e un percorso reale di ritorno o avanzamento**.

## Fonti

- [NIST – SP 800-34 Rev. 1, Contingency Planning](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)
- [Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)
- [Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)
- [IETF RFC 7643 – SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643)
- [IETF RFC 7644 – SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
- [Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)
- [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [NIST – SP 800-57 Part 1 Rev. 5, Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [NIST – SP 800-88 Rev. 1, Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
