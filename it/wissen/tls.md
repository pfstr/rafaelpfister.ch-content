---
title: "TLS: protocollo, verifica dei certificati e gestione operativa"
blatt: "tls"
description: "Transport Layer Security dal punto di vista dell’amministratore: handshake e Record Layer, piano delle chiavi TLS 1.3, convalida del percorso X.509 e PKIX, SAN e SNI, truststore, integrazione SMTP e LDAP, gestione delle chiavi e diagnosi stratificata."
fakten:
  - label: Nome completo
    wert: Transport Layer Security
    href: https://datatracker.ietf.org/doc/html/rfc9846
  - label: Obiettivi di protezione
    wert: Autenticazione · riservatezza · integrità
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Parti del protocollo
    wert: Handshake Protocol · Record Protocol · Alerts
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Base
    wert: flusso di byte affidabile e ordinato
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Autenticazione
    wert: Server sempre · client facoltativo
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-1
  - label: Scambio di chiavi
    wert: (EC)DHE · PSK · combinato
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2
  - label: Protezione dei record
    wert: AEAD con Traffic Secrets separati
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-5
  - label: Profilo del certificato
    wert: X.509 v3 e convalida del percorso PKIX
    href: https://datatracker.ietf.org/doc/html/rfc5280
  - label: Identità del servizio
    wert: ID di riferimento rispetto a subjectAltName
    href: https://datatracker.ietf.org/doc/html/rfc9525
  - label: Modelli di avvio
    wert: TLS implicito o STARTTLS specifico del protocollo
    href: https://datatracker.ietf.org/doc/html/rfc8314
  - label: Ripresa
    wert: Session Resumption basata su PSK
    href: https://datatracker.ietf.org/doc/html/rfc9846#section-2.2
  - label: Raccomandazione operativa
    wert: BCP 195
    href: https://www.rfc-editor.org/info/bcp195
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - smtp-mailflow
translationSourceHash: 50ea371b919edcc4811a303d26ec6c575ef8dc334ce6e681fdc3f74de9ca2058
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:43:15.239Z
translationReview: automatic
---

# TLS: protocollo, verifica dei certificati e gestione operativa

**Transport Layer Security** (TLS) protegge una singola connessione tra due endpoint. Prima client e server concordano procedure crittografiche e chiavi, poi SMTP, LDAP, HTTP o un altro protocollo applicativo transitano attraverso questo canale protetto. TLS impedisce l’intercettazione e le modifiche non rilevate su questa connessione e autentica almeno il server. Tuttavia, non conferma né il mittente di un’e-mail né garantisce che anche il successivo hop SMTP utilizzi TLS ([RFC 9846, sezione 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1), [RFC 3207, sezione 6](https://datatracker.ietf.org/doc/html/rfc3207#section-6)).

Nell’esercizio operativo devono quindi superare diverse verifiche in sequenza: l’host e la porta corretti devono essere raggiungibili, client e server devono avere parametri TLS comuni, la catena del certificato deve condurre a un’ancora attendibile e il nome contattato deve figurare nel certificato. Solo allora inizia il protocollo applicativo vero e proprio. L’articolo segue esattamente questo flusso e mostra, in ogni punto, ciò che un amministratore deve osservare, configurare e proteggere.

## Posizione nello stack di protocolli

Nella normale operatività dei server, TLS si colloca tra [TCP](/kb/tcp) e il protocollo applicativo. TCP fornisce il flusso di dati ordinato; TLS lo protegge; [SMTP](/kb/smtp), [LDAP](/kb/ldap), IMAP o HTTP definiscono quando TLS si avvia e quale nome del server è atteso. Questa integrazione spiega perché una porta aperta non equivale ancora a un test TLS riuscito e un handshake riuscito non dimostra ancora il funzionamento di un servizio di posta o di directory ([RFC 9846, sezione 1](https://datatracker.ietf.org/doc/html/rfc9846#section-1)).

| Livello | Oggetto visibile | Tipica domanda dell’amministratore |
|---|---|---|
| Selezione del nome e del servizio | Nome DNS, destinazione MX o SRV, porta | Quale endpoint è stato effettivamente selezionato? |
| Trasporto | Connessione TCP e macchina a stati | SYN, SYN-ACK e ACK arrivano in entrambe le direzioni? |
| Handshake TLS | Versione, Key Share, firma, certificati, Finished | Esistono parametri comuni e il peer viene accettato? |
| TLS Record Layer | Record cifrati con protezione AEAD | I record vengono trasmessi, scartati o terminati con un Alert? |
| Protocollo applicativo | Dialogo SMTP, LDAP, IMAP o HTTP | Quali comandi e identità si applicano dopo l’handshake? |

QUIC è un’importante eccezione al modello classico «TLS su TCP»: utilizza l’handshake TLS 1.3, ma non il TLS Record Layer; QUIC protegge i propri pacchetti con chiavi derivate da TLS ([RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)). Per i protocolli di posta e directory qui considerati, il canale TLS basato su TCP rimane il modello rilevante.

## Architettura: Handshake, Record Layer e Alerts

TLS svolge due compiti in sequenza. Nell’**Handshake Protocol** le controparti negoziano parametri, si autenticano e generano materiale crittografico condiviso. Successivamente il **Record Protocol** suddivide i dati utili in record protetti. Gli **Alerts** segnalano errori di protocollo o una chiusura ordinata; in caso di errore improvviso di rete o di processo, tuttavia, la connessione TCP può terminare anche senza un Alert utilizzabile ([RFC 9846, sezioni da 4 a 6](https://datatracker.ietf.org/doc/html/rfc9846#section-4)).

Questa separazione è decisiva nell’esercizio operativo. Un errore di certificato appartiene alla fase di autenticazione dell’handshake. Un `bad_record_mac` o un `decrypt_error` riguarda l’elaborazione crittografica. Un timeout senza ServerHello, invece, indica spesso una confusione di protocollo, un filtro, una porta errata o un server che si aspetta prima un dialogo in chiaro e un comando STARTTLS.

## Handshake TLS 1.3 passo per passo

L’handshake TLS 1.3 completo può essere letto come una conversazione: il client fa un’offerta, il server sceglie da essa, dimostra la propria identità ed entrambe le parti confermano il protocollo svolto finora. Già dopo il **ServerHello**, i successivi messaggi dell’handshake sono protetti. Con i messaggi **Finished** di entrambe le parti, risulta infine certo che possiedono la stessa trascrizione e gli stessi segreti ([RFC 9846, sezione 2](https://datatracker.ietf.org/doc/html/rfc9846#section-2)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-tls-handshake.svg?v=20260813" title="Interaktive Infografik: TLS-1.3-Handshake, Zertifikatskette, Identitätsprüfung und Admin-Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tls-handshake.svg?v=20260813">Apri l’infografica sull’handshake TLS e la verifica dei certificati</a>
</iframe>

Il flusso in dettaglio:

1. Il **ClientHello** offre, tra l’altro, versioni TLS supportate, Cipher Suites, algoritmi di firma e gruppi crittografici. Un `key_share` contiene normalmente già la componente pubblica effimera del client. SNI può trasportare il nome del server desiderato, ALPN i possibili protocolli applicativi.
2. Il **ServerHello** seleziona versione, Cipher Suite e Key Share. Dalle componenti del client e del server nasce il segreto condiviso (EC)DHE. Se non esiste un gruppo adatto inviato in precedenza, il server può richiedere un nuovo ClientHello con **HelloRetryRequest**.
3. **EncryptedExtensions** conferma ulteriori estensioni negoziate. Se il server richiede un certificato client, segue anche **CertificateRequest**.
4. **Certificate** trasporta il certificato Leaf e normalmente i certificati Intermediate necessari al client. **CertificateVerify** firma la trascrizione dell’handshake con la chiave privata associata al certificato Leaf. In questo modo il server dimostra il possesso della chiave; il solo certificato non lo dimostra.
5. **Finished** è un MAC sulla trascrizione fino a quel momento e conferma i segreti dell’handshake derivati. Il client convalida parametri, percorso del certificato e identità del servizio, quindi invia il proprio Finished e, se è richiesta l’autenticazione reciproca, anche Certificate e CertificateVerify.
6. Per i dati applicativi vengono derivati Traffic Secrets separati. I successivi messaggi **NewSessionTicket** consentono una ripresa basata su PSK, senza che il server debba inviare nuovamente il certificato originale in ogni handshake successivo.

Il numero osservato di round trip dipende quindi da DNS, TCP, handshake TLS completo o ripreso, HelloRetryRequest e applicazione. Un packet capture deve misurare separatamente queste fasi; «TLS era lento» non è una diagnosi senza questa suddivisione.

## Piano delle chiavi e Record Layer

Dal materiale (EC)DHE o PSK concordato, TLS 1.3 non genera un’unica chiave di sessione, bensì diversi segreti separati tramite **HKDF**. Handshake, dati applicativi, esportazioni e successiva ripresa utilizzano materiale proprio; inoltre client e server possiedono chiavi e vettori di inizializzazione separati per ciascuna direzione. Questa suddivisione è importante per l’analisi e l’incident response, poiché una chiave Record divulgata non decifra automaticamente tutte le fasi e le direzioni ([RFC 9846, sezione 7.1](https://datatracker.ietf.org/doc/html/rfc9846#section-7.1)).

Il Record Layer utilizza esclusivamente **Authenticated Encryption with Associated Data** (AEAD). In TLS 1.3, la Cipher Suite denomina l’algoritmo AEAD e l’hash per HKDF; tipo di certificato, algoritmo di firma e gruppo di scambio delle chiavi vengono negoziati separatamente. Ciò è diverso da TLS 1.2, dove nomi come `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` raggruppano più decisioni in una suite.

| Caratteristica | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Semantica della Cipher Suite | Scambio di chiavi, autenticazione, cifratura e hash raggruppati | Procedura AEAD e hash HKDF; Key Exchange e firma separati |
| Forward Secrecy | dipende dalla suite selezionata | negoziazione a chiave pubblica solo con (EC)DHE effimero |
| Visibilità dell’handshake | gran parte in chiaro | cifrato dopo ServerHello |
| Ripresa | Session ID o ticket secondo il modello precedente | modello PSK uniforme |
| 0-RTT | non parte del protocollo | facoltativo, ma ripetibile e non adatto a operazioni non idempotenti |

Lo 0-RTT risparmia un round trip durante una ripresa, ma possiede proprietà di replay più deboli. Un server deve quindi decidere in base all’applicazione quali dati anticipati accettare. Per operazioni amministrative, di posta o di directory che modificano lo stato, una bassa latenza non deve essere confusa con la sicurezza contro i replay ([RFC 9846, sezioni 2.3 e 8](https://datatracker.ietf.org/doc/html/rfc9846#section-2.3)).

## Certificato X.509 e chiave privata

Un certificato X.509 è la descrizione firmata di una chiave pubblica. Indica tra l’altro emittente, periodo di validità, utilizzi consentiti e nomi per i quali la chiave può essere impiegata. La relativa chiave privata non è parte del certificato e deve rimanere protetta presso il gestore. RFC 5280 definisce il profilo PKIX utilizzato a tale scopo, con campi quali Key Usage, Extended Key Usage, Basic Constraints e Subject Alternative Name ([RFC 5280, sezione 4](https://datatracker.ietf.org/doc/html/rfc5280#section-4)).

X.509 è codificato in binario con ASN.1 DER. Le righe spesso visibili `BEGIN CERTIFICATE` e `END CERTIFICATE` sono un involucro testuale basato su Base64 descritto da RFC 7468. Un contenitore **PKCS #12** può invece trasportare insieme chiave privata, catena del certificato e ulteriori attributi ([RFC 7468](https://datatracker.ietf.org/doc/html/rfc7468), [RFC 7292](https://datatracker.ietf.org/doc/html/rfc7292)). Estensioni di file come `.cer`, `.crt`, `.pem`, `.p12` o `.pfx` sono convenzioni, ma non una prova affidabile di contenuto o codifica.

Le estensioni importanti svolgono compiti diversi:

| Estensione | Domanda di verifica | Errore operativo frequente |
|---|---|---|
| Subject Alternative Name | contiene l’identità DNS, IP, URI o SRV attesa dal client? | manca il nome di gestione o l’alias del Load Balancer |
| Basic Constraints | il certificato può operare come CA e quale profondità può avere il percorso? | Intermediate senza `CA=TRUE` o Path Length errata |
| Key Usage | per quale operazione crittografica di base può essere utilizzata la chiave? | la CA non può firmare oppure il Leaf non è adatto all’uso di firma |
| Extended Key Usage | la chiave è autorizzata per l’autenticazione del server o del client? | un certificato S/MIME viene usato erroneamente come certificato server TLS |
| Authority / Subject Key Identifier | quali varianti di chiave aiutano nella costruzione del percorso? | selezionato l’Intermediate errato con lo stesso Subject |
| CRL Distribution Points / AIA | dove trova il client informazioni di revoca o sull’Issuer? | destinazione non raggiungibile dalla rete del server o proxy mancante |

## Percorso del certificato e confine di fiducia

Con il proprio **certificato Leaf**, un server invia normalmente i necessari **certificati Intermediate**. Il client cerca di costruire da essi un percorso verso una Root CA di cui si fida già localmente. Proprio quest’ultimo punto è decisivo: una Root CA fornita dal server non diventa affidabile per questo solo motivo. Il truststore del processo che effettivamente si connette determina quali ancore di fiducia sono valide ([RFC 5280, sezione 6](https://datatracker.ietf.org/doc/html/rfc5280#section-6)).

La costruzione e la convalida del percorso sono più di tre semplici segni di spunta. Il client deve verificare, tra l’altro, firme, intervalli di validità, Basic Constraints, Path Length, Name Constraints, Key Usage, Extended Key Usage, estensioni critiche, policy e, se necessario, revoca. Successivamente segue l’identità del servizio specifica del protocollo. Client diversi possono costruire percorsi validi differenti a partire dallo stesso certificato Leaf, perché i loro truststore, il recupero AIA, le Cross-Signatures e le regole sugli algoritmi differiscono.

Il termine **truststore** non indica quindi un luogo universale. Un’applicazione Windows può utilizzare Schannel e l’archivio certificati Windows, un’applicazione Java JSSE con un proprio file `cacerts` o dell’applicazione, un container un CA bundle incorporato e un’appliance uno store proprietario. Anche sullo stesso host, due processi possono fidarsi diversamente dello stesso server. In caso di disturbo, devono essere identificati il processo che effettivamente convalida e il suo truststore.

La revoca è un’ulteriore decisione di policy. Le CRL distribuiscono elenchi firmati di certificati revocati; OCSP risponde a richieste di stato per singoli certificati. Se un client verifica, come reagisce a un timeout e se utilizza OCSP Stapling dipende dalla libreria e dall’applicazione ([RFC 5280, sezione 5](https://datatracker.ietf.org/doc/html/rfc5280#section-5), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)). «Il certificato è temporalmente valido» non significa quindi automaticamente «il certificato viene accettato da questo client».

## Identità del servizio: SAN, ID di riferimento e SNI

Dopo aver accettato il percorso, il client verifica il nome del servizio. Il punto di partenza è l’**identità di riferimento**, di solito il nome host configurato o la parte host di un URL. Deve corrispondere a una voce del tipo appropriato nel **Subject Alternative Name** del certificato Leaf. Per i nomi DNS vale `dNSName`, per gli indirizzi IP configurati direttamente `iPAddress`; il Common Name formulato liberamente non è una moderna soluzione alternativa. Le wildcard possono sostituire solo l’intera etichetta più a sinistra ([RFC 9525, sezioni 4 e 6](https://datatracker.ietf.org/doc/html/rfc9525#section-4)).

**Server Name Indication** (SNI) va distinta da questo. Il client invia il nome del server desiderato nel ClientHello affinché un listener condiviso possa selezionare il certificato appropriato e la configurazione virtuale corretta. SNI indica al server che cosa il client desidera raggiungere; non sostituisce né il confronto SAN né la convalida del percorso ([RFC 6066, sezione 3](https://datatracker.ietf.org/doc/html/rfc6066#section-3)). Un test senza SNI può quindi mostrare il certificato predefinito, anche se il client produttivo ne riceve un altro.

**Application-Layer Protocol Negotiation** (ALPN) permette a client e server di selezionare un protocollo di livello superiore all’interno dell’handshake TLS. È visibile soprattutto con HTTP e QUIC, ma concettualmente non è nemmeno una verifica del certificato ([RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)).

## TLS implicito e STARTTLS

Il protocollo applicativo determina quando inizia l’handshake. Con **TLS implicito**, il ClientHello segue immediatamente la creazione della connessione TCP. Con **STARTTLS**, entrambe le parti parlano prima in chiaro e passano a TLS solo dopo un comando specifico del protocollo. Chi invia un ClientHello direttamente a una porta STARTTLS usa quindi TLS, ma nel punto sbagliato del dialogo.

| Percorso del servizio | Porta | Modello di avvio | Regola di protocollo |
|---|---:|---|---|
| Relay SMTP tra MTA | 25 | ESMTP `STARTTLS` | dopo un handshake riuscito SMTP viene reimpostato; il client invia nuovamente EHLO |
| Message Submission | 587 | STARTTLS | Submission può richiedere TLS prima dell’autenticazione |
| Implicit TLS Submission | 465 | TLS immediato | descritto da RFC 8314 per Mail Submission |
| IMAP / IMAPS | 143 / 993 | STARTTLS / TLS immediato | l’accesso in chiaro è considerato obsoleto per l’uso moderno |
| POP3 / POP3S | 110 / 995 | STLS / TLS immediato | anche qui percorsi distinti in chiaro e Implicit TLS |
| LDAP / LDAPS | 389 / 636 | LDAP StartTLS Extended Operation / TLS immediato | Bind e verifica dell’identità seguono la policy di sicurezza LDAP |
| HTTP / HTTPS | 80 / 443 | schemi URI separati; su 443 TLS immediato | Host/SNI, identità del certificato e versione HTTP sono decisioni separate |

Le assegnazioni delle porte provengono dalla [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); la semantica deriva dal rispettivo standard applicativo. La porta 465 non è quindi genericamente «SMTPS per la consegna server-to-server», bensì il percorso Implicit TLS registrato per Submission.

SMTP STARTTLS definisce un cambiamento di stato particolarmente importante. Prima dell’upgrade, la risposta EHLO e l’offerta `STARTTLS` sono manipolabili. Dopo l’handshake entrambe le parti devono scartare le informazioni SMTP apprese in precedenza e il client invia nuovamente EHLO ([RFC 3207, sezione 4.2](https://datatracker.ietf.org/doc/html/rfc3207#section-4.2)). Per LDAP, StartTLS è una Extended Operation all’interno della sessione LDAP; non equivale a una nuova porta TCP ([RFC 4511, sezione 4.14](https://datatracker.ietf.org/doc/html/rfc4511#section-4.14)).

## Policy TLS per SMTP

Con SMTP sulla porta 25, TLS è spesso opportunistico: l’MTA mittente cifra se la controparte offre STARTTLS e la policy locale accetta l’handshake. Senza una policy aggiuntiva, un attaccante può rimuovere l’offerta o disturbare l’handshake. L’MTA mittente deve quindi decidere se consegnare in chiaro, ritentare più tardi o interrompere definitivamente. RFC 3207 chiarisce quindi che un hop SMTP protetto non equivale alla protezione end-to-end del messaggio.

Due meccanismi possono vincolare più strettamente il dominio destinatario e il requisito TLS:

- **DANE for SMTP** pubblica dati TLSA sotto DNSSEC e deriva l’identità da verificare dal percorso MX. Un resolver validante e una catena DNSSEC integra diventano così parte della base di fiducia ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).
- **MTA-STS** pubblica una policy tramite DNS e HTTPS. Indica pattern MX consentiti e richiede TLS convalidato tramite PKIX durante la durata della policy ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)). **SMTP TLS Reporting** segnala quadri aggregati di successi e errori di queste policy ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)).

Questi modelli non sono intercambiabili. DANE ancora l’affermazione in DNSSEC e TLSA; MTA-STS usa Web-PKIX e una policy ottenuta tramite HTTPS. Un amministratore deve quindi osservare, oltre a certificato e Cipher, anche [DNS](/kb/dns), selezione MX, cache della policy e canale di reporting.

## Autenticazione TLS reciproca

**Mutual TLS** (mTLS) integra la normale prova del server con un certificato client. Il server lo richiede tramite CertificateRequest; il client invia la propria catena e dimostra il possesso della chiave privata con CertificateVerify. Il server verifica quindi percorso, validità, Extended Key Usage e il proprio mapping locale. Il certificato identifica inizialmente un titolare della chiave. Quali diritti per account applicativo, LDAP o relay SMTP ne derivino continua a essere deciso dall’applicazione.

Negli ambienti eterogenei sorgono spesso problemi di fiducia asimmetrici: il client si fida della CA del server, ma il server non della CA del client; il server richiede solo determinati Issuer; il certificato client possiede solo `serverAuth`; oppure un Load Balancer termina TLS e non inoltra alcuna identità client verificata al backend. Un handshake riuscito autenticato dal server non dimostra quindi il funzionamento di mTLS.

## Gestione di chiavi, certificati e rinnovi

La sostituzione di un certificato è più della sostituzione di un file. Chiave, certificato, catena Intermediate, binding del listener e stato del rollout devono combaciare. Un ciclo di vita affidabile comprende almeno:

1. **Inventario:** rilevare nome del servizio, porta, modello di avvio, processo che termina TLS, SAN attesi, Issuer, numero di serie, fingerprint, `notBefore`/`notAfter`, Key Store, truststore e automazione responsabile.
2. **Generazione della chiave:** generare una nuova coppia di chiavi nel contesto previsto di software, TPM o HSM. Le chiavi private rimangono riservate e richiedono protezione, controllo degli accessi e, a seconda del modello di recovery, una strategia documentata di backup o riemissione ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).
3. **Richiesta ed emissione:** generare un CSR con identità e parametri corretti; soddisfare la policy della CA e le prove richieste. ACME standardizza l’ordine e il rinnovo automatizzati, ma non garantisce ancora il corretto binding del servizio ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).
4. **Installazione:** portare Leaf, bundle Intermediate appropriato e chiave privata nel formato effettivamente letto dal processo. Verificare diritti sui file, accesso alla password e corrispondenza della coppia di chiavi.
5. **Attivazione:** ricaricare il servizio o commutare il binding in modo atomico. In presenza di cluster, Load Balancer e più ruoli di protocollo, trattare separatamente ogni terminatore TLS.
6. **Convalida:** verificare dall’esterno e dall’interno con SNI, porta e modalità STARTTLS produttivi; rilevare catena fornita, SAN, validità, algoritmo, policy e test dell’applicazione.
7. **Sostituzione:** rimuovere il vecchio certificato solo quando tutti i nodi distribuiscono quello nuovo e nessuna dipendenza punta più al fingerprint o al vecchio Issuer. In caso di compromissione, revocare inoltre il certificato e trattare la chiave privata come compromessa.

Un backup deve adattarsi a questo modello. Una parte pubblica del certificato esportata non ripristina un servizio se mancano chiave privata, password, binding del provider, ACL o oggetto HSM. Al contrario, ogni copia non protetta della chiave privata amplia la superficie di attacco. Il recovery di Root e Issuing CA, il recovery delle chiavi server e la riemissione rapida sono procedure diverse e non devono confluire sotto «backup dei certificati».

La Web-PKI pubblica, la PKI Enterprise interna e le ancore di fiducia gestite autonomamente hanno regole diverse di emissione e durata. Invece di usare un valore massimo statico come obiettivo operativo, il monitoraggio dovrebbe misurare il concreto `notAfter` di ogni certificato distribuito e prevedere tempo sufficiente per emissione, rollout, convalida e rollback.

## Stack di implementazione e tecnologie

Quale stack TLS operi effettivamente dipende dal processo. I servizi Windows possono usare Schannel, le applicazioni Java JSSE, i servizi Unix OpenSSL o GnuTLS e le appliance una libreria propria. Di conseguenza cambiano truststore, procedure supportate, strumenti diagnostici e luogo di configurazione, sebbene tutte le controparti «parlino TLS».

| Stack | Runtime e API | Tipica fonte di trust e chiavi | Rilevanza amministrativa |
|---|---|---|---|
| [OpenSSL](https://docs.openssl.org/master/) ([Codice sorgente](https://github.com/openssl/openssl)) | libreria nativa e CLI, prevalentemente C | file/directory CA, PEM, DER, PKCS #12, Provider | molti servizi Unix e appliance; `s_client`, `x509`, `verify` |
| [Schannel](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) | Windows Security Support Provider tramite SSPI | archivio certificati Windows, chiavi CNG/CSP | applicazioni di sistema e .NET possono utilizzare policy e store del sistema operativo |
| [JSSE](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) | API Java `SSLSocket`, `SSLEngine`, `SSLContext` e Provider | KeyStore, TrustStore, `cacerts`, PKCS #12 | un’applicazione JVM può differire dalla fiducia del sistema operativo |
| [GnuTLS](https://www.gnutls.org/documentation.html) | libreria TLS nativa, API C e CLI | file, PKCS #11, trust di sistema secondo l’integrazione | libreria alternativa in software Unix e GNU |

Sopra questo livello si trova la configurazione dell’applicazione. Un mail server può selezionare certificato e policy per listener o nome SNI; un reverse proxy termina TLS davanti al backend; un prodotto Java può fornire un proprio JRE incluso il truststore; un’immagine container può congelare un CA bundle al momento della build. «Il certificato installato» a livello di sistema operativo non è quindi un’affermazione affidabile senza prova del processo e del binding.

Anche il supporto degli algoritmi è l’intersezione di versione del protocollo, libreria, crypto provider, certificato, offerta del client e policy locale. Un certificato ECDSA non aiuta se il client non supporta la firma; una stringa Cipher TLS 1.3 non modifica le suite TLS 1.2; un’impostazione Schannel a livello di sistema non controlla un runtime OpenSSL o Java fornito separatamente. [BCP 195](https://www.rfc-editor.org/info/bcp195) offre la linea guida di sicurezza IETF, ma l’attivazione concreta dipende dall’implementazione e dal ciclo di vita.

## Diagnosi dalla porta all’applicazione

Una diagnosi TLS non inizia quindi dal certificato, ma dall’esatto percorso di connessione. Ogni test deve includere nome di destinazione, indirizzo risolto, porta, SNI, modalità STARTTLS, libreria client, truststore, momento e policy applicabile. Solo test con gli stessi parametri sono confrontabili tra loro.

### Verificare separatamente la raggiungibilità TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP-Erreichbarkeit vor TLS">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection directory.example.ch -Port 636 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz mail.example.ch 25
nc -vz directory.example.ch 636</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) verificano qui solo la risoluzione dei nomi e la creazione TCP. Un test riuscito non dice ancora nulla su versione TLS, certificato, SNI o applicazione.

### Testare SMTP STARTTLS con SNI produttivo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mail.example.ch:25 -starttls smtp `
  -servername mail.example.ch -verify_hostname mail.example.ch `
  -showcerts -verify_return_error</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mail.example.ch:25 -starttls smtp \
  -servername mail.example.ch -verify_hostname mail.example.ch \
  -showcerts -verify_return_error</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) è un client diagnostico, non una prova della policy dell’MTA produttivo. `-starttls smtp` esegue il dialogo in chiaro fino all’upgrade, `-servername` imposta SNI, `-showcerts` mostra l’elenco dei certificati inviati dal server e `-verify_return_error` interrompe in caso di errore di convalida. Per un test completo dell’identità occorre indicare inoltre `-verify_hostname mail.example.ch` e il percorso CA effettivamente utilizzato. OpenSSL non è un componente di sistema integrato in Windows; `openssl.exe` deve provenire da un’installazione controllata.

### Distinguere LDAP StartTLS e TLS implicito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für LDAP-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect directory.example.ch:636 `
  -servername directory.example.ch -verify_hostname directory.example.ch `
  -verify_return_error
openssl.exe s_client -connect directory.example.ch:389 -starttls ldap `
  -servername directory.example.ch -verify_hostname directory.example.ch `
  -verify_return_error</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect directory.example.ch:636 \
  -servername directory.example.ch -verify_hostname directory.example.ch \
  -verify_return_error
openssl s_client -connect directory.example.ch:389 -starttls ldap \
  -servername directory.example.ch -verify_hostname directory.example.ch \
  -verify_return_error</code></pre>
  </div>
</div>

La prima chiamata si aspetta TLS immediato su 636, la seconda esegue su 389 la LDAP StartTLS Extended Operation. Un test OpenSSL riuscito non dimostra ancora che la libreria LDAP produttiva utilizzi lo stesso truststore, la stessa policy di revoca o la stessa identità di riferimento.

### Esaminare localmente file e percorso del certificato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">certutil.exe -dump server.cer
certutil.exe -verify -urlfetch -sslpolicy mail.example.ch server.cer
certutil.exe -store My</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl x509 -in server.pem -noout -subject -issuer -serial -dates \
  -fingerprint -sha256 -ext subjectAltName,extendedKeyUsage
openssl verify -show_chain -CAfile roots.pem \
  -untrusted intermediates.pem server.pem</code></pre>
  </div>
</div>

[`certutil`](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) utilizza la logica dei certificati Windows per `-verify` e può includere un nome del server con `-sslpolicy`. [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) mostra i campi di un certificato; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) costruisce e verifica un percorso rispetto alle Trust Anchors e agli Intermediate specificati esplicitamente. [`keytool`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) è l’interfaccia ufficiale adatta per la gestione e l’ispezione di Key Store e truststore JVM; un test con OpenSSL non sostituisce questa prospettiva.

### Assegnare il sintomo di errore a un passaggio di verifica

| Osservazione | Livello probabile | Verifica successiva |
|---|---|---|
| Timeout TCP | DNS, route, firewall o listener | indirizzo risolto, SYN/SYN-ACK, percorso di ritorno, binding del listener |
| Banner in chiaro invece di risposta TLS | modello di avvio errato | porta e `-starttls smtp` oppure `-starttls ldap` |
| Alert TLS immediato dopo ClientHello | nessun parametro comune o policy | versioni, gruppi, algoritmi di firma, Cipher, SNI |
| certificato errato | SNI o binding del listener | SNI inviato, VHost predefinito, configurazione del Load Balancer |
| `unable to get local issuer certificate` | Intermediate o Trust Anchor mancante | elenco del server, AIA, truststore client, costruzione del percorso |
| `hostname mismatch` | ID di riferimento non corrisponde al SAN | nome effettivamente usato, tipo SAN, alias e limite wildcard |
| fallisce solo l’applicazione Java | policy JSSE o truststore proprio | JVM, `cacerts`, truststore dell’app, disabledAlgorithms |
| solo singoli nodi del cluster distribuiscono il vecchio certificato | rollout o binding | testare ogni endpoint e ogni IP con lo stesso SNI |
| SMTP torna talvolta al chiaro | policy opportunistica o downgrade | log MTA, offerta STARTTLS, stato MTA-STS/DANE, TLS-RPT |
| handshake mTLS riesce, accesso resta negato | autorizzazione dopo TLS | EKU client, mapping del certificato, inoltro proxy, ruolo applicativo |

[`Wireshark`](https://www.wireshark.org/docs/wsug_html/index.html) può rendere visibili ClientHello, ServerHello, estensioni e Alerts; senza chiavi di sessione, i dati applicativi e molti messaggi di handshake TLS 1.3 restano cifrati. Inoltre, una cattura non mostra automaticamente quale truststore utilizza l’applicazione o perché la sua policy rifiuta un percorso.

## Monitoraggio e criteri operativi

Un controllo della porta e un avviso prima della data di scadenza non sono sufficienti. Il monitoraggio dovrebbe verificare lo stesso percorso di un client reale: selezionare la destinazione, stabilire l’handshake, convalidare certificato e nome e infine raggiungere il protocollo applicativo. Sono utili in particolare i seguenti segnali:

- selezione della destinazione e SNI per servizio, listener e nodo del cluster;
- protocollo, Cipher Suite, gruppo e algoritmo di firma negoziati;
- fingerprint Leaf, numero di serie, SAN, Issuer nonché `notBefore` e `notAfter`;
- catena Intermediate effettivamente inviata e convalida del percorso dal punto di vista dei truststore rilevanti;
- raggiungibilità OCSP/CRL e comportamento reale fail-open/fail-closed;
- latenza dell’handshake, quota di resumption, classi di Alert e tasso di errori;
- offerta STARTTLS, upgrade riusciti e fallback in chiaro per ciascuna destinazione SMTP;
- stato di MTA-STS, DANE e TLS-RPT dove sono impiegati questi meccanismi di policy;
- uniformità di tutti i terminatori TLS dopo il rinnovo e test applicativo riuscito dietro il livello TLS.

Il monitoraggio solo dall’interno non vede la catena pubblica, il DNS esterno o i terminatori edge; un controllo solo esterno non vede truststore interni e mTLS. Per le piattaforme di messaggistica e directory sono quindi necessari almeno un percorso client esterno, un percorso applicativo interno e una verifica diretta di ogni terminatore.

## Storia tecnica

La storia inizia con SSL di Netscape negli anni Novanta. SSL 2.0 è stato in seguito espressamente vietato ([RFC 6176](https://datatracker.ietf.org/doc/html/rfc6176)); SSL 3.0 è documentato ormai solo come protocollo storico e non è più ammesso dal RFC 7568. L’IETF ne sviluppò **TLS 1.0** nel 1999, **TLS 1.1** nel 2006 e **TLS 1.2** nel 2008. RFC 8996 ha dichiarato obsoleti TLS 1.0 e 1.1 nel 2021. Per gli amministratori, questa discendenza spiega perché vecchi nomi di prodotti, sintassi Cipher e interruttori di compatibilità compaiano ancora oggi nelle configurazioni.

**TLS 1.3** è apparso nel 2018 con RFC 8446. Ha rimosso lo scambio statico di chiavi RSA, le vecchie procedure non AEAD e altre funzioni legacy, ha cifrato prima l’handshake e ha introdotto il piano delle chiavi basato su HKDF e il modello di ripresa PSK. RFC 9846 ha sostituito RFC 8446 nel luglio 2026 come specifica TLS 1.3 consolidata e ha integrato le correzioni e i requisiti fino ad allora documentati separatamente. Il valore di protocollo `1.3` è rimasto lo stesso; è cambiato il documento standard normativo ([RFC 9846, sezione 1.2](https://datatracker.ietf.org/doc/html/rfc9846#section-1.2)).

Parallelamente si è evoluto il lato dell’identità e dell’operatività: X.509-PKIX è stato consolidato in RFC 5280, SNI e altre estensioni in RFC 6066, ALPN in RFC 7301, l’emissione automatizzata in ACME/RFC 8555 e la moderna verifica dell’identità del servizio in RFC 9525. La storia di TLS non è quindi solo una sequenza di versioni, ma lo sviluppo congiunto di crittografia, percorsi dei certificati, integrazione applicativa e automazione operativa.

## Fonti

- [IETF RFC 8314 – Cleartext Considered Obsolete](https://datatracker.ietf.org/doc/html/rfc8314)
- [RFC 9846 – The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc9846) – Handshake, Record Layer, piano delle chiavi, ripresa e modello di sicurezza.
- [RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207) – STARTTLS, cambiamento di stato SMTP e policy opportunistica.
- [RFC 9001 – Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001) – distinzione tra handshake TLS e protezione dei pacchetti QUIC.
- [RFC 5280 – Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280) – campi dei certificati, estensioni e convalida del percorso.
- [RFC 7468 – Textual Encodings of PKIX, PKCS, and CMS Structures](https://datatracker.ietf.org/doc/html/rfc7468) – involucro testuale PEM.
- [RFC 7292 – PKCS #12](https://datatracker.ietf.org/doc/html/rfc7292) – contenitore per chiavi e certificati.
- [RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960) – verifica dello stato OCSP.
- [RFC 9525 – Service Identity in TLS](https://datatracker.ietf.org/doc/html/rfc9525) – identità di riferimento, subjectAltName, wildcard e verifica dei nomi.
- [RFC 6066, sezione 3](https://datatracker.ietf.org/doc/html/rfc6066)
- [RFC 7301 – TLS Application-Layer Protocol Negotiation](https://datatracker.ietf.org/doc/html/rfc7301) – negoziazione ALPN.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – porte e nomi di servizio registrati.
- [RFC 4511, sezione 4.14](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 7672 – SMTP Security via DANE TLS](https://datatracker.ietf.org/doc/html/rfc7672) – autenticazione SMTP basata su DNSSEC/TLSA.
- [RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461) – policy MTA-STS e verifica MX.
- [RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460) – report aggregati sugli errori TLS.
- [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) – gestione delle chiavi e necessità di protezione.
- [RFC 8555 – Automatic Certificate Management Environment](https://datatracker.ietf.org/doc/html/rfc8555) – emissione e rinnovo automatizzati.
- [OpenSSL](https://docs.openssl.org/master/)
- [Codice sorgente](https://github.com/openssl/openssl)
- [Microsoft Learn – Schannel SSP](https://learn.microsoft.com/windows-server/security/tls/tls-ssl-schannel-ssp-overview) – implementazione TLS Windows e classificazione SSPI.
- [Oracle – JSSE Reference Guide](https://docs.oracle.com/en/java/javase/24/security/java-secure-socket-extension-jsse-reference-guide.html) – API TLS Java, KeyManager e TrustManager.
- [GnuTLS Documentation](https://www.gnutls.org/documentation.html) – libreria TLS nativa alternativa e famiglia di strumenti.
- [BCP 195 – Recommendations for Secure Use of TLS and DTLS](https://www.rfc-editor.org/info/bcp195) – linea guida di sicurezza IETF aggiornata per la configurazione TLS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – raggiungibilità TCP in Windows.
- [OpenBSD Manual – nc](https://man.openbsd.org/nc) – test di connessione TCP in Unix.
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/) – client diagnostico generico TLS e STARTTLS.
- [Microsoft Learn – certutil](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil) – store certificati Windows e Chain Engine.
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/) – campi del certificato e opzioni di output.
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/) – costruzione e convalida esplicite del percorso.
- [Oracle – keytool](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html) – gestione di Key Store e truststore Java.
- [Wireshark User’s Guide](https://www.wireshark.org/docs/wsug_html/index.html) – analisi di handshake, record e chiavi di sessione.
- [RFC 6176 – Prohibiting SSL Version 2.0](https://datatracker.ietf.org/doc/html/rfc6176) – dismissione formale di SSL 2.0.
