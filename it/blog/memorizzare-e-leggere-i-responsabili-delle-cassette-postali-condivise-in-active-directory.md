---
title: "Memorizzare e leggere i responsabili delle cassette postali condivise in Active Directory"
navTitle: "Responsabili delle cassette postali condivise"
description: "In caso di messaggio bloccato da una cassetta postale condivisa, le soluzioni DLP devono sapere chi informare. Quali attributi AD sono adatti, quali rappresentano solo diritti di accesso, perché le autorizzazioni di Exchange Online non arrivano nell'AD locale e come un fornitore legge automaticamente l'associazione tramite LDAP o Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min di lettura"
themen:
  - active-directory-entra
  - microsoft-365-exchange
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "memorizzare-e-leggere-i-responsabili-delle-cassette-postali-condivise-in-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/it/blog/memorizzare-e-leggere-i-responsabili-delle-cassette-postali-condivise-in-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:52:27.062Z
translationReview: automatic
---

# Memorizzare e leggere i responsabili delle cassette postali condivise in Active Directory

In un progetto di Data Loss Prevention mi è giunta la seguente richiesta: non appena un messaggio proveniente da una cassetta postale condivisa (Shared Mailbox) viene bloccato, le persone responsabili devono essere avvisate affinché possano intervenire. Il fornitore della soluzione DLP deve leggere automaticamente questa associazione da Active Directory. La domanda è quindi: dove viene indicato nell'AD chi è responsabile di una cassetta postale condivisa?

In breve: l'AD conosce solo i diritti di accesso, non la responsabilità. Alcuni attributi indicano chi può accedere a una cassetta postale o inviare messaggi da essa, e anche questi in modo incompleto non appena la cassetta postale si trova in Exchange Online. Per un'automazione affidabile, la persona responsabile deve essere registrata esplicitamente in un attributo. L'attributo standard `manager` è adatto a questo scopo, viene sincronizzato da Entra Connect e può essere interrogato tramite LDAP e Microsoft Graph.

## L'accesso non è responsabilità

Nell'AD, una cassetta postale condivisa è un oggetto utente disabilitato con attributi Exchange. Chi la utilizza risulta dalle autorizzazioni: accesso completo (FullAccess), invio come (Send As) e invio per conto di (Send on Behalf). Questi elenchi contengono spesso dieci o venti persone, oltre a delegati, apprendisti e account di servizio. Una notifica DLP inviata a tutti gli autorizzati raggiunge molte persone, senza che nessuna si senta responsabile.

La questione della responsabilità è organizzativa: una persona, o un piccolo gruppo, decide se un messaggio bloccato era legittimo e gestisce le autorizzazioni. Questa informazione esiste nell'AD solo se qualcuno la inserisce.

## Quali attributi AD possono essere presi in considerazione

| Attributo | Contenuto | Idoneità per i responsabili |
|---|---|---|
| `manager` | DN di una persona, a valore singolo, parte delle classi di persone e quindi disponibile nell'oggetto utente della cassetta postale | Adatto. Viene sincronizzato da Entra Connect in Entra ID ed è leggibile tramite Graph |
| `managedBy` | DN di un gestore, a valore singolo | Non disponibile nell'oggetto utente. Secondo lo schema, è definito tra l'altro per gruppi, contatti, computer e OU, non per la classe User |
| `msExchDelegateListLink` | DN delle persone che hanno accesso completo con Automapping, a valori multipli; riferimento inverso `msExchDelegateListLinkBL` nell'utente | Solo accesso, e incompleto: mancano l'accesso completo senza Automapping, i diritti tramite gruppi e i diritti assegnati in Exchange Online |
| `publicDelegates` | DN delle persone con invio per conto di, a valori multipli | Solo accesso. Negli ambienti ibridi viene riscritto da Entra Connect |
| Elenco autorizzazioni (`nTSecurityDescriptor`) | Diritto esteso «Send As» nell'oggetto utente | Solo invio come assegnato localmente. I diritti assegnati in Exchange Online non sono presenti |
| `msExchMailboxSecurityDescriptor` | Autorizzazioni delle cassette postali locali, binario | Solo accesso, solo per cassette postali su server Exchange locali, difficile da valutare per i fornitori |
| `extensionAttribute1` fino a `extensionAttribute15` | Campi di testo liberi | Possibile, ma solo come testo; nessun riferimento a un oggetto, nessun aggiornamento automatico in caso di rinomina o uscita |

In pratica, l'indicazione che l'associazione sia presente «direttamente nell'AD» si riferisce di solito a `msExchDelegateListLink`: in ADUC o ADSI Edit vi si vedono effettivamente delle persone. Tuttavia, l'attributo nasce solo come effetto collaterale dell'Automapping di Outlook e non è adatto come fonte dati per le responsabilità.

## La posizione della cassetta postale determina la fonte dati

Se le cassette postali si trovano in Exchange Online e gli utenti vengono sincronizzati dall'AD locale (ibrido), vale quanto segue: le autorizzazioni assegnate in Exchange Online rimangono in Exchange Online. Per Exchange Hybrid, Entra Connect riscrive solo un piccolo elenco di attributi, tra cui `publicDelegates` e `proxyAddresses`, ma né `msExchDelegateListLink` né i diritti Send As. In questo caso, un AD locale mostra lo stato precedente alla migrazione più quanto è stato successivamente aggiornato localmente.

Per Exchange Online, lo stato effettivo delle autorizzazioni è fornito solo da Exchange Online stesso (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Il fornitore avrebbe bisogno dell'accesso a Exchange Online PowerShell, che risulta sproporzionato per una semplice associazione. Un attributo gestito nell'oggetto utente è più semplice e funziona indipendentemente da dove si trova la cassetta postale.

## Raccomandazione: `manager` come attributo del responsabile

Per ogni cassetta postale condivisa, la persona responsabile viene inserita nell'attributo `manager`. Motivi:

- L'attributo fa parte dello schema standard, quindi non è necessaria un'estensione dello schema.
- Contiene un riferimento (DN) anziché testo. Se la persona viene rinominata o spostata, il riferimento resta valido.
- Entra Connect sincronizza `manager` in Entra ID. Il fornitore può leggere a scelta tramite LDAP nell'AD locale o tramite Microsoft Graph.
- Tramite il riferimento inverso `directReports`, ogni persona responsabile, e ogni report, può vedere di quali cassette postali è responsabile.

La limitazione: `manager` è a valore singolo. Quando una delega è indispensabile, esistono due possibilità. Possibilità 1: un gruppo per cassetta postale, i cui membri sono i responsabili; il suo nome segue una convenzione, ad esempio `SM-Verantwortlich-buchhaltung`, oppure è riportato in un `extensionAttribute`. Possibilità 2: la delega non viene ricavata dal fornitore dall'AD, ma dal `manager` della persona responsabile, con escalation alla persona superiore. La possibilità 2 non richiede oggetti aggiuntivi ed è sufficiente nella maggior parte delle organizzazioni.

L'attributo viene impostato solo nel sistema master: nell'AD locale per gli oggetti sincronizzati, in Entra ID per gli oggetti esclusivamente cloud.

## Rilevare lo stato attuale: chi ha oggi accesso?

Per il popolamento iniziale è utile un elenco degli attuali autorizzati. La persona responsabile è solitamente tra questi; il reparto competente conferma la scelta. Per le cassette postali in Exchange Online:

```powershell
Connect-ExchangeOnline

$postfaecher = Get-EXOMailbox -RecipientTypeDetails SharedMailbox -ResultSize Unlimited

$bestand = foreach ($pf in $postfaecher) {
    $voll = Get-EXOMailboxPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'FullAccess' -and
            -not $_.IsInherited -and
            $_.User -notlike 'NT AUTHORITY\*'
        }
    $sendas = Get-EXORecipientPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'SendAs' -and
            $_.Trustee -notlike 'NT AUTHORITY\*'
        }
    [pscustomobject]@{
        Postfach  = $pf.PrimarySmtpAddress
        Vollzugriff = ($voll.User -join '; ')
        SendenAls = ($sendas.Trustee -join '; ')
    }
}

$bestand | Export-Csv -Path .\shared-mailbox-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Solo cassette postali condivise, nessuna cassetta postale utente, di sala o di apparecchiatura |
| `-ResultSize Unlimited` | Tutte le cassette postali anziché il limite standard di 1000 |
| `-Identity` | Cassetta postale di cui vengono interrogate le autorizzazioni |
| `AccessRights -contains 'FullAccess'` | Filtra per accesso completo; le altre autorizzazioni della cassetta postale vengono escluse |
| `-not $_.IsInherited` | Rimuove le voci ereditate, come amministratori e account di sistema |
| `User -notlike 'NT AUTHORITY\*'` | Rimuove la voce `NT AUTHORITY\SELF` della cassetta postale stessa |
| `-NoTypeInformation` | Nessuna riga del tipo all'inizio del file CSV |
| `-Encoding UTF8` | I caratteri accentati nei nomi vengono mantenuti |

</details>

Se è autorizzato un gruppo, il suo nome appare nell'elenco; i membri possono essere risolti con `Get-DistributionGroupMember` o `Get-ADGroupMember`.

## Impostare i responsabili

L'associazione confermata viene preparata come CSV con le colonne `Postfach` e `Verantwortlich` (entrambe indirizzi e-mail) e impostata nell'AD locale:

```powershell
Import-Module ActiveDirectory

$zuordnung = Import-Csv -Path .\shared-mailbox-verantwortliche.csv

foreach ($eintrag in $zuordnung) {
    $postfach = Get-ADUser -Filter "mail -eq '$($eintrag.Postfach)'"
    $person = Get-ADUser -Filter "mail -eq '$($eintrag.Verantwortlich)'"
    Set-ADUser -Identity $postfach -Manager $person
}
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Filter "mail -eq '…'"` | Cerca l'oggetto in base all'indirizzo e-mail principale |
| `-Identity` | Oggetto utente della cassetta postale condivisa |
| `-Manager` | Scrive il DN della persona responsabile nell'attributo `manager` |

</details>

Per gli oggetti esclusivamente cloud senza AD locale, `Set-MgUserManagerByRef` dell'SDK PowerShell di Microsoft Graph imposta lo stesso valore in Entra ID.

## Lettura da parte del fornitore: LDAP

Le cassette postali condivise possono essere riconosciute nell'AD tramite `msExchRecipientTypeDetails`: `4` indica una cassetta postale condivisa su un server Exchange locale, `34359738368` una cassetta postale condivisa in Exchange Online (RemoteSharedMailbox). La query che il fornitore può integrare nella propria soluzione:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

Il valore in `manager` è un DN. L'indirizzo e-mail della persona responsabile si ottiene con una seconda query su questo DN con l'attributo `mail`. È sufficiente un account di lettura senza ulteriori diritti, poiché `manager` e `mail` sono leggibili dagli utenti autenticati. La connessione deve avvenire tramite LDAPS, porta 636.

Per verificare l'associazione in PowerShell:

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

$verantwortlich = @{
    Name       = 'Verantwortlich'
    Expression = {
        if ($_.manager) {
            (Get-ADUser -Identity $_.manager -Properties mail).mail
        }
    }
}

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Select-Object mail, $verantwortlich |
    Sort-Object mail
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-LDAPFilter` | Filtro LDAP sui due tipi di destinatario per cassette postali condivise |
| `-Properties mail, manager` | Carica i due attributi che `Get-ADUser` non fornisce per impostazione predefinita |
| `Expression = { … }` | Risolve il DN in `manager` nell'indirizzo e-mail della persona responsabile |

</details>

Per l'associazione dei messaggi, il fornitore deve considerare tutti gli indirizzi in `proxyAddresses`, non solo `mail`: un messaggio può essere inviato con un indirizzo secondario della cassetta postale.

## Lettura da parte del fornitore: Microsoft Graph

Se il fornitore non dispone di accesso LDAP alla rete interna, Microsoft Graph è un'alternativa adatta. Le cassette postali condivise esistono in Entra ID come oggetti utente disabilitati e `manager` vi è presente grazie alla sincronizzazione:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

A tale scopo, la registrazione dell'app del fornitore richiede l'autorizzazione applicativa `User.Read.All`. Graph non fornisce direttamente un elenco di tutte le cassette postali condivise, poiché Entra ID non conosce il tipo di cassetta postale. Il fornitore interroga quindi in modo mirato l'indirizzo del mittente del messaggio bloccato, il che è comunque sufficiente per il caso d'uso DLP.

## Integrazione nella notifica DLP

Il modo in cui la notifica viene attivata tecnicamente dipende dalla soluzione DLP. Per Microsoft Purview DLP, va osservato che le notifiche possono essere inviate anche a persone indicate in modo fisso, ma solo a singoli destinatari, non a gruppi o liste di distribuzione. Un'associazione dinamica «cassetta postale a persona responsabile» in genere non è prevista. Con poche cassette postali è ipotizzabile una regola per ciascuna; con molte cassette postali è necessario un processo a valle che recuperi gli incidenti, risolva l'indirizzo del mittente tramite LDAP o Graph e informi la persona responsabile.

Per una soluzione esterna vale la stessa logica: determinare l'indirizzo del mittente del messaggio bloccato, trovare la cassetta postale tramite `mail` o `proxyAddresses`, risolvere `manager`, inviare la notifica. Se manca `manager`, la soluzione dovrebbe segnalare a un indirizzo collettivo definito, ad esempio Service Desk o protezione dei dati, anziché scartare la notifica.

## Gestione operativa

Per mantenere aggiornata l'associazione, nella pratica sono sufficienti due regole:

- Le nuove cassette postali condivise vengono create solo con una persona responsabile registrata; la richiesta contiene il relativo campo.
- Un report mensile elenca le cassette postali senza `manager` e quelle la cui persona responsabile è disabilitata.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

In caso di uscita di una persona, `directReports` nel suo oggetto mostra quali cassette postali necessitano di una nuova persona responsabile. Questo controllo può essere integrato nel processo di uscita.

## Fonti

1.  [Manager attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): attributo DN a valore singolo delle classi di persone, riferimento inverso `directReports`.

2.  [Managed-By attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): elenco delle classi per le quali è definito `managedBy`; la classe User non ne fa parte.

3.  [Microsoft Entra Connect Sync: Attributes synchronized to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): attributi sincronizzati, incluso `manager`, nonché l'elenco degli attributi di writeback di Exchange Hybrid.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): query delle autorizzazioni delle cassette postali in Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): query delle autorizzazioni Send As in Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): articolo di supporto Microsoft sulla relazione tra accesso completo e `msExchDelegateListLink` in Exchange 2016 e 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): endpoint Graph e autorizzazioni necessarie per leggere il manager.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): destinatari delle notifiche DLP in Microsoft Purview; solo singoli destinatari, nessun gruppo.
