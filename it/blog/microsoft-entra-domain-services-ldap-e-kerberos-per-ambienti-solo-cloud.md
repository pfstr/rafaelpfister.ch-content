---
title: "Microsoft Entra Domain Services: LDAP e Kerberos per ambienti solo cloud"
navTitle: "Entra Domain Services"
description: "Entra ID non supporta LDAP né Kerberos. Microsoft Entra Domain Services fornisce un dominio Active Directory gestito, sincronizza gli utenti da Entra ID e offre protocolli classici. Funzionamento, limiti, costi e un caso pratico con un gateway e-mail."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min di lettura"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-e-kerberos-per-ambienti-solo-cloud"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
url: https://rafaelpfister.ch/it/blog/microsoft-entra-domain-services-ldap-e-kerberos-per-ambienti-solo-cloud
translationSourceHash: 00f01b9fa1426d692146e27b2e15e6926e04ea3cccd4855bd0b18c8c10e36e0d
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:22:12.679Z
translationReview: automatic
---

# Microsoft Entra Domain Services: LDAP e Kerberos per ambienti solo cloud

Chi ha trasferito completamente i propri utenti a Microsoft Entra ID (in precedenza Azure Active Directory) se ne accorge al più tardi con la prima appliance o applicazione legacy: Entra ID risponde alle richieste tramite Microsoft Graph e protocolli di autenticazione moderni come OAuth e SAML, ma non tramite LDAP, Kerberos o NTLM. Un bind LDAP verso Entra ID semplicemente non è possibile. Per tutto ciò che si aspetta un classico Active Directory, Microsoft offre un servizio dedicato: Microsoft Entra Domain Services, precedentemente Azure AD Domain Services.

## Cosa fornisce il servizio

Entra Domain Services è un dominio Active Directory gestito. Microsoft gestisce due domain controller Windows in un VNet di Azure, si occupa di patching, replica e backup e sincronizza automaticamente utenti e gruppi da Entra ID al dominio. La sincronizzazione funziona solo in una direzione, da Entra ID al dominio gestito; le modifiche eseguite direttamente nel dominio non vengono riportate indietro.

All'esterno, il dominio si comporta come un normale Active Directory: risponde alle richieste LDAP e LDAPS, supporta l'autenticazione Kerberos e NTLM, consente l'aggiunta di VM al dominio e offre criteri di gruppo limitati. Applicazioni e dispositivi non devono essere adattati: vedono un domain controller.

## A cosa serve

Il servizio è rivolto ad ambienti che sono sostanzialmente solo cloud, ma utilizzano singoli componenti con requisiti di directory classici:

- **Appliance e applicazioni specialistiche con integrazione LDAP:** dispositivi che cercano utenti tramite LDAP, valutano appartenenze ai gruppi o verificano gli accessi tramite bind LDAP.
- **Migrazioni lift-and-shift:** workload server che devono restare associati al dominio (Kerberos, NTLM, join al dominio), senza dover gestire domain controller propri in Azure.
- **Ambienti senza AD locale:** dove non è mai esistito un Active Directory o è stato dismesso, il dominio gestito sostituisce la creazione di DC propri e il relativo onere operativo.

Importante per delimitare il caso d'uso: chi gestisce ancora un Active Directory locale con sincronizzazione Entra Connect in genere non ha bisogno del servizio; l'appliance interroga quindi l'AD esistente. Entra Domain Services colma il divario quando Entra ID è l'unica fonte di utenti.

## Architettura e configurazione

Il dominio gestito viene distribuito in un VNet o una subnet dedicati e riceve due indirizzi fissi dei domain controller. I workload in altri VNet lo raggiungono tramite peering VNet; i server DNS dei VNet coinvolti devono puntare ai domain controller affinché il nome del dominio e gli oggetti possano essere risolti. L'accesso viene limitato tramite Network Security Groups alle origini e alle porte realmente necessarie.

Alcune particolarità del dominio gestito rilevanti per il collegamento delle applicazioni:

- Gli utenti sincronizzati si trovano nell'OU **AADDC Users**, mentre il dominio ha senza configurazione aggiuntiva il suffisso **onmicrosoft.com**. Search Base e Bind DN devono riflettere questa struttura, ad esempio CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Non esiste un Domain Administrator. L'amministrazione avviene tramite il gruppo delegato AAD DC Administrators; non sono possibili estensioni dello schema.
- Per gli account di bind LDAP è sufficiente un account dedicato e senza privilegi; per le sole query di directory in Entra ID è necessario il ruolo Directory Readers.

## La trappola degli hash delle password

Un aspetto fa regolarmente perdere tempo nei test: gli accessi Kerberos e NTLM nonché i bind LDAP richiedono hash delle password nel dominio gestito. Per gli account solo cloud, Entra ID genera questi hash solo alla successiva modifica della password dopo l'attivazione del servizio. Un utente appena sincronizzato è quindi visibile nella directory, ma può accedere solo dopo aver modificato una volta la propria password. Per gli account ibridi, gli hash devono essere sincronizzati dall'AD locale tramite Entra Connect.

## Secure LDAP passo dopo passo

All'interno del dominio, LDAP funziona per impostazione predefinita in chiaro sulla porta 389. Per gli accessi e per ogni accesso al di fuori di reti strettamente isolate, la connessione deve usare Secure LDAP (LDAPS, porta 636); il servizio offre comunque accesso dall'esterno del VNet solo in forma crittografata. La configurazione consiste in quattro passaggi.

**1. Ottenere il certificato.** Secure LDAP richiede un certificato dedicato, da caricare come PFX includendo la chiave privata. Il Subject o SAN deve coprire il dominio gestito con una wildcard (ad esempio *.example.onmicrosoft.com), poiché le richieste possono arrivare a uno qualsiasi dei due domain controller. Come fonte si possono usare una CA pubblica, la propria PKI o un certificato autofirmato creato appositamente. Con un certificato autofirmato, la catena deve essere configurata come attendibile su ogni sistema che effettua richieste; non tutte le appliance lo consentono. Se si può scegliere, la propria PKI o una CA pubblica offrono maggiore tranquillità.

**2. Attivare Secure LDAP.** Nel portale, in Settings > Secure LDAP, si attiva la funzione e si carica il PFX con la relativa password. Facoltativamente è possibile abilitare l'accesso tramite Internet; il dominio gestito riceve quindi un indirizzo IP pubblico.

**3. Rete e DNS.** L'indirizzo IP esterno è disponibile in Properties. La relativa regola NSG apre TCP/636 e dovrebbe essere limitata agli indirizzi IP sorgente realmente necessari, non a Any. Per la risoluzione dei nomi, una voce DNS (ad esempio ldaps.example.com) punta a questo IP; deve corrispondere al certificato. Gli accessi interni continuano ad avvenire direttamente verso gli indirizzi dei domain controller.

**4. Testare la connessione.** Prima di modificare l'applicazione, vale la pena eseguire un test con un browser LDAP, ldp.exe o ldapsearch sulla porta 636: bind con l'account di servizio, quindi una ricerca nell'OU AADDC Users. Solo quando bind e ricerca funzionano correttamente, è il momento di passare all'applicazione.

Per configurare il servizio stesso, l'account del portale necessita dei ruoli Application Administrator, Domain Services Contributor e Groups Administrator; la distribuzione del dominio gestito richiede poco più di un'ora. Nelle impostazioni di sicurezza è inoltre possibile imporre TLS 1.2 come versione minima.

## Costi

Entra Domain Services è una voce di costo continuativa: il servizio viene fatturato a ore in base alla SKU, a cui si aggiungono VNet, peering ed eventuali VM di test. Per un singolo piccolo caso d'uso LDAP è un prezzo considerevole; l'alternativa di gestire domain controller propri come VM ottiene però il risparmio a fronte della responsabilità di patching, backup e disponibilità.

## Caso pratico: gateway e-mail con integrazione LDAP

Un esempio concreto della categoria appliance è il SEPPmail Secure E-Mail Gateway. Utilizza LDAP per la creazione degli utenti e le verifiche delle autorizzazioni e, dal firmware 15.0.6, anche per l'[accesso alla GUI di amministrazione](/blog/seppmail-admin-gui-ldap-authentifizierung). Un'appliance nel VNet di Azure raggiunge il dominio gestito tramite peering VNet con un account di bind dedicato (Directory Readers), protetto tramite NSG. Al più tardi per l'accesso alla GUI di amministrazione, la cui opzione TLS è attiva per impostazione predefinita, la connessione deve usare Secure LDAP.

## Conclusione

Entra Domain Services non sostituisce Entra ID, ma funge da ponte: il servizio traduce una base utenti cloud in un dominio AD classico per tutto ciò che richiede LDAP, Kerberos o join al dominio. Chi deve collegare una sola applicazione dovrebbe valutare i costi ricorrenti rispetto a una modernizzazione dell'applicazione. Una volta configurato il servizio, appliance e applicazioni legacy si comportano come in un familiare ambiente AD, incluse le particolarità descritte relative alla struttura OU, alle autorizzazioni e agli hash delle password.

## Fonti

1.  [Microsoft Learn – «Che cos'è Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): funzionalità del dominio gestito, protocolli supportati e distinzione rispetto a Entra ID e ai domain controller gestiti autonomamente.

2.  [Microsoft Learn – «Sincronizzazione in Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): sincronizzazione unidirezionale, struttura OU e comportamento degli hash delle password per gli account solo cloud e ibridi.

3.  [Microsoft Learn – «Configurare Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS con certificato proprio per accessi LDAP crittografati.

4.  [Collegare la GUI di amministrazione SEPPmail ad Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): configurazione dell'accesso LDAP alla GUI di amministrazione dal firmware 15.0.6.
