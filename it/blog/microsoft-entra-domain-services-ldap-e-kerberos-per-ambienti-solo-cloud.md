---
title: "Microsoft Entra Domain Services: LDAP e Kerberos per ambienti solo cloud"
navTitle: "Entra Domain Services"
description: "Entra ID non parla LDAP né Kerberos. Microsoft Entra Domain Services fornisce un dominio Active Directory gestito, sincronizza gli utenti da Entra ID e offre protocolli classici. Funzionamento, limiti, costi e un caso pratico con un gateway e-mail."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min di lettura"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-e-kerberos-per-ambienti-solo-cloud"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:20:49.485Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/microsoft-entra-domain-services-ldap-e-kerberos-per-ambienti-solo-cloud
---

# Microsoft Entra Domain Services: LDAP e Kerberos per ambienti solo cloud

Chi ha migrato completamente i propri utenti a Microsoft Entra ID (in precedenza Azure Active Directory) se ne accorge al più tardi con la prima appliance o applicazione legacy: Entra ID risponde alle query tramite Microsoft Graph e protocolli di autenticazione moderni come OAuth e SAML, ma non tramite LDAP, Kerberos o NTLM. Un LDAP bind verso Entra ID semplicemente non è possibile. Per tutto ciò che si aspetta un classico Active Directory, Microsoft offre un servizio dedicato: Microsoft Entra Domain Services, in precedenza Azure AD Domain Services.

## Cosa offre il servizio

Entra Domain Services è un dominio Active Directory gestito. Microsoft gestisce a questo scopo due controller di dominio Windows in una VNet di Azure, si occupa di patching, replica e backup e sincronizza automaticamente utenti e gruppi da Entra ID nel dominio. La sincronizzazione avviene in una sola direzione, da Entra ID al dominio gestito; le modifiche effettuate direttamente nel dominio non vengono replicate indietro.

Verso l'esterno, il dominio si comporta come un normale Active Directory: risponde alle query LDAP e LDAPS, supporta l'autenticazione Kerberos e NTLM, consente l'aggiunta di VM al dominio e offre criteri di gruppo limitati. Le applicazioni e i dispositivi non devono essere adattati; vedono un controller di dominio.

## A cosa serve

Il servizio è destinato ad ambienti che sono in realtà solo cloud, ma che eseguono singoli componenti con requisiti di directory tradizionali:

- **Appliance e applicazioni specialistiche con connessione LDAP:** dispositivi che cercano utenti tramite LDAP, valutano le appartenenze ai gruppi o verificano gli accessi tramite LDAP bind.
- **Migrazioni lift-and-shift:** carichi di lavoro server che devono rimanere vincolati al dominio (Kerberos, NTLM, aggiunta al dominio), senza dover gestire controller di dominio propri in Azure.
- **Ambienti senza AD locale:** dove non è mai esistito un Active Directory o è stato dismesso, il dominio gestito sostituisce la realizzazione di DC propri e il relativo onere operativo.

Importante per delimitare il contesto: chi gestisce ancora un Active Directory locale con sincronizzazione Entra Connect, in genere non ha bisogno del servizio; l'appliance interroga allora l'AD esistente. Entra Domain Services colma il divario quando Entra ID è l'unica origine utenti.

## Architettura e configurazione

Il dominio gestito viene distribuito in una VNet o subnet dedicata e riceve due indirizzi fissi dei controller di dominio. I carichi di lavoro in altre VNet lo raggiungono tramite peering VNet; i server DNS delle VNet coinvolte devono puntare ai controller di dominio affinché il nome del dominio e gli oggetti siano risolvibili. L'accesso viene limitato tramite Network Security Groups alle fonti e alle porte effettivamente necessarie.

Alcune particolarità del dominio gestito rilevanti per il collegamento delle applicazioni:

- Gli utenti sincronizzati si trovano nell'OU **AADDC Users**; senza una configurazione propria, il dominio utilizza il suffisso **onmicrosoft.com**. Search Base e Bind DN devono riflettere questa struttura, ad esempio CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Non esiste un Domain Administrator. La gestione avviene tramite il gruppo delegato AAD DC Administrators; non sono possibili estensioni dello schema.
- Per gli account LDAP bind è sufficiente un account dedicato e senza privilegi; per le sole query di directory in Entra ID serve il ruolo Directory Readers.

## Il problema degli hash delle password

Un punto richiede regolarmente tempo nei test: gli accessi Kerberos e NTLM, nonché i LDAP bind, necessitano degli hash delle password nel dominio gestito. Per gli account solo cloud, Entra ID genera questi hash solo alla successiva modifica della password dopo l'attivazione del servizio. Un utente appena sincronizzato è quindi visibile nella directory, ma può accedere solo dopo aver modificato una volta la propria password. Per gli account ibridi, gli hash devono essere sincronizzati dal sistema AD locale tramite Entra Connect.

## Secure LDAP passo per passo

All'interno del dominio, LDAP funziona per impostazione predefinita senza crittografia sulla porta 389. Per gli accessi e per ogni accesso al di fuori di reti strettamente isolate, la connessione deve usare Secure LDAP (LDAPS, porta 636); il servizio offre comunque l'accesso dall'esterno della VNet solo in forma crittografata. La configurazione si compone di quattro passaggi.

**1. Procurarsi un certificato.** Secure LDAP richiede un certificato dedicato, caricato come PFX incluso di chiave privata. Il Subject o SAN deve coprire il dominio gestito tramite wildcard (ad esempio *.example.onmicrosoft.com), poiché le richieste possono raggiungere ciascuno dei due controller di dominio. Si può usare una CA pubblica, la propria PKI o un certificato autofirmato creato appositamente. Con un certificato autofirmato, la catena deve essere configurata come attendibile su ogni sistema che effettua richieste; non tutte le appliance lo consentono. Chi può scegliere, è più tranquillo con la propria PKI o una CA pubblica.

**2. Attivare Secure LDAP.** Nel portale, in Settings > Secure LDAP, si attiva la funzione e si carica il PFX con la relativa password. Facoltativamente è possibile abilitare l'accesso tramite Internet; il dominio gestito riceve quindi un indirizzo IP pubblico.

**3. Rete e DNS.** L'indirizzo IP esterno è indicato in Properties. La relativa regola NSG apre TCP/636 e dovrebbe essere limitata agli indirizzi IP sorgente realmente necessari, non a Any. Per la risoluzione dei nomi, un record DNS (ad esempio ldaps.example.com) punta a questo IP; deve corrispondere al certificato. Gli accessi interni continuano a rivolgersi direttamente agli indirizzi dei controller di dominio.

**4. Testare la connessione.** Prima di modificare l'applicazione, conviene effettuare un test con un browser LDAP, ldp.exe o ldapsearch sulla porta 636: bind con l'account di servizio, quindi una ricerca nell'OU AADDC Users. Solo quando bind e ricerca funzionano correttamente, si passa all'applicazione.

Per configurare il servizio stesso, l'account del portale necessita dei ruoli Application Administrator, Domain Services Contributor e Groups Administrator; la distribuzione del dominio gestito richiede poco più di un'ora. Nelle impostazioni di sicurezza è inoltre possibile imporre TLS 1.2 come versione minima.

## Costi

Entra Domain Services è una voce di costo continuativa: il servizio viene fatturato a ore in base alla SKU, a cui si aggiungono VNet, peering ed eventuali VM di test. Per un singolo piccolo caso d'uso LDAP è un prezzo considerevole; l'alternativa di gestire controller di dominio propri come VM, tuttavia, recupera il risparmio con la responsabilità di patching, backup e disponibilità.

## Caso pratico: gateway e-mail con connessione LDAP

Un esempio concreto della categoria appliance è il SEPPmail Secure E-Mail Gateway. Utilizza LDAP per la creazione degli utenti e le verifiche delle autorizzazioni e, dalla versione firmware 15.0.6, anche per la [connessione alla GUI di amministrazione](/blog/seppmail-admin-gui-ldap-authentifizierung). Un'appliance nella VNet di Azure raggiunge il dominio gestito tramite peering VNet con un account bind dedicato (Directory Readers), protetto tramite NSG. Al più tardi per l'accesso alla GUI di amministrazione, la cui opzione TLS è attiva per impostazione predefinita, la connessione deve usare Secure LDAP.

## Conclusione

Entra Domain Services non è un sostituto di Entra ID, bensì un ponte: il servizio traduce una base utenti cloud in un dominio AD classico per tutto ciò che richiede LDAP, Kerberos o l'aggiunta al dominio. Chi deve collegare una sola applicazione dovrebbe valutare i costi correnti rispetto a una modernizzazione dell'applicazione. Una volta disponibile il servizio, appliance e applicazioni legacy si comportano come in un consueto ambiente AD, incluse le particolarità descritte relative alla struttura delle OU, alle autorizzazioni e agli hash delle password.

## Fonti

1.  [Microsoft Learn – «Che cos'è Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): funzionalità del dominio gestito, protocolli supportati e distinzione rispetto a Entra ID e controller di dominio gestiti in proprio.

2.  [Microsoft Learn – «Sincronizzazione in Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): sincronizzazione unidirezionale, struttura OU e comportamento degli hash delle password per account solo cloud e ibridi.

3.  [Microsoft Learn – «Configurare Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS con certificato proprio per accessi LDAP crittografati.

4.  [Collegare la GUI di amministrazione SEPPmail ad Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): configurazione dell'accesso LDAP alla GUI di amministrazione dalla versione firmware 15.0.6.
