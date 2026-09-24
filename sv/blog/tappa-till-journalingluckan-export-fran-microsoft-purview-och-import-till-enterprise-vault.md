---
title: "Täppa till journalingluckan: export från Microsoft Purview och import till Enterprise Vault"
navTitle: "Journalinglucka"
description: "Om journalrapporter från Exchange Online uteblir under några dagar går de inte att skapa i efterhand. Innehållet finns dock fortfarande kvar i postlådorna. Så exporterar du det som PST via nya Purview eDiscovery, varför PST Migrator i Enterprise Vault oftast inte är ett alternativ och hur efterimporten via journalpostlådan fungerar utan att tränga undan det löpande journalflödet, med två generiska skript för import, flytt och loggning."
date: "2026-09-22"
kategorie: "Arkivering och journaling"
timeToRead: "18 min lästid"
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
slug: "tappa-till-journalingluckan-export-fran-microsoft-purview-och-import-till-enterprise-vault"
translationId: "article-5b24118ac7567d96"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/sv/blog/tappa-till-journalingluckan-export-fran-microsoft-purview-och-import-till-enterprise-vault
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:42:07.249Z
translationReview: automatic
---

# Täppa till journalingluckan: export från Microsoft Purview och import till Enterprise Vault

Journaling är en transportprocess. Exchange skapar journalrapporten när meddelandet passerar transporten och levererar den till journalpostlådan. Om denna leverans misslyckas, till exempel eftersom en anslutning eller ett mottagarobjekt är felkonfigurerat, finns det ingen funktion som hämtar rapporten i efterhand. Det som passerade transporten under den perioden saknas i arkivet.

Meddelandeinnehållet finns dock fortfarande i de berördas postlådor och, med en bevarandeprincip utan utgångstid, även om användarna har raderat meddelandena. Därmed återstår den enda genomförbara vägen för komplettering: exportera perioden från alla postlådor och importera kopiorna till journalarkivet. Följande procedur använder nya eDiscovery i Microsoft Purview på exportsidan och Veritas Enterprise Vault 15 på importsidan. Mätvärdena kommer från en efterimport av flera hundra tusen meddelanden, och de två skripten skapades i samband med detta.

## Vad en export kan ersätta och vad den inte kan

En journalrapport består av kuvertet och innehållet. Kuvertet innehåller de mottagare som transporten faktiskt levererade till: även hemliga kopior och de upplösta medlemmarna i distributionslistor. En postlådekopia innehåller inte dessa uppgifter. Vem som fick ett meddelande som hemlig kopia går inte längre att fastställa efter efterimporten. Själva innehållet kan däremot rekonstrueras fullständigt, förutsatt att två villkor är uppfyllda.

För det första måste det finnas bevarande över postlådorna, så att även raderade meddelanden fortfarande finns bland återställningsbara objekt. Detta kontrollerar du i Security & Compliance-PowerShell:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

En princip med `RetentionComplianceAction: Keep` och `RetentionDuration: Unlimited` via `ExchangeLocation: All` säkrar innehållet fullständigt. Undantagslistan i `ExchangeLocationException` anger de postlådor som detta inte gäller för. För dessa kan endast det som fortfarande finns kvar rekonstrueras.

För det andra räknar en export postlådekopior, inte meddelanden. Ett meddelande till femton interna mottagare förekommer femton gånger i resultatet. I det beskrivna fallet motsvarade 98'131 kopior för en enskild dag omkring 6'800 unika meddelanden, fastställda med meddelandespårning. Hur dessa kopior ska hanteras avgörs av Compliance, inte IT (se avsnittet om deduplicering).

## Fastställa perioden

Början på luckan finns i själva journalpostlådan, på Exchange-servern som har den. Tidpunkten för den senast levererade rapporten är början:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Detsamma gäller postlådan som anges i `JournalingReportNdrTo` i transportkonfigurationen. Den samlar olevererbara journalrapporter, och dess senaste post bekräftar tidpunkten. Slutet på luckan är den tidpunkt då journalregeln ändrades till det reparerade målet. Mellan dessa ligger exportperioden, i UTC.

Meddelandespårningen ger målvärdet för avstämningen: en rad per mottagare och meddelande, med `MessageId`. En historisk rapport täcker hela perioden:

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

De unika `MessageId`-värdena från denna rapport är det antal som efterimporten till slut mäts mot.

## Export från nya eDiscovery

Microsoft stängde av de klassiska gränssnitten för Content Search och eDiscovery (Standard) den 31 augusti 2025. Det som fortfarande anges i många instruktioner, särskilt alternativet för deduplicering vid export, finns inte längre i nya eDiscovery. Processen i Purview-portalen:

1. Skapa eller öppna ett ärende under *eDiscovery*. För exporten behöver det utförande kontot rollen *eDiscovery Manager* och en Microsoft 365 E3- eller E5-licens.
2. Skapa en sökning. Datakälla: alla postlådor eller, för en testkörning, en enskild postlåda.
3. Använd KQL som fråga. Parenteserna är avgörande eftersom `AND` har starkare bindning än `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Utan `kind:email` räknar sökningen även kalenderposter, kontakter och uppgifter. I det beskrivna fallet innebar detta skillnaden mellan 473'725 och 98'131 träffar för en dag. Utan de yttre parenteserna gäller det andra datumvillkoret endast för `sent`, och resultatet innehåller meddelanden från hela postlådans innehåll.

4. Kör *Generate statistics*. Vid sökningar i hela klientorganisationen påskyndar detta exporten avsevärt eftersom endast postlådor med träffar behandlas. Platser som avslutas med fel ska hämtas in via *Retry failed locations* innan exporten sker, annars saknas de i resultatet och detta märks först efter nedladdningen.
5. Välj *Export* med följande inställningar.

| Alternativ | Effekt |
|---|---|
| *Export type: Export items with items report* | Meddelanden plus `items.csv`; den rena rapporten innehåller inget innehåll |
| *Export format: Create PSTs for messages* | En PST per postlåda i stället för enskilda `.msg`-filer |
| *Maximum PST package size* | Vid denna storlek delas en PST upp i delar (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Storlek på nedladdningspaketen; måste minst motsvara PST-storleken |
| *Organize data from different locations into separate folders or PSTs* | **Aktivera**: en fil per postlåda, med namnet `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Inaktivera**: alla meddelanden hamnar i mappen `Items` i PST-filen i stället för i postlådans mappstruktur |
| *Give each item a friendly name* | Saknar effekt vid PST-export |

Inställningen *Include folder and path of the source* avgör den senare importen. Med mappstruktur skapar importen i journalpostlådan mapparna för varje postlåda, på respektive användares språk: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Utan mappstruktur ligger allt i en enda mapp `Items`, och efterimporten har en enda plats att arbeta vidare från.

Nedladdningen ger Zip-paket. Microsoft rekommenderar 7-Zip eller ett jämförbart verktyg i stället för Windows Explorer. Paketen upphör att gälla 14 dagar efter att de skapats. Filen `items.csv` från exportens processrapport (under *Process manager*, Export, Rapporter) är beviset på vad exporten innehåller och vad som hoppades över; den hör till underlaget.

### Deduplicering

Den klassiska Content Search hade kryssrutan *Enable de-duplication*: meddelanden med samma `InternetMessageId`, `ConversationTopic` och `BodyTagInfo` exporterades bara en gång, medan övriga fyndplatser fanns i `Results.csv`. Nya eDiscovery erbjuder inte detta vid direkt export från en sökning. Den enda dokumenterade vägen är ett *Review Set*: läs in sökresultaten, kör Analytics, använd det automatiskt skapade filtret *For Review*, som utesluter dubbletter, och exportera från Review Set. Detta kräver eDiscovery Premium och därmed en E5-licens för den utförande användaren. Flera användare rapporterar i Microsoft Q&A om dubbletter trots denna procedur. En testkörning med en dag rekommenderas före full export.

Utan deduplicering gäller följande: Enterprise Vault deduplicerar på lagringsnivå (identiska bilagor och meddelandetexter lagras en gång), men inte på objektnivå. Varje postlådekopia blir en egen arkivpost, en egen träff i sökningen och en egen räknare. Räkningen för bevisning kan då inte härledas från arkivet utan endast från `items.csv` jämfört med meddelandespårningen. Beslutet att acceptera kopior eller exportera på nytt via ett Review Set fattas före importen, inte efteråt.

## Vägarna in i Enterprise Vault

Enterprise Vault innehåller PST Migrator för PST-filer, både som guide i administrationskonsolen (*Archives*, högerklick, *Import PST*) och som skriptbar variant via Policy Manager EVPM. För efterimport till ett journalarkiv har denna väg tre begränsningar.

För det första licensen. Migrator kräver funktionen `EVPSTM` (*Exchange PST Migrator*). I en Site som endast använder journaling är den ofta inte licensierad, och guiden avbryts på första sidan med *Required license not installed*. Policy Manager använder samma Migrator och visar samma meddelande.

För det andra målet. Guiden erbjuder inte journalarkiv som mål, utan endast postlådearkiv och Internet Mail Archives. Import direkt till journalarkivet är endast möjlig via EVPM med `ArchiveName`, och då kringgås Journaling Tasks bearbetningsregler. Som mål återstår ett eget Shared Archive i journalens Vault Store, som sökningen omfattar tillsammans med journalarkivet.

För det tredje spårbarheten: Migrator skriver till PST-filen (den markerar migrerade objekt), vilket innebär att underlaget ändras, och migreringsrapporterna per fil måste stämmas av mot exportsiffrorna.

Den andra vägen kräver ingen extra licens: Exchange Journaling Task i Enterprise Vault arkiverar inte bara journalrapporter. Om den identifierar ett objekt som en journalrapport packar den upp det och tar över kuvertdata. Ett vanligt objekt arkiverar den direkt, precis som den alltid har gjort under tider med enkel Exchange-journaling utan kuvert. Den som importerar PST-filerna till journalpostlådan får meddelandena skrivna av den reguljära tasken till det reguljära journalarkivet, med samma bevarandekategori, samma indexering och utan förändring av arkiveringslandskapet.

Detta kan bevisas med ett enskilt testmeddelande till journalpostlådan: om det kort därefter inte längre finns i någon mapp, inte heller i *Invalid Journal Report*, och sökningen hittar det i journalarkivet, arkiverar tasken vanliga meddelanden.

Denna väg har två egenskaper som styr proceduren. Tasken bearbetar enbart den översta nivån i Inkorgen, inga undermappar. Det kan utläsas i journalpostlådan: sökmappen *Initial Trawl With Pending* under *Enterprise Vault Search Folders* räknar exakt objekten i Inkorgen, utan undermappar. Och import via Exchange skapar alltid mappar; hur framgår av nästa avsnitt.

## Import till journalpostlådan

`New-MailboxImportRequest` importerar en PST-fil på serversidan via Mailbox Replication Service. Filen måste ligga på en utdelning där *Exchange Trusted Subsystem* har fullständig behörighet, och det utförande kontot behöver rollen *Mailbox Import Export*, som som standard inte är tilldelad någon:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

Själva importen:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Mailbox` | Målpostlåda, här journalpostlådan |
| `-FilePath` | UNC-sökväg till PST-filen; lokala sökvägar avvisas |
| `-TargetRootFolder` | Mapp där PST-innehållet placeras; utan angivelse mappas PST-mappar till postlådemappar med samma namn |
| `-SourceRootFolder` | Mapp i PST-filen från vilken importen börjar; i praktiken skapas själva mappen ändå (se texten) |
| `-Name` | Unikt namn på jobbet; härlett från filnamnet vid tusentals filer |
| `-BatchName` | Gruppering för frågor och statistik |

</details>

Tre observationer från testkörningen styr den fortsatta processen. Utan `TargetRootFolder` och med en strukturerad PST (mappstruktur exporterad) skapar MRS PST-filens rot som en egen mapp bredvid Inkorgen, i fallet med en tysk postlåda `Oberste-Ebene-des-Informationsspeichers` med undermapparna `Posteingang`, `Gesendete-Elemente` och så vidare, samt `Recoverable-Items` med `Deletions`. Med `TargetRootFolder "Inbox"` och en platt PST hamnar alla meddelanden i `/Inbox/Items`. Och `SourceRootFolder "Items"` i kombination med `TargetRootFolder "Inbox"` gav i testet också `/Inbox/Items`; nivån skalades inte bort.

I samtliga tre fall ligger meddelandena utanför Journaling Tasks synfält. Det sista steget, att flytta dem platt till Inkorgen, kan inte utföras med inbyggda Exchange-funktioner; det sker via EWS. EWS Managed API finns som `Microsoft.Exchange.WebServices.dll` i installationskatalogen för Enterprise Vault, och Vault Service Account har fullständig behörighet till journalpostlådan, så ingen ytterligare behörighet krävs. Skriptet körs på EV-servern i Windows PowerShell 5.1 eftersom biblioteket bygger på .NET Framework.

Detta flyttsteg är samtidigt strypningen för hela proceduren: det avgör hur många meddelanden tasken ser åt gången.

## Testkörningen med en postlåda

Innan tusentals filer körs kan hela vägen kontrolleras med en enda postlåda. Lämpligen används postlådan för den person som genomförde exporten: personen har mandatet, det är egna data och filen är liten. Processen:

1. Kopiera PST-filen till en egen mapp och spara SHA-256-hashen för originalet i en CSV-fil.
2. Importera med `TargetRootFolder "Inbox"`, ta fram statistik med `Get-MailboxImportRequestStatistics`: `ItemsTransferred` är målvärdet.
3. Ta fram mappstatistik för journalpostlådan: var ligger objekten?
4. Flytta från `/Inbox/Items` till Inkorgen via EWS.
5. Räkna gamla objekt i Inkorgen med några minuters mellanrum: objekt med `DateTimeReceived` före början av det löpande flödet är de importerade. När antalet minskar arkiverar tasken.
6. Stäm av: sök i journalarkivet (i Discovery Accelerator eller EV-sökningen) med period och postlådeägare, och jämför antalet träffar med `ItemsTransferred`.

I det beskrivna fallet importerades 128 objekt och 126 arkiverades. De två återstående var ett utkast och ett objekt utan avsändare; tasken lämnar objekt utan avsändare kvar. Utkast har aldrig passerat transporten och hör inte hemma i något journalarkiv, därför utesluter flyttskriptet dem via flaggan `IsDraft`, språkoberoende. Objekt från mapparna *Versions* och *Konflikte* (tidigare versioner under bevarande, synkroniseringsrester) flyttas inte heller.

Händelseloggen `Veritas Enterprise Vault` visar efter testkörningen vad tasken avvisade. Händelse 3071 (*could not be archived as it may be corrupt*) och 3288 (*no longer archive pending*) upprepas vid varje genomgång för samma objekt; sådana objekt bör flyttas till en mapp utanför Inkorgen så att tasken inte försöker igen vid varje genomgång.

## Skript 1: Import i batchar

Det första skriptet körs i Exchange Management Shell på en Exchange-server. Det skapar importjobb i batchar, väntar, loggar status och `ItemsTransferred` per fil i en CSV-fil och tar bort slutförda jobb. Upprepade körningar hoppar över filer som redan finns i loggen. Prefixet `NI` står för efterimport.

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
<summary>Förklaring av funktioner</summary>

| Funktion | Effekt |
|---|---|
| `Get-NIDateien` | Lista över alla PST-filer från källmapparna med härlett jobbnamn; samma filnamn i olika mappar (exempelvis en export per dag) förblir särskiljbara |
| `Start-NICharge -Anzahl` | Skapar upp till N importjobb för filer som ännu inte har loggats |
| `Wait-NICharge` | Väntar tills inga jobb längre körs, skriver status och `ItemsTransferred` till loggen och tar bort slutförda jobb; misslyckade jobb lämnas kvar för analys. Statistik och borttagning körs via pipelinen eftersom `-Identity` i ett fjärranslutet skal inte accepterar det deserialiserade Identity-objektet |
| `Resume-NICharge -Anzahl -MaxLager` | Frisläpper pausade jobb endast om färre än `MaxLager` meddelanden väntar i Inkorgens undermappar |
| `Get-NIImportStand` | Sammanfattning per status, summa av importerade objekt, totalt antal filer |

</details>

Du bör känna till två beteenden hos MRS. Exchange tillåter som standard tio samtidiga jobb till samma målpostlåda; alla övriga står med en orsak som `StalledDueToTarget_MailboxCapacityExceeded` eller `StalledDueToTarget_MdbReplication` i kön och flyttas fram senare. Det är arbetsbelastningshantering, inte ett kapacitetsproblem, även om namnet antyder det. Och `Get-MailboxImportRequest` visar status med fördröjning; statistiken är mer aktuell.

Genomströmningen låg på omkring sex PST-filer per minut vid en genomsnittlig storlek på några få megabyte. MRS är därmed betydligt snabbare än Enterprise Vault. Allt som importerats ligger i journalpostlådan tills tasken arkiverar och raderar det. `Resume-NICharge` kopplar därför frisläppningen av ytterligare jobb till mängden som ännu inte har flyttats, så att postlådan inte växer till storleken på hela exporten.

## Skript 2: Flytt och takt

Det andra skriptet körs som Vault Service Account på EV-servern i Windows PowerShell 5.1. Det flyttar meddelanden från alla undermappar i Inkorgen platt till Inkorgen, utelämnar utkast samt mapparna *Versions* och *Konflikte*, och fyller endast på när Inkorgen ligger under ett tröskelvärde.

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
<summary>Förklaring av funktioner</summary>

| Funktion | Effekt |
|---|---|
| `Get-NIRueckstand` | Antal objekt i Inkorgen och antal objekt som är äldre än två timmar; de senare är de efterimporterade eftersom det löpande journalflödet endast har aktuella mottagningstider |
| `Get-NIUnterordner` | Alla innehållande undermappar i Inkorgen, utan *Versions* och *Konflikte* |
| `Move-NIPortion -Max` | Flyttar upp till N meddelanden utan utkastflagga till Inkorgen, sidvis med 100 via `MoveItems` med en typad `ItemId`-lista; en PowerShell-array passar inte signaturen |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Oändlig slinga: mäter Inkorgen var femte minut och fyller endast på när den ligger under tröskelvärdet; avslutas så snart filen `STOP.txt` finns |
| `Get-NIMoveStand` | Summa av flyttade objekt, antal fel, aktuellt eftersläp |

</details>

Tröskelvärdet `MaxPosteingang` är procedurens viktigaste siffra. Det måste ligga över taskens normala arbetslager (i det beskrivna fallet omkring 1'500 rapporter vid 100 meddelanden per minut och omkring 20 minuters fördröjning) och så lågt att tasken aldrig lämnar det löpande flödet kvar. Varför detta behövs visas i nästa avsnitt.

## Mätvärden och strypningen

Med Journaling Tasks standardinställningar (5 samtidiga anslutningar till Exchange-servern, 1'000 objekt per genomgång) arkiverade Enterprise Vault de efterimporterade meddelandena den första natten med omkring 5'600 per timme. På morgonen visade mappstatistiken vad detta hade kostat: Inkorgen låg på 16'500 i stället för 1'500. Det löpande journalflödet, omkring 6'000 rapporter per timme, låg mer än två timmar efter. Tasken hade bearbetat de gamla meddelandena och prioriterat ned de aktuella rapporterna.

Taskens kapacitet är summan av båda flödena. Efterimporten får endast använda den del som återstår efter den dagliga verksamheten. Därför mäter den kontinuerliga körningen själva Inkorgen, inte antalet gamla objekt, och fyller endast på när flödet är aktuellt.

Tre mätningar visar var taskens gräns ligger. Köerna mellan tasken och Storage Service är MSMQ-köer; om Storage Services köer är tomma medan taskens är fyllda bromsar tasken vid läsning från Exchange, inte lagringen:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

På Exchange-sidan räknar RPC-latensen per klienttyp, inte antalet förfrågningar; värden under 10 ms innebär att servern har reserver:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Och strypningsprincipen för Vault Service Account: Veritas kräver en princip med `RcaMaxConcurrency: Unlimited`; med standardprincipen begränsar Exchange samtidiga MAPI-anslutningar per konto, och ytterligare taskanslutningar kommer inte fram:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Om tasken bromsar finns den relevanta inställningen i taskegenskaperna (*Enterprise Vault Servers*, server, *Tasks*, Journaling Task, fliken *Settings*):

| Inställning | Standard | Effekt |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Trådar som parallellt läser från postlådan; verkar linjärt så länge Exchange-latens och Storage-köer förblir opåverkade |
| *Maximum number of items per target per pass* | 1000 | Objekt per genomgång; vid stort eftersläp behöver arbetet påbörjas mer sällan |

Ändringar gäller efter omstart av tasken (högerklicka, *Stop*, sedan *Start*). Det är lämpligt att fördubbla och sedan mäta med tidsstämplarna i flyttloggen, därefter gå till nästa steg. En andra Journaling Task mot samma postlåda är inte möjlig; en journalpostlåda är tilldelad exakt en task. Verklig parallellism uppnås med en andra journalpostlåda och en egen task på en andra EV-server, till vars arkiv den andra hälften av filerna importeras. Detta är en förändring av arkiveringslandskapet och måste samordnas med driften.

## Vad som framträder i utkanten

En efterimport i denna storlek blottlägger sådant som ingen tidigare har tittat på. Två exempel från det beskrivna fallet, eftersom de sannolikt återkommer.

Övervakningen rapporterade *RPC Requests/sec* över tröskelvärdet på Exchange-servern, med standardvärdena 60 och 70 förfrågningar per sekund från kontrollmallen. Serverns grundbelastning låg redan över den kritiska tröskeln utan import. Förfrågningar per sekund är ett belastningsvärde; hälsovärdet är latensen, och den låg under en millisekund. Tröskelvärdet bör sättas om baserat på grundbelastningen i övervakningshistoriken, till exempel till två och tre gånger den typiska dagstoppen, medan latensgränsen behålls.


I journalpostlådans Inkorg låg sedan åratal fyra objekt som tasken försökte behandla på nytt vid varje genomgång och avvisade med händelse 3071. De hade inte uppmärksammats i någon utvärdering eftersom ingen regelbundet läste händelseloggen `Veritas Enterprise Vault`. Detsamma gällde uppsamlingspostlådan från `JournalingReportNdrTo`: 117'000 olevererbara journalrapporter sedan 2020, en indikation på att arkiveringen redan före den aktuella luckan upprepade gånger hade förlorat rapporter.

## Bevisning och avslut

I slutet av efterimporten finns tre filer och en sökning. `items.csv` från Purview-exporten bevisar vad som exporterades. `import-protokoll.csv` innehåller per fil vad Exchange tog emot. `verschieben-protokoll.csv` innehåller vad som överlämnades till tasken och när. Sökningen i journalarkivet med luckans tidsperiod, uppdelad per dag, ger antalet i arkivet. Skillnaderna mellan dessa siffror är förklarliga: utkast, objekt utan avsändare, mapparna *Versions* och *Konflikte*, postlådor utanför bevarande. Just denna förklaring är budskapet till Compliance, tillsammans med de två begränsningar som ingen efterimport upphäver: saknade kuvertmottagare och, utan deduplicering, kopior i stället för meddelanden.

Därefter återstår upprensningen. De tomma undermapparna och kvarlämnade utkasten tas bort från journalpostlådan, utdelningen med PST-filerna tas bort och PST-filerna raderas så snart avstämningen är klar. Rolltilldelningen *Mailbox Import Export* tas tillbaka, medan taskinställningarna lämnas kvar om driften känner till mätvärdena. Och själva incidenten får övervakning av journalleveransen, eftersom luckan upptäcktes genom ett slumpmässigt meddelande, inte genom övervakning.

## Källor

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): fullständig lista över exportalternativ i nya eDiscovery, paketstorlekar, beteendet för *Organize data* och *Include folder and path*, paketens utgång efter 14 dagar.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): jämförelseegenskaper för klassisk deduplicering och hänvisningen till att de klassiska gränssnitten stängdes av den 31 augusti 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): exportvägen via ett Review Set, som i nya eDiscovery är den enda som utesluter dubbletter.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): bekräftelse på att direkt export inte längre erbjuder deduplicering, med erfarenhetsrapporter om Review Set-vägen.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): parametrar för importjobbet, förutsättningar för utdelning och rollen *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): arbetsbelastningshantering med tio samtidiga jobb per mål och `StalledDueToTarget`-tillstånden som förväntat beteende.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): PST Migrators process, Storage Services åtkomstkrav och tillåtna målarkivtyper.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): erfarenheter från communityn om varför guiden inte erbjuder journalarkiv och hur EVPM arbetar med `ArchiveName`.
