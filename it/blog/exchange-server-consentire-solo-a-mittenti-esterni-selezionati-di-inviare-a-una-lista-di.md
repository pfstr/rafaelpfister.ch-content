---
title: "Exchange Server: consentire solo a mittenti esterni selezionati di inviare a una lista di distribuzione"
navTitle: "Lista di distribuzione: mittenti esterni"
description: "Una lista di distribuzione in Exchange Server può essere configurata in modo che solo determinati mittenti esterni possano raggiungerla. Due metodi con PowerShell: restrizione di consegna con contatti e-mail o regola di trasporto. Include i limiti della verifica del mittente e come DMARC li colma."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min di lettura"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-message-tracking-und-receive-connectoren-analysieren
slug: "exchange-server-consentire-solo-a-mittenti-esterni-selezionati-di-inviare-a-una-lista-di"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
url: https://rafaelpfister.ch/it/blog/exchange-server-consentire-solo-a-mittenti-esterni-selezionati-di-inviare-a-una-lista-di
translationSourceHash: cff4b579945841f2469f23b5e44d834f6be7528cf9817f9d7ade0fd613f1c582
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:22:26.006Z
translationReview: automatic
---

# Exchange Server: consentire solo a mittenti esterni selezionati di inviare a una lista di distribuzione

Per impostazione predefinita, una lista di distribuzione in Exchange Server accetta messaggi solo da mittenti autenticati, quindi interni. Se anche un partner, un fornitore o un cliente deve poter raggiungere la lista senza renderla accessibile a tutta Internet, esistono due possibilità: una restrizione di consegna nel gruppo stesso oppure una regola di trasporto. Entrambe funzionano in Exchange Server 2016, 2019 e Subscription Edition.

## Situazione iniziale: RequireSenderAuthenticationEnabled

La proprietà `RequireSenderAuthenticationEnabled` determina se un gruppo accetta messaggi esterni. Se è impostata su `$true` (impostazione predefinita per i nuovi gruppi di distribuzione), Exchange rifiuta ogni messaggio proveniente da un mittente non autenticato. Entrambe le varianti seguenti presuppongono quindi che questo controllo sia disattivato:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Identity` | Il gruppo di distribuzione (nome, alias o indirizzo e-mail). |
| `-RequireSenderAuthenticationEnabled $false` | Il gruppo accetta anche messaggi da mittenti esterni non autenticati. |

</details>

Senza ulteriori restrizioni, dopo questo passaggio qualsiasi mittente esterno può scrivere alla lista. La limitazione a singoli mittenti viene configurata nel passaggio successivo, idealmente nella stessa finestra di manutenzione.

## Variante 1: restrizione di consegna con contatti e-mail

L'elenco dei mittenti autorizzati (`AcceptMessagesOnlyFromSendersOrMembers`) accetta solo oggetti della propria directory: cassette postali, utenti di posta elettronica, contatti e-mail e gruppi. Non è possibile inserirvi un semplice indirizzo SMTP. Per ogni mittente esterno, create quindi un contatto e-mail. Per evitare che il contatto appaia nell'elenco indirizzi globale, nascondetelo:

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Name` | Nome visualizzato del contatto in Active Directory. Un prefisso uniforme come «Esterno» facilita la gestione successiva. |
| `ExternalEmailAddress` | L'indirizzo esterno dal quale il mittente scrive. |
| `OrganizationalUnit` | OU di destinazione in Active Directory. Se non specificato, il contatto viene inserito nel contenitore predefinito `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Nasconde il contatto dall'elenco indirizzi globale e da tutti gli elenchi indirizzi. |

</details>

Successivamente, aggiungete il contatto come mittente autorizzato. La tabella hash con `Add` integra l'elenco esistente invece di sovrascriverlo:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Elenco dei mittenti autorizzati a raggiungere il gruppo. Sono possibili singoli mittenti e gruppi; per i gruppi, tutti i membri possono inviare. |
| `@{ Add = ... }` | Aggiunge voci senza rimuovere quelle esistenti. Con `@{ Remove = ... }` le voci vengono rimosse; un valore senza tabella hash sostituisce l'intero elenco. |

</details>

Non appena questo elenco contiene almeno una voce, Exchange rifiuta **tutti gli altri mittenti**, inclusi quelli interni. Se i collaboratori devono continuare a poter scrivere alla lista, aggiungete anche un gruppo adeguato, ad esempio il gruppo di distribuzione stesso (in tal caso tutti i membri possono inviare) oppure un gruppo con tutti i collaboratori. I mittenti rifiutati ricevono un rapporto di mancato recapito con lo stato `5.7.1`.

Per verificare la configurazione corrente:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` è un parametro collettivo: Exchange distribuisce le voci tra le proprietà `AcceptMessagesOnlyFrom` (singoli mittenti) e `AcceptMessagesOnlyFromDLMembers` (gruppi). Per questo motivo la query mostra queste due proprietà. La stessa impostazione è disponibile nell'Exchange Admin Center, nelle proprietà del gruppo, in Gestione recapito (Delivery management).

Questa variante è adatta se si aggiungono solo pochi mittenti esterni e l'elenco dei mittenti autorizzati deve essere visibile direttamente nell'oggetto gruppo. Lo svantaggio è l'onere di gestione: ogni nuovo mittente richiede un contatto in Active Directory.

## Variante 2: regola di trasporto senza contatti

Se internamente tutti devono continuare a poter scrivere alla lista e solo i mittenti esterni devono essere limitati, una regola di trasporto è più semplice. Non richiede contatti e può essere applicata anche a interi domini. Anche in questo caso `RequireSenderAuthenticationEnabled` deve essere impostata su `$false`.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Name` | Nome della regola nell'elenco delle regole di trasporto. |
| `SentTo` | Condizione: il messaggio è indirizzato a questo gruppo. |
| `FromScope NotInOrganization` | Condizione: il mittente non appartiene alla propria organizzazione. I messaggi interni non vengono quindi interessati. |
| `ExceptIfFromAddressMatchesPatterns` | Eccezione: indirizzi dei mittenti che corrispondono a una delle espressioni regolari. `^…$` delimita un singolo indirizzo, `@domain$` autorizza un intero dominio. Eseguite l'escape dei punti con `\.`. |
| `RejectMessageReasonText` | Testo nel rapporto di mancato recapito inviato al mittente rifiutato. |
| `RejectMessageEnhancedStatusCode` | Codice di stato esteso nel rapporto di mancato recapito; `5.7.1` indica un'autorizzazione mancante. |

</details>

Senza la delimitazione con `^` e `$`, un modello come `max\.muster@partner\.example` corrisponderebbe anche a `max.muster@partner.example.net`. Aggiungete nuovi mittenti con `Set-TransportRule` e l'elenco completo aggiornato dei modelli; il parametro sostituisce l'elenco esistente.

## Limite di entrambe le varianti: l'indirizzo del mittente è falsificabile

Entrambe le varianti verificano solo l'indirizzo del mittente. Exchange non stabilisce se il messaggio proviene effettivamente dal partner autorizzato. Chiunque conosca l'indirizzo di un mittente autorizzato può falsificarlo e raggiungere così la lista. Per una lista con partecipanti interni, questo rappresenta un punto di ingresso per il phishing che, inoltre, dà l'impressione di provenire da un indirizzo noto.

SPF, DKIM e DMARC colmano questa lacuna. Se il dominio del partner dispone di una policy DMARC con `p=reject` o `p=quarantine` e il vostro gateway e-mail applica tale policy, i messaggi falsificati di questo dominio non raggiungono nemmeno l'Exchange Server. Per i domini senza policy DMARC o con `p=none` questa protezione manca.

Se il gateway scrive il risultato della verifica nell'header `Authentication-Results`, una seconda regola di trasporto può rifiutare ulteriormente i mittenti autorizzati quando DMARC non è superato:

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `FromAddressMatchesPatterns` | Condizione: la regola si applica solo ai mittenti autorizzati, con gli stessi modelli della prima regola. |
| `ExceptIfHeaderMatchesMessageHeader` | Nome dell'header verificato dall'eccezione. |
| `ExceptIfHeaderMatchesPatterns` | Eccezione: se l'header contiene `dmarc=pass`, il messaggio viene recapitato. Tutti gli altri messaggi dei mittenti autorizzati vengono rifiutati. |

</details>

Questa regola funziona in modo affidabile solo se il gateway rimuove o sovrascrive gli header `Authentication-Results` in arrivo da Internet. Altrimenti, un mittente può aggiungere autonomamente l'header con `dmarc=pass`. La documentazione del gateway descrive quali header imposta e se rimuove gli header estranei; Exchange Server non valuta DMARC autonomamente.

## Quale variante scegliere

La restrizione di consegna con contatti è adatta a liste con una cerchia piccola e fissa di mittenti esterni e, se desiderato, limita anche i mittenti interni. La regola di trasporto è adatta quando internamente tutti possono inviare, quando devono essere autorizzati interi domini di partner o quando non si vogliono creare contatti in Active Directory. Indipendentemente dalla variante, verificate la policy DMARC per ogni dominio autorizzato e documentate chi gestisce l'elenco dei mittenti autorizzati.

## Fonti

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): parametri AcceptMessagesOnlyFromSendersOrMembers e RequireSenderAuthenticationEnabled, tipi di oggetti consentiti.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): come Categorizer valuta le restrizioni di consegna e l'autenticazione del mittente.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): creazione di contatti e-mail per indirizzi esterni.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): condizioni, eccezioni e azioni per le regole di trasporto, inclusi FromScope e HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): specifica delle policy DMARC p=none, quarantine e reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): struttura dell'header Authentication-Results e indicazione che i sistemi riceventi dovrebbero rimuovere le istanze estranee.
