---
title: "Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6"
navTitle: "Login LDAP amministratore"
description: "Dalla versione firmware 15.0.6, gli amministratori dell'appliance SEPPmail possono autenticarsi tramite un server LDAP esterno come Active Directory, incluso il mapping dei gruppi al gruppo locale admin. Configurazione passo per passo in User > Advanced Settings."
date: "2026-07-29"
kategorie: "SEPPmail"
timeToRead: "7 min di lettura"
themen:
  - seppmail
slug: "collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione"
translationId: "article-21092a3dad6b84cb"
draft: false
translationOf: seppmail-admin-gui-ldap-authentifizierung
url: https://rafaelpfister.ch/it/blog/collegare-la-gui-di-amministrazione-seppmail-ad-active-directory-configurare-l-autenticazione
translationSourceHash: bb8386d1f880934d4811eb317bcd51d47900fdd493dad90b1d7752bfc25ba55c
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:47:31.344Z
translationReview: automatic
---

# Collegare la GUI di amministrazione SEPPmail ad Active Directory: configurare l'autenticazione LDAP dalla versione 15.0.6

Fino al firmware 15.0.5, l'interfaccia di amministrazione del SEPPmail Secure E-Mail Gateway conosceva solo account locali. Per lavorare in modo ordinato, si creava un utente locale dedicato per ciascun amministratore e lo si aggiungeva al gruppo admin. Funziona, ma presenta i consueti svantaggi degli account locali: password separate per ogni appliance, nessun offboarding centralizzato e nessuna applicazione delle policy password del servizio di directory. Con la patch release 15.0.6 la situazione cambia. La GUI di amministrazione autentica gli amministratori, su richiesta, tramite un server LDAP esterno come Active Directory e mappa i gruppi AD sui gruppi locali dell'appliance.

Le ulteriori modifiche della release sono riassunte nell'articolo [SEPPmail 15.0.6 e 15.0.6.1](/blog/seppmail-releases-15-0-6-und-15-0-6-1). Qui si tratta esclusivamente della nuova autenticazione esterna.

## Cosa offre la funzione

Secondo le Extended Release Notes, la versione 15.0.6 aggiunge una nuova sezione **External Authentication** in **User > Advanced Settings**. In questo modo la GUI di amministrazione si autentica tramite un server LDAP esterno e i gruppi esterni, come i gruppi di sicurezza AD, vengono mappati sui gruppi locali dell'appliance.

Gli utenti autenticati esternamente compaiono localmente sull'appliance e si comportano come utenti locali, con una differenza: la loro password non può essere modificata sull'appliance, perché risiede nel server LDAP esterno. La gestione delle password passa quindi completamente alla directory.

Importante per distinguere le due funzioni: l'appliance disponeva già in precedenza di un'autenticazione esterna, ma solo per l'interfaccia web GINA, configurata per ogni Managed Domain, nella sezione External authentication della configurazione di dominio. La novità della versione 15.0.6 è che anche l'accesso all'interfaccia di amministrazione stessa avviene tramite LDAP.

Sto ancora verificando se anche l'HIN Mailgateway ha ricevuto l'accesso LDAP e integrerò successivamente l'articolo. Poiché le appliance HIN si basano sullo stesso firmware SEPPmail, ritengo che sia così.

## Prerequisiti

Prima della configurazione devono essere disponibili quattro elementi:

- **Firmware 15.0.6.1:** La funzione arriva con la versione 15.0.6; a causa dei due errori RuleEngine della release, l'hotfix 15.0.6.1 è direttamente la scelta giusta.
- **Una directory compatibile con LDAP:** Active Directory, OpenLDAP o equivalente. Se gli utenti risiedono solo in Entra ID, che non parla LDAP, [Microsoft Entra Domain Services](/blog/microsoft-entra-domain-services-ldap-kerberos) fa da ponte.
- **Un account di bind nella directory:** Un account di servizio dedicato e non privilegiato con accesso in lettura, utilizzato dall'appliance per eseguire le ricerche LDAP. Non un amministratore di dominio.
- **Un gruppo AD per gli amministratori del gateway:** Ad esempio un gruppo di sicurezza SEPPmail-Admins, che verrà in seguito mappato al gruppo locale admin. L'appartenenza a questo gruppo determina quindi l'accesso amministrativo completo.

TLS è attivato per impostazione predefinita nelle impostazioni di connessione e dovrebbe rimanere tale; le credenziali degli amministratori non devono transitare in rete in chiaro. L'appliance deve poter raggiungere il server LDAP sulla porta configurata, normalmente 636 per LDAPS.

## Configurazione in User > Advanced Settings

La configurazione si trova nella GUI di amministrazione in **User > Advanced Settings**, nella sezione **External Authentication**, ed è composta da quattro blocchi.

**1. Connection Settings:** La casella *Authenticate users to external LDAP server (e.g. Active Directory)* attiva la funzione. Seguono l'indirizzo del server, la porta, l'opzione *TLS required*, nonché Bind DN e Bind Password dell'account di servizio.

**2. User Attributes:** Qui si definisce come l'appliance trova gli oggetti utente: la LDAP Object Class, tipicamente person in Active Directory, la Search Base, ovvero l'OU o il contenitore in cui si trovano gli account amministrativi, e l'attributo e-mail, predefinito: mail.

**3. Group Attributes:** Analogamente, qui vengono indicate le informazioni per gli oggetti gruppo, affinché l'appliance possa risolvere le appartenenze ai gruppi.

**4. Mapping Settings:** La parte decisiva. In *Remote Group* si seleziona il gruppo dal server LDAP, mentre in *Local Group* uno o più gruppi locali ai quali viene mappato. Per l'accesso amministrativo completo si tratta del gruppo admin; i suoi membri sono equiparati all'utente standard admin. Chi desidera differenziare i privilegi può invece mappare gruppi limitati, come readonly admin, oppure gruppi dell'appliance correlati a specifiche funzioni.

Prima del salvataggio conviene utilizzare il **Login Test** integrato: con nome utente e password di un account di test è possibile verificare connessione, ricerca e autenticazione prima di attivare la configurazione.

## Configurazioni di esempio

I valori seguenti devono essere adattati al proprio ambiente, con example.com come dominio di esempio. I nomi dei campi corrispondono alla sezione External Authentication dell'appliance.

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

Note su Active Directory: come server è adatto qualsiasi Domain Controller raggiungibile; in ambienti con più sedi si consiglia un DC nella stessa sede o un alias che punti a più DC. La porta 636 è LDAPS; il certificato del DC deve quindi essere validabile dall'appliance. La Search Base deve essere delimitata in modo sufficientemente ristretto da contenere gli account amministrativi, ma non l'intera directory. L'attributo mail deve essere valorizzato negli account AD.

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

Note su OpenLDAP: nelle configurazioni tipiche, gli utenti sono memorizzati come inetOrgPerson in ou=people. Per i gruppi, groupOfNames è la scelta affidabile, poiché l'appartenenza è rappresentata tramite l'attributo member con il DN completo. I gruppi posixGroup, invece, indicano i membri solo come memberUid, ovvero il nome utente anziché il DN; non è documentato se l'appliance sia in grado di risolverlo e va verificato con il Login Test prima della migrazione. Se il server funziona esclusivamente con STARTTLS sulla porta 389, occorre indicare la porta corrispondente nel campo Server; la connessione non deve in alcun caso avvenire in chiaro.

## Indicazioni operative

Tre punti meritano attenzione prima che l'accesso LDAP diventi l'unico modo per accedere all'appliance:

- **Mantenere un accesso locale di emergenza.** Le password degli utenti esterni risiedono nel server LDAP. Se la directory non è raggiungibile, per un problema di rete, manutenzione AD o perché il gateway deve proprio risolvere un problema su quella rete, è comunque necessario un account amministrativo locale con password conservata in modo sicuro. L'utente standard admin non deve quindi essere eliminato, ma mantenuto come accesso di emergenza documentato.
- **L'MFA resta rilevante.** La versione 15.0.6 ha rivisto anche l'accesso MFA: il secondo fattore non viene più aggiunto alla password, ma richiesto in un campo separato. L'autenticazione esterna non sostituisce il secondo fattore.
- **Offboarding tramite la directory.** Il vero vantaggio del collegamento: quando un amministratore lascia l'azienda, basta disattivare l'account AD o rimuoverlo dal gruppo mappato. Non è più necessario aggiornare gli account locali su ogni appliance. Gli oggetti utente autenticati esternamente, visibili localmente, dovrebbero comunque essere confrontati periodicamente con la directory.

## Conclusione

L'autenticazione LDAP per la GUI di amministrazione colma una lacuna presente da tempo nell'appliance: gli accessi degli amministratori possono ora essere gestiti centralmente nella directory anziché per singolo dispositivo. Insieme al campo MFA separato, la versione 15.0.6 rende l'accesso all'interfaccia di amministrazione sensibilmente più maturo in un'unica release. Chi introduce la funzione dovrebbe mantenere il mapping dei gruppi deliberatamente restrittivo e non sacrificare l'accesso locale di emergenza.

## Fonti

1.  [SEPPmail Extended Release Notes 15.0](https://downloads.seppmail.com/extrelnotes/150/ERN15.0.html): Voce relativa all'autenticazione della GUI di amministrazione con descrizione della funzione, posizione della configurazione e comportamento degli utenti autenticati esternamente.

2.  [Documentazione SEPPmail – «User > Advanced Settings»](https://docs.seppmail.com/ch/07_mi_15_usr_04_advanced-settings.html): Riferimento dei campi della sezione External Authentication (Connection, User Attributes, Group Attributes, Mapping, Login Test).

3.  [Documentazione SEPPmail – «Groups»](https://docs.seppmail.com/ch/07_mi_16_groups.html): Gruppi predefiniti dell'appliance; i membri del gruppo admin dispongono di accesso amministrativo illimitato.

4.  [Documentazione SEPPmail – «Revision History»](https://docs.seppmail.com/ch/20_revision-history.html): Release Notes ufficiali della versione 15.0.6 con la voce relativa all'autenticazione della GUI di amministrazione tramite server LDAP esterni.

5.  [SEPPmail 15.0.6 e 15.0.6.1: correzioni di sicurezza e nuove funzioni amministrative](/blog/seppmail-releases-15-0-6-und-15-0-6-1): Panoramica di tutte le modifiche delle due release.
