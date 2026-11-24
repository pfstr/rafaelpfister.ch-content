---
title: "Microsoft To Do avvecklas: migrering till Planner ur ett administratörsperspektiv"
navTitle: "To Do avvecklas"
description: "Microsoft avvecklar To Do-apparna den 30 juni 2027. Företagskonton går över till Planner, befintliga uppgifter migreras inte och Planner lagrar nya privata uppgifter utanför Exchange-postlådan. För schweiziska klientorganisationer ändras därmed dataplats och compliance-täckning; berörda användare kan kartläggas med Microsoft Graph."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min läsning"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-avvecklas-migrering-till-planner-ur-ett-administratorsperspektiv"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/sv/blog/microsoft-to-do-avvecklas-migrering-till-planner-ur-ett-administratorsperspektiv
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:00:23.098Z
translationReview: automatic
---

# Microsoft To Do avvecklas: migrering till Planner ur ett administratörsperspektiv

Microsoft avvecklar Microsoft To Do. Appar för webben, Windows, Mac, iOS och Android stängs av den 30 juni 2027, både för företags- och skolkonton samt privata Microsoft-konton. För företags- och skolkonton tar Microsoft Planner över de personliga uppgifterna; den omarbetade vyn ”Mina uppgifter” (My tasks) rullas ut från december 2026. Microsoft publicerade ändringen i november 2026 i en supportartikel med tidsplan, skillnader och information för administratörer.

Kort sammanfattning: Det finns ingen automatisk migrering. Befintliga To Do-uppgifter förblir synliga i Planner till den 30 juni 2027 och lagras fortsatt i Exchange Online-postlådan. Från utrullningen lagrar Planner nya privata uppgifter i en privat Planner-plan, alltså inte längre i Exchange. Därmed ändras dataplats och Purview-täckning för schweiziska klientorganisationer. Kontrollera licenser, delade listor, integrationer och bevarandekrav före mitten av 2027. Privata Microsoft-konton får ingen ersättning.

## Tidsplan

| Tidpunkt | Ändring |
|---|---|
| December 2026 | Utrullningen av den omarbetade vyn ”Mina uppgifter” i Planner för företags- och skolkonton inleds. Planner lagrar från då nya privata uppgifter i en privat Planner-plan. |
| före den 30 juni 2027 | Microsoft tillhandahåller en exportfunktion för To Do-data. Format, omfattning och förutsättningar har ännu inte publicerats. |
| 30 juni 2027 | To Do-apparna för webben, Windows, Mac, iOS och Android avvecklas. Befintliga To Do-uppgifter är därefter inte längre synliga ens i Planner. |
| senare, ännu utan datum | To Do-tjänsten, API:erna och synkroniseringen avvecklas separat. Microsoft meddelar datumen separat. |

Planner är redan i dag tillgängligt för företagskonton på webben via `planner.cloud.microsoft` och som app i Teams. Microsoft meddelar att de befintliga ingångarna till To Do i Microsoft 365 anpassas. Microsoft har för Thurrott förtydligat att de befintliga integrationerna, däribland Outlook och Teams, övergår till Planner.

## Befintliga uppgifter migreras inte

Microsoft överför inte befintliga To Do-uppgifter och -listor till Planner. De kan fortfarande visas och redigeras i Planner till den 30 juni 2027, men inte därefter. De tas inte med till den nya privata planen.

Data raderas inte när apparna upphör. Enligt Microsoft omfattar avvecklingen av apparna ingen radering; Microsoft hanterar lagring och åtkomst separat. To Do lagrar uppgifter som Exchange-uppgifter i användarens Exchange Online-postlåda, i samma postlåda som e-post, kalenderposter, kontakter och anteckningar. Beståndet finns därför tills vidare kvar i postlådan efter den 30 juni 2027.

Klassiska Outlook påverkas inte av avvecklingen. Microsoft skriver att dess uppgiftsfunktion är separat från To Do, fortsätter att fungera och synkroniseras med Exchange. To Do-listor visas i klassiska Outlook som uppgiftsmappar under ”Mina uppgifter”; användare med klassiska Outlook kan alltså fortsatt läsa beståndet där. Vad som händer med data när Microsoft senare stänger själva To Do-tjänsten har ännu inte beskrivits.

## Nya uppgifter lagras utanför Exchange

Med den nya vyn ändras lagringsplatsen. Enligt vanliga frågor för Planner-administratörer finns uppgifter från To Do och Outlook i Exchange, medan planer och deras uppgifter finns i Azure. Planner lagrar bilagor till planuppgifter på gruppens SharePoint-webbplats och premiumplaner i Dataverse. Nya privata uppgifter från ”Mina uppgifter” finns därmed inte längre i användarens postlåda. Detta får två konsekvenser för administratörer.

### Dataplats

Exchange Online hör till tjänsterna med permanent åtagande om dataplats. Om en klientorganisation har skapats i Schweiz lagrar Microsoft enligt Product Terms postlådeinnehåll (e-posttext, kalenderposter och bilagor) i vila endast i Schweiz. Uppgifter nämns inte uttryckligen i åtagandet, men de finns i samma postlåda.

Microsoft listar Planner bland övriga Microsoft 365-tjänster utan permanent åtagande. För klientorganisationer med Schweiz som land anger tabellen ”EUR1”, makroregionen Europa, som lagringsplats. Planner deltar dessutom inte i Advanced Data Residency. Personliga uppgifter som To Do tidigare lagrade i postlådan i Schweiz lagrar Planner efter bytet i makroregionen Europa. Lagringsplatserna per tjänst för din klientorganisation anges på Data Location Card i Microsoft 365 Admin Center. Organisationer som har garanterat dataplats i Schweiz avtalsmässigt eller i interna riktlinjer måste bedöma denna ändring.

### Kvarhållning och eDiscovery

Exchange-kvarhållningsprinciper i Microsoft Purview omfattar uppgifter i postlådan endast om de har ett slutdatum; uppgifter utan slutdatum omfattas inte. Denna begränsning gäller redan i dag för To Do-uppgifter.

För Planner stöder Purview enligt Microsoft eDiscovery-sökning och Legal Hold endast för uppgifter i planer som delas med en grupp. Personliga planer (Roster-planer), planer i Loop-komponenter, premiumplaner samt planer i delade och privata Teams-kanaler stöds uttryckligen inte. Microsoft har rullat ut principer för kvarhållning och radering av Planner-innehåll sedan mitten av juli 2026, med planerat slutförande i början av augusti (MC1384412); enligt meddelandet kan de tillämpas på användar- och gruppbaserade planer. Vilken plantyp den nya privata planen tekniskt motsvarar framgår inte av tillkännagivandet. Kontrollera efter utrullningen om dina kvarhållningsprinciper och holds omfattar de nya privata uppgifterna.

## Skillnader mellan To Do och Planner

Microsoft listar följande skillnader:

| Område | Skillnad enligt Microsoft |
|---|---|
| Organisation | Planner samlar privata uppgifter och tilldelat arbete under ”Mina uppgifter”, i dag med rutnäts- och tavlevy. En listvy kommer med omarbetningen. |
| Påminnelser | Aviseringarna om förfallodatum i Planner skiljer sig från de schemalagda påminnelserna per uppgift i To Do. |
| Delning | Planner arbetar med planer och tilldelningar. Medlemskap, tilldelningar och behörigheter för delade To Do-listor överförs inte. |
| Skrivbord och offline | Planner finns på webben och i Teams. Inga inbyggda appar för Windows och Mac är planerade; webben och Teams erbjuder ingen jämförbar offlinefunktion. |
| Kategorier | Planner-etiketter fungerar annorlunda än Outlook-kategorier och synkroniseras inte med dem. |
| Bilagor | Filbilagor till uppgifter stöds för närvarande inte i den nya privata Planner-planen. |
| Stjärna och prioritet | Den personliga stjärnan används för egen planering och kommer med den nya vyn. En uppgifts prioritet gäller för alla deltagare i en delad plan. |
| Listgrupper | Har ingen direkt motsvarighet i Planner. |
| Skriva ut, skicka lista via e-post | Är för närvarande inte planerat för ersättningslösningen. |

Användare som arbetar offline med To Do-appen i Windows eller Mac har ingen likvärdig ersättning i Planner.

## Licenser och åtkomst till Planner

Planner ingår bland annat i Microsoft 365 Business Basic, Business Standard och Business Premium samt i Office 365 E1, E3 och E5. Om Planner saknas i en licens erbjuder Microsoft Planner Plan 1 eller Planner and Project Plan 3 som separata prenumerationer; dessa krävs även för premiumfunktionerna. Enligt Microsoft krävs ingen ny konfiguration, och det finns ingen separat växel med vilken administratörer kan styra eller skjuta upp bytet.

Om Planner är inaktiverat i licenstilldelningen (tjänstplan `PROJECTWORKMANAGEMENT`) behöver berörda användare åter få åtkomst om Planner ska ersätta To Do. Enligt Planner-FAQ döljer borttagning av Planner-licensen ändå endast rutan: via `planner.cloud.microsoft` kan användare utan licens fortfarande skapa och redigera planer. Om du hittills har spärrat Planner via licensen bör du se över detta beslut med tanke på bytet.

## Kartlägg beståndet med Microsoft Graph

Vilka användare som använder To Do och vilka listor som delas kan läsas ut via To Do-API:t i Microsoft Graph. Följande skript skriver alla listor per användare, med delningsstatus och antal öppna uppgifter, till en CSV-fil. Det kräver modulerna `Microsoft.Graph.Authentication` och `Microsoft.Graph.Users` samt en appregistrering med programbehörigheterna `Tasks.Read.All` och `User.Read.All` (med administratörsgodkännande) och ett certifikat för inloggning. `Tasks.Read.All` ger läsåtkomst till alla användares uppgifter; ta bort behörigheten eller appregistreringen igen efter kartläggningen.

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
        # Ingen Exchange Online-postlåda eller ingen åtkomst: hoppa över användaren
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
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `ClientId`, `TenantId` | Appregistrering och klientorganisation som skriptet loggar in mot |
| `CertificateThumbprint` | Certifikatets tumavtryck i den aktuella användarens certifikatarkiv; inloggning som app utan användarkonto |
| `-NoWelcome` | Undertrycker välkomstmeddelandet från `Connect-MgGraph` |
| `-All` | Läser alla sidor i ett resultat i stället för endast den första |
| `-Filter "accountEnabled eq true"` | Endast aktiverade konton; inaktiverade konton såsom delade postlådor utelämnas |
| `-Property Id, UserPrincipalName` | Läser endast in de två egenskaper som behövs |
| `-UserId` | Användare vars listor eller uppgifter hämtas |
| `-ErrorAction Stop` | Gör ett fel fångbart så att `catch` kan hoppa över användaren |
| `-TodoTaskListId` | Lista vars uppgifter hämtas |
| `WellknownListName` | `defaultList` för standardlistan ”Uppgifter”, `flaggedEmails` för flaggade meddelanden, `none` för egna listor |
| `IsShared` | `True` om listan delas med andra användare |
| `IsOwner` | `False` om listan tillhör en annan användare och endast har delats |
| `Status -ne 'completed'` | Räknar alla ej slutförda uppgifter |
| `-NoTypeInformation` | Ingen typrad i början av CSV-filen |
| `-Encoding UTF8` | Umlauter i listnamn bevaras |

</details>

Konton utan Exchange Online-postlåda ger ett fel och hoppas över. I stora klientorganisationer är en testkörning med ett urval lämplig, till exempel med `-Top 20` i stället för `-All` vid `Get-MgUser`.

Utvärdering av CSV-filen:

- **Delade listor** (`Geteilt` = `True`) visar var samarbete sker via To Do. Microsoft överför inte dessa medlemskap; listorna måste byggas upp på nytt som Planner-planer med berörda personer.
- **Öppna uppgifter** i egna listor (`Typ` = `none` eller `defaultList`) måste användarna före den 30 juni 2027 manuellt föra över till Planner eller säkra med den aviserade exportfunktionen.
- **Flaggade meddelanden** (`Typ` = `flaggedEmails`) finns kvar som flaggning på e-postmeddelandet i postlådan.

## Privata Microsoft-konton

To Do-apparna avvecklas även för privata Microsoft-konton, inklusive Microsoft 365 Personal och Family, den 30 juni 2027. Planner är inte tillgängligt för dessa konton. Microsoft rekommenderar att välja en annan uppgiftshantering och säkra data med den aviserade exportfunktionen. Enligt Microsoft innebär en export inte att data kan importeras till Planner eller någon annan app. Användare med både företags- och privatkonto måste kontrollera båda kontona separat.

## Förberedelser för administratörer

1. **Åtkomst och licenser:** Säkerställ att alla To Do-användare har en licens med Planner och att tjänstplanen `PROJECTWORKMANAGEMENT` är aktiverad.

2. **Delade listor och integrationer:** Kartlägg beståndet med skriptet ovan. Identifiera Power Automate-flöden och tredjepartsapplikationer som använder To Do-API:t i Microsoft Graph. Microsoft avvecklar API:t separat och senare; enligt Microsoft kan integrationer ändå behöva anpassas.

3. **Kvarhållning och dataplats:** Kontrollera om befintliga bevarandekrav gäller för To Do-uppgifter och om Purview-principer och holds omfattar den nya privata planen. Bedöm om lagringsplatsen i makroregionen Europa är förenlig med era krav. Involvera de compliance-ansvariga innan uppgiftsdata raderas eller omorganiseras.

4. **Informera användare:** Informera användarna om att ingen migrering sker, hur länge beståndet förblir synligt i Planner och vilka funktioner som ändras (offlineanvändning, påminnelser, bilagor, listgrupper).

5. **Följ Message Center:** Microsoft meddelar där detaljer om utrullningen, exportfunktionen och tidpunkterna för tjänsten och API:erna.

## Källor

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): Tillkännagivande med tidsplan, skillnader mellan To Do och Planner, förberedelsesteg och FAQ.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): Lagring av To Do-uppgifter som uppgifter i Exchange Online-postlådan.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): Lagringsplatser för uppgifter (Exchange, Azure, SharePoint, Dataverse), tjänstplan `PROJECTWORKMANAGEMENT` och åtkomst via `planner.cloud.microsoft` utan licens.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): Prenumerationer med Planner samt Planner Plan 1 och Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner utan permanent åtagande om dataplats, lagringsplats ”EUR1” för klientorganisationer med Schweiz som land.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): Product Terms-åtagande för postlådeinnehåll i vila i Schweiz.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): Lagring av plandata i Azure; Planner deltar inte i Advanced Data Residency. Status augusti 2025, före införandet av kvarhållningsprinciper för Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery och Legal Hold endast för uppgifter i gruppbaserade planer, lista över plantyper som inte stöds.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): Spegel av Message Center-meddelandet om principer för kvarhållning och radering för Planner, utrullning juli till augusti 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): Postlådeobjekt som stöds; uppgifter endast med slutdatum.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): Resurser i To Do-API:t i Microsoft Graph (listor, uppgifter, checklistor, länkade resurser).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): Slutpunkt, behörigheter och egenskaperna `isShared`, `isOwner` och `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): Slutpunkt för uppgifter i en lista och programbehörighet `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): Rapport med Microsofts senare uttalande om integrationerna i Outlook och Teams.
