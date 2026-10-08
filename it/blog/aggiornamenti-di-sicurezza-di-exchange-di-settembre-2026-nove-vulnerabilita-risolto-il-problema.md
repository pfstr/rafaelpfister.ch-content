---
title: "Aggiornamenti di sicurezza di Exchange di settembre 2026: nove vulnerabilità, risolto il problema dei wrapper, rilasciata la v2"
navTitle: "Exchange SU 09/2026"
description: "L'aggiornamento di sicurezza di settembre chiude nove vulnerabilità in Exchange SE e 2019 (otto in Exchange 2016), tra cui una falla di spoofing con CVSS 9.3, e risolve il problema dei wrapper negli ambienti ibridi. Il 2 ottobre è seguita una v2 con un'ulteriore CVE; si aggiungono tre problemi noti con workaround e un SettingOverride che ora deve essere rimosso."
date: "2026-10-07"
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
slug: "aggiornamenti-di-sicurezza-di-exchange-di-settembre-2026-nove-vulnerabilita-risolto-il-problema"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/it/blog/aggiornamenti-di-sicurezza-di-exchange-di-settembre-2026-nove-vulnerabilita-risolto-il-problema
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:45:09.884Z
translationReview: automatic
---

# Aggiornamenti di sicurezza di Exchange di settembre 2026: nove vulnerabilità, risolto il problema dei wrapper, rilasciata la v2

L'8 settembre 2026 Microsoft ha pubblicato aggiornamenti di sicurezza (SU) per Exchange Server. Chiudono nove vulnerabilità in Exchange SE e Exchange 2019, otto in Exchange 2016. Nessuna era nota pubblicamente in anticipo, nessuna risulta sfruttata attivamente secondo la Security Update Guide e Microsoft le classifica tutte come *Important* con «Exploitation Less Likely». Il valore CVSS più alto, pari a 9.3, è tuttavia nettamente superiore a quello del mese precedente. Questo mese è particolare per tre motivi: il SU risolve il problema, aperto da giugno, dei *messaggi wrapper* nelle cassette postali condivise, introduce tre nuovi o persistenti problemi noti e il 2 ottobre Microsoft ha rilasciato una **Version 2** che chiude un'ulteriore vulnerabilità.

## Per quali versioni di Exchange è disponibile l'aggiornamento

I SU dell'8 settembre 2026 sono disponibili per le seguenti versioni:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, build 15.2.2562.49; disponibile pubblicamente.
- **Exchange Server 2019 CU15**: KB5121609, build 15.2.1748.51; solo tramite il **programma ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5121610, build 15.2.1544.46; solo tramite ESU Period 2.
- **Exchange Server 2016 CU23**: KB5121611, build 15.1.2507.73; solo tramite ESU Period 2.

Exchange 2016 e 2019 sono fuori supporto. Secondo Microsoft, i SU da maggio a ottobre 2026 vengono ricevuti solo dalle organizzazioni iscritte al programma ESU Period 2. Secondo gli articoli KB, questa idoneità è valida fino a ottobre 2026. Si aggiunge la pressione di Exchange Online: dalla seconda settimana di settembre Exchange Online limita e blocca il flusso di posta ibrido dai server con una versione precedente a ottobre 2025; i dettagli sono disponibili nell'[articolo sull'applicazione del transport](/blog/exchange-online-transport-enforcement-hybrid-server). Exchange Online stesso è già protetto secondo l'annuncio; negli ambienti ibridi, tuttavia, ogni server Exchange necessita comunque del SU, così come le macchine con gli Exchange Management Tools.

È possibile confrontare la propria versione con la panoramica dei [numeri di build di Exchange](/tools/exchange-builds).

## Panoramica delle vulnerabilità

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Spoofing (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Spoofing (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 non riguarda Exchange 2016; la Security Update Guide indica per questa CVE solo Exchange SE e 2019 CU14/CU15. Un dettaglio sulla documentazione: negli articoli KB dei quattro SU di settembre manca CVE-2026-69380 nell'elenco delle CVE. La Security Update Guide indica tuttavia esattamente le quattro build di settembre come correzione per questa CVE (al 7 ottobre 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** ha il valore più alto, CVSS 9.3. Secondo Microsoft, un aggressore non autenticato può inviare un invito al calendario predisposto con un collegamento a una riunione dannoso; se il destinatario apre la riunione e seleziona il collegamento per partecipare, viene attivato il Cross-Site Scripting. È quindi necessaria un'interazione dell'utente, ma non un account nell'organizzazione.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) richiede solo un account con pochi privilegi e una cassetta postale assegnata. Secondo le FAQ nella Security Update Guide, un aggressore può sfruttare debolezze nella verifica delle richieste e dei token di identità per impersonare un altro utente e prendere il controllo delle cassette postali di tutti gli utenti Exchange: leggere e inviare e-mail e scaricare allegati. Come punto di partenza è sufficiente un singolo account utente compromesso.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) porta allo stesso risultato, la compromissione di tutte le cassette postali, ma richiede l'appartenenza a un gruppo di ruoli con privilegi elevati.

Le condizioni iniziali delle due falle di esecuzione di codice remoto sono diverse: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) richiede un account autenticato con privilegi limitati, [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) può essere attivata senza autenticazione tramite un allegato Visio predisposto, ma secondo Microsoft richiede una persistente scarsità di memoria sul sistema di destinazione. Le altre quattro falle: CVE-2026-69378 (DoS tramite ricorsione non controllata, senza autenticazione), CVE-2026-69361 (SSRF, il server invia richieste HTTP a sistemi interni o di loopback), CVE-2026-69375 (un aggressore autenticato può sostituire il contenuto dei file) e CVE-2026-69382 (divulgazione di credenziali tramite un algoritmo crittografico debole, richiede un cookie di autenticazione già sottratto).

## Versione 2 del 2 ottobre: aggiunta CVE-2026-96940

Il 2 ottobre 2026 Microsoft ha pubblicato la «Version 2» dei SU di settembre. Secondo l'annuncio, l'unica differenza rispetto alla prima versione è la correzione aggiuntiva per **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Questa falla di Elevation of Privilege ha CVSS 8.8, non è né nota pubblicamente né sfruttata, ma Microsoft la valuta come l'unica delle dieci CVE con **«Exploitation More Likely»**. Un aggressore autenticato può così accedere alle cassette postali di altri utenti della stessa organizzazione e leggere e-mail e allegati. Exchange Online è già stato corretto lato server.

| Versione | KB | Build v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

In pratica significa che la correzione per CVE-2026-96940 è presente solo nelle build v2. I server sui quali è già in esecuzione il SU dell'8 settembre necessitano inoltre della v2. Chi esegue la patch solo ora può installare direttamente la v2, poiché i SU sono cumulativi. Gli articoli KB non indicano espressamente se i server con il primo SU di settembre debbano obbligatoriamente installare la v2; poiché la nuova CVE viene chiusa solo con la v2, questa è l'interpretazione più ovvia.

## Problema dei wrapper risolto: rimuovere ora il SettingOverride

Il problema noto dal SU di giugno, per cui negli ambienti ibridi compaiono *messaggi wrapper* nella Posta in arrivo delle cassette postali condivise, è risolto con il SU di settembre in tutte e quattro le versioni. Nell'[articolo di agosto](/blog/exchange-security-updates-august-2026) era ancora indicato che il SettingOverride documentato come workaround poteva rimanere. Dopo l'installazione del SU di settembre vale il contrario: Microsoft raccomanda nell'articolo di supporto associato di verificare e rimuovere l'override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Comando | Effetto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Verifica se l'override del workaround è impostato nell'organizzazione. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Rimuove l'override non appena il SU di settembre è installato. |

</details>

Se il primo comando segnala che l'oggetto `DisableBlockSharedAndUserMailboxHeaders` non è stato trovato, secondo Microsoft non occorre fare altro.

Per Exchange SE, il SU risolve inoltre un errore nelle richieste di disponibilità ibrida tramite Microsoft Graph: gli utenti on-premises vedevano gli orari occupati delle cassette postali di Exchange Online spostati del proprio scarto UTC, senza messaggi di errore.

## Problemi noti

**I calendari pubblicati (.ics) restituiscono HTTP 500 alle app calendario.** Il problema persiste dal SU di agosto (Exchange SE dalla build 15.2.2562.46, nonché Exchange 2019 e 2016) e non è risolto né nel SU di settembre né nella v2. Le sottoscrizioni a calendari pubblicati in forma anonima non si aggiornano più; la stessa URL funziona nel browser. Secondo Microsoft, la causa è che Exchange riconosce i client tramite lo User-Agent: le app calendario, senza identificazione del browser, finiscono in un percorso di codice disattivato dal SU di agosto. Come workaround Microsoft descrive una regola di riscrittura URL nel sito «Exchange Back End» in IIS che, per richieste .ics sotto `/owa/calendar/`, aggiunge il parametro `layout=premium`; per questo deve essere installato il modulo IIS URL Rewrite. I passaggi esatti, tramite IIS Manager o direttamente nel file `applicationHost.config`, sono riportati nell'[articolo di supporto KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft non indica una data per la correzione (al 7 ottobre 2026).

**Disponibilità per cassette postali delegate in ambienti ibridi (solo Exchange SE).** Se la query di disponibilità è configurata esclusivamente tramite Graph API, le richieste per le cassette postali di Exchange Online tramite accesso on-premises delegato non riescono. Outlook segnala «Your server location could not be determined», OWA mostra «No information» e nei log EWS compare `(403) Forbidden`. Il workaround documentato instrada nuovamente le richieste tramite EWS anziché Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | L'override che controlla l'instradamento delle richieste di disponibilità tramite Microsoft Graph. |
| `-Parameters "Enabled=False"` | Disattiva il percorso Graph; le richieste passano nuovamente tramite EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Indirizza la chiamata diagnostica al servizio di topologia. |
| `-Component VariantConfiguration -Argument Refresh` | Ricarica la Variant Configuration affinché l'override abbia effetto senza attesa. |

</details>

Nell'annuncio della v2 Microsoft elenca questo problema tra quelli risolti. Chi ha impostato il workaround dovrebbe verificare, dopo l'installazione della v2, nell'[articolo di supporto KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) se debba essere annullato; al 7 ottobre 2026 non vi erano ancora istruzioni in merito.

**Deadlock di ContentEngine dovuto a file WordBreaker coreani mancanti (solo Exchange SE).** Con il SU di settembre (build 15.2.2562.49 e build v2 15.2.2562.53), i file di regole per il WordBreaker coreano aggiornato non vengono installati. Le conseguenze sono risultati di ricerca mancanti, recapito delle e-mail ritardato, client Outlook o MAPI che si bloccano o perdono la connessione. Secondo l'annuncio, il problema riguarda i messaggi in lingua coreana. Il workaround nell'[articolo di supporto KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) consiste nell'estrarre i due file `ko.token.rule.bin` e `ko.complex.rule.bin` da SQL Server 2025 Express RTM, verificarne gli hash SHA256, copiarli nella directory `Native` dell'installazione di Exchange e riavviare il servizio Search Host Controller. Microsoft sta ancora indagando sul problema.

## Installazione e attività successive

Microsoft raccomanda la procedura nota: inventariare con [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinare il percorso con [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) se la versione è obsoleta, installare il SU, riavviare il server e verificare che tutti i servizi Exchange siano in esecuzione. In seguito, Health Checker mostra anche se il SU è stato installato correttamente. La Security Update Guide indica che per gli aggiornamenti è necessario un riavvio.

Dopo l'installazione sono previste tre attività:

1. Rimuovere il SettingOverride dei wrapper `DisableBlockSharedAndUserMailboxHeaders` se è impostato (vedere sopra).

2. In Exchange SE, verificare se si verificano i problemi di disponibilità e WordBreaker e, se necessario, applicare i workaround.

3. Per i calendari pubblicati con sottoscrittori esterni, configurare la regola di riscrittura URL da KB5126672, se ciò non è già stato fatto dal SU di agosto.

Da luglio resta inoltre da verificare che la mitigazione CVE-2026-42897 (M2.1.0) sia ancora attiva; come rimuoverla è indicato nell'[articolo sul SU di luglio](/blog/exchange-security-updates-juli-2026).

## Procedura consigliata

Installare direttamente le build v2 del 2 ottobre su tutti i server Exchange e sulle macchine con Management Tools; i server con il SU dell'8 settembre necessitano inoltre della v2 per CVE-2026-96940. La falla di spoofing con CVSS 9.3 e la compromissione delle cassette postali tramite un account utente con privilegi limitati (CVE-2026-69380) sono motivi sufficienti per non attendere il prossimo Patch Tuesday. Rimuovere quindi l'override dei wrapper, verificare i tre problemi noti ed eseguire Health Checker. Per Exchange 2016 e 2019 il programma ESU termina nell'ottobre 2026; la migrazione a Exchange SE non può più essere rimandata.

## Fonti

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Annuncio ufficiale del rilascio con versioni supportate, indicazione ESU, problemi noti, problemi risolti e procedura di installazione (consultato tramite il feed RSS dell'Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Annuncio della v2 del 2 ottobre 2026; l'unica differenza è CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Elenco delle CVE, problemi risolti e i tre problemi noti per Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Articolo KB per Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Articolo KB per Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Articolo KB per Exchange 2016 CU23, senza CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravità, valutazione dello sfruttamento e FAQ su tutte le nove CVE di settembre e su CVE-2026-96940, comprese le build interessate per ciascuna CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Articolo KB sulla v2 per Exchange SE; gli articoli v2 per 2019 e 2016 sono KB5129956 fino a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numeri di build dei SU di settembre e della v2 del 2 ottobre 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Correzione nel SU di settembre e istruzioni per rimuovere il SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa e workaround di riscrittura URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Sintomi e workaround con SettingOverride per Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Build SE interessate e workaround manuale.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Errore di fuso orario in Exchange SE risolto con il SU di settembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Analisi in lingua tedesca delle nove CVE con valori CVSS e build.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Riepilogo in lingua tedesca con indicazioni sul successivo aggiornamento sostitutivo.
