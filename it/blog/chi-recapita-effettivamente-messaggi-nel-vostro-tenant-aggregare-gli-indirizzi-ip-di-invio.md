---
title: "Chi recapita effettivamente messaggi nel vostro tenant? Aggregare gli indirizzi IP di invio"
navTitle: "IP di invio"
description: "Un’unica analisi mostra quali sistemi recapitano effettivamente e-mail nel vostro tenant: connettori dimenticati, applicazioni che inviano direttamente e fornitori di servizi che nessuno ha documentato. Inclusi i problemi della logica di paginazione e dell’interpretazione."
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
url: https://rafaelpfister.ch/it/blog/chi-recapita-effettivamente-messaggi-nel-vostro-tenant-aggregare-gli-indirizzi-ip-di-invio
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:12:45.695Z
translationReview: automatic
---

# Chi recapita effettivamente messaggi nel vostro tenant? Aggregare gli indirizzi IP di invio

Quasi nessun ambiente di posta sa ancora con precisione chi vi recapita messaggi. Negli anni si accumulano connettori di migrazioni, applicazioni che inviano direttamente, fornitori di servizi il cui contratto è scaduto da tempo e configurazioni di test mai smantellate. Finché la posta continua a fluire, nessuno se ne accorge.

Un’unica analisi fa chiarezza: il raggruppamento di tutti i messaggi in entrata per indirizzo IP di origine. Si prepara in due minuti e l’elenco dei risultati è regolarmente sorprendente. Questo articolo mostra la query, spiega come ottenerla **completa** e, soprattutto, come leggere correttamente i numeri. Perché l’interpretazione è la parte più difficile.

## Perché ne vale la pena

L’elenco risponde a quattro domande che altrimenti richiederebbero verifiche individuali laboriose. Quali sistemi inviano effettivamente messaggi nel vostro tenant? Tutto passa attraverso i percorsi documentati o esiste un secondo ingresso? Un connettore che volete dismettere è ancora utilizzato? E: un’applicazione invia direttamente al servizio bypassando il gateway, aggirando quindi il filtraggio?

Soprattutto l’ultima domanda è rilevante per la sicurezza. Chi recapita direttamente non aggira soltanto il filtraggio, ma spesso anche la registrazione su cui volete poter contare in caso di emergenza.

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

## Insidia 1: l’elenco è quasi sempre incompleto

`Get-MessageTraceV2` restituisce i risultati a pagine, con un massimo di 5000 righe per chiamata. Con un volume elevato, una pagina copre solo una frazione della finestra temporale. Raggruppate quindi un campione e scambiate il risultato per il totale.

Lo riconoscete da questo avviso:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Se compare questo avviso, la vostra analisi è inutile.** In particolare, l’assenza di una voce non può essere interpretata come assenza effettiva. Un indirizzo con tre messaggi al giorno non comparirà comunque in un campione.

Ci sono due soluzioni. Quella semplice: riducete la finestra temporale finché l’avviso non scompare. Con 5000 messaggi all’ora, saranno 55 minuti e non sette giorni. Per la domanda «quali sistemi inviano effettivamente messaggi», una breve finestra completa è di solito più che sufficiente.

L’approccio approfondito sfoglia tutte le pagine e raccoglie i risultati:

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

Per 24 ore in un ambiente di medie dimensioni, considerate alcuni minuti di esecuzione. Per un inventario una tantum, è tempo ben investito.

## Insidia 2: i numeri non significano ciò che sembrano significare

L’elenco dei risultati contiene quattro tipi di voci completamente diversi e chi li mette tutti nello stesso calderone trae conclusioni errate.

**`255.255.255.255` non rappresenta un sistema.** Questo valore compare quando per il messaggio non c’è stata una connessione SMTP in entrata dall’esterno. Riguarda i messaggi generati dal servizio stesso: rapporti di journaling, notifiche di mancato recapito, risposte automatiche di assenza, messaggi tra cassette postali dello stesso tenant. In quasi ogni ambiente questa è la voce più grande, ed è del tutto normale. Non allarmatevi.

**Gli indirizzi privati RFC 1918** provengono dalla vostra rete. Negli ambienti ibridi vedrete qui i server di trasporto locali, poiché il loro indirizzo interno viene mantenuto al momento della consegna al servizio. Sono i numeri più grandi dell’elenco e, di norma, rappresentano il percorso principale previsto.

**Gli indirizzi di servizi di sicurezza e filtraggio** si riconoscono dal gestore, non dal valore numerico. Proxy cloud, gateway di posta a monte e servizi di web security compaiono con molti indirizzi adiacenti e numeri medi. Di solito ne fanno parte, ma dovrebbero essere riportati nel manuale operativo.

**Gli indirizzi pubblici singoli con numeri bassi** sono quelli interessanti. È proprio lì che si nascondono le applicazioni dimenticate, i vecchi fornitori di servizi e sistemi di cui nessuno ricorda più l’esistenza.

## La risoluzione: dagli indirizzi ai nomi

Per tutto ciò che non riuscite ad attribuire immediatamente, è utile la risoluzione inversa. Non è sempre configurata e non è sempre affidabile, ma nella maggior parte dei casi fornisce l’indizio decisivo:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

L’assenza di un PTR non è la prova di qualcosa di malevolo, ma è un buon motivo per approfondire. Per tali indirizzi, esaminate i messaggi associati:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

Mittente e oggetto vi diranno di norma immediatamente quale applicazione vi si nasconde dietro.

## Il confronto: quale indirizzo appartiene a quale connettore?

Ora arriva il vero valore informativo. Confrontate l’elenco dei risultati con i connettori configurati:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Tre configurazioni sono rivelatrici.

**Un indirizzo recapita messaggi, ma non è indicato in alcun connettore.** In questo caso l’e-mail arriva come normale posta Internet. È consentito, ma significa che questa applicazione non riceve alcun trattamento speciale e che i suoi messaggi sono soggetti al filtraggio completo. Se qualcuno afferma che per questo sistema è configurato un connettore, evidentemente non è più così.

**Un connettore indica indirizzi da cui non arriva nulla.** È un candidato alla dismissione. Prima di eliminarlo, verificate se si tratta di sistemi stagionali o rari e disattivatelo prima di rimuoverlo definitivamente.

**Un connettore imposta `TreatMessagesAsInternal` o `CloudServicesMailEnabled` su vero.** Qui vale la pena guardare attentamente. Entrambe le impostazioni fanno sì che i messaggi provenienti da questo percorso vengano trattati come interni all’organizzazione. Se da lì arriva posta da Internet, vengono così aggirati controlli pensati per i messaggi esterni, compresa la protezione dai mittenti contraffatti appartenenti al proprio dominio. Per un connettore ibrido puro è corretto; per un connettore attraverso il quale possono recapitare sistemi qualsiasi, è un riscontro significativo.

## Cosa troverete tipicamente

Dalla pratica, senza pretesa di completezza: un connettore di test di una migrazione, attivo da anni. Un’applicazione specialistica che invia direttamente al servizio, anche se tutti credono che passi attraverso il gateway. Un fornitore di newsletter il cui contratto è scaduto, ma che può ancora effettuare consegne. E regolarmente un connettore con condizioni troppo permissive, creato una volta da qualcuno per risolvere un problema urgente.

Nessuno di questi riscontri è drammatico di per sé. Insieme, delineano un ambiente di cui nessuno ha più una visione completa, ed è proprio questo il vero rischio.

## Limiti del metodo

Dovreste conoscere tre limitazioni.

Il tracciamento dei messaggi tramite il cmdlet risale solo a circa dieci giorni. Per periodi più lunghi vi serve la ricerca storica, che viene eseguita in modo asincrono e copre fino a 90 giorni. Altrimenti vi sfuggiranno i sistemi rari che inviano mensilmente.

`FromIP` non significa la stessa cosa ovunque. Per la posta da Internet è l’indirizzo del server che effettua la consegna. Per la posta ibrida è l’indirizzo del vostro server di trasporto locale, non quello del mittente originario. L’analisi mostra quindi **l’ultima stazione prima del servizio**, non l’origine.

Inoltre, l’associazione a un connettore non è visibile direttamente nel tenant. La deducete dall’indirizzo, dal certificato e dal dominio del mittente. Per una valutazione affidabile dell’uso di un singolo connettore, il report dei connettori nell’Exchange Admin Center, in Report e flusso di posta, è la fonte migliore, perché aggrega lato server su periodi più lunghi.

## Come controllo ricorrente

Questa analisi si presta bene come routine trimestrale. Archiviate il risultato e confrontatelo al passaggio successivo. I nuovi indirizzi nell’elenco sono modifiche documentate oppure qualcosa che vorrete conoscere.

Se state già verificando la configurazione e-mail dei vostri domini: il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) mostra SPF, DKIM, DMARC e gli altri standard e-mail per qualsiasi dominio direttamente nel browser, anche per i domini secondari e di marketing che, in questi inventari, vengono notoriamente dimenticati. E per le query stesse, il [Generatore di comandi](https://rafaelpfister.ch/tools/command-builder) fornisce componenti pronti per PowerShell e Unix shell.

Come seguire ulteriormente singoli messaggi sospetti è spiegato in [Analizzare il flusso di posta Exchange: Message Tracking, protocolli SMTP e connettori di ricezione](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Fonti

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): elenco dei campi, inclusi FromIP e ToIP, nonché la logica di paginazione con StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): tracciamento asincrono dei messaggi fino a 90 giorni per periodi più vecchi.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parametri dei connettori in entrata, tra cui SenderIPAddresses e TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interazione tra i tipi di connettore e quando si applica ciascuno di essi.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): definisce gli intervalli di indirizzi privati che dovete distinguere dagli indirizzi pubblici nell’analisi.
