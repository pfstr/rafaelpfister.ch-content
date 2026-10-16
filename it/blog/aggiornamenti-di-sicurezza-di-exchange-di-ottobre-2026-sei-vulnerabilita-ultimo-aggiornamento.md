---
title: "Aggiornamenti di sicurezza di Exchange di ottobre 2026: sei vulnerabilità, ultimo aggiornamento per Exchange 2016 e 2019"
navTitle: "Exchange SU 10/2026"
description: "L'SU di ottobre chiude sei vulnerabilità, tra cui un'esecuzione di codice remoto critica tramite un'e-mail predisposta, e risolve l'errore .ics nonché il deadlock di WordBreaker. Per Exchange 2016 e 2019 è l'ultimo aggiornamento, anche nel programma ESU."
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
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-ottobre-2026-sei-vulnerabilita-ultimo-aggiornamento
translationSourceHash: ba4b8563ea0b25f5badf03f09e69f9b8329bb75bf4c2422cb5c3fbb97f55d137
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T08:50:36.125Z
translationReview: automatic
---

# Aggiornamenti di sicurezza di Exchange di ottobre 2026: sei vulnerabilità, ultimo aggiornamento per Exchange 2016 e 2019

Il 13 ottobre 2026 Microsoft ha pubblicato aggiornamenti di sicurezza (SU) per Exchange Server SE, Exchange 2019 e Exchange 2016. Secondo gli articoli KB, chiudono sei vulnerabilità. Nessuna era nota pubblicamente in anticipo e, secondo la Security Update Guide, nessuna è sfruttata attivamente. Microsoft classifica però due di esse come «Exploitation More Likely», tra cui un'esecuzione di codice remoto valutata *Critical* che un aggressore non autenticato può attivare con un'e-mail predisposta. Questo mese è particolare per due ragioni: l'SU risolve i due Known Issues aperti degli aggiornamenti di settembre (HTTP 500 per i calendari pubblicati e il deadlock di ContentEngine in Exchange SE), e per Exchange 2016 e 2019 è, secondo Microsoft, l'ultimo aggiornamento di sicurezza in assoluto, anche per i clienti nel programma ESU Period 2.

## Per quali versioni di Exchange è disponibile l'aggiornamento

Gli SU del 13 ottobre 2026 sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, build 15.2.2562.56; disponibile pubblicamente.
- **Exchange Server 2019 CU15**: KB5128896, build 15.2.1748.55; solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5128897, build 15.2.1544.50; solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5128898, build 15.1.2507.77; solo tramite ESU Period 2.

Potete confrontare la vostra versione con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

Exchange 2016 e 2019 sono fuori supporto. Secondo gli articoli KB, le organizzazioni nel programma ESU Period 2 ricevono aggiornamenti di sicurezza fino alla fine di ottobre 2026; tutte le altre devono migrare a Exchange SE. Secondo l'annuncio, Exchange Online è già protetto; negli ambienti ibridi ogni server Exchange necessita comunque dell'SU, inclusi i server di sola gestione e le macchine con gli Exchange Management Tools. Il significato concreto della fine dell'ESU è illustrato nella sezione [Ultimo aggiornamento per Exchange 2016 e 2019](#letztes-update-für-exchange-2016-und-2019).

## Panoramica delle vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-77910 | Elevation of Privilege | 8.8 |
| CVE-2026-70571 | Remote Code Execution | 8.1 |
| CVE-2026-87692 | Elevation of Privilege | 7.8 |
| CVE-2026-87695 | Elevation of Privilege | 7.8 |
| CVE-2026-87696 | Elevation of Privilege | 7.5 |

Tutti e quattro gli articoli KB elencano le stesse sei CVE. Nella Security Update Guide, tuttavia, per CVE-2026-87692 manca Exchange 2019 CU15; vi sono indicati come interessati solo Exchange SE, 2019 CU14 e 2016 CU23 (situazione al 14 ottobre 2026). Frankys Web interpreta questo dato nel senso che 2019 CU15 non è interessato da questa vulnerabilità. Per l'installazione la differenza non ha rilevanza, poiché l'SU per CU15 chiude comunque tutte le altre vulnerabilità.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** è l'unica vulnerabilità del mese che Microsoft classifica come *Critical* (Remote Code Execution, CVSS 8.1). La causa è un buffer overflow basato sullo stack. Secondo le FAQ della Security Update Guide, un aggressore non autenticato può inviare un'e-mail predisposta a un server Exchange interessato e attivare così un processo interno di conversione; non è necessaria alcuna interazione dell'utente. Secondo il vettore CVSS, la complessità dell'attacco è elevata (AC:H); Microsoft precisa che lo sfruttamento riuscito non è garantito e richiede una profonda comprensione dell'ambiente di destinazione. Ciononostante, Microsoft valuta lo sfruttamento come «Exploitation More Likely». L'aggressore non necessita di un account nell'organizzazione: è sufficiente che il server accetti e-mail.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** ha il valore più elevato, CVSS 9.3, ed è anch'essa classificata come «Exploitation More Likely». Si tratta di un Cross-Site Scripting: un aggressore non autenticato invia un link predisposto; se un amministratore lo apre, l'aggressore può eseguire azioni con i suoi privilegi e quindi accedere ai dati dell'organizzazione o modificarli. Secondo le FAQ, ciò richiede l'interazione dell'utente, tipicamente tramite social engineering. Il livello di gravità è *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (Elevation of Privilege, CVSS 8.8) richiede solo un account con privilegi ridotti. Secondo le FAQ, un aggressore può manipolare una cache condivisa dell'Exchange Control Panel e far così installare un componente aggiuntivo di posta dannoso nella cassetta postale di un altro utente, senza che quest'ultimo debba fare nulla. Microsoft descrive lo stesso vettore d'attacco per [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), in quel caso come conseguenza di una verifica insufficiente delle firme crittografiche e con elevata complessità dell'attacco.

Le altre due vulnerabilità, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserializzazione di dati non attendibili) e [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), sono escalation locali dei privilegi con CVSS 7.8: secondo Microsoft, un aggressore autenticato con privilegi ridotti può ottenere privilegi SYSTEM. Entrambe sono classificate come «Exploitation Less Likely», così come CVE-2026-77910 e CVE-2026-87696.

Riguardo a **CVE-2026-96940** della versione v2 di settembre: l'annuncio di ottobre non menziona la v2. Secondo le FAQ dell'annuncio, gli SU sono cumulativi; chi utilizza un CU supportato installa solo l'SU più recente. Frankys Web conferma che l'SU di ottobre contiene la correzione della v2. I server su cui è ancora in esecuzione il primo SU di settembre dell'8 settembre possono quindi installare direttamente l'SU di ottobre; i dettagli sulla v2 sono disponibili nell'[articolo sulla v2 di settembre](/blog/exchange-security-updates-september-2026-v2).

## Ultimo aggiornamento per Exchange 2016 e 2019

L'annuncio è inequivocabile su questo punto: ottobre 2026 è l'ultimo mese in cui i clienti del programma ESU Period 2 ricevono nuovi aggiornamenti di sicurezza per Exchange 2016 e 2019. Dalla versione successiva, gli aggiornamenti saranno pubblicati solo per Exchange SE. I futuri annunci e le segnalazioni CVE non menzioneranno più 2016 e 2019, e Microsoft scrive esplicitamente: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» Il fatto che una CVE non menzioni queste versioni non significa quindi che non siano interessate; Microsoft sottolinea nell'annuncio che questo non rende sicure le vecchie versioni. Frankys Web rileva inoltre che Exchange SE e 2019 CU15 sono tecnicamente molto simili, pertanto è altamente probabile che le future vulnerabilità di SE riguardino anche 2019.

Secondo l'annuncio, ESU Period 2 può ancora essere acquistato per un periodo limitato. Tuttavia, porta un'organizzazione solo al livello dell'aggiornamento di ottobre 2026; non seguiranno ulteriori aggiornamenti. Chi ha già acquistato un ESU e ha domande sull'accesso può, secondo gli articoli KB, rivolgersi a `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

Per l'esercizio questo significa:

1. **Exchange 2016 e 2019 con ESU**: installate l'SU di ottobre. In seguito la versione resterà congelata e ogni vulnerabilità resa nota successivamente rimarrà aperta.

2. **Exchange 2016 e 2019 senza ESU**: questi server non ricevono più aggiornamenti di sicurezza da maggio 2026. Microsoft raccomanda di migrare a Exchange SE il prima possibile.

3. **Ambienti ibridi**: oltre all'assenza di patch, Exchange Online limita e blocca il flusso di posta dai server on-premises obsoleti; le regole sono riportate nell'[articolo sul Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problemi risolti e cosa fare successivamente

**I calendari pubblicati (.ics) restituiscono HTTP 500 alle app calendario** (tutte le versioni). Il problema esistente dall'SU di agosto è risolto con l'SU di ottobre; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) indica ora come soluzione l'installazione dei quattro aggiornamenti di ottobre o di versioni successive. L'articolo di supporto non descrive più il workaround precedente di URL rewrite sul sito «Exchange Back End» e Microsoft non fornisce istruzioni per rimuovere la regola (situazione al 14 ottobre 2026). Se avete configurato la regola, documentate su quali server è attiva e testate un abbonamento a un calendario prima di rimuoverla.

**Deadlock di ContentEngine dovuto alla mancanza di file WordBreaker coreani** (solo Exchange SE). Secondo [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), il problema è risolto con KB5128895 o un aggiornamento successivo e, dopo l'installazione dell'aggiornamento di ottobre, non è necessaria alcuna azione aggiuntiva. Questo vale anche per i server sui quali avete applicato il workaround manuale dell'[articolo di settembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft non richiede di annullare la copia dei file di regole. I server senza workaround non devono più aggiungere manualmente i file.

**Disponibilità per cassette postali delegate in ambienti ibridi con configurazione solo Graph API** (solo Exchange SE). Questo problema non viene risolto dall'SU di ottobre: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) riporta ora «The issue has been fixed in the service». La contraddizione rispetto all'[articolo v2](/blog/exchange-security-updates-september-2026-v2) è quindi risolta. Se avete impostato il workaround, rimuovetelo secondo KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Mostra il valore corrente dell'override; il comando successivo è necessario solo con `Enabled=False`. |
| `-Parameters "Enabled=True"` | Riattiva l'inoltro delle richieste di disponibilità tramite Microsoft Graph. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Indirizza la chiamata diagnostica al servizio di topologia. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché la modifica abbia effetto immediato. |

</details>

Successivamente verificate la richiesta di disponibilità di una cassetta postale delegata di Exchange Online da Outlook o OWA.

## Problemi noti

Né l'annuncio né i quattro articoli KB menzionano Known Issues per l'SU di ottobre (situazione al 14 ottobre 2026). I tre problemi aperti degli aggiornamenti di settembre sono quindi risolti: due dall'SU di ottobre, uno da una correzione in Exchange Online (vedi sopra). Il problema del wrapper nelle cassette postali condivise era già stato risolto con l'SU di settembre.

## Installazione e attività successive

Microsoft indica nell'annuncio la procedura consueta: inventariare con [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinare il percorso con [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) in caso di CU obsoleto, installare l'SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. In caso di errori, Microsoft rimanda allo [script SetupAssist](https://aka.ms/ExSetupAssist). Dopo l'installazione, secondo KB5128895 l'Health Checker indica se l'SU è installato correttamente e se sono necessari ulteriori passaggi. La Security Update Guide non indica un riavvio obbligatorio per le build di ottobre; l'annuncio lo raccomanda comunque dopo l'installazione.

Secondo KB5128895, il file di installazione per Exchange SE si chiama `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), per Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, per CU14 `Exchange2019-KB5128897-x64-en.exe` e per Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. I pacchetti sono specifici per CU.

Per gli ambienti ibridi, secondo le FAQ, vale quanto segue: se il certificato di autenticazione viene modificato dopo l'installazione di un SU, è necessario eseguire nuovamente Hybrid Configuration Wizard. In Windows Server 2025 gli SU di Exchange installati non vengono visualizzati nel Pannello di controllo; Microsoft sconsiglia di disinstallare gli SU.

Dai mesi precedenti rimangono aperte queste attività successive, se non sono ancora state svolte: rimuovere il SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders` (vedere l'[articolo di settembre](/blog/exchange-security-updates-september-2026)) e verificare se la mitigazione CVE-2026-42897 (M2.1.0) è ancora attiva (vedere l'[articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026)).

## Procedura consigliata

Installate tempestivamente l'SU di ottobre su tutti i server Exchange e sulle macchine con Management Tools: l'esecuzione di codice remoto critica CVE-2026-70571 può essere attivata senza un account tramite un'e-mail, e Microsoft ritiene il suo sfruttamento probabile quanto quello del Cross-Site Scripting CVE-2026-70279. I server con il primo SU di settembre ricevono così anche la correzione per CVE-2026-96940. Successivamente rimuovete l'override Disponibilità, se impostato, e verificate i calendari pubblicati. Per Exchange 2016 e 2019 questo è l'ultimo aggiornamento; pianificate ora la migrazione a Exchange SE, poiché da novembre questi server non riceveranno più aggiornamenti di sicurezza.

## Fonti

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): annuncio del 13 ottobre 2026 con versioni supportate, problemi risolti, fine degli aggiornamenti per Exchange 2016/2019 e FAQ (consultato tramite il feed RSS dell'Exchange Team Blog; nessun commento del team Exchange, situazione al 14 ottobre 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): elenco CVE, problemi risolti, file di installazione e indicazione sull'Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): articolo KB per Exchange 2019 CU15 con indicazione ESU.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): articolo KB per Exchange 2019 CU14 con indirizzo di contatto ESU.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): articolo KB per Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): tipo, CVSS, gravità, valutazione dello sfruttamento, FAQ e build interessate delle sei CVE di ottobre nonché di CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): numeri di build degli SU di ottobre e degli aggiornamenti precedenti.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): correzione con gli aggiornamenti di ottobre.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): correzione con KB5128895, nessuna ulteriore azione necessaria.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): correzione nel servizio e comandi per ripristinare SettingOverride.

11. [Nuovi aggiornamenti di sicurezza per Exchange Server (ottobre 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): analisi in lingua tedesca delle sei CVE, indicazione della correzione v2 inclusa e classificazione della fine dell'ESU.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): inventario degli stati CU/SU prima e dopo l'installazione.
