---
slug: "entra-connect-sync-2-6-84-0-cosa-cambia-e-chi-dovrebbe-aggiornare-ora"
title: "Entra Connect Sync 2.6.84.0: cosa cambia e chi dovrebbe aggiornare ora"
navTitle: "Entra Connect 2.6.84"
description: "La release di sicurezza introduce il supporto per le passkey e modifiche all'autenticazione delle app, a PowerShell e a Password Hash Sync. La versione precedente è stata ritirata; per questo l'aggiornamento richiede una decisione graduale."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min di lettura"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
translationId: article-85bd27acb917e406
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
translatedAt: 2026-09-05T07:45:09.665Z
url: https://rafaelpfister.ch/it/blog/entra-connect-sync-2-6-84-0-cosa-cambia-e-chi-dovrebbe-aggiornare-ora
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0: cosa cambia e chi dovrebbe aggiornare ora

Microsoft ha rilasciato Entra Connect Sync 2.6.84.0 il 7 luglio 2026 come release di sicurezza e raccomanda un aggiornamento rapido. Al contempo, la versione immediatamente precedente, la 2.6.79.0, è stata ritirata a causa di un problema dell'installer scoperto successivamente. La conseguenza non è né «installarla subito ovunque» né «aspettare e ignorarla»: i sistemi interessati e quelli che stanno per uscire dal supporto dovrebbero migrare rapidamente, mentre tutti gli altri possono prima testare l'aggiornamento in modo controllato.

## Perché questa release richiede particolare cautela

La linea 2.6 di Entra Connect Sync ha avuto un avvio accidentato. Un breve riepilogo, poiché è rilevante per la decisione di aggiornamento:

- **2.6.1.0** (febbraio 2026) ha corretto, tra l'altro, un errore per cui la modifica della configurazione del connettore Entra ID nel Synchronization Service Manager cancellava i parametri dell'Application-Based Authentication, causando il fallimento del wizard e della rotazione dei certificati. Per tutte le versioni 2.5 valeva quindi la singolare raccomandazione di non usare affatto l'interfaccia di gestione del prodotto.
- **2.6.3.0** (marzo 2026) era un hotfix per un problema che poteva arrestare inaspettatamente il server Entra Connect durante l'aggiornamento automatico. La soluzione temporanea era la seguente: l'aggiornamento automatico riconosce i file di configurazione modificati manualmente e semplicemente salta tali server.
- **2.6.79.0** (giugno 2026) è stata ritirata completamente dopo la pubblicazione. L'installer non è più disponibile; secondo Microsoft, chi ha installato questa versione deve disinstallarla e installare la 2.6.84.0. Microsoft non documenta quale fosse esattamente il problema.

Alla data odierna, la versione 2.6.84.0 è disponibile solo per il download tramite il Microsoft Entra Admin Center («Released for download»). Non è stato ancora annunciato il rollout dell'aggiornamento automatico. Anche questo è un segnale: Microsoft stessa non sta ancora distribuendo la versione su vasta scala alle installazioni esistenti.

## Nuove funzionalità

### Accesso resistente al phishing nel wizard di configurazione (Preview)

Il wizard di configurazione ora supporta l'accesso con passkey e chiavi di sicurezza FIDO2 tramite Windows Web Account Manager (WAM). Il contesto è il seguente: dal 2024/2025 Microsoft sta imponendo gradualmente l'MFA per gli accessi alle interfacce amministrative di Azure e Entra, e molte organizzazioni hanno limitato i propri account amministrativi, tramite Conditional Access, a metodi resistenti al phishing (FIDO2, passkey, autenticazione basata su certificati). Proprio questi account adeguatamente protetti non potevano finora accedere al wizard di Entra Connect, perché la finestra di accesso incorporata non supportava tali metodi. Nella pratica questo portava a workaround poco eleganti, ad esempio account di configurazione dedicati con requisiti di autenticazione più deboli, solo per completare il wizard. Questa lacuna viene ora colmata, anche se per il momento come Preview.

### Supporto per la Sovereign Cloud francese

La 2.6.84.0 introduce il supporto per l'ambiente Sovereign Cloud francese, inclusi Pass-through Authentication, Seamless Single Sign-On, Password Writeback e il monitoraggio Health Agent. Di conseguenza è stato corretto anche un errore per cui il nome della cloud Application Proxy nella France Cloud non veniva risolto correttamente e la registrazione PTA falliva con «EnvironmentName attribute is invalid».

## Modifiche di comportamento nel dettaglio

La parte più interessante della release non sono le nuove funzioni, bensì i comportamenti modificati. Diversi di essi correggono decisioni progettuali che nella pratica hanno causato sorprese.

### L'aggiornamento automatico non distrugge più i file di configurazione personalizzati

Questa è la modifica con la storia più lunga. Finora, l'aggiornamento automatico sovrascriveva completamente il file `miiserver.exe.config` durante l'update. Le modifiche manuali andavano perse. Può sembrare un caso marginale, ma non lo era: Microsoft stessa aveva indicato agli amministratori in ambienti FIPS di modificare proprio questo file affinché Password Hash Synchronization funzionasse con la modalità FIPS attivata. Chi aveva seguito le istruzioni ufficiali disponeva quindi di un file di configurazione «modificato».

Le conseguenze si sono manifestate durante l'aggiornamento alla 2.5.190.0 e alla 2.6.1.0 come problema noto: se l'installer rileva un file `miiserver.exe.config` modificato, lo lascia intatto; tuttavia manca quindi il nuovo assembly binding e il servizio di sincronizzazione fallisce dopo l'aggiornamento con `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Il workaround documentato: aggiungere manualmente un bindingRedirect nella sezione `assemblyBinding` di `miiserver.exe.config` (in `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Quindi riavviare il servizio ADSync. L'hotfix 2.6.3.0 ha attenuato il problema solo per l'aggiornamento automatico: i server interessati venivano semplicemente saltati e rimanevano alla versione precedente. Con la 2.6.84.0 arriva la soluzione vera e propria: il processo di aggiornamento unisce le personalizzazioni del cliente alla nuova configurazione e ne convalida il risultato prima di applicarlo. Chi esegue un aggiornamento manuale da una versione interessata dovrebbe comunque verificare prima lo stato di `miiserver.exe.config` ed eseguire il backup del file: il meccanismo di merge è nuovo e pertanto non ancora collaudato nella pratica.

### Application-Based Authentication: fine del fallback silenzioso e della conversione silenziosa

Per ricordare: dalla versione 2.5.76.0, l'Application-Based Authentication (ABA) è generally available ed è lo standard. Invece del precedente Directory Synchronization Account, un account cloud con password memorizzata, il server di sincronizzazione si autentica come applicazione Entra ID mediante un certificato, idealmente protetto da TPM. Si tratta di un'architettura molto più robusta: nessuna password che possa essere sottratta e una credenziale vincolata alla macchina.

La 2.6.84.0 corregge due comportamenti che hanno indebolito questo guadagno in termini di sicurezza:

**Niente più fallback silenzioso.** Se la configurazione ABA falliva nel wizard, finora il setup tornava senza commenti all'account legacy. Il risultato: l'amministratore riteneva di avere un accesso basato su certificato, ma il server funzionava in realtà con il vecchio account con password. Un classico schema fail-open. Ora il wizard si interrompe con un chiaro messaggio di errore («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), affinché venga risolta la vera causa invece di nasconderla.

**Nessuna conversione automatica in background.** Finora Entra Connect convertiva autonomamente i server esistenti dall'account legacy ad ABA durante il normale funzionamento della sincronizzazione. Ben intenzionato dal punto di vista della sicurezza, ma un rischio significativo dal punto di vista operativo: un metodo di autenticazione cambia senza preavviso, senza finestra di change e senza che qualcuno ne sia al corrente. E se qualcosa va storto (problemi TPM, conflitti con Conditional Access, firewall), la sincronizzazione si ferma. La nuova regola è: solo le nuove installazioni configurano ABA automaticamente; i server esistenti passano solo quando un amministratore avvia il wizard e seleziona esplicitamente **Configure application-based authentication to Microsoft Entra ID**. La conversione torna così dove dovrebbe essere: in un change pianificato.

Inoltre, è stata migliorata la gestione del TPM: il setup ora verifica in anticipo la capacità di firma di un certificato e gestisce correttamente la verifica della firma TPM. Sui server con firmware TPM difettoso, che non è in grado di generare una firma valida, il setup ripiega in modo controllato su un certificato basato sul software. Anche questo ha una storia: gli errori ABA legati al TPM si sono protratti attraverso diverse release precedenti (2.5.79.0, 2.5.190.0), anche a causa di incompatibilità tra implementazioni TPM e il metodo di firma standard della libreria MSAL.

### I cmdlet PowerShell richiedono ora un accesso amministrativo esplicito

Una modifica che gli operatori degli script devono conoscere: i cmdlet `Set-ADSyncAADCompanyFeature` e `Set-ADSyncAADPasswordSyncState`, che modificano la configurazione cloud, richiedono ora il parametro `-AADUsername` per un'autenticazione amministrativa interattiva. Anche il wizard stesso non esegue più modifiche cloud con credenziali di servizio memorizzate, ma tramite un accesso MSAL interattivo. Inoltre, il wizard di disinstallazione richiede credenziali amministrative per ripulire la configurazione cloud; se si salta questo passaggio, viene eseguita solo la pulizia locale.

Il contesto segue lo stesso filo conduttore dell'ABA: le azioni sul tenant devono essere attribuibili a un'identità amministrativa reale e tracciabile, invece che a un account di servizio anonimo. Questo si collega a una correzione di bug della stessa release: in precedenza, il logging di audit amministrativo registrava l'identità dell'account di servizio anziché quella dell'amministratore che aveva effettivamente apportato le modifiche alle regole di sincronizzazione: una traccia di audit che non assolve al suo scopo. Solo insieme, le due modifiche producono un auditing utilizzabile. La conseguenza pratica è che chi finora chiamava questi cmdlet senza supervisione negli script deve ristrutturare tali processi: autenticazione interattiva e automazione non si conciliano.

### Rimosso il self-healing di PHS

La modifica più discreta, ma concettualmente interessante: Password Hash Synchronization non riattiva più autonomamente in background il proprio feature flag cloud. Se il flag è disattivato, un amministratore deve riattivarlo esplicitamente.

Finora valeva quanto segue: se PHS veniva disattivato a livello di tenant, intenzionalmente o per errore, la funzionalità si «autoriparava» e si riattivava. Per gli ambienti che avevano disabilitato intenzionalmente PHS, ad esempio per motivi di compliance, perché gli hash delle password non devono fluire nel cloud, oppure durante una fase di migrazione, si trattava di una funzione che prevaleva su una decisione amministrativa documentata. Che proprio un meccanismo che sincronizza hash di password si riattivasse autonomamente era difficile da giustificare.

Non va però taciuto il rovescio della medaglia: il self-healing ha anche salvato ambienti in cui il flag era stato disattivato da un errore o da uno script mal riuscito, senza che nessuno se ne accorgesse. Questa protezione ora viene meno. Chi utilizza PHS in produzione, anche solo come fallback per l'accesso di emergenza, dovrebbe in futuro monitorare attivamente lo stato di PHS, ad esempio tramite Entra Connect Health o controllando i valori heartbeat della sincronizzazione.

### Componenti aggiornati: SQL LocalDB 2022, MSAL, runtime VC++

Meno spettacolare, ma attesa da tempo, è la modernizzazione dei componenti inclusi:

- **SQL Server LocalDB 2019 → 2022.** Il database interno di Entra Connect si basava finora su SQL Server 2019 Express LocalDB, una versione il cui supporto mainstream è terminato nel febbraio 2025. Con SQL Server 2022, l'installazione torna a una versione con supporto ancora attivo.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library è il componente centrale per l'acquisizione di tutti i token (ABA, accesso al wizard, PowerShell). Il salto di circa venti versioni minor incorpora le correzioni e i miglioramenti accumulati nella libreria.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Qui è notevole non tanto l'aggiornamento quanto l'eredità del passato: fino a questa release, Entra Connect richiedeva un ambiente runtime il cui supporto è terminato nell'aprile 2024. La dipendenza da VC++ 2013 è ora completamente rimossa.

A ciò si aggiunge l'indicazione generica nelle note di rilascio secondo cui sono state risolte «multiple security vulnerabilities in bundled third-party dependencies». Questo dovrebbe essere il motivo principale della classificazione come release di sicurezza: componenti inclusi obsoleti non sono un problema cosmetico in un prodotto che opera, con privilegi prossimi a quelli di Domain Admin, al centro dell'infrastruttura di identità.

## Le altre correzioni di bug

Per completezza, le restanti correzioni:

- **Ricerca nel metaverse nel Synchronization Service Manager** riparata. Dopo l'avvertenza di non utilizzare affatto l'interfaccia nelle versioni precedenti, ora sembra che venga nuovamente mantenuta.
- **Report diagnostico PowerShell (HTML)** di nuovo renderizzato correttamente; rilevante per chi usa `Invoke-ADSyncDiagnostics` nei casi di supporto.
- **Connettore Generic SQL:** la creazione del profilo falliva perché i parametri obbligatori non venivano popolati durante la configurazione. Riguarda gli ambienti che connettono directory aggiuntive tramite il connettore GSQL.
- **China Cloud:** il nome dell'istanza non veniva risolto correttamente dall'API dell'endpoint Discovery, con il rischio di far fallire il rilevamento dell'istanza cloud.
- **Logging di audit amministrativo** ora registra l'amministratore effettivo anziché l'account di servizio per le modifiche alle regole di sincronizzazione (vedi sopra).

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

Chi utilizza ancora la 2.5.3.0 ha quindi solo due settimane di supporto rimanenti. La domanda non è se aggiornare, ma solo a quale versione. Microsoft sottolinea inoltre che le versioni fuori supporto possono smettere di funzionare «unexpectedly»; per le versioni 1.x ritirate, la sincronizzazione è ormai effettivamente disattivata lato server. I requisiti minimi rimangono .NET Framework 4.7.2 e TLS 1.2; l'installer è disponibile esclusivamente nell'Entra Admin Center (Entra ID → Entra Connect → Get started), non più nel Download Center.

## Raccomandazione in base alla versione di partenza

Microsoft consiglia di aggiornare «il più rapidamente possibile». Tuttavia, questa raccomandazione figurava con le stesse parole anche per la versione 2.6.79.0, quella successivamente ritirata. La recente storia delle release, con installer ritirato, hotfix per server arrestati e avvisi sull'interfaccia per più versioni, giustifica una valutazione sobria anziché una reazione riflessa.

La mia valutazione per gli ambienti tipici:

**Aspettare alcune settimane è giustificabile** se si esegue una versione ancora supportata (2.5.190.0 o più recente), nessuno dei problemi corretti incide con urgenza e non è necessaria alcuna delle nuove funzionalità. In base alle note di rilascio, le vulnerabilità di sicurezza corrette risiedono in componenti di terze parti inclusi; un server Entra Connect dovrebbe comunque essere isolato in modo tale, nessun accesso a Internet tranne che agli endpoint Microsoft, nessun accesso interattivo, trattamento Tier 0, da rendere questo intervallo di tempo accettabile. Se la versione resta per alcune settimane senza richiamo e Microsoft avvia il rollout dell'aggiornamento automatico, si tratta di un segnale di qualità decisamente migliore di qualsiasi annuncio.

**Dovreste agire rapidamente** se si applica uno di questi punti:

- **Avete installato la 2.6.79.0.** In tal caso l'istruzione è inequivocabile: disinstallare e installare la 2.6.84.0, senza aspettare.
- **Utilizzate la 2.5.3.0** (fine del supporto il 31 luglio 2026) o una versione ancora più vecchia, già fuori supporto.
- **Uno dei problemi corretti vi riguarda concretamente**, ad esempio la configurazione ABA su server TPM, il connettore GSQL oppure il requisito di audit che le modifiche alle regole siano attribuite all'amministratore corretto.

Per l'aggiornamento stesso vale la procedura consueta, particolarmente consigliata vista questa cronologia di release: esportare prima la configurazione, il wizard offre **View or export current configuration**, applicare prima l'update a un server in modalità staging e verificare lì cicli di sincronizzazione, wizard e rotazione dei certificati; solo dopo aggiornare il server attivo. Chi dispone di un file `miiserver.exe.config` personalizzato ne esegue il backup prima dell'update e controlla in seguito se il nuovo meccanismo di merge ha acquisito correttamente le personalizzazioni. E chi utilizza script con `Set-ADSyncAADCompanyFeature` o `Set-ADSyncAADPasswordSyncState` li testa prima del rollout in produzione; altrimenti si interromperanno sul nuovo parametro obbligatorio.

## Fonti

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Note di rilascio ufficiali della 2.6.84.0, compreso l'avviso di ritiro della 2.6.79.0, la tabella di ritiro e il problema noto relativo a miiserver.exe.config modificato.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procedura di aggiornamento, inclusa la migrazione swing tramite un server in modalità staging.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Funzionamento dell'Application-Based Authentication, che sostituisce il precedente account di servizio.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): Il nuovo accesso tramite passkey/FIDO2 nel wizard di configurazione tramite Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Meccanismo e requisiti dell'aggiornamento automatico, il cui rollout per la 2.6.84.0 è ancora in attesa.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): Il logging di audit amministrativo, la cui attribuzione dell'identità per le regole di sincronizzazione è stata corretta in questa release.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Date di supporto della base LocalDB finora inclusa, il cui supporto mainstream è terminato nel febbraio 2025.
