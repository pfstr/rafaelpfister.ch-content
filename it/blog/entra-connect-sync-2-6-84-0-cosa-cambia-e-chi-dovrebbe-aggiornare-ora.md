---
slug: "entra-connect-sync-2-6-84-0-cosa-cambia-e-chi-dovrebbe-aggiornare-ora"
title: "Entra Connect Sync 2.6.84.0: cosa cambia e chi dovrebbe aggiornare ora"
navTitle: "Entra Connect 2.6.84"
description: "La release di sicurezza introduce il supporto per le passkey e modifiche all'autenticazione delle applicazioni, a PowerShell e a Password Hash Sync. La versione precedente è stata ritirata; l'aggiornamento richiede quindi una decisione graduale."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min di lettura"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/it/blog/entra-connect-sync-2-6-84-0-cosa-cambia-e-chi-dovrebbe-aggiornare-ora"
---

# Entra Connect Sync 2.6.84.0: cosa cambia e chi dovrebbe aggiornare ora

Microsoft ha rilasciato Entra Connect Sync 2.6.84.0 il 7 luglio 2026 come release di sicurezza e raccomanda un upgrade rapido. Allo stesso tempo, il predecessore diretto 2.6.79.0 è stato ritirato a causa di un problema dell'installer scoperto successivamente. La conseguenza non è né «installare subito ovunque» né «aspettare e ignorare»: i sistemi interessati e quelli che stanno per uscire dal supporto dovrebbero passare rapidamente alla nuova versione, mentre tutti gli altri possono prima testare l'aggiornamento in modo controllato.

## Perché questa release merita particolare cautela

La linea 2.6 di Entra Connect Sync ha avuto un avvio accidentato. Un breve riepilogo, perché è rilevante per la decisione di aggiornamento:

- **2.6.1.0** (febbraio 2026) ha corretto, tra l'altro, un errore per cui la modifica della configurazione del connettore Entra ID nel Synchronization Service Manager eliminava i parametri di Application-Based Authentication, causando l'errore del wizard e della rotazione dei certificati. Per tutte le versioni 2.5 era quindi valida la notevole raccomandazione di non utilizzare affatto l'interfaccia di amministrazione del prodotto.
- **2.6.3.0** (marzo 2026) era un hotfix per un problema che poteva arrestare inaspettatamente il server Entra Connect durante l'auto-upgrade. La soluzione provvisoria all'epoca: l'auto-upgrade rileva i file di configurazione modificati manualmente e semplicemente salta tali server.
- **2.6.79.0** (giugno 2026) è stata completamente ritirata dopo la pubblicazione. L'installer non è più disponibile; secondo Microsoft, chi ha installato la versione deve disinstallarla e installare la 2.6.84.0. Microsoft non documenta quale fosse esattamente il problema.

Alla data odierna, la versione 2.6.84.0 è disponibile solo come download tramite il Microsoft Entra Admin Center («Released for download»). Non è stato ancora annunciato un rollout dell'auto-upgrade. Anche questo è un segnale: Microsoft stessa non sta ancora distribuendo la versione su larga scala alle installazioni esistenti.

## Nuove funzionalità

### Accesso resistente al phishing nel wizard di installazione (Preview)

Il wizard di installazione supporta ora l'accesso con passkey e chiavi di sicurezza FIDO2 tramite Windows Web Account Manager (WAM). Il contesto: dal 2024/2025 Microsoft impone gradualmente l'MFA per gli accessi alle interfacce di amministrazione Azure ed Entra, e molte organizzazioni hanno limitato tramite Conditional Access i propri account amministrativi ai metodi resistenti al phishing (FIDO2, passkey, autenticazione basata su certificati). Proprio questi account adeguatamente protetti finora non potevano accedere al wizard Entra Connect, perché la finestra di accesso incorporata non supportava tali metodi. Nella pratica ciò portava a workaround poco eleganti: ad esempio account «di installazione» dedicati con requisiti di autenticazione più deboli, soltanto per completare il wizard. Questa lacuna viene ora colmata, anche se per il momento in Preview.

### Supporto per la Sovereign Cloud francese

La versione 2.6.84.0 introduce il supporto per l'ambiente Sovereign Cloud francese, inclusi Pass-through Authentication, Seamless Single Sign-On, Password Writeback e monitoraggio dell'agente Health. Coerentemente, è stato corretto un errore per cui il nome della cloud Application Proxy non veniva risolto correttamente nella France Cloud e la registrazione PTA falliva con «EnvironmentName attribute is invalid».

## Modifiche comportamentali nel dettaglio

La parte più interessante della release non sono le nuove funzionalità, bensì i comportamenti modificati. Molti di essi correggono decisioni di progettazione che nella pratica hanno causato sorprese.

### L'auto-upgrade non distrugge più i file di configurazione personalizzati

Questa è la modifica con la storia più lunga. Finora, l'auto-upgrade sovrascriveva completamente il file `miiserver.exe.config` durante l'aggiornamento. Le personalizzazioni manuali andavano perse. Sembra un caso marginale, ma non lo era: Microsoft stessa aveva istruito gli amministratori in ambienti FIPS a modificare proprio questo file affinché Password Hash Synchronization funzionasse con la modalità FIPS attivata. Chi aveva seguito le istruzioni ufficiali disponeva quindi di un file di configurazione «modificato».

Le conseguenze si sono manifestate con l'upgrade alla 2.5.190.0 e alla 2.6.1.0 come problema noto: se l'installer rileva un file `miiserver.exe.config` modificato, lo lascia inalterato; tuttavia manca quindi il nuovo assembly binding e il servizio di sincronizzazione termina dopo l'upgrade con `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Il workaround documentato: aggiungere manualmente un bindingRedirect nella sezione `assemblyBinding` di `miiserver.exe.config` (sotto `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Successivamente riavviare il servizio ADSync. L'hotfix 2.6.3.0 ha mitigato il problema solo per l'auto-upgrade: i server interessati venivano semplicemente saltati e rimanevano alla versione precedente. Con la 2.6.84.0 arriva la vera soluzione: il processo di upgrade unisce le personalizzazioni del cliente alla nuova configurazione e convalida il risultato prima di applicarlo. Chi esegue un upgrade manuale da una versione interessata dovrebbe comunque controllare prima lo stato del proprio file `miiserver.exe.config` ed eseguirne un backup: il meccanismo di merge è nuovo e pertanto non ancora collaudato nella pratica.

### Application-Based Authentication: fine del fallback silenzioso e della conversione silenziosa

Per ricordare: dalla 2.5.76.0, Application-Based Authentication (ABA) è generally available ed è lo standard. Al posto del vecchio Directory Synchronization Account, un account cloud con password salvata, il server Sync si autentica come applicazione Entra ID con un certificato, idealmente protetto da TPM. Si tratta di un'architettura molto più robusta: nessuna password che possa essere sottratta e una credenziale vincolata alla macchina.

La versione 2.6.84.0 interviene su due comportamenti che hanno compromesso questo guadagno di sicurezza:

**Nessun fallback silenzioso.** Se la configurazione ABA nel wizard falliva, finora l'installazione tornava senza avviso all'account legacy. Il risultato: l'amministratore riteneva di avere un accesso basato su certificati, ma in realtà il server operava con il vecchio account con password. Un classico schema fail-open. Ora il wizard si interrompe con un messaggio di errore chiaro («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), affinché venga risolta la causa effettiva invece di nasconderla.

**Nessuna conversione automatica in background.** Finora Entra Connect convertiva autonomamente i server esistenti dall'account legacy ad ABA durante il normale funzionamento della sincronizzazione. Benintenzionato dal punto di vista della sicurezza, un incubo dal punto di vista operativo: un metodo di autenticazione cambia senza richiesta, senza finestra di change e senza che nessuno ne sia informato. E se qualcosa va storto (problemi TPM, conflitti di Conditional Access, firewall), la sincronizzazione si ferma. Ora vale quanto segue: solo le nuove installazioni configurano ABA automaticamente; i server esistenti passano alla nuova modalità soltanto quando un amministratore avvia il wizard e seleziona esplicitamente **Configure application-based authentication to Microsoft Entra ID**. Il cambio torna quindi dove deve stare: in un change pianificato.

Inoltre, è stata migliorata la gestione del TPM: l'installazione verifica ora preventivamente la capacità di firma di un certificato e gestisce correttamente il controllo della firma TPM. Sui server con firmware TPM difettoso, che non può generare una firma valida, l'installazione ricorre in modo controllato a un certificato basato su software. Anche questo ha una storia: i fallimenti ABA correlati al TPM si sono protratti per diverse release precedenti (2.5.79.0, 2.5.190.0), tra l'altro a causa di incompatibilità tra implementazioni TPM e il metodo di firma predefinito della libreria MSAL.

### I cmdlet PowerShell richiedono ora un accesso amministrativo esplicito

Una modifica che gli operatori di script devono tenere presente: i cmdlet `Set-ADSyncAADCompanyFeature` e `Set-ADSyncAADPasswordSyncState`, che modificano la configurazione cloud, richiedono ora il parametro `-AADUsername` per un'autenticazione amministrativa interattiva. Anche il wizard stesso non esegue più modifiche cloud con credenziali di servizio salvate, bensì tramite un accesso MSAL interattivo. E il wizard di disinstallazione richiede credenziali amministrative per pulire la configurazione cloud; se si salta questo passaggio, viene eseguita la pulizia solo in locale.

Il contesto è lo stesso filo conduttore di ABA: le azioni sul tenant devono essere attribuite a una reale identità amministrativa tracciabile anziché a un account di servizio anonimo. Questo si collega a un bugfix della stessa release: finora, l'admin audit logging registrava l'identità dell'account di servizio invece dell'amministratore effettivamente operante per le modifiche alle regole di sincronizzazione: una traccia di audit che non assolve al proprio scopo. Solo insieme le due modifiche producono un auditing utile. La conseguenza pratica: chi finora ha richiamato questi cmdlet senza supervisione negli script deve ristrutturare tali procedure: autenticazione interattiva e automazione non sono compatibili.

### Rimosso il self-healing di PHS

La modifica più discreta, ma concettualmente più interessante: Password Hash Synchronization non riattiva più autonomamente in background il proprio feature flag cloud. Se il flag è disattivato, un amministratore deve riattivarlo esplicitamente.

Finora valeva quanto segue: se PHS veniva disattivato a livello di tenant, intenzionalmente o accidentalmente, la funzionalità si «riparava» da sola e si riattivava. Per gli ambienti che avevano disattivato PHS intenzionalmente, ad esempio per ragioni di compliance perché gli hash delle password non possono essere inviati al cloud, oppure durante una fase di migrazione, si trattava di una funzionalità che prevaleva su una decisione documentata dell'amministratore. Era difficile giustificare il fatto che proprio un meccanismo che sincronizza hash di password si riattivasse autonomamente.

Tuttavia non va taciuto il rovescio della medaglia: il self-healing ha anche salvato ambienti in cui il flag era stato disattivato da un errore o da uno script non riuscito, senza che nessuno se ne accorgesse. Questa protezione viene ora meno. Chi utilizza PHS in produzione, anche solo come fallback per l'accesso di emergenza, dovrebbe monitorarne attivamente lo stato in futuro, ad esempio tramite Entra Connect Health o controllando i valori heartbeat della sincronizzazione.

### Componenti aggiornati: SQL LocalDB 2022, MSAL, runtime VC++

Meno spettacolare, ma attesa da tempo, è la modernizzazione dei componenti inclusi:

- **SQL Server LocalDB 2019 → 2022.** Finora il database interno di Entra Connect era basato su SQL Server 2019 Express LocalDB, una versione il cui supporto mainstream è terminato nel febbraio 2025. Con SQL Server 2022 l'installazione torna a utilizzare una versione ancora supportata.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library è il componente centrale per l'acquisizione di tutti i token (ABA, accesso del wizard, PowerShell). Il salto di circa venti versioni minori include le correzioni e i miglioramenti accumulati nella libreria.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Qui è degna di nota meno la novità dell'aggiornamento che l'eredità precedente: fino a questa release, Entra Connect richiedeva un ambiente runtime il cui supporto è terminato nell'aprile 2024. La dipendenza da VC++ 2013 è stata ora completamente rimossa.

A questo si aggiunge l'indicazione generica nelle note di rilascio secondo cui sono state corrette «multiple security vulnerabilities in bundled third-party dependencies». Questo dovrebbe essere il motivo principale della classificazione come release di sicurezza: componenti inclusi obsoleti non sono un problema cosmetico in un prodotto che opera con privilegi vicini a Domain Admin al centro dell'infrastruttura di identità.

## Gli altri bugfix

Per completezza, le correzioni rimanenti:

- **Ricerca Metaverse nel Synchronization Service Manager** corretta. Dopo l'avvertimento di non utilizzare affatto l'interfaccia nelle versioni precedenti, ora sembra essere nuovamente mantenuta.
- **Report diagnostico PowerShell (HTML)** nuovamente renderizzato correttamente; rilevante per chi utilizza `Invoke-ADSyncDiagnostics` nei casi di supporto.
- **Connettore Generic SQL:** la creazione del profilo non riusciva perché i parametri obbligatori non venivano compilati nella configurazione. Riguarda gli ambienti che collegano directory aggiuntive tramite il connettore GSQL.
- **China Cloud:** il nome dell'istanza non veniva risolto correttamente dall'API dell'endpoint di discovery, il che poteva causare il fallimento del rilevamento dell'istanza cloud.
- **Admin audit logging** ora registra l'amministratore effettivo anziché l'account di servizio per le modifiche alle regole di sincronizzazione (vedi sopra).

## Scadenze del supporto: chi deve comunque agire ora

Da marzo 2023, per Entra Connect Sync 2.x si applica una rigorosa politica di ritiro: ogni versione esce dal supporto dodici mesi dopo la pubblicazione della versione successiva. Le scadenze attuali:

| Versione | Fine del supporto |
| --- | --- |
| 2.5.3.0 | **31 luglio 2026** |
| 2.5.76.0 | 1 settembre 2026 |
| 2.5.79.0 | 23 ottobre 2026 |
| 2.5.190.0 | 2 febbraio 2027 |
| 2.6.1.0 | 10 marzo 2027 |
| 2.6.3.0 | 7 luglio 2027 |

Chi utilizza ancora la 2.5.3.0 ha quindi soltanto due settimane di supporto rimanenti. Qui la domanda non è se aggiornare, ma soltanto a quale versione. Microsoft sottolinea inoltre che le versioni non più supportate possono smettere di funzionare «unexpectedly»; per le versioni 1.x ritirate, la sincronizzazione è ormai effettivamente disattivata lato server. I requisiti minimi restano .NET Framework 4.7.2 e TLS 1.2; l'installer è disponibile esclusivamente nell'Entra Admin Center (Entra ID → Entra Connect → Get started), non più nel Download Center.

## Raccomandazione in base alla versione di partenza

Microsoft raccomanda di aggiornare «il prima possibile». Tuttavia, la stessa raccomandazione figurava identica anche per la versione 2.6.79.0, poi ritirata. La recente storia delle release, con installer ritirato, hotfix per server arrestati e avvisi sull'interfaccia per diverse versioni, giustifica una valutazione sobria anziché un riflesso automatico.

La mia valutazione per gli ambienti tipici:

**Attendere alcune settimane è giustificabile** se si utilizza una versione ancora supportata (2.5.190.0 o successiva), nessuno dei problemi corretti vi riguarda in modo urgente e non sono necessarie le nuove funzionalità. In base alle note di rilascio, le vulnerabilità di sicurezza corrette si trovano in componenti di terze parti inclusi; un server Entra Connect dovrebbe comunque essere isolato in modo tale, senza accesso a Internet tranne che agli endpoint Microsoft, senza accessi interattivi e con trattamento Tier 0, da rendere responsabile questa finestra temporale. Se la versione rimane disponibile senza richiamo per alcune settimane e Microsoft avvia il rollout dell'auto-upgrade, questo sarà un segnale di qualità decisamente migliore di qualsiasi annuncio.

**Dovreste agire rapidamente** se si applica uno di questi punti:

- **Avete installato la 2.6.79.0.** In questo caso l'istruzione è chiara: disinstallare e installare la 2.6.84.0, senza aspettare.
- **Utilizzate la 2.5.3.0** (fine del supporto il 31 luglio 2026) o una versione ancora più vecchia, già scaduta.
- **Uno dei problemi corretti vi riguarda concretamente**, ad esempio la configurazione ABA su server TPM, il connettore GSQL o il requisito di audit per cui le modifiche alle regole devono essere attribuite all'amministratore corretto.

Per l'upgrade stesso si applica la consueta procedura, particolarmente consigliabile vista questa cronologia delle release: esportare prima la configurazione, il wizard offre **View or export current configuration**, installare dapprima l'aggiornamento su un server in Staging Mode e verificare lì cicli di sincronizzazione, wizard e rotazione dei certificati, quindi procedere con il server attivo. Chi dispone di un file `miiserver.exe.config` personalizzato deve eseguirne il backup prima dell'aggiornamento e controllare successivamente che il nuovo meccanismo di merge abbia acquisito correttamente le personalizzazioni. Chi utilizza script con `Set-ADSyncAADCompanyFeature` o `Set-ADSyncAADPasswordSyncState` deve inoltre testarli prima del rollout in produzione; altrimenti si interromperanno a causa del nuovo parametro obbligatorio.

## Fonti

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Note di rilascio ufficiali della 2.6.84.0, incluso l'avviso di ritiro della 2.6.79.0, la tabella di ritiro e il problema noto relativo a miiserver.exe.config modificato.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procedura di upgrade, inclusa la migrazione swing tramite un server in Staging Mode.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Funzionamento di Application-Based Authentication, che sostituisce l'account di servizio legacy.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Il nuovo accesso con passkey/FIDO2 nel wizard di installazione tramite Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Meccanismo e requisiti dell'auto-upgrade, il cui rollout per la 2.6.84.0 è ancora in attesa.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): L'admin audit logging, la cui attribuzione dell'identità per le regole di sincronizzazione è stata corretta in questa release.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Date di supporto della base LocalDB precedentemente inclusa, il cui supporto mainstream è terminato nel febbraio 2025.
