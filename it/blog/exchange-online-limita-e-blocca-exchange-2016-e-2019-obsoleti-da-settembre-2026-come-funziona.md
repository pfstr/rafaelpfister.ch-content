---
title: "Exchange Online limita e blocca i server Exchange 2016 e 2019 obsoleti da settembre 2026: come funziona il Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "Dalla seconda settimana di settembre 2026, Exchange Online richiede almeno la SU di ottobre 2025 dai server ibridi; in caso contrario, il flusso di posta viene limitato e successivamente bloccato. Contesto sul Transport Enforcement in vigore dal 2023, livelli di escalation con codici SMTP, report nell’Admin Center, pausa di 90 giorni tramite PowerShell e perché il prossimo innalzamento consentirà il passaggio solo ai clienti ESU e a Exchange SE."
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
translationSourceHash: d2bb7589193052361cd6a083ace0d3dbc58552e5e66691c352cd82b693fae6bd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:49:50.159Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/exchange-online-limita-e-blocca-exchange-2016-e-2019-obsoleti-da-settembre-2026-come-funziona
---

# Exchange Online limita e blocca i server Exchange 2016 e 2019 obsoleti da settembre 2026: come funziona il Transport Enforcement

Il team di Exchange ha annunciato il 2 settembre 2026 l’innalzamento della versione minima per Exchange 2016 e Exchange 2019 nel flusso di posta ibrido. Dalla seconda settimana di settembre 2026, Exchange Online richiederà ai server che inviano messaggi tramite un Inbound Connector di tipo `OnPremises` almeno il livello dell’ultimo aggiornamento di sicurezza pubblico di ottobre 2025. Tutto ciò che è inferiore verrà limitato e successivamente bloccato. In breve: chi non ha applicato patch ai propri server ibridi da ottobre 2025 perderà gradualmente la consegna della posta a Exchange Online nelle prossime settimane. E il prossimo innalzamento, che Microsoft prospetta per i mesi a venire, sarà superiore a ogni aggiornamento disponibile pubblicamente: a quel punto soddisferanno il requisito solo i clienti del programma ESU a pagamento o gli ambienti con Exchange Server Subscription Edition (SE).

L’annuncio in sé è breve. Ciò che significa nella pratica emerge dal sistema di enforcement che Microsoft ha sviluppato gradualmente dal 2023: quali risposte SMTP vedrà il vostro server, come verificare lo stato nell’Admin Center e tramite PowerShell e quali opzioni restano nel periodo di transizione fino alla fine del programma ESU nell’ottobre 2026.

## Cosa si applica dalla seconda settimana di settembre 2026

Il nuovo limite minimo corrisponde agli aggiornamenti di sicurezza del 14 ottobre 2025. È stato l’ultimo Patch Tuesday in cui Microsoft ha reso pubblici aggiornamenti per Exchange 2016 e 2019; tutte le SU da dicembre 2025 sono disponibili solo tramite il programma ESU.

| Versione | Livello minimo | KB | Build |
|---|---|---|---|
| Exchange 2019 CU15 | SU di ottobre 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU di ottobre 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

Per Exchange 2019 CU14 esiste anch’esso un SU di ottobre 2025 (KB5066368, build 15.2.1544.36). Tuttavia, nel contributo Microsoft viene indicato esplicitamente CU15 SU5 come versione minima; CU14 non è più uno stato consigliato dalla pubblicazione di CU15 nel febbraio 2025. Pianificate quindi anche il passaggio a CU15 se utilizzate CU14.

Tre delimitazioni sono importanti:

- **È interessato solo il flusso di posta ibrido.** Exchange Online verifica la versione dei server di invio per i messaggi che arrivano tramite un Inbound Connector di tipo `OnPremises`. Si tratta della classica configurazione ibrida creata da Hybrid Configuration Wizard. Le email che arrivano tramite un gateway di terze parti o un connettore di tipo `Partner` non passano attraverso questo enforcement.
- **La versione viene letta dalle intestazioni.** Un server Exchange scrive la propria build nella riga `Received` di ogni messaggio che inoltra (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online valuta questa indicazione. Conta quindi il livello del server che consegna effettivamente il messaggio a Exchange Online, ossia in molti ambienti l’Edge Transport Server o il Mailbox Server con il Send Connector verso `*.mail.protection.outlook.com`.
- **Exchange SE non è interessato.** L’enforcement si applica a Exchange 2016 e 2019; Exchange Server SE è al di sopra di ogni limite minimo, purché venga aggiornato regolarmente.

## Contesto: il Transport Enforcement dal 2023

L’annuncio di settembre non è una nuova misura, ma il livello successivo di un sistema che Microsoft ha presentato nel marzo 2023 con il titolo «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft definisce «persistently vulnerable» qualsiasi server Exchange che abbia raggiunto la fine del supporto o resti privo di patch per vulnerabilità note. L’obiettivo è proteggere i destinatari di Exchange Online dai messaggi provenienti da server compromettibili e, al contempo, esercitare pressione sugli operatori affinché aggiornino o spengano i server.

Il sistema è stato attivato gradualmente per versione:

| Periodo | Versione interessata |
|---|---|
| Agosto 2023 | Exchange 2007 |
| Settembre 2023 | Exchange 2010 |
| Dicembre 2023 | Exchange 2013 |
| Marzo 2024 | Exchange 2016 e 2019 (livelli SU molto obsoleti) |
| Settembre 2026 | Exchange 2016 e 2019: limite minimo = SU di ottobre 2025 |
| «tra alcuni mesi» | Exchange 2016 e 2019: limite minimo superiore all’ultimo aggiornamento pubblico |

Per Exchange 2016 e 2019, il limite minimo era finora costituito da livelli «significantly behind on security updates». La novità è che Microsoft porta il limite all’ultimo aggiornamento pubblico, colpendo per la prima volta server che meno di un anno fa erano ancora completamente aggiornati.

## I livelli di escalation

L’enforcement opera in tre funzioni che Microsoft chiama «reporting», «throttling» e «blocking». Non appena un server scende sotto il limite minimo, inizia un ciclo di 90 giorni. I livelli descritti nell’articolo di base del 2023:

| Periodo | Misura | Risposta SMTP |
|---|---|---|
| Giorno 0-30 | Solo report nell’Exchange Admin Center | nessuna |
| Giorno 30-40 | Limitazione per 5 minuti all’ora | `450 4.7.230` |
| Giorno 40-50 | Limitazione per 10 minuti all’ora | `450 4.7.230` |
| Giorno 50-60 | Limitazione per 20 minuti all’ora | `450 4.7.230` |
| Giorno 60-70 | Limitazione per 30 minuti all’ora, più blocco per 5 minuti all’ora | `450 4.7.230` e `550 5.7.230` |
| Giorno 70-80 | Blocco per 10 minuti all’ora | `550 5.7.230` |
| Giorno 80-90 | Blocco per 20 minuti all’ora | `550 5.7.230` |
| Dal giorno 90 | Blocco completo | `550 5.7.230` |

Le due risposte, testualmente, sono:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La differenza è decisiva per l’esercizio. Con `450`, Exchange Online rifiuta temporaneamente la connessione; il server on-premises mantiene il messaggio nella propria coda e ritenta. Gli utenti notano inizialmente solo ritardi; nel Queue Viewer o in `Get-Queue` cresce la coda verso il Send Connector per Exchange Online, con stato `Retry` e il messaggio 4.7.230 come `LastError`. Con `550` il rifiuto è definitivo: il mittente riceve un NDR con il codice 5.7.230 e il messaggio viene perso, a meno che non venga inviato nuovamente. Poiché inizialmente il blocco è attivo solo per alcuni minuti all’ora, il comportamento dell’errore appare dapprima sporadico: una parte dei messaggi arriva, una parte fallisce con NDR. Chi osserva questo schema nel Message Tracking dovrebbe prima verificare la versione, prima di cercare problemi di rete o certificati.

L’annuncio non precisa se Microsoft avvierà il ciclo completo di 90 giorni per il nuovo limite minimo dalla seconda settimana di settembre o se entrerà già in un livello successivo. L’articolo di base specifica che, dopo una pausa, il sistema prosegue dal livello precedentemente raggiunto. Non fate quindi affidamento su 30 giorni di tolleranza.

## Report nell’Exchange Admin Center e tramite PowerShell

Exchange Online elenca i server on-premises rilevati con la relativa versione in un report dedicato: nell’Exchange Admin Center, sotto *Reports*, *Mail flow*, report sui server Exchange on-premises connessi obsoleti («out-of-date connecting on-premises Exchange servers»). Per ogni server, il report mostra la build rilevata, se è al di sotto del limite minimo e a quale livello si trova l’enforcement.

Le stesse informazioni sono disponibili con Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Comando | Effetto |
|---|---|
| `Connect-ExchangeOnline` | Apre la sessione con Exchange Online (modulo `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Elenca i server on-premises rilevati da Exchange Online con build, stato e livello di enforcement. |

</details>

Il report conosce solo i server che consegnano effettivamente email a Exchange Online. Un server di gestione senza flusso di posta o una macchina con i soli Management Tools non compare. Questo è irrilevante per l’enforcement, ma non per la sicurezza: anche questi sistemi necessitano delle SU.

## Mettere in pausa l’enforcement: 90 giorni all’anno

Per gli ambienti che non riescono a raggiungere il limite minimo nel breve termine, Microsoft offre una pausa. Può essere attivata per un totale di 90 giorni all’anno, continuativamente o in più periodi:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-TenantExemptionInfo` | Mostra se e per quanto tempo è attiva una pausa per il tenant. |
| `New-TenantExemptionInfo` | Crea una nuova pausa. |
| `-BlockingScenario UnpatchedOnPremServer` | Seleziona lo scenario «server on-premises obsoleto»; attualmente non esistono altri scenari per questo cmdlet. |
| `-NumberOfDays 30` | Durata della pausa in giorni. Il contingente è di 90 giorni all’anno e l’indicazione viene sottratta da esso. |

</details>

Due caratteristiche della pausa sono importanti nella pratica. Primo, alla scadenza l’enforcement prosegue dal livello in cui era stato fermato; la pausa non reimposta il ciclo di 90 giorni. Secondo, non esiste alcun cmdlet per terminare anticipatamente una pausa in corso: chi crea una pausa di 90 giorni e completa l’aggiornamento dopo due settimane ha esaurito il contingente annuale. Create quindi la pausa per il periodo più breve possibile e prolungatela se necessario.

Inoltre, la pausa è una soluzione solo per il limite minimo attuale. Se Microsoft innalzerà il limite tra alcuni mesi al di sopra dell’ultimo aggiornamento pubblico, un contingente esaurito non sarà più d’aiuto.

## Perché il prossimo innalzamento è la vera scadenza

Exchange 2016 e 2019 sono fuori supporto dal 14 ottobre 2025. Microsoft ha successivamente predisposto due periodi ESU a pagamento: periodo 1 fino ad aprile 2026, periodo 2 da maggio a ottobre 2026. Con l’annuncio del periodo 2 del 15 aprile 2026, il team di Exchange ha chiarito che non ci sarà alcuna ulteriore proroga. Le SU da dicembre 2025 a ottobre 2026 (da ultimo build 15.2.1748.55 per 2019 CU15 e 15.1.2507.77 per 2016 CU23, entrambe del 13 ottobre 2026) sono disponibili esclusivamente per i clienti ESU e non vengono offerte pubblicamente per il download.

Ne deriva la seguente situazione:

- **Oggi** un server con la SU di ottobre 2025 soddisfa il limite minimo, con o senza ESU.
- **Con il prossimo innalzamento** il limite minimo, secondo Microsoft, sarà superiore al livello di ottobre 2025. Senza un contratto ESU non esiste un modo legale per raggiungere quel livello. Il flusso di posta ibrido di questi server verrà quindi limitato e bloccato, indipendentemente da quanto bene sia gestito il resto dell’ambiente.
- **Il 31 ottobre 2026** termina anche il periodo 2. Microsoft ha pubblicato gli ultimi aggiornamenti per Exchange 2016 e 2019 con la SU di ottobre del 13 ottobre 2026; dopo non ci saranno più SU per nessuno. Al più tardi il successivo innalzamento dopo quello prossimo interesserà quindi anche i clienti ESU.

Il programma ESU acquista dunque, nel migliore dei casi, pochi mesi. L’unico livello permanente consentito dall’enforcement è Exchange Server SE. Microsoft ha inoltre annunciato che Exchange SE CU2 (originariamente previsto per la seconda metà del 2026; poiché Microsoft non indica attualmente una data nemmeno per il precedente CU1, non è disponibile una data neppure per CU2) interromperà la coesistenza con Exchange 2016 e 2019: l’installazione verrà annullata se nell’organizzazione vengono trovati server meno recenti. La migrazione è quindi necessaria non solo per il flusso di posta, ma anche per poter continuare a installare aggiornamenti per SE.

Per gli ambienti che mantengono Exchange On-Premises solo per la gestione degli attributi in una configurazione ibrida, l’alternativa è rimuovere l’ultimo server: da Exchange 2019 CU12, gli attributi dei destinatari possono essere gestiti tramite i Management Tools senza un server Exchange in esecuzione. In tal caso non esiste più un flusso di posta on-premises e l’enforcement non è più pertinente.

## Determinare la versione

`Get-ExchangeServer` mostra in `AdminDisplayVersion` solo la CU, non la SU. È affidabile la versione del file di `ExSetup.exe` o l’[Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), che segnala anche i passaggi manuali mancanti. Per una rapida panoramica di tutti i server:

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
<summary>Opzioni spiegate</summary>

| Elemento | Effetto |
|---|---|
| `Get-ExchangeServer` | Elenca tutti i server Exchange dell’organizzazione. |
| `\\<Server>\C$\...\ExSetup.exe` | Percorso di condivisione amministrativa al file di installazione; adattare in caso di percorso di installazione diverso. |
| `VersionInfo.ProductVersion` | Versione del file, corrispondente alla build SU installata (ad es. `15.1.2507.61`). |

</details>

Se la versione è inferiore a `15.2.1748.39` (2019 CU15) o rispettivamente `15.1.2507.61` (2016 CU23), il server sarà al di sotto del limite minimo dalla seconda settimana di settembre.

## Procedura consigliata

1. **Inventariare lo stato** come descritto sopra, inclusi Edge Transport Server e server di gestione.

2. **Verificare il report in Exchange Online.** `Get-OnPremServerReportInfo` mostra quali server Exchange Online rileva effettivamente e se è già attivo un livello di enforcement. Confrontate l’elenco con l’inventario: i server assenti non consegnano tramite il connettore `OnPremises`.

3. **Installare almeno la SU di ottobre 2025.** KB5066367 (2019 CU15) e KB5066369 (2016 CU23) sono ancora disponibili pubblicamente nel Microsoft Download Center. Le SU sono cumulative; un server al livello di agosto 2025 può essere aggiornato direttamente a ottobre 2025. Con CU14, installate prima CU15. Dopo l’installazione, riavviate, controllate lo stato dei servizi ed eseguite nuovamente Health Checker.

4. **Utilizzare la pausa solo come soluzione ponte.** Se l’aggiornamento non riesce nella prima metà di settembre, create `New-TenantExemptionInfo` con una durata breve e non considerate la pausa una riserva di pianificazione per il prossimo innalzamento.

5. **Pianificare la migrazione a Exchange SE.** Senza contratto ESU, il prossimo innalzamento è la scadenza inderogabile; con ESU, lo è il 31 ottobre 2026. Exchange 2019 CU15 può essere portato a SE tramite aggiornamento in-place; Exchange 2016 richiede il passaggio attraverso una nuova installazione di SE e lo spostamento delle cassette postali o dei ruoli. Chi usa Exchange solo per la gestione degli attributi rimuove l’ultimo server e continua a lavorare con i Management Tools.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: le SU di ottobre del 13 ottobre 2026 (build 15.2.1748.55 e 15.1.2507.77) sono gli ultimi aggiornamenti per Exchange 2016 e 2019. Per Exchange SE CU2 non è prevista alcuna data a causa del rinvio di CU1.

## Fonti

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): L’annuncio del 2 settembre 2026 con il nuovo limite minimo (SU di ottobre 2025), la data di avvio nella seconda settimana di settembre e l’indicazione del prossimo innalzamento oltre l’ultimo aggiornamento pubblico.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): L’articolo di base del 2023 con la definizione di «persistently vulnerable», i livelli Reporting, Throttling e Blocking, il ciclo di 90 giorni, le risposte SMTP 4.7.230 e 5.7.230 e il piano di rollout per versione.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): I cmdlet `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` e `New-TenantExemptionInfo`, incluso il report nell’Exchange Admin Center e il contingente annuale di 90 giorni.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Numeri di build delle SU di ottobre 2025 e dei successivi aggiornamenti ESU fino a ottobre 2026; include anche l’indicazione che le SU da dicembre 2025 sono disponibili solo per i clienti ESU.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): L’articolo KB relativo al livello minimo per Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): L’articolo KB relativo al livello minimo per Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): La fine del supporto il 14 ottobre 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Condizioni del primo periodo ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Durata da maggio a ottobre 2026, l’affermazione che non seguirà alcuna ulteriore proroga e l’integrazione del 13 ottobre 2026 sugli ultimi aggiornamenti per Exchange 2016 e 2019.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Analisi tabellare degli otto livelli di enforcement e delle date di rollout per ogni versione di Exchange; fonte di terze parti.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario dei livelli CU/SU e dei passaggi manuali ancora aperti.

12. [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Il team di Exchange sul rinvio di Exchange SE CU1 senza una data prevista.
