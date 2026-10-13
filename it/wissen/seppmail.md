---
title: "SEPPmail: gateway di posta, chiavi e consegna GINA"
blatt: "seppmail"
description: "SEPPmail per amministratori di messaggistica: architettura SMTP inline, pipeline Postfix e Ruleset, S/MIME, OpenPGP e GINA, stato di chiavi e certificati, integrazione directory, cluster, monitoraggio, backup e ripristino."
fakten:
  - label: Prodotto
    wert: SEPPmail Secure E-Mail Gateway
    href: https://www.seppmail.com/products/
  - label: Ruolo di sistema
    wert: Gateway SMTP nel percorso dei messaggi
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Nucleo di trasporto
    wert: Postfix MTA
    href: https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html
  - label: Base di sistema
    wert: Stack appliance basato su OpenBSD
    href: https://docs.seppmail.com/ch/20_revision-history.html
  - label: Crittografia dei messaggi
    wert: S/MIME · OpenPGP · chiavi di dominio
    href: https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html
  - label: Senza chiave del destinatario
    wert: Consegna web GINA tramite HTTPS
    href: https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html
  - label: Modello di regole
    wert: Ruleset generato · policy · Custom Commands
    href: https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html
  - label: Directory
    wert: LDAP/LDAPS per query di chiavi, identità e regole
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Interfacce web
    wert: GINA 443 · Admin 8443 · REST 8445
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Monitoraggio
    wert: Postmaster · SNMP · NRPE · Syslog
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Trasporto cluster
    wert: SSH tra appliance · NTP obbligatorio
    href: https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html
  - label: Origine
    wert: Sviluppo di prodotto svizzero dal 2001
    href: https://www.seppmail.com/de/die-seppmail-story/
werbung:
  - tools
  - newsletter
ctaThemen:
  - seppmail
translationSourceHash: f01b161a8f003ca453398dd3d393ca671b7346ad63164cd311dabb11055526fe
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:48:33.580Z
translationReview: automatic
---

# SEPPmail: gateway di posta, chiavi e consegna GINA

SEPPmail è un gateway di posta specializzato che riceve messaggi nel percorso SMTP, li elabora secondo regole centralizzate e li inoltra a un sistema di posta interno, a uno Smarthost o al successivo interlocutore Internet. Tra le modalità di elaborazione documentate figurano la firma e la cifratura S/MIME e OpenPGP, la cifratura di dominio basata su S/MIME, requisiti TLS relativi alla connessione e la consegna web GINA per destinatari privi di una chiave pubblica utilizzabile ([SEPPmail: Products](https://www.seppmail.com/products/), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

L'elaborazione centralizzata semplifica il client: gli utenti possono inviare dal proprio sistema di posta abituale, mentre il gateway cerca le chiavi, valuta le policy ed esegue le operazioni crittografiche. Al contempo, l'appliance diventa così un confine di fiducia e disponibilità. Vede il contenuto del messaggio prima della cifratura in uscita o dopo la decifratura in entrata, conserva materiale di chiavi private e, dopo l'accettazione SMTP riuscita, può diventare il nuovo punto di responsabilità per il messaggio. Un amministratore non deve quindi solo dimostrare che la «cifratura è attiva», ma deve poter tracciare l'intero percorso composto da routing, regola, scelta della chiave, metodo di consegna, coda, logging e ripristino ([RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321#section-6.1), [RFC 8551: S/MIME](https://datatracker.ietf.org/doc/html/rfc8551), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

La spiegazione segue un messaggio dall'accettazione SMTP, attraverso Ruleset e metodo di cifratura scelto, fino alla consegna o al portale GINA. Chiavi, cluster, diagnostica e ripristino vengono quindi spiegati lungo questo percorso.

SEPPmail si colloca nel percorso dei messaggi come gateway con stato. Accetta SMTP, determina policy e capacità del destinatario, seleziona S/MIME, OpenPGP, TLS o GINA e quindi inoltra. Architettura, esercizio e ripristino si orientano a questo processo.

## Approccio architetturale: gateway inline con stato

Nella configurazione più semplice, il sistema di posta interno invia messaggi in uscita tramite [SMTP](/kb/smtp) a SEPPmail; per i messaggi in entrata, il record [MX](/kb/dns#dns-im-mail--und-identitätsbetrieb) pubblico punta direttamente o tramite uno Smarthost a monte al gateway. Per ogni dominio gestito è definito un Forwarding Server interno. In uscita, l'appliance può consegnare autonomamente tramite DNS oppure utilizzare uno Smarthost SMTP definito. La matrice firewall documentata dal produttore rappresenta separatamente queste direzioni e indica TCP 25 tra server di posta, appliance, Internet o Smarthost ([SEPPmail: Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Il gateway non è un dispositivo trasparente di filtraggio dei pacchetti. Termina una sessione SMTP e ne apre una nuova per l'inoltro. Dopo `250` alla fine della fase DATA, la responsabilità della consegna è del sistema SMTP che accetta; la riuscita instaurazione della connessione o `STARTTLS` da soli non dimostrano tale passaggio ([RFC 5321, sezione 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207: SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)). Dal punto di vista operativo, risultano almeno cinque oggetti di verifica distinti: sessione SMTP in entrata, elaborazione locale, hop SMTP successivo, stato di chiavi e policy ed eventualmente stato web GINA.

Per Exchange Online, SEPPmail descrive un modello esplicito di connettori e regole di flusso della posta. Negli ambienti multi-tenant, la guida del produttore raccomanda connettori basati su certificati; più host gateway devono inoltre essere considerati nei connettori Microsoft e, se necessario, nella configurazione antispam. Anche Microsoft documenta i connettori come relazione definita di fiducia e routing, non come disattivazione generale di tutti gli altri percorsi di consegna ([SEPPmail: Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html), [Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)).

## Stack tecnologico e stati persistenti

La cronologia delle revisioni pubblicamente disponibile documenta uno stack appliance basato su OpenBSD con Postfix come MTA e, tra gli altri, Apache HTTP Server, OpenSSL o LibreSSL, GnuPG, OpenLDAP, Dovecot, Fetchmail, ClamAV, OpenSSH e Perl. Questi componenti svolgono ruoli diversi; le rispettive versioni riportate in una specifica Release Note non sono caratteristiche senza tempo del prodotto. Per l'esercizio è determinante quale interfaccia sia supportata dal produttore: parametri SMTP e TLS vengono ad esempio gestiti tramite «Mail System», mentre le «Extended Postfix MTA settings» sono esplicitamente descritte come rischiose e non sincronizzate nel cluster ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html), [SEPPmail: Mail System – Extended Postfix settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Il flusso di posta visibile è soltanto una parte dello stato. Per un inventario affidabile occorre inventariare separatamente almeno i seguenti oggetti:

| Spazio di stato | Esempi | Perché viene verificato separatamente |
|---|---|---|
| SMTP e routing | Managed Domains, Forwarding Server, Smarthost, reti relay, destinazioni TLS | determina l'accettazione e l'hop successivo |
| Elaborazione | Ruleset generato, Encryption Policies, Custom Commands, trigger di header e oggetto | determina firma, cifratura, GINA, bounce o percorso in chiaro |
| Crittografia | chiavi private S/MIME/OpenPGP, chiavi pubbliche dei destinatari, fiducia CA, OCSP/CRL, chiavi di dominio | determina se e per chi un'operazione è possibile e verificabile |
| GINA | domini, layout, account, password, cronologia di consegna, messaggi e chiavi memorizzati | costituisce un ulteriore percorso HTTPS, di identità e conservazione |
| Piattaforma | configurazione appliance, nome di sistema, rete, certificati, licenze, ruolo cluster | determina raggiungibilità, amministrazione e riavvio |
| Osservabilità | log di posta, sistema e audit, messaggi Postmaster, SNMP/NRPE/Syslog | fornisce evidenze e allarmi, ma non è completamente incluso nel backup |

L'assegnazione segue la struttura documentata di menu e componenti, la gestione degli account GINA, la descrizione del cluster e il Quick Setup Guide. È un modello di inventario per amministratori, non un'affermazione su una struttura interna di database non documentata ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html), [SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html), [SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-seppmail.svg?v=20260813" title="Interaktive Infografik: SEPPmail-SMTP-Pfad, Ruleset, Verschlüsselungsverfahren, GINA, Schlüsselzustand, Cluster, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-seppmail.svg?v=20260813">Apri direttamente il grafico interattivo</a>.
</iframe>

## Accettazione SMTP, routing e confine relay

Un Managed Domain collega uno spazio di nomi SMTP accettato al Forwarding Server interno e ad altre impostazioni specifiche del dominio. Facoltativamente, il gateway può verificare l'indirizzo del destinatario tramite una ricerca SMTP su questo server. Il percorso in uscita viene determinato dall'MTA integrato tramite DNS oppure mediante un server SMTP configurato; impostazioni TLS e di Sender-Based Routing specifiche del dominio possono sovrascrivere questo percorso predefinito ([SEPPmail: Mail System – Managed Domains and Outgoing Server](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

«Relaying allowed» definisce da quali indirizzi IP o reti l'appliance accetta messaggi per destinatari esterni. La documentazione del produttore avverte esplicitamente di non inserire genericamente il server in uscita, poiché ciò può creare un Open Relay. Apertura della porta, autorizzazione relay, dominio mittente gestito ed eventualmente autenticazione SMTP sono pertanto controlli distinti. Un test TCP positivo non dice nulla sulla decisione relay dopo `MAIL FROM` e `RCPT TO` ([SEPPmail: Mail System – Relaying](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [RFC 5321: SMTP commands](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1)).

Con più gateway, il load balancer a monte o il servizio mittente deve poter raggiungere ogni host produttivo. Al contempo, i sistemi a valle devono accettare tutte le identità gateway previste. In Exchange Online ciò riguarda certificati di connettore o intervalli IP; in un ambiente cluster, anche IP virtuali e la caratteristica documentata secondo cui si riceve con IP alias ma si invia tramite l'indirizzo fisico del sistema mittente ([SEPPmail: Firewall and IP aliases](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: Exchange Online clustering and multi-host](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)).

Dopo l'accettazione SMTP, il Ruleset decide quale percorso di protezione e consegna si applica. Questa decisione deve restare successivamente tracciabile da messaggio, identità, dati di directory e versione della regola.

## Ruleset, policy ed evidenza della decisione

SEPPmail genera regole di elaborazione eseguibili dalle impostazioni del Ruleset Generator. Encryption Policies aggiuntive possono imporre per Managed Domains selezionati e flag opzionali procedure quali solo S/MIME, solo OpenPGP, solo GINA o una selezione gerarchica. Dopo una modifica della policy, secondo il produttore il Ruleset deve essere rigenerato prima che la modifica diventi attiva. «Salvato nella GUI» e «efficace nel percorso attivo dei messaggi» sono quindi due stati diversi ([SEPPmail: Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)).

I Custom Commands estendono questa pipeline con condizioni e azioni. Il produttore documenta, tra l'altro, query LDAP, flag, creazione di utenti e chiavi, operazioni sugli header, firma, cifratura, consegna e rifiuto. Le macro vengono espanse durante la generazione del Ruleset; un errore di sintassi può impedire la generazione ([SEPPmail: Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

I trigger di header o oggetto sono input per una decisione di sicurezza. Possono essere valutati solo da una direzione affidabile o ripuliti prima della valutazione. Gli esempi SEPPmail per la firma tra tenant avvertono esplicitamente di non presumere senza verifica l'identità del mittente. Per ciascun trigger, la documentazione deve pertanto includere fonte, connettore ammesso, autenticazione, normalizzazione, priorità e comportamento fail-closed ([SEPPmail: Signing emails between clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html), [SEPPmail: Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)).

Un'evidenza di decisione riproducibile contiene almeno: mittente e destinatario Envelope, header `From:` visibile e, se applicabile, header `Sender:`, direzione e Managed Domain, policy o regola attivante, chiavi trovate, procedura scelta, azione finale e ID coda. Ciò è particolarmente importante perché l'identità della firma S/MIME viene normalmente verificata rispetto al mittente visibile, mentre il routing SMTP inizialmente opera con l'Envelope ([SEPPmail: Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html), [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.2)).

## S/MIME, OpenPGP e TLS come livelli di protezione distinti

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) incapsula contenuti MIME firmati o cifrati con Cryptographic Message Syntax. Identità e chiavi pubbliche sono tipicamente associate mediante certificati X.509; verifica del percorso, periodo di validità, uso della chiave e informazioni di revoca fanno pertanto parte della decisione ([RFC 5652: CMS](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280: PKIX certificate validation](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960: OCSP](https://datatracker.ietf.org/doc/html/rfc6960)). SEPPmail può gestire certificati di utenti e domini e ottenerli tramite interfacce MPKI; la documentazione firewall indica HTTPS a questo scopo, nonché HTTP o HTTPS per OCSP/CRL ([SEPPmail: Firewall – MPKI and revocation](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) utilizza un diverso modello di pacchetti, chiavi e fiducia. La presenza di una chiave pubblica non dimostra automaticamente che appartenga al destinatario previsto o debba ancora essere utilizzata. Fingerprint, provenienza, User ID, scadenza, revoca, algoritmo e possibilità di decifrazione privata devono essere gestiti come ciclo di vita. L'Encryption Policy distingue PGP/MIME e Inline PGP e può configurare separatamente firma e cifratura ([SEPPmail: Encryption Policy – OpenPGP options](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html), [RFC 9580: Key structures](https://datatracker.ietf.org/doc/html/rfc9580#section-5.5)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) protegge una connessione tra due hop. Senza una policy vincolante, STARTTLS è opportunistico: se non è disponibile, l'MTA mittente può consegnare in chiaro secondo la configurazione. SEPPmail può configurare TLS per destinazione con livelli, server fisso, fingerprint del certificato del server o altri requisiti. Questo non è né S/MIME né OpenPGP e non protegge il messaggio dagli MTA coinvolti ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672: SMTP DANE](https://datatracker.ietf.org/doc/html/rfc7672), [SEPPmail: Mail System – TLS Settings](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

Per una cifratura gateway, testo in chiaro e chiavi private del mittente si trovano entro il confine di fiducia SEPPmail. Ciò consente policy centralizzate, ma significa anche che ruoli amministrativi, backup, connessioni cluster, esportazioni di debug, log ed elaborazione dei contenuti a valle devono essere protetti come se potessero toccare dati di comunicazione riservati o chiavi. Se una specifica esportazione contenga effettivamente contenuti deve essere verificato sulla base della documentazione del produttore e con un test controllato.

Se non è disponibile una chiave del destinatario adatta o un percorso di protezione diretto, GINA sposta l'accesso a un portale web. Al percorso SMTP si aggiungono pertanto identità web, sessione, archiviazione ed e-mail di notifica.

## GINA come servizio web con identità e archiviazione

GINA è il percorso alternativo documentato quando un messaggio non viene consegnato con una chiave del destinatario adatta o una policy impone esplicitamente GINA. La comunicazione con il portale avviene via HTTPS su TCP 443. I GINA Domains controllano, tra l'altro, nome del portale, layout, comportamento di account e password, trasmissione iniziale della password e parametri di conservazione ([SEPPmail: GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html), [SEPPmail: Firewall – GINA](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Per gli amministratori, GINA non è un semplice formato di presentazione, bensì uno stato di servizio autonomo. Un account GINA possiede stato, impostazioni di password e, se applicabile, autenticazione esterna, log utente, cronologia della posta e una chiave. La documentazione del produttore avverte che l'eliminazione dell'account elimina irrevocabilmente anche la relativa chiave e può rendere illeggibili i messaggi GINA rimanenti ([SEPPmail: GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)).

L'esercizio di GINA include pertanto almeno:

- un hostname pubblico stabile e conforme al certificato e il relativo ciclo di vita [DNS](/kb/dns);
- certificato HTTPS, catena completa, rinnovo e raggiungibilità esterna;
- trasmissione o derivazione separata del segreto iniziale;
- blocco account, reset, identità esterna e processo di supporto;
- regole di conservazione ed eliminazione per messaggio, metadati, log e chiavi;
- un test di ripristino che apra effettivamente, dopo il ripristino, un messaggio GINA creato prima del backup.

Il Quick Setup Guide evidenzia che una modifica dell'hostname può rendere irraggiungibili messaggi GINA precedenti. Documenta inoltre che un ripristino su un nuovo sistema richiede la password impostata al momento del backup ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Hostname, certificato, dominio GINA, account e chiave devono pertanto essere considerati insieme.

## Directory e ciclo di vita di chiavi e certificati

LDAP/LDAPS può fornire chiavi pubbliche, valutare condizioni utente o gruppo e limitare la creazione automatica di account o chiavi. L'appliance può inoltre offrire propri endpoint LDAP per distribuire chiavi pubbliche raccolte o interne. Le direzioni e le porte richieste differiscono secondo il ruolo; la matrice firewall ufficiale indica 389/636 per query di directory esterne o interne, nonché porte separate per gli endpoint Keyserver integrati ([SEPPmail: Firewall – LDAP and Key server](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html), [SEPPmail: LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)).

Una query LDAP fa parte del percorso del messaggio non appena una regola non può decidere senza il suo risultato. Search Base, filtro, classe oggetto, identità Bind, fiducia TLS, timeout, comportamento cache e comportamento in caso di irraggiungibilità fanno quindi parte del modello operativo. Un risultato di ricerca vuoto, un timeout e un certificato non attendibile non devono essere trattati come lo stesso risultato. I requisiti generali di protocollo e TLS sono descritti nell'[articolo LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen).

Per chiavi e certificati sono necessari almeno i seguenti cicli di vita separatamente misurabili:

| Oggetto | Creazione o acquisizione | Controllo continuo | Evidenza di ripristino |
|---|---|---|---|
| Chiave utente S/MIME | locale, importazione o MPKI | scadenza, percorso CA, stato di revoca, associazione dell'indirizzo, algoritmo | decifrare un vecchio messaggio con la chiave privata storica |
| Chiave OpenPGP | locale, importazione o Keyserver | fingerprint, User ID, scadenza/revoca, capacità | decifrare un vecchio messaggio con la Secret Key corrispondente |
| Chiave di dominio | Managed Domain Service o scambio manuale | stato pubblicato dell'interlocutore, rotazione, percorso di posta completo | testare messaggio di dominio vecchio e nuovo oltre il confine della rotazione |
| Certificato SMTP/HTTPS | importazione o CA locale/pubblica | SAN, catena, scadenza, corrispondenza della chiave privata, policy peer | avviare il servizio con chiave ripristinata e catena prevista |
| Fiducia CA | importazione Root/Intermediate | scopo, ambito, rimozione di vecchie CA, OCSP/CRL | confrontare una catena valida nota e una revocata |

Le caratteristiche di verifica seguono i rispettivi standard e i controlli documentati di importazione, policy e firma. In caso di firma S/MIME non valida, SEPPmail cita in particolare fiducia CA, catena completa, associazione al mittente e modifica successiva del contenuto come cause ([SEPPmail: Reasons for an invalid signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Poiché gateway, chiavi, directory, GINA e coda agiscono insieme, l'alta disponibilità deve coprire più di un listener virtuale. La separazione frontend/backend determina quali stati devono essere disponibili congiuntamente.

## Cluster, separazione frontend/backend e domini di guasto

Secondo la matrice firewall, la comunicazione generale del cluster utilizza SSH tra le appliance; la sincronizzazione dell'ora tramite NTP è obbligatoria nel cluster. La matrice indica inoltre IP virtuali e sottolinea che, con IP alias, l'invio avviene tramite l'indirizzo fisico del sistema attivo. Le allowlist firewall, Smarthost e connettori devono quindi rappresentare il percorso mittente effettivo, non solo l'indirizzo virtuale di ricezione ([SEPPmail: Firewall – Cluster and NTP](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Uno speciale cluster frontend/backend separa l'elaborazione della posta dai dati di configurazione persistenti. Secondo la descrizione del produttore, sul frontend non esiste un database di configurazione; configurazione e materiale di chiave necessari vengono forniti dal backend per un'azione e quindi eliminati dal frontend. Con Large File Transfer attivato, sul frontend resta tuttavia necessario uno stato file aggiuntivo ([SEPPmail: Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)).

Questa architettura riduce la conservazione permanente di chiavi nella DMZ, ma non elimina tutte le dipendenze. Backend, percorso SSH, risoluzione dei nomi, orario, certificati ed eventualmente archiviazione LFT restano domini di guasto. «Il frontend accetta SMTP» non dimostra né che il backend sia raggiungibile né che un'azione crittografica sia eseguibile. Viceversa, un backend funzionante non dimostra che IP virtuale, percorso esterno e hop SMTP successivo funzionino.

Non tutte le impostazioni sono valide a livello di cluster. La documentazione Mail System definisce diversi parametri specifici della macchina e in particolare impostazioni Extended Postfix e determinate destinazioni TLS come da gestire localmente. Prima dei test di failover è quindi utile una matrice stato desiderato/stato effettivo per nodo: nome di sistema, IP, certificato, Outgoing Identity, rotte TLS, Extended Settings, servizi locali, destinazione di monitoraggio e replica cluster prevista ([SEPPmail: Mail System – cluster notes](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Monitoraggio, logging, backup e ripristino

Secondo il produttore, l'indirizzo Postmaster riceve importanti messaggi Watchdog e Daily Report; senza un indirizzo configurato e realmente raggiungibile manca questo canale di allarme. Inoltre, SEPPmail documenta SNMP tramite UDP 161, NRPE tramite TCP 5666 e Syslog tramite UDP 514 o TCP 6514 protetto da TLS. Quali valori ed eventi siano monitorati deve essere definito per ogni installazione; una semplice interfaccia di monitoraggio aperta non è un health check ([SEPPmail: Mail System – Postmaster](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html), [SEPPmail: Firewall – Monitoring and Syslog](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)).

Una dashboard significativa separa almeno accettazioni e rifiuti SMTP, età della coda per Next Hop, errori Ruleset, successo ed errore crittografico, scadenza dei certificati, raggiungibilità OCSP/CRL e MPKI, latenza LDAP, errori di login e consegna GINA, stato dei dischi, comunicazione cluster ed età del backup. Le soglie richiedono riferimento al throughput normale e a RPO/RTO concordati. Una coda elevata può indicare un interlocutore lento; una coda vuota può anche significare che non viene più accettato alcun messaggio.

Il Quick Setup Guide descrive un backup protetto da password che salva dati dell'appliance inclusi utenti e certificati registrati, ma esclude i log. La password è necessaria per il ripristino su un nuovo sistema. La guida descrive inoltre una consegna notturna al «Backup Operator» ([SEPPmail: Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)). Ne derivano tre controlli distinti:

1. Il file di backup deve arrivare al di fuori dell'appliance e la sua età deve essere monitorata.
2. La password di backup deve essere conservata separatamente, protetta da accessi e disponibile in caso di emergenza.
3. I log necessitano di un proprio percorso di esportazione e conservazione se devono essere mantenuti per fini forensi o di evidenza.

Un ripristino è testato solo quando il sistema di destinazione si avvia, rete e licenze corrispondono, le regole sono attive, SMTP funziona in entrambe le direzioni, i contenuti storici S/MIME/OpenPGP e GINA possono essere decifrati e un caso di test documentato esegue la policy prevista. La replica del cluster non sostituisce questo test: riguarda la disponibilità, non una copia indipendente contro configurazione errata, eliminazione o chiavi compromesse.

La diagnostica segue un messaggio dall'accettazione SMTP attraverso il Rulematch e la decisione della chiave fino all'uscita selezionata o al portale GINA.

## Strumenti di diagnostica

La ricerca guasti segue il messaggio dal nome pubblico attraverso SMTP e TLS fino al Ruleset e al percorso di cifratura scelto. In questo modo viene prima dimostrato il trasporto, prima di modificare funzioni di chiave o portale.

### DNS, MX e nomi pubblici

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName securemail.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.securemail.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A securemail.example.ch
dig +short AAAA securemail.example.ch
dig +short TLSA _25._tcp.securemail.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) e [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) verificano MX, indirizzo gateway ed eventualmente DANE-TLSA. Occorre testare separatamente la vista dei resolver esterni e interni. L'hostname GINA deve inoltre corrispondere al certificato; il solo MX non dice nulla sul Forwarding Server a valle o sul connettore Exchange Online ([RFC 5321: Address resolution and mail routing](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)).

### Raggiungibilità TCP, SMTP e STARTTLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-SMTP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection securemail.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection securemail.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://securemail.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz securemail.example.ch 25
nc -vz securemail.example.ch 443
openssl s_client -starttls smtp -connect securemail.example.ch:25 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) e [`nc`](https://man.openbsd.org/nc) dimostrano solo la raggiungibilità TCP. [`curl.exe`](https://curl.se/docs/manpage.html) per Windows può richiedere STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) mostra sui sistemi Unix capacità SMTP, handshake e catena di certificati. Per una transazione SMTP controllata con Envelope e DATA è adatto [`swaks`](https://jetmore.org/john/code/swaks/); va utilizzato solo con destinatari di test autorizzati. Un test di consegna è completo solo con ID coda, evento di log e arrivo all'hop successivo.

### HTTPS GINA e nome del certificato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-GINA-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) e [`curl`](https://curl.se/docs/manpage.html) verificano raggiungibilità HTTP e comportamento della risposta; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) rende visibili la sessione TLS negoziata e la catena presentata. Un `200`, `301` o `302` non dimostra ancora né l'accesso GINA riuscito né l'accesso a un messaggio storico. A tale scopo occorre un flusso utente sintetico con registrazione, login, lettura, risposta e reset secondo il dominio GINA configurato.

### Percorso dei pacchetti presso un punto di misurazione autorizzato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SEPPmail-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEPPmail-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seppmail.etl
pktmon stop
pktmon pcapng seppmail.etl -o seppmail.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seppmail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) e [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) acquisiscono solo il traffico che passa dal punto di misurazione scelto. Su un client di amministrazione, questo non corrisponde automaticamente al traffico di posta tra sistema di posta e gateway; occorrono un host autorizzato, Port Mirror o un altro punto di osservazione adeguato. SMTP prima di STARTTLS può contenere contenuti personali, HTTPS e STARTTLS forniscono almeno metadati. Acquisizione, accesso, conservazione ed eliminazione devono pertanto far parte del runbook per gli incidenti.

## Storia tecnica

SEPPmail descrive l'origine dell'azienda nel 2001: il fondatore Stefan Klein sviluppò inizialmente una soluzione per la comunicazione cifrata tra uno studio legale di Zurigo e i suoi clienti. Da questa esigenza nacque la successiva azienda produttrice ([SEPPmail: Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)).

La storia del prodotto documentata pubblicamente mostra l'evoluzione da gateway di cifratura centralizzato ad appliance modulare: le Release Notes storiche citano già nel 2007 una CA/RA interna, cifratura di dominio S/MIME, funzioni webmail, antispam/antivirus e un utente di backup limitato. Generazioni successive hanno aggiunto, tra l'altro, policy multi-tenant, interfacce REST, connettori MPKI, Large File Transfer, DKIM/ARC, procedure S/MIME moderne e ruoli cluster specializzati ([SEPPmail: Revision History](https://docs.seppmail.com/ch/20_revision-history.html)).

Anche la base tecnologica si è evoluta. La cronologia delle revisioni cita nel corso degli anni OpenBSD, Postfix, Apache, Perl, GnuPG, OpenLDAP, OpenSSL/LibreSSL, Dovecot, Fetchmail e ClamAV. Per gli amministratori, questa origine è rilevante poiché termini di log, parametri MTA, formati dei certificati e comportamento di rete derivano da questi componenti. Tuttavia, ciò non rende automaticamente supportati interventi diretti al di sotto dell'interfaccia del produttore; in particolare la documentazione avverte contro modifiche Extended Postfix e la loro mancata sincronizzazione nel cluster ([SEPPmail: Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)).

## Fonti

- [SEPPmail – Products](https://www.seppmail.com/products/)
- [SEPPmail – Mail System](https://docs.seppmail.com/en/07_mi_06_ms__mail-system.html)
- [SEPPmail – Add/Edit Encryption Policy](https://docs.seppmail.com/ch/07_mi_07_mp_01_ep_01_addedit-encryption-policy.html)
- [RFC 5321: SMTP responsibility](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [SEPPmail – Frontend/Backend Cluster](https://docs.seppmail.com/en/04_com_09_cl_05_frontend-backend-cluster.html)
- [SEPPmail – Setting Up The Firewall/Router](https://docs.seppmail.com/en/04_com_04_setup-firewall-router.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [SEPPmail – Setup with Exchange Online](https://docs.seppmail.com/en/04_com_powershell_setup-exo.html)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [SEPPmail – Revision History](https://docs.seppmail.com/ch/20_revision-history.html)
- [SEPPmail – GINA User Details](https://docs.seppmail.com/en/07_mi_18_ga_01_user-x.html)
- [SEPPmail – Quick Setup Guide](https://downloads.seppmail.com/wp-content/uploads/docu/Quick-Setup-Guide-en.pdf)
- [SEPPmail – Macros](https://docs.seppmail.com/ch/08_cr_01_gi_04_macros.html)
- [SEPPmail – LDAP-based user creation](https://docs.seppmail.com/en/09_ht_user_creation-with-ldap.html)
- [SEPPmail – Signing Emails Between Clients](https://docs.seppmail.com/en/09_ht_tenant_signing-e-mails-between-tenants.html)
- [SEPPmail – Internal X-header processing](https://docs.seppmail.com/en/08_cr_01_gi_06_x-header.html)
- [SEPPmail – Sender identity selection](https://docs.seppmail.com/en/09_ht_user_creation-based-on-other-criteria-than-from-header.html)
- [RFC 5322: Originator fields](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [SEPPmail – GINA Domains](https://docs.seppmail.com/en/07_mi_17_gd__gina-domains.html)
- [SEPPmail – Reasons For An Invalid Signature](https://docs.seppmail.com/en/09_ht_signature_reasons-for-an-invalid-signature.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [SEPPmail – Die SEPPmail-Story](https://www.seppmail.com/de/die-seppmail-story/)
