---
title: "Chi sta effettivamente inviando messaggi al vostro tenant? Aggregare gli indirizzi IP di invio"
navTitle: "IP di invio"
description: "Un’unica analisi mostra quali sistemi inviano effettivamente e-mail al vostro tenant: connettori dimenticati, applicazioni che inviano direttamente e fornitori che nessuno ha documentato, inclusi i tipici errori di analisi nella logica di paginazione e nell’interpretazione."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min di lettura"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "chi-recapita-effettivamente-messaggi-nel-vostro-tenant-aggregare-gli-indirizzi-ip-di-invio"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
translationSourceHash: 9209720819061360cb72bfa01ab6261e6af80e547a398c25f6802edfbe49bb6c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:05:45.210Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/chi-recapita-effettivamente-messaggi-nel-vostro-tenant-aggregare-gli-indirizzi-ip-di-invio
---

# Chi sta effettivamente inviando messaggi al vostro tenant? Aggregare gli indirizzi IP di invio

Quasi nessun ambiente di posta sa ancora con certezza chi gli invia messaggi. Negli anni si accumulano connettori provenienti da migrazioni, applicazioni che inviano direttamente, fornitori il cui contratto è scaduto da tempo e configurazioni di test mai smantellate. Finché la posta fluisce, nessuno se ne accorge.

Un’unica analisi fa chiarezza: il raggruppamento di tutti i messaggi in arrivo in base al loro indirizzo IP di origine. Si prepara in due minuti e l’elenco dei risultati riserva regolarmente sorprese. Questo articolo mostra la query, spiega come ottenerla **completa** e, soprattutto, come leggere correttamente i numeri. Perché l’interpretazione è la parte più difficile.

## Perché vale la pena

L’elenco risponde a quattro domande che altrimenti richiederebbero chiarimenti laboriosi e separati. Quali sistemi inviano effettivamente messaggi al vostro tenant? Tutto passa attraverso i percorsi documentati oppure esiste un secondo ingresso? Un connettore che volete dismettere è ancora in uso? E: un’applicazione invia direttamente al servizio aggirando il vostro gateway e quindi il vostro filtraggio?

Soprattutto l’ultima domanda è rilevante per la sicurezza. Chi invia direttamente non aggira solo il filtraggio, ma spesso anche la registrazione su cui vorrete fare affidamento in caso di emergenza.

## La query

Nel tenant raggruppate il tracciamento dei messaggi in base a `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Inizio della finestra di query, qui due ore fa |
| `-EndDate (Get-Date)` | Fine della finestra di query, il momento attuale |
| `-ResultSize 5000` | numero massimo di righe per chiamata; 5000 è anche il valore massimo |
| `Group-Object FromIP` | raggruppa i messaggi in base all’indirizzo IP che li ha inviati |
| `Sort-Object Count -Descending` | ordina i gruppi in ordine decrescente per numero di messaggi |
| `Format-Table Count, Name -AutoSize` | output a due colonne (numero, indirizzo IP) con larghezza automatica delle colonne |

</details>

Un output tipico:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Prima di trarre conclusioni, devono essere corrette due cose: l’elenco deve essere completo e dovete sapere cosa significano le voci.

## Fonte di errore 1: l’elenco è quasi sempre incompleto

`Get-MessageTraceV2` restituisce risultati a pagine, con un massimo di 5000 righe per chiamata. In caso di volumi elevati, una pagina copre solo una frazione della vostra finestra temporale. Raggruppate quindi un estratto e considerate il risultato come l’insieme completo.

Lo riconoscete da questo avviso:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Se appare questo avviso, la vostra analisi non vale nulla.** In particolare, l’assenza di una voce non deve essere interpretata come assenza effettiva. Un indirizzo con tre messaggi al giorno non comparirà comunque in un estratto.

Esistono due soluzioni. Quella semplice: riducete la finestra temporale finché l’avviso non compare più. Con 5000 messaggi all’ora, sono 55 minuti e non sette giorni. Per la domanda «quali sistemi inviano in assoluto», una breve finestra completa è di solito più che sufficiente.

L’approccio approfondito scorre tutte le pagine e raccoglie i risultati:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` / `-EndDate` | Finestra di query, qui le ultime 24 ore |
| `-StartingRecipientAddress` | Punto di continuazione della logica di paginazione: l’indirizzo del destinatario dal quale inizia la pagina successiva |
| `-ResultSize 5000` | Dimensione della pagina; una pagina piena segnala che seguiranno altri risultati |
| `Group-Object FromIP` | raggruppa l’intero insieme in base all’indirizzo IP che ha inviato i messaggi |
| `Sort-Object Count -Descending` | ordina i gruppi in ordine decrescente per numero di messaggi |
| `Format-Table Count, Name -AutoSize` | output del numero per indirizzo con larghezza automatica delle colonne |

</details>

Il ciclo richiama altre pagine finché una pagina restituisce esattamente 5000 righe e ogni volta riprende dall’ultimo indirizzo del destinatario della pagina precedente; solo l’insieme completo viene raggruppato.

Per 24 ore in un ambiente di medie dimensioni, calcolate alcuni minuti di esecuzione. Per un inventario una tantum, è tempo ben investito.

## Fonte di errore 2: i numeri non significano ciò che sembrano significare

L’elenco dei risultati contiene quattro tipi di voci completamente diversi e chi li mette tutti nello stesso calderone giunge a conclusioni errate.

**`255.255.255.255` non rappresenta un sistema.** Questo valore appare quando per il messaggio non c’è stata una connessione SMTP in entrata dall’esterno. Riguarda i messaggi generati dal servizio stesso: rapporti di journaling, notifiche di mancata consegna, risposte automatiche di assenza, messaggi tra cassette postali dello stesso tenant. In quasi ogni ambiente è la voce più consistente ed è del tutto normale.

**Gli indirizzi privati secondo RFC 1918** provengono dalla vostra rete. Negli ambienti ibridi qui vedrete i server di trasporto locali, poiché il loro indirizzo interno viene mantenuto al momento della consegna al servizio. Sono i numeri elevati dell’elenco e, nella maggior parte dei casi, il percorso principale previsto.

**Gli indirizzi dei servizi di sicurezza e filtraggio** si riconoscono dal gestore, non dal valore numerico. Cloud proxy, gateway di posta a monte e servizi di sicurezza web compaiono con molti indirizzi adiacenti e valori medi. Di solito ne fanno parte, ma dovrebbero essere riportati nel manuale operativo.

**I singoli indirizzi pubblici con numeri bassi** sono quelli interessanti. È proprio lì che si nascondono le applicazioni dimenticate, i vecchi fornitori e i sistemi di cui nessuno ha più memoria.

## La risoluzione: dagli indirizzi ai nomi

Per tutto ciò che non riuscite ad attribuire immediatamente, è utile la risoluzione inversa. Non è sempre configurata né sempre affidabile, ma nella maggior parte dei casi fornisce l’indizio decisivo:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | interroga il record inverso (PTR) del rispettivo indirizzo IP |
| `-ErrorAction Stop` | trasforma un record mancante in un errore intercettabile per il blocco `try`/`catch` |
| `[pscustomobject]@{ … }` | crea per ogni indirizzo un oggetto con IP e nome risolto per l’output tabellare |
| `Format-Table -AutoSize` | output con larghezza automatica delle colonne |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

L’assenza di un PTR non è di per sé un’indicazione di un problema, ma è un buon motivo per esaminare più da vicino la situazione. Per tali indirizzi, analizzate i messaggi corrispondenti:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Finestra di query e dimensione della pagina come nella query principale |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtra lato client per l’indirizzo di origine in questione |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | mostra per ogni messaggio ora di ricezione, mittente, destinatario, oggetto e stato di consegna |

</details>

Mittente e oggetto vi diranno di regola immediatamente quale applicazione c’è dietro.

## Il confronto: quale indirizzo appartiene a quale connettore?

Confrontate il vostro elenco dei risultati con i connettori configurati:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-InboundConnector` | elenca tutti i connettori in entrata del tenant; qui volutamente senza parametri restrittivi |
| `Format-List <Eigenschaften>` | output come elenco delle proprietà indicate, una per riga |
| `@{n='…'; e={…}}` | proprietà calcolata con nome (`n`) ed espressione (`e`) |
| `-join ', '` | trasforma l’array degli indirizzi o dei domini in una riga leggibile separata da virgole |

</details>

Tre configurazioni sono rivelatrici.

**Un indirizzo invia messaggi, ma non è indicato in alcun connettore.** La posta arriva quindi come normale posta Internet. È consentito, ma significa che questa applicazione non beneficia di alcun trattamento speciale e che i suoi messaggi sono soggetti al filtraggio completo. Se qualcuno sostiene che per questo sistema sia configurato un connettore, evidentemente non è più così.

**Un connettore indica indirizzi dai quali non arriva nulla.** È il candidato alla dismissione. Prima di eliminarlo, verificate se si tratta di sistemi stagionali o rari e disattivatelo prima, anziché rimuoverlo subito.

**Un connettore imposta `TreatMessagesAsInternal` o `CloudServicesMailEnabled` su vero.** Qui vale la pena osservare attentamente. Entrambe le impostazioni fanno sì che i messaggi ricevuti attraverso questo percorso vengano trattati come interni all’organizzazione. Se da qui arriva posta da Internet, essa aggira i controlli previsti per i messaggi esterni, inclusa la protezione contro mittenti contraffatti del proprio dominio. Per un connettore ibrido puro è corretto; per un connettore attraverso il quale inviano sistemi arbitrari, è un riscontro significativo.

## Cosa trovate tipicamente

Dalla pratica, senza pretesa di completezza: un connettore di test da una migrazione, attivo da anni. Un’applicazione aziendale che invia direttamente al servizio, anche se tutti credono che passi dal gateway. Un fornitore di newsletter il cui contratto è scaduto, ma che può ancora effettuare consegne. E regolarmente un connettore con condizioni troppo permissive, creato una volta per risolvere un problema urgente.

Nessuno di questi riscontri è drammatico di per sé. Insieme delineano un ambiente di cui nessuno ha più una visione completa, ed è proprio questo il rischio reale.

## Limiti del metodo

Dovreste conoscere tre limitazioni.

Il tracciamento dei messaggi tramite il cmdlet risale solo a circa dieci giorni. Per periodi più lunghi vi serve la ricerca storica, che viene eseguita in modo asincrono e copre fino a 90 giorni. Altrimenti i sistemi rari, che inviano mensilmente, vi sfuggiranno.

`FromIP` non significa la stessa cosa ovunque. Per la posta proveniente da Internet è l’indirizzo del server che ha consegnato il messaggio. Per la posta ibrida è l’indirizzo del vostro server di trasporto locale, non quello del mittente originale. L’analisi mostra quindi **l’ultima tappa prima del servizio**, non l’origine.

E l’associazione a un connettore non è visibile direttamente nel tenant. La deducete da indirizzo, certificato e dominio del mittente. Per un’affermazione affidabile sull’uso di un singolo connettore, il rapporto sui connettori nell’Exchange Admin Center, sotto Rapporti e Flusso di posta, è la fonte migliore, perché aggrega lato server su periodi più lunghi.

## Come verifica ricorrente

Questa analisi si presta bene come routine trimestrale. Archiviate il risultato e confrontatelo al passaggio successivo. Nuovi indirizzi nell’elenco sono modifiche documentate oppure qualcosa che vorrete conoscere.

Se state già verificando la configurazione di posta dei vostri domini: il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) mostra SPF, DKIM, DMARC e gli altri standard di posta per qualsiasi dominio direttamente nel browser, anche per domini secondari e di marketing che, secondo l’esperienza, vengono dimenticati in questi inventari. E per le query stesse, il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) fornisce moduli pronti per PowerShell e Unix shell.

Per sapere come seguire ulteriormente singoli messaggi sospetti, consultate [Analizzare il flusso di posta di Exchange: Message Tracking, protocolli SMTP e Receive Connector](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Fonti

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): elenco dei campi, inclusi FromIP e ToIP, nonché la logica di paginazione con StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni per periodi meno recenti.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametri dei connettori in entrata, tra cui SenderIPAddresses e TreatMessagesAsInternal.

4.  [Configurare il flusso di posta usando i connettori in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interazione tra i tipi di connettore e quando si applica ciascuno.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definisce gli intervalli di indirizzi privati che dovete distinguere dagli indirizzi pubblici nell’analisi.
