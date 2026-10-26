---
title: "Microsoft Entra Connect Sync 2.6.91.0 e 2.6.92.0: correzioni di sicurezza, migrazione a Cloud Sync e aggiornamento obbligatorio entro aprile 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "La versione 2.6.91.0 introduce correzioni di sicurezza, una transizione guidata a Cloud Sync e risolve i problemi noti della 2.6.84.0; l'hotfix 2.6.92.0 corregge la configurazione dell'autenticazione pass-through. Entro il 7 aprile 2027, tutti i server devono eseguire almeno la versione 2.6.84.0 con Application-Based Authentication."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min di lettura"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-e-2-6-92-0-correzioni-di-sicurezza-migrazione-a-cloud"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/it/blog/microsoft-entra-connect-sync-2-6-91-0-e-2-6-92-0-correzioni-di-sicurezza-migrazione-a-cloud
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:24:59.416Z
translationReview: automatic
---

# Microsoft Entra Connect Sync 2.6.91.0 e 2.6.92.0: correzioni di sicurezza, migrazione a Cloud Sync e aggiornamento obbligatorio entro aprile 2027

Microsoft ha rilasciato Entra Connect Sync 2.6.91.0 il 16 settembre 2026 e l'hotfix 2.6.92.0 il 23 settembre 2026, entrambi disponibili solo per il download tramite il Microsoft Entra Admin Center. La versione 2.6.91.0 risolve vulnerabilità di sicurezza in componenti di terze parti inclusi, rende generalmente disponibile l'accesso resistente al phishing nella procedura guidata di installazione, introduce un processo di migrazione guidata a Microsoft Entra Cloud Sync e corregge i tre problemi noti della 2.6.84.0. La versione 2.6.92.0 risolve un errore della 2.6.91.0 durante la configurazione di Pass-through Authentication. Microsoft indica inoltre una scadenza fissa: entro il 7 aprile 2027, tutti i server devono eseguire almeno la versione 2.6.84.0 e utilizzare Application-Based Authentication, altrimenti la sincronizzazione si interromperà. L'articolo è rivolto agli amministratori che gestiscono Entra Connect Sync. Lo stato precedente è descritto nell'articolo su [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Panoramica delle versioni

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Release di funzionalità e sicurezza (download) | Correzioni di sicurezza nei componenti di terze parti, assistente per la migrazione a Cloud Sync, disponibilità generale dell'accesso con passkey nella procedura guidata, nuove autorizzazioni Microsoft Graph, correzioni dei problemi noti della 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Hotfix con correzioni di sicurezza (download) | Risolve la registrazione non riuscita dell'Authentication Agent durante l'attivazione di Pass-through Authentication |

Nelle note sulla versione Microsoft classifica la 2.6.92.0 come release con correzioni di sicurezza e consiglia di aggiornare «il prima possibile». Per nessuna delle due versioni è stato annunciato un rollout di aggiornamento automatico (situazione al 26 ottobre 2026). Anche la 2.6.84.0 resta disponibile solo per il download.

## Aggiornamento obbligatorio entro il 7 aprile 2027

La cronologia delle versioni e le istruzioni di aggiornamento riportano ora un avviso identico: Entra Connect Sync deve essere aggiornato alla versione 2.6.84.0 o successiva e convertito ad Application-Based Authentication (ABA) entro il 7 aprile 2027. Microsoft ritira l'autenticazione legacy tramite il Directory Synchronization Account; i server che non soddisfano questi requisiti non sincronizzeranno più dopo tale data. Se la sincronizzazione si interrompe, secondo Microsoft un aggiornamento alla versione corrente seguito dalla configurazione di ABA ripristina il funzionamento.

Per la pianificazione è fondamentale una modifica del comportamento introdotta con la 2.6.84.0: da questa versione Entra Connect non converte più autonomamente i server esistenti ad ABA. Solo le nuove installazioni configurano ABA automaticamente. Il solo aggiornamento non soddisfa quindi la scadenza. I server esistenti devono essere convertiti nella procedura guidata tramite **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

La procedura guidata mostra il metodo attualmente utilizzato da un server in **View or export current configuration**. Microsoft indica inoltre questo cmdlet:

```powershell
Get-ADSyncEntraConnectorCredential
```

Il valore restituito `ConnectorIdentityType` è `ServiceAccount` (account legacy) oppure `Application` (ABA). Se viene visualizzato `ServiceAccount`, la conversione è necessaria prima del 7 aprile 2027. Dopo una conversione riuscita, Microsoft consiglia di rimuovere il vecchio Directory Synchronization Account da Entra ID.

## Versione 2.6.91.0

### Correzioni di sicurezza

Le note sulla versione indicano vulnerabilità di sicurezza risolte in componenti di terze parti inclusi, autorizzazioni rafforzate sui file temporanei locali e download protetti delle immagini del profilo nel connettore SharePoint. Microsoft non indica numeri CVE, valori CVSS o componenti interessati. Non è quindi possibile valutare la gravità sulla base delle fonti pubbliche.

### Nuove autorizzazioni Microsoft Graph e Conditional Access

Entra Connect riceve autorizzazioni Microsoft Graph aggiuntive. Microsoft avverte esplicitamente: chi utilizza criteri di Conditional Access basati sulle app dovrebbe verificare i criteri indirizzati a `Microsoft.Azure.SyncFabric` o al Microsoft 365 Reporting Service. Verificate questi criteri prima dell'aggiornamento, affinché i nuovi accessi non vengano bloccati.

### Migrazione guidata a Cloud Sync

Una novità è il processo guidato per il passaggio da Entra Connect Sync a Microsoft Entra Cloud Sync. Comprende una valutazione della configurazione esistente, l'installazione del Provisioning Agent, un'attivazione graduale e una convalida. La funzionalità è disponibile solo in Azure Public Cloud. Le istruzioni di aggiornamento di Microsoft consigliano comunque di verificare prima di un aggiornamento se Cloud Sync sia adatto al proprio ambiente.

### Accesso resistente al phishing nella procedura guidata disponibile a livello generale

L'accesso tramite Windows Web Account Manager, introdotto come anteprima nella 2.6.84.0, è ora generally available e attivo per impostazione predefinita. La finestra supporta passkey, chiavi di sicurezza FIDO2 e password, riutilizza una sessione di accesso esistente per altri servizi Entra e mantiene la rotazione delle chiavi Kerberos per Seamless Single Sign-On.

### Cmdlet PowerShell modificati

La versione 2.6.84.0 aveva reso obbligatorio il parametro `-AADUsername` per `Set-ADSyncAADCompanyFeature` e `Set-ADSyncAADPasswordSyncState`. Con la 2.6.91.0 il parametro è di nuovo facoltativo: se manca, Entra Connect ricava un suggerimento di accesso dalla configurazione del connettore e apre un accesso interattivo. Questo vale per `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` e `Set-ADSyncDirSyncConfiguration`. L'accesso interattivo resta pertanto parte di questi cmdlet; gli script che li richiamano senza supervisione dovrebbero essere testati dopo l'aggiornamento.

Chi configura Seamless Single Sign-On con il modulo PowerShell autonomo deve importare `ADSync.psd1` prima di `AzureADSSO.psd1`.

### Ulteriori modifiche

- La finestra **Select Containers** nel Synchronization Service Manager è ora di sola lettura. Le modifiche alla selezione dei contenitori vengono effettuate tramite **Customize synchronization options** nella procedura guidata.
- Supporto per un ulteriore ambiente Sovereign Cloud, incluse Pass-through Authentication, Seamless Single Sign-On, Password Writeback e monitoraggio dell'Health Agent. Le note sulla versione non indicano di quale ambiente si tratti.
- SQL Server 2022 LocalDB incluso è stato aggiornato dalla versione 16.0.4250.1 alla 16.0.4262.2.

### Correzioni di errori

- L'installazione o l'aggiornamento con un database ADSync esistente poteva non riuscire con l'errore `0xE0474352` (problema noto della 2.6.84.0).
- Un aggiornamento non aggiornava una Visual C++ 2015–2022 Redistributable installata precedente alla 14.41.
- La procedura guidata poteva chiudersi all'avvio se il criterio per Windows PowerShell Transcription conteneva una directory di output non valida.
- La rotazione dei certificati di Application-Based Authentication poteva rimuovere l'ultima Application Key utilizzabile in caso di ritardi nella replica della directory.
- Il Synchronization Service Manager poteva chiudersi durante l'elenco delle partizioni di directory in **Connector Properties** (problema noto della 2.6.84.0).
- **Change Credentials** non si apriva per le righe del connettore al di fuori dell'area visibile quando sono configurati molti connettori Active Directory.
- Espandendo un dominio precedentemente deselezionato nella pagina **Domain and OU filtering**, la procedura guidata poteva selezionare nuovamente il dominio e, al termine, attivare la sincronizzazione per l'intero dominio (problema noto della 2.6.84.0).

L'ultimo punto merita attenzione: chi utilizza la 2.6.84.0 e ha eseguito nuovamente la procedura guidata dopo l'aggiornamento dovrebbe controllare il filtro di domini e OU. Un dominio attivato involontariamente comporta oggetti aggiuntivi in Entra ID.

## Versione 2.6.92.0

L'hotfix risolve un errore della 2.6.91.0: l'attivazione di Pass-through Authentication tramite la procedura guidata poteva non riuscire durante la registrazione del Microsoft Entra Connect Authentication Agent installato localmente. Le note sulla versione non indicano ulteriori modifiche. Microsoft contrassegna la 2.6.92.0 come release con correzioni di sicurezza; non sono elencate singole vulnerabilità.

## Problemi noti

| Problema | Versione interessata | Stato |
|---|---|---|
| L'installazione o l'aggiornamento con un database ADSync esistente non riesce con `0xE0474352` | 2.6.84.0 | Risolto nella 2.6.91.0 |
| Il Synchronization Service Manager si chiude in **Connector Properties** | 2.6.84.0 | Risolto nella 2.6.91.0 |
| La procedura guidata seleziona nuovamente un dominio deselezionato | 2.6.84.0 | Risolto nella 2.6.91.0 |
| L'attivazione PTA nella procedura guidata non riesce durante la registrazione dell'agente | 2.6.91.0 | Risolto nella 2.6.92.0 |
| La sincronizzazione non riesce dopo l'aggiornamento (`System.IO.FileLoadException`), se `miiserver.exe.config` era stato modificato in precedenza | 2.5.190.0, 2.6.1.0 | Ancora elencato nella cronologia delle versioni; soluzione alternativa tramite bindingRedirect manuale, vedere [articolo sulla 2.6.84.0](/blog/entra-connect-2-6-84-0) |

La versione ritirata 2.6.79.0 continua a non essere disponibile. Microsoft indirizza ora gli utenti interessati alla 2.6.92.0: disinstallare e installare la versione corrente.

## Scadenze del supporto

Ogni versione 2.x esce dal supporto dodici mesi dopo la pubblicazione della versione successiva. Con le due nuove versioni risultano le seguenti scadenze:

| Versione | Rilasciata | Fine del supporto |
|---|---|---|
| 2.5.79.0 | 1 settembre 2025 | 23 ottobre 2026 (scaduto) |
| 2.5.190.0 | 19 novembre 2025 | 2 febbraio 2027 |
| 2.6.1.0 | 2 febbraio 2026 | 10 marzo 2027 |
| 2.6.3.0 | 10 marzo 2026 | 7 luglio 2027 |
| 2.6.84.0 | 7 luglio 2026 | 16 settembre 2027 |
| 2.6.91.0 | 16 settembre 2026 | 23 settembre 2027 |
| 2.6.92.0 | 23 settembre 2026 | ancora da definire |

Per le versioni dalla 2.5.190.0 alla 2.6.3.0 si applica inoltre la scadenza del 7 aprile 2027: sono inferiori alla versione minima richiesta 2.6.84.0 e non sincronizzeranno più dopo tale data, anche se il termine regolare del supporto della 2.6.3.0 è solo a luglio 2027. La versione 2.5.79.0 è senza supporto dal 23 ottobre 2026; secondo Microsoft, le versioni non più supportate possono smettere di funzionare in modo imprevisto.

## Procedura consigliata

La versione di destinazione è la 2.6.92.0. Include le correzioni di sicurezza di entrambe le release, risolve tutti i problemi noti della 2.6.84.0 e della 2.6.91.0 e soddisfa la versione minima prevista dalla scadenza del 7 aprile 2027. Il programma di installazione è disponibile solo nel Microsoft Entra Admin Center, in **Microsoft Entra Connect | Get started**, scheda **Manage**; i prerequisiti sono .NET Framework 4.7.2 e TLS 1.2. Prima dell'aggiornamento, verificate i criteri Conditional Access basati sulle app per `Microsoft.Azure.SyncFabric` e per Microsoft 365 Reporting Service, nonché gli script che richiamano i cmdlet di configurazione cloud citati. Per aggiornamenti a basso rischio, Microsoft consiglia la Swing Migration: aggiornare prima il server di staging, verificare la configurazione dopo Full Import e Full Synchronization, quindi scambiare i ruoli e aggiornare il server attivo precedente. Secondo Microsoft, chi non aggiorna da 12 a 18 mesi dovrebbe scegliere questo percorso. Dopo l'aggiornamento, convertite ad Application-Based Authentication i server esistenti che riportano ancora `ServiceAccount`, e controllate il filtro di domini e OU sui server che in precedenza eseguivano la 2.6.84.0.

## Fonti

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): note sulla versione relative alla 2.6.91.0 e alla 2.6.92.0, problemi noti della 2.6.84.0, tabella di ritiro e scadenza del 7 aprile 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): metodi di aggiornamento, Swing Migration tramite un server di staging e avviso sull'aggiornamento obbligatorio.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): conversione manuale dei server esistenti ad ABA, `Get-ADSyncEntraConnectorCredential` e rimozione dell'account legacy.
