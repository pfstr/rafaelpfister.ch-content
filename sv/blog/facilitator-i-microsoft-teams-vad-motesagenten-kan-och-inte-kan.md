---
title: "Facilitator i Microsoft Teams: Vad mötesagenten kan och inte kan"
navTitle: "Teams Facilitator"
description: "Facilitator är en Copilot-agent som skriver gemensamma anteckningar i schemalagda Teams-möten, hanterar agendan, övervakar tiden och besvarar frågor synligt för alla. Artikeln förklarar funktioner, licens- och administratörskrav, lämpliga användningsscenarier och begränsningar för mötestyper, språk och regelefterlevnad."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min läsning"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-i-microsoft-teams-vad-motesagenten-kan-och-inte-kan"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/sv/blog/facilitator-i-microsoft-teams-vad-motesagenten-kan-och-inte-kan
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:41:39.752Z
translationReview: automatic
---

Sedan Microsoft har rullat ut Facilitator i Teams-möten har en ny post dykt upp i mötesalternativen i många klientorganisationer: «Allow Copilot and Facilitator». Namnet säger inte mycket om vad som döljer sig bakom. Facilitator är en AI-agent som deltar i mötet och arbetar synligt för alla. Därmed blir både användningsmöjligheterna och begränsningarna tydliga.

## Vad Facilitator är

Facilitator är en agent från Microsoft 365 Copilot som tar rollen som mötesmoderator i ett schemalagt Teams-möte. Under mötet skriver den anteckningar på en gemensam Loop-sida som alla interna deltagare kan läsa och redigera. Den läser agendan från inbjudan, visar en agendatimer på mötesscenen och reagerar på @omnämnanden i möteschatten.

Skillnaden jämfört med de andra AI-funktionerna i Teams ligger i synligheten:

| Funktion | Vem ser resultatet | Tidpunkt |
|---|---|---|
| Copilot i Teams-möten | endast personen som frågar | under och efter mötet |
| Intelligent sammanfattning | deltagare med Teams Premium- eller Copilot-licens | efter mötet |
| Facilitator | alla interna deltagare | under mötet, även anteckningarna efteråt |

En fråga till Copilot förblir privat. En fråga till Facilitator och dess svar visas i möteschatten för alla. Facilitator arbetar alltså gemensamt för alla deltagare.

## Översikt för användare

För alla funktioner gäller följande: Mötet är ett schemalagt Teams-möte i din egen organisation utan externa deltagare, «Tillåt Copilot och Facilitator» är aktiverat i mötesalternativen och det talade språket har valts vid starten. Endast den som startar Facilitator eller ger den uppgifter behöver en Copilot-licens. Alla andra interna deltagare ser resultaten utan egen Copilot-licens.

| Funktion | Vad du kan göra med den | Vem kan använda den | Ytterligare krav |
|---|---|---|---|
| Starta Facilitator | Aktivera agenten före eller under mötet | Personer med Microsoft 365 Copilot-licens | Loop i Teams aktiverat, appen «Facilitator» inte blockerad |
| Liveanteckningar | Beslut, öppna frågor och uppgifter registreras löpande på fliken «Notes»; du kan korrigera och komplettera dem direkt | Läsa och redigera: alla interna deltagare | Transkribering körs (startar automatiskt); endast det som sägs på valt språk registreras, inte chattmeddelanden |
| Agenda och avprickning | Ärendelistan från inbjudan visas i chatten, avhandlade punkter bockas av | Se: alla interna deltagare | Agenda finns i inbjudan eller lämnas via @Facilitator i chatten; inbjudan är inte krypterad; minst en ytterligare person utöver organisatören deltar i mötet |
| Timer och påminnelser | Nedräkning för en punkt, påminnelse halvvägs, före mötesslut och vid överskriden tid | Begära timer: Copilot-licens; se: alla interna deltagare | inga |
| Frågor om mötet | Fråga via @Facilitator i chatten, till exempel «Vad beslutade vi om punkt 2?»; frågan och svaret syns för alla | Ställa frågor: Copilot-licens; läsa svar: alla interna deltagare | Frågor efter mötets slut endast med alternativet «Under och efter mötet» |
| Frågor med webbsökning | Kort research under mötet, till exempel om en produktversion | Copilot-licens | Webbsökning för Copilot är inte inaktiverad i klientorganisationen |
| Skapa Word-dokument | Skapa ett dokument av anteckningarna, till exempel ett utkast till protokoll | Copilot-licens | Sparas i den frågande personens OneDrive |
| Skapa PowerPoint-presentation | Skapa en presentation från mötet | Copilot-licens | Public Preview: deltagande i Teams Public Preview |
| Överföra uppgifter till Planner | Överför registrerade uppgifter till Planner med «Accept to sync» | Copilot-licens | Public Preview: deltagande i Teams Public Preview |
| Anteckningar vid fysiskt möte (smartphone) | Dokumentera ett samtal på plats med Teams-appen på smarttelefonen | Copilot-licens | Teams-app på smarttelefonen |
| Anteckningar i Teams Room | Visa chatt, anteckningar och timer på rumsskärmen | Copilot-licens endast för organisatören | Teams Rooms on Windows; för oplanerade fysiska möten Teams Rooms Pro (Public Preview) |

## Funktioner och status

Microsoft redovisar funktionerna separat i administratörsdokumentationen efter allmänt tillgängliga och Public Preview (status november 2026):

| Funktion | Status |
|---|---|
| Liveanteckningar på fliken «Notes» (Loop-sida, gemensamt redigerbar) | allmänt tillgänglig |
| Svara på frågor utifrån mötesinnehållet och via webbsökning | allmänt tillgänglig |
| Agendatimer med påminnelser halvvägs och före mötesslut | allmänt tillgänglig |
| Skapa Word-dokument från mötet | allmänt tillgänglig |
| @omnämnande av Facilitator i möteschatten | allmänt tillgänglig |
| Anteckningar på smarttelefonen för fysiska möten | allmänt tillgänglig |
| Teams Rooms: chatt, anteckningar och timer i rummet | allmänt tillgänglig |
| Registrera uppgifter och synkronisera med Planner | Public Preview |
| Skapa PowerPoint-presentation från mötet | Public Preview |
| Anteckningar för oplanerade fysiska möten i Teams Rooms | Public Preview |

Om agendan: Om inbjudan innehåller en ärendelista tar Facilitator över den, publicerar den i chatten och bockar av punkter när de har avhandlats. Avprickningen sker med intervall, inte omedelbart. Om mötet går över den planerade sluttiden påpekar den detta. På begäran kan den även sätta en nedräkning för en enskild punkt.

Om anteckningarna: Facilitator sammanfattar beslut, öppna frågor och uppgifter. Underlaget är enbart det talade ordet via transkriberingen. Det som endast står i möteschatten tas inte med i anteckningarna. Anteckningarna kan när som helst korrigeras och kompletteras manuellt; Facilitator fortsätter att skriva på samma sida.

## Förutsättningar

För att tjänsten ska fungera måste flera villkor uppfyllas samtidigt:

- **Licens:** Den som lägger till Facilitator i ett möte eller aktiverar den under mötet behöver en Microsoft 365 Copilot-licens utöver en lämplig Microsoft 365- och Teams-licens. Deltagare utan Copilot-licens kan inte ställa frågor till Facilitator, men ser anteckningarna, de andras frågor och dess svar.
- **Loop:** Loop-upplevelserna i Teams måste vara aktiverade i klientorganisationen. Utan Loop finns inga anteckningar.
- **Transkribering:** När Facilitator aktiveras startar transkriberingen automatiskt. Om mötesalternativet «Allow Copilot and Facilitator» är inställt på «Only during the meeting» körs Facilitator utan sparad transkribering; frågor efter mötets slut är då inte möjliga.
- **Appbehörighet:** Appen «Facilitator» är tillåten som standard, förutsatt att inte alla appar blockeras i Teams Admin Center.

I inbjudan aktiverar du Facilitator under **Options** > **Copilot and other AI** med inställningen «Allow Copilot and Facilitator». Alternativt kan den aktiveras under mötet via Copilot-menyn.

## Facilitator är nedtonad

I mötet visar Teams tre reglage i menyn «Mötes-AI»: Copilot, Facilitator och Intelligent sammanfattning. Om alla tre är nedtonade och meddelandet «Du har inte åtkomst till AI i det här mötet» visas, beror orsaken på licensen eller själva mötet. Om endast Facilitator är nedtonad beror det på appen eller Loop.

| Symtom | Möjlig orsak | Kontroll |
|---|---|---|
| Alla tre reglage är grå | Kontot har varken Microsoft 365 Copilot- eller Teams Premium-licens | Microsoft 365 Admin Center > Användare > Licenser. En nyligen tilldelad licens träder ofta i kraft i Teams först efter några timmar eller efter en ny inloggning |
| Alla tre reglage är grå | Direktmöte («Meet now»), möte startat från chatten, kanalmöte eller samtal | Skapa mötet som ett schemalagt Teams-möte via kalendern |
| Alla tre reglage är grå | Möte i en annan organisation, deltagande som extern person eller gäst | Facilitator är inte tillgänglig i externa möten |
| Alla tre reglage är grå | Mötesalternativet «Tillåt Copilot och Facilitator» är inställt på «Av», angett av organisatören, en mötesmall eller en känslighetsetikett | Mötesalternativ > Copilot och annan AI |
| Alla tre reglage är grå | Teams mötesprincip blockerar Copilot eller transkribering | Teams Admin Center > Möten > Mötesprinciper |
| Endast Facilitator är grå | Appen «Facilitator» är blockerad, till exempel eftersom alla appar är spärrade | Teams Admin Center > Teams-appar > Hantera appar |
| Endast Facilitator är grå eller inga anteckningar | Loop för mötesanteckningar är inaktiverat | `Get-SPOTenant` visar `IsCollabMeetingNotesFluidEnabled` |

Licensen är den vanligaste orsaken. Utan Microsoft 365 Copilot-licens förblir Facilitator-reglaget grått i varje möte, oavsett mötesalternativ och principer.

## Vad Facilitator passar för

Facilitator ger störst nytta i möten där någon annars för protokoll och därför inte kan delta i diskussionen. Typiska exempel:

- **Projekt- och statusmöten med fast ärendelista:** Agendatimer och påminnelser halvvägs håller tidsplanen, och anteckningarna innehåller beslut och öppna punkter utan separat protokoll.
- **Workshoppar med beslut:** Den gemensamma Loop-sidan är synlig under mötet. Deltagarna korrigerar formuleringar direkt i stället för att efteråt stämma av ett protokoll via e-post.
- **Frågor under mötet:** «Vad beslutade vi om punkt 2?» eller en kort webbsökning om en produktversion besvarar Facilitator i chatten, synligt för alla.
- **Senare sammanfattning:** På begäran skapar Facilitator ett Word-dokument av anteckningarna som lagras i den frågande personens OneDrive.
- **Fysiska möten:** Med Teams-appen på smarttelefonen eller i ett Teams Room dokumenteras även möten på plats.

## Vad Facilitator inte kan

Begränsningarna beskrivs tydligt i Microsofts dokumentation och gäller främst mötestyper, språk och regelefterlevnad.

**Mötestyper som inte stöds.** Facilitator fungerar endast i schemalagda Teams-möten. Direktmöten («Meet now»), möten startade från en chatt, Teams-samtal, kanalmöten samt 1:1- och gruppchattar stöds inte. Dokumentationen nämner inget stöd för webbseminarier och Town Halls.

**Inga externa deltagare.** AI-genererade anteckningar är inte tillgängliga i möten och chattar med externa personer. Externa deltagare kan i princip inte se anteckningarna. Facilitator kan därför inte användas som protokollsverktyg för möten med kunder eller leverantörer.

**Ett språk per möte.** Det talade språket väljs vid aktiveringen. Anteckningar skapas endast för avsnitt på detta språk. I ett möte som växlar mellan tyska och franska saknas de franska delarna. Om det valda språket skiljer sig från det talade skapas inga anteckningar. Schweizertyska anges inte som ett eget språk i listan över Copilot-språk som stöds.

**Ingen inspelning.** Facilitator aktiverar transkriberingen, men inte inspelning. Den som behöver bild och ljud måste starta inspelningen separat.

**Ingen agenda från skyddade inbjudningar.** Om inbjudan är krypterad med en känslighetsetikett kan Facilitator inte läsa agendan. Detsamma gäller så länge endast organisatören finns i mötet, eftersom den inte läser organisatörens kopia av inbjudan.

**Ingen ersättning för ett formellt protokoll.** Anteckningarna är en AI-sammanfattning av transkriberingen. För styrelsemöten, personalsamtal eller andra möten med rättslig betydelse behövs fortfarande ett granskat och godkänt protokoll.

## Datalagring och regelefterlevnad

Anteckningarna lagras som en `.loop`-fil i mappen «Meetings» i OneDrive för personen som startade Facilitator. Microsoft behandlar dem som transkriberingsdata. För compliance-ansvariga är tre begränsningar relevanta:

- **eDiscovery:** AI-genererade mötesanteckningar registreras inte automatiskt som molnbilagor i Microsoft Purview eDiscovery.
- **Bevarande:** Bevarandeetiketter stöds inte för molnbilagor i AI-genererade anteckningar.
- **Känslighetsetiketter:** Anteckningarna ärver inte mötets etikett. En etikett kan läggas till i efterhand på Loop-komponenten. Inställningar som «Prevent copy and paste» och vattenstämplar gäller inte för Facilitators svar och anteckningar.

För organisationer med bevarandekrav innebär detta att anteckningar som ska räknas som affärshandlingar måste överföras till en lagringsplats med fungerande bevarande. Microsoft beskriver Purview-aspekterna på en egen sida (se Källor).

## Styra och inaktivera Facilitator

I Teams Admin Center hittar du appen under **Teams apps** > **Manage apps** genom att söka efter «Facilitator». Där kan den tillåtas eller blockeras för hela klientorganisationen. För enskilda användargrupper styr du appen via principer för appbehörighet eller appcentrerad hantering.

Loop-anteckningarna i möten, som Facilitator bygger på, är knutna till en SharePoint-inställning på klientorganisationsnivå. Den gäller för alla användare och kan inte ställas in per person:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Förklaring av alternativen</summary>

| Alternativ | Effekt |
|---|---|
| `-Url` | Adressen till klientorganisationens SharePoint Admin Center (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` inaktiverar de gemensamma mötesanteckningarna och därmed Facilitators AI-anteckningar för alla användare, `$true` aktiverar dem igen |

</details>

Denna inställning tar även bort anteckningsfunktionen för möten utan Facilitator. Om endast agenten ska blockeras är det mer träffsäkert att blockera appen i Teams Admin Center. För möten utan sparad transkribering anger du alternativet för Copilot och Facilitator till «Only during the meeting» i mötesprincipen.

## Källor

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Administratörsdokumentation med licenskrav, funktionsstatus, datalagring, compliance-begränsningar och appstyrning (status 18.11.2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Användarguide om mötesalternativ, agenda, timer, mötestyper som stöds och språkval.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Fysiska möten i Teams Rooms on Windows, organisatörens licens, koppling av talare i rummet.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Microsoft 365 Insider-blogg från oktober 2025 om transkribering, underlag för anteckningar och drift utan sparad transkribering.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Purview-perspektiv på Facilitator-data för bevarande och eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Klientorganisationsinställningarna `IsLoopEnabled` och `IsCollabMeetingNotesFluidEnabled` för Loop-komponenter och mötesanteckningar.
