---
title: "MailHeaderAnalyzer: analizzare le intestazioni e-mail in PowerShell senza accesso alla rete"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Riferimento al modulo PowerShell MailHeaderAnalyzer: panoramica dei parametri, sintassi, descrizione, esempi e proprietà dei parametri di Get-MailHeaderAnalysis e ConvertTo-MailHeaderReport, nonché oggetto di output con catena di consegna, risultati dell'autenticazione, classificazione di Exchange Online e tutti i codici Finding."
date: "2026-09-24"
kategorie: "SMTP e flusso di posta"
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
translationSourceHash: 7f1ed02181dcd5665641de26ab47fc5acaeaa73a230c60093063ab8bb99e15d3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:31:03.908Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/mailheaderanalyzer-analizzare-le-intestazioni-e-mail-in-powershell-senza-accesso-alla-rete
---

# MailHeaderAnalyzer: analizzare le intestazioni e-mail in PowerShell senza accesso alla rete

MailHeaderAnalyzer è un modulo PowerShell con due cmdlet. `Get-MailHeaderAnalysis` analizza l'intestazione di un'e-mail: catena di consegna con ritardi e dati TLS, risultati SPF, DKIM, DMARC e ARC inclusa la verifica della provenienza rispetto all'authserv-id del gateway, allineamento DMARC, classificazione ibrida di Exchange Online, valutazioni di Microsoft Defender, SpamAssassin e Rspamd, nonché anomalie quali righe `From` duplicate o caratteri di controllo Unicode. `ConvertTo-MailHeaderReport` genera un report per i ticket. Il modulo opera interamente offline: nessuna query DNS, nessuna connessione HTTP. È la versione a riga di comando dell'[analizzatore di intestazioni su questo sito](/tools/header-analyzer) e usa la stessa logica di analisi.

| | |
|---|---|
| Modulo | [MailHeaderAnalyzer nella PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Codice sorgente | [pfstr/MailHeaderAnalyzer su GitHub](https://github.com/pfstr/MailHeaderAnalyzer), licenza MIT |
| Si applica a | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdlet | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Panoramica dei parametri

| Cmdlet | Parametro | Tipo | Obbligatorio | Pipeline | Effetto |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Sì (set di parametri Text) | Sì, per valore | L'intestazione come testo. Le righe dalla pipeline vengono combinate in un'intestazione |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Sì (set di parametri Path) | Sì, per nome proprietà | File con l'intestazione o messaggio `.eml` completo; accetta oggetti da `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Sì (set di parametri Clipboard) | No | Legge l'intestazione dagli appunti (solo Windows) |
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | No | No | authserv-id del gateway in ingresso; solo le righe di verifica con uno di questi ID sono considerate comprovate |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Sì | Sì, per valore | L'oggetto risultato di `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | No | No | `Markdown` (predefinito) oppure `Text` |

Entrambi i cmdlet supportano i Common Parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` e gli altri descritti in [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installazione

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Opzione | Effetto |
|---|---|
| `-Name MailHeaderAnalyzer` | Nome del modulo nella PowerShell Gallery |
| `-Scope CurrentUser` | Installa nella directory dei moduli dell'utente, senza diritti di amministratore |

Su un sistema senza accesso a Internet, scaricare il modulo su un altro computer con `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` e copiare la cartella `MailHeaderAnalyzer` in una directory di `$env:PSModulePath`, ad esempio `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) o `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). Gli aggiornamenti vengono ottenuti con `Update-Module -Name MailHeaderAnalyzer`, mentre `Get-Module -Name MailHeaderAnalyzer -ListAvailable` mostra la versione installata.

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

Il cmdlet suddivide l'intestazione grezza in campi, espande le piegature RFC 5322 e decodifica i valori RFC 2047 nell'oggetto e negli indirizzi. Dalle righe `Received` crea la catena di consegna in ordine cronologico, calcola il ritardo per ciascuna stazione e legge versione TLS, cifrario e classe di protocollo secondo RFC 3848. Da `Authentication-Results`, `Received-SPF`, `DKIM-Signature` e dalla catena ARC determina i risultati dell'autenticazione e assegna una provenienza a ogni riga di verifica (RFC 8601, sezione 5): comprovata se il suo authserv-id è in `-TrustedAuthServId`, altrimenti soltanto plausibile o non comprovata; vedere [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). Si aggiungono l'allineamento DMARC, la classificazione ibrida di Exchange Online, le valutazioni dei filtri antispam e un elenco di anomalie.

Il cmdlet non esegue query DNS né apre connessioni di rete. `Spf`, `Dkim`, `Dmarc` e `Arc` sono quindi sempre il giudizio del server ricevente. Le firme DKIM e ARC non vengono ricalcolate crittograficamente; per la catena ARC il cmdlet verifica soltanto la struttura (`ArcStructure`).

Gli input vengono letti in modo tollerante: una riga vuota termina l'intestazione e l'eventuale corpo del messaggio successivo viene ignorato. Le righe prive di nome campo e senza spazio iniziale, come possono risultare copiando da finestre di dialogo del client, appartengono al campo precedente. Una riga separatrice mbox `From ...` prima del primo campo viene ignorata e una Byte Order Mark viene rimossa. Vengono analizzate al massimo 200 righe `Received`, contando dalla consegna.

### Esempi

#### Esempio 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analizza l'intestazione negli appunti. In Outlook per Windows, l'intestazione si trova in File, Proprietà, Intestazioni Internet; in Outlook sul Web nelle opzioni del messaggio, sotto «Visualizza dettagli messaggio».

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

Analizza un messaggio salvato. Il file può contenere solo l'intestazione o il messaggio completo; il corpo del messaggio viene ignorato.

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

Legge l'intestazione riga per riga da un file di testo e mostra solo le anomalie. `-Raw` con `Get-Content` non è necessario, perché il cmdlet combina autonomamente le righe.

#### Esempio 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analizza tutti i messaggi di una cartella e scrive un file CSV con una riga per messaggio. La colonna calcolata riunisce i codici Finding.

| Opzione | Effetto |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Fornisce gli oggetti file; `-Path` acquisisce la loro proprietà `FullName` |
| `Select-Object … @{ Name; Expression }` | Colonna calcolata che riunisce tutti i codici Finding in una stringa |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV senza riga di intestazione del tipo, UTF-8 per gli umlaut nelle righe dell'oggetto |

#### Esempio 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Elenca i messaggi i cui risultati di verifica non provengono dal gateway in ingresso `mx.example.org`. Nelle analisi di phishing questo è un primo filtro. Senza `-TrustedAuthServId` si può filtrare soltanto in base a `Unmatched`; una falsificazione che includa anche una riga `Received` corrispondente apparirà allora come `Matched` e passerà il filtro.

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id che il gateway scrive in `Authentication-Results`; confronto esatto, senza sottodomini |
| `Where-Object AuthTrust -ne 'Trusted'` | Mantiene tutti i messaggi la cui riga di verifica determinante non reca un authserv-id attendibile |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | File, livello di provenienza, authserv-id della riga di verifica e stazione di consegna |

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

Scrive l'analisi completa in formato JSON. `-Depth 6` è necessario perché `Hops`, `DkimSignatures` e `Exchange` sono oggetti annidati; il valore predefinito 2 li mostrerebbe soltanto come nomi di tipo.

### Parametri

#### -Header

L'intestazione come testo. Il parametro accetta una singola stringa con l'intera intestazione oppure più stringhe; gli input della pipeline vengono raccolti e infine combinati in un'intestazione, pertanto `Get-Content datei | Get-MailHeaderAnalysis` funziona senza `-Raw`. Per analizzare separatamente più intestazioni, usare `-Path` con più file.

Alias: `Text`, `Raw`, `InputObject`

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Supporta caratteri jolly | No |

| Set di parametri Text | Valore |
|---|---|
| Posizione | 0 |
| Obbligatorio | Sì |
| Valore dalla pipeline | Sì |
| Valore dalla pipeline per nome proprietà | No |

#### -Path

Percorso di un file contenente l'intestazione o un messaggio `.eml` completo. I percorsi relativi vengono risolti rispetto alla directory corrente. Il file viene letto con `[System.IO.File]::ReadAllText`: viene considerata una Byte Order Mark; senza BOM si usa UTF-8. Per ogni file viene creato un oggetto risultato distinto; i file mancanti generano un errore non terminante.

Alias: `FullName`, `PSPath`, `LiteralPath`

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Supporta caratteri jolly | No |

| Set di parametri Path | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | Sì |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | Sì |

#### -FromClipboard

Legge l'intestazione dagli appunti con `Get-Clipboard -Raw`. Il parametro è disponibile solo in Windows; in Linux e macOS il cmdlet termina con un messaggio di errore, così come se gli appunti sono vuoti.

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `SwitchParameter` |
| Valore predefinito | `False` |
| Supporta caratteri jolly | No |

| Set di parametri Clipboard | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | Sì |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | No |

#### -TrustedAuthServId

L'authserv-id o gli authserv-id che il gateway in ingresso scrive in `Authentication-Results`, ad esempio `mx.example.org`. Il cmdlet confronta in modo esatto, senza distinzione tra maiuscole e minuscole e senza sottodomini. Le righe di verifica con uno di questi ID ricevono `AuthTrust = Trusted`, e solo esse confluiscono quindi in `Spf`, `Dkim`, `Dmarc` e `Arc`. Lo stesso vale per `receiver=` di una riga `Received-SPF`.

Il risultato è affidabile solo quanto il gateway: deve rimuovere le righe `Authentication-Results` in entrata che rivendicano il proprio authserv-id (RFC 8601, sezione 5). Non è possibile stabilire da un'intestazione se lo faccia. Se due righe con ID attendibile sono in contraddizione, il cmdlet segnala `AuthTrustedConflict`. Microsoft 365 non scrive alcun authserv-id nella propria riga di verifica; se Exchange Online riceve la posta, omettere il parametro.

Il valore può essere impostato come predefinito per un'intera sessione, ad esempio nel profilo PowerShell:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `String[]` |
| Valore predefinito | Nessuno |
| Supporta caratteri jolly | No |

| Set di parametri (tutti) | Valore |
|---|---|
| Posizione | Denominata |
| Obbligatorio | No |
| Valore dalla pipeline | No |
| Valore dalla pipeline per nome proprietà | No |

### Input

`System.String`: righe di intestazione o l'intera intestazione, a `-Header`.

`System.IO.FileInfo`: oggetti file di `Get-ChildItem`, la cui proprietà `FullName` viene associata a `-Path`.

### Output

`MailHeaderAnalyzer.Analysis`: un oggetto per ogni intestazione analizzata. Le proprietà sono descritte nella sezione [Oggetto di output](#ausgabeobjekt).

### Note

Il testo dell'analisi (le spiegazioni in `Findings`, `CompAuthReasonMeaning` e nei campi di significato) è in inglese, affinché possa essere inserito invariato nei ticket internazionali. L'output della visualizzazione standard può essere mostrato per intero con `Format-List *`.

## ConvertTo-MailHeaderReport

Genera un report in Markdown o testo da un oggetto di analisi.

### Sintassi

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Descrizione

Il report include oggetto, mittente, data e Message-ID, risultati dell'autenticazione con indicazione della provenienza e allineamento, tutti i Finding, la catena di consegna, la classificazione Exchange e i valori dei filtri antispam. I caratteri di controllo Unicode per la direzione di scrittura rimangono visibili nel report come `<U+...>`, affinché non possano passare dal report a un sistema di ticket. L'ultima riga indica la versione del modulo.

### Esempi

#### Esempio 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Genera un report Markdown e lo copia negli appunti.

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

Emette il report come testo senza formattazione Markdown, ad esempio per e-mail o registri della console.

#### Esempio 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Utilizzo nella Exchange Management Shell. Se `$env:PSModulePath` è limitato da criteri di gruppo, caricare il modulo con `Import-Module` e il percorso completo del file `.psd1`.

### Parametri

#### -Analysis

L'oggetto di analisi di `Get-MailHeaderAnalysis`. Il cmdlet rifiuta altri tipi di oggetto con un errore di associazione.

| Proprietà del parametro | Valore |
|---|---|
| Tipo | `MailHeaderAnalyzer.Analysis` |
| Valore predefinito | Nessuno |
| Supporta caratteri jolly | No |

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
| Supporta caratteri jolly | No |

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

`Get-MailHeaderAnalysis` restituisce per ogni input un oggetto di tipo `MailHeaderAnalyzer.Analysis`. La visualizzazione standard mostra il riepilogo dell'esempio 1; tutte le proprietà sono accessibili tramite `Select-Object`, `Format-List *` o `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Proprietà | Tipo | Contenuto |
|---|---|---|
| `Source` | String | Percorso del file, `Clipboard` o `Text` |
| `Subject` | String | Oggetto, decodificato RFC 2047 |
| `From`, `ReplyTo`, `ReturnPath` | Oggetto indirizzo | `Name`, `Address`, `Domain`, `Display`; `$null` se il campo manca |
| `Date` | DateTime (UTC) | Valore del campo `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Dominio del mittente envelope da `smtp.mailfrom` della verifica SPF, altrimenti da `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Risultato secondo la riga `Authentication-Results` determinante (`pass`, `fail`, `none`, `softfail` e altri); `$null` se non verificato. `Arc` è il giudizio del destinatario sulla catena ARC |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Codice motivo dell'autenticazione composta di Microsoft 365 e relativo significato |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` o `None`, vedere [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id della riga di verifica determinante |
| `AuthenticationResults` | Oggetto[] | Tutte le righe `Authentication-Results` con `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Oggetto | La riga `Received-SPF` con `Result` e `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` o `$null` |
| `Hops` | Hop[] | Catena di consegna in ordine cronologico, vedere [Oggetto Hop](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Metriche della catena |
| `DeliveredBy` | String | Host `by` della riga `Received` più recente, ossia la stazione di consegna |
| `DkimSignatures` | Oggetto[] | Per firma: `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Oggetto[] | Istanze ARC con `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | Struttura della catena ARC: `Consistent`, `Inconsistent` o `$null` senza intestazioni ARC, con le discrepanze rilevate. Solo verifica della struttura, nessuna verifica delle firme; vedere [Catena ARC](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Oggetto | Classificazione ibrida di Exchange Online, vedere [Oggetto Exchange](#mailheaderanalyzerexchangeclassification); `$null` senza intestazioni corrispondenti |
| `Spam` | Oggetto | Valutazioni dei filtri antispam, vedere [Oggetto Spam](#mailheaderanalyzerspamassessment); `$null` senza intestazioni corrispondenti |
| `List` | Oggetto | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` senza intestazioni di liste |
| `Findings` | Finding[] | Anomalie con `Severity`, `Code`, `Message`, vedere [Finding](#findings) |
| `Fields` | Oggetto[] | Tutti i campi con `Name`, `Value` (espanso) e `Raw` |
| `HadBody` | Bool | Indica se dopo l'intestazione era presente un corpo del messaggio |

### MailHeaderAnalyzer.Hop

Ogni voce in `Hops` corrisponde a una riga `Received`. L'ordine è cronologico, quindi inverso rispetto all'ordine nell'intestazione.

| Proprietà | Contenuto |
|---|---|
| `Index` | Numero progressivo, 1 = immissione |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Dati sul sistema di invio dalla parte `from`; l'IP proviene dalle parentesi quadre nel commento, il nome rDNS dal commento precedente |
| `ByHost`, `Software` | Sistema ricevente e relativo software (commento dopo `by`) |
| `Protocol`, `ProtocolClass` | Valore `with` e classe secondo RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS o indicazione TLS presente), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Dalle notazioni di Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) ed Exim |
| `Id`, `For`, `Via` | Altri componenti di `Received` |
| `Date` | Timestamp dopo il punto e virgola, UTC |
| `Delay` | TimeSpan rispetto all'hop precedente; negativo in caso di sfasamento degli orologi |
| `Provider` | Provider o gateway rilevato dai nomi host (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast e altri) |
| `Attested` | `$true` solo per l'ultimo hop: solo questa riga è stata scritta dal sistema ricevente stesso, tutte quelle sotto erano già nel messaggio |
| `Raw` | La riga originale |

### AuthTrust: provenienza dei risultati di verifica

Una riga `Authentication-Results` può essere inserita in un messaggio da qualsiasi mittente, così come una riga `Received` corrispondente. Pertanto, dalla sola intestazione non è possibile comprovare chi abbia scritto una riga di verifica. Secondo RFC 8601, sezione 5, è determinante soltanto la riga il cui authserv-id è noto come proprio all'organizzazione ricevente, e il gateway in ingresso deve rimuovere le righe in entrata con questo ID. Il cmdlet implementa questa regola tramite `-TrustedAuthServId`. Senza questo parametro confronta l'authserv-id soltanto con gli host `by` della catena `Received` (stesso dominio o sottodominio, sempre al confine del punto, nessuna sottostringa); è una verifica di plausibilità.

| Valore | Significato |
|---|---|
| `Trusted` | L'authserv-id è presente in `-TrustedAuthServId`. Non appena esiste una riga simile, soltanto le righe di questo livello confluiscono in `Spf`, `Dkim`, `Dmarc` e `Arc`. Affidabile purché il gateway rimuova le righe esterne con questo ID |
| `Matched` | L'authserv-id compare come host `by` nella catena. Plausibile, ma non comprovato: una falsificazione può includere la riga `Received` corrispondente. Senza `-TrustedAuthServId`, il Finding `AuthPlausibleOnly` lo segnala. Le righe di questo livello vengono conteggiate se non è presente alcuna riga `Trusted` |
| `Unmatched` | L'authserv-id non è né attendibile né presente nella catena. I risultati vengono visualizzati, ma sono considerati un'affermazione non comprovata; il Finding `AuthUnverified` lo segnala |
| `Absent` | La riga non reca alcun authserv-id. Microsoft 365 scrive la propria riga di verifica in questa forma, che inizia direttamente con `spf=` |
| `None` | Nessuna riga di verifica presente |

Se l'intestazione contiene righe di verifica di più provenienze, il Finding `AuthMixedOrigins` segnala la situazione. Se manca una riga di verifica ma è presente una riga `Received-SPF`, il relativo risultato viene adottato come `Spf` e contrassegnato con `ReceivedSpfOnly`.

Con `-TrustedAuthServId` vengono aggiunti due Finding: `AuthNotTrusted`, se nessuna riga di verifica reca un ID attendibile, e `AuthTrustedConflict`, se due righe di questo tipo riportano risultati diversi per `spf`, `dmarc`, `arc` o `compauth`. Il secondo caso significa che almeno una riga non proviene dal gateway e che il gateway non l'ha rimossa. In questo caso il cmdlet adotta la riga più in alto; dall'intestazione non è possibile stabilire quale delle due sia autentica. DKIM è escluso da questo confronto, poiché più firme possono legittimamente avere risultati differenti.

### Catena ARC: struttura e giudizio

Per ARC il cmdlet fornisce due indicazioni separate. `Arc` è il risultato `arc=` dalla riga di verifica determinante, ossia il giudizio del destinatario che ha verificato le firme della catena. `ArcStructure` è la verifica del modulo e riguarda soltanto la struttura secondo RFC 8617: numeri di istanza consecutivi da `i=1` a `i=n` (al massimo 50), esattamente un `ARC-Seal`, un `ARC-Message-Signature` e un `ARC-Authentication-Results` per istanza, nonché `cv=none` nell'istanza 1 e `cv=pass` in tutte le successive. Il modulo non ricalcola le firme; pertanto `Consistent` non dice nulla sull'autenticità della catena. Le discrepanze sono riportate in `ArcStructureIssues` e nel Finding `ArcStructureInconsistent`.

Le indicazioni in `ARC-Authentication-Results` sono affermazioni del rispettivo inoltro. Il Finding `DkimBrokenAfterForward` classifica quindi un errore DKIM come conseguenza di un inoltro soltanto se il destinatario stesso segnala `arc=pass` e un'istanza precedente ha registrato un DKIM-`pass` per lo stesso dominio.

### Allineamento DMARC

`SpfAlignment` confronta il dominio del mittente envelope con il dominio `From`, mentre `DkimAlignment` confronta il dominio `d=` della firma verificata con il dominio `From`. `Strict` significa dominio identico, `Relaxed` lo stesso dominio organizzativo, `None` nessuna corrispondenza. Il dominio organizzativo viene determinato euristicamente: le ultime due etichette oppure, per finali composti noti quali `co.uk` o `com.au`, le ultime tre. Non è inclusa una Public Suffix List completa.

### MailHeaderAnalyzer.ExchangeClassification

Se l'intestazione contiene i campi `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` o `X-MS-Exchange-CrossTenant-*`, il cmdlet popola la proprietà `Exchange`. I significati seguono l'articolo «Demystifying hybrid mail flow» del team Exchange; il contesto è disponibile nell'articolo [Intestazioni ibride Exchange: interne o esterne?](/blog/exchange-hybrid-header-intern-extern).

| Proprietà | Contenuto |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` o `Incoming` con spiegazione |
| `AuthAs`, `AuthAsMeaning` | `Internal` o `Anonymous` con le conseguenze per il filtraggio EOP |
| `AuthSource` | Server che ha effettuato la classificazione |
| `AuthMechanism`, `AuthMechanismMeaning` | Codice del meccanismo. Solo il valore 10 (Externally Secured) è documentato pubblicamente; per tutti gli altri codici il modulo dichiara esplicitamente che Microsoft non li documenta |
| `OriginatorOrg` | Dominio predefinito del tenant mittente, caratteristica non falsificabile del tenant alla ricezione da Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Classificazione e ID tenant al confine del tenant |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` o `HybridOnPrem` con spiegazione |
| `WrongTenantAttribution` | Valore di `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` quando il messaggio è stato associato a un tenant esterno |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Marcatori per intestazioni dell'organizzazione ricevute o rimosse dal connettore di invio |

### MailHeaderAnalyzer.SpamAssessment

La proprietà `Spam` riassume le valutazioni dei filtri noti. I valori provengono da sistemi esterni e vengono decodificati, ma non valutati.

| Proprietà | Fonte | Contenuto |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level con significato (-1 attendibile, 0/1 non spam, 5/6 sospetto spam, 9 molto probabilmente spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level da 0 a 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Classificazione come `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Risultato del filtro come `NSPM`, `SPM`, `SKA` (allowlist), `SKI` (intraorganizzativo) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP nella allowlist di connessione) o `NLI` (nessuna reputazione) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | IP di invio e Paese di origine |
| `Forefront` | `X-Forefront-Antispam-Report` | Tutte le coppie chiave-valore del campo |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Punteggio e test attivati |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Simboli con punteggio |

L'elenco completo dei codici motivo `compauth` è disponibile nell'articolo [Microsoft 365 compauth: codici motivo](/blog/microsoft-365-compauth-reason-codes).

### Finding

Il cmdlet fornisce le anomalie come oggetti in `Findings`, ciascuno con `Severity` (`Info`, `Warning`, `Fail`), un `Code` stabile per filtri e script, nonché una spiegazione in `Message`.

| Codice | Gravità | Significato |
|---|---|---|
| `DuplicateField` | Warning | Un campo che RFC 5322 limita a una sola istanza (`From`, `Subject`, `Date`, `Message-ID` e altri) compare più volte. Client di posta e filtri potrebbero scegliere istanze diverse; un modello noto nelle falsificazioni |
| `BidiControls` | Warning | Caratteri di controllo Unicode per la direzione di scrittura in un campo. Invertendo la direzione di lettura, `fdp.exe` viene visualizzato come `exe.pdf`. Il modulo li mostra come `<U+202E>` |
| `HopOverflow` | Warning | Più di 200 righe `Received`; quelle eccedenti non sono state analizzate |
| `AuthPlausibleOnly` | Info | L'authserv-id compare nella catena di consegna, ma `-TrustedAuthServId` non è stato specificato: plausibile, non comprovato |
| `AuthNotTrusted` | Warning | È stato specificato `-TrustedAuthServId`, ma nessuna riga di verifica reca uno di questi ID |
| `AuthTrustedConflict` | Warning | Due righe di verifica con ID attendibile riportano risultati diversi per lo stesso metodo; apparentemente il gateway non rimuove le righe esterne |
| `AuthUnverified` | Warning | I risultati di verifica recano un authserv-id che non è né attendibile né presente nella catena di consegna |
| `AuthMixedOrigins` | Warning | Presenti righe di verifica di più provenienze |
| `ReceivedSpfForeign` | Warning | Il valore `receiver=` della riga `Received-SPF` non compare nella catena |
| `ReceivedSpfOnly` | Info | Il risultato SPF proviene soltanto da `Received-SPF`, non da una riga di verifica |
| `NoAuthResults` | Info | Nessun risultato di verifica nell'intestazione |
| `DmarcFail` | Fail | DMARC non superato secondo il server ricevente |
| `SpfNotPass` | Warning | Risultato SPF `fail`, `softfail`, `permerror` o `temperror` |
| `DkimNotPass` | Warning | Risultato DKIM `fail`, `permerror` o `temperror`, senza che il destinatario confermi la catena ARC |
| `DkimBrokenAfterForward` | Info | DKIM non superato presso il destinatario, ma il destinatario segnala `arc=pass` e un'istanza ARC precedente ha registrato un DKIM-`pass` per lo stesso dominio: tipico di inoltri e mailing list |
| `DkimWeakHash` | Warning | Firma con `rsa-sha1` (RFC 8301 classifica SHA-1 come obsoleto) |
| `DkimBodyLength` | Warning | Il tag `l=` limita la lunghezza firmata del corpo del messaggio; il contenuto aggiunto non è coperto |
| `DkimExpired` | Warning | L'istante `x=` è nel passato |
| `DkimFromUnsigned` | Warning | Il campo `From` non è incluso in `h=`, sebbene RFC 6376 lo richieda |
| `ArcStructureInconsistent` | Warning | Le intestazioni ARC non formano una catena formalmente completa (lacune, intestazioni mancanti o duplicate, sequenza `cv=` errata); solo verifica della struttura |
| `ClockSkew` | Info | Un hop ha un timestamp precedente al suo predecessore; i ritardi sono solo valori approssimativi |
| `ReplyToMismatch` | Info | Il dominio `Reply-To` differisce dal dominio `From`; comune nelle newsletter, ma un modello nel phishing |
| `SpfNotAligned` | Info | Il dominio del mittente envelope e il dominio `From` appartengono a organizzazioni diverse; SPF non contribuisce quindi a DMARC |
| `ExchangeWrongTenant` | Warning | Il messaggio è stato associato a un tenant esterno; causa classica è un connettore in ingresso di un altro tenant con lo stesso certificato o gli stessi indirizzi IP |
| `ExchangeHeadersFiltered` | Warning | Il connettore di invio ha rimosso le intestazioni Cross-Premises (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: ingresso tramite un connettore di ricezione con «Externally Secured», filtraggio EOP ignorato |
| `SpamCategory` | Warning | Microsoft ha assegnato una categoria diversa da `NONE` |
| `SpamConfidence` | Warning | SCL 5 o superiore |

## Funzionamento e limiti

Il modulo legge ciò che è presente nell'intestazione e ne deduce ciò che è possibile comprovare senza query esterne. Ne derivano alcuni limiti:

- **Nessuna verifica crittografica.** Le firme DKIM e ARC non vengono ricalcolate e non vengono interrogati record DNS. `Spf`, `Dkim`, `Dmarc` e `Arc` sono sempre il giudizio del server ricevente; `ArcStructure` verifica solo la struttura della catena.
- **Provenienza comprovabile solo con conoscenza del gateway.** Senza `-TrustedAuthServId`, `AuthTrust` è una verifica di plausibilità. Con il parametro, l'affermazione dipende dal fatto che il gateway rimuova le righe di verifica esterne con il proprio authserv-id; il modulo non può verificarlo.
- **Solo l'ultima riga `Received` è comprovata.** Tutte le righe sottostanti sono state fornite dal mittente e possono essere modellate arbitrariamente. `Attested` contrassegna questa differenza; i ritardi degli hop precedenti si basano sulle indicazioni di queste righe.
- **Domini organizzativi euristici.** Per l'allineamento Relaxed, il modulo usa un breve elenco di finali composti e non una Public Suffix List completa.
- **AuthMechanism documentato solo in parte.** Oltre al valore 10, Microsoft non ha pubblicato i codici; il modulo non ne inventa i significati.
- **Set di caratteri.** I valori RFC 2047 sono decodificati con le codifiche note a .NET sul sistema specifico. I set di caratteri sconosciuti rimangono invariati.

## Protezione dei dati

Un'intestazione completa contiene nomi host interni, indirizzi IP, mittenti, destinatari e oggetto. Il motivo per cui questi dati non dovrebbero essere inseriti in uno strumento online è spiegato nell'articolo [Analizzare intestazioni e-mail senza caricare l'e-mail](/blog/e-mail-header-analysieren-ohne-upload). Per il modulo vale la stessa promessa della versione browser: nessun accesso alla rete. La suite di test del modulo contiene un test che fallisce non appena nel codice sorgente compare un cmdlet di rete o DNS.

## Codice sorgente e versioni

Il codice sorgente è disponibile con licenza MIT su [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). La logica di analisi è un porting della libreria usata anche dall'analizzatore di intestazioni di questo sito; entrambi condividono i casi di test. L'integrazione continua verifica ogni modifica con PSScriptAnalyzer e Pester su Windows PowerShell 5.1, PowerShell 7 in Windows, Ubuntu e macOS. Le pubblicazioni nella PowerShell Gallery avvengono automaticamente da tag versionati; le modifiche per versione sono riportate nel [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

La versione 0.2.0 del 26 settembre 2026 ha rafforzato il modello di provenienza in seguito a un'osservazione durante una revisione di @saltyslugga: parametro `-TrustedAuthServId`, livello `Trusted`, `Matched` solo come plausibilità. La proprietà `ArcValid` è stata rimossa e sostituita da `ArcStructure` e `ArcStructureIssues`; gli script che analizzano `ArcValid` devono essere adattati.

Segnalazioni di errori e richieste di miglioramento sono accettate come [issue su GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonimizzare gli header di test prima di segnalarli; i casi di test inclusi usano esclusivamente domini di esempio secondo RFC 2606 e indirizzi secondo RFC 5737.

## Cronologia delle modifiche

- **25 ottobre 2026:** sostituita la fonte RFC 7489 con RFC 9989, che sostituisce la specifica DMARC dal maggio 2026.

## Fonti

1.  [MailHeaderAnalyzer nella PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): pagina del pacchetto con comando di installazione e cronologia delle versioni.

2.  [pfstr/MailHeaderAnalyzer su GitHub](https://github.com/pfstr/MailHeaderAnalyzer): codice sorgente, suite di test, changelog e issue.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): modello per la struttura di questo riferimento (sintassi, descrizione, esempi, proprietà dei parametri).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): struttura di `Authentication-Results` e regola secondo cui è determinante soltanto la riga dell'organizzazione ricevente (sezione 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): struttura delle righe `Received` e raccomandazione di un limite massimo come protezione dai loop (sezione 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): i valori `with` `ESMTPS`, `ESMTPA` e `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): tag di `DKIM-Signature`, obbligo di firmare `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): classificazione di `rsa-sha1` come obsoleto.

9.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): allineamento Strict e Relaxed; sostituisce RFC 7489 dal maggio 2026.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): struttura della catena composta da `ARC-Seal`, `ARC-Message-Signature` e `ARC-Authentication-Results`, numeri di istanza e valori `cv=`.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): significato di SCL, BCL, CAT, SFV, IPV e codici motivo `compauth`.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): origine dei significati di MessageDirectionality, AuthAs e AuthMechanism.

13.  [Analizzatore di intestazioni su rafaelpfister.ch](/tools/header-analyzer): la versione browser con la stessa logica di analisi.
