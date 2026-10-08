---
title: "LDAP: protocollo, modello dati e gestione delle directory"
blatt: "ldap"
description: "LDAP per amministratori: stack di protocollo e formato wire BER, DIT, Distinguished Names, schema, Bind e SASL, ricerca e controlli, TLS, Active Directory, Global Catalog, limiti di replica, scalabilità e diagnostica."
fakten:
  - label: Nome
    wert: Lightweight Directory Access Protocol
    href: https://datatracker.ietf.org/doc/html/rfc4510
  - label: Versione del protocollo
    wert: LDAPv3 · RFC 4510 fino a 4519
    href: https://datatracker.ietf.org/doc/html/rfc4510#section-1
  - label: Formato wire
    wert: Strutture ASN.1, codificate BER
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5.1
  - label: Trasporto
    wert: TCP; opzionalmente TLS e SASL sopra di esso
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-5
  - label: Porte
    wert: 389 LDAP · 636 LDAP su TLS
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap
  - label: Modello dati
    wert: DIT di Entries e attributi denominati
    href: https://datatracker.ietf.org/doc/html/rfc4512#section-2
  - label: Nomi
    wert: DN da RDN ordinati
    href: https://datatracker.ietf.org/doc/html/rfc4514
  - label: Filtri
    wert: Sintassi prefissa secondo RFC 4515
    href: https://datatracker.ietf.org/doc/html/rfc4515
  - label: Bind
    wert: anonymous, simple o SASL
    href: https://datatracker.ietf.org/doc/html/rfc4513#section-5
  - label: StartTLS
    wert: Extended Operation su una sessione esistente
    href: https://datatracker.ietf.org/doc/html/rfc4511#section-4.14
  - label: Paginazione
    wert: Simple Paged Results Control
    href: https://datatracker.ietf.org/doc/html/rfc2696
  - label: AD Global Catalog
    wert: 3268 LDAP · 3269 LDAP su TLS
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
  - seppmail
translationSourceHash: 7b219213aa84d6ce78de262f2cd3a5ba23a220cc1669e134bceb3a325b06de33
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:22:35.656Z
translationReview: required
---

# LDAP: protocollo, modello dati e gestione delle directory

LDAP è il linguaggio comune con cui le applicazioni accedono ai servizi di directory. Un client può usarlo per cercare entry denominate, leggere o modificare attributi e autenticarsi presso la directory. Il protocollo definisce messaggi, operazioni e codici di errore. Il modo in cui un server memorizza i dati, li replica o li protegge dai guasti resta invece compito della rispettiva implementazione. Active Directory Domain Services, OpenLDAP e 389 Directory Server parlano quindi LDAP senza essere internamente la stessa piattaforma ([RFC 4510, sezioni 1 e 2](https://datatracker.ietf.org/doc/html/rfc4510#section-1), [RFC 4511, sezione 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3)).

Questa separazione è decisiva nell'esercizio della messaggistica. Un gateway può verificare prima dell'accettazione SMTP se un destinatario esiste, risolvere gruppi per una policy o autenticare un amministratore. Se questa interrogazione fallisce o restituisce dati obsoleti, non è semplicemente «LDAP non funzionante»: a seconda dell'integrazione, i messaggi vengono rifiutati, le regole applicate in modo errato o gli accessi bloccati. L'amministratore deve quindi sapere in quale punto è interrotto il percorso dal nome DNS all'attributo letto.

La spiegazione segue questo percorso. Prima il client trova un server e stabilisce una sessione protetta. Quindi si autentica, esegue una ricerca e interpreta le risposte. Solo quando questo flusso normale è chiaro si possono inquadrare correttamente schema, particolarità di Active Directory, replica, scalabilità e ripristino.

## Stack di protocollo e modello di sessione

Prima che un'applicazione possa cercare, necessita di una destinazione di servizio concreta. Negli ambienti Active Directory, i record DNS SRV forniscono possibili Domain Controller o Global Catalog; altri prodotti usano FQDN statici, service discovery proprietaria o un load balancer. Questa selezione determina non solo l'indirizzo IP, ma anche posizione, ruolo del server e nome rispetto al quale viene verificato il certificato. Un test di porta verso un server qualsiasi raggiungibile non risponde quindi ancora alla domanda se l'applicazione raggiunge la propria destinazione prevista.

Sulla destinazione scelta LDAP stabilisce una connessione [TCP](/kb/tcp). La porta 389 inizia come LDAP e può passare a una sessione protetta con la StartTLS Extended Operation. La porta 636 è registrata presso IANA come `ldaps` e viene usata, tra l'altro, da Active Directory per TLS avviato immediatamente. In entrambi i casi il client deve verificare la catena del certificato e il nome del server; «cifrato» e «connesso al server corretto» sono due verifiche diverse ([RFC 4511, sezioni 4.14 e 5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14), [IANA Service Name Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap), [MS-ADTS, Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81)).

All'interno di questa connessione LDAP non trasmette righe di comando leggibili come [SMTP](/kb/smtp). I messaggi sono descritti come strutture ASN.1 e codificati con Basic Encoding Rules, BER. Ogni `LDAPMessage` contiene un `messageID`, esattamente un'operazione e controlli opzionali. Tramite il Message-ID, una connessione di lunga durata può distinguere più operazioni in corso; le relative risposte non devono arrivare nell'ordine delle richieste. Un handshake TCP riuscito non dice quindi nulla sulla decodifica BER, sul Bind o su una ricerca completata integralmente ([RFC 4511, sezioni 3.1, 4.1.1 e 5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.1)).

| Livello | Contenuto standardizzato | Osservazione rilevante per l'amministratore |
|---|---|---|
| Applicazione | Bind, Search, Compare, Modify, Add, Delete, ModifyDN, Extended Operations e Controls | Result Code, `diagnosticMessage`, Entries, References e Controls |
| Codifica | Tipi di dati ASN.1 in BER | Errori del decoder, dimensione massima della richiesta, Message-ID e OID |
| Sicurezza | TLS nonché meccanismi SASL e relativi Security Layer | Nome del certificato, Trust Chain, metodo di Bind, Signing, Channel Binding |
| Trasporto | Connessione TCP di lunga durata | Destinazione DNS, porta, latenza di connessione, reset, Idle Timeout e stato del pool |
| Interno del server | DIT, schema, ACL, indice, storage e replica | non standardizzato da LDAP; specifico del prodotto e della topologia |

Per le modifiche vale un limite importante: una singola operazione LDAP è atomica nel proprio ambito, ma più entry non formano una transazione comune del protocollo di base. RFC 5805 descrive un'estensione sperimentale per transazioni, il cui supporto il client deve riconoscere nel Root DSE. Anche in questo caso occorre verificare come le repliche vedano la modifica. I processi di provisioning necessitano quindi di regole proprie per ripetizione, errori parziali e riconciliazione, anziché di una transazione di database tacitamente presunta ([RFC 4511, sezione 3](https://datatracker.ietf.org/doc/html/rfc4511#section-3), [RFC 5805, sezioni 1 e 3](https://datatracker.ietf.org/doc/html/rfc5805#section-1)).

## Modello dati: DIT, Entry, attributo e schema

Dopo l'instaurazione della sessione, il client deve poter indicare dove e cosa cerca. LDAP organizza a questo scopo i dati della directory come Directory Information Tree, in breve DIT. Ogni Entry dispone di un Distinguished Name univoco e di attributi. Lo schema descrive quali attributi esistono, come vengono confrontati i loro valori e quali Object Classes richiedono o consentono. Senza questo modello, base di ricerca, filtri e risultati restano semplici stringhe senza un significato affidabile ([RFC 4512, sezioni 2 e 3](https://datatracker.ietf.org/doc/html/rfc4512#section-2), [RFC 4511, sezione 4.1.7](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.7)).

Il Distinguished Name forma il percorso di un'entry nell'albero. In `cn=Mail Gateway,ou=Services,dc=example,dc=ch`, `cn=Mail Gateway` indica il Relative Distinguished Name locale; gli RDN successivi conducono attraverso il container fino alla radice dei nomi. Poiché gli RDN possono avere più valori e caratteri come virgola, più e backslash devono essere escaped, il software non deve elaborare un DN dividendo semplicemente sulle virgole. Richiede un parser conforme a RFC 4514 ([RFC 4512, sezione 2.3](https://datatracker.ietf.org/doc/html/rfc4512#section-2.3), [RFC 4514, sezioni 2 e 3](https://datatracker.ietf.org/doc/html/rfc4514#section-2)).

```text
dn: cn=Mail Gateway,ou=Services,dc=example,dc=ch
objectClass: top
objectClass: person
objectClass: organizationalPerson
cn: Mail Gateway
sn: Gateway
mail: mail-gateway@example.ch
```

Per esportazioni e importazioni esiste LDIF, una rappresentazione testuale standardizzata. LDIF rappresenta Entries o record di modifica, ma non è il formato wire della sessione LDAP in corso. Il folding delle righe, i valori Base64 e i Change Records seguono regole proprie. Soprattutto, un'esportazione contiene soltanto ciò che il server e le autorizzazioni rendono visibile; possono mancare attributi operativi, ACL o stato del backend. Un dump LDIF è quindi un estratto di dati, ma non automaticamente un backup del server ripristinabile ([RFC 2849, sezioni 2 e 4](https://datatracker.ietf.org/doc/html/rfc2849#section-2)).

Il significato di un valore di attributo deriva soltanto dallo schema. Una Matching Rule come `caseIgnoreMatch`, `integerMatch` o un confronto DN decide se due valori sono uguali e quali filtri funzionano su di essi. Sul wire, i valori appaiono inizialmente come Octet Strings; sintassi e tipo di attributo forniscono la loro interpretazione. Gli elementi di schema personalizzati necessitano pertanto di OID univoci permanenti, sintassi e Matching Rules definite, nonché di un rollout che consideri congiuntamente server e tutti i client dipendenti ([RFC 4512, sezione 4](https://datatracker.ietf.org/doc/html/rfc4512#section-4), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517), [RFC 4520](https://datatracker.ietf.org/doc/html/rfc4520)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ldap.svg?v=20260813" title="Interaktive Infografik: LDAP-Protokollstack, Nachrichtenschicht, DIT, Serverarchitektur und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-ldap.svg?v=20260813">Apri l'infografica sul protocollo LDAP e sull'architettura della directory</a>
</iframe>

## Operazioni e cambiamenti di stato

Con trasporto, nomi e schema è pronta la struttura di base; ora inizia il vero dialogo di protocollo. Il primo cambiamento di stato decisivo è di norma `Bind`. Esso stabilisce sotto quale identità e con quali diritti derivati vengono eseguite le operazioni successive. Un Bind rinnovato sostituisce questo stato. Finché il server elabora un Bind, il client non può avviare altre operazioni sulla stessa connessione ([RFC 4511, sezioni 3.1 e 4.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2)).

Dopo un Bind riuscito, il client può leggere o scrivere. Una ricerca non restituisce un'unica risposta grande, bensì zero o più messaggi `SearchResultEntry`, eventualmente References e, al termine, esattamente un `SearchResultDone`. Solo questo risultato finale indica se la sequenza era completa. `Modify`, `Add`, `Delete` e `ModifyDN` modificano le entry; `Compare` verifica un valore di attributo secondo la sua Matching Rule, senza restituire un normale risultato di ricerca ([RFC 4511, sezioni da 4.5 a 4.9](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5)).

Anche la fine della sessione ha una semantica chiara. `Unbind` è una richiesta unilaterale di chiusura e non possiede risposta. `Abandon` chiede al server di interrompere una specifica operazione, ma non ne garantisce l'interruzione. Se invece TCP si interrompe, tutte le operazioni in corso scompaiono. In caso di operazione di scrittura, il client non può allora sapere con certezza se la modifica è diventata effettiva prima o dopo la perdita della connessione; un retry necessita quindi prima di una riconciliazione dello stato anziché di una ripetizione cieca ([RFC 4511, sezioni 4.3 e 4.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.3)).

| Operazione | Utilizzo tipico | Limite che un client deve gestire |
|---|---|---|
| Bind | Account di servizio, verifica utente o autenticazione SASL | una connessione TCP/TLS riuscita non è ancora un Bind riuscito |
| Search | Destinatari, gruppi, indirizzi, policy e lettura del Root DSE | più Entries, References, limiti, Controls e risultato conclusivo |
| Compare | Verificare lato server un valore di attributo noto | il risultato è `compareTrue` o `compareFalse`, non un risultato Search |
| Modify/Add/Delete/ModifyDN | Provisioning e ciclo di vita | atomico per operazione, ma senza transazione del protocollo di base su più Entries |
| Extended Operation | StartTLS, Password Modify o funzioni specifiche del fornitore | verificare OID e supporto sul server di destinazione |
| Controls | Paging, ordinamento, Assertion, Sync o funzione del fornitore | un Control critico sconosciuto deve causare un errore |

Controls ed Extended Operations completano questa sequenza senza introdurre una nuova versione LDAP. Ogni Control possiede un OID, una Criticality e opzionalmente un valore codificato BER. Se un client marca come critico un Control sconosciuto o non eseguibile, l'operazione deve fallire con `unavailableCriticalExtension`; altrimenti il server può ignorarlo. Prima di Paging, Sync o di una funzione del fornitore, un client corretto legge quindi nel Root DSE, tra l'altro, `supportedControl`, `supportedExtension`, `supportedFeatures` e `supportedLDAPVersion` e `supportedSASLMechanisms` ([RFC 4511, sezione 4.1.11](https://datatracker.ietf.org/doc/html/rfc4511#section-4.1.11), [RFC 4512, sezione 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1)).

## Bind, SASL e confini di fiducia TLS

Il Bind decide a chi il server attribuisce la ricerca successiva. Nel Simple Bind occorre distinguere tre casi: DN vuoto e password vuota producono accesso anonimo. Un DN non vuoto con password vuota è un *unauthenticated Bind* e non conferma esplicitamente l'identità indicata, anche se il server può restituire `success`. Solo un DN non vuoto con password non vuota costituisce la normale autenticazione nome/password. I client dovrebbero quindi rifiutare password vuote già prima della richiesta; i server non dovrebbero abilitare inavvertitamente gli unauthenticated Binds ([RFC 4513, sezioni da 5.1.1 a 5.1.3 e 6.3.1](https://datatracker.ietf.org/doc/html/rfc4513#section-5.1)).

Con questa autenticazione a password, il server conosce il segreto presentato. Il trasporto deve quindi essere non solo cifrato, ma anche autenticato. Ciò include una catena di certificati valida e la verifica che il nome DNS configurato sia presente nel certificato. Chi accetta ogni certificato o usa un indirizzo IP può stabilire un canale cifrato verso la controparte sbagliata. La stessa verifica vale per StartTLS e LDAP con TLS immediato ([RFC 4513, sezioni 3.1 e 5.1.3](https://datatracker.ietf.org/doc/html/rfc4513#section-3.1), [RFC 9525, sezioni 2 e 4](https://datatracker.ietf.org/doc/html/rfc9525#section-4), [TLS](/kb/tls)).

SASL consente meccanismi di autenticazione diversi da un semplice password Bind e può inoltre negoziare una protezione per i messaggi LDAP successivi. In Active Directory si incontrano in particolare Negotiate, Kerberos e NTLM. LDAP Signing protegge l'integrità di determinate sessioni SASL; Channel Binding collega l'autenticazione alla connessione TLS sottostante. TLS, Signing e Channel Binding risolvono dunque problemi affini, ma non identici. Un test deve riprodurre il reale tipo di Bind del prodotto ([RFC 4513, sezione 5.2](https://datatracker.ietf.org/doc/html/rfc4513#section-5.2), [Microsoft: LDAP signing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [MS-ADTS, Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0)).

Per introdurre policy AD più rigide non basta quindi guardare la versione di Windows. Le nuove distribuzioni AD DS su Windows Server 2025 richiedono LDAP Signing per impostazione predefinita, mentre gli aggiornamenti mantengono le impostazioni esistenti. Microsoft indica gli eventi Directory Service dal 2886 al 2889 per Signing e dal 3039 al 3041 per Channel Binding. Questi dati di audit mostrano quali client, porte e metodi di Bind sarebbero effettivamente interessati; solo dopo è possibile pianificare l'applicazione sulla base di dati affidabili ([Microsoft: LDAP signing, Default Security Behavior e Event Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing), [Microsoft: LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023)).

## Search: base, scope, filtro e proiezione degli attributi

Dopo un Bind sicuro segue l'operazione da cui dipendono la maggior parte delle integrazioni: Search. La richiesta indica un Base DN, lo Scope, la gestione degli alias, i propri limiti di dimensione e tempo, un filtro e gli attributi desiderati. `baseObject` legge solo l'entry base, `singleLevel` i suoi figli diretti e `wholeSubtree` l'intero sottoalbero inclusa la base. Il server può impostare limiti più rigidi. Zero risultati con `success` sono una risposta valida; `noSuchObject` significa invece che la base di ricerca manca o non è visibile per questa identità ([RFC 4511, sezioni 4.5.1 e 4.5.2](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1)).

Il filtro non descrive una libera logica SQL, bensì un albero in notazione prefissa. `(&(objectClass=person)(mail=*@example.ch))` combina ad esempio un'espressione Equality con una Substring. `|` indica OR, `!` NOT, `=*` Presence e `:=` un Extensible Match. La Matching Rule del rispettivo attributo determina se un confronto usa maiuscole/minuscole, ordinamento numerico o semantica DN ([RFC 4511, sezione 4.5.1](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1), [RFC 4515](https://datatracker.ietf.org/doc/html/rfc4515), [RFC 4517](https://datatracker.ietf.org/doc/html/rfc4517)).

La generazione dei filtri diventa così un compito di sicurezza. I valori provenienti da input utente devono essere codificati secondo RFC 4515; in particolare `*`, parentesi, backslash, NUL e ottetti UTF-8 non validi non devono entrare in forma non elaborata nell'espressione. Una concatenazione di stringhe può altrimenti modificare la struttura del filtro e rendere possibile LDAP injection. L'escaping DN secondo RFC 4514 segue regole diverse e non sostituisce il filter encoding ([RFC 4515, sezione 3](https://datatracker.ietf.org/doc/html/rfc4515#section-3), [RFC 4514, sezione 3](https://datatracker.ietf.org/doc/html/rfc4514#section-3)).

Oltre al filtro, l'elenco di attributi determina quanto restituisce il server. Un elenco vuoto richiede tutti i normali attributi utente, `1.1` nessun attributo, `*` tutti gli attributi utente e `+` tutti gli attributi operativi secondo RFC 3673. Le ACL possono comunque nascondere valori. I client in produzione dovrebbero richiedere solo gli attributi necessari: grandi valori multivalore gravano su rete, decoder e memoria e possono attivare limiti propri del server ([RFC 4511, sezione 4.5.1.8](https://datatracker.ietf.org/doc/html/rfc4511#section-4.5.1.8), [RFC 3673](https://datatracker.ietf.org/doc/html/rfc3673)).

### Paging, ordinamento e risultati variabili

Le quantità di risultati più grandi vengono normalmente trasmesse con il Simple Paged Results Control. Il server allega a ogni pagina un cookie opaco, che il client restituisce insieme alla stessa richiesta. Questo cookie non è un offset né un cursore permanente. Se il contenuto della directory cambia durante la sequenza, le entry possono mancare o comparire due volte. Il paging limita dunque il volume di dati per risposta, ma non genera uno snapshot coerente ([RFC 2696, sezioni 2 e 3](https://datatracker.ietf.org/doc/html/rfc2696#section-2)).

Active Directory rende questa distinzione visibile nella pratica quotidiana: la LDAP Policy `MaxPageSize` limita per impostazione predefinita i risultati non paginati a 1000 oggetti. Un'importazione che riceve esattamente 1000 entry non ha quindi dimostrato la propria completezza. Il client deve elaborare correttamente pagine e cookie e rilevare un'interruzione. Altre policy limitano durata della query, Receive Buffer e Result Sets mantenuti contemporaneamente. Per l'esercizio devono pertanto essere registrati dimensione della pagina, numero di pagine, ultimo avanzamento del cookie, timeout e riavvio ([MS-ADTS, LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99), [Microsoft: Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results)).

## Active Directory come profilo di server LDAP

Le regole precedenti valgono per LDAP in generale. Active Directory Domain Services è un'implementazione server concreta con ruoli e convenzioni aggiuntivi. I suoi dati sono distribuiti su Naming Contexts; un Domain Controller mantiene almeno Schema, Configuration e il Domain Naming Context proprio. Il Root DSE ha il DN vuoto e indica tra l'altro `defaultNamingContext`, `configurationNamingContext`, `schemaNamingContext`, tutti i `namingContexts`, il nome del server e i meccanismi supportati. Dopo TCP e TLS, questa entry è il primo test che dice effettivamente qualcosa sul servizio di directory raggiunto ([RFC 4512, sezione 5.1](https://datatracker.ietf.org/doc/html/rfc4512#section-5.1), [Microsoft RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse), [MS-ADTS, rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db)).

La scelta tra Domain Controller e Global Catalog modifica il risultato della ricerca. Un DC serve LDAP sulla porta 389 o 636 e conosce il Domain Naming Context completo del proprio dominio. Il Global Catalog usa inoltre 3268 o 3269 e conserva una replica parziale di tutti i domini della foresta. Può trovare oggetti nell'intera foresta, ma per i domini remoti fornisce soltanto attributi del Partial Attribute Set. Un risultato trovato con successo non prova quindi ancora che l'attributo richiesto dall'applicazione sia disponibile ([MS-ADTS, Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a), [Microsoft: Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents), [Microsoft: Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog)).

Anche i filtri possono diventare specifici di AD. La Matching Rule `1.2.840.113556.1.4.1941`, `LDAP_MATCHING_RULE_TRANSITIVE_EVAL`, ad esempio, segue attributi collegati e può valutare gruppi annidati. Il relativo supporto non compare semplicemente in `supportedControl`. Inoltre, `memberOf` non contiene il Primary Group. Una decisione di autorizzazione basata sulle appartenenze a gruppi deve quindi considerare esplicitamente annidamento dei gruppi, Primary Group, scope del gruppo, visibilità ACL e stato di replica ([MS-ADTS, LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5), [Microsoft: Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group)).

Quale server risponde a queste interrogazioni è deciso nei client Windows dal DC Locator insieme ai record [DNS](/kb/dns)-SRV. I record correlati a sito e ruolo forniscono candidati con priorità e peso. Un IP inserito staticamente aggira questa selezione e complica la verifica del certificato. Un semplice load balancer TCP distribuisce sì le connessioni, ma senza logica aggiuntiva non conosce né DC scrivibili né Global Catalogs, Naming Contexts o lo stato della replica ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator), [Microsoft: Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created)).

Infine, la raggiungibilità LDAP non deve essere confusa con una replica sana. AD replica le modifiche alla directory mediante il Directory Replication Service Remote Protocol; `syncrepl` di OpenLDAP usa invece LDAP Content Synchronization con provider, consumer e cookie. Una ricerca di test può mostrare che un determinato server risponde. Se tutti i server possiedono le stesse modifiche e recuperano dopo un guasto deve essere verificato con gli strumenti della rispettiva piattaforma ([MS-DRSR, Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1), [RFC 4533](https://datatracker.ietf.org/doc/html/rfc4533), [OpenLDAP Administrator's Guide: Replication](https://www.openldap.org/doc/admin25/replication.html)).

## Modelli di integrazione e operativi

Per l'esercizio è ora meno importante che un prodotto «supporti LDAP», bensì come utilizza LDAP. Con una **lookup in tempo reale**, un messaggio o una sessione attende direttamente Search e la risposta del server. Con il **Credential Check**, un account tecnico cerca prima il DN dell'utente e quindi esegue un secondo Bind con la password inserita. Un **import o cache**, invece, legge molte entry e opera fino all'esecuzione successiva con una copia locale. Questi modelli hanno conseguenze diverse per latenza, gestione delle password, failover e aggiornamento dei dati; la documentazione del prodotto deve indicare il comportamento concreto ([RFC 4511, sezioni 4.2 e 4.5](https://datatracker.ietf.org/doc/html/rfc4511#section-4.2), [RFC 2696](https://datatracker.ietf.org/doc/html/rfc2696)).

| Modello | Percorso critico | Evidenza operativa immediata |
|---|---|---|
| Lookup in tempo reale | DNS, Connect, TLS, pool, Bind, Search e risposta del server per operazione | p50/p95/p99 per operazione, saturazione del pool, Result Codes, destinazione fallback |
| Credential Check | Ricerca utente più secondo Bind con password dell'utente | Risoluzione DN, blocco password vuota, verifica del nome TLS, comportamento di lockout |
| Importazione periodica | Enumerazione completa paginata e commit nella cache locale | Avanzamento pagina/cookie, numero di oggetti, modello di eliminazione, ultimo commit riuscito |
| Change Sync | Cursore di sincronizzazione specifico del fornitore o LDAP | Persistenza cursore, replay, resync e oggetti eliminati |

Indipendentemente dal modello, il client necessita di timeout separati per Connect, Bind, Operation e Idle. Un Connection Pool risparmia l'instaurazione di TCP, TLS e Bind, ma mantiene con sé lo stato di autenticazione della connessione. Le sessioni morte devono essere rilevate e una connessione non deve cambiare inavvertitamente tra utenti o tenant. Il failover richiede un ordine di destinazione comprensibile, ripetizioni limitate e un percorso di ritorno verso la destinazione preferita. Altrimenti i retry paralleli moltiplicano il carico proprio durante un guasto della directory ([RFC 4511, sezioni 3.1, 4.2 e 5.3](https://datatracker.ietf.org/doc/html/rfc4511#section-3.1)).

Sul lato server, Base DN, Scope, filtro e lista degli attributi determinano il lavoro. Una condizione di uguaglianza selettiva su un attributo indicizzato è diversa da una substring iniziale o da una grande espressione OR. LDAP non pubblica un piano di esecuzione né prescrive una tecnica di indicizzazione. L'amministratore deve pertanto correlare i filtri reali del prodotto con quantità di risultati, latenza p95/p99 e metriche del server. Una ricerca rapida di un singolo account di test non dimostra che una verifica del destinatario sia scalabile sotto carico di punta ([OpenLDAP Administrator's Guide: Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html), [MS-ADTS: LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99)).

Il monitoraggio dovrebbe scomporre il flusso nelle stesse fasi della ricerca guasti: selezione DNS, instaurazione TCP e TLS, Bind, latenza Search, Result Code, numero di risultati e avanzamento del paging. Si aggiungono occupazione del pool, tasso di retry e stato di importazione o sync. Un singolo Bind sintetico può confermare la raggiungibilità, ma non rileva né attributi mancanti né un'importazione incompleta o un partner di replica arretrato.

Anche per backup e recovery il contenuto visibile della directory non basta. Occorre salvaguardare schema, ACL, configurazione backend e server, chiavi e certificati, identità di replica nonché la procedura con cui un nodo ripristinato viene riammesso nella topologia. Active Directory usa a tale scopo System State e propri passaggi di forest recovery; OpenLDAP dipende dal proprio backend. Il manuale distingue ad esempio un backup LMDB da `slapcat` e avverte di stati LDIF semanticamente incoerenti in caso di modifiche multipart. LDAP stesso non definisce un meccanismo di backup ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [OpenLDAP Administrator's Guide: Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html)).

Un test di ripristino è concluso soltanto quando un client trova il servizio ripristinato tramite il nome DNS previsto, TLS e Bind hanno successo, Root DSE e schema sono corretti, ricerche reali restituiscono attributi completi e la replica riprende in modo controllato. Il recovery riporta così all'inizio dell'articolo: conta l'intero percorso, non soltanto un database avviato.

## Strumenti diagnostici

La diagnosi segue lo stesso percorso di una query in produzione. Inizia nella rete dell'applicazione interessata e usa il suo nome DNS, truststore, metodo di Bind, Base DN, filtro e lista degli attributi. Un test dal laptop dell'amministratore può altrimenti avere successo, mentre il gateway continua a usare un altro DC, un'altra CA o un altro scope. Gli esempi usano nomi riservati e leggono soltanto metadati; le password Bind non appartengono né alla shell history né agli argomenti di processo. `ldapsearch -W` le richiede in modo interattivo.

### Determinare le destinazioni del servizio tramite DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Dienstsuche">
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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano target, porte, priorità e pesi. Successivamente vanno verificate risoluzione A/AAAA, riferimento al sito e raggiungibilità di ogni target effettivamente selezionabile. Un singolo DC raggiungibile non corregge un set SRV errato ([Microsoft: DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator)).

### Verificare TCP e TLS implicito sulla porta 636

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) dimostra inizialmente soltanto la connessione TCP. Il successivo [.NET `SslStream`](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) o [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) verifica TLS con il nome DNS configurato. `s_client -showcerts` mostra soltanto i certificati inviati dal server e, da solo, non costituisce una prova riuscita della chain o dell'hostname. StartTLS sulla porta 389 può essere verificato separatamente sotto Unix con `openssl s_client -starttls ldap` ([OpenSSL `s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), [RFC 4511, sezione 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

### Leggere Root DSE e capacità

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Root-DSE-Prüfung">
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

[`Get-ADRootDSE`](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) usa qui il modulo ActiveDirectory e per impostazione predefinita l'identità Windows connessa. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) impone con `-ZZ` StartTLS riuscito e legge anonimamente soltanto gli attributi Root DSE rilasciati dal server. Un OID mancante dimostra che proprio questa destinazione non pubblica la funzione; non dice nulla sugli altri nodi del cluster.

### Riprodurre una ricerca reale con scope, filtro e paging

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-Suchprüfung">
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

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) accetta con `-LDAPFilter` la sintassi del filtro vicina a RFC e gestisce il paging tramite `-ResultPageSize`. [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) usa `-E pr=500/noprompt` per il Paged Results Control e `-W` per una richiesta di password interattiva. Oltre al risultato, il test deve documentare risultato finale, numero di pagine, attributi restituiti e durata.

### Attribuire un errore a un confine

| Osservazione | Significato del protocollo | Prossima evidenza affidabile |
|---|---|---|
| Timeout prima di TLS | Risoluzione della destinazione, routing, firewall, listener o pool esaurito | SRV/A/AAAA, handshake TCP, listener del server e latenza Connect |
| Errore di certificato | Chain, validità, nome o client trust non corrispondono | Chain inviata, Trust Anchor, SAN rispetto al FQDN esattamente configurato |
| `strongAuthRequired` / `confidentialityRequired` | Il server richiede un metodo di Bind o protezione più forte | Porta, successo StartTLS, meccanismo SASL, policy Signing/CBT |
| `invalidCredentials` | Identità Bind presentata o credenziali rifiutate | Tipo di Bind e DN esatti; nessuna registrazione della password |
| `invalidDNSyntax` | DN sintatticamente non valido | Codifica RFC 4514 e DN effettivo dal Search Result |
| `noSuchObject` con `matchedDN` | Base DN assente o invisibile a partire da un predecessore | Root DSE, Naming Context, visibilità ACL e `matchedDN` |
| `sizeLimitExceeded` | Limite client o server prima del risultato completo | Paging Control, Page Cookies, LDAP Policy e conteggio totale |
| `adminLimitExceeded` / `busy` / `unavailable` | Risorsa del server o limite amministrativo | Metriche del server, Query Policy, costo del filtro, tasso di retry e nodo di destinazione |
| zero risultati con `success` | Ricerca valida senza match visibile | Confrontare base, scope, filtro, ACL, nodo di destinazione e stato di replica |

I Result Codes numerici appartengono al protocollo LDAP; `diagnosticMessage` e ulteriori subcodici AD sono invece contesto specifico dell'implementazione. L'automazione dovrebbe quindi valutare prima il Result Code e registrare il testo come integrazione. Per `busy` e `unavailable` ogni client necessita di un budget di retry limitato con backoff. Ripetizioni illimitate trasformano un singolo problema di directory in un picco di carico su tutti i sistemi dipendenti ([RFC 4511, sezione 4.1.9 e appendice A](https://datatracker.ietf.org/doc/html/rfc4511#appendix-A)).

## Storia tecnica

LDAP non nacque come database di directory indipendente. Alla fine degli anni 1980, X.500 aveva definito un modello di directory completo e il Directory Access Protocol. RFC 1487 descrisse nel 1993 un accesso più leggero a questo modello; RFC 1777 seguì nel 1995 come LDAP Version 2. «Lightweight» si riferiva all'accesso al protocollo semplificato rispetto a DAP, non a directory piccole o a scarsa rilevanza operativa. Lo sviluppo iniziale è strettamente collegato a Tim Howes e all'University of Michigan ([RFC 1487](https://datatracker.ietf.org/doc/html/rfc1487), [RFC 1777](https://datatracker.ietf.org/doc/html/rfc1777)).

LDAPv3 fu pubblicato nel 1997 con RFC 2251 e documenti di accompagnamento. Operazioni estendibili, Controls, SASL, internazionalizzazione e il modello dati rivisto lo resero la base delle implementazioni odierne. Il lavoro LDAPbis riorganizzò questo stato nel 2006: RFC 4510 funge da roadmap, RFC 4511 descrive il protocollo, RFC 4512 il modello informativo e RFC 4513 la sicurezza; RFC da 4514 a 4519 completano rappresentazioni, URL, sintassi e schema ([RFC 2251](https://datatracker.ietf.org/doc/html/rfc2251), [RFC 4510, sezione 3](https://datatracker.ietf.org/doc/html/rfc4510#section-3)).

Parallelamente si svilupparono server molto diversi. OpenLDAP nacque nel 1998 dall'implementazione dell'University of Michigan e proseguì `slapd`, librerie e strumenti come progetto open source. Con Windows 2000, Active Directory portò nell'ampio uso aziendale un profilo LDAPv3 con schema proprio, Naming Contexts, Controls, Matching Rules e protocollo di replica separato ([OpenLDAP Release Road Map](https://www.openldap.org/software/roadmap.html), [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html), [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/)).

Questa storia spiega la regola operativa più importante: LDAP uniforma l'accesso, non l'architettura interna. Chi sposta un client da OpenLDAP a AD DS o tra due appliance deve quindi verificare più di host, porta e Bind DN. Schema, Controls, limiti, risoluzione dei gruppi, replica e recovery rimangono caratteristiche del prodotto.

## Fonti

- [RFC 4510, sezioni 1 e 2](https://datatracker.ietf.org/doc/html/rfc4510)
- [RFC 4511 – LDAP: The Protocol](https://datatracker.ietf.org/doc/html/rfc4511) – livello dei messaggi, operazioni, BER, TCP, StartTLS e Result Codes.
- [IANA – Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=ldap) – `ldap` 389 e `ldaps` 636.
- [MS-ADTS – Using SSL/TLS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/8e73932f-70cf-46d6-88b1-8d9f86235e81) – TLS implicito e StartTLS in Active Directory.
- [RFC 5805, sezioni 1 e 3](https://datatracker.ietf.org/doc/html/rfc5805)
- [RFC 4512 – LDAP Directory Information Models](https://datatracker.ietf.org/doc/html/rfc4512) – DIT, Entries, attributi, schema, Root DSE e subschema.
- [RFC 4514, sezioni 2 e 3](https://datatracker.ietf.org/doc/html/rfc4514)
- [RFC 2849, sezioni 2 e 4](https://datatracker.ietf.org/doc/html/rfc2849)
- [RFC 4517 – LDAP Syntaxes and Matching Rules](https://datatracker.ietf.org/doc/html/rfc4517) – sintassi standard e regole di confronto.
- [RFC 4520 – IANA Considerations for LDAP](https://datatracker.ietf.org/doc/html/rfc4520) – registrazione di OID e parametri del protocollo.
- [RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513) – metodi Bind, SASL, TLS e confini di sicurezza.
- [RFC 9525, sezioni 2 e 4](https://datatracker.ietf.org/doc/html/rfc9525)
- [Microsoft – LDAP signing for AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing) – Signing, Channel Binding, impostazioni predefinite ed eventi.
- [MS-ADTS – Channel Binding](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/2c3ea153-eff9-46a7-8614-19b677efa4e0) – LDAP Channel Binding in Active Directory.
- [Microsoft – LDAP session security after ADV190023](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023) – requisiti di sicurezza per modello Bind e TLS.
- [RFC 4515 – String Representation of Search Filters](https://datatracker.ietf.org/doc/html/rfc4515) – grammatica dei filtri e Value Encoding.
- [RFC 3673 – All Operational Attributes](https://datatracker.ietf.org/doc/html/rfc3673) – `+` come selettore di attributi per attributi operativi.
- [RFC 2696 – Simple Paged Results Control](https://datatracker.ietf.org/doc/html/rfc2696) – pagine, cookie e limiti di coerenza.
- [MS-ADTS – LDAP Policies](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/3f0137a1-63df-400c-bf97-e1040f055a99) – limiti amministrativi di ricerca e risorse.
- [Microsoft – Paging Search Results](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/paging-search-results) – Paged Search in Active Directory.
- [Microsoft – RootDSE](https://learn.microsoft.com/en-us/windows/win32/adschema/rootdse) – Naming Contexts e capacità del server.
- [MS-ADTS – rootDSE Attributes](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/96f7b086-1ca3-4764-9a08-33f8f7a543db) – attributi Root DSE specifici di AD.
- [MS-ADTS – Ports](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/1010e59a-cf64-410c-a05c-a3a6c715261a) – porte LDAP, LDAPS e Global Catalog.
- [Microsoft – Searching the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/searching-global-catalog-contents) – ricerca nell'intera foresta e replica parziale.
- [Microsoft – Attributes included in the Global Catalog](https://learn.microsoft.com/en-us/windows/win32/ad/attributes-included-in-the-global-catalog) – Partial Attribute Set del Global Catalog.
- [MS-ADTS – LDAP Matching Rules](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/4e638665-f466-4597-93c4-12f2ebfabab5) – OID Extensible Match specifici di AD.
- [Microsoft – Primary group membership](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/user-prov-sync/exclude-user-primary-group) – delimitazione tra `memberOf` e Primary Group.
- [Microsoft – DC Locator](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/dc-locator) – selezione DNS-SRV, LDAP Ping e riferimento al sito.
- [Microsoft – Verify LDAP SRV records](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/verify-srv-dns-records-have-been-created) – registrazione SRV dei Domain Controller.
- [MS-DRSR – Relationship to Other Protocols](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/0597aff6-0177-4d52-99f2-14a5441bc3c1) – delimitazione della replica AD da LDAP.
- [RFC 4533 – LDAP Content Synchronization Operation](https://datatracker.ietf.org/doc/html/rfc4533) – controlli LDAP Sync, cookie e modello di stato.
- [OpenLDAP Administrator's Guide – Replication](https://www.openldap.org/doc/admin25/replication.html) – `syncrepl`, cookie e modello provider/consumer.
- [OpenLDAP Administrator's Guide – Performance Tuning](https://www.openldap.org/doc/admin26/tuning.html) – indicizzazione e costi di ricerca interni al server.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – backup System State per AD DS.
- [OpenLDAP Administrator's Guide – Directory Backups](https://www.openldap.org/doc/admin26/maintenance.html) – backup LMDB, `slapcat` e limiti di coerenza.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – query DNS e SRV in Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – query DNS e SRV in sistemi Unix.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnostica della connessione TCP in Windows.
- [Microsoft Learn – SslStream](https://learn.microsoft.com/dotnet/api/system.net.security.sslstream) – handshake TLS e verifica del certificato con .NET.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – handshake TLS, verifica del nome e LDAP StartTLS.
- [Microsoft Learn – Get-ADRootDSE](https://learn.microsoft.com/powershell/module/activedirectory/get-adrootdse?view=windowsserver2025-ps) – diagnostica Root DSE in Windows.
- [OpenLDAP – ldapsearch(1)](https://www.openldap.org/software/man.cgi?query=ldapsearch&sektion=1&apropos=0&manpath=OpenLDAP+2.6-Release) – opzioni Search, StartTLS, SASL e Control.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser?view=windowsserver2025-ps) – LDAPFilter, SearchBase, Scope e Paging.
- [RFC 1487 – X.500 Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1487) – prima specifica LDAP del 1993.
- [RFC 1777 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc1777) – LDAPv2 e modello storico del protocollo.
- [RFC 2251 – Lightweight Directory Access Protocol v3](https://datatracker.ietf.org/doc/html/rfc2251) – prima specifica centrale LDAPv3.
- [OpenLDAP – Release Road Map](https://www.openldap.org/software/roadmap.html) – Release 1.0 nell'agosto 1998.
- [OpenLDAP Administrator's Guide – Preface](https://www.openldap.org/doc/admin26/preface.html) – LDAP dell'University of Michigan come base del progetto.
- [MS-ADTS – Active Directory Technical Specification](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/) – profilo di server LDAP di AD DS e AD LDS.
