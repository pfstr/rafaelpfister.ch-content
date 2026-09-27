---
title: "MailHeaderAnalyzer: analyser e-posthoder i PowerShell uten nettverkstilgang"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referanse for PowerShell-modulen MailHeaderAnalyzer: parameteroversikt, syntaks, beskrivelse, eksempler og parameterattributter for Get-MailHeaderAnalysis og ConvertTo-MailHeaderReport, samt utdataobjektet med leveringskjede, autentiseringsresultater, Exchange Online-klassifisering og alle funnkoder."
date: "2026-09-24"
kategorie: "SMTP og e-postflyt"
timeToRead: "16 min lesetid"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
  - "uebergreifend"
protokolle:
  - "powershell"
  - "mail-auth"
  - "smtp"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
slug: "mailheaderanalyzer-analyse-av-e-posthoder-i-powershell-uten-nettverkstilgang"
translationId: "article-041d7b2f9615f670"
translationOf: mailheaderanalyzer-powershell-modul
translationSourceHash: 41c3cb7c83800c0cc30f797ad6b2d9ec480d6c32bc054d2e01bd037b3dcd6c9b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T09:11:28.013Z
translationReview: automatic
url: https://rafaelpfister.ch/no/blog/mailheaderanalyzer-analyse-av-e-posthoder-i-powershell-uten-nettverkstilgang
---

# MailHeaderAnalyzer: analyser e-posthoder i PowerShell uten nettverkstilgang

MailHeaderAnalyzer er en PowerShell-modul med to cmdleter. `Get-MailHeaderAnalysis` analyserer hodet til en e-post: leveringskjeden med forsinkelser og TLS-informasjon, SPF-, DKIM-, DMARC- og ARC-resultater inkludert opprinnelseskontroll mot authserv-id-en til gatewayen din, DMARC-alignment, hybridklassifiseringen til Exchange Online, vurderingene fra Microsoft Defender, SpamAssassin og Rspamd samt avvik som doble `From`-linjer eller Unicode-kontrolltegn. `ConvertTo-MailHeaderReport` oppretter en rapport for saker. Modulen arbeider helt offline: ingen DNS-oppslag, ingen HTTP-forbindelser. Den er kommandolinjeversjonen av [Header Analyzer på dette nettstedet](/tools/header-analyzer) og bruker samme analyselogikk.

| | |
|---|---|
| Modul | [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Kildekode | [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT-lisens |
| Gjelder for | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdleter | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameteroversikt

| Cmdlet | Parameter | Type | Påkrevd | Pipeline | Virkning |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Ja (parametersett Text) | Ja, etter verdi | Hodet som tekst. Linjer fra pipelinen settes sammen til ett hode |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Ja (parametersett Path) | Ja, etter egenskapsnavn | Fil med hodet eller en fullstendig `.eml`-melding; tar imot objekter fra `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Ja (parametersett Clipboard) | Nei | Leser hodet fra utklippstavlen (kun Windows) |
| `Get-MailHeaderAnalysis` | `-TrustedAuthServId` | `String[]` | Nei | Nei | authserv-id(-er) for inngangsgatewayen din; bare kontrollinjer med en av disse ID-ene anses som dokumenterte |
| `ConvertTo-MailHeaderReport` | `-Analysis` | `MailHeaderAnalyzer.Analysis` | Ja | Ja, etter verdi | Resultatobjektet fra `Get-MailHeaderAnalysis` |
| `ConvertTo-MailHeaderReport` | `-Format` | `String` | Nei | Nei | `Markdown` (standard) eller `Text` |

Begge cmdletene støtter Common Parameters `-Verbose`, `-ErrorAction`, `-ErrorVariable`, `-OutVariable` og de øvrige fra [about_CommonParameters](https://learn.microsoft.com/de-de/powershell/module/microsoft.powershell.core/about/about_commonparameters).

## Installasjon

```powershell
Install-Module -Name MailHeaderAnalyzer -Scope CurrentUser
```

| Alternativ | Virkning |
|---|---|
| `-Name MailHeaderAnalyzer` | Navnet på modulen i PowerShell Gallery |
| `-Scope CurrentUser` | Installerer i brukerens modulkatalog uten administratorrettigheter |

På et system uten internettilgang laster du ned modulen på en annen maskin med `Save-Module -Name MailHeaderAnalyzer -Path C:\Temp` og kopierer mappen `MailHeaderAnalyzer` til en katalog fra `$env:PSModulePath`, for eksempel `%USERPROFILE%\Documents\WindowsPowerShell\Modules` (Windows PowerShell 5.1) eller `%USERPROFILE%\Documents\PowerShell\Modules` (PowerShell 7). Oppdateringer hentes med `Update-Module -Name MailHeaderAnalyzer`, og den installerte versjonen vises med `Get-Module -Name MailHeaderAnalyzer -ListAvailable`.

## Get-MailHeaderAnalysis

Analyserer hodet til en e-post og returnerer et analyseobjekt.

### Syntaks

#### Text (standard)

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

### Beskrivelse

Cmdleten deler råhodet inn i felt, folder ut RFC 5322-foldinger og dekoder RFC 2047-verdier i emne og adresser. Fra `Received`-linjene oppretter den leveringskjeden i kronologisk rekkefølge, beregner forsinkelsen per stasjon og leser TLS-versjon, chiffer og protokollklasse i henhold til RFC 3848. Fra `Authentication-Results`, `Received-SPF`, `DKIM-Signature` og ARC-kjeden fastslår den autentiseringsresultatene og tilordner hver kontrollinje en opprinnelse (RFC 8601, avsnitt 5): dokumentert dersom authserv-id-en finnes i `-TrustedAuthServId`, ellers bare plausibel eller udokumentert, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse). I tillegg kommer DMARC-alignment, hybridklassifiseringen til Exchange Online, vurderingene fra spamfiltrene og en liste over avvik.

Cmdleten utfører ingen DNS-oppslag og åpner ingen nettverksforbindelse. `Spf`, `Dkim`, `Dmarc` og `Arc` er derfor alltid mottaksserverens vurdering. DKIM- og ARC-signaturer blir ikke kryptografisk verifisert på nytt; for ARC-kjeden kontrollerer cmdleten bare strukturen (`ArcStructure`).

Inndata leses tolerant: En tom linje avslutter hodet, og påfølgende meldingstekst ignoreres. Linjer uten feltnavn og uten innledende mellomrom, slik de oppstår ved kopiering fra klientdialoger, hører til det foregående feltet. En mbox-skillelinje `From ...` før det første feltet hoppes over, og en Byte Order Mark fjernes. Høyst 200 `Received`-linjer analyseres, telt fra leveringen.

### Eksempler

#### Eksempel 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyserer hodet som ligger på utklippstavlen. I Outlook for Windows finner du hodet under Fil, Egenskaper, Internett-hoder; i Outlook på nettet finner du det i meldingsalternativene under «Vis meldingsdetaljer».

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

#### Eksempel 2

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml
```

Analyserer en lagret melding. Filen kan inneholde bare hodet eller hele meldingen; meldingsteksten ignoreres.

#### Eksempel 3

```powershell
(Get-MailHeaderAnalysis -Path .\nachricht.eml).Hops
```

Viser leveringskjeden som en tabell, med første hopp først.

```text
#   From                 IP              By                                     Protocol   TLS      Time (UTC)           Delay
-   ----                 --              --                                     --------   ---      ----------           -----
1   client.example.net   198.51.100.34   mail.example.org                       ESMTPSA    -        2026-08-03 09:14:28  -
2   mail.example.org     203.0.113.25    mx.eur02.prod.protection.outlook.com   Microsoft… TLS 1.3  2026-08-03 09:15:09  41 s
3   AM0EUR02FT056.eop…   -               ZR0P278MB0570.CHEP278.PROD.OUTLOOK.COM Microsoft… TLS 1.2  2026-08-03 09:15:10  1 s
```

#### Eksempel 4

```powershell
Get-Content -Path .\header.txt | Get-MailHeaderAnalysis | Select-Object -ExpandProperty Findings
```

Leser hodet linje for linje fra en tekstfil og viser bare avvikene. `-Raw` med `Get-Content` er ikke nødvendig; cmdleten setter selv sammen linjene.

#### Eksempel 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analyserer alle meldinger i en mappe og skriver en CSV-fil med én linje per melding. Den beregnede kolonnen sammenfatter funnkodene.

| Alternativ | Virkning |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Leverer filobjektene; `-Path` overtar egenskapen `FullName` |
| `Select-Object … @{ Name; Expression }` | Beregnet kolonne som sammenfatter alle funnkodene til én tekststreng |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV uten typeoverskrift, UTF-8 for norske tegn i emnelinjer |

#### Eksempel 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis -TrustedAuthServId 'mx.example.org' |
    Where-Object AuthTrust -ne 'Trusted' |
    Select-Object Source, AuthTrust, AuthServId, DeliveredBy
```

Lister meldinger hvis kontrollresultater ikke kommer fra din egen inngangsgateway `mx.example.org`. Ved phishing-analyser er dette et første filter. Uten `-TrustedAuthServId` kan du bare filtrere etter `Unmatched`; en forfalskning som i tillegg inneholder en passende `Received`-linje, vises da som `Matched` og slipper gjennom filteret.

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-TrustedAuthServId 'mx.example.org'` | authserv-id som gatewayen din skriver i `Authentication-Results`; eksakt sammenligning, uten underdomener |
| `Where-Object AuthTrust -ne 'Trusted'` | Beholder alle meldinger der den relevante kontrollinjen ikke har en klarert authserv-id |
| `Select-Object Source, AuthTrust, AuthServId, DeliveredBy` | Fil, opprinnelsesnivå, authserv-id for kontrollinjen og leverende stasjon |

</details>

#### Eksempel 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Viser stasjonen med størst forsinkelse. `SlowestHopIndex` er 1-basert, mens matrisen `Hops` er 0-basert.

#### Eksempel 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Skriver hele analysen som JSON. `-Depth 6` er nødvendig fordi `Hops`, `DkimSignatures` og `Exchange` er nestede objekter; standardverdien 2 ville bare vist dem som typenavn.

### Parametere

#### -Header

Hodet som tekst. Parameteren tar imot én enkelt streng med hele hodet eller flere strenger; pipeline-inndata samles og settes sammen til ett hode til slutt, derfor fungerer `Get-Content datei | Get-MailHeaderAnalysis` uten `-Raw`. Hvis flere hoder skal analyseres separat, bruker du `-Path` med flere filer.

Aliaser: `Text`, `Raw`, `InputObject`

| Parameteregenskap | Verdi |
|---|---|
| Type | `String[]` |
| Standardverdi | Ingen |
| Støtter jokertegn | Nei |

| Parametersett Text | Verdi |
|---|---|
| Posisjon | 0 |
| Påkrevd | Ja |
| Verdi fra pipeline | Ja |
| Verdi fra pipeline etter egenskapsnavn | Nei |

#### -Path

Sti til en fil som inneholder hodet eller en fullstendig `.eml`-melding. Relative stier løses opp mot gjeldende katalog. Filen leses med `[System.IO.File]::ReadAllText`: En Byte Order Mark tas hensyn til, og uten BOM brukes UTF-8. Det opprettes et eget resultatobjekt for hver fil; manglende filer gir en ikke-terminerende feil.

Aliaser: `FullName`, `PSPath`, `LiteralPath`

| Parameteregenskap | Verdi |
|---|---|
| Type | `String[]` |
| Standardverdi | Ingen |
| Støtter jokertegn | Nei |

| Parametersett Path | Verdi |
|---|---|
| Posisjon | Navngitt |
| Påkrevd | Ja |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Ja |

#### -FromClipboard

Leser hodet fra utklippstavlen med `Get-Clipboard -Raw`. Parameteren er bare tilgjengelig i Windows; på Linux og macOS avslutter cmdleten med en feilmelding, det samme gjelder ved tom utklippstavle.

| Parameteregenskap | Verdi |
|---|---|
| Type | `SwitchParameter` |
| Standardverdi | `False` |
| Støtter jokertegn | Nei |

| Parametersett Clipboard | Verdi |
|---|---|
| Posisjon | Navngitt |
| Påkrevd | Ja |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Nei |

#### -TrustedAuthServId

Authserv-id-en eller authserv-id-ene som inngangsgatewayen din skriver i `Authentication-Results`, for eksempel `mx.example.org`. Cmdleten sammenligner eksakt, uten hensyn til store og små bokstaver og uten underdomener. Kontrollinjer med en av disse ID-ene får `AuthTrust = Trusted`, og bare de inngår deretter i `Spf`, `Dkim`, `Dmarc` og `Arc`. Det samme gjelder `receiver=` i en `Received-SPF`-linje.

Resultatet er bare så pålitelig som gatewayen: Den må fjerne innkommende `Authentication-Results`-linjer som påberoper seg dens egen authserv-id (RFC 8601, avsnitt 5). Om den gjør det, kan ikke sees i et hode. Hvis to linjer med klarert ID motsier hverandre, rapporterer cmdleten `AuthTrustedConflict`. Microsoft 365 skriver ingen authserv-id i sin kontrollinje; dersom Exchange Online mottar e-posten, lar du parameteren være borte.

For en hel økt kan verdien lagres som standard, for eksempel i PowerShell-profilen:

```powershell
$PSDefaultParameterValues['Get-MailHeaderAnalysis:TrustedAuthServId'] = 'mx.example.org'
```

| Parameteregenskap | Verdi |
|---|---|
| Type | `String[]` |
| Standardverdi | Ingen |
| Støtter jokertegn | Nei |

| Parametersett (alle) | Verdi |
|---|---|
| Posisjon | Navngitt |
| Påkrevd | Nei |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Nei |

### Inndata

`System.String`: Hodelinjer eller hele hodet, til `-Header`.

`System.IO.FileInfo`: Filobjekter fra `Get-ChildItem`, der `FullName` bindes til `-Path`.

### Utdata

`MailHeaderAnalyzer.Analysis`: ett objekt per analysert hode. Egenskapene beskrives i avsnittet [Utdataobjekt](#ausgabeobjekt).

### Merknader

Analyseteksten (forklaringer i `Findings`, `CompAuthReasonMeaning` og betydningsfeltene) er på engelsk, slik at den kan brukes uendret i internasjonale saker. Utdata i standardvisningen kan vises fullstendig med `Format-List *`.

## ConvertTo-MailHeaderReport

Oppretter en rapport som Markdown eller tekst fra et analyseobjekt.

### Syntaks

```powershell
ConvertTo-MailHeaderReport
    [-Analysis] <Object>
    [-Format <String>]
    [<CommonParameters>]
```

### Beskrivelse

Rapporten omfatter emne, avsender, dato og Message-ID, autentiseringsresultatene med opprinnelsesangivelse og alignment, alle funn, leveringskjeden, Exchange-klassifiseringen og verdiene fra spamfiltrene. Unicode-kontrolltegn for skriveretning forblir synlige i rapporten som `<U+...>`, slik at de ikke overføres til et saksbehandlingssystem via rapporten. Den siste linjen angir modulversjonen.

### Eksempler

#### Eksempel 1

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-MailHeaderReport | Set-Clipboard
```

Oppretter en Markdown-rapport og legger den på utklippstavlen.

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

#### Eksempel 2

```powershell
Get-MailHeaderAnalysis -FromClipboard | ConvertTo-MailHeaderReport -Format Text
```

Skriver ut rapporten som tekst uten Markdown-formatering, for eksempel for e-post eller konsolllogger.

#### Eksempel 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Bruk i Exchange Management Shell. Dersom `$env:PSModulePath` der er begrenset av gruppepolicyer, laster du modulen med `Import-Module` og den fullstendige stien til `.psd1`-filen.

### Parametere

#### -Analysis

Analyseobjektet fra `Get-MailHeaderAnalysis`. Cmdleten avviser andre objekttyper med en bindingsfeil.

| Parameteregenskap | Verdi |
|---|---|
| Type | `MailHeaderAnalyzer.Analysis` |
| Standardverdi | Ingen |
| Støtter jokertegn | Nei |

| Parametersett (alle) | Verdi |
|---|---|
| Posisjon | 0 |
| Påkrevd | Ja |
| Verdi fra pipeline | Ja |
| Verdi fra pipeline etter egenskapsnavn | Nei |

#### -Format

Utdataformatet. Gyldige verdier:

- `Markdown`: Overskrifter, punktlister og leveringskjeden som tabell. Standard.
- `Text`: Overskrifter med store bokstaver, innrykkede linjer, leveringskjeden som nummerert liste.

| Parameteregenskap | Verdi |
|---|---|
| Type | `String` |
| Tillatte verdier | `Markdown`, `Text` |
| Standardverdi | `Markdown` |
| Støtter jokertegn | Nei |

| Parametersett (alle) | Verdi |
|---|---|
| Posisjon | Navngitt |
| Påkrevd | Nei |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Nei |

### Inndata

`MailHeaderAnalyzer.Analysis`: resultatet fra `Get-MailHeaderAnalysis`.

### Utdata

`System.String`: rapporten, én streng per analyseobjekt.

## Utdataobjekt

`Get-MailHeaderAnalysis` returnerer ett objekt av typen `MailHeaderAnalyzer.Analysis` per inndata. Standardvisningen viser sammendraget fra eksempel 1; alle egenskaper er tilgjengelige via `Select-Object`, `Format-List *` eller `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Egenskap | Type | Innhold |
|---|---|---|
| `Source` | String | Filsti, `Clipboard` eller `Text` |
| `Subject` | String | Emne, RFC 2047-dekodet |
| `From`, `ReplyTo`, `ReturnPath` | Adresseobjekt | `Name`, `Address`, `Domain`, `Display`; `$null` dersom feltet mangler |
| `Date` | DateTime (UTC) | Verdien av feltet `Date` |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Konvoluttavsenderdomene fra `smtp.mailfrom` i SPF-kontrollen, ellers fra `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultat ifølge den relevante `Authentication-Results`-linjen (`pass`, `fail`, `none`, `softfail` og flere); `$null` dersom ikke kontrollert. `Arc` er mottakerens vurdering av ARC-kjeden |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason-kode for den sammensatte autentiseringen i Microsoft 365 og dens betydning |
| `AuthTrust` | String | `Trusted`, `Matched`, `Unmatched`, `Absent` eller `None`, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id for den relevante kontrollinjen |
| `AuthenticationResults` | Objekt[] | Alle `Authentication-Results`-linjer med `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | `Received-SPF`-linjen med `Result` og `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` eller `$null` |
| `Hops` | Hop[] | Leveringskjede i kronologisk rekkefølge, se [Hop-objekt](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Nøkkeltall for kjeden |
| `DeliveredBy` | String | `by`-verten i den nyeste `Received`-linjen, altså den leverende stasjonen |
| `DkimSignatures` | Objekt[] | Per signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain` | Objekt[] | ARC-instanser med `Instance`, `SealDomain`, `ChainValidation`, `Methods` |
| `ArcStructure`, `ArcStructureIssues` | String, String[] | Struktur for ARC-kjeden: `Consistent`, `Inconsistent` eller `$null` uten ARC-hoder, samt avvikene som ble funnet. Kun strukturkontroll, ingen signaturkontroll, se [ARC-kjede](#arc-kette-aufbau-und-urteil) |
| `Exchange` | Objekt | Hybridklassifisering fra Exchange Online, se [Exchange-objekt](#mailheaderanalyzerexchangeclassification); `$null` uten tilsvarende hoder |
| `Spam` | Objekt | Vurderinger fra spamfiltrene, se [spamobjekt](#mailheaderanalyzerspamassessment); `$null` uten tilsvarende hoder |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` uten listehoder |
| `Findings` | Finding[] | Avvik med `Severity`, `Code`, `Message`, se [Funn](#findings) |
| `Fields` | Objekt[] | Alle felt med `Name`, `Value` (utfoldet) og `Raw` |
| `HadBody` | Bool | Om meldingstekst fulgte etter hodet |

### MailHeaderAnalyzer.Hop

Hver oppføring i `Hops` tilsvarer en `Received`-linje. Rekkefølgen er kronologisk, altså omvendt av rekkefølgen i hodet.

| Egenskap | Innhold |
|---|---|
| `Index` | Løpenummer, 1 = innlevering |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Opplysninger om det innleverende systemet fra `from`-delen; IP-adressen kommer fra hakeparentesen i kommentaren, rDNS-navnet fra kommentaren før den |
| `ByHost`, `Software` | Mottakende system og programvaren det bruker (kommentar etter `by`) |
| `Protocol`, `ProtocolClass` | `with`-verdi og klasse etter RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS eller TLS-informasjon finnes), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Fra skrivemåtene til Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) og Exim |
| `Id`, `For`, `Via` | Andre `Received`-bestanddeler |
| `Date` | Tidsstempel etter semikolonet, UTC |
| `Delay` | TimeSpan til forrige hopp; negativ ved klokkeavvik |
| `Provider` | Gjenkjent leverandør eller gateway basert på vertsnavnene (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast og flere) |
| `Attested` | `$true` kun for siste hopp: Bare denne linjen er skrevet av mottakersystemet selv, alle nedenfor fantes allerede i meldingen |
| `Raw` | Originallinjen |

### AuthTrust: Opprinnelse til kontrollresultatene

En `Authentication-Results`-linje kan enhver avsender selv skrive inn i en melding, det samme kan gjøres med en passende `Received`-linje. Fra hodet alene kan det derfor ikke dokumenteres hvem som skrev en kontrollinje. I henhold til RFC 8601, avsnitt 5, er bare linjen relevant dersom authserv-id-en er kjent som mottaksorganisasjonens egen, og inngangsgatewayen må fjerne innkommende linjer med denne ID-en. Cmdleten modellerer denne regelen med `-TrustedAuthServId`. Uten denne parameteren sammenligner den authserv-id-en bare med `by`-vertene i `Received`-kjeden (samme domene eller underdomene, alltid ved punktgrensen, ingen delstreng); dette er en plausibilitetskontroll.

| Verdi | Betydning |
|---|---|
| `Trusted` | Authserv-id-en finnes i `-TrustedAuthServId`. Så snart en slik linje finnes, inngår bare linjer på dette nivået i `Spf`, `Dkim`, `Dmarc` og `Arc`. Pålitelig forutsatt at gatewayen fjerner fremmede linjer med denne ID-en |
| `Matched` | Authserv-id-en forekommer som `by`-vert i kjeden. Plausibelt, men ikke dokumentert: En forfalskning kan inneholde den passende `Received`-linjen. Uten `-TrustedAuthServId` peker funnet `AuthPlausibleOnly` på dette. Linjer på dette nivået teller dersom ingen `Trusted`-linje finnes |
| `Unmatched` | Authserv-id-en er verken klarert eller finnes i kjeden. Resultatene vises, men anses som udokumenterte påstander; funnet `AuthUnverified` peker på dette |
| `Absent` | Linjen mangler authserv-id. Microsoft 365 skriver sin kontrollinje på denne formen; den begynner direkte med `spf=` |
| `None` | Ingen kontrollinje finnes |

Dersom hodet inneholder kontrollinjer med flere opprinnelser, rapporterer funnet `AuthMixedOrigins` forholdet. Mangler en kontrollinje, men en `Received-SPF`-linje finnes, overtas resultatet som `Spf` og merkes med `ReceivedSpfOnly`.

Med `-TrustedAuthServId` kommer to funn i tillegg: `AuthNotTrusted`, dersom ingen kontrollinje har en klarert ID, og `AuthTrustedConflict`, dersom to slike linjer rapporterer ulike resultater for `spf`, `dmarc`, `arc` eller `compauth`. Det andre tilfellet betyr at minst én linje ikke stammer fra gatewayen og at gatewayen ikke har fjernet den. Cmdleten overtar i dette tilfellet den øverste linjen; hvilken av de to som er ekte, kan ikke avgjøres ut fra hodet. DKIM er unntatt fra denne sammenligningen, siden flere signaturer legitimt kan ha ulike resultater.

### ARC-kjede: struktur og vurdering

For ARC leverer cmdleten to separate opplysninger. `Arc` er resultatet `arc=` fra den relevante kontrollinjen, altså vurderingen fra mottakeren som har kontrollert kjedens signaturer. `ArcStructure` er modulens egen kontroll og gjelder bare strukturen i henhold til RFC 8617: fortløpende instansnumre `i=1` til `i=n` (høyst 50), nøyaktig én `ARC-Seal`, én `ARC-Message-Signature` og én `ARC-Authentication-Results` per instans, samt `cv=none` ved instans 1 og `cv=pass` ved alle de øvrige. Modulen beregner ikke signaturene på nytt; `Consistent` sier derfor ingenting om hvorvidt kjeden er ekte. Avvikene står i `ArcStructureIssues` og i funnet `ArcStructureInconsistent`.

Opplysningene i `ARC-Authentication-Results` er påstander fra den aktuelle videresendingen. Funnet `DkimBrokenAfterForward` klassifiserer derfor bare en DKIM-feil som følge av videresending når mottakeren selv rapporterer `arc=pass` og en tidligere instans har registrert en DKIM-`pass` for samme domene.

### DMARC-alignment

`SpfAlignment` sammenligner konvoluttavsenderdomenet med `From`-domenet, og `DkimAlignment` sammenligner `d=`-domenet til den kontrollerte signaturen med `From`-domenet. `Strict` betyr identisk domene, `Relaxed` samme organisasjonsdomene, `None` ingen samsvar. Organisasjonsdomenet bestemmes heuristisk: De to siste etikettene, eller de tre siste ved kjente sammensatte endelser som `co.uk` eller `com.au`. En fullstendig Public Suffix List er ikke inkludert.

### MailHeaderAnalyzer.ExchangeClassification

Dersom hodet inneholder feltene `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` eller `X-MS-Exchange-CrossTenant-*`, fyller cmdleten ut egenskapen `Exchange`. Betydningene følger Exchange-teamets artikkel «Demystifying hybrid mail flow»; bakgrunnen beskrives i artikkelen [Exchange-hybridhoder: internt eller eksternt?](/blog/exchange-hybrid-header-intern-extern).

| Egenskap | Innhold |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` eller `Incoming` med forklaring |
| `AuthAs`, `AuthAsMeaning` | `Internal` eller `Anonymous` med konsekvensene for EOP-filtrering |
| `AuthSource` | Serveren som foretok klassifiseringen |
| `AuthMechanism`, `AuthMechanismMeaning` | Mekanismekode. Bare verdien 10 (Externally Secured) er offentlig dokumentert; for alle andre koder opplyser modulen uttrykkelig at Microsoft ikke dokumenterer dem |
| `OriginatorOrg` | Standarddomene for den sendende tenanten, det ikke-forfalskbare tenantkjennetegnet ved mottak fra Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Klassifisering og tenant-ID ved tenantgrensen |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` eller `HybridOnPrem` med forklaring |
| `WrongTenantAttribution` | Verdien av `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` dersom meldingen ble tilordnet en fremmed tenant |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markører for mottatte organisasjonshoder, eller hoder fjernet av sendekoblingen |

### MailHeaderAnalyzer.SpamAssessment

Egenskapen `Spam` sammenfatter vurderingene fra de kjente filtrene. Verdiene kommer fra eksterne systemer og dekodes, men vurderes ikke.

| Egenskap | Kilde | Innhold |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level med betydning (-1 klarert, 0/1 ikke spam, 5/6 mistanke om spam, 9 svært sannsynlig spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 til 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Klassifisering som `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filterresultat som `NSPM`, `SPM`, `SKA` (tillatelsesliste), `SKI` (intraorganisatorisk) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP på tillatelsesliste for forbindelser) eller `NLI` (intet omdømme) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Innleverende IP og opprinnelsesland |
| `Forefront` | `X-Forefront-Antispam-Report` | Alle nøkkel-verdi-par i feltet |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Poengsum og utløste tester |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symboler med poengsum |

Den fullstendige listen over `compauth`-Reason-koder finnes i artikkelen [Microsoft 365 compauth: Reason-koder](/blog/microsoft-365-compauth-reason-codes).

### Funn

Cmdleten leverer avvik som objekter i `Findings`, hver med `Severity` (`Info`, `Warning`, `Fail`), en stabil `Code` for filtre og skript samt en forklaring i `Message`.

| Kode | Alvorlighetsgrad | Betydning |
|---|---|---|
| `DuplicateField` | Warning | Et felt som RFC 5322 begrenser til én instans (`From`, `Subject`, `Date`, `Message-ID` og flere), forekommer flere ganger. E-postklienter og filtre kan velge ulike instanser; et kjent mønster ved forfalskninger |
| `BidiControls` | Warning | Unicode-kontrolltegn for skriveretning i et felt. De snur leseretningen, `fdp.exe` vises da som `exe.pdf`. Modulen viser dem som `<U+202E>` |
| `HopOverflow` | Warning | Flere enn 200 `Received`-linjer; de overskytende ble ikke analysert |
| `AuthPlausibleOnly` | Info | Authserv-id-en forekommer i leveringskjeden, men `-TrustedAuthServId` ble ikke angitt: plausibelt, ikke dokumentert |
| `AuthNotTrusted` | Warning | `-TrustedAuthServId` ble angitt, men ingen kontrollinje har en av disse ID-ene |
| `AuthTrustedConflict` | Warning | To kontrollinjer med klarert ID rapporterer ulike resultater for samme metode; gatewayen fjerner tydeligvis ikke fremmede linjer |
| `AuthUnverified` | Warning | Kontrollresultatene har en authserv-id som verken er klarert eller forekommer i leveringskjeden |
| `AuthMixedOrigins` | Warning | Kontrollinjer med flere opprinnelser finnes |
| `ReceivedSpfForeign` | Warning | `receiver=` i `Received-SPF`-linjen forekommer ikke i kjeden |
| `ReceivedSpfOnly` | Info | SPF-resultatet kommer bare fra `Received-SPF`, ikke fra en kontrollinje |
| `NoAuthResults` | Info | Ingen kontrollresultater i hodet |
| `DmarcFail` | Fail | DMARC ble ikke bestått ifølge mottaksserveren |
| `SpfNotPass` | Warning | SPF-resultat `fail`, `softfail`, `permerror` eller `temperror` |
| `DkimNotPass` | Warning | DKIM-resultat `fail`, `permerror` eller `temperror`, uten at mottakeren bekrefter ARC-kjeden |
| `DkimBrokenAfterForward` | Info | DKIM ble ikke bestått hos mottakeren, men mottakeren rapporterer `arc=pass`, og en tidligere ARC-instans har registrert en DKIM-`pass` for samme domene: typisk for videresendinger og e-postlister |
| `DkimWeakHash` | Warning | Signatur med `rsa-sha1` (RFC 8301 klassifiserer SHA-1 som foreldet) |
| `DkimBodyLength` | Warning | `l=`-taggen begrenser den signerte lengden på meldingsteksten; vedlagt innhold dekkes ikke |
| `DkimExpired` | Warning | `x=`-tidspunktet ligger i fortiden |
| `DkimFromUnsigned` | Warning | Feltet `From` er ikke inkludert i `h=`, selv om RFC 6376 krever det |
| `ArcStructureInconsistent` | Warning | ARC-hodene danner ikke en formelt komplett kjede (hull, manglende eller doble hoder, feil `cv=`-følge); kun strukturkontroll |
| `ClockSkew` | Info | Et hopp har et tidligere tidsstempel enn forgjengeren; forsinkelsene er bare tilnærmede verdier |
| `ReplyToMismatch` | Info | `Reply-To`-domenet avviker fra `From`-domenet; vanlig for nyhetsbrev, et mønster ved phishing |
| `SpfNotAligned` | Info | Konvoluttavsenderdomenet og `From`-domenet tilhører forskjellige organisasjoner; SPF bidrar da ikke til DMARC |
| `ExchangeWrongTenant` | Warning | Meldingen ble tilordnet en fremmed tenant; en vanlig årsak er en innkommende kobling fra en annen tenant med samme sertifikat eller de samme IP-adressene |
| `ExchangeHeadersFiltered` | Warning | Sendekoblingen fjernet Cross-Premises-hodene (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: Mottak via en mottakskobling med «Externally Secured», EOP-filtrering hoppet over |
| `SpamCategory` | Warning | Microsoft har tildelt en annen kategori enn `NONE` |
| `SpamConfidence` | Warning | SCL 5 eller høyere |

## Virkemåte og begrensninger

Modulen leser det som står i hodet og utleder av det hva som kan dokumenteres uten eksterne oppslag. Dette medfører enkelte begrensninger:

- **Ingen kryptografisk kontroll.** DKIM- og ARC-signaturer beregnes ikke på nytt, og DNS-oppføringer slås ikke opp. `Spf`, `Dkim`, `Dmarc` og `Arc` er alltid mottaksserverens vurdering; `ArcStructure` kontrollerer bare kjedens struktur.
- **Opprinnelse kan bare dokumenteres med kjennskap til gatewayen.** Uten `-TrustedAuthServId` er `AuthTrust` en plausibilitetskontroll. Med parameteren avhenger utsagnet av at gatewayen fjerner fremmede kontrollinjer med sin authserv-id; modulen kan ikke kontrollere dette.
- **Bare den siste `Received`-linjen er dokumentert.** Alle linjene under ble levert av avsenderen og kan være utformet vilkårlig. `Attested` markerer forskjellen; forsinkelsene i tidligere hopp bygger på opplysningene i disse linjene.
- **Organisasjonsdomener bestemmes heuristisk.** For Relaxed-alignment bruker modulen en kort liste over sammensatte endelser, ikke en fullstendig Public Suffix List.
- **AuthMechanism er bare delvis dokumentert.** Microsoft har ikke publisert kodene utenom verdien 10; modulen finner ikke på betydninger.
- **Tegnsett.** RFC 2047-verdier dekodes med kodingene som .NET kjenner på det aktuelle systemet. Ukjente tegnsett blir stående uendret.

## Personvern

Et fullstendig hode inneholder interne vertsnavn, IP-adresser, avsendere, mottakere og emner. Hvorfor slike opplysninger ikke hører hjemme i et nettbasert verktøy, forklares i artikkelen [Analyser e-posthoder uten å laste opp e-posten](/blog/e-mail-header-analysieren-ohne-upload). For modulen gjelder samme løfte som for nettleserversjonen: ingen nettverkstilgang. Testpakken til modulen inneholder en test som feiler så snart en nettverks- eller DNS-cmdlet forekommer i kildekoden.

## Kildekode og versjoner

Kildekoden er tilgjengelig under MIT-lisensen på [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Analyselogikken er en portering av biblioteket som også Header Analyzer på dette nettstedet bruker; begge deler testtilfellene. Continuous Integration kontrollerer hver endring med PSScriptAnalyzer og Pester i Windows PowerShell 5.1, PowerShell 7 på Windows, Ubuntu og macOS. Publiseringer i PowerShell Gallery skjer automatisk fra versjonerte tagger; endringene per versjon finnes i [changelog](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Versjon 0.2.0 fra 26. september 2026 skjerpet opprinnelsesmodellen etter en kommentar i en gjennomgang fra @saltyslugga: parameteren `-TrustedAuthServId`, nivået `Trusted`, `Matched` nå bare som plausibilitet. Egenskapen `ArcValid` er utgått og erstattet av `ArcStructure` og `ArcStructureIssues`; skript som analyserer `ArcValid` må tilpasses.

Feil og utvidelsesønsker mottar jeg som [Issue på GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Anonymiser testhoder for feilrapporter på forhånd; de medfølgende testtilfellene bruker utelukkende eksempeldomener i henhold til RFC 2606 og adresser i henhold til RFC 5737.

## Kilder

1.  [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): Pakkeside med installasjonskommando og versjonshistorikk.

2.  [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer): Kildekode, testpakke, changelog og Issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): Mal for oppbyggingen av denne referansen (syntaks, beskrivelse, eksempler, parameteregenskaper).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): Struktur for `Authentication-Results` og regelen om at bare mottaksorganisasjonens linje er relevant (avsnitt 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Struktur for `Received`-linjene og anbefalingen om en øvre grense som beskyttelse mot løkker (avsnitt 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): `with`-verdiene `ESMTPS`, `ESMTPA` og `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Taggene i `DKIM-Signature`, krav om signering av `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): Klassifisering av `rsa-sha1` som foreldet.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Strict- og Relaxed-alignment.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): Struktur for kjeden med `ARC-Seal`, `ARC-Message-Signature` og `ARC-Authentication-Results`, instansnumre og `cv=`-verdier.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Betydningen av SCL, BCL, CAT, SFV, IPV og `compauth`-Reason-kodene.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): Opprinnelse til betydningene av MessageDirectionality, AuthAs og AuthMechanism.

13.  [Header Analyzer på rafaelpfister.ch](/tools/header-analyzer): Nettleserversjonen med samme analyselogikk.
