---
title: "Lukke journaling-gapet: Eksport fra Microsoft Purview og import til Enterprise Vault"
navTitle: "Journaling-gap"
description: "Hvis journalrapporter fra Exchange Online uteblir i noen dager, kan de ikke opprettes i ettertid. Innholdet finnes imidlertid fortsatt i postboksene. Slik eksporterer du det som PST via den nye Purview-eDiscovery, hvorfor Enterprise Vaults PST-migrator som regel ikke er aktuelt, og hvordan etterimporten går via journalpostboksen uten å fortrenge den løpende journalstrømmen, med to generiske skript for import, flytting og logging."
date: "2026-09-22"
kategorie: "Arkivering og journaling"
timeToRead: "18 min lesetid"
themen:
  - archivierung-journaling
  - microsoft-365-exchange
  - exchange-onprem-hybrid
produkte:
  - "exchange-online"
  - "exchange-on-premises"
protokolle:
  - "backup-dr"
  - "powershell"
  - "troubleshooting"
slug: "lukke-journaling-gapet-eksport-fra-microsoft-purview-og-import-til-enterprise-vault"
translationId: "article-5b24118ac7567d96"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/no/blog/lukke-journaling-gapet-eksport-fra-microsoft-purview-og-import-til-enterprise-vault
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:43:22.378Z
translationReview: automatic
---

# Lukke journaling-gapet: Eksport fra Microsoft Purview og import til Enterprise Vault

Journaling er en transportprosess. Exchange oppretter journalrapporten idet meldingen går gjennom transporten, og leverer den til journalpostboksen. Hvis denne leveringen mislykkes, for eksempel fordi en kobling eller et mottakerobjekt er feilkonfigurert, finnes det ingen funksjon som henter inn rapporten senere. Det som gikk gjennom transporten i dette tidsrommet, mangler i arkivet.

Innholdet i meldingene finnes imidlertid fortsatt i postboksene til de involverte, og ved en oppbevaringspolicy uten utløp også når brukere har slettet meldingene. Dermed gjenstår bare én farbar vei for etterføring: Eksporter tidsrommet fra alle postbokser og importer kopiene til journalarkivet. Følgende fremgangsmåte bruker den nye eDiscovery i Microsoft Purview på eksportsiden og Veritas Enterprise Vault 15 på importsiden; måleverdiene stammer fra en etterimport av flere hundre tusen meldinger, og de to skriptene ble utviklet i den forbindelse.

## Hva en eksport kan erstatte, og hva den ikke kan

En journalrapport består av konvolutten og innholdet. Konvolutten inneholder mottakerne som transporten faktisk leverte til: også blindkopier og oppløste medlemmer av distribusjonslister. En postboks-kopi inneholder ikke disse opplysningene. Hvem som mottok en melding som blindkopi, kan ikke lenger fastslås etter etterimporten. Selve innholdet kan derimot rekonstrueres fullstendig, forutsatt at to betingelser er oppfylt.

For det første må det ligge en oppbevaring over postboksene, slik at også slettede meldinger fortsatt finnes i de gjenopprettelige elementene. Dette kontrollerer du i Security & Compliance-PowerShell:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

En policy med `RetentionComplianceAction: Keep` og `RetentionDuration: Unlimited` via `ExchangeLocation: All` sikrer innholdet fullstendig. Unntakslisten i `ExchangeLocationException` angir postboksene dette ikke gjelder for. For disse kan bare det som fortsatt finnes der, rekonstrueres.

For det andre teller en eksport postboks-kopier, ikke meldinger. En melding til femten interne mottakere forekommer femten ganger i resultatet. I det beskrevne tilfellet tilsvarte 98'131 kopier for én enkelt dag omtrent 6'800 unike meldinger, fastslått fra meldingssporing. Hvordan disse kopiene skal håndteres, avgjøres av compliance, ikke IT (se avsnittet om deduplisering).

## Bestemme tidsrommet

Starten på gapet finnes i selve journalpostboksen, på Exchange-serveren som holder den. Tidspunktet for den sist leverte rapporten er starten:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Det samme gjelder postboksen som står i `JournalingReportNdrTo` i transportkonfigurasjonen. Den samler journalrapporter som ikke kan leveres, og det nyeste elementet bekrefter tidspunktet. Slutten på gapet er tidspunktet da journalregelen ble endret til det reparerte målet. Mellom disse ligger eksportperioden, i UTC.

Meldingssporing gir forventet antall for avstemming: én rad per mottaker og melding, med `MessageId`. En historisk rapport dekker hele tidsrommet:

```powershell
Connect-ExchangeOnline
$bericht = @{
  ReportTitle   = "Journal-Luecke"
  ReportType    = "MessageTrace"
  StartDate     = "2026-09-10"
  EndDate       = "2026-09-12"
  NotifyAddress = "admin@example.com"
}
Start-HistoricalSearch @bericht
```

De unike `MessageId`-verdiene fra denne rapporten er tallet som etterimporten til slutt måles mot.

## Eksport fra den nye eDiscovery

Microsoft avviklet de klassiske grensesnittene for Content Search og eDiscovery (Standard) 31. august 2025. Det som fortsatt står i mange veiledninger, særlig alternativet for deduplisering ved eksport, finnes ikke lenger i den nye eDiscovery. Fremgangsmåten i Purview-portalen:

1. Under *eDiscovery* oppretter eller åpner du en sak. For eksport trenger kontoen som utfører den, rollen *eDiscovery Manager* og en Microsoft 365 E3- eller E5-lisens.
2. Opprett et søk. Datakilde: alle postbokser, eller én enkelt postboks for en testkjøring.
3. Bruk KQL som spørring. Parentesene er avgjørende fordi `AND` binder sterkere enn `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Uten `kind:email` teller søket også kalenderoppføringer, kontakter og oppgaver. I det beskrevne tilfellet utgjorde dette forskjellen mellom 473'725 og 98'131 treff for én dag. Uten de ytre parentesene gjelder den andre datobetingelsen bare for `sent`, og resultatet inneholder meldinger fra hele postboksbeholdningen.

4. Kjør *Generate statistics*. Ved søk på tvers av leietakeren akselererer dette eksporten betydelig, fordi bare postbokser med treff blir behandlet. Steder som ender med feil, hentes inn med *Retry failed locations* før eksport; ellers mangler de i resultatet, og dette oppdages først etter nedlastingen.
5. Velg *Export* med følgende innstillinger.

| Alternativ | Virkning |
|---|---|
| *Export type: Export items with items report* | Meldinger pluss `items.csv`; den rene rapporten inneholder ikke innhold |
| *Export format: Create PSTs for messages* | Én PST per postboks i stedet for enkeltstående `.msg`-filer |
| *Maximum PST package size* | Fra denne størrelsen deles en PST i deler (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Størrelsen på nedlastingspakkene; må minst tilsvare PST-størrelsen |
| *Organize data from different locations into separate folders or PSTs* | **Slå på**: én fil per postboks, kalt `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Slå av**: alle meldinger havner i mappen `Items` i PST-en i stedet for i postboksens mappestruktur |
| *Give each item a friendly name* | Ingen effekt ved PST-eksport |

Innstillingen *Include folder and path of the source* avgjør den senere importen. Med mappestruktur oppretter importen i journalpostboksen mappene til hver postboks, på språket til den aktuelle brukeren: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Uten mappestruktur ligger alt i én mappe `Items`, og etterimporten har ett enkelt sted å arbeide videre fra.

Nedlastingen leverer ZIP-pakker. Microsoft anbefaler 7-Zip eller et tilsvarende verktøy fremfor Windows Utforsker. Pakkene utløper 14 dager etter opprettelsen. Filen `items.csv` fra prosessrapporten for eksporten (under *Process manager*, Export, Rapporter) dokumenterer hva eksporten inneholder og hva som ble hoppet over; den hører til materialet.

### Deduplisering

Den klassiske Content Search hadde avkrysningsboksen *Enable de-duplication*: Meldinger med samme `InternetMessageId`, `ConversationTopic` og `BodyTagInfo` ble bare eksportert én gang, og de øvrige funnstedene sto i `Results.csv`. Den nye eDiscovery tilbyr ikke dette ved direkte eksport fra et søk. Den eneste dokumenterte veien er et *Review Set*: Last inn søkeresultatene, kjør Analytics, bruk det automatisk opprettede filteret *For Review*, som utelukker duplikater, og eksporter fra Review Set. Dette krever eDiscovery Premium og dermed en E5-lisens for brukeren som utfører arbeidet. I Microsoft Q&A rapporterer flere brukere om duplikater til tross for denne fremgangsmåten; en testkjøring med én dag anbefales før full eksport.

Uten deduplisering gjelder følgende: Enterprise Vault dedupliserer på lagringsnivået (identiske vedlegg og meldingskropper lagres én gang), men ikke på elementnivået. Hver postboks-kopi blir en egen arkivoppføring, et eget søketreff og en egen teller. Tellingen for dokumentasjon kan da ikke utledes fra arkivet, men bare fra `items.csv` mot meldingssporing. Denne beslutningen – akseptere kopier eller eksportere på nytt via et Review Set – tas før importen, ikke etterpå.

## Veiene inn i Enterprise Vault

Enterprise Vault leveres med PST-migratoren for PST-filer, som veiviser i administrasjonskonsollen (*Archives*, høyreklikk, *Import PST*) og som skriptbar variant via Policy Manager EVPM. For etterimport til et journalarkiv har denne veien tre begrensninger.

For det første lisensen. Migratoren krever funksjonen `EVPSTM` (*Exchange PST Migrator*). I et område som kun driver journaling, er den ofte ikke lisensiert, og veiviseren avbryter på første side med *Required license not installed*. Policy Manager bruker den samme migratoren og melder det samme.

For det andre målet. Veiviseren tilbyr ikke journalarkiver som mål, bare postboksarkiver og Internet Mail-arkiver. En import direkte til journalarkivet er bare mulig via EVPM med `ArchiveName`, og den omgår da behandlingsreglene til Journaling Task. Som mål gjenstår et eget Shared Archive i Vault Store for journalen, som søket dekker sammen med journalarkivet.

For det tredje sporbarheten: Migratoren skriver til PST-filen (den markerer migrerte elementer), slik at materialet endres, og migreringsrapportene per fil må avstemmes mot eksporttallene.

Den andre veien krever ingen ekstra lisens: Exchange Journaling Task i Enterprise Vault arkiverer ikke bare journalrapporter. Hvis den gjenkjenner et element som en journalrapport, pakker den det ut og overtar konvoluttdataene. Et vanlig element arkiverer den direkte, slik den alltid har gjort i tiden med enkel Exchange-journaling uten konvolutt. Den som importerer PST-filene til journalpostboksen, får meldingene skrevet av den vanlige oppgaven til det vanlige journalarkivet, med samme oppbevaringskategori, samme indeksering og uten endring av arkiveringslandskapet.

Dette kan bevises med én enkelt testmelding til journalpostboksen: Hvis den kort tid senere ikke lenger står i noen mappe, heller ikke i *Invalid Journal Report*, og søket finner den i journalarkivet, arkiverer oppgaven vanlige meldinger.

Denne veien har to egenskaper som bestemmer prosessen. Oppgaven behandler utelukkende øverste nivå i innboksen, ikke undermapper. Dette kan sees i journalpostboksen: Søkemappen *Initial Trawl With Pending* under *Enterprise Vault Search Folders* teller nøyaktig elementene i innboksen, undermapper er ikke med. Og import via Exchange oppretter alltid mapper; hvordan, står i neste avsnitt.

## Import til journalpostboksen

`New-MailboxImportRequest` importerer en PST-fil på serversiden via Mailbox Replication Service. Filen må ligge på en deling der *Exchange Trusted Subsystem* har full tilgang, og kontoen som utfører arbeidet trenger rollen *Mailbox Import Export*, som som standard ikke er tildelt noen:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

Selve importen:

```powershell
$import = @{
  Mailbox          = "journal@example.com"
  FilePath         = "\\EVSERVER01\pstimport$\unzipped\user@example.com.001.pst"
  TargetRootFolder = "Inbox"
  Name             = "NI-user_example.com.001"
  BatchName        = "Nachimport"
}
New-MailboxImportRequest @import
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-Mailbox` | Målpostboks, her journalpostboksen |
| `-FilePath` | UNC-bane til PST-filen; lokale baner avvises |
| `-TargetRootFolder` | Mappe som PST-innholdet plasseres under; uten angivelse tilordnes PST-mapper til postboksmapper med samme navn |
| `-SourceRootFolder` | Mappe i PST-en det importeres fra; i praksis opprettes likevel selve mappen (se teksten) |
| `-Name` | Entydig navn på jobben; avledet fra filnavnet ved tusenvis av filer |
| `-BatchName` | Gruppering for spørringer og statistikk |

</details>

Tre observasjoner fra testkjøringen avgjør den videre prosessen. Uten `TargetRootFolder` og med en strukturert PST (mappestruktur eksportert) oppretter MRS roten av PST-en som en egen mappe ved siden av innboksen, i tilfellet med en tysk postboks `Oberste-Ebene-des-Informationsspeichers` med undermappene `Posteingang`, `Gesendete-Elemente` og så videre, samt `Recoverable-Items` med `Deletions`. Med `TargetRootFolder "Inbox"` og en flat PST havner alle meldingene i `/Inbox/Items`. Og `SourceRootFolder "Items"` i kombinasjon med `TargetRootFolder "Inbox"` ga i testen også `/Inbox/Items`; nivået ble ikke fjernet.

I alle tre tilfeller ligger meldingene utenfor synligheten til Journaling Task. Det siste steget, å flytte dem flatt til innboksen, kan ikke utføres med Exchange-standardverktøy; det gjøres via EWS. EWS Managed API ligger som `Microsoft.Exchange.WebServices.dll` i installasjonskatalogen til Enterprise Vault, og Vault Service Account har full tilgang til journalpostboksen, slik at ingen ekstra tillatelse er nødvendig. Skriptet kjører på EV-serveren i Windows PowerShell 5.1, fordi biblioteket bygger på .NET Framework.

Dette flyttesteget er samtidig strupingen for hele prosessen: Det avgjør hvor mange meldinger oppgaven ser samtidig.

## Testkjøringen med én postboks

Før tusenvis av filer kjøres, kan hele veien kontrolleres med én enkelt postboks. Postboksen til personen som utførte eksporten, er egnet: Vedkommende har mandatet, det er egne data, og filen er liten. Fremgangsmåten:

1. Kopi av PST-filen i en egen mappe, SHA-256-hash av originalen i en CSV-fil.
2. Import med `TargetRootFolder "Inbox"`, statistikk med `Get-MailboxImportRequestStatistics`: `ItemsTransferred` er forventet antall.
3. Mappestatistikk for journalpostboksen: Hvor ligger elementene?
4. Flytt fra `/Inbox/Items` til innboksen via EWS.
5. Tell de gamle elementene i innboksen hvert par minutter: Elementer med `DateTimeReceived` før starten på den løpende strømmen, er de importerte. Når tallet faller, arkiverer oppgaven.
6. Avstemming: Søk i journalarkivet (i Discovery Accelerator eller EV-søket) med tidsrom og postbokseier, antall treff mot `ItemsTransferred`.

I det beskrevne tilfellet ble 128 elementer importert og 126 arkivert. De to gjenværende var et utkast og et element uten avsender; oppgaven lar elementer uten avsender ligge. Utkast har aldri gått gjennom transporten og hører ikke hjemme i et journalarkiv, derfor utelukker flytteskriptet dem via flagget `IsDraft` uavhengig av språk. Elementer fra mappene *Versions* og *Konflikte* (tidligere versjoner under oppbevaring, synkroniseringsrester) flyttes heller ikke.

Hendelsesloggen `Veritas Enterprise Vault` viser etter testkjøringen hva oppgaven har avvist. Hendelse 3071 (*could not be archived as it may be corrupt*) og 3288 (*no longer archive pending*) gjentas ved hver kjøring for de samme elementene; slike elementer bør flyttes til en mappe utenfor innboksen, slik at oppgaven ikke forsøker dem på nytt ved hver gjennomgang.

## Skript 1: Import i batcher

Det første skriptet kjører i Exchange Management Shell på en Exchange-server. Det oppretter importjobber i batcher, venter, logger status og `ItemsTransferred` per fil til en CSV-fil og fjerner fullførte jobber. Gjentatte kjøringer hopper over filer som allerede står i loggen. Prefikset `NI` står for etterimport.

```powershell
$NIPostfach  = "journal@example.com"
$NIQuellen   = @("\\EVSERVER01\pstimport$\unzipped", "\\EVSERVER01\pstimport$\unzipped2")
$NIProtokoll = "\\EVSERVER01\pstimport$\protokoll\import-protokoll.csv"

function Schreibe-NIImport($Ordner, $Datei, $Request, $Status, $Items, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Ordner = $Ordner; Datei = $Datei
    Request = $Request; Status = $Status; Items = $Items; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIDateien {
  foreach ($q in $NIQuellen) {
    $o = Split-Path $q -Leaf
    Get-ChildItem $q -Filter *.pst | Sort-Object Name | ForEach-Object {
      [pscustomobject]@{
        Ordner = $o; Datei = $_.Name; Pfad = $_.FullName
        MB = [math]::Round($_.Length / 1MB, 1)
        Request = "NI-" + $o + "-" + ($_.Name -replace "@", "_" -replace "\.pst$", "")
      }
    }
  }
}

function Get-NIErledigt {
  if (Test-Path $NIProtokoll) {
    Import-Csv $NIProtokoll |
      Where-Object { $_.Status -in @("angelegt", "Completed", "CompletedWithWarning") } |
      Select-Object -ExpandProperty Request -Unique
  }
}

function Start-NICharge([int]$Anzahl = 100) {
  $erledigt = @(Get-NIErledigt)
  $alle  = @(Get-NIDateien)
  $offen = @($alle | Where-Object { $erledigt -notcontains $_.Request } | Select-Object -First $Anzahl)
  foreach ($d in $offen) {
    $auftrag = @{
      Mailbox          = $NIPostfach
      FilePath         = $d.Pfad
      TargetRootFolder = "Inbox"
      Name             = $d.Request
      BatchName        = "Nachimport-" + $d.Ordner
      ErrorAction      = "Stop"
    }
    try {
      $null = New-MailboxImportRequest @auftrag
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "angelegt" "" ""
    } catch {
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "Fehler-Anlage" "" $_.Exception.Message
    }
  }
  "angelegt: $($offen.Count)   verbleibend: $($alle.Count - $erledigt.Count - $offen.Count)"
}

function Wait-NICharge {
  do {
    $alle = @(Get-MailboxImportRequest -Mailbox $NIPostfach | Where-Object { $_.Name -like "NI-*" })
    $laufend = @($alle | Where-Object { $_.Status -notin @("Completed", "Failed", "CompletedWithWarning") })
    "$(Get-Date -Format HH:mm:ss)  laufend: $($laufend.Count)   fertig: $($alle.Count - $laufend.Count)"
    if ($laufend.Count -gt 0) { Start-Sleep -Seconds 60 }
  } while ($laufend.Count -gt 0)
  foreach ($r in $alle) {
    $s = $r | Get-MailboxImportRequestStatistics
    if (-not $s) { Schreibe-NIImport "" "" $r.Name "Statistik-Fehler" "" ""; continue }
    $ordner = $r.BatchName -replace "^Nachimport-", ""
    $datei  = if ($s.FilePath) { Split-Path $s.FilePath -Leaf } else { "" }
    Schreibe-NIImport $ordner $datei $r.Name $s.Status $s.ItemsTransferred $s.Message
    if ($s.Status -in @("Completed", "CompletedWithWarning")) {
      $r | Remove-MailboxImportRequest -Confirm:$false
    }
  }
  Get-MailboxStatistics $NIPostfach | Format-List ItemCount, TotalItemSize
}

function Resume-NICharge([int]$Anzahl = 200, [int]$MaxLager = 20000) {
  $stat  = Get-MailboxFolderStatistics $NIPostfach
  $lager = ($stat | Where-Object { $_.FolderPath -like "/Inbox/*" } | Measure-Object ItemsInFolder -Sum).Sum
  "in Unterordnern wartend: $lager"
  if ($lager -gt $MaxLager) { "Lager voll, nichts freigegeben"; return }
  Get-MailboxImportRequest -Mailbox $NIPostfach -Status Suspended |
    Select-Object -First $Anzahl |
    Resume-MailboxImportRequest -Confirm:$false
  "freigegeben: $Anzahl"
}

function Get-NIImportStand {
  $p = Import-Csv $NIProtokoll
  $p | Group-Object Status | Select-Object Name, Count | Format-Table -AutoSize
  "Elemente importiert: " + (($p | Where-Object { $_.Status -like "Completed*" } | Measure-Object Items -Sum).Sum)
  "Dateien gesamt: " + @(Get-NIDateien).Count
}
```

<details class="options-details">
<summary>Forklaring av funksjoner</summary>

| Funksjon | Virkning |
|---|---|
| `Get-NIDateien` | Liste over alle PST-filer fra kildemappene med avledet jobbnavn; like filnavn i forskjellige mapper (for eksempel én eksport per dag) forblir mulig å skille |
| `Start-NICharge -Anzahl` | Oppretter opptil N importjobber for filer som ennå ikke er logget |
| `Wait-NICharge` | Venter til ingen jobb lenger kjører, skriver status og `ItemsTransferred` til loggen, fjerner fullførte jobber; mislykkede blir stående for analyse. Statistikk og fjerning kjører via pipeline, fordi `-Identity` i et fjernstyrt skall ikke aksepterer det deserialiserte Identity-objektet |
| `Resume-NICharge -Anzahl -MaxLager` | Frigir stoppede jobber bare når færre enn `MaxLager` meldinger venter i undermappene til innboksen |
| `Get-NIImportStand` | Sammendrag etter status, sum av importerte elementer, totalt antall filer |

</details>

To MRS-atferder bør du kjenne til. Exchange tillater som standard ti samtidige jobber mot samme målpostboks; alle øvrige står i kø med en årsak som `StalledDueToTarget_MailboxCapacityExceeded` eller `StalledDueToTarget_MdbReplication` og rykker frem etter hvert. Dette er arbeidslaststyring, ikke et kapasitetsproblem, selv om navnet kan antyde det. Og `Get-MailboxImportRequest` viser status med forsinkelse; statistikken er mer oppdatert.

Gjennomstrømningen var omtrent seks PST-filer per minutt med en gjennomsnittsstørrelse på noen få megabyte. MRS er dermed betydelig raskere enn Enterprise Vault. Alt som er importert, ligger i journalpostboksen til oppgaven arkiverer og sletter det. `Resume-NICharge` knytter derfor frigivelsen av flere jobber til mengden som ennå ikke er flyttet, slik at postboksen ikke vokser til størrelsen på hele eksporten.

## Skript 2: Flytting og takt

Det andre skriptet kjører som Vault Service Account på EV-serveren i Windows PowerShell 5.1. Det flytter meldinger fra alle undermapper i innboksen flatt til innboksen, lar utkast og mappene *Versions* og *Konflikte* være, og fyller bare på når innboksen ligger under en terskel.

```powershell
Add-Type -Path "C:\Program Files (x86)\Enterprise Vault\Microsoft.Exchange.WebServices.dll"
$NIProtokoll = "E:\pstimport\protokoll\verschieben-protokoll.csv"
$ews = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService(
  [Microsoft.Exchange.WebServices.Data.ExchangeVersion]::Exchange2013_SP1)
$ews.UseDefaultCredentials = $true
$ews.Url = New-Object Uri("https://mailserver01.example.com/EWS/Exchange.asmx")
$mb = New-Object Microsoft.Exchange.WebServices.Data.Mailbox("journal@example.com")

function Schreibe-NIMove($Schritt, $Wert, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Schritt = $Schritt; Wert = $Wert; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIInbox {
  [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ews,
    (New-Object Microsoft.Exchange.WebServices.Data.FolderId(
      [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox, $mb)))
}

function Get-NIRueckstand {
  $inbox = Get-NIInbox
  $f = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsLessThan(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::DateTimeReceived, (Get-Date).AddHours(-2))
  $r = $inbox.FindItems($f, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(1)))
  [pscustomobject]@{ Zeit = Get-Date -Format "HH:mm:ss"; Posteingang = $inbox.TotalCount; Alt = $r.TotalCount }
}

function Get-NIUnterordner {
  $inbox = Get-NIInbox
  $liste = New-Object System.Collections.ArrayList
  $offset = 0
  do {
    $fv = New-Object Microsoft.Exchange.WebServices.Data.FolderView(500, $offset)
    $fv.Traversal = [Microsoft.Exchange.WebServices.Data.FolderTraversal]::Deep
    $res = $inbox.FindFolders($fv)
    foreach ($f in $res.Folders) { $null = $liste.Add($f) }
    $offset += 500
  } while ($res.MoreAvailable)
  $ausgeschlossen = @("Versions", "Konflikte", "Conflicts", "Conflitti", "Conflits")
  $liste | Where-Object { $_.TotalCount -gt 0 -and $_.DisplayName -notin $ausgeschlossen }
}

function Move-NIPortion([int]$Max = 2000) {
  $inbox = Get-NIInbox
  $keinEntwurf = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsEqualTo(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::IsDraft, $false)
  $n = 0
  foreach ($ordner in Get-NIUnterordner) {
    $runden = 0
    do {
      $seite = $ordner.FindItems($keinEntwurf, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(100)))
      if ($seite.Items.Count -gt 0) {
        $ids = New-Object 'System.Collections.Generic.List[Microsoft.Exchange.WebServices.Data.ItemId]'
        foreach ($i in $seite.Items) { $ids.Add($i.Id) }
        $antwort = $ews.MoveItems($ids, $inbox.Id)
        $fehler = @($antwort | Where-Object { $_.Result -ne "Success" }).Count
        if ($fehler -gt 0) { Schreibe-NIMove "Fehler" $fehler $ordner.DisplayName; break }
        $n += $seite.Items.Count
      }
      $runden++
    } while ($seite.Items.Count -gt 0 -and $n -lt $Max -and $runden -lt 200)
    if ($n -ge $Max) { break }
  }
  Schreibe-NIMove "Verschoben" $n ""
  return $n
}

function Start-NIDauerlauf([int]$Portion = 2000, [int]$MaxPosteingang = 3000) {
  $stop = "E:\pstimport\protokoll\STOP.txt"
  while (-not (Test-Path $stop)) {
    $s = Get-NIRueckstand
    if ($s.Posteingang -gt $MaxPosteingang) {
      "$($s.Zeit)  Posteingang $($s.Posteingang)  alt $($s.Alt)  warte auf EV"
      Schreibe-NIMove "Rueckstand" $s.Alt ("Posteingang " + $s.Posteingang)
      Start-Sleep -Seconds 300
      continue
    }
    $n = Move-NIPortion -Max $Portion
    "$(Get-Date -Format HH:mm:ss)  verschoben $n  (Posteingang vorher $($s.Posteingang))"
    if ($n -eq 0) { Start-Sleep -Seconds 600 }
  }
  "STOP-Datei gefunden, Dauerlauf beendet"
}

function Get-NIMoveStand {
  $p = Import-Csv $NIProtokoll
  "verschoben gesamt: " + (($p | Where-Object { $_.Schritt -eq "Verschoben" } | Measure-Object Wert -Sum).Sum)
  "Fehler: " + @($p | Where-Object { $_.Schritt -eq "Fehler" }).Count
  Get-NIRueckstand
}
```

<details class="options-details">
<summary>Forklaring av funksjoner</summary>

| Funksjon | Virkning |
|---|---|
| `Get-NIRueckstand` | Antall elementer i innboksen og antall elementer som er eldre enn to timer; sistnevnte er de etterimporterte, fordi den løpende journalstrømmen bare har aktuelle mottakstidspunkter |
| `Get-NIUnterordner` | Alle undermapper i innboksen med innhold, uten *Versions* og *Konflikte* |
| `Move-NIPortion -Max` | Flytter opptil N meldinger uten utkastflagg til innboksen, sidevis med 100 via `MoveItems` med en typet `ItemId`-liste; en PowerShell-array passer ikke til signaturen |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Endeløs løkke: Mål innboksen hvert femte minutt, fyll bare på når den ligger under terskelen; avsluttes så snart filen `STOP.txt` finnes |
| `Get-NIMoveStand` | Sum av flyttede elementer, antall feil, gjeldende restmengde |

</details>

Terskelen `MaxPosteingang` er det viktigste tallet i prosessen. Den må ligge over oppgavens normale arbeidsbeholdning (i det beskrevne tilfellet rundt 1'500 rapporter ved 100 meldinger per minutt og omtrent 20 minutters forsinkelse) og lavt nok til at oppgaven aldri lar den løpende strømmen bli liggende. Neste avsnitt viser hvorfor dette er nødvendig.

## Måleverdier og strupingen

Med standardinnstillingene for Journaling Task (5 samtidige tilkoblinger til Exchange-serveren, 1'000 elementer per gjennomgang) arkiverte Enterprise Vault de etterimporterte meldingene den første natten med omtrent 5'600 per time. Om morgenen viste mappestatistikken hva dette kostet: Innboksen sto på 16'500 i stedet for 1'500. Den løpende journalstrømmen, omtrent 6'000 rapporter per time, var mer enn to timer forsinket. Oppgaven hadde behandlet de gamle meldingene samtidig og gitt aktuelle rapporter lavere prioritet.

Kapasiteten til oppgaven er summen av begge strømmene. Etterimporten kan bare få den delen som er igjen etter den daglige driften. Derfor måler den kontinuerlige kjøringen selve innboksen, ikke antallet gamle elementer, og fyller bare på når strømmen er aktuell.

Tre målinger viser hvor grensen til oppgaven går. Køene mellom oppgaven og Storage Service er MSMQ-køer; hvis Storage Service-køene er tomme mens oppgavens køer er fylte, bremser oppgaven ved lesing fra Exchange, ikke lagringen:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

På Exchange-siden teller RPC-latensen per klienttype, ikke antallet forespørsler; verdier under 10 ms betyr at serveren har reserver:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Og strupingspolicyen for Vault Service Account: Veritas krever en policy med `RcaMaxConcurrency: Unlimited`; med standardpolicyen begrenser Exchange samtidige MAPI-tilkoblinger per konto, og ytterligere tilkoblinger fra oppgaven kommer ikke frem:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Hvis oppgaven bremser, finnes den riktige innstillingen i oppgaveegenskapene (*Enterprise Vault Servers*, server, *Tasks*, Journaling Task, fanen *Settings*):

| Innstilling | Standard | Virkning |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Tråder som leser parallelt fra postboksen; virker lineært så lenge Exchange-latens og Storage-køer forblir upåfallende |
| *Maximum number of items per target per pass* | 1000 | Elementer per gjennomgang; starter sjeldnere på nytt ved stor restmengde |

Endringer gjelder etter omstart av oppgaven (høyreklikk, *Stop*, deretter *Start*). En dobling med påfølgende måling via tidsstemplene i flytteloggen er fornuftig, deretter neste trinn. En annen Journaling Task mot samme postboks er ikke mulig; en journalpostboks er tilordnet nøyaktig én oppgave. Reell parallellitet oppnår du med en annen journalpostboks og egen oppgave på en annen EV-server, der den andre halvparten av filene importeres til arkivet. Dette er en endring av arkiveringslandskapet og må koordineres med driften.

## Hva som blir synlig i randsonen

En etterimport i denne størrelsesordenen avdekker ting ingen tidligere har sett på. To fra det beskrevne tilfellet, fordi de sannsynligvis gjentar seg.

Overvåkingen meldte *RPC Requests/sec* over terskelen på Exchange-serveren, med standardverdier på 60 og 70 forespørsler per sekund fra sjekkmalen. Serverens grunnbelastning lå allerede over den kritiske terskelen uten import. Forespørsler per sekund er et belastningstall; helsetallet er latensen, og den lå under ett millisekund. Terskelen bør settes på nytt basert på grunnbelastningen fra overvåkingshistorikken, omtrent til det dobbelte og tredobbelte av den typiske dagstoppen, mens latensterskelen beholdes.


I journalpostboksens innboks hadde det i årevis ligget fire elementer som oppgaven prøvde på nytt i hver gjennomgang og avviste med hendelse 3071. De hadde ikke blitt oppdaget i noen rapportering, fordi ingen regelmessig leste hendelsesloggen `Veritas Enterprise Vault`. Det samme gjaldt oppsamlingspostboksen fra `JournalingReportNdrTo`: 117'000 journalrapporter som ikke kunne leveres siden 2020, en indikasjon på at arkiveringen gjentatte ganger hadde mistet rapporter allerede før det aktuelle gapet.

## Dokumentasjon og avslutning

Ved slutten av etterimporten finnes tre filer og ett søk. `items.csv` fra Purview-eksporten dokumenterer hva som ble eksportert. `import-protokoll.csv` inneholder per fil hva Exchange tok imot. `verschieben-protokoll.csv` inneholder hva som ble overlevert til oppgaven, og når. Søket i journalarkivet med tidsrommet for gapet, fordelt på dager, gir antallet i arkivet. Forskjellene mellom disse tallene kan forklares: utkast, elementer uten avsender, mappene *Versions* og *Konflikte*, postbokser utenfor oppbevaringen. Nettopp denne forklaringen er utsagnet til compliance, sammen med de to grensene som ingen etterimport opphever: manglende konvoluttmottakere og, uten deduplisering, kopier i stedet for meldinger.

Deretter ryddes det opp. De tomme undermappene og utkastene som ble liggende igjen, fjernes fra journalpostboksen, delingen med PST-filene fjernes, og PST-filene slettes når avstemmingen er avsluttet. Rolletildelingen *Mailbox Import Export* trekkes tilbake, oppgaveinnstillingene beholdes dersom driften kjenner måleverdiene. Selve hendelsen får overvåking av journalleveringen, siden gapet ble oppdaget gjennom en tilfeldig melding, ikke av overvåking.

## Kilder

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): fullstendig liste over eksportalternativene i den nye eDiscovery, pakkestørrelser, oppførselen til *Organize data* og *Include folder and path*, 14-dagers utløp for pakkene.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): sammenligningsegenskaper for klassisk deduplisering og informasjonen om avviklingen av de klassiske grensesnittene 31. august 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): eksportveien via et Review Set, som er den eneste i den nye eDiscovery som utelukker duplikater.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): bekreftelse på at direkte eksport ikke lenger tilbyr deduplisering, med erfaringsrapporter om Review Set-veien.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): parametere for importjobben, forutsetninger for deling og rollen *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): arbeidslaststyring med ti samtidige jobber per mål og `StalledDueToTarget`-tilstandene som forventet oppførsel.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): prosessen for PST-migratoren, tilgangskravene til Storage Service og de tillatte målarkivtypene.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): erfaringer fra fellesskapet om hvorfor veiviseren ikke tilbyr journalarkiver, og hvordan EVPM arbeider med `ArchiveName`.
