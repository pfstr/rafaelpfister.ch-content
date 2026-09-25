---
title: "MailHeaderAnalyzer: Analyse av e-posthoder i PowerShell uten nettverkstilgang"
navTitle: "MailHeaderAnalyzer (PowerShell)"
description: "Referanse for PowerShell-modulen MailHeaderAnalyzer: parameteroversikt, syntaks, beskrivelse, eksempler og parameteregenskaper for Get-MailHeaderAnalysis og ConvertTo-MailHeaderReport, samt utdataobjektet med leveringskjede, autentiseringsresultater, Exchange Online-klassifisering og alle funnkoder."
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
url: https://rafaelpfister.ch/no/blog/mailheaderanalyzer-analyse-av-e-posthoder-i-powershell-uten-nettverkstilgang
translationSourceHash: 7c01d42e83d3aa175486c74e2f88855bc7d9dee519026ce748edb239fdd355cc
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T18:57:10.573Z
translationReview: automatic
---

# MailHeaderAnalyzer: Analyse av e-posthoder i PowerShell uten nettverkstilgang

MailHeaderAnalyzer er en PowerShell-modul med to cmdleter. `Get-MailHeaderAnalysis` analyserer hodet i en e-post: leveringskjeden med forsinkelser og TLS-opplysninger, SPF-, DKIM-, DMARC- og ARC-resultater, inkludert kontroll av om resultatene stammer fra mottaksserveren, DMARC-justering, hybridklassifiseringen i Exchange Online, vurderingene fra Microsoft Defender, SpamAssassin og Rspamd samt avvik som doble `From`-linjer eller Unicode-kontrolltegn. `ConvertTo-MailHeaderReport` oppretter en rapport for saker. Modulen fungerer helt offline: ingen DNS-oppslag, ingen HTTP-forbindelser. Den er kommandolinjeversjonen av [Header Analyzer på dette nettstedet](/tools/header-analyzer) og bruker samme analyselogikk.

| | |
|---|---|
| Modul | [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer) |
| Kildekode | [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer), MIT-lisens |
| Gjelder for | Windows PowerShell 5.1, PowerShell 7.x; Windows, Linux, macOS; Exchange Management Shell |
| Cmdleter | `Get-MailHeaderAnalysis`, `ConvertTo-MailHeaderReport` |

## Parameteroversikt

| Cmdlet | Parameter | Type | Obligatorisk | Pipeline | Virkning |
|---|---|---|---|---|---|
| `Get-MailHeaderAnalysis` | `-Header` | `String[]` | Ja (parametergruppe Text) | Ja, etter verdi | Hodet som tekst. Linjer fra pipelinen settes sammen til ett hode |
| `Get-MailHeaderAnalysis` | `-Path` | `String[]` | Ja (parametergruppe Path) | Ja, etter egenskapsnavn | Fil med hodet eller en fullstendig `.eml`-melding; mottar objekter fra `Get-ChildItem` |
| `Get-MailHeaderAnalysis` | `-FromClipboard` | `Switch` | Ja (parametergruppe Clipboard) | Nei | Leser hodet fra utklippstavlen (kun Windows) |
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

Analyserer hodet i en e-post og returnerer et analyseobjekt.

### Syntaks

#### Text (standard)

```powershell
Get-MailHeaderAnalysis
    [-Header] <String[]>
    [<CommonParameters>]
```

#### Path

```powershell
Get-MailHeaderAnalysis
    -Path <String[]>
    [<CommonParameters>]
```

#### Clipboard

```powershell
Get-MailHeaderAnalysis
    -FromClipboard
    [<CommonParameters>]
```

### Beskrivelse

Cmdleten deler råhodet inn i felt, bretter ut RFC-5322-foldinger og dekoder RFC-2047-verdier i emne og adresser. Fra `Received`-linjene oppretter den leveringskjeden i kronologisk rekkefølge, beregner forsinkelsen per stasjon og leser TLS-versjon, chiffer og protokollklasse i henhold til RFC 3848. Fra `Authentication-Results`, `Received-SPF`, `DKIM-Signature` og ARC-kjeden fastslår den autentiseringsresultatene og kontrollerer om kontrollinjen faktisk stammer fra en stasjon i leveringskjeden (RFC 8601, avsnitt 5). I tillegg kommer DMARC-justering, hybridklassifiseringen i Exchange Online, vurderingene fra spamfiltrene og en liste over avvik.

Cmdleten utfører ingen DNS-oppslag og åpner ingen nettverksforbindelse. `Spf`, `Dkim` og `Dmarc` er derfor alltid mottaksserverens vurdering, supplert med kontrollen av om vurderingen stammer fra den. DKIM-signaturer etterberegnes ikke kryptografisk.

Inndata leses tolerant: En tom linje avslutter hodet, og påfølgende meldingstekst ignoreres. Linjer uten feltnavn og uten innledende mellomrom, slik de kan oppstå ved kopiering fra klientdialoger, hører til det foregående feltet. En mbox-skillelinje `From ...` før det første feltet hoppes over, og en Byte Order Mark fjernes. Maksimalt 200 `Received`-linjer analyseres, regnet fra leveringen.

### Eksempler

#### Eksempel 1

```powershell
Get-MailHeaderAnalysis -FromClipboard
```

Analyserer hodet som finnes på utklippstavlen. I Outlook for Windows finner du hodet under Fil, Egenskaper, Internett-hoder; i Outlook på nettet under meldingsalternativene, «Vis meldingsdetaljer».

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

Viser leveringskjeden som tabell, med første hopp først.

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

Leser hodet linje for linje fra en tekstfil og viser bare avvikene. `-Raw` ved `Get-Content` er ikke nødvendig; cmdleten setter selv sammen linjene.

#### Eksempel 5

```powershell
Get-ChildItem -Path .\export\*.eml |
    Get-MailHeaderAnalysis |
    Select-Object Source, Subject, Spf, Dkim, Dmarc, AuthTrust, HopCount,
        @{ Name = 'Findings'; Expression = { ($_.Findings.Code | Sort-Object -Unique) -join ',' } } |
    Export-Csv -Path .\auswertung.csv -NoTypeInformation -Encoding UTF8
```

Analyserer alle meldinger i en mappe og skriver en CSV-fil med én linje per melding. Den beregnede kolonnen oppsummerer funnkodene.

| Alternativ | Virkning |
|---|---|
| `Get-ChildItem -Path .\export\*.eml` | Leverer filobjektene; `-Path` overtar egenskapen `FullName` |
| `Select-Object … @{ Name; Expression }` | Beregnet kolonne som oppsummerer alle funnkoder til én tegnstreng |
| `Export-Csv -NoTypeInformation -Encoding UTF8` | CSV uten typeoverskrift, UTF-8 for diakritiske tegn i emnelinjer |

#### Eksempel 6

```powershell
Get-ChildItem .\export\*.eml |
    Get-MailHeaderAnalysis |
    Where-Object AuthTrust -eq 'Unmatched' |
    Select-Object Source, AuthServId, DeliveredBy
```

Lister meldinger der `Authentication-Results`-linjen ikke stammer fra det leverende systemet. I phishing-analyser er dette et raskt første filter.

#### Eksempel 7

```powershell
$a = Get-MailHeaderAnalysis -Path .\nachricht.eml
$a.Hops[$a.SlowestHopIndex - 1] | Format-List Index, FromHost, ByHost, Delay
```

Viser stasjonen med størst forsinkelse. `SlowestHopIndex` er 1-basert, mens arrayet `Hops` er 0-basert.

#### Eksempel 8

```powershell
Get-MailHeaderAnalysis -Path .\nachricht.eml | ConvertTo-Json -Depth 6 | Set-Content .\analyse.json
```

Skriver hele analysen som JSON. `-Depth 6` er nødvendig fordi `Hops`, `DkimSignatures` og `Exchange` er nestede objekter; standardverdien 2 ville bare vist dem som typenavn.

### Parametere

#### -Header

Hodet som tekst. Parameteren godtar én enkelt streng med hele hodet eller flere strenger; pipelineinndata samles og settes sammen til ett hode til slutt, derfor fungerer `Get-Content datei | Get-MailHeaderAnalysis` uten `-Raw`. Hvis flere hoder skal analyseres separat, bruk `-Path` med flere filer.

Aliaser: `Text`, `Raw`, `InputObject`

| Parameteregenskap | Verdi |
|---|---|
| Type | `String[]` |
| Standardverdi | Ingen |
| Jokertegn støttes | Nei |

| Parametergruppe Text | Verdi |
|---|---|
| Posisjon | 0 |
| Obligatorisk | Ja |
| Verdi fra pipeline | Ja |
| Verdi fra pipeline etter egenskapsnavn | Nei |

#### -Path

Bane til en fil som inneholder hodet eller en fullstendig `.eml`-melding. Relative baner løses mot gjeldende katalog. Filen leses med `[System.IO.File]::ReadAllText`: En Byte Order Mark tas hensyn til, uten BOM brukes UTF-8. Det opprettes ett eget resultatobjekt for hver fil; manglende filer gir en ikke-avbrytende feil.

Aliaser: `FullName`, `PSPath`, `LiteralPath`

| Parameteregenskap | Verdi |
|---|---|
| Type | `String[]` |
| Standardverdi | Ingen |
| Jokertegn støttes | Nei |

| Parametergruppe Path | Verdi |
|---|---|
| Posisjon | Navngitt |
| Obligatorisk | Ja |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Ja |

#### -FromClipboard

Leser hodet med `Get-Clipboard -Raw` fra utklippstavlen. Parameteren er bare tilgjengelig i Windows; på Linux og macOS avbrytes cmdleten med en feilmelding, det samme gjelder ved tom utklippstavle.

| Parameteregenskap | Verdi |
|---|---|
| Type | `SwitchParameter` |
| Standardverdi | `False` |
| Jokertegn støttes | Nei |

| Parametergruppe Clipboard | Verdi |
|---|---|
| Posisjon | Navngitt |
| Obligatorisk | Ja |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Nei |

### Inndata

`System.String`: hodelinjer eller hele hodet, til `-Header`.

`System.IO.FileInfo`: filobjekter fra `Get-ChildItem`, der `FullName` bindes til `-Path`.

### Utdata

`MailHeaderAnalyzer.Analysis`: ett objekt per analysert hode. Egenskapene er beskrevet i avsnittet [Utdataobjekt](#ausgabeobjekt).

### Merknader

Analyseteksten (forklaringer i `Findings`, `CompAuthReasonMeaning` og betydningsfeltene) er på engelsk, slik at den kan brukes uendret i internasjonale saker. Utdataene i standardvisningen kan vises fullt ut med `Format-List *`.

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

Rapporten omfatter emne, avsender, dato og Message-ID, autentiseringsresultatene med opprinnelsesangivelse og justering, alle funn, leveringskjeden, Exchange-klassifiseringen og verdiene fra spamfiltrene. Unicode-kontrolltegn for skriveretningen forblir synlige som `<U+...>` i rapporten, slik at de ikke overføres til et saksbehandlingssystem gjennom rapporten. Den siste linjen angir modulversjonen.

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

Gir rapporten som tekst uten Markdown-formatering, for eksempel for e-post eller konsolllogger.

#### Eksempel 3

```powershell
Import-Module MailHeaderAnalyzer
Get-MailHeaderAnalysis -Path 'C:\Temp\bounce.eml' | ConvertTo-MailHeaderReport -Format Text
```

Bruk i Exchange Management Shell. Hvis `$env:PSModulePath` der er begrenset av gruppepolicyer, last inn modulen med `Import-Module` og den fullstendige banen til `.psd1`-filen.

### Parametere

#### -Analysis

Analyseobjektet fra `Get-MailHeaderAnalysis`. Cmdleten avviser andre objekttyper med en bindingsfeil.

| Parameteregenskap | Verdi |
|---|---|
| Type | `MailHeaderAnalyzer.Analysis` |
| Standardverdi | Ingen |
| Jokertegn støttes | Nei |

| Parametergruppe (alle) | Verdi |
|---|---|
| Posisjon | 0 |
| Obligatorisk | Ja |
| Verdi fra pipeline | Ja |
| Verdi fra pipeline etter egenskapsnavn | Nei |

#### -Format

Utdataformatet. Gyldige verdier:

- `Markdown`: overskrifter, punktlister og leveringskjeden som tabell. Standard.
- `Text`: overskrifter med store bokstaver, innrykkede linjer, leveringskjeden som nummerert liste.

| Parameteregenskap | Verdi |
|---|---|
| Type | `String` |
| Tillatte verdier | `Markdown`, `Text` |
| Standardverdi | `Markdown` |
| Jokertegn støttes | Nei |

| Parametergruppe (alle) | Verdi |
|---|---|
| Posisjon | Navngitt |
| Obligatorisk | Nei |
| Verdi fra pipeline | Nei |
| Verdi fra pipeline etter egenskapsnavn | Nei |

### Inndata

`MailHeaderAnalyzer.Analysis`: resultatet fra `Get-MailHeaderAnalysis`.

### Utdata

`System.String`: rapporten, én streng per analyseobjekt.

## Utdataobjekt

`Get-MailHeaderAnalysis` returnerer ett objekt av typen `MailHeaderAnalyzer.Analysis` per inndata. Standardvisningen viser sammendraget fra eksempel 1; alle egenskapene er tilgjengelige via `Select-Object`, `Format-List *` eller `ConvertTo-Json`.

### MailHeaderAnalyzer.Analysis

| Egenskap | Type | Innhold |
|---|---|---|
| `Source` | String | Filbane, `Clipboard` eller `Text` |
| `Subject` | String | Emne, RFC 2047-dekodet |
| `From`, `ReplyTo`, `ReturnPath` | Adresseobjekt | `Name`, `Address`, `Domain`, `Display`; `$null` når feltet mangler |
| `Date` | DateTime (UTC) | Verdien i `Date`-feltet |
| `MessageId` | String | `Message-ID` |
| `MailFromDomain` | String | Konvoluttavsenderdomene fra `smtp.mailfrom` i SPF-kontrollen, ellers fra `Return-Path` |
| `Spf`, `Dkim`, `Dmarc`, `Arc`, `CompAuth` | String | Resultat ifølge den relevante `Authentication-Results`-linjen (`pass`, `fail`, `none`, `softfail` og flere); `$null` når ikke kontrollert |
| `CompAuthReason`, `CompAuthReasonMeaning` | String | Reason-kode for sammensatt autentisering fra Microsoft 365 og dens betydning |
| `AuthTrust` | String | `Matched`, `Unmatched`, `Absent` eller `None`, se [AuthTrust](#authtrust-herkunft-der-prüfergebnisse) |
| `AuthServId` | String | authserv-id for den relevante kontrollinjen |
| `AuthenticationResults` | Objekt[] | Alle `Authentication-Results`-linjer med `AuthServId`, `Methods`, `Trust`, `Raw` |
| `ReceivedSpf` | Objekt | `Received-SPF`-linjen med `Result` og `Properties` |
| `SpfAlignment`, `DkimAlignment` | String | `Strict`, `Relaxed`, `None` eller `$null` |
| `Hops` | Hop[] | Leveringskjeden i kronologisk rekkefølge, se [Hop-objekt](#mailheaderanalyzerhop) |
| `HopCount`, `TotalDuration`, `SlowestHopIndex`, `HasClockSkew` | Int, TimeSpan, Int, Bool | Nøkkeltall for kjeden |
| `DeliveredBy` | String | `by`-vert for den nyeste `Received`-linjen, altså den leverende stasjonen |
| `DkimSignatures` | Objekt[] | Per signatur `Domain`, `Selector`, `Algorithm`, `Canonicalization`, `SignedHeaders`, `BodyLength`, `Timestamp`, `Expires`, `ReceiverResult`, `Tags` |
| `ArcChain`, `ArcValid` | Objekt[], Bool | ARC-instanser med `Instance`, `SealDomain`, `ChainValidation`, `Methods`; `ArcValid` er `$null` uten ARC-kjede |
| `Exchange` | Objekt | Hybridklassifisering fra Exchange Online, se [Exchange-objekt](#mailheaderanalyzerexchangeclassification); `$null` uten tilsvarende hoder |
| `Spam` | Objekt | Vurderinger fra spamfiltrene, se [Spam-objekt](#mailheaderanalyzerspamassessment); `$null` uten tilsvarende hoder |
| `List` | Objekt | `ListId`, `Unsubscribe`, `OneClick` (RFC 8058); `$null` uten listehoder |
| `Findings` | Finding[] | Avvik med `Severity`, `Code`, `Message`, se [Findings](#findings) |
| `Fields` | Objekt[] | Alle felt med `Name`, `Value` (utbrettet) og `Raw` |
| `HadBody` | Bool | Om meldingstekst fulgte etter hodet |

### MailHeaderAnalyzer.Hop

Hver oppføring i `Hops` tilsvarer én `Received`-linje. Rekkefølgen er kronologisk, altså motsatt av rekkefølgen i hodet.

| Egenskap | Innhold |
|---|---|
| `Index` | Løpende nummer, 1 = innlevering |
| `FromHost`, `Helo`, `ReverseDns`, `IPAddress`, `IsPrivateIP` | Opplysninger om det innleverende systemet fra `from`-delen; IP-adressen kommer fra hakeparentesen i kommentaren, rDNS-navnet fra kommentaren foran |
| `ByHost`, `Software` | Mottakende system og dets programvare (kommentar etter `by`) |
| `Protocol`, `ProtocolClass` | `with`-verdi og klasse i henhold til RFC 3848: `TlsAuthenticated` (ESMTPSA), `Tls` (ESMTPS eller TLS-opplysning finnes), `Authenticated` (ESMTPA), `Plain`, `Http`, `Mapi`, `Local` |
| `TlsVersion`, `TlsCipher` | Fra skrivemåtene til Microsoft (`version=TLS1_2, cipher=…`), Postfix (`using TLSv1.3 with cipher …`) og Exim |
| `Id`, `For`, `Via` | Andre `Received`-bestanddeler |
| `Date` | Tidsstempel etter semikolon, UTC |
| `Delay` | TimeSpan til forrige hopp; negativ ved klokkeavvik |
| `Provider` | Gjenkjent leverandør eller gateway basert på vertsnavn (Microsoft 365, Google Workspace, SEPPmail, HIN, Proofpoint, Mimecast og flere) |
| `Attested` | `$true` bare ved siste hopp: bare denne linjen ble skrevet av det mottakende systemet selv, alle linjene under fantes allerede i meldingen |
| `Raw` | Originallinjen |

### AuthTrust: opprinnelse for kontrollresultater

En `Authentication-Results`-linje kan enhver avsender selv skrive inn i en melding. Ifølge RFC 8601, avsnitt 5, er bare linjen fra den mottakende organisasjonen relevant, og dens authserv-id må kunne knyttes til en stasjon i leveringskjeden. Cmdleten sammenligner authserv-id-en i hver linje med `by`-vertene i `Received`-kjeden (samme domene eller underdomene, alltid på punktgrensen, ingen delstreng).

| Verdi | Betydning |
|---|---|
| `Matched` | Authserv-id-en forekommer som `by`-vert i kjeden. Bare slike linjer inngår i `Spf`, `Dkim`, `Dmarc` så snart minst én finnes |
| `Unmatched` | Authserv-id-en forekommer ikke i kjeden. Resultatene vises, men regnes som en udokumentert påstand; funnet `AuthUnverified` gjør oppmerksom på dette |
| `Absent` | Linjen har ingen authserv-id. Microsoft 365 skriver kontrollinjen i denne formen, den starter direkte med `spf=` |
| `None` | Ingen kontrollinje finnes |

Hvis hodet inneholder kontrollinjer med flere opprinnelser, rapporterer funnet `AuthMixedOrigins` dette. Hvis en kontrollinje mangler, men en `Received-SPF`-linje finnes, overtas dens resultat som `Spf` og merkes med `ReceivedSpfOnly`.

### DMARC-justering

`SpfAlignment` sammenligner konvoluttavsenderdomenet med `From`-domenet, `DkimAlignment` sammenligner `d=`-domenet i den kontrollerte signaturen med `From`-domenet. `Strict` betyr identisk domene, `Relaxed` samme organisasjonsdomene, `None` ingen samsvar. Organisasjonsdomenet fastsettes heuristisk: De siste to etikettene, eller de siste tre ved kjente sammensatte endelser som `co.uk` eller `com.au`. En komplett Public Suffix List er ikke inkludert.

### MailHeaderAnalyzer.ExchangeClassification

Hvis hodet inneholder feltene `X-MS-Exchange-Organization-*`, `X-OriginatorOrg` eller `X-MS-Exchange-CrossTenant-*`, fyller cmdleten egenskapen `Exchange`. Betydningene følger Exchange-teamets artikkel «Demystifying hybrid mail flow»; bakgrunnen er beskrevet i artikkelen [Exchange-hybridhoder: internt eller eksternt?](/blog/exchange-hybrid-header-intern-extern).

| Egenskap | Innhold |
|---|---|
| `Directionality`, `DirectionalityMeaning` | `Originating` eller `Incoming` med forklaring |
| `AuthAs`, `AuthAsMeaning` | `Internal` eller `Anonymous` med konsekvensene for EOP-filtrering |
| `AuthSource` | Serveren som foretok klassifiseringen |
| `AuthMechanism`, `AuthMechanismMeaning` | Mekanismekode. Bare verdien 10 (Externally Secured) er offentlig dokumentert; for alle andre koder oppgir modulen uttrykkelig at Microsoft ikke har dokumentert dem |
| `OriginatorOrg` | Standarddomene for den sendende leieren, det ikke-forfalskbare leierkjennetegnet ved mottak fra Microsoft 365 |
| `CrossTenantAuthAs`, `CrossTenantAuthSource`, `CrossTenantId` | Klassifisering og leier-ID ved leiergrensen |
| `CrossTenantFromEntity`, `CrossTenantFromMeaning` | `Internet`, `Hosted` eller `HybridOnPrem` med forklaring |
| `WrongTenantAttribution` | Verdien av `X-MS-Exchange-CrossTenant-OriginalAttributedTenantConnectingIp` når meldingen ble tilordnet en annen leier |
| `OrganizationHeadersPreserved`, `CrossPremisesHeadersFiltered` | Markører for organisasjonshoder som er mottatt eller fjernet av sendekonnektoren |

### MailHeaderAnalyzer.SpamAssessment

Egenskapen `Spam` oppsummerer vurderingene fra de kjente filtrene. Verdiene kommer fra eksterne systemer og dekodes, men vurderes ikke.

| Egenskap | Kilde | Innhold |
|---|---|---|
| `Scl`, `SclMeaning` | `X-Forefront-Antispam-Report`, `X-Microsoft-Antispam` | Spam Confidence Level med betydning (-1 pålitelig, 0/1 ikke spam, 5/6 mistanke om spam, 9 svært sannsynlig spam) |
| `Bcl` | `X-Microsoft-Antispam` | Bulk Complaint Level 0 til 9 |
| `Category`, `CategoryMeaning` | `CAT:` | Klassifisering som `SPM`, `PHSH`, `HPHSH`, `MALW`, `SPOOF`, `DIMP`, `UIMP`, `BULK` |
| `SpamFilterVerdict`, `SpamFilterMeaning` | `SFV:` | Filterresultat som `NSPM`, `SPM`, `SKA` (tillatelsesliste), `SKI` (intraorganisatorisk) |
| `IPVerdict`, `IPVerdictMeaning` | `IPV:` | `CAL` (IP på tilkoblingstillatelsesliste) eller `NLI` (ingen omdømme) |
| `Direction`, `DirectionMeaning` | `DIR:` | `INB`, `OUT`, `INT` |
| `ConnectingIP`, `Country` | `CIP:`, `CTRY:` | Innleverende IP og opprinnelsesland |
| `Forefront` | `X-Forefront-Antispam-Report` | Alle nøkkel-verdi-par i feltet |
| `SpamAssassinScore`, `SpamAssassinTests` | `X-Spam-Status` | Poengsum og utløste tester |
| `RspamdSymbols` | `X-Spamd-Result`, `X-Spam-Report` | Symboler med poengsum |

Den fullstendige listen over `compauth`-reason-koder finnes i artikkelen [Microsoft 365 compauth: Reason-koder](/blog/microsoft-365-compauth-reason-codes).

### Funn

Cmdleten leverer avvik som objekter i `Findings`, hver med `Severity` (`Info`, `Warning`, `Fail`), en stabil `Code` for filtre og skript, samt en forklaring i `Message`.

| Kode | Alvorlighetsgrad | Betydning |
|---|---|---|
| `DuplicateField` | Warning | Et felt som RFC 5322 begrenser til én instans (`From`, `Subject`, `Date`, `Message-ID` og flere), forekommer flere ganger. E-postklienter og filtre kan velge ulike instanser; et kjent mønster ved forfalskninger |
| `BidiControls` | Warning | Unicode-kontrolltegn for skriveretning i et felt. De snur leseretningen, `fdp.exe` vises da som `exe.pdf`. Modulen viser dem som `<U+202E>` |
| `HopOverflow` | Warning | Mer enn 200 `Received`-linjer; de overskytende ble ikke analysert |
| `AuthUnverified` | Warning | Kontrollresultatene har en authserv-id som ikke forekommer i leveringskjeden |
| `AuthMixedOrigins` | Warning | Kontrollinjer med flere opprinnelser finnes |
| `ReceivedSpfForeign` | Warning | `receiver=` i `Received-SPF`-linjen forekommer ikke i kjeden |
| `ReceivedSpfOnly` | Info | SPF-resultatet stammer bare fra `Received-SPF`, ikke fra en kontrollinje |
| `NoAuthResults` | Info | Ingen kontrollresultater i hodet |
| `DmarcFail` | Fail | DMARC besto ikke ifølge mottaksserveren |
| `SpfNotPass` | Warning | SPF-resultat `fail`, `softfail`, `permerror` eller `temperror` |
| `DkimNotPass` | Warning | DKIM-resultat `fail`, `permerror` eller `temperror` uten ARC-vitne |
| `DkimBrokenAfterForward` | Info | DKIM besto ikke hos mottakeren, men et ARC-segl fra samme domene vitner om en tidligere gyldig signatur: typisk for videresendinger og e-postlister |
| `DkimWeakHash` | Warning | Signatur med `rsa-sha1` (RFC 8301 klassifiserer SHA-1 som foreldet) |
| `DkimBodyLength` | Warning | `l=`-taggen begrenser den signerte lengden på meldingsteksten; vedlagt innhold dekkes ikke |
| `DkimExpired` | Warning | `x=`-tidspunktet ligger i fortiden |
| `DkimFromUnsigned` | Warning | `From`-feltet er ikke inkludert i `h=`, selv om RFC 6376 krever det |
| `ClockSkew` | Info | Et hopp har et tidligere tidsstempel enn forgjengeren; forsinkelsene er bare tilnærmingsverdier |
| `ReplyToMismatch` | Info | `Reply-To`-domenet avviker fra `From`-domenet; vanlig for nyhetsbrev, et mønster ved phishing |
| `SpfNotAligned` | Info | Konvoluttavsenderdomenet og `From`-domenet tilhører ulike organisasjoner; SPF bidrar da ikke til DMARC |
| `ExchangeWrongTenant` | Warning | Meldingen ble tilordnet en annen leier; en klassisk årsak er en innkommende konnektor fra en annen leier med samme sertifikat eller de samme IP-adressene |
| `ExchangeHeadersFiltered` | Warning | Sendekonnektoren fjernet Cross-Premises-hodene (KB3212872) |
| `ExchangeExternallySecured` | Warning | AuthMechanism 10: innkommende via en mottakskonnektor med «Externally Secured», EOP-filtrering ble hoppet over |
| `SpamCategory` | Warning | Microsoft har tildelt en kategori ulik `NONE` |
| `SpamConfidence` | Warning | SCL 5 eller høyere |

## Virkemåte og begrensninger

Modulen leser det som står i hodet og utleder av det hva som kan dokumenteres uten eksterne oppslag. Dette medfører noen begrensninger:

- **Ingen kryptografisk kontroll.** DKIM-signaturer etterberegnes ikke, og DNS-oppføringer slås ikke opp. `Spf`, `Dkim` og `Dmarc` er alltid mottaksserverens vurdering, supplert med kontrollen av om vurderingen i det hele tatt stammer fra den.
- **Bare den siste `Received`-linjen er dokumentert.** Alle linjene under ble levert av avsenderen og kan være utformet vilkårlig. `Attested` markerer forskjellen; forsinkelsene for tidligere hopp bygger på opplysningene i disse linjene.
- **Organisasjonsdomener heuristisk.** For Relaxed Alignment bruker modulen en kort liste over sammensatte endelser, ikke en komplett Public Suffix List.
- **AuthMechanism bare delvis dokumentert.** Microsoft har ikke publisert kodene bortsett fra verdien 10; modulen finner ikke på betydninger.
- **Tegnsett.** RFC-2047-verdier dekodes med kodingene som .NET kjenner på det aktuelle systemet. Ukjente tegnsett forblir uendret.

## Personvern

Et fullstendig hode inneholder interne vertsnavn, IP-adresser, avsendere, mottakere og emne. Hvorfor disse opplysningene ikke hører hjemme i et nettbasert verktøy, beskrives i artikkelen [Analyser e-posthoder uten å laste opp e-posten](/blog/e-mail-header-analysieren-ohne-upload). For modulen gjelder det samme løftet som for nettleserversjonen: ingen nettverkstilganger. Modulens testpakke inneholder en test som feiler så snart en nettverks- eller DNS-cmdlet forekommer i kildekoden.

## Kildekode og versjoner

Kildekoden er tilgjengelig under MIT-lisensen på [GitHub](https://github.com/pfstr/MailHeaderAnalyzer). Analyselogikken er en portering av biblioteket som også brukes av Header Analyzer på dette nettstedet; begge deler testtilfellene. Kontinuerlig integrasjon kontrollerer hver endring med PSScriptAnalyzer og Pester på Windows PowerShell 5.1, PowerShell 7 på Windows, Ubuntu og macOS. Publiseringer i PowerShell Gallery skjer automatisk fra versjonerte tagger; endringene per versjon finnes i [endringsloggen](https://github.com/pfstr/MailHeaderAnalyzer/blob/main/CHANGELOG.md).

Feil og ønsker om utvidelser mottar jeg som [Issue på GitHub](https://github.com/pfstr/MailHeaderAnalyzer/issues). Testhoder for feilrapporter må anonymiseres på forhånd; de medfølgende testtilfellene bruker utelukkende eksempeldomener i henhold til RFC 2606 og adresser i henhold til RFC 5737.

## Kilder

1.  [MailHeaderAnalyzer i PowerShell Gallery](https://www.powershellgallery.com/packages/MailHeaderAnalyzer): pakkeside med installasjonskommando og versjonshistorikk.

2.  [pfstr/MailHeaderAnalyzer på GitHub](https://github.com/pfstr/MailHeaderAnalyzer): kildekode, testpakke, endringslogg og Issues.

3.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2?view=exchange-ps): mal for strukturen i denne referansen (syntaks, beskrivelse, eksempler, parameteregenskaper).

4.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://www.rfc-editor.org/rfc/rfc8601): oppbygning av `Authentication-Results` og regelen om at bare linjen fra den mottakende organisasjonen er relevant (avsnitt 5).

5.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): oppbygning av `Received`-linjene og anbefalingen om en øvre grense som løkkebeskyttelse (avsnitt 6.3).

6.  [RFC 3848: ESMTP and LMTP Transmission Types Registration](https://www.rfc-editor.org/rfc/rfc3848): `with`-verdiene `ESMTPS`, `ESMTPA` og `ESMTPSA`.

7.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): tagger i `DKIM-Signature`, kravet om å signere `From`.

8.  [RFC 8301: Cryptographic Algorithm and Key Usage Update to DKIM](https://www.rfc-editor.org/rfc/rfc8301): klassifisering av `rsa-sha1` som foreldet.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Strict og Relaxed Alignment.

10.  [RFC 8617: The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617): `ARC-Seal`, `ARC-Authentication-Results` og `cv=`-kjedevalideringen.

11.  [Microsoft Learn: Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): betydningen av SCL, BCL, CAT, SFV, IPV og `compauth`-reason-kodene.

12.  [Exchange Team Blog: Demystifying hybrid mail flow](https://techcommunity.microsoft.com/blog/exchange/demystifying-hybrid-mail-flow-when-is-a-message-internal/1420838): opprinnelsen til betydningene av MessageDirectionality, AuthAs og AuthMechanism.

13.  [Header Analyzer på rafaelpfister.ch](/tools/header-analyzer): nettleserversjonen med samme analyselogikk.
