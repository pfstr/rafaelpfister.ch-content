---
title: "Aggiornamenti di sicurezza di Exchange di novembre 2026: nove vulnerabilità, primo aggiornamento solo per Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "L'SU di novembre chiude nove vulnerabilità in Exchange Server SE, tra cui un'esecuzione di codice in modalità remota critica per la quale è sufficiente un account con privilegi ridotti. Exchange 2016 e 2019 non ricevono più aggiornamenti. L'aggiornamento di Windows di novembre risolve le interruzioni di Exchange successive agli aggiornamenti di Windows di ottobre."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min. di lettura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "aggiornamenti-di-sicurezza-di-exchange-di-novembre-2026-nove-vulnerabilita-primo-aggiornamento"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-novembre-2026-nove-vulnerabilita-primo-aggiornamento
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:24:27.558Z
translationReview: automatic
---

# Aggiornamenti di sicurezza di Exchange di novembre 2026: nove vulnerabilità, primo aggiornamento solo per Exchange SE

Il 10 novembre 2026 Microsoft ha pubblicato un aggiornamento di sicurezza (SU) per Exchange Server SE. Secondo l'articolo KB, corregge nove vulnerabilità. Nessuna era nota pubblicamente in anticipo e, secondo la Security Update Guide, nessuna viene sfruttata attivamente. Microsoft classifica un'esecuzione di codice in modalità remota come *Critical* e «Exploitation More Likely»; per sfruttarla è sufficiente un account con privilegi ridotti. Questo mese è particolare per due motivi: è il primo SU pubblicato solo per Exchange SE e le interruzioni di Exchange successive agli aggiornamenti di Windows Server di ottobre sono risolte con gli aggiornamenti di Windows di novembre.

## Per quali versioni di Exchange è disponibile l'aggiornamento

L'SU del 10 novembre 2026 è disponibile per una sola versione:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, build 15.2.2562.57; disponibile pubblicamente.

Potete confrontare la vostra versione con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

Per Exchange 2016 e 2019 non è disponibile alcun aggiornamento. L'[SU di ottobre](/blog/exchange-security-updates-oktober-2026) è stato l'ultimo, anche per i clienti del programma ESU Period 2, e secondo l'annuncio il programma è terminato. Secondo Microsoft, Exchange Online è già protetto; negli ambienti ibridi ogni server Exchange richiede comunque l'SU, inclusi i server destinati esclusivamente alla gestione e le macchine con gli Exchange Management Tools.

## Panoramica delle vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Esecuzione di codice in modalità remota | 8.8 |
| CVE-2026-118149 | Esecuzione di codice in modalità remota | 8.8 |
| CVE-2026-118150 | Elevazione dei privilegi | 8.8 |
| CVE-2026-118154 | Elevazione dei privilegi | 8.8 |
| CVE-2026-118153 | Denial of Service | 7.5 |
| CVE-2026-118151 | Elevazione dei privilegi | 7.2 |
| CVE-2026-118157 | Elevazione dei privilegi | 7.2 |
| CVE-2026-118208 | Spoofing | 7.1 |
| CVE-2026-118152 | Spoofing | 6.5 |

Tutte e nove le vulnerabilità sono raggiungibili tramite la rete (AV:N) e, secondo il vettore CVSS, hanno una bassa complessità di attacco (AC:L). La Security Update Guide indica come unico prodotto interessato Exchange Server SE RTM.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) è l'unica vulnerabilità del mese che Microsoft classifica come *Critical* (Remote Code Execution, CVSS 8.8) e l'unica con la valutazione «Exploitation More Likely». Secondo le FAQ nella Security Update Guide, un aggressore autenticato con privilegi ridotti può inviare una richiesta predisposta a un server interessato ed eseguire così codice sul server; non è necessaria alcuna interazione dell'utente. Microsoft indica come causa una convalida insufficiente dell'input (CWE-706). Per l'attacco basta quindi un normale account di posta elettronica, ad esempio con una password ottenuta tramite phishing.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) è la seconda vulnerabilità di esecuzione di codice in modalità remota (CVSS 8.8, *Important*). Anche in questo caso, secondo le FAQ, un aggressore autenticato invia una richiesta predisposta a un servizio interessato e può quindi eseguire codice sul sistema di destinazione; la causa è una code injection (CWE-94). Microsoft valuta lo sfruttamento come «Exploitation Unlikely».

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (Denial of Service, CVSS 7.5) è l'unica vulnerabilità che non richiede né un account né l'interazione dell'utente. Secondo la descrizione, un aggressore non autenticato può mettere fuori servizio il servizio tramite la rete; la causa è una complessità algoritmica inefficiente (CWE-407). Secondo il vettore, è interessata solo la disponibilità, non la riservatezza o l'integrità.

Le quattro elevazioni dei privilegi differiscono nei requisiti: per [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) e [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (entrambi CVSS 8.8) basta un account con privilegi ridotti, mentre per [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) e [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (entrambi CVSS 7.2) l'aggressore deve già disporre di privilegi elevati. Per CVE-2026-118151, secondo le FAQ, può ottenere privilegi SYSTEM.

Delle due vulnerabilità di spoofing, [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) richiede un account con privilegi ridotti. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) è un Cross-Site Request Forgery: l'aggressore non necessita di un account, ma un utente deve collaborare (UI:R). Tutte le vulnerabilità, eccetto CVE-2026-118207 e CVE-2026-118149, sono classificate come «Exploitation Less Likely».

## Exchange 2016 e 2019: nessun aggiornamento, nessuna verifica ulteriore

L'annuncio menziona Exchange 2016 e 2019 solo in una nota: queste versioni non vengono più testate per le vulnerabilità e non verranno più pubblicati aggiornamenti. Microsoft scrive che devono essere considerate permanentemente vulnerabili («permanently vulnerable») e che l'organizzazione deve essere aggiornata immediatamente. Il fatto che le nove CVE di novembre indichino solo Exchange SE non dice quindi nulla sul fatto che le versioni precedenti siano interessate. Frankys Web sottolinea che Exchange SE e 2019 CU15 sono tecnicamente molto vicini e che almeno una parte delle vulnerabilità è probabilmente presente anche in Exchange 2019.

Per l'esercizio, questo significa:

1. **Exchange 2019 CU15 e precedenti, Exchange 2016**: la versione rimane congelata all'SU di ottobre (con ESU) o a una versione precedente. Le nove vulnerabilità di novembre rimangono aperte su questi server, se presenti, e Microsoft non le verifica più.

2. **Migrazione**: Microsoft raccomanda il passaggio immediato a Exchange SE.

3. **Ambienti ibridi**: Exchange Online limita e blocca il flusso di posta dai server On-Premises obsoleti; le regole sono riportate nell'[articolo sul Transport Enforcement](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problemi risolti

**Interruzioni di Exchange dopo gli aggiornamenti di Windows Server di ottobre** (tutte le versioni). Il problema con `MapiExceptionMdbOffline` (ec=1142) non viene risolto dall'SU di Exchange: secondo l'annuncio, occorre installare gli aggiornamenti di Windows Server di novembre 2026. Le note di rilascio di questi aggiornamenti riportano la correzione in «[Exchange Server]» e indicano il problema come risolto con gli aggiornamenti a partire dal 10 novembre 2026: KB5131080 per Windows Server 2025, KB5131090 per Windows Server 2022, KB5131085 per Windows Server 2019 e KB5131291 per Windows Server 2016. Secondo l'annuncio, l'ordine in cui installate l'SU di Exchange e l'aggiornamento di Windows non è rilevante. Se avete impostato il valore del Registro `EpmapSavedContextCap` dall'[articolo sulle interruzioni di Exchange](/blog/exchange-windows-updates-oktober-2026), potete lasciarlo; secondo Microsoft, dopo l'installazione degli aggiornamenti di novembre per Windows Server non è più necessario.

**Visualizzazione errata della disponibilità libero/occupato tramite Microsoft Graph negli ambienti ibridi** (solo Exchange SE). Secondo [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867), gli utenti con una cassetta postale On-Premises non vedevano informazioni corrette su libero/occupato per le cassette postali di Exchange Online nel classico Outlook quando la Graph API era attivata; venivano visualizzati solo blocchi di tempo. Con Exchange Web Services (EWS) al posto della Graph API, la visualizzazione era corretta. Il problema è risolto con KB5133453 o un aggiornamento successivo. L'articolo del supporto non menziona alcuna attività successiva. Se siete tornati a EWS a causa di questo errore, dopo l'installazione verificate la richiesta libero/occupato di una cassetta postale di Exchange Online prima di riattivare la Graph API.

## Problemi noti

Né l'annuncio né KB5133453 indicano Known Issues per l'SU di novembre (stato all'11 novembre 2026). L'annuncio non contiene commenti del team Exchange e Microsoft segnala che la documentazione potrebbe essere ancora incompleta al momento della pubblicazione. Dai mesi precedenti non rimane aperto alcun Known Issue: i problemi degli aggiornamenti di settembre sono risolti dall'SU di ottobre e il problema di Windows di ottobre è risolto con gli aggiornamenti di novembre per Windows Server (vedere sopra).

## Installazione e attività successive

Nell'annuncio Microsoft indica la procedura nota: effettuare l'inventario con l'[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) per stabilire quali server richiedono aggiornamenti o passaggi manuali, installare l'SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. Se dopo il setup alcuni servizi sono disattivati, l'installazione è stata interrotta. In caso di errori durante o dopo l'installazione, Microsoft rimanda allo [script SetupAssist](https://aka.ms/ExSetupAssist). Dopo l'installazione, secondo KB5133453, l'Health Checker indica se l'SU è stato installato correttamente e se sono necessari ulteriori passaggi.

Secondo KB5133453, il file di installazione si chiama `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: «SU12»). L'SU sostituisce l'SU di ottobre KB5128895. Gli SU sono cumulativi: chi ha installato un SU precedente su Exchange SE RTM può installare direttamente l'SU di novembre.

Per gli ambienti ibridi, secondo le FAQ, vale quanto segue: se dopo l'installazione di un SU viene cambiato il certificato di autenticazione, occorre eseguire nuovamente Hybrid Configuration Wizard. In Windows Server 2025 gli SU di Exchange installati non vengono visualizzati nel Pannello di controllo; Microsoft sconsiglia di disinstallare gli SU.

Dai mesi precedenti restano aperte queste attività successive, se non sono già state completate: rimuovere l'override libero/occupato `EnableRouteThroughMSGraphFeature` (vedere [articolo di ottobre](/blog/exchange-security-updates-oktober-2026)), rimuovere il SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders` (vedere [articolo di settembre](/blog/exchange-security-updates-september-2026)) e verificare se la mitigazione CVE-2026-42897 (M2.1.0) è ancora attiva (vedere [articolo sull'SU di luglio](/blog/exchange-security-updates-juli-2026)).

## Procedura consigliata

Installate tempestivamente l'SU di novembre su tutti i server con Exchange SE e sulle macchine con gli Management Tools: per la vulnerabilità critica di esecuzione di codice in modalità remota CVE-2026-118207 è sufficiente un account con privilegi ridotti e Microsoft ritiene più probabile il suo sfruttamento. Nella stessa finestra di manutenzione installate gli aggiornamenti di novembre per Windows Server, in modo da risolvere la causa delle interruzioni di ottobre. Per Exchange 2016 e 2019 non esistono più patch; secondo Microsoft questi server devono essere considerati permanentemente vulnerabili e dovrebbero essere sostituiti immediatamente con Exchange SE.

## Fonti

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): annuncio del 10 novembre 2026 con versione supportata, nota su Exchange 2016/2019, problema Windows risolto e FAQ (nessun commento del team Exchange, stato all'11 novembre 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): elenco delle CVE, problema risolto, file di installazione, aggiornamento sostituito e nota sull'Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): tipo, CVSS, gravità, valutazione dello sfruttamento, FAQ e prodotto interessato delle nove CVE di novembre.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): numero di build dell'SU di novembre.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): sintomi e correzione con KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): contributo del team Exchange sul problema di Windows di ottobre, a cui rimanda l'annuncio.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): note di rilascio per Windows Server 2025 con la correzione di Exchange e il Known Issue contrassegnato come risolto.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): note di rilascio per Windows Server 2022 con la stessa correzione.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): note di rilascio per Windows Server 2019 con la stessa correzione.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): note di rilascio per Windows Server 2016 con la stessa correzione.

11. [Nuovi aggiornamenti di sicurezza per Exchange Server (novembre 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): analisi in lingua tedesca delle nove CVE e valutazione per Exchange 2016 e 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): inventario delle versioni CU/SU prima e dopo l'installazione.
