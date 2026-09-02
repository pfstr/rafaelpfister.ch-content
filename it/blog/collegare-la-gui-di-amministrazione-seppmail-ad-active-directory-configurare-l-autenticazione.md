---
title: "Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6"
navTitle: "Accesso LDAP per amministratori"
description: "A partire dal firmware 15.0.6, gli amministratori dell'appliance SEPPmail possono autenticarsi tramite un server LDAP esterno come Active Directory, incluso il mapping dei gruppi al gruppo locale admin. Configurazione passo dopo passo in User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min di lettura"
themen:
  - seppmail
slug: "collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: aad5af6607824c7af146d3214d622067cdb1dfe98b82358fbc7566a32256464a
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:23:26.398Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione
---

# Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6

Fino al firmware 15.0.5, l'interfaccia di amministrazione del SEPPmail Secure E-Mail Gateway supportava solo account locali. Per lavorare in modo corretto, si creava un utente locale separato per ogni amministratore e lo si aggiungeva al gruppo admin. Questo funziona, ma presenta i consueti svantaggi degli account locali: password specifiche per ogni appliance, nessun offboarding centralizzato e nessuna applicazione delle policy delle password del servizio di directory. Con la patch release 15.0.6 la situazione cambia. La GUI di amministrazione autentica, su richiesta, gli amministratori tramite un server LDAP esterno come Active Directory e mappa i gruppi AD ai gruppi locali dell'appliance.

Le ulteriori modifiche della release sono riassunte nell'articolo su [SEPPmail 15.0.6 e 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Qui si tratta esclusivamente della nuova autenticazione esterna.

## Cosa offre la funzione

Secondo le Extended Release Notes, la versione 15.0.6 aggiunge una nuova sezione **External Authentication** in **User > Advanced Settings**. In questo modo, la GUI di amministrazione si autentica tramite un server LDAP esterno e i gruppi esterni, ad esempio i gruppi di sicurezza AD, vengono mappati ai gruppi locali dell'appliance.

Gli utenti autenticati esternamente vengono visualizzati localmente sull'appliance e si comportano come utenti locali, con una differenza: la loro password non può essere modificata sull'appliance, poiché risiede nel server LDAP esterno. La gestione delle password passa quindi interamente alla directory.

Importante per distinguerla: l'appliance disponeva già in precedenza di un'autenticazione esterna, ma solo per l'interfaccia web GINA, configurata per ogni Managed Domain (sezione External authentication nella configurazione del dominio). La novità della versione 15.0.6 è che anche l'accesso all'interfaccia di amministrazione stessa avviene tramite LDAP.

Sto ancora verificando se anche il gateway di posta HIN ha ricevuto l'accesso LDAP e completerò l'articolo in seguito. Poiché le appliance HIN si basano sullo stesso firmware SEPPmail, lo presumo.

## Prerequisiti

Prima della configurazione devono essere disponibili quattro elementi:

- **Firmware 15.0.6.1:** La funzione arriva con la versione 15.0.6; a causa dei due errori RuleEngine della release, la scelta corretta è direttamente l'hotfix 15.0.6.1.
- **Una directory compatibile con LDAP:** Active Directory, OpenLDAP o equivalente. Se gli utenti sono presenti solo in Entra ID, che non parla LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) funge da ponte.
- **Un account di bind nella directory:** Un account di servizio dedicato e senza privilegi, con accesso in lettura, che l'appliance utilizza per eseguire le ricerche LDAP. Non un amministratore di dominio.
- **Un gruppo AD per gli amministratori del gateway:** Ad esempio, un gruppo di sicurezza SEPPmail-Admins, che verrà successivamente mappato al gruppo locale admin. L'appartenenza a questo gruppo determina quindi l'accesso amministrativo completo.

TLS è attivato per impostazione predefinita nelle impostazioni di connessione e dovrebbe rimanere tale; le credenziali degli amministratori non devono transitare in rete senza crittografia. L'appliance deve poter raggiungere il server LDAP sulla porta configurata, generalmente 636 per LDAPS.

## Configurazione in User > Advanced Settings

La configurazione si trova nella GUI di amministrazione, in **User > Advanced Settings**, nella sezione **External Authentication** e consiste di quattro blocchi.

**1. Connection Settings:** La casella *Authenticate users to external LDAP server (e.g. Active Directory)* attiva la funzione. Seguono l'indirizzo del server, la porta, l'opzione *TLS required* nonché il Bind DN e il Bind Password dell'account di servizio.

**2. User Attributes:** Qui si definisce come l'appliance individua gli oggetti utente: la LDAP Object Class, in genere person in Active Directory, la Search Base, ovvero l'OU o il contenitore in cui si trovano gli account amministrativi, e l'attributo e-mail, predefinito: mail.

**3. Group Attributes:** Analogamente, qui si inseriscono le informazioni per gli oggetti gruppo, affinché l'appliance possa risolvere le appartenenze ai gruppi.

**4. Mapping Settings:** La parte decisiva. In *Remote Group* si seleziona il gruppo dal server LDAP; in *Local Group* uno o più gruppi locali a cui viene mappato. Per l'accesso amministrativo completo si tratta del gruppo admin; i suoi membri sono equiparati all'utente predefinito admin. Chi desidera differenziare può invece effettuare il mapping verso gruppi limitati come readonly admin o verso gruppi dell'appliance legati a specifiche funzioni.

Prima di salvare, vale la pena utilizzare il **Login Test** integrato: con nome utente e password di un account di test è possibile verificare connessione, ricerca e autenticazione prima che la configurazione diventi attiva.

## Configurazioni di esempio

I valori seguenti devono essere adattati al proprio ambiente, con dominio di esempio example.com. I nomi dei campi corrispondono alla sezione External Authentication dell'appliance.

### Active Directory

| Campo | Valore |
|---|---|
| Server | dc01.example.com |
| Porta | 636 |
| TLS required | attivato |
| Bind DN | CN=svc-seppmail,OU=ServiceAccounts,DC=example,DC=com |
| Bind Password | Password dell'account di servizio |
| User: LDAP Object Class | person |
| User: Search Base | OU=IT,DC=example,DC=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | group |
| Group: Search Base | OU=Groups,DC=example,DC=com |
| Mapping: Remote Group | SEPPmail-Admins |
| Mapping: Local Group | admin |

Note su Active Directory: come server è adatto qualsiasi Domain Controller raggiungibile; negli ambienti con più sedi è consigliabile un DC nella stessa sede o un alias che punti a più DC. La porta 636 è LDAPS; il certificato del DC deve quindi poter essere convalidato dall'appliance. La Search Base dovrebbe essere limitata in modo da contenere gli account amministrativi, ma non l'intera directory. L'attributo mail deve essere valorizzato negli account AD.

### OpenLDAP

| Campo | Valore |
|---|---|
| Server | ldap01.example.com |
| Porta | 636 |
| TLS required | attivato |
| Bind DN | cn=seppmail,ou=services,dc=example,dc=com |
| Bind Password | Password dell'account di servizio |
| User: LDAP Object Class | inetOrgPerson |
| User: Search Base | ou=people,dc=example,dc=com |
| User: E-Mail Attribute | mail |
| Group: LDAP Object Class | groupOfNames |
| Group: Search Base | ou=groups,dc=example,dc=com |
| Mapping: Remote Group | seppmail-admins |
| Mapping: Local Group | admin |

Note su OpenLDAP: nelle configurazioni tipiche, gli utenti sono memorizzati come inetOrgPerson in ou=people. Per i gruppi, groupOfNames è la scelta affidabile, poiché l'appartenenza viene rappresentata tramite l'attributo member con DN completo. I gruppi posixGroup, invece, elencano i membri solo come memberUid, ovvero nome utente anziché DN; non è documentato se l'appliance riesca a risolverli e occorre verificarlo con il Login Test prima della migrazione. Se il server funziona solo con STARTTLS sulla porta 389, va inserita la porta corrispondente nel campo Server; la connessione non deve in nessun caso avvenire senza crittografia.

## Indicazioni operative

Tre aspetti meritano attenzione prima che l'accesso LDAP diventi l'unico modo per accedere all'appliance:

- **Mantenere un accesso locale di emergenza.** Le password degli utenti esterni risiedono nel server LDAP. Se la directory non è raggiungibile, a causa di un problema di rete, della manutenzione AD o perché il gateway deve proprio risolvere un problema su quella rete, rimane necessario un account amministrativo locale con password conservata in modo sicuro. L'utente predefinito admin non dovrebbe quindi essere eliminato, ma mantenuto come accesso di emergenza documentato.
- **L'MFA rimane rilevante.** La versione 15.0.6 ha rivisto anche l'accesso MFA: il secondo fattore non viene più aggiunto alla password, ma richiesto in un campo separato. L'autenticazione esterna non sostituisce il secondo fattore.
- **Offboarding tramite la directory.** Il vero vantaggio dell'integrazione: quando un amministratore lascia l'azienda, è sufficiente disattivare l'account AD o rimuoverlo dal gruppo mappato. Non è più necessario aggiornare gli account locali su ogni appliance. Gli oggetti utente autenticati esternamente, visibili localmente, dovrebbero comunque essere confrontati periodicamente con la directory.

## Conclusione

L'autenticazione LDAP per la GUI di amministrazione colma una lacuna presente a lungo nell'appliance: ora gli accessi amministrativi possono essere gestiti centralmente nella directory anziché per singolo dispositivo. Insieme al campo MFA separato, la versione 15.0.6 migliora sensibilmente l'accesso all'interfaccia di amministrazione in un'unica release. Chi introduce questa funzione dovrebbe mantenere volutamente restrittivo il mapping dei gruppi e conservare l'accesso locale di emergenza.

## Fonti

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Voce sull'autenticazione della GUI di amministrazione con descrizione della funzione, posizione della configurazione e comportamento degli utenti autenticati esternamente.

2.  [Documentazione SEPPmail – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Riferimento dei campi della sezione External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentazione SEPPmail – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Gruppi predefiniti dell'appliance; i membri del gruppo admin dispongono di accesso amministrativo illimitato.

4.  [Documentazione SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Release notes ufficiali della versione 15.0.6 con la voce relativa all'autenticazione della GUI di amministrazione tramite server LDAP esterni.

5.  [SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Panoramica di tutte le modifiche delle due release.
