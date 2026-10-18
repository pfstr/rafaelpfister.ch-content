---
title: "Aggiornamenti di sicurezza di Exchange di ottobre 2026: sei vulnerabilità, ultimo aggiornamento per Exchange 2016 e 2019"
navTitle: "Exchange SU 10/2026"
description: "L'SU di ottobre risolve sei vulnerabilità, tra cui un'esecuzione di codice in modalità remota critica tramite un'e-mail appositamente predisposta, e corregge l'errore .ics e il deadlock di WordBreaker. Per Exchange 2016 e 2019 è l'ultimo aggiornamento, anche nel programma ESU."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min di lettura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "aggiornamenti-di-sicurezza-di-exchange-di-ottobre-2026-sei-vulnerabilita-ultimo-aggiornamento"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
translationSourceHash: a834949e09b3f4b6e7f16be2dd6cf0ecedd0ccf1f4fc8dd6f409336407aaaca2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:24:16.853Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-ottobre-2026-sei-vulnerabilita-ultimo-aggiornamento
---

# Aggiornamenti di sicurezza di Exchange di ottobre 2026: sei vulnerabilità, ultimo aggiornamento per Exchange 2016 e 2019

Il 13 ottobre 2026 Microsoft ha pubblicato aggiornamenti di sicurezza (SU) per Exchange Server SE, Exchange 2019 ed Exchange 2016. Secondo gli articoli KB, risolvono sei vulnerabilità. Nessuna era nota pubblicamente in anticipo e, secondo la Security Update Guide, nessuna è sfruttata attivamente. Microsoft ne classifica tuttavia due come «Exploitation More Likely», inclusa una vulnerabilità di esecuzione di codice in modalità remota valutata *Critical*, che un aggressore non autenticato può attivare mediante un'e-mail appositamente predisposta. Questo mese è particolare per due motivi: l'SU risolve i due Known Issues aperti degli aggiornamenti di settembre (HTTP 500 per i calendari pubblicati e il deadlock di ContentEngine in Exchange SE), e per Exchange 2016 e 2019 è, secondo Microsoft, l'ultimo aggiornamento di sicurezza in assoluto, anche per i clienti del programma ESU Period 2.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Gli SU del 13 ottobre 2026 sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, build 15.2.2562.56; disponibile pubblicamente.
- **Exchange Server 2019 CU15**: KB5128896, build 15.2.1748.55; solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5128897, build 15.2.1544.50; solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5128898, build 15.1.2507.77; solo tramite ESU Period 2.

È possibile confrontare la propria versione con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

Exchange 2016 e 2019 non sono più supportati. Secondo gli articoli KB, le organizzazioni nel programma ESU Period 2 ricevono aggiornamenti di sicurezza fino alla fine di ottobre 2026; tutte le altre devono migrare a Exchange SE. Exchange Online è già protetto secondo l'annuncio; negli ambienti ibridi, tuttavia, ogni server Exchange necessita comunque dell'SU, inclusi i server di sola gestione e le macchine con gli Exchange Management Tools. Il significato concreto della fine dell'ESU è illustrato nella sezione [Ultimo aggiornamento per Exchange 2016 e 2019](#letztes-update-für-exchange-2016-und-2019).

## Panoramica delle vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

Tutti e quattro gli articoli KB elencano le stesse sei CVE. Nella Security Update Guide, tuttavia, Exchange 2019 CU15 manca per CVE-2026-87692; vi sono elencati come interessati solo Exchange SE, 2019 CU14 e 2016 CU23 (situazione al 14 ottobre 2026). Frankys Web interpreta questo dato nel senso che 2019 CU15 non è interessato da questa vulnerabilità. Per l'installazione la differenza non ha importanza, poiché l'SU su CU15 risolve comunque tutte le altre vulnerabilità.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** è l'unica vulnerabilità del mese che Microsoft classifica come *Critical* (Remote Code Execution, CVSS 8.1). La causa è un buffer overflow basato sullo stack. Secondo le FAQ della Security Update Guide, un aggressore non autenticato può inviare un'e-mail appositamente predisposta a un server Exchange interessato, attivando così un processo di conversione interno; non è necessaria alcuna interazione dell'utente. Secondo il vettore CVSS, la complessità dell'attacco è elevata (AC:H); Microsoft scrive che lo sfruttamento riuscito non è garantito e richiede una conoscenza approfondita dell'ambiente di destinazione. Ciononostante, Microsoft valuta lo sfruttamento come «Exploitation More Likely». L'aggressore non necessita di un account nell'organizzazione: è sufficiente che il server accetti e-mail.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** ha il valore più alto, CVSS 9.3, ed è anch'essa classificata come «Exploitation More Likely». Si tratta di un Cross-Site Scripting: un aggressore non autenticato invia un link appositamente predisposto; se un amministratore lo apre, l'aggressore può eseguire azioni con i suoi privilegi e quindi accedere ai dati dell'organizzazione o modificarli. Secondo le FAQ, ciò richiede un'interazione dell'utente, tipicamente tramite social engineering. Il livello di gravità è *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) richiede soltanto un account con privilegi limitati. Secondo le FAQ, un aggressore può manipolare una cache condivisa dell'Exchange Control Panel e fare così installare un componente aggiuntivo di posta dannoso nella cassetta postale di un altro utente, senza che quest'ultimo debba fare nulla. Microsoft descrive lo stesso vettore di attacco per [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), in quel caso come conseguenza di una verifica insufficiente delle firme crittografiche e con alta complessità dell'attacco.

Le altre due vulnerabilità, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserializzazione di dati non attendibili) e [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), sono elevazioni locali dei privilegi con CVSS 7.8: secondo Microsoft, un aggressore autenticato con privilegi limitati può ottenere privilegi SYSTEM. Entrambe sono classificate come «Exploitation Less Likely», così come CVE-2026-77910 e CVE-2026-87696.

Riguardo a **CVE-2026-96940** di settembre v2: l'annuncio di ottobre non menziona la v2. Secondo le FAQ dell'annuncio, gli SU sono cumulativi; chi utilizza una CU supportata installa solo l'SU più recente. Frankys Web conferma che l'SU di ottobre contiene la correzione della v2. I server che eseguono ancora il primo SU di settembre dell'8 settembre possono quindi installare direttamente l'SU di ottobre; i dettagli sulla v2 sono disponibili nell'[articolo su settembre v2](/blog/exchange-security-updates-september-2026-v2).

## Ultimo aggiornamento per Exchange 2016 e 2019

L'annuncio è chiaro su questo punto: ottobre 2026 è l'ultimo mese in cui i clienti del programma ESU Period 2 ricevono nuovi aggiornamenti di sicurezza per Exchange 2016 e 2019. Dal rilascio successivo, gli aggiornamenti saranno pubblicati solo per Exchange SE. Gli annunci e le notifiche CVE future non citeranno più 2016 e 2019, e Microsoft scrive espressamente: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» Il fatto che una CVE non menzioni queste versioni non significa dunque che non siano interessate; Microsoft sottolinea nell'annuncio che le vecchie versioni non diventano sicure per questo motivo. Frankys Web evidenzia inoltre che Exchange SE e 2019 CU15 sono tecnicamente molto simili, quindi è altamente probabile che le future vulnerabilità di SE riguardino anche 2019.

Secondo l'annuncio, ESU Period 2 è ancora acquistabile per un periodo limitato. Tuttavia, porta un'organizzazione soltanto al livello dell'aggiornamento di ottobre 2026; non seguiranno ulteriori aggiornamenti. Chi ha già acquistato un ESU e ha domande sull'accesso può, secondo gli articoli KB, rivolgersi a `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

Per l'esercizio, ciò significa:

1. **Exchange 2016 e 2019 con ESU**: installare l'SU di ottobre. Successivamente il livello rimarrà congelato e ogni vulnerabilità scoperta in seguito resterà aperta.

2. **Exchange 2016 e 2019 senza ESU**: questi server non ricevono più aggiornamenti di sicurezza da maggio 2026. Microsoft raccomanda di migrare a Exchange SE il prima possibile.

3. **Ambienti ibridi**: oltre all'assenza di patch, Exchange Online limita e blocca il flusso di posta dai server On-Premises obsoleti; le regole sono disponibili nell'[articolo sul Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problemi risolti e operazioni necessarie dopo l'aggiornamento

**I calendari pubblicati (.ics) restituiscono HTTP 500 alle app calendario** (tutte le versioni). Il problema esistente dall'SU di agosto è risolto con l'SU di ottobre; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) indica ora come soluzione l'installazione dei quattro aggiornamenti di ottobre o successivi. L'articolo di supporto non descrive più il precedente workaround di riscrittura URL nel sito «Exchange Back End» e Microsoft non fornisce istruzioni per rimuovere la regola (situazione al 14 ottobre 2026). Se avete configurato la regola, documentate su quali server è attiva e testate un abbonamento a un calendario prima di rimuoverla.

**Deadlock di ContentEngine dovuto a file WordBreaker coreani mancanti** (solo Exchange SE). Secondo [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), il problema è risolto con KB5128895 o un aggiornamento più recente e dopo l'installazione dell'aggiornamento di ottobre non è necessaria alcuna azione aggiuntiva. Ciò vale anche per i server su cui è stato applicato il workaround manuale dell'[articolo di settembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft non richiede di annullare la copia dei file delle regole. I server senza workaround non devono più aggiungere manualmente i file.

**Disponibilità free/busy per cassette postali delegate in ambienti ibridi con configurazione solo Graph API** (solo Exchange SE). Questo problema non viene risolto dall'SU di ottobre: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) ora riporta «The issue has been fixed in the service». La contraddizione dell'[articolo v2](/blog/exchange-security-updates-september-2026-v2) è pertanto risolta. Se avete impostato il workaround, rimuovetelo secondo KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Mostra il valore corrente dell'override; il comando successivo è necessario solo con `Enabled=False`. |
| `-Parameters "Enabled=True"` | Riattiva l'inoltro delle richieste di disponibilità tramite Microsoft Graph. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Indirizza la chiamata diagnostica al servizio di topologia. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché la modifica abbia effetto immediatamente. |

</details>

Successivamente, verificate la richiesta free/busy di una cassetta postale delegata di Exchange Online da Outlook o OWA.

## Problemi noti

Né l'annuncio né i quattro articoli KB menzionano Known Issues per l'SU di ottobre (situazione al 14 ottobre 2026). Il 15 ottobre 2026 Microsoft ha integrato l'annuncio: gli aggiornamenti di Windows Server di ottobre 2026 possono causare l'arresto dei server Exchange dopo circa un giorno con `MapiExceptionMdbOffline` (ec=1142), anche senza un SU di Exchange installato. Sintomi e procedura sono disponibili nell'[articolo sui guasti di Exchange dopo gli aggiornamenti di Windows](/blog/exchange-windows-updates-oktober-2026). I tre problemi aperti degli aggiornamenti di settembre sono quindi risolti: due dall'SU di ottobre, uno da una correzione in Exchange Online (vedi sopra). Il problema del wrapper nelle cassette postali condivise era già stato risolto con l'SU di settembre.

## Installazione e attività successive

Microsoft indica nell'annuncio la procedura nota: inventariare con l'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinare il percorso con l'[Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) in caso di CU obsoleta, installare l'SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. In caso di errori, Microsoft rimanda allo [script SetupAssist](https://aka.ms/ExSetupAssist). Dopo l'installazione, secondo KB5128895, l'Health Checker mostra se l'SU è installato correttamente e se sono necessari ulteriori passaggi. La Security Update Guide non indica un riavvio obbligatorio per le build di ottobre; l'annuncio lo raccomanda comunque dopo il setup.

Secondo KB5128895, il file di installazione per Exchange SE si chiama `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), per Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, per CU14 `Exchange2019-KB5128897-x64-en.exe` e per Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. I pacchetti sono specifici per CU.

Per gli ambienti ibridi, secondo le FAQ, vale quanto segue: se dopo l'installazione di un SU viene cambiato il certificato di autenticazione, è necessario eseguire nuovamente Hybrid Configuration Wizard. In Windows Server 2025, gli SU di Exchange installati non compaiono nel Pannello di controllo; Microsoft sconsiglia di disinstallare gli SU.

Dai mesi precedenti restano aperte queste attività, se non sono già state eseguite: rimuovere il SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders` (vedi [articolo di settembre](/blog/exchange-security-updates-september-2026)) e verificare se la mitigazione CVE-2026-42897 (M2.1.0) è ancora attiva (vedi [articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026)).

## Procedura consigliata

Installate tempestivamente l'SU di ottobre su tutti i server Exchange e sulle macchine con Management Tools: la vulnerabilità critica di esecuzione di codice in modalità remota CVE-2026-70571 può essere attivata tramite e-mail senza account, e Microsoft considera più probabile il suo sfruttamento, così come quello del Cross-Site Scripting CVE-2026-70279. I server con il primo SU di settembre riceveranno così anche la correzione per CVE-2026-96940. Successivamente rimuovete l'override free/busy, se impostato, e verificate i calendari pubblicati. Per Exchange 2016 e 2019 questo è l'ultimo aggiornamento; pianificate ora la migrazione a Exchange SE, poiché da novembre questi server non riceveranno più aggiornamenti di sicurezza.

## Fonti

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Annuncio del 13 ottobre 2026 con versioni supportate, problemi risolti, fine degli aggiornamenti per Exchange 2016/2019 e FAQ (consultato tramite il feed RSS dell'Exchange Team Blog; nessun commento del team Exchange, situazione al 14 ottobre 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): Elenco delle CVE, problemi risolti, file di installazione e indicazione sull'Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): Articolo KB per Exchange 2019 CU15 con indicazione ESU.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): Articolo KB per Exchange 2019 CU14 con indirizzo di contatto ESU.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): Articolo KB per Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravità, valutazione dello sfruttamento, FAQ e build interessate delle sei CVE di ottobre e di CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numeri di build degli SU di ottobre e degli aggiornamenti precedenti.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Correzione con gli aggiornamenti di ottobre.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Correzione con KB5128895, nessuna ulteriore azione necessaria.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Correzione nel servizio e comandi per reimpostare il SettingOverride.

11. [Nuovi aggiornamenti di sicurezza per Exchange Server (ottobre 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): Analisi in lingua tedesca delle sei CVE, indicazione della correzione v2 inclusa e contestualizzazione della fine dell'ESU.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario degli stati CU/SU prima e dopo l'installazione.
