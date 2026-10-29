---
title: "Chi recapita effettivamente messaggi al vostro tenant? Aggregare gli indirizzi IP mittenti"
navTitle: "IP mittenti"
description: "Un’unica analisi mostra quali sistemi recapitano effettivamente e-mail al vostro tenant: connettori dimenticati, applicazioni che inviano direttamente e fornitori di servizi che nessuno ha documentato, inclusi i tipici errori di analisi nella logica di paginazione e nell’interpretazione."
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
translationSourceHash: 291ed92e756d5fa011212a5cf9d3ce9061df96d73857f508c838183965ba25b4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:38:47.259Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/chi-recapita-effettivamente-messaggi-nel-vostro-tenant-aggregare-gli-indirizzi-ip-di-invio
---

# Chi recapita effettivamente messaggi al vostro tenant? Aggregare gli indirizzi IP mittenti

Quasi nessun ambiente di posta sa più con completezza chi recapita messaggi al suo interno. Nel corso degli anni si accumulano connettori derivanti da migrazioni, applicazioni che inviano direttamente, fornitori di servizi il cui contratto è scaduto da tempo e configurazioni di test mai dismesse. Finché la posta continua a fluire, nessuno se ne accorge.

Un’unica analisi fa chiarezza: il raggruppamento di tutti i messaggi in entrata in base al loro indirizzo IP di origine. Si prepara in due minuti e l’elenco dei risultati riserva regolarmente sorprese. Questo articolo mostra la query, spiega come ottenerla **completa** e, soprattutto, come leggere correttamente i numeri. Perché l’interpretazione è la parte più difficile.

## Perché conviene

L’elenco risponde a quattro domande che altrimenti richiederebbero verifiche separate e laboriose. Quali sistemi inviano effettivamente messaggi al vostro tenant? Tutto passa attraverso i percorsi documentati oppure esiste un secondo punto di ingresso? Un connettore che volete dismettere è ancora in uso? E ancora: un’applicazione invia direttamente al servizio, bypassando il vostro gateway e quindi eludendo il filtraggio?

L’ultima domanda in particolare è rilevante per la sicurezza. Chi invia direttamente non aggira solo il filtraggio, ma spesso anche la registrazione su cui volete poter contare in caso di incidente.

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Inizio della finestra di query, qui due ore fa |
| `-EndDate (Get-Date)` | Fine della finestra di query, il momento attuale |
| `-ResultSize 5000` | numero massimo di righe per chiamata; 5000 è anche il valore massimo |
| `Group-Object FromIP` | raggruppa i messaggi in base all’indirizzo IP mittente |
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

Prima di trarre conclusioni, devono essere vere due cose: l’elenco deve essere completo e dovete sapere cosa significano le voci.

## Fonte di errore 1: l’elenco è quasi sempre incompleto

`Get-MessageTraceV2` restituisce i risultati a pagine, con un massimo di 5000 righe per chiamata. In presenza di un volume elevato, una pagina copre solo una frazione della finestra temporale. Raggruppate quindi un estratto e considerate il risultato come se fosse il totale.

Lo riconoscete da questo avviso:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Se compare questo avviso, la vostra analisi è priva di valore.** In particolare, l’assenza di una voce non deve essere interpretata come assenza effettiva. Un indirizzo con tre messaggi al giorno non comparirà comunque in un estratto.

Esistono due soluzioni. La più semplice: riducete la finestra temporale finché l’avviso non scompare. Con 5000 messaggi all’ora si tratta di 55 minuti, non di sette giorni. Per rispondere alla domanda «quali sistemi inviano in assoluto?» di solito è sufficiente una breve finestra completa.

Il metodo più accurato scorre tutte le pagine e raccoglie i risultati:

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
    if ($seite.Count -eq 5000) {
        $naechster = $seite[-1].RecipientAddress
        $ende      = $seite[-1].Received
    } else {
        $naechster = $null
    }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` / `-EndDate` | Finestra di query, qui le ultime 24 ore |
| `-StartingRecipientAddress` | punto di ripresa della logica di paginazione: l’indirizzo del destinatario dell’ultima voce della pagina precedente; insieme a `-EndDate` = la relativa ora di ricezione (`Received`) inizia lì la pagina successiva |
| `-ResultSize 5000` | dimensione della pagina; una pagina completa segnala che seguono altri risultati |
| `Group-Object FromIP` | raggruppa l’intero insieme in base all’indirizzo IP mittente |
| `Sort-Object Count -Descending` | ordina i gruppi in ordine decrescente per numero di messaggi |
| `Format-Table Count, Name -AutoSize` | output del numero per indirizzo con larghezza automatica delle colonne |

</details>

Il ciclo richiama ulteriori pagine finché riceve una pagina di esattamente 5000 righe e, ogni volta, riprende con l’indirizzo del destinatario e l’ora di ricezione dell’ultima voce della pagina precedente, come previsto dalla documentazione di `Get-MessageTraceV2`; solo l’intero insieme completo viene raggruppato.

Per 24 ore in un ambiente di medie dimensioni, calcolate alcuni minuti di esecuzione. Per un inventario una tantum è tempo ben investito.

## Fonte di errore 2: i numeri non significano ciò che sembrano significare

L’elenco dei risultati contiene quattro tipi di voci profondamente diversi e chi li mette tutti nello stesso calderone trae conclusioni errate.

**`255.255.255.255` non rappresenta un sistema.** Questo valore compare quando per il messaggio non esiste una connessione SMTP in entrata dall’esterno. Riguarda messaggi generati dal servizio stesso: rapporti di journaling, notifiche di mancato recapito, risposte automatiche di assenza, messaggi tra caselle di posta dello stesso tenant. In quasi tutti gli ambienti è la voce più grande ed è del tutto normale.

**Gli indirizzi privati RFC 1918** provengono dalla vostra rete. Negli ambienti ibridi vedrete qui i server di trasporto locali, poiché il loro indirizzo interno viene mantenuto al momento del passaggio al servizio. Sono i numeri elevati nell’elenco e, nella maggior parte dei casi, rappresentano il percorso principale atteso.

**Gli indirizzi dei servizi di sicurezza e filtraggio** si riconoscono dal gestore, non dal valore numerico. Proxy cloud, gateway di posta a monte e servizi di sicurezza web compaiono con molti indirizzi adiacenti e numeri intermedi. Di solito sono previsti, ma dovrebbero figurare nel manuale operativo.

**I singoli indirizzi pubblici con numeri bassi** sono quelli interessanti. È proprio lì che si nascondono applicazioni dimenticate, vecchi fornitori di servizi e sistemi di cui nessuno ha più memoria.

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | interroga il record inverso (PTR) del rispettivo indirizzo IP |
| `-ErrorAction Stop` | trasforma un record mancante in un errore gestibile dal blocco `try`/`catch` |
| `[pscustomobject]@{ … }` | crea per ogni indirizzo un oggetto con IP e nome risolto per l’output tabellare |
| `Format-Table -AutoSize` | output con larghezza automatica delle colonne |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

Un PTR mancante non è di per sé un’indicazione di un problema, ma è un buon motivo per esaminare più da vicino l’indirizzo. Per questi indirizzi, analizzate i messaggi associati:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Finestra di query e dimensione della pagina come nella query principale |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtra lato client per l’indirizzo di origine in questione |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | mostra per messaggio ora di ricezione, mittente, destinatario, oggetto e stato di recapito |

</details>

Mittente e oggetto vi diranno quasi sempre subito quale applicazione si nasconde dietro.

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
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `Get-InboundConnector` | elenca tutti i connettori in entrata del tenant; qui volutamente senza parametri restrittivi |
| `Format-List <Eigenschaften>` | output come elenco delle proprietà indicate, una per riga |
| `@{n='…'; e={…}}` | proprietà calcolata con nome (`n`) ed espressione (`e`) |
| `-join ', '` | trasforma l’array degli indirizzi o dei domini in una riga leggibile separata da virgole |

</details>

Tre configurazioni sono rivelatrici.

**Un indirizzo invia messaggi, ma non è indicato in alcun connettore.** In questo caso la posta arriva come normale e-mail Internet. È consentito, ma significa che questa applicazione non beneficia di alcun trattamento speciale e che i suoi messaggi sono soggetti al filtraggio completo. Se qualcuno sostiene che per questo sistema sia configurato un connettore, evidentemente non è più così.

**Un connettore indica indirizzi da cui non arriva nulla.** È il candidato alla dismissione. Prima di eliminarlo, verificate se si tratta di sistemi stagionali o usati raramente e disabilitatelo prima di rimuoverlo definitivamente.

**Un connettore imposta `TreatMessagesAsInternal` o `CloudServicesMailEnabled` su vero.** Vale la pena esaminarlo attentamente. Entrambe le impostazioni fanno sì che i messaggi provenienti da questo percorso vengano trattati come interni all’organizzazione. Se da lì entra posta da Internet, aggira quindi controlli pensati per i messaggi esterni, incluso il controllo contro mittenti contraffatti del proprio dominio. Per un connettore ibrido puro è corretto; per un connettore attraverso cui possono inviare sistemi arbitrari, è un riscontro rilevante.

## Cosa trovate di solito

Dalla pratica, senza pretesa di completezza: un connettore di test di una migrazione attivo da anni. Un’applicazione aziendale che invia direttamente al servizio, sebbene tutti ritengano che passi dal gateway. Un fornitore di newsletter il cui contratto è scaduto, ma che può ancora recapitare messaggi. E regolarmente un connettore con condizioni troppo permissive, creato una volta per risolvere un problema urgente.

Nessuno di questi riscontri è drammatico di per sé. Insieme, però, delineano un ambiente che nessuno controlla più completamente, ed è proprio questo il vero rischio.

## Limiti del metodo

Dovreste conoscere tre limitazioni.

`Get-MessageTraceV2` copre gli ultimi 90 giorni, ma restituisce al massimo dieci giorni per query. Per periodi più lunghi, eseguite quindi query su più finestre temporali o utilizzate la ricerca storica, che viene eseguita in modo asincrono e copre anch’essa fino a 90 giorni. Altrimenti i sistemi rari, che inviano una volta al mese, vi sfuggiranno.

`FromIP` non significa la stessa cosa in ogni contesto. Per la posta da Internet è l’indirizzo del server mittente. Per la posta ibrida è l’indirizzo del vostro server di trasporto locale, non quello del mittente originario. L’analisi mostra quindi **l’ultima stazione prima del servizio**, non l’origine.

Inoltre, l’associazione a un connettore non è visibile direttamente nel tenant. La deducete da indirizzo, certificato e dominio mittente. Per una valutazione affidabile dell’utilizzo di un singolo connettore, il rapporto sui connettori nell’Exchange Admin Center, in Rapporti e Flusso di posta, è una fonte migliore, perché aggrega lato server su periodi più lunghi.

## Come verifica ricorrente

Questa analisi è adatta come routine trimestrale. Archiviate il risultato e confrontatelo alla successiva esecuzione. Nuovi indirizzi nell’elenco corrispondono a modifiche documentate oppure a qualcosa che dovreste conoscere.

Se state già verificando la configurazione di posta dei vostri domini: il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) mostra SPF, DKIM, DMARC e gli altri standard e-mail per qualsiasi dominio direttamente nel browser, anche per domini secondari e di marketing che, secondo l’esperienza, vengono dimenticati in questi inventari. E per le query stesse, il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) fornisce blocchi pronti per PowerShell e shell Unix.

Come seguire singoli messaggi sospetti è descritto in [Analizzare il flusso di posta Exchange: Message Tracking, protocolli SMTP e connettori di ricezione](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: la logica di paginazione di Get-MessageTraceV2 riprende con l’indirizzo del destinatario e l’ora di ricezione (EndDate) dell’ultima voce. Corretto: il cmdlet copre gli ultimi 90 giorni, ma restituisce al massimo dieci giorni per query.

## Fonti

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): elenco dei campi incluso FromIP e ToIP, intervallo di query (90 giorni, massimo dieci giorni per query) e ripresa con StartingRecipientAddress e EndDate.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni per periodi meno recenti.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametri dei connettori in entrata, inclusi SenderIPAddresses e TreatMessagesAsInternal.

4.  [Configurare il flusso di posta mediante connettori in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interazione tra i tipi di connettori e casi in cui ciascuno viene applicato.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definisce gli intervalli di indirizzi privati che dovete distinguere dagli indirizzi pubblici nell’analisi.
