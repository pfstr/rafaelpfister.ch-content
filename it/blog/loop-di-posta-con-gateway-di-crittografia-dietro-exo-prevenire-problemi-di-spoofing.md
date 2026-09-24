---
title: "Loop di posta con gateway di crittografia dietro EXO - prevenire problemi di spoofing"
navTitle: "Loop del gateway"
description: "Se il gateway di crittografia (nell'esempio HIN) si trova dietro Exchange Online, ogni messaggio in arrivo raggiunge EOP una seconda volta: con un dominio mittente esterno, una firma DKIM non valida e l'IP del gateway come origine. Il risultato è un verdetto di spoofing e posta indesiderata. Quattro impostazioni lo impediscono senza disattivare il filtraggio tramite SCL -1: record PTR, Enhanced Filtering disattivato, eccezione di spoofing per l'infrastruttura del gateway e CloudServicesMailEnabled su entrambi i connettori."
date: "2026-09-23"
kategorie: "Flusso di posta e SMTP"
timeToRead: "9 min di lettura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
  - hin-gateway
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "loop-di-posta-con-gateway-di-crittografia-dietro-exo-prevenire-problemi-di-spoofing"
translationId: "article-b773c8d303aee87a"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
translationOf: verschluesselungsgateway-hinter-exchange-online
url: https://rafaelpfister.ch/it/blog/loop-di-posta-con-gateway-di-crittografia-dietro-exo-prevenire-problemi-di-spoofing
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:35:18.388Z
translationReview: automatic
---

# Loop di posta con gateway di crittografia dietro EXO - prevenire problemi di spoofing

Molte organizzazioni del settore sanitario svizzero utilizzano un gateway HIN, altre un SEPPmail o totemomail per S/MIME e PGP. Se il record MX punta a Microsoft e il gateway si trova dietro Exchange Online, ogni messaggio in arrivo attraversa il filtraggio due volte. Al secondo passaggio, EOP vede un messaggio con un dominio mittente esterno, una firma DKIM non valida e l'IP del gateway come origine. Dal punto di vista del filtro, questo è il modello di una falsificazione del mittente e la posta legittima finisce nella cartella della posta indesiderata. Ho configurato questa architettura più volte e qui descrivo le quattro impostazioni che consentono al loop di funzionare senza verdetto di spoofing e perché ciascuna di esse è necessaria. L'esempio utilizza il gateway HIN; il meccanismo è identico per qualsiasi altro gateway nella stessa posizione.

## L'architettura

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

Una regola di trasporto inoltra i messaggi in arrivo al gateway tramite un connettore in uscita. Il gateway decrittografa, verifica le firme e riconsegna il messaggio a Exchange Online tramite un connettore in entrata. Un campo di intestazione impostato dal gateway impedisce che la regola venga applicata nuovamente.

L'architettura ha valide ragioni: Microsoft filtra per primo, il gateway riceve solo posta già verificata e l'operatività non deve esporre un proprio MX a Internet. Il prezzo è il secondo passaggio, che non può essere disattivato, ma solo configurato correttamente.

## Perché SCL -1 è la risposta sbagliata

Il rimedio diffuso è una regola di trasporto sul percorso di ritorno che imposta lo Spam Confidence Level su `-1`. In questo modo Exchange Online salta completamente la verifica del contenuto nel secondo passaggio. Funziona subito e presenta due svantaggi: il secondo passaggio non serve più a nulla e la regola dipende da una condizione (connettore o IP) che cambia a ogni modifica. Microsoft descrive inoltre SCL -1 come input per il filtraggio, non come decisione definitiva; il valore stampato può differire. Le quattro impostazioni seguenti risolvono il problema nel punto in cui nasce: nella valutazione dell'infrastruttura di consegna.

## Le quattro impostazioni

1. Pubblicare nel DNS pubblico un record PTR per l'IP pubblico del gateway HIN. Senza questa voce l'intero percorso non funziona.
2. Disattivare completamente Enhanced Filtering sul connettore in entrata tramite il quale HIN consegna a Exchange Online.
3. Creare un'eccezione di spoofing per l'infrastruttura HIN che effettua la consegna nella Tenant Allow/Block List.
4. Attivare `CloudServicesMailEnabled` sul connettore in uscita verso HIN e sul connettore in entrata da HIN.

Per il punto 2, verificare innanzitutto quali connettori sono interessati:

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

Quindi disattivarlo sul connettore HIN:

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Parametro | Effetto |
|---|---|
| `EFSkipLastIP = $false` | L'ultimo hop non viene più saltato automaticamente. Se contemporaneamente non è presente alcun IP in `EFSkipIPs`, Enhanced Filtering è disattivato sul connettore. |
| `EFSkipIPs = $null` | Svuota l'elenco degli indirizzi IP da saltare. |
| `EFUsers = $null` | Rimuove la limitazione a singoli destinatari; senza Enhanced Filtering attivo il valore non ha significato, ma rimane così inequivocabile. |
| `EFTestMode` | Solo nella query: indica se il connettore è in modalità di test. Microsoft indica il parametro come interno, ma è leggibile. |

</details>

Punto 3, l'eccezione di spoofing:

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Parametro | Effetto |
|---|---|
| `Identity = "Default"` | L'elenco stesso; ne esiste uno solo. |
| `Action = "Allow"` | Consente la combinazione. `Block` la classifica invece come phishing. |
| `SpoofedUser = "*"` | L'indirizzo visibile nel campo Da. Il carattere jolly rappresenta qualsiasi mittente. È consentito un carattere jolly su un lato della coppia, non su entrambi. |
| `SendingInfrastructure` | L'origine: il dominio del record PTR dell'IP che effettua la consegna (punto 1). Senza record PTR, l'elenco accetta solo `<IP>/24`. |
| `SpoofType = "External"` | Vale per domini mittente esterni. `Internal` copre i propri domini accettati e richiede una seconda voce. |

</details>

Punto 4, le intestazioni Cross-Premises:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

Spiego sotto, al punto 4, perché `TreatMessagesAsInternal` è incluso nello stesso comando per il connettore in entrata.

## Sul punto 1: record PTR

EOP identifica l'infrastruttura di consegna tramite la ricerca inversa dell'IP di origine. Il valore PTR compare nell'intestazione `Authentication-Results` come sending infrastructure, ed è esattamente il valore a cui si riferisce l'eccezione di spoofing del punto 3. Se manca il record PTR, Exchange Online valuta ogni messaggio sul percorso di ritorno con `PTR:InfoDomainNonexistent`, e l'eccezione deve riferirsi a un intero `/24` anziché a un nome. Inoltre, l'assenza di una voce DNS inversa è da tempo un segnale negativo consolidato in qualsiasi filtro antispam. La voce dovrebbe risolvere in avanti nuovamente sullo stesso IP.

Con HIN, l'IP che effettua la consegna è quello del mail gateway HIN tramite il quale i messaggi ritornano a Exchange Online. Chi gestisce il record PTR per questo IP dipende dal modello operativo; la responsabilità va chiarita prima del passaggio.

## Sul punto 2: disattivare Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) è concepito per l'architettura inversa: gateway davanti a Microsoft 365, MX sul gateway. In quel caso EOP salta l'ultimo hop e valuta la vera origine del messaggio.

Nella configurazione a loop questo non funziona. Microsoft precisa nella documentazione che Enhanced Filtering non è pensato per servizi che elaborano la posta dopo Microsoft 365 e indica il routing non lineare (Internet, Microsoft 365, sistema esterno, Microsoft 365) come non supportato. Come conseguenza, la documentazione cita esattamente il sintomo riscontrato nella pratica: Microsoft 365 verifica nuovamente la posta di ritorno, assegna un valore `compauth` e il messaggio può essere classificato come spam.

Se Enhanced Filtering resta attivo, EOP salta l'IP HIN e valuta l'IP precedente. Nel loop, questo è o un IP Microsoft del primo passaggio oppure il mittente esterno originale. Entrambe le situazioni sono errate:

* La valutazione si basa su un IP che, nel secondo passaggio, non consente più alcuna conclusione sul messaggio.
* L'eccezione di spoofing del punto 3 non viene mai applicata perché è collegata all'infrastruttura HIN, che EOP ha appena saltato.

Perciò Enhanced Filtering deve essere disattivato su questo connettore: solo allora HIN è visibile e indirizzabile come infrastruttura di consegna.

## Sul punto 3: eccezione di spoofing

Una voce di spoofing è sempre una coppia composta da Spoofed user (l'indirizzo Da o il relativo dominio) e Sending infrastructure (l'origine). Viene consentita esclusivamente questa combinazione.

`SpoofedUser = "*"` insieme all'infrastruttura HIN significa quindi: qualsiasi indirizzo Da può consegnare tramite HIN senza che scatti il verdetto di spoofing. Qualsiasi altra origine che utilizza gli stessi mittenti continua a essere verificata. L'eccezione non è un bypass del filtro antispam: la verifica di spam, contenuto e minacce continua invariata nel secondo passaggio. Un messaggio può quindi ancora essere scartato per il suo contenuto, ma non viene più considerato una falsificazione.

Nel comando sopra compare intenzionalmente solo `SpoofType = "External"`. Le proprie e-mail, ossia i messaggi con mittente appartenente ai propri domini accettati, non dovrebbero tornare tramite il gateway. Vengono inviate da sistemi interni o da Exchange on-premises direttamente a Exchange Online, senza passare da HIN. Se questo avvenga effettivamente nel proprio ambiente è mostrato dalla valutazione di Spoof Intelligence:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

Se vi compaiono domini propri con l'infrastruttura HIN e `SpoofType Internal`, è necessaria una seconda voce con `SpoofType = "Internal"`, o meglio ancora: individuare la causa per cui la posta interna passa dal gateway.

Le voci di spoofing non scadono automaticamente. Se il gateway viene rimosso o la sua infrastruttura cambia, la voce deve essere eliminata.

## Sul punto 4: CloudServicesMailEnabled

Questo è l'unico punto che Microsoft non documenta esplicitamente per questo scenario. È una mia deduzione dal comportamento documentato del parametro: preserva le intestazioni Cross-Premises lungo il loop.

Il parametro controlla il trattamento delle intestazioni interne `X-MS-Exchange-Organization-*`. Sul connettore in uscita vengono trasformate in `X-MS-Exchange-CrossPremises-*` e sopravvivono così al percorso attraverso HIN. Sul connettore in entrata vengono nuovamente riscritte come `X-MS-Exchange-Organization-*`, sostituendo nel contempo intestazioni con lo stesso nome già presenti nel messaggio. Se il parametro è impostato su `$false`, il connettore rimuove queste intestazioni.

In pratica significa che quanto Exchange Online ha determinato nel primo passaggio, tra cui lo stato di autenticazione e la marcatura interna, sopravvive al loop anziché essere rimosso al rientro. Nelle intestazioni del messaggio consegnato compare quindi `X-CrossPremisesHeadersPromoted`, e i risultati delle verifiche originali restano disponibili in `Authentication-Results-Original`. Questo da solo non impedisce un verdetto di spoofing (a tale scopo servono i punti da 1 a 3), ma preserva la valutazione del primo passaggio.

Occorre tenere presenti tre aspetti:

* Se i connettori sono stati creati da Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), una successiva esecuzione di HCW sovrascrive l'impostazione. I connettori HIN dovrebbero quindi essere connettori propri, creati manualmente.
* Sul connettore in entrata, `CloudServicesMailEnabled $true` e `TreatMessagesAsInternal $true` si escludono a vicenda. Se `TreatMessagesAsInternal` è già impostato su `$true`, Exchange Online rifiuta il comando. Per questo entrambi i parametri devono figurare nella stessa chiamata `Set-InboundConnector`, come mostrato sopra.
* Microsoft raccomanda di impostare il parametro solo su indicazione del supporto o della documentazione del prodotto. Per il loop non esiste tale documentazione; la decisione spetta a voi e va documentata.

## Proteggere la consegna

Poiché, dopo il punto 4, Exchange Online accetta le intestazioni dal percorso di ritorno e, dopo il punto 3, accetta qualsiasi indirizzo Da proveniente dall'infrastruttura HIN, il connettore in entrata deve essere limitato in modo che solo HIN possa consegnare tramite esso. Per un connettore di tipo `Partner`, si tratta di `RestrictDomainsToCertificate` con `TlsSenderCertificateName` oppure di `RestrictDomainsToIPAddresses` con `SenderIPAddresses`, in entrambi i casi insieme a `RequireTls`. Senza questa limitazione, qualsiasi origine che raggiunga il connettore potrebbe consegnare con mittenti arbitrari e intestazioni dell'organizzazione accettate.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Verifica

Dopo il passaggio, inviare tramite il gateway un messaggio di prova da un dominio esterno con policy DMARC e controllare le intestazioni del messaggio consegnato. Nell'intestazione `Authentication-Results` del secondo passaggio, il dominio HIN dovrebbe comparire come sending infrastructure e `compauth` dovrebbe essere impostato su `pass` con una motivazione nell'ambito delle voci Tenant Allow, non più su `fail reason=001`. `X-CrossPremisesHeadersPromoted` e `Authentication-Results-Original` mostrano che il punto 4 è efficace. L'[analizzatore delle intestazioni di posta](/tools/header-analyzer) su questo sito rappresenta entrambi i passaggi come diagramma di flusso e evidenzia i salti tra Exchange Online e il gateway.

## Fonti

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): delimitazione tra routing lineare e non lineare, indicazione sui servizi dietro Microsoft 365, conseguenza `compauth` e classificazione come spam, SCL -1 come input anziché decisione, parametri PowerShell `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): comportamento di `CloudServicesMailEnabled` (trasformazione e promozione delle intestazioni Cross-Premises, rimozione con `$false`), esclusione reciproca con `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` e `RestrictDomainsToIPAddresses` per i connettori partner.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` sul connettore in uscita.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): sintassi delle voci di spoofing, regole dei caratteri jolly, determinazione dell'infrastruttura di invio tramite record PTR o `/24`, copertura dei verdetti di spoofing condizionati da DMARC.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): parametri `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): valutazione delle coppie di spoofing rilevate negli ultimi sette giorni.
