---
title: "Kerberos: ticket, chiavi e servizi attendibili"
blatt: "kerberos"
description: "Kerberos per amministratori: KDC, flusso AS/TGS/AP, principal e SPN, ticket, chiavi di sessione, keytab e KVNO, GSS-API, Active Directory e PAC, trust, delega, crittografia, DNS, tempo, operatività e diagnosi."
fakten:
  - label: Funzione
    wert: Autenticazione di rete tramite ticket
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.1
  - label: Versione del protocollo
    wert: Kerberos V5 · RFC 4120
    href: https://datatracker.ietf.org/doc/html/rfc4120
  - label: Autorità di fiducia
    wert: KDC con Authentication Service e TGS
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.2
  - label: Scambio
    wert: AS · TGS · AP client/server
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3
  - label: Porta KDC
    wert: 88 su UDP e TCP
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos
  - label: Servizio password
    wert: kpasswd · Port 464
    href: https://datatracker.ietf.org/doc/html/rfc3244
  - label: Nome del servizio
    wert: service/host@REALM
    href: https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html
  - label: Chiave del servizio
    wert: Principal · KVNO · Enctype · Key
    href: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
  - label: Finestra temporale
    wert: Clock Skew è una policy del realm e del client
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3
  - label: API applicativa
    wert: GSS-API / SSPI · spesso SPNEGO
    href: https://datatracker.ietf.org/doc/html/rfc4121
  - label: Active Directory
    wert: I controller di dominio integrano il KDC
    href: https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview
  - label: Autorizzazione AD
    wert: PAC come Authorization Data del ticket
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
translationSourceHash: b5eccfbd21dfaa9b065424961e28940ca63617bba9dbb9f5ab68c1db128df31f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:08:55.924Z
translationReview: required
---

# Kerberos: ticket, chiavi e servizi attendibili

Kerberos consente a un utente o a un processo di autenticarsi presso un servizio di rete senza inviare la password al servizio. A tale scopo, client e server si fidano di un Key Distribution Center (KDC), che rilascia ticket e chiavi di sessione a tempo limitato. In caso di accesso basato su password, dalla password viene comunque derivata una chiave a lungo termine; la qualità della password, la pre-autenticazione e la protezione del client restano quindi rilevanti per la sicurezza ([RFC 4120, sezioni 1.1 e 1.2](https://datatracker.ietf.org/doc/html/rfc4120#section-1.1), [RFC 3961, sezione 3](https://datatracker.ietf.org/doc/html/rfc3961#section-3)).

Per gli amministratori di messaggistica e directory, Kerberos si affianca spesso a [LDAP](/kb/ldap), non lo sostituisce. LDAP trasporta operazioni di directory; Kerberos può fornire l'identità per una sessione LDAP tramite SASL. Le interfacce web usano HTTP Negotiate, i servizi Windows SSPI, le applicazioni Unix di norma GSS-API. Un errore può quindi derivare da DNS, orario, raggiungibilità del KDC, mappatura del realm, cache dei ticket, SPN, account di servizio, keytab, tipo di crittografia, PAC o policy di delega, anche se l'applicazione segnala soltanto «Integrated Authentication failed».

La spiegazione segue un principal dal primo contatto con il KDC, attraverso TGT e service ticket, fino al servizio di destinazione. Solo in seguito vengono approfondite le estensioni di Active Directory, la delega, la dipendenza dal tempo, la diagnosi e il recovery.

## Approccio architetturale: terza istanza attendibile

Kerberos distribuisce chiavi simmetriche tramite un'istanza attendibile. Il KDC accede alle chiavi a lungo termine dei principal del proprio realm e riunisce logicamente due servizi: l'Authentication Service, AS, rilascia un Ticket-Granting Ticket, TGT; il Ticket-Granting Service, TGS, scambia questo TGT con un ticket per uno specifico servizio applicativo. Lo scambio client/server, AP, avviene successivamente tra client e servizio. Un servizio può normalmente verificare un service ticket con la propria chiave, senza richiamare nuovamente il KDC a ogni accesso ([RFC 4120, sezioni 1.2 e 3](https://datatracker.ietf.org/doc/html/rfc4120#section-1.2)).

Questo modello riduce la trasmissione delle password e le verifiche online centralizzate, ma crea chiari domini di guasto. Senza un KDC raggiungibile non possono essere rilasciati nuovi ticket; i ticket già presenti possono continuare a funzionare fino alla fine della loro validità. Un database KDC compromesso o una chiave del realm compromettono invece la catena di fiducia dell'intero realm. Kerberos non è quindi una piattaforma stateless di firma di token, bensì un sistema distribuito costituito da stato del KDC, cache dei client, chiavi dei servizi, orologi e servizi dei nomi.

| Livello | Componente tecnica | Stato responsabile | Evidenza amministrativa |
|---|---|---|---|
| Applicazione | HTTP, SMB, LDAP, database, SMTP/IMAP con SASL o servizio proprietario | Sessione e autorizzazione dell'applicazione | metodo di autenticazione e nome di destinazione effettivamente scelti |
| API di integrazione | GSS-API, Windows SSPI e spesso SPNEGO | Security Context, delega e Channel Binding | meccanismo negoziato, initiator, acceptor e flag |
| Kerberos AP | `KRB_AP_REQ`, opzionale `KRB_AP_REP`, eventualmente GSS Wrap/MIC | Service ticket, authenticator e session key | principal di destinazione, tempi del ticket, enctype e autenticazione reciproca |
| Kerberos KDC | Scambio AS e TGS | Database dei principal, chiavi del realm, policy e flag dei ticket | risultato KDC, chiave selezionata, KVNO ed evento di audit |
| Discovery e trasporto | DNS SRV, UDP/TCP 88, servizio password 464 | Cache del resolver, mapping del realm, routing e dimensione dei pacchetti | KDC effettivamente scelto, trasporto, tempo di risposta e fallback |
| Persistenza | AD DS o database Kerberos, keytab, cache delle credenziali e di replay | Chiavi a lungo termine, ticket, PAC, replica e stato di replay | stato della replica, contenuto della cache, permessi dei file e procedura di recovery |

Kerberos autentica i principal e può fornire integrità o riservatezza per un contesto di sicurezza GSS. Non cifra automaticamente l'intero flusso di dati utili di un'applicazione e non offre Perfect Forward Secrecy per le chiavi di sessione distribuite nel nucleo Kerberos. Le applicazioni possono usare Kerberos per autenticare un canale protetto separatamente; [TLS](/kb/tls) resta quindi un livello distinto con propria identità server e verifica del certificato per HTTPS o LDAPS ([RFC 4120, sezione 10](https://datatracker.ietf.org/doc/html/rfc4120#section-10), [RFC 4121, sezioni 2 e 4](https://datatracker.ietf.org/doc/html/rfc4121#section-2)).

## Principal, realm e materiale delle chiavi

Un principal Kerberos è un nome all'interno di un realm. Gli utenti sono spesso rappresentati come `alice@EXAMPLE.CH`, i servizi come `HTTP/intranet.example.ch@EXAMPLE.CH`. La parte prima di `@` può contenere più componenti; nel modello di denominazione Kerberos le maiuscole/minuscole sono fondamentalmente rilevanti. Un nome di dominio DNS e un realm Kerberos sono spazi dei nomi diversi, anche se i realm hanno di norma l'aspetto di domini DNS in maiuscolo. I client necessitano quindi di un'associazione verificabile da hostname o dominio DNS a realm ([RFC 4120, sezioni 6.1 e 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-6.1), [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html#mapping-hostnames-onto-kerberos-realms)).

Le chiavi a lungo termine appartengono ai principal. Le chiavi di sessione sono generate per uno scambio limitato nel tempo. Un ticket contiene tra l'altro client, server, realm, campi temporali, flag, session key e Authorization Data; la sua parte cifrata è protetta con una chiave del servizio di destinazione. Il client riceve la stessa session key in una parte della risposta protetta per lui. Può quindi trasportare il contenuto del ticket, ma non modificare autonomamente la parte cifrata per il servizio ([RFC 4120, sezioni 5.3 e 5.4](https://datatracker.ietf.org/doc/html/rfc4120#section-5.3)).

Il Key Version Number, KVNO, distingue le generazioni di una chiave principal. L'Encryption Type, Enctype, definisce algoritmo, lunghezza della chiave, string-to-key e checksum. Un servizio può mantenere più voci keytab per lo stesso principal con KVNO o enctype diversi, per gestire una rotazione controllata. Se KVNO ed enctype del ticket non corrispondono ad alcuna chiave di servizio disponibile, il servizio non può decifrare il ticket. «SPN presente» non dimostra quindi ancora che sul sistema di destinazione sia disponibile una chiave adatta ([RFC 4120, sezione 5.2.9](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.9), [MIT Kerberos: Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)).

| Oggetto | Posizione | Riservatezza | Limite operativo |
|---|---|---|---|
| chiave principal a lungo termine | Database KDC; presso il servizio anche keytab o key store del sistema operativo | altamente critica | Rotazione, replica, KVNO ed enctype consentiti |
| TGT | Credential cache del client; parte cifrata del ticket per `krbtgt/REALM` | simile a bearer più session key | Lifetime, flag Forwardable/Renewable, isolamento della cache |
| Service ticket | Credential cache e successivamente presso il servizio di destinazione | decifrabile solo dal servizio nominato | SPN, account di servizio, host di destinazione, PAC ed enctype del ticket |
| Authenticator | Nuovo a ogni AP Request, protetto con session key | Protezione dal replay | Ora del client, subkey, sequenza e replay cache lato server |
| Keytab | File o altro keytab store presso il servizio | da proteggere come una password non interattiva | Permessi del file, distribuzione, inventario, rotazione e cancellazione sicura |
| Replay cache | Stato locale dell'acceptor | Stato di integrità | Da verificare per istanza del servizio, host e architettura del cluster |

Active Directory memorizza gli SPN nell'attributo multivalore `servicePrincipalName` di un account utente o computer. Lo SPN collega il nome del servizio composto dal client all'esatto account la cui chiave il KDC usa per il service ticket. Un alias, il nome di un load balancer o un account di servizio modificato richiedono quindi un'associazione SPN consapevole. Gli SPN duplicati sono ambigui nel rispettivo ambito di ricerca; uno SPN sull'account errato porta tipicamente al fatto che il servizio non possa decifrare il ticket rilasciato ([Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Una keytab non è un file di esportazione con un'identità ripristinabile arbitrariamente, bensì una raccolta di chiavi reali a lungo termine. Ogni voce contiene principal, KVNO, enctype e chiave. Chi può leggere il file può impersonare quel principal. MIT raccomanda archiviazione locale e restrittiva e nessun trasferimento non protetto. Nell'interoperabilità con Active Directory, [`ktpass`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) può collegare principal, account e keytab; i parametri scelti possono influire su password, salt, KVNO o enctype e devono far parte di un processo di rotazione testato, non di una nota di installazione una tantum ([MIT Kerberos: Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-kerberos.svg?v=20260813" title="Interaktive Infografik: Kerberos-Discovery, AS-, TGS- und AP-Fluss, Tickets, Schlüssel, Active-Directory-PAC und Delegationsgrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-kerberos.svg?v=20260813">Apri l'infografica sul flusso del protocollo Kerberos e sui suoi confini di fiducia</a>
</iframe>

Chiariti principal, realm e chiavi, il flusso dei ticket può essere letto come tre conversazioni successive: AS, TGS e infine il servizio applicativo.

## Flusso del protocollo: AS, TGS e AP

Lo scambio AS inizia con `KRB_AS_REQ`. Un KDC può rispondere con `KDC_ERR_PREAUTH_REQUIRED` e indicare i metodi di pre-autenticazione supportati. Con il diffuso timestamp cifrato, il client dimostra di conoscere la propria chiave a lungo termine prima che il KDC rilasci un TGT. `KRB_AS_REP` contiene il TGT, cifrato per il TGS, e una parte della risposta per il client. PKINIT sostituisce questa prima prova con crittografia a chiave pubblica e certificati; Kerberos FAST può rafforzare la pre-autenticazione in un tunnel protetto ([RFC 4120, sezioni 3.1 e 5.2.7](https://datatracker.ietf.org/doc/html/rfc4120#section-3.1), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113, sezione 5](https://datatracker.ietf.org/doc/html/rfc6113#section-5)).

Nello scambio TGS, il client invia `KRB_TGS_REQ` con il TGT, un authenticator e il service principal richiesto. Il KDC verifica policy del realm, flag del ticket, principal di destinazione e chiavi supportate e fornisce in `KRB_TGS_REP` un service ticket più una nuova client/service session key. La password dell'utente non è più necessaria. Un TGT già presente può quindi essere usato per molti servizi, finché lifetime, policy o stato della cache non richiedono una nuova autenticazione iniziale ([RFC 4120, sezione 3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3)).

Nello scambio AP, il client presenta al servizio `KRB_AP_REQ`: il service ticket e un authenticator nuovo, protetto con la session key. Il servizio decifra il ticket con la propria chiave a lungo termine, verifica principal di destinazione, tempi, flag e stato di replay e ne ricava la session key. Se il client richiede l'autenticazione reciproca, il servizio risponde con `KRB_AP_REP`. Solo questa risposta dimostra crittograficamente al client che la controparte possiede la chiave del servizio ([RFC 4120, sezioni 3.2 e 5.5](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2)).

| Scambio | Request | Risposta di successo | Input critici | Tipico confine di errore |
|---|---|---|---|---|
| AS | `KRB_AS_REQ` | `KRB_AS_REP` con TGT | Client principal, realm, pre-auth, enctype consentiti | account sconosciuto, pre-auth, orario, chiave client mancante |
| TGS | `KRB_TGS_REQ` | `KRB_TGS_REP` con service ticket | TGT, authenticator, SPN, flag, chiavi di destinazione | SPN, referral del realm, policy, delega, enctype |
| AP | `KRB_AP_REQ` | opzionale `KRB_AP_REP` | Service ticket, authenticator, chiave di servizio, replay cache | account di servizio errato, keytab/KVNO, orario, replay |
| Errore | una delle request | `KRB_ERROR` | Codice di errore più e-data e ora del server opzionali | mantenere codice numerico e fase del protocollo interessata |

La pre-autenticazione non protegge da ogni attacco offline, ma modifica il presupposto. Gli account senza pre-autenticazione obbligatoria possono fornire una risposta AS la cui parte protetta da password può essere verificata offline. Anche i service ticket sono analizzabili offline; password deboli degli account di servizio e RC4 aggravano questo rischio. La protezione efficace consiste in pre-autenticazione, chiavi di servizio casuali robuste o gMSA, enctype moderni, diritti limitati e dati di audit, non nella mera presenza di Kerberos ([RFC 6113, sezione 1](https://datatracker.ietf.org/doc/html/rfc6113#section-1), [RFC 8429, sezione 5](https://datatracker.ietf.org/doc/html/rfc8429#section-5), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

## Ticket, tempi, flag e trasporto

Un ticket possiede `authtime`, opzionalmente `starttime`, `endtime` e, per i ticket rinnovabili, `renew-till`. Il TGT non è un token di accesso universale: è indirizzato al TGS e serve a ottenere altri ticket. Un service ticket è indirizzato a un preciso server principal. Lifetime e rinnovo sono regolati dalla policy del KDC e possono essere limitati per realm o account. Valori fissi quali «dieci ore» sono quindi default di prodotto, non una proprietà dello standard Kerberos ([RFC 4120, sezioni 2.3 e 5.3](https://datatracker.ietf.org/doc/html/rfc4120#section-2.3)).

Flag quali `forwardable`, `forwarded`, `proxiable`, `proxy`, `renewable`, `initial`, `pre-authent` e `ok-as-delegate` modificano l'utilizzabilità di un ticket. Non sono campi diagnostici decorativi. Un double hop può fallire anche se il primo service ticket è valido, perché manca un flag necessario o una policy KDC. Viceversa, un TGT forwardable amplia l'impatto di un servizio compromesso. I flag dei ticket fanno quindi parte di ogni evidenza relativa a delega e incidenti ([RFC 4120, sezione 2](https://datatracker.ietf.org/doc/html/rfc4120#section-2)).

Gli authenticator e alcuni metodi di pre-autenticazione usano il tempo per verificare la freshness. RFC 4120 lascia la deviazione consentita a una policy locale. Active Directory usa cinque minuti per impostazione predefinita; questa tolleranza non sostituisce una sincronizzazione precisa dell'ora. Sono determinanti client, KDC e servizio di destinazione: un client può ricevere un ticket e fallire comunque sul servizio con `KRB_AP_ERR_SKEW` se il suo orologio è fuori fase ([RFC 4120, sezioni 3.2.3 e 7.5.1](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance), [Microsoft: Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service)).

Kerberos usa la porta 88 su UDP e TCP. RFC 4120 richiede il supporto TCP e descrive UDP come opzionale; risposte UDP troppo grandi possono attivare un nuovo tentativo TCP con `KRB_ERR_RESPONSE_TOO_BIG`. PAC, appartenenze ai gruppi e ulteriori Authorization Data ingrandiscono i ticket. Un firewall che consente solo piccoli test UDP o blocca TCP 88 può quindi generare guasti dipendenti dall'utente o dal gruppo ([RFC 4120, sezione 7.2.1](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.1), [Microsoft: Kerberos KDC configuration keys](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys)).

Il service ticket è soltanto la prova crittografica. Affinché HTTP, LDAP o SMB possano usarlo, GSS-API, SSPI e SPNEGO incorporano Kerberos nel rispettivo protocollo applicativo.

## Integrazione nei protocolli applicativi

GSS-API fornisce alle applicazioni un Security Context indipendente dal meccanismo; RFC 4121 definisce a tale scopo il meccanismo Kerberos V5. Windows SSPI svolge un ruolo comparabile. SPNEGO negozia tra i meccanismi GSS proposti e in HTTP appare spesso come `Negotiate`. Il nome dell'header visibile non prova automaticamente che sia stato scelto Kerberos anziché NTLM. La diagnosi deve rilevare il meccanismo effettivamente negoziato e il Target Name richiesto ([RFC 4121](https://datatracker.ietf.org/doc/html/rfc4121), [RFC 4178](https://datatracker.ietf.org/doc/html/rfc4178), [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

SASL GSS-API integra lo stesso meccanismo Kerberos nei protocolli applicativi. Ciò è possibile, ad esempio, con [LDAP](/kb/ldap), IMAP o SMTP, se client e server offrono il meccanismo. Dopo un contesto GSS riuscito, i Security Layer SASL possono fornire integrità o riservatezza. Se vengano effettivamente usati e come interagiscano con TLS dipende dalla configurazione della specifica applicazione; `GSSAPI` nell'elenco delle capability da solo non dimostra né SPN né Channel Protection ([RFC 4752](https://datatracker.ietf.org/doc/html/rfc4752)).

Il client forma il nome del servizio da service class e host di destinazione. Per HTTP è tipicamente rilevante `HTTP/fqdn`, per LDAP `ldap/fqdn`. Alias, CNAME, reverse lookup, proxy, nome del cluster e hostname nell'URL possono modificare l'identità composta. Il client deve richiedere il ticket per lo stesso principal la cui chiave possiede il servizio accettante. Un load balancer non risolve questo vincolo; tutte le istanze backend necessitano di un'identità di servizio e di una strategia delle chiavi coerenti ([MIT Kerberos: Application servers – DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html#getting-dns-information-correct), [Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)).

## Active Directory come realm Kerberos

In Active Directory Domain Services, il KDC è integrato nel controller di dominio. Account utente, computer e servizio sono principal Kerberos; la directory fornisce chiavi, SPN, gruppi, account flag e policy. L'account `krbtgt` rappresenta il TGS del dominio. Disponibilità del KDC e coerenza dei dati KDC seguono quindi DC Locator, DNS, replica AD, siti e modello di recovery di AD DS, non un protocollo cluster Kerberos separato ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [MS-KILE: Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13), [DC Locator](/kb/ldap#active-directory-als-ldap-serverprofil)).

AD aggiunge normalmente ai ticket un Privilege Attribute Certificate, PAC, come Authorization Data. Può contenere tra l'altro SID, appartenenze ai gruppi, informazioni di profilo e policy, nonché firme. Il KDC genera e firma questi dati; il servizio li usa per l'autorizzazione Windows o eventualmente li fa convalidare. Autenticazione Kerberos di base e autorizzazione AD sono quindi affermazioni distinte: un principal crittograficamente valido non possiede automaticamente l'autorizzazione desiderata ([MS-PAC](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962), [MS-KILE: PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909)).

Il materiale delle chiavi e gli attributi SPN vengono replicati con Active Directory. Dopo modifiche a account, password o SPN, diversi DC possono quindi usare temporaneamente stati diversi. Se il client usa un DC per TGS e l'amministratore un altro DC per il controllo, un errore appare intermittente. Un'evidenza affidabile indica KDC emittente, DC di destinazione della query di directory, KVNO, enctype del ticket e stato della replica. Ciò vale in particolare per rotazioni manuali di keytab e servizi distribuiti tra siti.

La selezione dell'enctype è l'intersezione tra offerta del client, policy KDC, chiavi dell'account di destinazione e supporto del servizio. Un software compatibile con AES non è sufficiente se l'account non possiede le chiavi corrispondenti o se `msDS-SupportedEncryptionTypes` e la policy di dominio le escludono. RFC 8429 classifica RC4 e 3DES per Kerberos come obsoleti; Microsoft documenta l'inventario mediante gli eventi di sicurezza 4768 e 4769. Una migrazione inizia con misurazione e generazione delle chiavi, non con la disattivazione globale di un bit ([RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

Per i servizi Windows, i Group Managed Service Accounts, gMSA, riducono la gestione manuale di password e SPN. Non sostituiscono però la verifica dell'identità sotto cui il processo è effettivamente in esecuzione, degli host autorizzati a leggere la managed password e degli SPN registrati sull'account. Per appliance o servizi Unix è spesso ancora necessaria una keytab; la sua rotazione deve essere sincronizzata con l'account AD ([Microsoft: Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)).

All'interno di un dominio AD questo percorso è diretto. Per l'accesso oltre confini di realm o dominio si aggiungono ticket cross-realm ed eventualmente più stazioni intermedie.

## Trust e percorsi cross-realm

Un trust tra realm non fonde i database dei principal. L'autenticazione cross-realm usa TGT per `krbtgt/TARGET@SOURCE` ed eventualmente più realm intermedi. Il client segue un percorso di ticket fino a raggiungere il realm del servizio di destinazione. RFC 6806 aggiunge referral e canonicalizzazione dei nomi, come usati in particolare dagli ambienti AD. Il percorso visibile nella cache è quindi più significativo dell'affermazione generica «il trust è verde» ([RFC 4120, sezioni 1.1 e 3.3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3.3), [RFC 6806](https://datatracker.ietf.org/doc/html/rfc6806)).

Direzione del trust, transitività, Selective Authentication, filtraggio SID, Name Suffix Routing ed enctype disponibili limitano ciò che un TGT cross-realm consente in pratica. Gli SPN devono essere individuabili e univoci nella foresta corretta. Un riscontro locale con `setspn -Q` non dimostra l'univocità tra foreste; [`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) dispone a questo scopo di opzioni forest e domain. La diagnosi documenta ogni referral TGT, non solo l'ultimo errore del service ticket ([Microsoft: Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

Un ticket verso il frontend non autorizza automaticamente quest'ultimo a contattare un backend a nome dell'utente. Qui inizia esattamente il problema del double hop.

## Delega e double hop

Nel normale scambio AP, un frontend non riceve una chiave utente liberamente utilizzabile. Se deve accedere a un backend a nome dell'utente, necessita di un modello di delega. La delegated TGT, ovvero unconstrained delegation, consegna al servizio un TGT riutilizzabile e ne amplia l'impatto ben oltre un singolo backend. Un frontend compromesso può usarlo per ottenere ticket verso altri servizi; questa forma è quindi una grande estensione della fiducia ([RFC 4120, sezioni 2.5 e 2.6](https://datatracker.ietf.org/doc/html/rfc4120#section-2.5), [MS-SFU: Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a)).

Le estensioni Service-for-User di Microsoft suddividono il problema. Con S4U2self un servizio può ottenere un ticket verso sé stesso a nome di un utente, ad esempio dopo un'altra autenticazione frontend. Con S4U2proxy richiede, nel rispetto della policy KDC, un ticket verso un secondo servizio a nome di tale utente. La constrained delegation classica memorizza le destinazioni consentite sull'account frontend; la resource-based constrained delegation registra i chiamanti consentiti sull'account della risorsa. In entrambi i casi, SPN di destinazione, flag del ticket, impostazioni dell'account e percorso di trust fanno parte della decisione ([MS-SFU: Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf), [MS-SFU: Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d)).

La delega è autorizzazione alla propagazione dell'identità, non una mera opzione di compatibilità. L'amministratore inventaria principal frontend, SPN backend, classi di utenti consentite, Protocol Transition, confini di trust e l'ambito tecnico minimo di destinazione. Un primo hop riuscito non dimostra il secondo; viceversa, un test backend diretto nel contesto utente aggira il confine della delega e può fornire un falso positivo.

## Modello operativo, monitoraggio e recovery

Dopo flusso dei ticket e delega sorge la questione operativa: quale componente Kerberos deve essere disponibile, quale evidenza ne dimostra lo stato e cosa può essere ripristinato in caso di emergenza? La tabella associa queste domande ai ruoli coinvolti.

| Ruolo | Stato da monitorare | Metrica o evidenza guida | Punto cieco frequente |
|---|---|---|---|
| Client | Mapping del realm, selezione KDC, orologio e credential cache | Latenza AS/TGS, lifetime della cache, KDC e codice di errore | Il test usa un altro nome DNS o un'altra sessione di accesso utente |
| KDC | Chiavi principal, policy, replica, trust e audit | 4768/4769/4771, tasso di errore per codice, enctype e DC emittente | Errore complessivo senza SPN di destinazione e offerta del client |
| Servizio | Account di servizio, SPN, keytab/key store, replay cache e orologio | Successo AP, KVNO, enctype del ticket, principal di destinazione | Porta aperta, ma processo in esecuzione con un'altra identità |
| Frontend con delega | Policy S4U/forwarding e destinazioni backend | Primo e secondo hop separati, percorso di delega e flag del ticket | Il test backend diretto aggira il double hop |
| Operatività realm/foresta | Database KDC, chiavi del realm, replica AD e recovery | Stato delle chiavi replicate, configurazione salvata, restore testato | Keytab del servizio e generazione KDC divergono |

Le cache dei ticket sono stato operativo. Un processo utente, account di servizio, container o sessione di accesso Windows può vedere una cache diversa da quella della shell amministrativa interattiva. Eliminare e richiedere nuovamente un ticket è un test mirato, ma non ripara la causa sottostante relativa a SPN, chiave o replica. Prima del purge vengono acquisiti principal, SPN di destinazione, KDC, KVNO, enctype, flag e campi temporali; altrimenti scompare la migliore prova dell'errore.

Windows registra le richieste TGT con 4768, i service ticket con 4769, gli errori di pre-autenticazione con 4771 e altri errori AS con 4772, se sono attive le opportune Advanced Audit Policies. Il volume è elevato sui KDC. Il monitoraggio richiede pertanto aggregazione per Result Code, client, servizio di destinazione, DC ed enctype, nonché baseline, anziché trattare ogni TGS Request riuscita come un allarme ([Microsoft: Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration)).

Il recovery dipende dall'implementazione KDC. In AD DS, i dati KDC e `krbtgt` fanno parte del modello di System State e Forest Recovery; un file di database arbitrario o un'esportazione LDIF non costituiscono un backup Kerberos valido. Gli operatori di realm MIT autonomi devono salvaguardare insieme database KDC, materiale stash/master key, configurazione, ACL e replica. Le keytab di servizio vanno inventariate inoltre e, dopo un restore, verificate per coerenza di KVNO e chiavi ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [MIT Kerberos: Backups of secure hosts](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html)).

Per la risoluzione dei problemi, il percorso viene verificato a ritroso: nome di destinazione e SPN, service ticket presente, risposta TGS, TGT, individuazione del realm, DNS e orario.

## Strumenti di diagnosi

La diagnosi inizia dalla stessa zona di rete, con lo stesso nome di destinazione, realm, contesto utente o di servizio e la stessa sessione di accesso dell'applicazione. I dati di test usano nomi riservati. Gli output di ticket e keytab possono esporre principal e infrastruttura; il materiale delle chiavi non deve mai finire in ticket, chat o argomenti di processo.

### Individuare KDC e servizio password tramite DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Dienstsuche">
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

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostrano priorità, peso, porta e target. Successivamente devono essere verificati la risoluzione A/AAAA e la raggiungibilità di ogni target restituito. RFC 4120 definisce la discovery DNS SRV, ma consente una configurazione locale del realm; un test SRV vuoto prova pertanto un errore solo se il client concreto usa la DNS discovery ([RFC 4120, sezione 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.3), [RFC 2782](https://datatracker.ietf.org/doc/html/rfc2782)).

### Confrontare orario e raggiungibilità TCP

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Zeit- und Portprüfung">
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

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) e [`timedatectl`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) mostrano fonte e stato di sincronizzazione; [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) aggiunge dati di offset e tracking per Chrony. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) oppure [`nc`](https://man.openbsd.org/nc) dimostrano soltanto TCP 88. UDP, protocollo KDC, realm e pre-autenticazione richiedono un vero test AS/TGS.

### Richiedere e visualizzare ticket in modo mirato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Ticketprüfung">
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

Windows-[`klist`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) opera nel contesto della sessione di accesso selezionata. In MIT Kerberos, [`kdestroy`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) elimina, [`kinit`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) e [`kvno`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) ottengono, mentre [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) visualizza le credenziali. [`KRB5_TRACE`](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) rende visibili selezione KDC e percorso del protocollo. Prima di `purge` o `kdestroy` occorre documentare un ticket di errore esistente.

### Verificare reciprocamente SPN e chiave del servizio

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-SPN- und Keytabprüfung">
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

[`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) mostra l'account di destinazione AD e cerca duplicati nell'intera foresta con `-X -F`. [`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) legge SPN ed enctype dichiarati; un valore mancante ha una semantica di fallback specifica del prodotto e non deve essere interpretato genericamente come «nessun AES». MIT-[`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) mostra principal, KVNO ed enctype della keytab; [`kvno -k`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) richiede un ticket e lo convalida rispetto alla keytab indicata.

### Assegnare gli errori a una fase del protocollo

| Codice o sintomo | Fase e confine frequente | Evidenza successiva |
|---|---|---|
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | AS: client principal sconosciuto nel realm selezionato | Mapping del realm, UPN/principal, KDC emittente e replica |
| `KDC_ERR_PREAUTH_FAILED` | AS: chiave, password, certificato o metodo pre-auth rifiutato | Ora del client, tipo di pre-auth, chiave dell'account e audit KDC 4771 |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | TGS: SPN di destinazione non trovato o non risolvibile in modo univoco | SPN esatto richiesto, ricerca nell'intera foresta e realm di destinazione |
| `KDC_ERR_ETYPE_NOSUPP` | AS/TGS: nessuna intersezione comune tra enctype e chiavi | Offerta del client, policy KDC, chiavi dell'account, keytab e 4768/4769 |
| `KRB_AP_ERR_MODIFIED` | AP: il ticket non corrisponde alla chiave del servizio che risponde | Account SPN, identità del processo, keytab, KVNO, enctype e nodo backend |
| `KRB_AP_ERR_SKEW` | AS/AP: ora fuori dalla tolleranza | Ora di client, KDC e servizio, nonché rispettiva fonte dell'ora |
| `KRB_AP_ERR_TKT_EXPIRED` | AP: ticket fuori dalla finestra di validità | Cache, `endtime`, rinnovo, ora client e reinizializzazione |
| `KDC_ERR_BADOPTION` | TGS/S4U: flag, delega o policy non consentiti | Flag Forwardable, account frontend, SPN backend e configurazione della delega |
| Ticket Kerberos presente, l'applicazione usa NTLM | Negoziazione del meccanismo o Target Name | Risultato SPNEGO, URL/FQDN, policy zona/client e SPN effettivamente composto |

I codici di errore sono standardizzati in RFC 4120; Windows aggiunge contesto di stato e audit. L'automazione dovrebbe mantenere codice numerico, fase, KDC, client principal e principal di destinazione. Il testo libero da solo non è né stabile né univoco. Per l'analisi dei pacchetti, [Wireshark](https://www.wireshark.org/docs/dfref/k/kerberos.html) può filtrare per `kerberos`; le parti cifrate dei ticket restano intenzionalmente illeggibili senza chiavi appropriate ([RFC 4120, sezione 7.5.9](https://datatracker.ietf.org/doc/html/rfc4120#section-7.5.9), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)).

## Storia tecnica

Kerberos nacque nei primi anni Ottanta nell'ambito del MIT Project Athena. Il protocollo si basa concettualmente sui lavori di trusted third party di Needham e Schroeder nonché di Denning e Sacco. Le versioni da 1 a 4 furono sviluppate nell'ambiente Athena; la versione 4 fu la prima ampiamente utilizzata. Il nome rimanda a Cerbero, il guardiano dalle molte teste della mitologia greca, e rappresenta il ruolo centrale di fiducia del KDC ([RFC 4120, sezione 1](https://datatracker.ietf.org/doc/html/rfc4120#section-1), [MIT Kerberos Consortium: Documentation](https://kerberos.org/docs/index.html)).

Kerberos V5 eliminò le limitazioni della versione 4 in termini di naming, lifetime dei ticket, crittografia, cross-realm ed estendibilità. RFC 1510 standardizzò V5 nel 1993. RFC 4120 sostituì questa specifica nel 2005 con chiarimenti e una descrizione ASN.1 completa. La famiglia di protocolli fu poi ampliata modularmente, tra l'altro con PKINIT, Pre-Authentication Framework e FAST, GSS-API, referral e nuovi profili AES ([RFC 1510](https://datatracker.ietf.org/doc/html/rfc1510), [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113](https://datatracker.ietf.org/doc/html/rfc6113)).

Microsoft rese Kerberos V5 il protocollo centrale di autenticazione del dominio con Windows 2000 e lo collegò a principal AD, SPN, PAC, SSPI, trust referral ed estensioni di delega. Parallelamente, MIT Kerberos, Heimdal e altre implementazioni sono rimasti interoperabili tramite protocolli IETF e GSS-API. La crittografia si evolse da DES e successivamente RC4 a profili AES; RFC 8429 avviò nel 2018 la dismissione di 3DES e RC4. La storia tecnica spiega perché dispositivi legacy, vecchi account di servizio e trust key rendano tuttora visibili limiti di enctype, senza fissare nell'articolo uno stato effimero delle versioni di prodotto ([MS-KILE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/), [RFC 3962](https://datatracker.ietf.org/doc/html/rfc3962), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429)).

## Fonti

- [IETF RFC 3244 – Microsoft Windows 2000 Kerberos Change Password and Set Password Protocols](https://datatracker.ietf.org/doc/html/rfc3244)
- [MIT Kerberos – Mapping Hostnames onto Kerberos Realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)
- [IANA – Kerberos service names and ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos)
- [RFC 4120 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) – architettura, messaggi, ticket, flag, codici di errore, trasporto e modello di sicurezza.
- [RFC 3961, sezione 3](https://datatracker.ietf.org/doc/html/rfc3961)
- [RFC 4121 – Kerberos V5 GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4121) – contesto GSS, token, integrità e riservatezza.
- [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)
- [MIT Kerberos – Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html) – contenuto della keytab, KVNO, enctype e chiave.
- [Microsoft – Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) – univocità SPN e associazione ad account di servizio.
- [Microsoft – setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) – query SPN, registrazione e ricerca dei duplicati.
- [Microsoft – ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) – interoperabilità AD/keytab.
- [MIT Kerberos – Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html) – protezione della keytab, rotazione, orario e DNS.
- [RFC 4556 – PKINIT](https://datatracker.ietf.org/doc/html/rfc4556) – pre-autenticazione a chiave pubblica.
- [RFC 6113 – Generalized Framework for Kerberos Pre-Authentication](https://datatracker.ietf.org/doc/html/rfc6113) – framework pre-auth e FAST.
- [RFC 8429 – Deprecate 3DES and RC4 in Kerberos](https://datatracker.ietf.org/doc/html/rfc8429) – IETF Best Current Practice per enctype obsoleti.
- [Microsoft – Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) – inventario enctype, chiavi degli account ed eventi.
- [Microsoft – Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance) – codici di errore Windows, orario, SPN e double hop.
- [Microsoft – Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service) – servizio orario e dipendenza Kerberos.
- [Microsoft – Kerberos KDC registry and protocol settings](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys) – dimensione UDP, fallback TCP e diagnosi KDC.
- [RFC 4178 – SPNEGO](https://datatracker.ietf.org/doc/html/rfc4178) – negoziazione di meccanismi GSS.
- [Microsoft – Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) – architettura Windows, KDC, SSPI e autenticazione reciproca.
- [RFC 4752 – SASL GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4752) – integrazione in LDAP, IMAP, SMTP e altri protocolli SASL.
- [MS-KILE – Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13) – scambi AS, TGS e AP in Windows.
- [MS-PAC – Privilege Attribute Certificate](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962) – gruppi, SID, profili, policy e firme.
- [MS-KILE – PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909) – generazione di PAC Authorization Data.
- [RFC 8009 – AES with HMAC-SHA2 for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc8009) – enctype AES-SHA2.
- [Microsoft – Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) – gMSA, gestione password e SPN.
- [RFC 6806 – Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://datatracker.ietf.org/doc/html/rfc6806) – referral e canonicalizzazione dei nomi.
- [Microsoft – Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts) – trust, Protocol Transition e constrained delegation.
- [MS-SFU – Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a) – flussi di delega e rischio di TGT inoltrati.
- [MS-SFU – Service for User Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf) – S4U2self e S4U2proxy.
- [MS-SFU – Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d) – Protocol Transition e constrained delegation.
- [Microsoft – Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration) – eventi 4768, 4769, 4771 e 4772.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – backup AD DS/KDC.
- [MIT Kerberos – kdb5_util](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html) – database KDC e backup in MIT Kerberos.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – query DNS SRV in Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – query DNS SRV nei sistemi Unix.
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – priorità SRV, peso, porta e target.
- [Microsoft – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) – sorgente oraria Windows e diagnosi dell'offset.
- [Linux man-pages – timedatectl(1)](https://man7.org/linux/man-pages/man1/timedatectl.1.html) – stato della sincronizzazione oraria in Linux.
- [Chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html) – diagnosi di offset, sorgente e tracking.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnosi della connettività TCP in Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – test della porta TCP nei sistemi Unix.
- [Microsoft – klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) – cache ticket Windows e test dei service ticket.
- [MIT Kerberos – kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) – eliminazione della credential cache.
- [MIT Kerberos – kinit](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) – ottenimento TGT e opzioni pre-auth.
- [MIT Kerberos – kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) – service ticket e convalida keytab.
- [MIT Kerberos – klist](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) – visualizzazione di credential cache e keytab.
- [MIT Kerberos – kerberos environment](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) – `KRB5_TRACE` e variabili di cache/keytab.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) – attributi SPN ed enctype di un account di servizio.
- [Wireshark – Kerberos display filter reference](https://www.wireshark.org/docs/dfref/k/kerberos.html) – campi del protocollo e filtri di visualizzazione.
- [MIT Kerberos Consortium – Documentation](https://kerberos.org/docs/index.html) – origine nel Project Athena e storia delle versioni.
- [RFC 1510 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc1510) – specifica storica di base V5 del 1993.
- [MS-KILE – Kerberos Protocol Extensions](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/) – Kerberos Active Directory ed estensioni Microsoft.
- [RFC 3962 – AES Encryption for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc3962) – profili AES-SHA1.
