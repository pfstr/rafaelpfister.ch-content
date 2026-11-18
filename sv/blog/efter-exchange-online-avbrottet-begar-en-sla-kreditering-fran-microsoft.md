---
title: "Efter Exchange Online-avbrottet: begär en SLA-kreditering från Microsoft"
navTitle: "SLA-kreditering"
description: "Exchange Online-avbrottet den 12 och 13 november 2026 varade i cirka 22 timmar och innebär därför för många klientorganisationer en tillgänglighet under den utlovade nivån på 99,9 procent. Microsoft beviljar inte krediteringen automatiskt; den måste begäras senast den 31 december 2026. Så beräknar du tillgängligheten, vilka underlag Microsoft kräver och varför små företag bör gå via sin partner medan stora företag bör vända sig direkt till Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min läsning"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "efter-exchange-online-avbrottet-begar-en-sla-kreditering-fran-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
url: https://rafaelpfister.ch/sv/blog/efter-exchange-online-avbrottet-begar-en-sla-kreditering-fran-microsoft
translationSourceHash: 537caf78f05fd364041d0a3155f020be11b80f4c936d732e61f0a01f443853c1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T09:23:52.293Z
translationReview: required
---

# Efter Exchange Online-avbrottet: begär en SLA-kreditering från Microsoft

Från den 12 november 2026 klockan 18:54 till den 13 november 2026 klockan 17:10 (schweizisk tid) saknade många organisationer helt eller delvis åtkomst till sina Exchange Online-postlådor. Microsoft registrerade störningen i Admin Center under EX1550543 (Exchange Online) och MO1550821 (andra Microsoft 365-tjänster). Jag har dokumenterat förloppet i [liverapporteringen om avbrottet](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). För Exchange Online garanterar Microsoft en månatlig tillgänglighet på 99,9 procent. Ett avbrott på drygt 22 timmar ligger tydligt under detta. Du får dock bara en kreditering om du ansöker om den, och vägen dit beror på hur din organisation har köpt Microsoft 365.

## Vad SLA:t garanterar

Det avgörande dokumentet är ”Service Level Agreement for Microsoft Online Services”, som Microsoft publicerar på nytt varje månad. Den här artikeln bygger på versionen från den 17 november 2026. För Exchange Online definierar SLA:t en driftstörning som varje tidsperiod då användare inte kan skicka eller ta emot e-post med Outlook på webben. Det finns inga planerade underhållsfönster för Exchange Online.

Tillgängligheten beräknas i användarminuter:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Användarminuter är antalet minuter i månaden multiplicerat med antalet användare. Avbrottsminuter är varje incidentens varaktighet multiplicerad med antalet berörda användare. Om tillgängligheten understiger en tröskel ges en kreditering på månadsavgiften för den berörda tjänsten:

| Tillgänglighet under månaden | Kreditering |
|---|---|
| under 99,9 % | 25 % |
| under 99 % | 50 % |
| under 95 % | 100 % |

För Microsoft Teams, SharePoint Online och OneDrive gäller samma trösklar med egna definitioner av avbrottstid. För Teams räknas det bara om användare inte kan chatta eller starta möten. Under EX1550543 fungerade chattar och möten enligt Microsoft till största delen, medan kalendrar och inbjudningar stördes. En Teams-ansökan har därför sannolikt små utsikter.

Utöver postlådetillgänglighet har Exchange Online ytterligare service levels, till exempel leveranstiden inom Microsoft 365 och mottagandet av inkommande e-post. Om en incident berör flera av dem måste du enligt SLA:t välja ett. För novemberavbrottet är postlådetillgängligheten via Outlook på webben den mest närliggande måttstocken.

## Vad avbrottet innebär i procent

November 2026 har 30 dagar, alltså 43'200 minuter per användare. Detta ger följande gränsvärden per användare:

| Tröskel | Tillåten avbrottstid i november |
|---|---|
| 99,9 % | 43 minuter |
| 99 % | 7 timmar 12 minuter |
| 95 % | 36 timmar |

Störningen varade 1'336 minuter från den första rapporten till åtgärdandet. Två räkneexempel för en organisation med 200 postlådor:

| Fall | Avbrottsminuter | Tillgänglighet | Kreditering |
|---|---|---|---|
| Samtliga 200 användare utan åtkomst under hela störningen | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 användare utan åtkomst under 10 timmar | 120 x 600 = 72'000 | 99,17 % | 25 % |

Det totala antalet användarminuter är i båda fallen 200 x 43'200 = 8'640'000. Avgörande är hur många användare som faktiskt var berörda och under hur lång tid. Microsoft granskar detta utifrån sin egen telemetri, men kräver dina uppgifter i ansökan. Många klientorganisationer i Europa påverkades först sent på kvällen den 12 november eller endast periodvis. Beräkna därför med de tider som du kan styrka, inte med den totala varaktigheten i Microsofts meddelande.

## Vad krediteringen är och inte är

SLA:t reglerar några punkter som bör vara klara innan ansökan:

1.  **Endast efter ansökan.** Microsoft granskar och beviljar inte krediteringar på eget initiativ. En statusrapport från Microsoft utgör enligt SLA:t uttryckligen inte en bekräftelse på att en service level inte har uppnåtts.

2.  **Kreditering, ingen återbetalning.** Krediteringen kvittas mot en framtida faktura. Den är begränsad till månadsavgiften för den berörda tjänsten.

3.  **Exklusivt rättsmedel.** Enligt SLA:t är krediteringar det enda rättsmedlet vid tillgänglighetsproblem. Microsoft ersätter inte intäktsbortfall, övertid eller andra följdkostnader. Du får inte på eget initiativ minska fakturor.

4.  **Endast den berörda tjänsten.** Om du har en svit som Microsoft 365 E3 eller Business Premium avser krediteringen Exchange Online-andelen, inte hela licenspriset. Microsoft fastställer hur stor denna andel är vid granskningen; SLA:t anger ingen formel för detta.

5.  **Ingen kreditering för gratis- och testerbjudanden.** Testversioner, förhandsversioner och kostnadsfria tjänster är undantagna, liksom donerade licenser till ideella organisationer. Privatkunder med Microsoft 365 Personal eller Family omfattas inte av detta SLA.

6.  **Open-avtal i dagar.** Den som har licensierat via Open, Open Value eller Open Value Subscription får en eventuell kreditering som ytterligare användningsdagar i stället för ett penningbelopp.

7.  **Tidsfrist.** För Microsoft 365-tjänster måste ansökan med alla uppgifter ha inkommit senast i slutet av månaden efter incidentmånaden. För novemberavbrottet är det **31 december 2026**. Azure har en egen tidsfrist på 60 dagar.

## Vilka uppgifter Microsoft kräver

SLA:t anger fem obligatoriska uppgifter. Om de saknas avvisar Microsoft ansökan:

1.  en exakt beskrivning av incidenten,
2.  tidpunkt och varaktighet för avbrottet,
3.  de berörda resurserna (för Exchange Online: klientorganisation och tjänst),
4.  antal och platser för de berörda användarna,
5.  en beskrivning av de fel som uppstod.

Incidentens identifierare (EX1550543) och klientorganisations-ID:t ska alltid ingå. Båda kan hämtas med Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | behörighet att läsa meddelanden om tjänsthälsa |
| `-Scopes "Organization.Read.All"` | behörighet att läsa organisationsdata, inklusive klientorganisations-ID:t |
| `(Get-MgOrganization).Id` | visar klientorganisations-ID:t |
| `-ServiceHealthIssueId "EX1550543"` | hämtar exakt denna incident |
| `Select-Object …` | visar identifierare, titel, start, slut och incidentklassificering |

</details>

Klassificeringen (`Classification`) visar om Microsoft hanterar incidenten som en `incident` eller som en `advisory`. Meddelanden på nivån ”Advisory” berättigar enligt Microsofts partnerdokumentation vanligtvis inte till kreditering. EX1550543 var klassificerad som Incident.

För antal och plats för användarna räcker en sammanställning av de licensierade postlådorna efter användningsplats:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Alternativen förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `$abfrage = @{ … }` | samlar parametrarna för `Get-MgUser` i en hashtabell (splatting), så att raden hålls kort |
| `All` | läser alla användare, inte bara första sidan |
| `Filter 'assignedLicenses/$count ne 0'` | endast användare med minst en tilldelad licens |
| `ConsistencyLevel eventual` | krävs för filtret `$count` (utökad fråga) |
| `CountVariable anzahl` | lagrar det totala antalet i variabeln `$anzahl` |
| `Property UsageLocation` | hämtar även användningsplatsen |
| `Group-Object UsageLocation` | räknar användarna per land |

</details>

Detta antal är den övre gränsen. Hur många användare som faktiskt saknade åtkomst ska du styrka med egna data: helpdeskärenden med tidsstämplar, felmeddelanden från Outlook och Outlook på webben (skärmbilder), resultat från tillgänglighetstester i din övervakning samt inloggnings- och åtkomstloggar. Ju mer exakta dessa uppgifter är, desto mindre beror resultatet enbart på Microsofts telemetri.

## Små företag: oftast via partnern

Små och medelstora företag köper vanligtvis Microsoft 365 i Schweiz via en partner i Cloud Solution Provider Program (CSP) eller direkt med kreditkort i Admin Center. Vilken väg som gäller ser du i Microsoft 365 Admin Center under **Fakturering > Dina produkter**: om en partner anges som inköpskälla är avtalet ett CSP-avtal.

### Köpt via en CSP-partner

Microsoft tar i detta fall inte emot ansökan direkt från dig. Enligt Partner Center får endast CSP-direktleverantörer och indirekta leverantörer (distributörer) ansöka om krediteringar, inte indirekta återförsäljare. Om din IT-leverantör är en indirekt återförsäljare går ansökan alltså via denne till distributören och därifrån till Microsoft. Varje led har egna, ofta tidigare tidsfrister. Kontakta därför partnern nu och inte först i slutet av december.

Din partner behöver från dig:

1.  klientorganisations-ID:t,
2.  incidentens identifierare (EX1550543),
3.  tidsperiod, antal och platser för de berörda användarna samt de observerade felen,
4.  ett e-postmeddelande där du ansöker om SLA-krediteringen. Det måste skickas från en adress i den berörda domänen; Microsoft accepterar inte e-post från en privat adress.

Microsoft känner redan till själva incidenten. Ansökan måste styrka att din organisation var berörd: enbart namn, klientorganisations-ID och ärendenummer räcker enligt Microsoft inte. Om krediteringen beviljas krediterar Microsoft partnern, som sedan förmedlar den till dig. Om och hur detta sker regleras av ditt avtal med partnern. Fråga om inget syns på nästa faktura.

### Köpt direkt från Microsoft

Om du har betalat Microsoft 365 direkt till Microsoft skickar du själv in ansökan i Admin Center. Det kräver en administratörsroll:

1.  Öppna **Hjälp och support** längst ned till höger under `admin.cloud.microsoft`.
2.  Beskriv ärendet, till exempel ”Request an SLA service credit for a Microsoft 365 outage”, och välj **Kontakta support**.
3.  Välj e-post som kontaktväg. Rubriken ska innehålla ”SLA service credit request” och identifieraren EX1550543.
4.  Ange de fem obligatoriska uppgifterna från föregående avsnitt i beskrivningen och bifoga underlagen.

En separat ansökan krävs för varje tjänst och incident. Enligt SLA:t handlägger Microsoft normalt ansökningar inom 45 dagar.

### Är det värt det för små företag?

För tio användare är krediteringen liten: för en svit räknas endast Exchange Online-andelen, och av denna 25 eller 50 procent för en månad. Arbetet är också begränsat om du sammanställer uppgifterna nu. Ansökan har en andra nytta: den dokumenterar för partnern och ledningen hur hårt din organisation påverkades. Det hjälper senare vid frågan om en reservlösning för e-post behövs.

## Stora företag: direkt med Microsoft, med egna mätdata

Stora organisationer licensierar vanligtvis Microsoft 365 via ett Enterprise Agreement (EA) eller Microsoft Customer Agreement for Enterprise (MCA-E). Ansökan går då direkt till Microsoft, antingen via Admin Center enligt beskrivningen ovan eller via Unified-supportavtalet. Om du har en Customer Success Account Manager (CSAM) bör du informera denne om ansökan; personen kan följa handläggningen. Vid ett EA via en Licensing Solution Partner (LSP) bör du dessutom klargöra på vilken faktura krediteringen visas.

Vid flera tusen användare handlar det om betydande belopp, och Microsoft granskar därför uppgifterna särskilt noggrant. Skillnaden jämfört med en liten klientorganisation ligger i bevisningen:

1.  **Fastställ berörda användare exakt.** Stora klientorganisationer är fördelade över flera databaser och regioner i Microsofts datacenter, och alla postlådor var inte störda lika länge. Att schablonmässigt rapportera alla användare som berörda är sårbart för stora klientorganisationer. Mätdata är mer tillförlitliga: syntetiska tester mot Outlook på webben per plats, tillgänglighetsdata från den egna övervakningen och analyser av helpdeskärenden efter plats och tid.

2.  **Ange platser.** SLA:t kräver platserna för de berörda användarna. För internationella organisationer lönar det sig att dela upp efter land, eftersom störningen först drabbade Nord- och Sydamerika och först senare andra regioner.

3.  **Avgränsa hybridmiljön tydligt.** Fördröjningar i det egna datacentret, exempelvis e-post i kön på lokala Exchange-servrar på väg till klientorganisationen, undantas av SLA:t. Styrk att felet låg inom Microsoft 365. Köerna på de lokala servrarna är ett bra underlag för detta, eftersom de visar att Exchange Online inte tog emot e-post eller bara tog emot den fördröjt.

4.  **Välj service level.** Om det hos er främst var leveranstiderna som påverkades och postlådeåtkomsten i mindre grad, kan service level för leveranstid vara en bättre grund. Endast en ansökan är tillåten per incident och tjänst, så valet bör göras före ansökan.

5.  **Involvera inköp och licenshantering.** Krediteringen visas på fakturan, inte i IT-kostnadsstället. Stäm av med inköp vem som skickar ansökan, vem som kontrollerar krediteringen och hur den bokförs.

Stora organisationer bör dessutom förbereda processen inför nästa avbrott: en mall med de fem obligatoriska uppgifterna, övervakning som mäter tillgängligheten för Outlook på webben per plats och ett tydligt ansvar för SLA-ansökningar.

## Mall för ansökan

Följande mall innehåller de obligatoriska uppgifterna från SLA:t. Microsoft handlägger ansökningar på engelska snabbast:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Källor

1.  [Service Level Agreement for Microsoft Online Services (översikt över versioner)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): version från den 17 november 2026, definition av avbrottstid för Exchange Online, krediteringsnivåer, obligatoriska uppgifter, tidsfrister och undantag.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): ansökningsprocess för CSP-partner, undantag för indirekta återförsäljare, nödvändiga underlag, klassificering av Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet för att hämta en tjänsthälsoincident via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): offentliga statusmeddelanden om EX1550543 och MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): ansökningsväg i Admin Center för novemberavbrottet, information om privatkunder.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): undantag för privat-, gratis- och ideella licenser, en ansökan per tjänst och incident.

7.  [Liverapportering: Exchange Online, globalt avbrott i postlådeåtkomst](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): störningens förlopp den 12 och 13 november 2026 med Microsofts kronologi.
