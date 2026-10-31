---
title: "Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6"
navTitle: "Accesso LDAP admin"
description: "Dalla firmware 15.0.6, gli amministratori dell'appliance SEPPmail possono autenticarsi tramite un server LDAP esterno come Active Directory, incluso il mapping dei gruppi al gruppo locale admin. Configurazione passo dopo passo in User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min di lettura"
themen:
  - seppmail
slug: "collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
translationSourceHash: b9a1bdfe75f57458eb57c2e1dcb32fdfd25e1279eb720d4cd4754d49b5aac624
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:23:42.667Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione
---

# Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6

Fino alla firmware 15.0.5, l'interfaccia di amministrazione del SEPPmail Secure E-Mail Gateway conosceva solo account locali. Per lavorare in modo corretto, si creava un utente locale distinto per ogni amministratore e lo si aggiungeva al gruppo admin. Funziona, ma presenta i consueti svantaggi degli account locali: password separate per ogni appliance, nessun offboarding centralizzato e nessuna applicazione delle policy password del servizio di directory. Con la patch release 15.0.6 questo cambia. La GUI di amministrazione autentica, se richiesto, gli amministratori tramite un server LDAP esterno come Active Directory e mappa i gruppi AD ai gruppi locali dell'appliance.

Le ulteriori modifiche della release sono riassunte nell'articolo [SEPPmail 15.0.6 e 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Qui si tratta esclusivamente della nuova autenticazione esterna.

## Cosa offre la funzione

Secondo le Extended Release Notes, la versione 15.0.6 aggiunge una nuova sezione **External Authentication** in **User > Advanced Settings**. In questo modo, la GUI di amministrazione si autentica tramite un server LDAP esterno e i gruppi esterni, ad esempio i gruppi di sicurezza AD, vengono mappati ai gruppi locali dell'appliance.

Gli utenti autenticati esternamente compaiono localmente sull'appliance e si comportano come utenti locali, con una differenza: la loro password non può essere modificata sull'appliance, poiché risiede nel server LDAP esterno. La gestione delle password passa quindi interamente alla directory.

Importante per distinguerla: l'appliance supportava già in precedenza un'autenticazione esterna, ma solo per l'interfaccia web GINA, configurata per ogni Managed Domain, nella sezione External authentication della configurazione di dominio. La novità della 15.0.6 è che anche l'accesso all'interfaccia di amministrazione stessa avviene tramite LDAP.

Sto ancora verificando se anche l'HIN Mailgateway ha ricevuto l'accesso LDAP e integrerò successivamente l'articolo. Poiché le appliance HIN si basano sulla stessa firmware SEPPmail, lo presumo.

## Prerequisiti

Prima della configurazione, devono essere disponibili tre elementi:

- **Firmware 15.0.7 o successiva:** La funzionalità è stata introdotta con la 15.0.6. Dal 2 settembre 2026 è disponibile la patch release 15.0.7, che corregge tra l'altro un aggiramento dell'obbligo MFA (SEC-100) e una Command Injection nell'elaborazione della configurazione (CVE-2026-84830). L'appliance dovrebbe disporre almeno di questa versione; anche il ramo funzionale 15.1, attualmente Hotfix 15.1.0.1 del 29 settembre 2026, include la funzione.
- **Una directory compatibile con LDAP:** Active Directory, OpenLDAP o equivalente. Se gli utenti risiedono soltanto in Entra ID, che non parla LDAP direttamente, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) funge da collegamento.
- **Un account di bind nella directory:** Un account di servizio dedicato e senza privilegi, con accesso in lettura, che l'appliance utilizza per eseguire la ricerca LDAP. Non un amministratore di dominio.
- **Un gruppo AD per gli amministratori del gateway:** Ad esempio un gruppo di sicurezza SEPPmail-Admins, che verrà successivamente mappato al gruppo locale admin. L'appartenenza a questo gruppo determina quindi l'accesso amministrativo completo.

TLS è attivato per impostazione predefinita nelle impostazioni di connessione e dovrebbe rimanere tale; le credenziali degli amministratori non devono transitare in rete senza crittografia. L'appliance deve poter raggiungere il server LDAP sulla porta configurata, solitamente 636 per LDAPS.

## Configurazione in User > Advanced Settings

La configurazione si trova nella GUI di amministrazione, in **User > Advanced Settings**, nella sezione **External Authentication**, e comprende quattro blocchi.

**1. Connection Settings:** La casella *Authenticate users to external LDAP server (e.g. Active Directory)* attiva la funzione. Seguono quindi l'indirizzo del server, la porta, l'opzione *TLS required*, nonché Bind DN e Bind Password dell'account di servizio.

**2. User Attributes:** Qui si definisce come l'appliance individua gli oggetti utente: LDAP Object Class, tipicamente person in Active Directory, Search Base, ovvero l'OU o il container in cui risiedono gli account amministrativi, e l'attributo e-mail, mail per impostazione predefinita.

**3. Group Attributes:** Analogamente, qui vengono indicate le informazioni per gli oggetti gruppo, affinché l'appliance possa risolvere le appartenenze ai gruppi.

**4. Mapping Settings:** La parte decisiva. In *Remote Group* viene selezionato il gruppo dal server LDAP, mentre in *Local Group* uno o più gruppi locali ai quali viene mappato. Per l'accesso amministrativo completo è il gruppo admin; i suoi membri sono equiparati all'utente predefinito admin. Chi desidera differenziare può invece effettuare il mapping verso gruppi limitati quali readonly admin o verso gruppi dell'appliance correlati a specifiche funzioni.

Prima di salvare, vale la pena usare il **Login Test** integrato: con nome utente e password di un account di prova è possibile verificare che connessione, ricerca e autenticazione funzionino prima che la configurazione diventi attiva.

## Configurazioni di esempio

I seguenti valori devono essere adattati al proprio ambiente, con dominio di esempio example.com. I nomi dei campi corrispondono alla sezione External Authentication dell'appliance.

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

Note su Active Directory: come server è adatto qualsiasi Domain Controller raggiungibile; in ambienti con più sedi, si raccomanda un DC nella stessa sede o un alias che punti a più DC. La porta 636 è per LDAPS; a tale scopo, il certificato del DC deve poter essere convalidato dall'appliance. La Search Base dovrebbe essere definita in modo sufficientemente restrittivo da includere gli account amministrativi, ma non l'intera directory. L'attributo mail deve essere compilato negli account AD.

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

Note su OpenLDAP: nelle configurazioni tipiche, gli utenti risiedono come inetOrgPerson sotto ou=people. Per i gruppi, groupOfNames è la scelta più affidabile, poiché l'appartenenza viene rappresentata tramite l'attributo member con il DN completo. I gruppi posixGroup, invece, elencano i membri solo come memberUid, ovvero il nome utente anziché il DN; non è documentato se l'appliance sia in grado di risolverli e ciò dovrebbe essere verificato con il Login Test prima della migrazione. Se il server funziona solo con STARTTLS sulla porta 389, occorre inserire la porta corrispondente nel campo Server; la connessione non deve in alcun caso essere non cifrata.

## Indicazioni operative

Tre punti meritano attenzione prima che l'accesso LDAP diventi l'unico modo per accedere all'appliance:

- **Mantenere un accesso locale di emergenza.** Le password degli utenti esterni risiedono nel server LDAP. Se la directory non è raggiungibile, a causa di un problema di rete, della manutenzione AD o perché il gateway deve proprio risolvere un problema su quella rete, è comunque necessario un account amministrativo locale con password archiviata in modo sicuro. L'utente predefinito admin non dovrebbe quindi essere eliminato, ma mantenuto come accesso di emergenza documentato.
- **L'MFA resta rilevante.** La 15.0.6 ha anche rivisto l'accesso MFA: il secondo fattore non viene più aggiunto alla password, ma richiesto in un campo separato. L'autenticazione esterna non sostituisce il secondo fattore. Una vulnerabilità che, in determinate condizioni, consentiva l'accesso prima del completamento di tutti i passaggi MFA è stata corretta solo con la 15.0.7.
- **Offboarding tramite la directory.** Il vero vantaggio dell'integrazione: se un amministratore lascia l'azienda, è sufficiente disattivare l'account AD o rimuoverlo dal gruppo mappato. Non è più necessario aggiornare gli account locali su ogni appliance. Gli oggetti utente autenticati esternamente, visibili localmente, dovrebbero comunque essere confrontati periodicamente con la directory.

## Conclusione

L'autenticazione LDAP per la GUI di amministrazione colma una lacuna presente da tempo nell'appliance: gli accessi amministrativi possono ora essere gestiti centralmente nella directory anziché dispositivo per dispositivo. Insieme al campo MFA separato, la 15.0.6 migliora quindi sensibilmente l'accesso all'interfaccia di amministrazione in un'unica release. Chi introduce la funzione dovrebbe mantenere il mapping dei gruppi consapevolmente restrittivo e conservare l'accesso locale di emergenza.

## Cronologia delle modifiche

- **25 ottobre 2026:** Versione minima consigliata innalzata dalla 15.0.6.1 alla 15.0.7, che corregge un aggiramento dell'obbligo MFA (SEC-100) e CVE-2026-84830. Aggiunto: anche il ramo funzionale 15.1, attualmente 15.1.0.1, include la funzione.

## Fonti

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): voce relativa all'autenticazione della GUI di amministrazione con descrizione della funzione, posizione della configurazione e comportamento degli utenti autenticati esternamente; correzioni di sicurezza della 15.0.7 (SEC-100, CVE-2026-84830).

2.  [Documentazione SEPPmail – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): riferimento dei campi della sezione External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentazione SEPPmail – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): gruppi predefiniti dell'appliance; i membri del gruppo admin dispongono di accesso amministrativo senza limitazioni.

4.  [Documentazione SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Release Notes ufficiali della 15.0.6 con la voce relativa all'autenticazione della GUI di amministrazione tramite server LDAP esterni, nonché delle release successive 15.0.7, 15.1.0 e 15.1.0.1.

5.  [SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni di amministrazione](/blog/seppmail-releases-15-0-6-und-15-0-6-1): panoramica di tutte le modifiche delle due release.
