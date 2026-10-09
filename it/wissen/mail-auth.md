---
title: "Autenticazione e-mail: SPF, DKIM e DMARC"
blatt: "mail-auth"
description: "SPF, DKIM e DMARC inquadrati tecnicamente: identità, valutazione DNS, firme, alignment, policy, reporting, inoltri, confini di fiducia e gestione operativa per amministratori di messaggistica."
fakten:
  - label: Scopo di SPF
    wert: L’IP autorizza RFC5321.MailFrom o HELO
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-2
  - label: Pubblicazione SPF
    wert: un record TXT con v=spf1
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-3
  - label: Budget DNS SPF
    wert: al massimo 10 termini che attivano lookup
    href: https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4
  - label: Scopo di DKIM
    wert: firma di dominio su header selezionati e body
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3
  - label: Chiave DKIM
    wert: selector._domainkey.signing-domain
    href: https://datatracker.ietf.org/doc/html/rfc6376#section-3.6.2.1
  - label: Crittografia DKIM
    wert: RSA-SHA256 o Ed25519-SHA256
    href: https://datatracker.ietf.org/doc/html/rfc8463#section-3
  - label: Norma DMARC
    wert: RFC 9989; reporting in RFC 9990 e 9991
    href: https://datatracker.ietf.org/doc/html/rfc9989
  - label: Identità DMARC
    wert: una Author Domain da un unico campo From
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.2
  - label: Pass DMARC
    wert: SPF o DKIM ha esito positivo ed è aligned
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Alignment
    wert: relaxed per impostazione predefinita; strict opzionale
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.4
  - label: Policy
    wert: none · quarantine · reject
    href: https://datatracker.ietf.org/doc/html/rfc9989#section-4.7
  - label: Prova operativa
    wert: Authentication-Results più Aggregate Reports
    href: https://datatracker.ietf.org/doc/html/rfc8601
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 1247c1771c81b476bf23da2eeee6feb35a3d16c7146e2c61d6c0fa5625c55bbd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:29:46.083Z
translationReview: automatic
---

# Autenticazione e-mail: SPF, DKIM e DMARC

SPF, DKIM e DMARC rispondono a tre domande diverse sul dominio del mittente utilizzato. SPF verifica l’IP mittente, DKIM una firma crittografica e DMARC il rapporto tra entrambi i risultati e il dominio From visibile. Nessuno di questi meccanismi autentica una persona né dimostra che un messaggio sia innocuo. Un pass DMARC indica soltanto che l’uso dell’Author Domain è stato autorizzato secondo le regole pubblicate ([RFC 7208, sezione 1](https://datatracker.ietf.org/doc/html/rfc7208#section-1), [RFC 6376, sezione 1](https://datatracker.ietf.org/doc/html/rfc6376#section-1), [RFC 9989, sezione 1](https://datatracker.ietf.org/doc/html/rfc9989#section-1)).

Per gli amministratori di messaggistica, il sistema è soprattutto una catena di responsabilità. Il servizio in uscita deve generare domini Envelope appropriati e firme DKIM. Il [DNS](/kb/dns) autorevole deve fornire correttamente e tempestivamente policy SPF, chiavi DKIM e policy DMARC. Il server perimetrale [SMTP](/kb/smtp) in ricezione necessita dell’IP client originario, di un resolver, della verifica crittografica e di un confine di fiducia definito per i risultati. Un collector di report deve elaborare XML non attendibile in modo sicuro, riconoscere i duplicati e rendere i dati analizzabili nel tempo. Un `dmarc=pass` è soltanto il risultato finale di questa pipeline distribuita.

La spiegazione segue le identità di un’e-mail: mittente Envelope, dominio From visibile e firma DKIM. SPF e DKIM vengono dapprima illustrati separatamente; DMARC ne collega poi i risultati mediante alignment, policy e reporting.

## Identità e confini di fiducia

Un messaggio presenta più nozioni di mittente, che non sono intercambiabili. Il **RFC5321.MailFrom** è il Reverse Path dell’Envelope SMTP e il principale oggetto SPF. Con un Reverse Path vuoto, come previsto per i rapporti di consegna, SPF deriva l’identità da HELO/EHLO. Il **RFC5322.From** si trova nell’header del messaggio, viene visualizzato dal client e-mail e fornisce a DMARC l’Author Domain. Una firma DKIM indica con `d=` il proprio Signing Domain e con `s=` il selettore della chiave pubblica. SMTP AUTH, a sua volta, autentica un client presso un servizio di submission, ma non è né SPF, né DKIM, né DMARC ([RFC 5321, sezioni 3.3 e 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321#section-3.3), [RFC 5322, sezione 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2), [RFC 7208, sezioni 2.3 e 2.4](https://datatracker.ietf.org/doc/html/rfc7208#section-2.3), [RFC 6376, sezione 3.5](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

| Identità | Fonte | Verificatore | Affermazione primaria |
|---|---|---|---|
| indirizzo IP connesso | connessione TCP presso l’MTA ricevente | SPF | questo host è o non è autorizzato per il dominio SMTP verificato |
| dominio HELO/EHLO | dialogo SMTP | SPF | identità di dominio del client SMTP |
| dominio RFC5321.MailFrom | Envelope SMTP | SPF e DMARC | dominio Bounce o Return-Path |
| DKIM `d=` e `s=` | `DKIM-Signature` | DKIM e DMARC | Signing Domain e selettore della chiave |
| dominio RFC5322.From | header visibile del messaggio | DMARC | Author Domain con cui deve essere stabilito l’alignment |
| `authserv-id` | `Authentication-Results` | consumer interni | quale servizio di verifica attendibile ha generato il risultato |

Questa separazione è un confine di sicurezza. Un aggressore può superare completamente SPF e DKIM con un proprio dominio e menzionare comunque un marchio altrui nel nome visualizzato. DMARC limita l’uso non autorizzato del dominio nel `From:` visibile, non i domini look-alike, la frode del nome visualizzato, i mittenti legittimi compromessi o i contenuti dannosi ([RFC 7208, sezione 11.2](https://datatracker.ietf.org/doc/html/rfc7208#section-11.2), [RFC 9989, sezioni 2.2 e 11.4](https://datatracker.ietf.org/doc/html/rfc9989#section-11.4)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-mail-auth.svg?v=20260813" title="Interaktive Infografik: Identitäten, Alignment, DMARC-Entscheidung, Reporting und indirekte Nachrichtenflüsse" loading="lazy">
  <a href="/images/kb-interaktiv-mail-auth.svg?v=20260813">Apri l’infografica su SPF, DKIM e DMARC</a>
</iframe>

## SPF: autorizzazione dell’IP connesso

Il Sender Policy Framework è un’autorizzazione basata sul DNS per il dominio in `MAIL FROM` o HELO. Il destinatario valuta l’IP client, il dominio verificato, l’identità del mittente e il nome host locale con la funzione `check_host()` definita nell’RFC 7208. Il record si trova come risorsa TXT direttamente presso il dominio interessato e inizia con `v=spf1`; il tipo DNS RR SPF abbandonato non viene utilizzato ([RFC 7208, sezioni 3.1 e 4.1](https://datatracker.ietf.org/doc/html/rfc7208#section-3.1)).

Un record come `v=spf1 ip4:192.0.2.0/24 include:_spf.sender.example -all` viene valutato da sinistra a destra. Un meccanismo corrispondente termina l’elaborazione con il proprio qualifier. `+` significa `pass` ed è il valore predefinito, `-` significa `fail`, `~` significa `softfail`, `?` significa `neutral`. `all`, `ip4` e `ip6` non richiedono lookup DNS aggiuntivi durante la normale valutazione; `include`, `a`, `mx`, `ptr`, `exists` e `redirect` li richiedono ([RFC 7208, sezioni 4.6.1–4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6)).

| Risultato | Significato di protocollo | Domanda dell’amministratore |
|---|---|---|
| `pass` | l’IP è autorizzato per questa identità SPF | Questo stesso dominio è anche aligned con RFC5322.From? |
| `fail` | la policy pubblicata non autorizza l’IP | fonte errata, uso abusivo del dominio o policy obsoleta? |
| `softfail` | debole affermazione negativa del dominio | serve ancora una fase di transizione controllata o nasconde del drift? |
| `neutral` | nessuna affermazione di autorizzazione | manca un meccanismo finale o la neutralità è intenzionale? |
| `none` | nessuna policy SPF applicabile | è stato verificato il dominio MailFrom/HELO corretto? |
| `temperror` | errore temporaneo, tipicamente DNS | verificare resolver, timeout e raggiungibilità autorevole |
| `permerror` | record o valutazione permanentemente non validi | verificare sintassi, record SPF multipli, ricorsione e budget di lookup |

### Budget di lookup e dipendenze

SPF limita a dieci termini la somma di `include`, `a`, `mx`, `ptr`, `exists` e `redirect` nell’intera valutazione ricorsiva. Un superamento deve produrre `permerror`. Per `mx` e `ptr` vigono ulteriori limiti di indirizzi; anche più di due risposte vuote o risultati NXDOMAIN, i cosiddetti Void Lookups, dovrebbero portare a `permerror` ([RFC 7208, sezione 4.6.4](https://datatracker.ietf.org/doc/html/rfc7208#section-4.6.4)).

Il budget è un limite di runtime e non una mera verifica dei caratteri. Un singolo `include` può introdurre ulteriori include, risoluzioni MX e aree di guasto. `include` delega soltanto la domanda se l’host corrente ottenga lì un `pass`; `redirect` rileva, dopo una verifica dei meccanismi senza successo, l’intera policy di un altro dominio. L’RFC 7208 raccomanda `include` per oltrepassare confini amministrativi e `redirect` piuttosto per centralizzare domini gestiti in modo uniforme ([RFC 7208, sezioni 5.2 e 6.1](https://datatracker.ietf.org/doc/html/rfc7208#section-5.2)). Per l’esercizio, l’inventario deve quindi includere proprietario, scopo, percorso di modifica e budget di worst case misurato di ogni include esterno.

### Inoltro e dominio SPF

Un forwarder classico si connette al destinatario successivo con il proprio IP, ma conserva l’`MAIL FROM` originario. L’IP viene così verificato rispetto alla policy SPF di un dominio altrui e SPF può fallire, sebbene l’inoltro originario fosse legittimo. L’RFC 7208 descrive la riscrittura del Reverse Path su un dominio dell’intermediario come contromisura; le mailing list lo fanno spesso comunque ([RFC 7208, appendice D.2](https://datatracker.ietf.org/doc/html/rfc7208#appendix-D.2)). Ciò risolve il pass SPF per il nuovo dominio Envelope, ma genera un pass DMARC soltanto se il nuovo dominio è aligned con il `From:` visibile. Per i percorsi indiretti, una firma DKIM preservata e aligned è quindi particolarmente importante.

## DKIM: firma di un dominio

DomainKeys Identified Mail aggiunge un campo header `DKIM-Signature`. Il signer canonizza header selezionati e il body, calcola l’hash del body `bh=`, firma i dati definiti e pubblica la chiave in `s=._domainkey.d=`. Il verifier ricostruisce gli stessi dati, interroga la chiave DNS TXT e verifica firma e hash del body. Un pass dimostra che i componenti firmati non sono stati modificati in modo rilevabile dalla firma e che il signer controllava la chiave privata dello Signing Domain. DKIM non dimostra né una persona fisica né la veridicità del contenuto ([RFC 6376, sezioni 3.5–3.8](https://datatracker.ietf.org/doc/html/rfc6376#section-3.5)).

I tag importanti sono `a=` per l’algoritmo, `c=` per la canonizzazione di header/body, `d=` per lo Signing Domain, `s=` per il selettore, `h=` per i nomi degli header firmati, `bh=` per l’hash del body e `b=` per la firma. `t=` e `x=` possono trasportare l’istante della firma e quello di scadenza, ma non costituiscono una protezione affidabile contro il replay. L’opzionale `l=` limita l’area firmata del body e può quindi consentire l’aggiunta di contenuti non firmati; l’RFC 6376 descrive esplicitamente questa superficie di abuso ([RFC 6376, sezioni 3.5 e 8.2](https://datatracker.ietf.org/doc/html/rfc6376#section-8.2)).

La canonizzazione `simple` non tollera quasi alcuna modifica. `relaxed` normalizza tra l’altro determinate rappresentazioni e whitespace, affinché le comuni modifiche di trasporto non rompano inutilmente una firma. Header e body possono usare procedure diverse; senza `c=` vale `simple/simple`. La canonizzazione non modifica il messaggio trasmesso, bensì soltanto la sua forma di input per firma o verifica ([RFC 6376, sezione 3.4](https://datatracker.ietf.org/doc/html/rfc6376#section-3.4)).

### Chiavi, algoritmi e rotazione

L’RFC 8301 richiede almeno 1024 bit per RSA, raccomanda ai signer almeno 2048 bit e dichiara RSA-SHA1 storico. L’RFC 8463 aggiunge Ed25519-SHA256 e consente firme parallele con selettori differenti per compatibilità di transizione ([RFC 8301, sezioni 3.1 e 3.2](https://datatracker.ietf.org/doc/html/rfc8301#section-3), [RFC 8463, sezioni 5 e 6](https://datatracker.ietf.org/doc/html/rfc8463#section-5)). L’algoritmo impiegato resta una decisione di interoperabilità: un obbligo standardizzato lato verifier non dimostra automaticamente che ogni piattaforma ricevente reale lo implementi senza errori.

I selettori separano il cambio di chiave dal dominio. Per una rotazione si pubblica dapprima la nuova chiave pubblica, poi si firma con la nuova chiave privata e la vecchia chiave DNS viene rimossa soltanto quando i messaggi meno recenti non devono più essere verificati regolarmente. L’RFC 6376 sconsiglia di riutilizzare un selettore con una nuova chiave, poiché altrimenti non è possibile distinguere i vecchi errori di firma dalle falsificazioni. Un `p=` vuoto nel record della chiave revoca la chiave ([RFC 6376, sezioni 3.1 e 6.1.2](https://datatracker.ietf.org/doc/html/rfc6376#section-3.1)).

La chiave privata non appartiene né al DNS né a repository di configurazione generici. Il progetto operativo deve definire proprietario della chiave, generazione, archiviazione protetta, accesso del signer, rotazione, revoca d’emergenza, decisione di backup e traccia di audit. L’RFC 6376 richiede attenzione nella protezione delle chiavi private e cita l’archiviazione cifrata e hardware crittografico come possibili misure protettive. Più piattaforme di invio dovrebbero avere selettori separati, affinché una piattaforma compromessa possa essere isolata senza un cambio di chiave globale. Questa architettura segue la suddivisione amministrativa dello spazio dei nomi dei selettori prevista da DKIM ([RFC 6376, sezioni 3.1 e 8.3 nonché appendice C](https://datatracker.ietf.org/doc/html/rfc6376#section-8.3)).

SPF può fallire dopo un inoltro, anche se il messaggio è rimasto invariato. DKIM può invece sopravvivere a un inoltro, ma rompersi per un footer modificato. DMARC collega quindi entrambi i metodi tramite il cosiddetto alignment.

## DMARC: alignment, policy e valutazione

DMARC presuppone un singolo campo RFC5322.From correttamente formato ed estrae da esso esattamente un’**Author Domain**. Considera il dominio autenticato da SPF e tutti gli Signing Domain DKIM verificati con successo. Si ha un pass DMARC se almeno un risultato SPF o DKIM è `pass` e il suo dominio è aligned con l’Author Domain. Entrambi i meccanismi non devono quindi necessariamente superare la verifica contemporaneamente; in esercizio sono entrambi desiderabili perché possono fallire su flussi indiretti diversi ([RFC 9989, sezioni 4.2–4.4 e 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-4.2)).

Con **strict alignment** i domini devono essere identici. Con **relaxed alignment** devono avere lo stesso Organizational Domain. `adkim=s` rispettivamente `aspf=s` richiede strict; senza questi tag vale relaxed. L’Organizational Domain viene determinato secondo l’RFC 9989 tramite un DNS Tree Walk limitato e non più soltanto mediante una Public Suffix List. Il walk interroga al massimo otto livelli di nome e considera `psd=y` rispettivamente `psd=n` ([RFC 9989, sezioni 4.4 e 4.10](https://datatracker.ietf.org/doc/html/rfc9989#section-4.10)). Questa modifica è rilevante con verifier vecchi e nuovi misti; l’RFC 9989 stessa segnala possibili risultati di alignment diversi.

### Record di policy e tag

Il record DMARC si trova in `_dmarc.<domain>` e usa la sintassi tag/valore. `v=DMARC1` identifica il formato. `p=` descrive il trattamento desiderato per i messaggi non superati del dominio della policy: `none`, `quarantine` o `reject`. `sp=` può trattare diversamente i sottodomini esistenti, `np=` quelli inesistenti. `rua=` indica le destinazioni degli Aggregate Reports, `ruf=` le destinazioni opzionali dei Failure Reports. `t=y` contrassegna la modalità di test definita nell’RFC 9989. I tag sconosciuti vengono ignorati ([RFC 9989, sezioni 4.5–4.8](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)).

Il precedente rollout percentuale con `pct=` non fa più parte del protocollo secondo l’RFC 9989. L’esperienza operativa ha mostrato un’applicazione non uniforme; `t=` sostituisce soltanto il precedente effetto speciale di `pct=0`, non passaggi percentuali arbitrari. L’introduzione graduale deve quindi essere pianificata tramite Author Domain separate, policy per sottodomini, flussi e-mail mirati e una robusta valutazione dei report, non tramite un presunto regolatore percentuale normato ([RFC 9989, appendice A.6](https://datatracker.ietf.org/doc/html/rfc9989#appendix-A.6)).

Una policy è una preferenza pubblicata dal Domain Owner. Il destinatario può discostarsene per policy locali, reputazione, flussi indiretti o altre informazioni e documentare eventualmente gli override nei report. `p=reject` non rende automaticamente un messaggio non recapitabile in senso SMTP presso ogni destinatario; crea un’affermazione univoca e automatizzabile per la gestione di DMARC-Fail ([RFC 9989, sezioni 5.3 e 5.4](https://datatracker.ietf.org/doc/html/rfc9989#section-5.3)).

Una policy DMARC dovrebbe essere resa più severa soltanto quando tutti i percorsi di invio legittimi sono noti. Gli Aggregate Reports forniscono i dati per questo inventario e mostrano quali sistemi inviano effettivamente sotto un dominio.

## Reporting come pipeline di dati operativa

L’RFC 9990 separa l’Aggregate Reporting dalla specifica core DMARC. Un report riassume i messaggi per IP di origine, policy valutata, disposition, alignment e risultati di autenticazione. Il formato dati è XML; il file dovrebbe essere compresso con GZIP e porta quindi l’estensione `.xml.gz`. I singoli record contengono tra l’altro `source_ip`, `count`, `header_from`, risultati SPF e DKIM nonché possibili motivi di policy override ([RFC 9990, sezioni 3.1 e 3.4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.1)).

Un collector è quindi un servizio di ingest rilevante per la sicurezza. Riceve e-mail e allegati non richiesti, decomprime dati esterni, analizza XML, deduplica report e aggrega volumi. Limiti di dimensione, budget di decompressione, parser XML senza entità esterne, protezione antimalware, quarantena per file difettosi, idempotenza e conservazione fanno parte dell’architettura. L’RFC 9989 avverte di report intenzionalmente malformati e DoS contro destinazioni di reporting; l’RFC 9990 disciplina duplicati e destinazioni esterne ([RFC 9989, sezione 11.2](https://datatracker.ietf.org/doc/html/rfc9989#section-11.2), [RFC 9990, sezioni 3.5.4 e 4](https://datatracker.ietf.org/doc/html/rfc9990#section-3.5.4)).

Se `rua=` è esterno all’Organizational Domain, il destinatario del report deve autorizzare tale relazione tramite un record DNS TXT aggiuntivo. In questo modo un dominio non può inondare terzi arbitrari con report ([RFC 9990, sezione 4](https://datatracker.ietf.org/doc/html/rfc9990#section-4)). I Failure Reports secondo RFC 9991 possono rivelare informazioni di singoli messaggi. A causa dei rischi di protezione dei dati, molti operatori li limitano; gli Aggregate Reports sono il canale di visibilità raccomandato senza contenuto dell’utente finale ([RFC 9991, sezione 7](https://datatracker.ietf.org/doc/html/rfc9991#section-7)).

Per la valutazione amministrativa, le serie temporali sono più importanti di un singolo valore giornaliero: volume per IP di origine e Author Domain, SPF aligned, DKIM aligned, DMARC-Fail, `temperror`, `permerror`, selettori sconosciuti, policy override e copertura dei reporter. I report sono osservazioni di singoli destinatari e non una contabilità completa degli invii; i destinatari non sono obbligati a fornire ogni valutazione o disposition desiderata ([RFC 9989, sezioni 1 e 6](https://datatracker.ietf.org/doc/html/rfc9989#section-6)).

## Fidarsi correttamente di Authentication-Results

Il servizio di verifica ricevente può inserire i risultati SPF, DKIM e DMARC nel campo header `Authentication-Results`. `authserv-id` identifica il servizio o l’Administrative Management Domain che ha effettuato la verifica. Questo campo ha valore probatorio soltanto entro un confine di fiducia definito. Un mittente esterno può inserire autonomamente un convincente `Authentication-Results: ... dmarc=pass` ([RFC 8601, sezioni 1.5.4–1.6](https://datatracker.ietf.org/doc/html/rfc8601#section-1.5.4)).

Il Border MTA deve quindi rimuovere le istanze esterne o consentire soltanto produttori esplicitamente attendibili. I consumer interni necessitano di un elenco di valori `authserv-id` consentiti e devono considerare la posizione nella catena `Received` attendibile oppure un modello interno di provenienza dei metadati. L’RFC 8601 avverte esplicitamente di non attivare il campo header per decisioni di filtro senza un Border MTA verificato e conforme ([RFC 8601, sezioni 5 e 7.1](https://datatracker.ietf.org/doc/html/rfc8601#section-5)).

ARC secondo RFC 8617 può trasportare risultati di autenticazione e modifiche successive attraverso intermediari in una catena firmata. ARC è pubblicato come protocollo sperimentale e non fornisce una radice di fiducia globale: il destinatario finale decide ancora di quali ARC-Sealer fidarsi. Uno stato ARC-Chain valido è quindi contesto per la policy locale, non un sostituto di DMARC e non una decisione di accettazione automatica ([RFC 8617, sezioni 1 e 5](https://datatracker.ietf.org/doc/html/rfc8617#section-1)).

Fin qui si è trattata la consegna diretta. Inoltri, liste e gateway modificano tuttavia indirizzo IP, Envelope o contenuto, e quindi proprio gli input delle tre verifiche.

## Flussi indiretti e tipici punti di rottura

Inoltri, mailing list, gateway di sicurezza e sistemi di ticketing modificano parti diverse del messaggio. Un inoltro cambia l’IP connesso e può rompere SPF. Una mailing list può modificare Subject, List-Header, body footer o struttura MIME e quindi rompere DKIM. Può inoltre riscrivere il mittente Envelope e il `From:` visibile. L’RFC 7960 descrive questi problemi di interoperabilità e i rispettivi effetti collaterali; non esiste una correzione universale che lasci contemporaneamente invariati identità, funzione della lista e policy del destinatario esistente ([RFC 7960, sezioni 3 e 4](https://datatracker.ietf.org/doc/html/rfc7960#section-3)).

Per la ricerca guasti, gli amministratori devono pertanto confrontare lo stato **prima e dopo ogni mediatore**: IP client, HELO, MailFrom, RFC5322.From, firme DKIM presenti, `Authentication-Results`, nuovi campi `Received` e mutazioni del contenuto. Un risultato finale `dmarc=fail` da solo non indica quale hop abbia perso l’identità aligned.

## Struttura tecnica di una piattaforma produttiva

L’autenticazione e-mail non è una singola appliance, bensì un piano di controllo e dati distribuito:

| Componente | Stack tecnologico | Stato persistente | Area di guasto centrale |
|---|---|---|---|
| DNS autorevole | TXT-RRset, DNSSEC opzionale, modifica basata su zona o API | SPF, chiavi pubbliche DKIM, policy DMARC | record obsoleti, split horizon, TTL, delega errata |
| Signer in uscita | filtro MTA, libreria o gateway; RSA/Ed25519 e SHA-256 | chiavi private, configurazione del selettore, policy di firma | accesso alla chiave, `d=` errato, firma assente su flussi parziali |
| Verifier in entrata | Border MTA/filtro, resolver ricorsivo, libreria crittografica, policy engine | cache del resolver, confine di fiducia, override locali | IP client errato dopo proxy, timeout DNS, Auth-Results manipolati |
| Modulo policy DMARC | alignment, DNS Tree Walk, logica di dominio/policy | cache e versione della policy | vecchio modello RFC 7489, Organizational Domain errato |
| Generatore di report | telemetria MTA, aggregatore, XML/GZIP, invio SMTP | finestra temporale, record, ID report | perdita di dati, duplicati, copertura reporter incompleta |
| Collector di report | mailbox, decompressore, parser XML, database, dashboard | report grezzi, normalizzazione, serie temporali | attacco al parser, deduplicazione errata, conservazione incontrollata |

Linguaggio di programmazione e prodotto sono intercambiabili, gli oggetti di protocollo e i confini di fiducia no. Un MTA può implementare firma e verifica in C, Rust, Java, Go oppure tramite un processo filtro separato. Per l’esercizio contano le stesse domande: da dove proviene l’IP client dopo i load balancer? Quale resolver e quale cache vengono usati? Dove si trova la chiave privata? Quale processo può firmare? Quali header vengono rimossi prima del confine di fiducia? Come vengono correlati versione della policy, risposta DNS, Message-ID, Queue-ID e risultato? Gli RFC definiscono la semantica wire e di valutazione, non il modello di processo concreto.

### Processo controllato di introduzione e modifica

L’RFC 9989 descrive una sequenza robusta per i Domain Owner: pubblicare SPF aligned, configurare DKIM aligned, predisporre una mailbox per Aggregate Report, pubblicare inizialmente DMARC in Monitoring Mode con `p=none`, valutare i report, correggere le lacune e decidere sull’enforcement soltanto dopo ([RFC 9989, sezione 5.1](https://datatracker.ietf.org/doc/html/rfc9989#section-5.1)). Da ciò deriva per il Change Management un processo verificabile:

1. Inventariare tutti gli Author Domain, domini Envelope, nomi HELO, prodotti di invio, tenant, relay, forwarder e proprietari delle chiavi.
2. Dimostrare almeno un’identità aligned per ogni mailstream legittimo; SPF e DKIM insieme riducono la dipendenza da un singolo percorso indiretto.
3. Rendere produttiva l’accettazione e la valutazione sicura dei report prima di `rua=`.
4. Utilizzare `p=none` come fase di misurazione, senza confonderlo con un effetto protettivo.
5. Classificare le fonti sconosciute: legittime e configurate erroneamente, dismesse, abusive o falsate da un flusso indiretto.
6. Introdurre l’enforcement per ogni dominio controllabile, documentare le eccezioni e monitorarlo mediante report e telemetria di consegna.
7. Esercitare regolarmente la rotazione delle chiavi, il cambio di provider, il rollback DNS, il guasto dei report e un signer compromesso.

Un’approvazione non dovrebbe limitarsi ad accettare un record TXT sintatticamente valido. Richiede messaggi di test per ogni percorso di invio, la prova degli header presso il destinatario, dati di report da più domini destinatari, la misurazione del budget SPF, la verifica dei TTL dei selettori e un rollback che non lasci l’Author Domain privo di protezione o non recapitabile.

Da queste dipendenze deriva una sequenza diagnostica fissa: prima identificare i domini utilizzati, poi verificare DNS e firma, infine valutare alignment e policy DMARC.

## Strumenti diagnostici

Le seguenti query usano `example.ch` e il selettore di esempio `s2026a`. I nomi produttivi e i messaggi memorizzati localmente possono essere analizzati soltanto in ambienti autorizzati.

### Leggere SPF, DMARC e DKIM nel DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail-Authentifizierungs-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type TXT -DnsOnly
Resolve-DnsName _dmarc.example.ch -Type TXT -DnsOnly
Resolve-DnsName s2026a._domainkey.example.ch -Type TXT -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +noall +answer example.ch TXT
dig +noall +answer _dmarc.example.ch TXT
dig +noall +answer s2026a._domainkey.example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano i TXT-RRset, non la loro completa valutazione di protocollo. Più Character Strings di un singolo record TXT devono essere concatenati senza caratteri aggiuntivi; più record SPF o DMARC con lo stesso nome sono invece un errore ([RFC 7208, sezioni 3.2 e 4.5](https://datatracker.ietf.org/doc/html/rfc7208#section-3.2), [RFC 9989, sezione 4.5](https://datatracker.ietf.org/doc/html/rfc9989#section-4.5)). Per questioni di split horizon o DNSSEC, la vista autorevole e quella ricorsiva devono essere verificate separatamente.

### Trovare risultati attendibili nell’header del messaggio

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Header-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Content .\message.eml |
  Select-String -Pattern '^(Authentication-Results|DKIM-Signature|Received):' -Context 0,8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">grep -E -A8 '^(Authentication-Results|DKIM-Signature|Received):' message.eml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) e [`Select-String`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string), rispettivamente [`grep`](https://www.gnu.org/software/grep/manual/grep.html), aiutano nella prima ispezione. A causa del folding degli header e di campi multipli, la ricerca testuale non sostituisce un parser conforme agli RFC. Sono decisivi il `authserv-id` attendibile, la sua posizione rispetto al proprio confine `Received`, i domini effettivamente verificati e la distinzione tra risultato grezzo e alignment.

### Verificare strutturalmente un Aggregate Report

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DMARC-XML-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">[xml]$report = Get-Content .\report.xml -Raw
$report.SelectNodes("//*[local-name()='record']").Count
$report.SelectSingleNode("//*[local-name()='org_name']").InnerText</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">xmllint --noout report.xml
xmllint --xpath 'count(//*[local-name()="record"])' report.xml
xmllint --xpath 'string(//*[local-name()="org_name"])' report.xml</code></pre>
  </div>
</div>

[`Get-Content`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) qui carica un file XML locale già decompresso; [`xmllint`](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) verifica struttura e query XPath. I report sconosciuti non dovrebbero essere aperti in modo interattivo con strumenti desktop privilegiati. Questa singola verifica non dimostra né la conformità allo schema né l’elaborazione sicura in massa, il rilevamento dei duplicati o l’aggregazione corretta.

### Assegnare sistematicamente gli errori a un confine

| Osservazione | Causa probabile | Prossima prova affidabile |
|---|---|---|
| `spf=none` | identità errata o assenza di record SPF | MailFrom/HELO dal risultato SMTP o Auth e TXT sul nome esatto |
| `spf=permerror` | sintassi, record multipli, ricorsione o budget DNS | valutare l’intero albero SPF ricorsivo e i contatori lookup/Void |
| SPF supera la verifica, DMARC fallisce | dominio SPF non aligned | confrontare RFC5321.MailFrom con RFC5322.From nella modalità di alignment configurata |
| `dkim=fail (body hash did not verify)` | body modificato dopo la firma | confrontare hop di firma, normalizzazione MIME, footer/disclaimer e canonizzazione |
| `dkim=temperror` | interrogazione della chiave temporaneamente fallita | nome del selettore, risposta del resolver, timeout e raggiungibilità DNS autorevole |
| DKIM supera la verifica, DMARC fallisce | supera soltanto una firma non aligned | verificare singolarmente tutti i domini `d=` rispetto all’Author Domain |
| I destinatari segnalano risultati DMARC diversi | percorsi, cache DNS, modelli di verifier o mutazioni differenti | correlare lo stesso Message-ID/firma con un header completo e l’istante DNS per ciascuno |
| IP di origine sconosciuto negli Aggregate Reports | nuovo mittente legittimo, forwarder o abuso | determinare volume, esempio di header, associazione reverse/provider e service owner interno |
| `Authentication-Results` si contraddicono | più hop di verifica o campo esterno falsificato | usare soltanto risultati entro il confine di fiducia definito |
| `p=reject`, ma il messaggio viene consegnato | override locale presso il destinatario | verificare `disposition`, motivo dell’override e ulteriori risultati dei filtri |

## Storia tecnica

SPF nacque da varie proposte di autorizzazione del mittente SMTP e fu pubblicato nel 2006 come RFC 4408 sperimentale. L’RFC 7208 portò SPF sullo Standards Track nel 2014, rimosse il tipo DNS RR SPF separato e precisò tra l’altro limiti DNS e Void ([RFC 4408](https://datatracker.ietf.org/doc/html/rfc4408), [RFC 7208, appendice B](https://datatracker.ietf.org/doc/html/rfc7208#appendix-B)). La sua architettura rimase volutamente orientata alla connessione SMTP e al Reverse Path.

DKIM riunì le esperienze di DomainKeys e Identified Internet Mail. L’RFC 4871 standardizzò DKIM nel 2007; l’RFC 6376 lo sostituì nel 2011 e affinò il modello di firma, chiave e verifica. L’RFC 8301 aggiornò nel 2018 gli algoritmi e le lunghezze delle chiavi RSA, l’RFC 8463 aggiunse Ed25519-SHA256 ([RFC 4871](https://datatracker.ietf.org/doc/html/rfc4871), [RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376), [RFC 8301](https://datatracker.ietf.org/doc/html/rfc8301), [RFC 8463](https://datatracker.ietf.org/doc/html/rfc8463)).

DMARC fu inizialmente pubblicato nel 2015 con RFC 7489 come documento informativo. L’RFC 9989 sostituì nel 2026 RFC 7489 e l’estensione PSD RFC 9091 come specifica Standards Track; Aggregate Reporting e Failure Reporting furono al contempo separati in RFC 9990 e RFC 9991. Il cambiamento introdusse tra l’altro DNS Tree Walk, `np`, `psd` e `t`, e rimosse `pct` ([RFC 9989, appendice C](https://datatracker.ietf.org/doc/html/rfc9989#appendix-C), [RFC 9990](https://datatracker.ietf.org/doc/html/rfc9990), [RFC 9991](https://datatracker.ietf.org/doc/html/rfc9991)).

La trasmissione leggibile dalle macchine dei risultati di verifica si sviluppò da RFC 5451 attraverso RFC 7001 e RFC 7601 fino a RFC 8601. ARC fu pubblicato nel 2019 con RFC 8617 come tentativo sperimentale di inoltrare firmati i risultati di autenticazione dei flussi indiretti ([RFC 8601, sezione 6](https://datatracker.ietf.org/doc/html/rfc8601#section-6), [RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)). Questa storia spiega perché piattaforme reali possano mostrare contemporaneamente vecchi tag DMARC, Organizational Domain basati su PSL, diversi algoritmi DKIM e differenti modelli di fiducia per Auth-Results.

## Fonti

- [RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208) – identità SPF, valutazione dei record, risultati e limiti DNS.
- [RFC 6376 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc6376) – modello di firma, chiave e verifica DKIM.
- [RFC 9989 – Domain-Based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc9989) – protocollo core DMARC, alignment, policy, DNS Tree Walk e gestione operativa.
- [RFC 5321, sezioni 3.3 e 4.5.5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 5322, sezione 3.6.2](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 8301 – DKIM Cryptographic Algorithm and Key Usage Update](https://datatracker.ietf.org/doc/html/rfc8301) – SHA-256 e lunghezze delle chiavi RSA.
- [RFC 8463 – Ed25519-SHA256 for DKIM](https://datatracker.ietf.org/doc/html/rfc8463) – algoritmo aggiuntivo di firma e chiave.
- [RFC 9990 – DMARC Aggregate Reporting](https://datatracker.ietf.org/doc/html/rfc9990) – modello dati XML, trasporto, duplicati e destinazioni esterne per report.
- [RFC 9991 – DMARC Failure Reporting](https://datatracker.ietf.org/doc/html/rfc9991) – Failure Reports per messaggio e protezione dei dati.
- [RFC 8601 – Authentication-Results](https://datatracker.ietf.org/doc/html/rfc8601) – formato header, `authserv-id` e confine di fiducia.
- [RFC 8617 – Authenticated Received Chain](https://datatracker.ietf.org/doc/html/rfc8617) – catena ARC sperimentale per flussi indiretti.
- [RFC 7960, sezioni 3 e 4](https://datatracker.ietf.org/doc/html/rfc7960)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – query DNS in Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – query DNS in sistemi Unix.
- [Microsoft Learn – Get-Content](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-content) – analisi locale di file e header in Windows.
- [Microsoft Learn – Select-String](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/select-string) – analisi di pattern in PowerShell.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) – ricerca testuale in Linux e Unix.
- [libxml2 – xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) – verifica XML e XPath.
- [RFC 4408 – Sender Policy Framework, Experimental](https://datatracker.ietf.org/doc/html/rfc4408) – predecessore dell’RFC 7208.
- [RFC 4871 – DomainKeys Identified Mail Signatures](https://datatracker.ietf.org/doc/html/rfc4871) – predecessore dell’RFC 6376.
