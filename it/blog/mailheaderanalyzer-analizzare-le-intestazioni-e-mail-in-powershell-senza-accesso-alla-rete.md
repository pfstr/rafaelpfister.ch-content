---
title: "MailHeaderAnalyzer: analizzare le intestazioni e-mail in PowerShell senza accesso alla rete"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Riferimento al modulo PowerShell MailHeaderAnalyzer: panoramica dei parametri, sintassi, descrizione, esempi e proprietà dei parametri di Get-MailHeaderAnalysis e ConvertTo-MailHeaderReport, nonché l'oggetto di output con catena di consegna, risultati di autenticazione, classificazione di Exchange Online e tutti i codici di rilevamento."
date: "2026-09-24"
kategorie: "SMTP e flusso della posta"
timeToRead: "16 min di lettura"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analizzare-le-intestazioni-e-mail-in-powershell-senza-accesso-alla-rete"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
translationSourceHash: 41c3cb7c83800c0cc30f797ad6b2d9ec480d6c32bc054d2e01bd037b3dcd6c9b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-27T09:26:18.887Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/mailheaderanalyzer-analizzare-le-intestazioni-e-mail-in-powershell-senza-accesso-alla-rete
---

# MailHeaderAnalyzer: analizzare le intestazioni e-mail in PowerShell senza accesso alla rete

MailHeaderAnalyzer è un modulo PowerShell con due cmdlet. `Get-MailHeaderAnalysis` analizza l'intestazione di un'e-mail: catena di consegna con ritardi e dati TLS, risultati SPF, DKIM, DMARC e ARC inclusa la verifica dell'origine rispetto all'authserv-id del gateway, allineamento DMARC, classificazione ibrida di Exchange Online, valutazioni di Microsoft Defender, SpamAssassin e Rspamd, nonché anomalie quali righe `From` duplicate o caratteri di controllo Unicode. `ConvertTo-MailHeaderReport` genera un report per i ticket. Il modulo opera interamente offline: nessuna query DNS, nessuna connessione HTTP. È la versione da riga di comando dell'[analizzatore delle intestazioni su questo sito](/tools/header-analyzer) e utilizza la stessa logica di analisi.

| | |
|---|---|
| Modulo | [MailHeaderAnalyzer nella PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Codice sorgente | [pfstr/MailHeaderAnalyzer su GitHub](https://github.com/pfstr/MailHeaderAnalyzer), licenza MIT |
| Si applica a | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlet | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Panoramica dei parametri

| Cmdlet | Parametro | Tipo | Obbligatorio | Pipeline | Effetto |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Sì (set di parametri Text) | Sì, per valore | L'intestazione come testo. Le righe dalla pipeline vengono unite in un'intestazione |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Sì (set di parametri Path) | Sì, per nome proprietà | File con l'intestazione o messaggio `.eml` completo; accetta oggetti da `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Sì (set di parametri Clipboard) | No | Legge l'intestazione dagli appunti (solo Windows) |
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | No | No | authserv-id del gateway in ingresso; solo le righe di verifica con uno di questi ID sono considerate comprovate |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Sì | Sì, per valore | L'oggetto risultato di `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | No | No | `Markdown` (predefinito) o `Text` |

Entrambi i cmdlet supportano i Common Parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` e gli altri di [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installazione

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Opzione | Effetto |
|---|---|
| `-Name MailHeaderAnalyzer` | Nome del modulo nella PowerShell Gallery |
| `-Scope CurrentUser` | Installa nella directory dei moduli dell'utente, senza diritti di amministratore |

Su un sistema senza accesso a Internet, scaricare il modulo su un altro computer con `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` e copiare la cartella `MailHeaderAnalyzer` in una directory di `$env:PSModulePath`, ad esempio `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) o `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). Gli aggiornamenti sono ottenuti da `Update-Module -Name MailHeaderAnalyzer`, mentre `Get-Module -Name MailHeaderAnalyzer -ListAvailable` mostra la versione installata.

## Get-MailHeaderAnalysis

Analizza l'intestazione di un'e-mail e restituisce un oggetto di analisi.

### Sintassi

#### Text (predefinito)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [-TrustedAuthServId <String[]>]
    [<CommonParameters>]
```

### Descrizione

Il cmdlet scompone l'intestazione grezza in campi, espande le piegature RFC 5322 e decodifica i valori RFC 2047 nell'oggetto e negli indirizzi. Dalle righe `Received` forma la catena di consegna in ordine cronologico, calcola il ritardo per ogni stazione e legge versione TLS, cipher e classe di protocollo secondo RFC 3848. Da `Authentication-Results`, `Received-SPF`, `DKIM-Signature` e dalla catena ARC determina i risultati di autenticazione e assegna a ogni riga di verifica un'origine (RFC 8601, sezione 5): comprovata se il suo authserv-id è in `-TrustedAuthServId`, altrimenti soltanto plausibile o non comprovata, vedere [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). Si aggiungono l'allineamento DMARC, la classificazione ibrida di Exchange Online, le valutazioni dei filtri antispam e un elenco di anomalie.

Il cmdlet non esegue query DNS e non apre connessioni di rete. `Spf`, `Dkim`, `Dmarc` e `Arc` sono quindi sempre il giudizio del server ricevente. Le firme DKIM e ARC non vengono ricalcolate crittograficamente; per la catena ARC il cmdlet verifica soltanto la struttura (`ArcStructure`).

Gli input sono letti in modo tollerante: una riga vuota termina l'intestazione e il testo del messaggio successivo viene ignorato. Le righe senza nome di campo e senza spazio iniziale, come quelle generate copiando dalle finestre di dialogo dei client, appartengono al campo precedente. Una riga separatrice mbox `From ...` prima del primo campo viene ignorata e un Byte Order Mark viene rimosso. Vengono analizzate al massimo 200 righe `Received`, conteggiate dalla consegna.

### Esempi

#### Esempio 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analizza l'intestazione presente negli appunti. In Outlook per Windows l'intestazione si trova in File, Proprietà, Intestazioni Internet; in Outlook sul Web, nelle opzioni del messaggio sotto «Visualizza dettagli messaggio».

```text
Subject     : Service-Report für März
From        : Beispiel Newsletter <news@example.org>
Date        : 2026-08-03 09:14:27Z
SPF         : pass
DKIM        : pass
DMARC       : pass
ARC         : -
CompAuth    : pass (reason 100)
AuthTrust   : Absent (no authserv-id, Microsoft 365 style)
Hops        : 3 (total 00:00:42)
DeliveredBy : zr0p278mb0570.chep278.prod.outlook.com
Findings    : none
```

#### Esempio 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analizza un messaggio salvato. Il file può contenere solo l'intestazione o il messaggio completo; il testo del messaggio viene ignorato.

#### Esempio 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Mostra la catena di consegna come tabella, con il primo hop per primo.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Esempio 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Legge l'intestazione riga per riga da un file di testo e mostra solo le anomalie. `-Raw` con `Get-Content` non è necessario, poiché il cmdlet unisce le righe autonomamente.

#### Esempio 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analizza tutti i messaggi di una cartella e scrive un file CSV con una riga per messaggio. La colonna calcolata riunisce i codici di rilevamento.

| Opzione | Effetto |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Restituisce gli oggetti file; `-Path` accetta la relativa proprietà `FullName` |
| `Select-Object … @{ Name; Expression }` | Colonna calcolata che riunisce tutti i codici di rilevamento in una stringa |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV senza intestazione di tipo, UTF-8 per gli umlaut nelle righe dell'oggetto |

#### Esempio 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Elenca i messaggi i cui risultati di verifica non provengono dal gateway in ingresso `mx.example.org` dell'organizzazione. Nelle analisi di phishing questo è un primo filtro. Senza `-TrustedAuthServId` si può filtrare soltanto per `Unmatched`; una falsificazione che include anche una riga `Received` corrispondente appare quindi come `Matched` e passa il filtro.

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id che il gateway scrive in `Authentication-Results`; confronto esatto, senza sottodomini |
| `Where-Object AuthTrust -ne 'Trusted'` | Mantiene tutti i messaggi la cui riga di verifica determinante non reca un authserv-id attendibile |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | File, livello di origine, authserv-id della riga di verifica e stazione di consegna |

</details>

#### Esempio 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Mostra la stazione con il ritardo maggiore. `SlowestHopIndex` è basato su 1, mentre l'array `Hops` è basato su 0.

#### Esempio 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Scrive l'analisi completa come JSON. `-Depth 6` è necessario perché `Hops`, `DkimSignatures` e `Exchange` sono oggetti annidati; il valore predefinito 2 li mostrerebbe solo come nomi di tipo.

### Parametri

#### -Header

L'intestazione come testo. Il parametro accetta una singola stringa con l'intera intestazione oppure più stringhe; gli input della pipeline vengono raccolti e alla fine uniti in un'intestazione, pertanto `Get-Content datei | Get-MailHeaderAnalysis` funziona senza `-Raw`. Per analizzare più intestazioni separatamente, usare `-Path` con più file.

Alias: `Text`, `Raw`, `InputObject`

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Caratteri jolly supportati | No |

| Set di parametri Text | Valore |
|---|---|
| Posizione | 0 |
| Obbligatorio | Sì |
| Valore dalla pipeline | Sì |
| Valore dalla pipeline per nome proprietà | No |

#### -Path

Percorso di un file contenente l'intestazione o un messaggio `.eml` completo. I percorsi relativi vengono risolti rispetto alla directory corrente. Il file viene letto con `[System.IO.File]::ReadAllText`: viene considerato un Byte Order Mark, senza BOM viene usato UTF-8. Per ogni file viene creato un oggetto risultato distinto; i file mancanti generano un errore non terminante.

Alias: `FullName`, `PSPath`, `LiteralPath`

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Caratteri jolly supportati | No |

| Set di parametri Path | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | Sì |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | Sì |

#### -FromClipboard

Legge l'intestazione dagli appunti con `Get-Clipboard -Raw`. Il parametro è disponibile solo in Windows; su Linux e macOS il cmdlet termina con un messaggio di errore, così come con appunti vuoti.

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `SwitchParameter` |
| Valore predefinito | `False` |
| Caratteri jolly supportati | No |

| Set di parametri Clipboard | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | Sì |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | No |

#### -TrustedAuthServId

L'authserv-id o gli authserv-id che il gateway in ingresso scrive in `Authentication-Results`, ad esempio `mx.example.org`. Il cmdlet effettua il confronto esatto, senza distinzione tra maiuscole e minuscole e senza sottodomini. Le righe di verifica con uno di questi ID ricevono `AuthTrust = Trusted`, e soltanto esse confluiscono quindi in `Spf`, `Dkim`, `Dmarc` e `Arc`. Lo stesso vale per il `receiver=` di una riga `Received-SPF`.

Il risultato è affidabile solo quanto il gateway: esso deve rimuovere le righe `Authentication-Results` in ingresso che rivendicano il proprio authserv-id (RFC 8601, sezione 5). Non è possibile stabilire da un'intestazione se lo faccia. Se due righe con ID attendibile sono in conflitto, il cmdlet segnala `AuthTrustedConflict`. Microsoft 365 non scrive un authserv-id nella propria riga di verifica; se Exchange Online riceve la posta, omettere il parametro.

Per un'intera sessione, il valore può essere memorizzato come predefinito, ad esempio nel profilo PowerShell:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Caratteri jolly supportati | No |

| Set di parametri (tutti) | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | No |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | No |

### Input

`System.String`: righe di intestazione o l'intera intestazione, a `-Header`.

`System.IO.FileInfo`: oggetti file da `Get-ChildItem`, la cui proprietà `FullName` viene associata a `-Path`.

### Output

`MailHeaderAnalyzer.Analysis`: un oggetto per ogni intestazione analizzata. Le proprietà sono descritte nella sezione [Oggetto di output](#ausgabeobjekt).

### Note

Il testo dell'analisi (spiegazioni in `Findings`, `CompAuthReasonMeaning` e nei campi di significato) è in inglese per poter essere inserito invariato nei ticket internazionali. L'output della visualizzazione predefinita può essere mostrato completamente con `Format-List *`.

## ConvertTo-MailHeaderReport

Genera da un oggetto di analisi un report in Markdown o testo.

### Sintassi

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Descrizione

Il report comprende oggetto, mittente, data e Message-ID, risultati di autenticazione con indicazione dell'origine e allineamento, tutti i rilevamenti, la catena di consegna, la classificazione Exchange e i valori dei filtri antispam. I caratteri di controllo Unicode per la direzione della scrittura rimangono visibili nel report come `<U+...>`, affinché non arrivino in un sistema di ticket tramite il report. L'ultima riga indica la versione del modulo.

### Esempi

#### Esempio 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Genera un report Markdown e lo inserisce negli appunti.

```markdown
# Email header analysis

- Subject: Service-Report für März
- From: Beispiel Newsletter <news@example.org>
- Date: 2026-08-03T09:14:27Z
- Message-ID: <20260803091427.4Wq2Wx5RbTz@mail.example.org>

## Authentication

- spf: pass
- dkim: pass
- dmarc: pass
- compauth: pass (reason=100: passed authentication)
- Results without authserv-id (Microsoft 365 style)
- DMARC alignment: SPF Strict, DKIM Strict

## Delivery chain (total: 42 s)

| # | From | By | Protocol | TLS | Time (UTC) | Delay |
|---|---|---|---|---|---|---|
| 1 | client.example.net (198.51.100.34) | mail.example.org | ESMTPSA | - | 2026-08-03T09:14:28Z | - |
| 2 | mail.example.org (203.0.113.25) | mx.eur02.prod.protection.outlook.com | Microsoft SMTP Server | TLS 1.3 | 2026-08-03T09:15:09Z | 41 s |
| 3 | AM0EUR02FT056.eop-EUR02.prod.protection.outlook.com | ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM | Microsoft SMTP Server | TLS 1.2 | 2026-08-03T09:15:10Z | 1 s |
```

#### Esempio 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Restituisce il report come testo senza formattazione Markdown, ad esempio per e-mail o registri della console.

#### Esempio 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Utilizzo nella Exchange Management Shell. Se `$env:PSModulePath` è limitato da criteri di gruppo, caricare il modulo con `Import-Module` e il percorso completo del file `.psd1`.

### Parametri

#### -Analysis

L'oggetto di analisi da `Get-MailHeaderAnalysis`. Il cmdlet rifiuta altri tipi di oggetti con un errore di associazione.

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `MailHeaderAnalyzer.Analysis` |
| Valore predefinito | Nessuno |
| Caratteri jolly supportati | No |

| Set di parametri (tutti) | Valore |
|---|---|
| Posizione | 0 |
| Obbligatorio | Sì |
| Valore dalla pipeline | Sì |
| Valore dalla pipeline per nome proprietà | No |

#### -Format

Il formato di output. Valori validi:

- `Markdown`: intestazioni, elenchi e catena di consegna come tabella. Predefinito.
- `Text`: intestazioni in maiuscolo, righe rientrate, catena di consegna come elenco numerato.

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String` |
| Valori consentiti | `Markdown`, `Text` |
| Valore predefinito | `Markdown` |
| Caratteri jolly supportati | No |

| Set di parametri (tutti) | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | No |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | No |

### Input

`MailHeaderAnalyzer.Analysis`: il risultato di `Get-MailHeaderAnalysis`.

### Output

`System.String`: il report, una stringa per oggetto di analisi.

## Oggetto di output

`Get-MailHeaderAnalysis` restituisce per ogni input un oggetto di tipo `MailHeaderAnalyzer.Analysis`. La visualizzazione predefinita mostra il riepilogo dell'esempio 1; tutte le proprietà sono accessibili tramite `Select-Object`, `Format-List *` o `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Proprietà | Tipo | Contenuto |
|---|---|---|
| `Source` | String | Percorso file, `Clipboard` o `Text` |
| `Subject` | String | Oggetto, decodificato RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Oggetto indirizzo | `Name`, `Address`, `Domain`, `Display`; `$null` se il campo manca |
| `Date` | DateTime (UTC) | Valore del campo `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Dominio del mittente envelope da `smtp.mailfrom` della verifica SPF, altrimenti da `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Risultato secondo la riga `Authentication-Results` determinante (`pass`, `fail`, `none`, `softfail` e altri); `$null` se non verificato. `Arc` è il giudizio del destinatario sulla catena ARC |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Codice motivo dell'autenticazione composita di Microsoft 365 e relativo significato |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` o `None`, vedere [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id della riga di verifica determinante |
| `AuthenticationResults` | Oggetto[] | Tutte le righe `Authentication-Results` con `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Oggetto | La riga `Received-SPF` con `Result` e `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` o `$null` |
| `Hops` | Hop[] | Catena di consegna in ordine cronologico, vedere [oggetto Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Metriche della catena |
| `DeliveredBy` | String | Host `by` della riga `Received` più recente, ovvero la stazione di consegna |
| `DkimSignatures` | Oggetto[] | Per ogni firma `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Oggetto[] | Istanze ARC con `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | Struttura della catena ARC: `Consistent`, `Inconsistent` o `$null` senza intestazioni ARC, oltre alle discrepanze rilevate. Solo verifica strutturale, nessuna verifica della firma, vedere [catena ARC](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Oggetto | Classificazione ibrida di Exchange Online, vedere [oggetto Exchange](#mailheaderanalyzerexchangeclassification); `$null` senza intestazioni corrispondenti |
| `Spam` | Oggetto | Valutazioni dei filtri antispam, vedere [oggetto Spam](#mailheaderanalyzerspamassessment); `$null` senza intestazioni corrispondenti |
| `List` | Oggetto | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` senza intestazioni di lista |
| `Findings` | Finding[] | Anomalie con `Severity`, `Code`, `Message`, vedere [rilevamenti](#findings) |
| `Fields` | Oggetto[] | Tutti i campi con `Name`, `Value` (espanso) e `Raw` |
| `HadBody` | Bool | Se dopo l'intestazione seguiva un testo del messaggio |

### MailHeaderAnalyzer.Hop

Ogni voce in `Hops` corrisponde a una riga `Received`. L'ordine è cronologico, quindi inverso rispetto all'ordine nell'intestazione.

| Proprietà | Contenuto |
|---|---|
| `Index` | Numero progressivo, 1 = immissione |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Dati sul sistema mittente dalla parte `from`; l'IP proviene dalle parentesi quadre nel commento, il nome rDNS dal commento precedente |
| `ByHost`, `Software` | Sistema ricevente e relativo software (commento dopo `by`) |
| `Protocol`, `ProtocolClass` | Valore `with` e classe secondo RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS o indicazione TLS presente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Dalle notazioni di Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) ed Exim |
| `Id`, `For`, `Via` | Altri componenti di `Received` |
| `Date` | Timestamp dopo il punto e virgola, UTC |
| `Delay` | TimeSpan rispetto all'hop precedente; negativo in caso di sfasamento degli orologi |
| `Provider` | Provider o gateway riconosciuto dai nomi host (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast e altri) |
| `Attested` | `$true` solo nell'ultimo hop: solo questa riga è stata scritta dal sistema ricevente stesso, tutte quelle sottostanti erano già nel messaggio |
| `Raw` | La riga originale |

### AuthTrust: origine dei risultati di verifica

Una riga `Authentication-Results` può essere scritta nel messaggio dal mittente stesso, così come una riga `Received` corrispondente. Dalla sola intestazione non è pertanto possibile dimostrare chi abbia scritto una riga di verifica. Secondo RFC 8601, sezione 5, è determinante soltanto la riga il cui authserv-id è noto all'organizzazione ricevente come proprio, e il gateway in ingresso deve rimuovere le righe in ingresso con questo ID. Il cmdlet implementa questa regola con `-TrustedAuthServId`. Senza questo parametro confronta l'authserv-id solo con gli host `by` della catena `Received` (stesso dominio o sottodominio, sempre al confine del punto, nessuna sottostringa); si tratta di una verifica di plausibilità.

| Valore | Significato |
|---|---|
| `Trusted` | L'authserv-id è in `-TrustedAuthServId`. Non appena esiste una riga di questo tipo, solo le righe di questo livello confluiscono in `Spf`, `Dkim`, `Dmarc` e `Arc`. Affidabile, a condizione che il gateway rimuova le righe estranee con questo ID |
| `Matched` | L'authserv-id appare come host `by` nella catena. Plausibile, ma non una prova: una falsificazione può fornire la riga `Received` corrispondente. Senza `-TrustedAuthServId`, il rilevamento `AuthPlausibleOnly` lo segnala. Le righe di questo livello vengono conteggiate se non è presente una riga `Trusted` |
| `Unmatched` | L'authserv-id non è né attendibile né presente nella catena. I risultati vengono mostrati, ma sono considerati un'affermazione non comprovata; il rilevamento `AuthUnverified` lo segnala |
| `Absent` | La riga non contiene un authserv-id. Microsoft 365 scrive la propria riga di verifica in questa forma: inizia direttamente con `spf=` |
| `None` | Nessuna riga di verifica presente |

Se l'intestazione contiene righe di verifica di origini diverse, il rilevamento `AuthMixedOrigins` segnala la situazione. Se manca una riga di verifica ma è presente una riga `Received-SPF`, il relativo risultato viene adottato come `Spf` e contrassegnato con `ReceivedSpfOnly`.

Con `-TrustedAuthServId` vengono aggiunti due rilevamenti: `AuthNotTrusted`, se nessuna riga di verifica reca un ID attendibile, e `AuthTrustedConflict`, se due righe di questo tipo riportano risultati diversi per `spf`, `dmarc`, `arc` o `compauth`. Il secondo caso indica che almeno una riga non proviene dal gateway e che il gateway non l'ha rimossa. In questo caso il cmdlet adotta la riga più alta; dalla sola intestazione non è possibile decidere quale delle due sia autentica. DKIM è escluso da questo confronto, poiché più firme possono legittimamente avere risultati diversi.

### Catena ARC: struttura e giudizio

Per ARC il cmdlet fornisce due informazioni separate. `Arc` è il risultato `arc=` della riga di verifica determinante, ossia il giudizio del destinatario che ha verificato le firme della catena. `ArcStructure` è la verifica propria del modulo e riguarda solo la struttura secondo RFC 8617: numeri di istanza consecutivi `i=1` fino a `i=n` (al massimo 50), esattamente un `ARC-Seal`, un `ARC-Message-Signature` e un `ARC-Authentication-Results` per istanza, nonché `cv=none` nell'istanza 1 e `cv=pass` in tutte le successive. Il modulo non ricalcola le firme; `Consistent` non afferma quindi nulla sull'autenticità della catena. Le discrepanze sono riportate in `ArcStructureIssues` e nel rilevamento `ArcStructureInconsistent`.

Le informazioni in `ARC-Authentication-Results` sono dichiarazioni del rispettivo inoltro. Il rilevamento `DkimBrokenAfterForward` classifica quindi un errore DKIM come conseguenza di un inoltro solo se il destinatario stesso riporta `arc=pass` e un'istanza precedente ha registrato un DKIM-`pass` per lo stesso dominio.

### Allineamento DMARC

`SpfAlignment` confronta il dominio del mittente envelope con il dominio `From`, mentre `DkimAlignment` confronta il dominio `d=` della firma verificata con il dominio `From`. `Strict` indica un dominio identico, `Relaxed` lo stesso dominio organizzativo, `None` nessuna corrispondenza. Il dominio organizzativo viene determinato euristicamente: le ultime due etichette, oppure le ultime tre per terminazioni multipartite note come `co.uk` o `com.au`. Non è inclusa una Public Suffix List completa.

### MailHeaderAnalyzer.ExchangeClassification

Se l'intestazione contiene i campi `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` o `X-MS-Exchange-CrossTenant-*`, il cmdlet popola la proprietà `Exchange`. I significati seguono l'articolo «Demystifying hybrid mail flow» del team Exchange; il contesto è illustrato nell'articolo [Intestazioni ibride Exchange: interne o esterne?](/blog/exchange-hybrid-header-intern-extern).

| Proprietà | Contenuto |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` o `Incoming` con spiegazione |
| `AuthAs`, `AuthAsMeaning` | `Internal` o `Anonymous` con le conseguenze per il filtraggio EOP |
| `AuthSource` | Server che ha eseguito la classificazione |
| `AuthMechanism`, `AuthMechanismMeaning` | Codice del meccanismo. Solo il valore 10 (Externally Secured) è documentato pubblicamente; per tutti gli altri codici il modulo indica esplicitamente che Microsoft non li documenta |
| `OriginatorOrg` | Dominio predefinito del tenant mittente, la caratteristica del tenant non falsificabile alla ricezione da Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classificazione e ID tenant al confine del tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` o `HybridOnPrem` con spiegazione |
| `WrongTenantAttribution` | Valore di `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` se il messaggio è stato associato a un tenant estraneo |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marcatori per intestazioni organizzative ricevute o rimosse dal connettore di invio |

### MailHeaderAnalyzer.SpamAssessment

La proprietà `Spam` riunisce le valutazioni dei filtri noti. I valori provengono da sistemi esterni e vengono decodificati, ma non valutati.

| Proprietà | Fonte | Contenuto |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level con significato (-1 attendibile, 0/1 non spam, 5/6 sospetto spam, 9 spam molto probabile) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level da 0 a 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classificazione come `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Risultato del filtro come `NSPM`, `SPM`, `SKA` (allowlist), `SKI` (intraorganizzativo) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP nella allowlist di connessione) o `NLI` (nessuna reputazione) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | IP mittente e paese di origine |
| `Forefront` | `X-Forefront-Antispam-Report` | Tutte le coppie chiave-valore del campo |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Punteggio e test attivati |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Simboli con punteggio |

L'elenco completo dei codici motivo `compauth` è disponibile nell'articolo [Microsoft 365 compauth: codici motivo](/blog/microsoft-365-compauth-reason-codes).

### Rilevamenti

Il cmdlet restituisce le anomalie come oggetti in `Findings`, ciascuno con `Severity` (`Info`, `Warning`, `Fail`), un `Code` stabile per filtri e script e una spiegazione in `Message`.

| Codice | Gravità | Significato |
|---|---|---|
| `DuplicateField` | Warning | Un campo che RFC 5322 limita a una singola istanza (`From`, `Subject`, `Date`, `Message-ID` e altri) appare più volte. I client di posta e i filtri possono scegliere istanze diverse; è un modello noto nelle falsificazioni |
| `BidiControls` | Warning | Caratteri di controllo Unicode per la direzione della scrittura in un campo. Invertiscono la direzione di lettura: `fdp.exe` appare quindi come `exe.pdf`. Il modulo li mostra come `<U+202E>` |
| `HopOverflow` | Warning | Più di 200 righe `Received`; quelle eccedenti non sono state analizzate |
| `AuthPlausibleOnly` | Info | L'authserv-id appare nella catena di consegna, ma `-TrustedAuthServId` non è stato specificato: plausibile, non comprovato |
| `AuthNotTrusted` | Warning | È stato specificato `-TrustedAuthServId`, ma nessuna riga di verifica reca uno di questi ID |
| `AuthTrustedConflict` | Warning | Due righe di verifica con ID attendibile riportano risultati diversi per lo stesso metodo; il gateway apparentemente non rimuove le righe estranee |
| `AuthUnverified` | Warning | I risultati di verifica recano un authserv-id che non è né attendibile né presente nella catena di consegna |
| `AuthMixedOrigins` | Warning | Sono presenti righe di verifica di origini diverse |
| `ReceivedSpfForeign` | Warning | Il `receiver=` della riga `Received-SPF` non appare nella catena |
| `ReceivedSpfOnly` | Info | Il risultato SPF proviene solo da `Received-SPF`, non da una riga di verifica |
| `NoAuthResults` | Info | Nessun risultato di verifica nell'intestazione |
| `DmarcFail` | Fail | DMARC non superato secondo il server ricevente |
| `SpfNotPass` | Warning | Risultato SPF `fail`, `softfail`, `permerror` o `temperror` |
| `DkimNotPass` | Warning | Risultato DKIM `fail`, `permerror` o `temperror`, senza che il destinatario confermi la catena ARC |
| `DkimBrokenAfterForward` | Info | DKIM non superato dal destinatario, ma il destinatario riporta `arc=pass`, e un'istanza ARC precedente ha registrato un DKIM-`pass` per lo stesso dominio: tipico di inoltri e mailing list |
| `DkimWeakHash` | Warning | Firma con `rsa-sha1` (RFC 8301 classifica SHA-1 come obsoleto) |
| `DkimBodyLength` | Warning | Il tag `l=` limita la lunghezza firmata del testo del messaggio; il contenuto aggiunto non è coperto |
| `DkimExpired` | Warning | L'istante `x=` è nel passato |
| `DkimFromUnsigned` | Warning | Il campo `From` non è contenuto in `h=`, benché RFC 6376 lo richieda |
| `ArcStructureInconsistent` | Warning | Le intestazioni ARC non formano una catena formalmente completa (lacune, intestazioni mancanti o duplicate, sequenza `cv=` errata); solo verifica strutturale |
| `ClockSkew` | Info | Un hop presenta un timestamp precedente rispetto al suo predecessore; i ritardi sono soltanto valori approssimativi |
| `ReplyToMismatch` | Info | Il dominio `Reply-To` differisce dal dominio `From`; comune nelle newsletter, un modello nel phishing |
| `SpfNotAligned` | Info | Il dominio del mittente envelope e il dominio `From` appartengono a organizzazioni diverse; SPF non contribuisce quindi a DMARC |
| `ExchangeWrongTenant` | Warning | Il messaggio è stato associato a un tenant estraneo; una causa classica è un connettore in ingresso di un altro tenant con lo stesso certificato o gli stessi indirizzi IP |
| `ExchangeHeadersFiltered` | Warning | Il connettore di invio ha rimosso le intestazioni Cross-Premises (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: ingresso tramite un connettore di ricezione con «Externally Secured», filtraggio EOP ignorato |
| `SpamCategory` | Warning | Microsoft ha assegnato una categoria diversa da `NONE` |
| `SpamConfidence` | Warning | SCL pari o superiore a 5 |

## Funzionamento e limiti

Il modulo legge ciò che è presente nell'intestazione e ne deduce ciò che può essere dimostrato senza query esterne. Ne derivano alcuni limiti:

- **Nessuna verifica crittografica.** Le firme DKIM e ARC non vengono ricalcolate e le voci DNS non vengono interrogate. `Spf`, `Dkim`, `Dmarc` e `Arc` sono sempre il giudizio del server ricevente; `ArcStructure` verifica soltanto la struttura della catena.
- **L'origine è comprovabile solo conoscendo il gateway.** Senza `-TrustedAuthServId`, `AuthTrust` è una verifica di plausibilità. Con il parametro, l'affermazione dipende dal fatto che il gateway rimuova le righe di verifica estranee con il proprio authserv-id; il modulo non può verificarlo.
- **Solo l'ultima riga `Received` è comprovata.** Tutte le righe sottostanti sono state fornite dal mittente e possono essere strutturate arbitrariamente. `Attested` contrassegna questa differenza; i ritardi degli hop precedenti si basano sulle informazioni di tali righe.
- **Domini organizzativi euristici.** Per l'allineamento relaxed, il modulo utilizza un breve elenco di terminazioni multipartite, non una Public Suffix List completa.
- **AuthMechanism documentato solo in parte.** Oltre al valore 10, Microsoft non ha pubblicato i codici; il modulo non inventa significati.
- **Set di caratteri.** I valori RFC 2047 vengono decodificati con le codifiche note a .NET nel sistema in uso. I set di caratteri sconosciuti restano invariati.

## Privacy

Un'intestazione completa contiene nomi host interni, indirizzi IP, mittenti, destinatari e oggetto. Il motivo per cui queste informazioni non dovrebbero essere inserite in uno strumento online è illustrato nell'articolo [Analizzare le intestazioni e-mail senza caricare l'e-mail](/blog/e-mail-header-analysieren-ohne-upload). Per il modulo vale la stessa promessa della versione browser: nessun accesso alla rete. La suite di test del modulo contiene un test che fallisce non appena nel codice sorgente appare un cmdlet di rete o DNS.

## Codice sorgente e versioni

Il codice sorgente è disponibile con licenza MIT su [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La logica di analisi è una conversione della libreria utilizzata anche dall'analizzatore delle intestazioni su questo sito; entrambi condividono i casi di test. La Continuous Integration verifica ogni modifica con PSScriptAnalyzer e Pester su Windows PowerShell 5.1, PowerShell 7 in Windows, Ubuntu e macOS. Le pubblicazioni nella PowerShell Gallery avvengono automaticamente da tag con versione; le modifiche per versione sono disponibili nel [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

La versione 0.2.0 del 26 settembre 2026 ha rafforzato il modello di origine dopo un'osservazione nella revisione di @saltyslugga: parametro `-TrustedAuthServId`, livello `Trusted`, `Matched` solo come plausibilità. La proprietà `ArcValid` è stata rimossa e sostituita da `ArcStructure` e `ArcStructureIssues`; gli script che analizzano `ArcValid` devono essere adeguati.

Accetto segnalazioni di errori e richieste di estensione come [issue su GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonimizzare prima le intestazioni di test per le segnalazioni di errori; i casi di test inclusi utilizzano esclusivamente domini di esempio secondo RFC 2606 e indirizzi secondo RFC 5737.

## Fonti

1.  [MailHeaderAnalyzer nella PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): pagina del pacchetto con comando di installazione e cronologia delle versioni.

2.  [pfstr/MailHeaderAnalyzer su GitHub](https://github.com/pfstr/MailHeaderAnalyzer): codice sorgente, suite di test, changelog e issue.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): modello per la struttura di questo riferimento (sintassi, descrizione, esempi, proprietà dei parametri).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): struttura di `Authentication-Results` e regola secondo cui è determinante solo la riga dell'organizzazione ricevente (sezione 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): struttura delle righe `Received` e raccomandazione di un limite massimo come protezione dai loop (sezione 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): valori `with` `ESMTPS`, `ESMTPA` e `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): tag di `DKIM-Signature`, obbligo di firmare `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): classificazione di `rsa-sha1` come obsoleto.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): allineamento strict e relaxed.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): struttura della catena composta da `ARC-Seal`, `ARC-Message-Signature` e `ARC-Authentication-Results`, numeri di istanza e valori `cv=`.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): significato di SCL, BCL, CAT, SFV, IPV e dei codici motivo `compauth`.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): origine dei significati di MessageDirectionality, AuthAs e AuthMechanism.

13.  [Analizzatore delle intestazioni su rafaelpfister.ch](/tools/header-analyzer): la versione browser con la stessa logica di analisi.
