---
title: "Voicemail di Teams via e-mail: nuovo mittente e nuovo oggetto da marzo 2027 (MC1511072)"
navTitle: "Mittente voicemail"
description: "Microsoft modifica la consegna via e-mail delle voicemail di Teams: il mittente sarà «Microsoft Teams» dal proprio dominio tenant e i dati del chiamante saranno spostati nell’oggetto. L’articolo mostra chi è interessato, come verificare regole di trasporto, liste di autorizzazione e automazioni e cosa adeguare prima di marzo 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min di lettura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "voicemail-di-teams-via-e-mail-nuovo-mittente-e-nuovo-oggetto-da-marzo-2027-mc1511072"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/it/blog/voicemail-di-teams-via-e-mail-nuovo-mittente-e-nuovo-oggetto-da-marzo-2027-mc1511072
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:36:58.177Z
translationReview: automatic
---

Con il messaggio del Centro messaggi MC1511072 del 16 ottobre 2026, Microsoft annuncia una nuova modalità di recapito delle e-mail di voicemail di Teams. Secondo il piano attuale, il rollout inizierà all’inizio di marzo 2027 e dovrebbe concludersi entro la fine di giugno 2027; Microsoft si riserva modifiche e intende comunicare le date vincolanti e le eventuali azioni richieste ai clienti in un ulteriore messaggio. Lo scopo è supportare domini mittente personalizzati (Bring Your Own Domain), affinché le e-mail di voicemail siano conformi ai criteri SPF, DKIM e DMARC dell’organizzazione.

## Cosa cambia

Il campo «Da» non conterrà più il nome, il numero di telefono o l’indirizzo e-mail del chiamante. Questi dati saranno invece indicati nell’oggetto e continueranno a figurare nel corpo del messaggio. In Teams, la visualizzazione rimane invariata.

| Caratteristica | Oggi | Dal rollout |
|---|---|---|
| Mittente (voicemail personale) | Nome o numero del chiamante | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` oppure dominio personalizzato, se configurato |
| Dominio mittente | `skype.voicemail.microsoft.com` per le chiamate PSTN, dominio tenant per le chiamate da Teams a Teams | Dominio tenant predefinito o dominio personalizzato |
| Oggetto (voicemail personale) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` oppure `Unidentified: +19371817273 - Voicemail (00:22)` |
| Coda di chiamata (voicemail condivisa) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Centralino automatico (voicemail condivisa) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

Per gli utenti, questo comporta due conseguenze visibili: nella posta in arrivo tutte le voicemail appariranno con lo stesso mittente, «Microsoft Teams», e poiché l’indirizzo è un no-reply, la funzione «Rispondi» non raggiungerà più il chiamante.

## Chi è interessato

Sono interessate tutte le organizzazioni che utilizzano Teams Calling con voicemail, ovvero Teams Phone con Calling Plan, Operator Connect o Direct Routing, nonché code di chiamata e centralini automatici con voicemail condivisa. Il messaggio non indica limitazioni relative a licenze o cloud; il rollout è annunciato a livello mondiale.

È necessario intervenire quando qualcosa dipende dall’attuale mittente o oggetto:

- regole di trasporto di Exchange che verificano `skype.voicemail.microsoft.com`, il chiamante come mittente oppure l’oggetto `Voice mail` o `Shared Voicemail`
- voci nelle liste di autorizzazione: Tenant Allow/Block List, autorizzazioni per spoofing, `AllowedSenderDomains` nei criteri antispam
- regole della posta in arrivo di utenti e cassette postali di gruppo delle code di chiamata, che classificano o inoltrano le voicemail in base al mittente o all’oggetto
- flussi Power Automate, sistemi di ticketing e integrazioni CRM che leggono il numero di telefono del chiamante dal mittente
- gateway a monte o a valle, ad esempio con flusso di posta centralizzato in un ambiente ibrido, che contengono regole o eccezioni per il dominio voicemail

## Come verificare se siete interessati

Le seguenti query vengono eseguite in una sessione PowerShell di Exchange Online (`Connect-ExchangeOnline`).

### Cercare le regole di trasporto

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `$muster` | Espressione regolare per l’attuale dominio mittente e gli attuali testi dell’oggetto |
| `Format-List *` | Restituisce tutte le proprietà della regola, per cercare anche condizioni ed eccezioni |
| `Out-String` | Converte l’output in testo sul quale viene applicato `-match` |
| `Select-Object Name, State, Priority` | Mostra solo nome, stato e ordine delle regole trovate |

</details>

Verificate ogni regola trovata per condizioni quali `FromAddressContainsWords`, `SenderDomainIs` o `SubjectContainsWords`.

### Liste di autorizzazione e criteri antispam

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-ListType Sender` | Legge le voci del mittente nella Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtra le voci il cui valore contiene il dominio voicemail |
| `Get-TenantAllowBlockListSpoofItems` | Elenca le autorizzazioni e i blocchi per spoofing |
| `AllowedSenderDomains, AllowedSenders` | Autorizzazioni direttamente nei criteri antispam |

</details>

### Determinare il traffico e-mail effettivo

Il tracciamento dei messaggi mostra se e dove le voicemail vengono recapitate oggi:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-StartDate` / `-EndDate` | Intervallo della ricerca; Exchange Online restituisce al massimo gli ultimi 90 giorni e non più di 10 giorni per query |
| `-ResultSize 5000` | Numero massimo di risultati per query |
| `Where-Object …` | Mantiene le e-mail dal dominio PSTN voicemail o con oggetto voicemail |
| `Group-Object RecipientAddress` | Conta le voicemail per destinatario |

</details>

L’elenco dei destinatari mostra quali cassette postali e cassette postali di gruppo ricevono voicemail. In questi casi vale la pena verificare le regole della posta in arrivo (`Get-InboxRule -Mailbox <Adresse>`) e i flussi collegati.

## Cosa fare

1.  Adeguare le regole all’oggetto anziché al mittente: configurare le nuove condizioni in base ai futuri modelli di oggetto `- Voicemail (` e `- Shared voicemail in `. Finché il rollout non ha raggiunto il proprio tenant, mantenere in parallelo la vecchia e la nuova condizione.

2.  Rimuovere dopo il rollout le autorizzazioni per `skype.voicemail.microsoft.com`. Le e-mail dal proprio dominio tenant non richiedono eccezioni; autorizzare il proprio dominio sarebbe un rischio per la sicurezza, poiché faciliterebbe lo spoofing.

3.  Adeguare le automazioni che leggono il numero del chiamante dal mittente. In futuro il numero sarà nell’oggetto tra parentesi oppure, per i chiamanti sconosciuti, direttamente dopo `Unidentified:`.

4.  Preparare un dominio mittente personalizzato: Microsoft annuncia istruzioni di configurazione per l’invio tramite un dominio personalizzato nel messaggio successivo. Se DMARC per questo dominio è impostato su `p=reject`, SPF e DKIM devono coprire l’invio da Exchange Online, altrimenti le voicemail finiranno in quarantena.

5.  Informare utenti e service desk: mittente «Microsoft Teams», dati del chiamante nell’oggetto, non sarà più possibile rispondere al chiamante via e-mail.

## Fonti

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): testo archiviato del messaggio del Centro messaggi con calendario e confronto tra mittente e oggetto attuali e futuri.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): parametri e limiti del tracciamento dei messaggi in Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): lettura della Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): nozioni di base sulla voicemail di Teams e sul suo recapito via e-mail.
