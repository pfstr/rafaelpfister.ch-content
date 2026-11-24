---
title: "Microsoft To Do avvikles: overgang til Planner sett fra et administratorperspektiv"
navTitle: "To Do avvikles"
description: "Microsoft avvikler To Do-appene 30. juni 2027. Bedriftskontoer går over til Planner, eksisterende oppgaver migreres ikke, og Planner lagrer nye private oppgaver utenfor Exchange-postboksen. For sveitsiske leiere endres dermed datalagringssted og samsvarsdekning; berørte brukere kan kartlegges via Microsoft Graph."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min. lesetid"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-avvikles-overgang-til-planner-sett-fra-et-administratorperspektiv"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/no/blog/microsoft-to-do-avvikles-overgang-til-planner-sett-fra-et-administratorperspektiv
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:01:13.399Z
translationReview: automatic
---

# Microsoft To Do avvikles: overgang til Planner sett fra et administratorperspektiv

Microsoft avvikler Microsoft To Do. Appene for nett, Windows, Mac, iOS og Android blir tatt ut av drift 30. juni 2027, både for jobb- og skolekontoer og for private Microsoft-kontoer. For jobb- og skolekontoer overtar Microsoft Planner de personlige oppgavene; den reviderte visningen «Mine oppgaver» (My tasks) rulles ut fra desember 2026. Microsoft publiserte endringen i november 2026 i en støtteartikkel med tidsplan, forskjeller og informasjon for administratorer.

Kort oppsummert: Det finnes ingen automatisk migrering. Eksisterende To Do-oppgaver forblir synlige i Planner frem til 30. juni 2027 og lagres fortsatt i Exchange Online-postboksen. Fra utrullingen lagrer Planner nye private oppgaver i en privat Planner-plan, altså ikke lenger i Exchange. Dermed endres datalagringssted og Purview-dekning for sveitsiske leiere. Gå gjennom lisenser, delte lister, integrasjoner og oppbevaringsforpliktelser innen midten av 2027. Private Microsoft-kontoer får ingen erstatning.

## Tidsplan

| Tidspunkt | Endring |
|---|---|
| Desember 2026 | Start på utrullingen av den reviderte visningen «Mine oppgaver» i Planner for jobb- og skolekontoer. Fra da av lagrer Planner nyopprettede private oppgaver i en privat Planner-plan. |
| før 30. juni 2027 | Microsoft tilbyr en eksportfunksjon for To Do-data. Format, omfang og forutsetninger er ennå ikke publisert. |
| 30. juni 2027 | To Do-appene for nett, Windows, Mac, iOS og Android avvikles. Eksisterende To Do-oppgaver er deretter heller ikke synlige i Planner. |
| senere, uten dato | To Do-tjenesten, API-ene og synkroniseringen avvikles separat. Microsoft oppgir datoene særskilt. |

Planner er allerede tilgjengelig for bedriftskontoer på nettet via `planner.cloud.microsoft` og som app i Teams. Microsoft varsler at de eksisterende inngangspunktene til To Do i Microsoft 365 vil bli tilpasset. Microsoft har overfor Thurrott presisert at de eksisterende integrasjonene, inkludert Outlook og Teams, blir lagt om til Planner.

## Eksisterende oppgaver migreres ikke

Microsoft overfører ikke eksisterende To Do-oppgaver og -lister til Planner. Frem til 30. juni 2027 kan de fortsatt vises og redigeres i Planner, men ikke etterpå. De overføres ikke til den nye private planen.

Dataene slettes ikke når appene avvikles. Ifølge Microsoft innebærer ikke avviklingen av appene noen sletting; Microsoft behandler oppbevaring og tilgang separat. To Do lagrer oppgaver som Exchange-oppgaver i brukerens Exchange Online-postboks, i samme postboks som e-post, avtaler, kontakter og notater. Beholdningen blir derfor foreløpig værende i postboksen etter 30. juni 2027.

Klassisk Outlook påvirkes ikke av avviklingen. Microsoft skriver at oppgavefunksjonen der er separat fra To Do, fortsatt fungerer og synkroniseres med Exchange. To Do-lister vises i klassisk Outlook som oppgavemapper under «Mine oppgaver»; brukere med klassisk Outlook kan derfor fortsatt lese beholdningen der. Hva som skjer med dataene når Microsoft senere slår av selve To Do-tjenesten, er ennå ikke beskrevet.

## Nye oppgaver lagres utenfor Exchange

Med den nye visningen endres lagringsstedet. Ifølge FAQ-en for Planner-administratorer ligger oppgaver fra To Do og Outlook i Exchange, mens planer og oppgavene deres ligger i Azure. Planner lagrer vedlegg til planoppgaver på gruppens SharePoint-nettsted, og Premium-planer i Dataverse. Nye private oppgaver fra «Mine oppgaver» ligger dermed ikke lenger i brukerens postboks. Dette får to konsekvenser for administratorer.

### Datalagringssted

Exchange Online er blant tjenestene med en varig garanti for datalagringssted. Hvis en leier er opprettet i Sveits, lagrer Microsoft ifølge Product Terms postboksinnhold (e-posttekst, kalenderoppføringer og vedlegg) i hvile kun i Sveits. Oppgaver nevnes ikke uttrykkelig i garantien, men de ligger i samme postboks.

Microsoft fører Planner opp blant de øvrige Microsoft 365-tjenestene uten varig garanti. For leiere med land Sveits angir tabellen «EUR1», makroregionen Europa, som lagringssted. Planner deltar heller ikke i Advanced Data Residency. Personlige oppgaver som To Do hittil har lagret i postboksen i Sveits, lagrer Planner etter overgangen i makroregionen Europa. Lagringsstedene per tjeneste for leieren din er oppført på Data Location Card i Microsoft 365 Admin Center. Organisasjoner som har garantert datalagringssted i Sveits kontraktsmessig eller i interne retningslinjer, må vurdere denne endringen.

### Oppbevaring og eDiscovery

Exchange-oppbevaringspolicyer i Microsoft Purview omfatter oppgaver i postboksen bare dersom de har en sluttdato; oppgaver uten sluttdato er ikke dekket. Denne begrensningen gjelder allerede i dag for To Do-oppgaver.

For Planner støtter Purview eDiscovery-søk og Legal Hold ifølge Microsoft bare for oppgaver i planer som er delt med en gruppe. Personlige planer (Roster-planer), planer i Loop-komponenter, Premium-planer samt planer i delte og private Teams-kanaler støttes uttrykkelig ikke. Microsoft har rullet ut oppbevarings- og slettingspolicyer for Planner-innhold siden midten av juli 2026, med planlagt ferdigstillelse tidlig i august (MC1384412); ifølge meldingen kan de brukes på bruker- og gruppebaserte planer. Kunngjøringen opplyser ikke hvilken plantype den nye private planen teknisk tilsvarer. Kontroller etter utrullingen om oppbevaringspolicyene og holdene dine omfatter de nye private oppgavene.

## Forskjeller mellom To Do og Planner

Microsoft lister opp følgende forskjeller:

| Område | Forskjell ifølge Microsoft |
|---|---|
| Organisering | Planner samler private oppgaver og tildelt arbeid under «Mine oppgaver», i dag med rutenett- og tavlevisning. En listevisning kommer med revisjonen. |
| Påminnelser | Varslingene om forfallsdato i Planner skiller seg fra påminnelsene som planlegges per oppgave i To Do. |
| Deling | Planner bruker planer og tildelinger. Medlemskap, tildelinger og tillatelser for delte To Do-lister overføres ikke. |
| Skrivebord og frakoblet bruk | Planner finnes på nettet og i Teams. Det er ikke planlagt egne apper for Windows og Mac; nett og Teams har ingen tilsvarende frakoblet funksjon. |
| Kategorier | Planner-etiketter fungerer annerledes enn Outlook-kategorier og synkroniseres ikke med dem. |
| Vedlegg | Filvedlegg til oppgaver støttes for øyeblikket ikke i den nye private Planner-planen. |
| Stjerne og prioritet | Den personlige stjernen brukes til egen planlegging og kommer med den nye visningen. Prioriteten til en oppgave gjelder for alle deltakere i en delt plan. |
| Listegrupper | Ingen direkte ekvivalent i Planner. |
| Utskrift, sende liste per e-post | Ikke planlagt for erstatningsløsningen for øyeblikket. |

Brukere som arbeider frakoblet med To Do-appen i Windows eller Mac, har ingen tilsvarende erstatning i Planner.

## Lisenser og tilgang til Planner

Planner er blant annet inkludert i Microsoft 365 Business Basic, Business Standard og Business Premium samt i Office 365 E1, E3 og E5. Hvis Planner mangler i en lisens, tilbyr Microsoft Planner Plan 1 eller Planner and Project Plan 3 som egne abonnementer; disse kreves også for Premium-funksjonene. Ifølge Microsoft er ingen ny konfigurasjon nødvendig, og det finnes ingen egen bryter som administratorer kan bruke til å styre eller utsette overgangen.

Hvis Planner er deaktivert i lisenstildelingen (tjenesteplan `PROJECTWORKMANAGEMENT`), må de berørte brukerne få tilgang igjen dersom Planner skal erstatte To Do. Ifølge Planner-FAQ-en skjuler fjerning av Planner-lisensen uansett bare flisen: Via `planner.cloud.microsoft` kan brukere uten lisens fortsatt opprette og redigere planer. Hvis du hittil har sperret Planner via lisensen, bør du gjennomgå denne beslutningen med tanke på overgangen.

## Kartlegg beholdningen med Microsoft Graph

Hvilke brukere som bruker To Do, og hvilke lister som er delt, kan hentes ut via To Do-API-et i Microsoft Graph. Skriptet nedenfor skriver alle lister per bruker med delingsstatus og antall åpne oppgaver til en CSV-fil. Det krever modulene `Microsoft.Graph.Authentication` og `Microsoft.Graph.Users`, samt en appregistrering med programtillatelsene `Tasks.Read.All` og `User.Read.All` (med administratorgodkjenning) og et sertifikat for pålogging. `Tasks.Read.All` gir lesetilgang til oppgavene til alle brukere; fjern tillatelsen eller appregistreringen igjen etter kartleggingen.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # Ingen Exchange Online-postboks eller ingen tilgang: hopp over brukeren
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `ClientId`, `TenantId` | Appregistreringen og leieren som skriptet logger på mot |
| `CertificateThumbprint` | Fingeravtrykket til sertifikatet i sertifikatlageret til gjeldende bruker; pålogging som app uten brukerkonto |
| `-NoWelcome` | Undertrykker velkomstmeldingen fra `Connect-MgGraph` |
| `-All` | Leser alle sider i et resultat i stedet for bare den første |
| `-Filter "accountEnabled eq true"` | Bare aktiverte kontoer; deaktiverte kontoer som delte postbokser utelates |
| `-Property Id, UserPrincipalName` | Laster bare de to nødvendige egenskapene |
| `-UserId` | Brukeren som lister eller oppgaver hentes for |
| `-ErrorAction Stop` | Gjør en feil fangbar, slik at `catch` kan hoppe over brukeren |
| `-TodoTaskListId` | Listen det hentes oppgaver for |
| `WellknownListName` | `defaultList` for standardlisten «Oppgaver», `flaggedEmails` for flaggede meldinger, `none` for egne lister |
| `IsShared` | `True`, når listen er delt med andre brukere |
| `IsOwner` | `False`, når listen eies av en annen bruker og bare er delt |
| `Status -ne 'completed'` | Teller alle oppgaver som ikke er fullført |
| `-NoTypeInformation` | Ingen typeinje i starten av CSV-filen |
| `-Encoding UTF8` | Omlyder i listenavn bevares |

</details>

Kontoer uten Exchange Online-postboks gir en feil og hoppes over. I store leiere er det fornuftig med en testkjøring med et utvalg, for eksempel med `-Top 20` i stedet for `-All` ved `Get-MgUser`.

Evaluering av CSV-filen:

- **Delte lister** (`Geteilt` = `True`) viser hvor samarbeid skjer via To Do. Microsoft overfører ikke disse medlemskapene; listene må bygges opp på nytt som en Planner-plan med de aktuelle personene.
- **Åpne oppgaver** i egne lister (`Typ` = `none` eller `defaultList`) må brukerne overføre manuelt til Planner innen 30. juni 2027 eller sikre med den varslede eksportfunksjonen.
- **Flaggede meldinger** (`Typ` = `flaggedEmails`) beholdes som flagg på e-posten i postboksen.

## Private Microsoft-kontoer

To Do-appene avvikles også for private Microsoft-kontoer, inkludert Microsoft 365 Personal og Family, 30. juni 2027. Planner er ikke tilgjengelig for disse kontoene. Microsoft anbefaler å velge en annen oppgavebehandling og sikre dataene med den varslede eksportfunksjonen. Ifølge Microsoft betyr ikke en eksport at dataene kan importeres til Planner eller et annet program. Brukere med både jobb- og privatkonto må kontrollere begge kontoene separat.

## Forberedelser for administratorer

1. **Tilgang og lisenser:** Sørg for at alle To Do-brukere har en lisens med Planner, og at tjenesteplanen `PROJECTWORKMANAGEMENT` er aktivert.

2. **Delte lister og integrasjoner:** Kartlegg beholdningen med skriptet ovenfor. Identifiser Power Automate-flyter og tredjepartsprogrammer som bruker To Do-API-et i Microsoft Graph. Microsoft avvikler API-et separat og senere; ifølge Microsoft kan integrasjoner likevel kreve tilpasninger.

3. **Oppbevaring og datalagringssted:** Kontroller om eksisterende oppbevaringsforpliktelser gjelder for To Do-oppgaver, og om Purview-policyer og hold omfatter den nye private planen. Vurder om lagringsstedet i makroregionen Europa er forenlig med kravene dine. Involver de compliance-ansvarlige før du sletter eller reorganiserer oppgavedata.

4. **Informer brukerne:** Opplys brukerne om at ingen migrering finner sted, hvor lenge beholdningen forblir synlig i Planner, og hvilke funksjoner som endres (frakoblet bruk, påminnelser, vedlegg, listegrupper).

5. **Følg Message Center:** Microsoft varsler detaljer om utrullingen, eksportfunksjonen og datoene for tjenesten og API-ene der.

## Kilder

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): Kunngjøring med tidsplan, forskjeller mellom To Do og Planner, forberedelsestrinn og FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): Lagring av To Do-oppgaver som oppgaver i Exchange Online-postboksen.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): Lagringssteder for oppgaver (Exchange, Azure, SharePoint, Dataverse), tjenesteplan `PROJECTWORKMANAGEMENT` og tilgang via `planner.cloud.microsoft` uten lisens.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): Abonnementer med Planner samt Planner Plan 1 og Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner uten varig garanti for datalagringssted, lagringssted «EUR1» for leiere med land Sveits.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): Garantien i Product Terms for postboksinnhold i hvile i Sveits.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): Lagring av plandata i Azure; Planner deltar ikke i Advanced Data Residency. Status august 2025, før innføringen av oppbevaringspolicyer for Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery og Legal Hold bare for oppgaver i gruppebaserte planer, liste over ikke-støttede plantyper.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): Speil av Message Center-meldingen om oppbevarings- og slettingspolicyer for Planner, utrulling fra juli til august 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): Støttede postbokselementer; oppgaver bare med sluttdato.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): Ressurser i To Do-API-et i Microsoft Graph (lister, oppgaver, sjekklister, tilknyttede ressurser).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): Endepunkt, tillatelser og egenskapene `isShared`, `isOwner` og `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): Endepunkt for oppgaver i en liste og programtillatelsen `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): Rapport med ettersendt uttalelse fra Microsoft om integrasjonene i Outlook og Teams.
