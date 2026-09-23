---
title: "Exchange Online rallenta e blocca i server Exchange 2016 e 2019 obsoleti da settembre 2026: come funziona il Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Dalla seconda settimana di settembre 2026, Exchange Online richiede ai server ibridi almeno la SU di ottobre 2025, altrimenti il flusso di posta viene rallentato e in seguito bloccato. Contesto sul Transport Enforcement in vigore dal 2023, livelli di escalation con codici SMTP, report nell’Admin Center, sospensione di 90 giorni tramite PowerShell e perché il prossimo innalzamento della soglia consentirà solo i clienti ESU e Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min di lettura"
themen:
  - exchange-onprem-hybrid
  - exchange-updates
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-limita-e-blocca-exchange-2016-e-2019-obsoleti-da-settembre-2026-come-funziona"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
translationSourceHash: b2c2e61a4e97d49b1046134bf215a91e0d46ed817201023d8364945f3549d539
translationModel: gpt-5.6-terra
translatedAt: 2026-09-08T08:13:24.964Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/exchange-online-limita-e-blocca-exchange-2016-e-2019-obsoleti-da-settembre-2026-come-funziona
---

# Exchange Online rallenta e blocca i server Exchange 2016 e 2019 obsoleti da settembre 2026: come funziona il Transport Enforcement

Il team di Exchange ha annunciato il 2 settembre 2026 l’innalzamento della versione minima per Exchange 2016 ed Exchange 2019 nel flusso di posta ibrido. Dalla seconda settimana di settembre 2026, Exchange Online richiede ai server che inviano messaggi tramite un connettore in ingresso di tipo `OnPremises` almeno il livello dell’ultimo aggiornamento di sicurezza pubblico di ottobre 2025. Tutto ciò che è inferiore viene rallentato e successivamente bloccato. In breve: chi non ha applicato patch ai propri server ibridi da ottobre 2025 perderà gradualmente, nelle prossime settimane, la consegna della posta verso Exchange Online. E il prossimo innalzamento della soglia, che Microsoft prospetta per i prossimi mesi, sarà superiore a qualsiasi aggiornamento disponibile pubblicamente: a quel punto soddisferanno il requisito soltanto i clienti del programma ESU a pagamento o gli ambienti con Exchange Server Subscription Edition (SE).

L’annuncio in sé è breve. Il suo significato pratico deriva dal sistema di enforcement che Microsoft ha costruito gradualmente dal 2023: quali risposte SMTP vedrà il vostro server, come verificare lo stato nell’Admin Center e tramite PowerShell e quali opzioni restano durante la transizione fino alla fine del programma ESU nell’ottobre 2026.

## Cosa si applica dalla seconda settimana di settembre 2026

La nuova soglia minima corrisponde agli aggiornamenti di sicurezza del 14 ottobre 2025. È stato l’ultimo Patch Tuesday in cui Microsoft ha reso pubblicamente disponibili aggiornamenti per Exchange 2016 ed Exchange 2019; tutte le SU da dicembre 2025 sono disponibili solo tramite il programma ESU.

| Versione | Livello minimo | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU di ottobre 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU di ottobre 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Per Exchange 2019 CU14 esiste anch’essa una SU di ottobre 2025 (KB5066368, build 15.2.1544.36). Tuttavia, nel contributo Microsoft è indicata esplicitamente CU15 SU5 come versione minima; CU14 non è più uno stato consigliato dalla pubblicazione di CU15 nel febbraio 2025. Pianificate quindi anche il passaggio a CU15 per CU14.

Sono importanti tre delimitazioni:

- **È interessato solo il flusso di posta ibrido.** Exchange Online verifica la versione dei server mittenti per i messaggi che arrivano tramite un connettore in ingresso di tipo `OnPremises`. Si tratta della classica configurazione ibrida creata da Hybrid Configuration Wizard. I messaggi che arrivano tramite un gateway di terze parti o un connettore di tipo `Partner` non passano attraverso questo enforcement.
- **La versione viene letta dalle intestazioni.** Un server Exchange scrive la propria build nella riga `Received` di ogni messaggio che inoltra (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online valuta questa indicazione. Conta quindi il livello del server che consegna effettivamente il messaggio a Exchange Online, ossia in molti ambienti l’Edge Transport Server o il server Mailbox con il Send Connector verso `*.mail.protection.outlook.com`.
- **Exchange SE non è interessato.** L’enforcement si applica a Exchange 2016 ed Exchange 2019; Exchange Server SE supera ogni soglia minima finché riceve regolarmente le patch.

## Contesto: il Transport Enforcement dal 2023

L’annuncio di settembre non è una nuova misura, bensì il livello successivo di un sistema che Microsoft ha presentato nel marzo 2023 con il titolo «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definisce «persistently vulnerable» qualsiasi server Exchange che abbia raggiunto la fine del supporto o rimanga privo di patch per vulnerabilità note. L’obiettivo è proteggere i destinatari di Exchange Online dai messaggi provenienti da server potenzialmente compromettibili e, al contempo, esercitare pressione sugli operatori affinché applichino patch o spengano i server.

Il sistema è stato attivato progressivamente per versione:

| Momento | Versione interessata |
|---|---|
| Agosto 2023 | Exchange 2007 |
| Settembre 2023 | Exchange 2010 |
| Dicembre 2023 | Exchange 2013 |
| Marzo 2024 | Exchange 2016 ed Exchange 2019 (livelli SU notevolmente obsoleti) |
| Settembre 2026 | Exchange 2016 ed Exchange 2019: soglia minima = SU di ottobre 2025 |
| «tra alcuni mesi» | Exchange 2016 ed Exchange 2019: soglia minima superiore all’ultimo aggiornamento pubblico |

Per Exchange 2016 ed Exchange 2019, finora la soglia minima riguardava livelli «significantly behind on security updates». La novità è che Microsoft porta il limite all’ultimo aggiornamento pubblico, interessando così per la prima volta server che meno di un anno fa erano ancora completamente aggiornati.

## I livelli di escalation

L’enforcement opera in tre funzioni, che Microsoft chiama «reporting», «throttling» e «blocking». Non appena un server scende sotto la soglia minima, inizia un ciclo di 90 giorni. I livelli descritti nell’articolo introduttivo del 2023:

| Periodo | Misura | Risposta SMTP |
|---|---|---|
| Giorno 0-30 | Solo report nell’Exchange Admin Center | nessuna |
| Giorno 30-40 | Rallentamento per 5 minuti all’ora | `450 4.7.230` |
| Giorno 40-50 | Rallentamento per 10 minuti all’ora | `450 4.7.230` |
| Giorno 50-60 | Rallentamento per 20 minuti all’ora | `450 4.7.230` |
| Giorno 60-70 | Rallentamento per 30 minuti all’ora, più blocco per 5 minuti all’ora | `450 4.7.230` e `550 5.7.230` |
| Giorno 70-80 | Blocco per 10 minuti all’ora | `550 5.7.230` |
| Giorno 80-90 | Blocco per 20 minuti all’ora | `550 5.7.230` |
| Dal giorno 90 | Blocco completo | `550 5.7.230` |

Le due risposte sono, testualmente:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La differenza è determinante per l’operatività. In caso di `450`, Exchange Online rifiuta temporaneamente la connessione; il server on-premises mantiene il messaggio nella propria coda e riprova. Gli utenti inizialmente notano solo ritardi; in Queue Viewer o in `Get-Queue` cresce la coda verso il Send Connector per Exchange Online con stato `Retry` e il messaggio 4.7.230 come `LastError`. Con `550`, invece, il rifiuto è definitivo: il mittente riceve un NDR con il codice 5.7.230 e il messaggio è perso, salvo che venga inviato nuovamente. Poiché all’inizio il blocco è attivo solo per alcuni minuti all’ora, il comportamento appare dapprima sporadico: una parte dei messaggi arriva, un’altra fallisce con NDR. Chi osserva un simile schema nel Message Tracking dovrebbe verificare innanzitutto la versione, prima di cercare problemi di rete o certificati.

L’annuncio non specifica se Microsoft avvierà il ciclo completo di 90 giorni dalla seconda settimana di settembre per la nuova soglia minima o se inizierà già da un livello successivo. L’articolo introduttivo precisa che il sistema, dopo una pausa, continua dal livello precedentemente raggiunto. Non fate quindi affidamento su 30 giorni di tolleranza.

## Report nell’Exchange Admin Center e tramite PowerShell

Exchange Online elenca i server on-premises rilevati, con relativa versione, in un report dedicato: nell’Exchange Admin Center, in *Reports*, *Mail flow*, report sui server Exchange on-premises connessi obsoleti («out-of-date connecting on-premises Exchange servers»). Per ciascun server, il report mostra la build rilevata, se è sotto la soglia minima e a quale livello di enforcement si trova.

Le stesse informazioni sono fornite da Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Comando | Effetto |
|---|---|
| `Connect-ExchangeOnline` | Apre la sessione con Exchange Online (modulo `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Elenca i server on-premises rilevati da Exchange Online con build, stato e livello di enforcement. |

</details>

Il report conosce solo i server che consegnano effettivamente messaggi a Exchange Online. Un server di gestione senza flusso di posta o una macchina con i soli Management Tools non vi appare. Questo è irrilevante per l’enforcement, ma non per la sicurezza: anche questi sistemi necessitano delle SU.

## Sospendere l’enforcement: 90 giorni all’anno

Per gli ambienti che non riescono a raggiungere rapidamente la soglia minima, Microsoft offre una sospensione. Può essere attivata per un totale di 90 giorni all’anno, in un’unica soluzione o in più periodi:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-TenantExemptionInfo` | Mostra se e per quanto tempo è attiva una sospensione per il tenant. |
| `New-TenantExemptionInfo` | Crea una nuova sospensione. |
| `-BlockingScenario UnpatchedOnPremServer` | Seleziona lo scenario «server on-premises obsoleto»; al momento non esistono altri scenari per questo cmdlet. |
| `-NumberOfDays 30` | Durata della sospensione in giorni. Il contingente è di 90 giorni all’anno e il valore indicato viene detratto da esso. |

</details>

Due caratteristiche della sospensione sono importanti nella pratica. In primo luogo, alla sua scadenza l’enforcement prosegue dal livello in cui era stato fermato; la sospensione non reimposta il ciclo di 90 giorni. In secondo luogo, non esiste alcun cmdlet per terminare anticipatamente una sospensione in corso: chi imposta 90 giorni e completa le patch dopo due settimane ha esaurito il contingente annuale. Create quindi la sospensione per il periodo più breve possibile ed estendetela se necessario.

Inoltre, la sospensione è una soluzione solo per la soglia minima attuale. Se Microsoft innalzerà il limite tra alcuni mesi oltre l’ultimo aggiornamento pubblico, un contingente esaurito non sarà più d’aiuto.

## Perché il prossimo innalzamento è la vera scadenza

Exchange 2016 ed Exchange 2019 sono fuori supporto dal 14 ottobre 2025. Microsoft ha poi introdotto due periodi ESU a pagamento: il periodo 1 fino ad aprile 2026, il periodo 2 da maggio a ottobre 2026. Con l’annuncio del periodo 2, il 15 aprile 2026, il team di Exchange ha chiarito che non ci saranno ulteriori proroghe. Le SU da dicembre 2025 ad agosto 2026 (l’ultima è la build 15.2.1748.49 per 2019 CU15 e 15.1.2507.72 per 2016 CU23) sono disponibili esclusivamente per i clienti ESU e non vengono offerte pubblicamente per il download.

Ne deriva la seguente situazione:

- **Oggi** un server con la SU di ottobre 2025 soddisfa la soglia minima, con o senza ESU.
- **Al prossimo innalzamento** la soglia minima sarà, secondo Microsoft, superiore al livello di ottobre 2025. Senza un contratto ESU non esiste un modo legale per raggiungere quel livello. Il flusso di posta ibrido di questi server verrà quindi rallentato e bloccato, indipendentemente da quanto correttamente sia gestito il resto dell’ambiente.
- **Il 31 ottobre 2026** termina anche il periodo 2. Dopo tale data non ci saranno più SU per Exchange 2016 ed Exchange 2019, per nessuno. Al più tardi il successivo innalzamento dopo quello prossimo interesserà quindi anche i clienti ESU.

Il programma ESU acquista dunque, nel migliore dei casi, pochi mesi. L’unico livello permanente accettato dall’enforcement è Exchange Server SE. Microsoft ha inoltre annunciato che Exchange SE CU2, previsto per la seconda metà del 2026, interromperà la coesistenza con Exchange 2016 ed Exchange 2019: l’installazione si interromperà se nell’organizzazione vengono trovati server più vecchi. La migrazione è quindi necessaria non solo per il flusso di posta, ma anche per poter continuare a installare aggiornamenti per SE.

Per gli ambienti che mantengono Exchange on-premises solo per gestire gli attributi in una configurazione ibrida, l’alternativa è rimuovere l’ultimo server: da Exchange 2019 CU12, gli attributi dei destinatari possono essere gestiti con i Management Tools senza un server Exchange in esecuzione. In questo caso non esiste più flusso di posta on-premises e l’enforcement non è più pertinente.

## Determinare il livello di versione

`Get-ExchangeServer` mostra in `AdminDisplayVersion` solo la CU, non la SU. È affidabile la versione del file `ExSetup.exe` oppure [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), che segnala anche i passaggi manuali mancanti. Per una rapida panoramica di tutti i server:

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Elemento | Effetto |
|---|---|
| `Get-ExchangeServer` | Elenca tutti i server Exchange dell’organizzazione. |
| `\\<Server>\C$\...\ExSetup.exe` | Percorso di condivisione amministrativa al file di installazione; adattarlo in caso di percorso di installazione diverso. |
| `VersionInfo.ProductVersion` | Versione del file, che corrisponde alla build SU installata (ad es. `15.1.2507.61`). |

</details>

Se la versione è inferiore a `15.2.1748.39` (2019 CU15) o a `15.1.2507.61` (2016 CU23), il server scenderà sotto la soglia minima dalla seconda settimana di settembre.

## Procedura consigliata

1. **Inventariare il livello** come descritto sopra, inclusi Edge Transport Server e server di gestione.

2. **Verificare il report in Exchange Online.** `Get-OnPremServerReportInfo` mostra quali server Exchange Online vede effettivamente e se è già attivo un livello di enforcement. Confrontate l’elenco con l’inventario: i server che non vi compaiono non inviano tramite il connettore `OnPremises`.

3. **Installare almeno la SU di ottobre 2025.** KB5066367 (2019 CU15) e KB5066369 (2016 CU23) sono ancora disponibili pubblicamente nel Microsoft Download Center. Le SU sono cumulative; un server al livello di agosto 2025 può essere aggiornato direttamente a ottobre 2025. Per CU14, installate prima CU15. Dopo l’installazione, riavviate, controllate lo stato dei servizi ed eseguite nuovamente Health Checker.

4. **Usare la sospensione solo come soluzione ponte.** Se l’aggiornamento non riesce nella prima metà di settembre, create `New-TenantExemptionInfo` con una durata breve e non considerate la sospensione come riserva di pianificazione per il prossimo innalzamento.

5. **Pianificare la migrazione a Exchange SE.** Senza contratto ESU, il prossimo innalzamento è la scadenza inderogabile; con ESU, lo è il 31 ottobre 2026. Exchange 2019 CU15 può essere portato a SE tramite aggiornamento in-place; Exchange 2016 richiede il passaggio attraverso una nuova installazione di SE e lo spostamento delle cassette postali o dei ruoli. Chi usa Exchange solo per la gestione degli attributi rimuove l’ultimo server e continua a lavorare con i Management Tools.

## Fonti

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): L’annuncio del 2 settembre 2026 con la nuova soglia minima (SU di ottobre 2025), la data di avvio nella seconda settimana di settembre e l’indicazione del prossimo innalzamento oltre l’ultimo aggiornamento pubblico.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): L’articolo introduttivo del 2023 con la definizione di «persistently vulnerable», i livelli Reporting, Throttling e Blocking, il ciclo di 90 giorni, le risposte SMTP 4.7.230 e 5.7.230 e il piano di rilascio progressivo per versione.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): I cmdlet `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` e `New-TenantExemptionInfo`, incluso il report nell’Exchange Admin Center e il contingente annuale di 90 giorni.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numeri di build delle SU di ottobre 2025 e dei successivi aggiornamenti ESU fino ad agosto 2026; include anche l’indicazione che le SU da dicembre 2025 sono disponibili solo per i clienti ESU.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): L’articolo KB relativo al livello minimo per Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): L’articolo KB relativo al livello minimo per Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): La fine del supporto il 14 ottobre 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Condizioni del primo periodo ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Durata da maggio a ottobre 2026 e dichiarazione che non seguiranno ulteriori proroghe.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Suddivisione tabellare degli otto livelli di enforcement e delle date di rollout per ogni versione di Exchange; fonte di terze parti.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario dei livelli CU/SU e dei passaggi manuali ancora aperti.
