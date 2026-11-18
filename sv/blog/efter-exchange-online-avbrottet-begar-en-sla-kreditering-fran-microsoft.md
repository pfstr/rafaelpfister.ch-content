---
title: "Efter Exchange Online-avbrottet: begär SLA-kredit från Microsoft"
navTitle: "SLA-kredit"
description: "Exchange Online-avbrottet den 12 och 13 november 2026 varade i cirka 22 timmar och innebar därmed för många klientorganisationer en tillgänglighet under den garanterade nivån på 99,9 procent. Microsoft krediterar inte beloppet automatiskt; ansökan måste lämnas in senast den 31 december 2026. Så beräknar du tillgängligheten, vilka underlag Microsoft kräver och varför små företag bör gå via sin partner medan stora företag bör vända sig direkt till Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lästid"
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
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:23:50.489Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/efter-exchange-online-avbrottet-begar-en-sla-kreditering-fran-microsoft
---

# Efter Exchange Online-avbrottet: begär SLA-kredit från Microsoft

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Kostnadsfri hjälp med SLA-ansökan</p>
<p>Vi hjälper dig kostnadsfritt att ansöka om kredit för Exchange Online-avbrottet den 12 och 13 november 2026 hos Microsoft eller din partner: beräkna tillgänglighet, sammanställa underlag, lämna in ansökan. Tidsfristen löper ut den 31 december 2026. Kontakta oss via <a href="https://rafaelpfister.ch/kontakt">kontaktformuläret</a>.</p>
</div>

Från den 12 november 2026 klockan 18:54 till den 13 november 2026 klockan 17:10 (schweizisk tid) hade många organisationer ingen eller endast periodvis åtkomst till sina Exchange Online-postlådor. Microsoft registrerade störningen i Admin Center under EX1550543 (Exchange Online) och MO1550821 (andra Microsoft 365-tjänster). Jag har dokumenterat förloppet i [liverapporteringen om avbrottet](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). För Exchange Online garanterar Microsoft en månatlig tillgänglighet på 99,9 procent. Ett avbrott på drygt 22 timmar ligger klart under detta. Du får dock bara en kredit om du ansöker om den, och vägen dit beror på hur din organisation har köpt Microsoft 365.

## Vad SLA:t garanterar

Det som gäller är «Service Level Agreement for Microsoft Online Services», som Microsoft publicerar på nytt varje månad. Den här artikeln bygger på versionen från den 17 november 2026. För Exchange Online definierar SLA:t driftstopp som varje tidsperiod då användare inte kan skicka eller ta emot e-post med Outlook på webben. Det finns inga planerade underhållsfönster för Exchange Online.

Tillgängligheten beräknas i användarminuter:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Användarminuterna är antalet minuter i månaden multiplicerat med antalet användare. Avbrottsminuterna är varaktigheten för varje incident multiplicerat med antalet berörda användare. Om tillgängligheten understiger en tröskel ges en kredit på månadsavgiften för den berörda tjänsten:

| Tillgänglighet under månaden | Kredit |
|---|---|
| under 99,9 % | 25 % |
| under 99 % | 50 % |
| under 95 % | 100 % |

Samma trösklar gäller för Microsoft Teams, SharePoint Online och OneDrive, men med egna definitioner av driftstopp. För Teams räknas det endast om användare inte kan chatta eller starta möten. Under EX1550543 fungerade chattar och möten enligt Microsoft till största delen; det var kalendrar och inbjudningar som stördes. En ansökan för Teams har därför sannolikt små utsikter att godkännas.

Utöver postlådetillgängligheten har Exchange Online fler servicenivåer, exempelvis leveranstid inom Microsoft 365 och mottagning av inkommande e-post. Om en incident påverkar flera av dem måste du enligt SLA:t välja en. För novemberavbrottet är postlådetillgänglighet via Outlook på webben det närliggande måttet.

## Vad avbrottet innebär i procent

November 2026 har 30 dagar, alltså 43'200 minuter per användare. Detta ger följande gränsvärden per användare:

| Tröskel | Tillåten avbrottstid i november |
|---|---|
| 99,9 % | 43 minuter |
| 99 % | 7 timmar 12 minuter |
| 95 % | 36 timmar |

Störningen varade 1'336 minuter från den första rapporten till åtgärd. Två räkneexempel för en organisation med 200 postlådor:

| Fall | Avbrottsminuter | Tillgänglighet | Kredit |
|---|---|---|---|
| Samtliga 200 användare utan åtkomst under hela störningen | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 användare utan åtkomst i 10 timmar | 120 x 600 = 72'000 | 99,17 % | 25 % |

Det totala antalet användarminuter är i båda fallen 200 x 43'200 = 8'640'000. Avgörande är hur många användare som faktiskt var berörda och hur länge. Microsoft kontrollerar detta med egen telemetri, men kräver att du uppger detta i ansökan. Många klientorganisationer i Europa påverkades först från sent på kvällen den 12 november eller endast periodvis. Beräkna därför med de tider du kan styrka, inte med hela varaktigheten i Microsofts meddelande.

## Vad krediten är och inte är

SLA:t reglerar några punkter som bör vara klara före ansökan:

1.  **Endast på ansökan.** Microsoft granskar och beviljar inte krediter på eget initiativ. En statusnotis från Microsoft utgör enligt SLA:t uttryckligen inte en bekräftelse på att en servicenivå inte har uppnåtts.

2.  **Kredit, inte återbetalning.** Krediten kvittas mot en framtida faktura. Den är begränsad till månadsavgiften för den berörda tjänsten.

3.  **Enda rättsmedlet.** Enligt SLA:t är krediter det enda rättsmedlet vid tillgänglighetsproblem. Microsoft ersätter inte intäktsbortfall, övertid eller andra följdkostnader. Du får inte självständigt minska fakturor.

4.  **Endast den berörda tjänsten.** Har du en svit som Microsoft 365 E3 eller Business Premium avser krediten Exchange Online-andelen, inte hela licenspriset. Microsoft fastställer hur stor den andelen är vid granskningen; SLA:t anger ingen formel för detta.

5.  **Ingen kredit för gratis- och testerbjudanden.** Testversioner, förhandsversioner och kostnadsfria tjänster är undantagna, liksom donerade licenser för ideella organisationer. Privatkunder med Microsoft 365 Personal eller Family omfattas inte av detta SLA.

6.  **Open-avtal i dagar.** Den som har licensierat via Open, Open Value eller Open Value Subscription får en eventuell kredit som ytterligare användningsdagar i stället för ett penningbelopp.

7.  **Tidsfrist.** För Microsoft 365-tjänster måste ansökan med alla uppgifter ha inkommit senast vid utgången av månaden efter incidentmånaden. För novemberavbrottet är det **31 december 2026**. Azure har en egen tidsfrist på 60 dagar.

## Vilka uppgifter Microsoft kräver

SLA:t anger fem obligatoriska uppgifter. Om de saknas avslår Microsoft ansökan:

1.  en detaljerad beskrivning av incidenten,
2.  tidpunkt och varaktighet för driftstoppet,
3.  de berörda resurserna (för Exchange Online: klientorganisation och tjänst),
4.  antal och platser för berörda användare,
5.  en beskrivning av de fel som uppstod.

Incidentens identifierare (EX1550543) och klientorganisations-ID:t ska alltid inkluderas. Båda kan hämtas med Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | Behörighet att läsa meddelanden om tjänstens hälsa |
| `-Scopes "Organization.Read.All"` | Behörighet att läsa organisationsdata, inklusive klientorganisations-ID:t |
| `(Get-MgOrganization).Id` | visar klientorganisations-ID:t |
| `-ServiceHealthIssueId "EX1550543"` | hämtar just denna incident |
| `Select-Object …` | visar incidentens identifierare, titel, start, slut och klassificering |

</details>

Klassificeringen (`Classification`) visar om Microsoft registrerar incidenten som `incident` eller som `advisory`. Enligt Microsofts partnerdokumentation är meddelanden på nivån «Advisory» normalt inte berättigade till kredit. EX1550543 klassificerades som Incident.

För antal och plats för användarna räcker en sammanställning av licensierade postlådor efter användningsplats:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `$abfrage = @{ … }` | samlar parametrarna för `Get-MgUser` i en hashtabell (splatting) så att raden hålls kort |
| `All` | läser alla användare, inte bara första sidan |
| `Filter 'assignedLicenses/$count ne 0'` | endast användare med minst en tilldelad licens |
| `ConsistencyLevel eventual` | krävs för filtret `$count` (utökad fråga) |
| `CountVariable anzahl` | lagrar totalantalet i variabeln `$anzahl` |
| `Property UsageLocation` | hämtar även användningsplatsen |
| `Group-Object UsageLocation` | räknar användarna per land |

</details>

Detta antal är den övre gränsen. Hur många användare som faktiskt saknade åtkomst styrker du med egna data: helpdeskärenden med tidsstämplar, felmeddelanden från Outlook och Outlook på webben (skärmbilder), resultat från tillgänglighetstester i din övervakning samt inloggnings- och åtkomstloggar. Ju mer exakta dessa uppgifter är, desto mindre beror resultatet enbart på Microsofts telemetri.

## Små företag: oftast via partnern

Små och medelstora företag köper vanligtvis Microsoft 365 i Schweiz via en partner i Cloud Solution Provider Program (CSP) eller direkt med kreditkort i Admin Center. Vilken väg som gäller ser du i Microsoft 365 Admin Center under **Fakturering > Dina produkter**: Om en partner anges som inköpskälla är avtalet ett CSP-avtal.

### Köpt via en CSP-partner

Microsoft tar i detta fall inte emot ansökan direkt från dig. Enligt Partner Center får endast CSP-direktleverantörer och indirekta leverantörer (distributörer) ansöka om krediter, inte indirekta återförsäljare. Om din IT-leverantör är en indirekt återförsäljare går ansökan alltså via denne till dess distributör och därifrån till Microsoft. Varje steg har egna, ofta tidigare tidsfrister. Hör därför av dig nu och inte först i slutet av december.

Din partner behöver från dig:

1.  klientorganisations-ID:t,
2.  incidentens identifierare (EX1550543),
3.  tidsperiod, antal och platser för de berörda användarna samt de observerade felen,
4.  ett e-postmeddelande där du ansöker om SLA-krediten. Det måste komma från en adress i den berörda domänen; Microsoft accepterar inte e-post från en privat adress.

Microsoft känner själv till incidenten. Ansökan måste styrka att din organisation var berörd: enligt Microsoft räcker inte namn, klientorganisations-ID och ärendenummer ensamma. Om krediten beviljas krediterar Microsoft partnern, och partnern för den vidare till dig. Om och hur detta sker regleras i ditt avtal med partnern. Fråga om inget syns på nästa faktura.

### Köpt direkt från Microsoft

Om du har betalat Microsoft 365 direkt till Microsoft lämnar du själv in ansökan i Admin Center. Det kräver en administratörsroll:

1.  Öppna **Hjälp och support** längst ned till höger under `admin.cloud.microsoft`.
2.  Beskriv ärendet, till exempel «Request an SLA service credit for a Microsoft 365 outage», och välj **Kontakta support**.
3.  Välj e-post som kontaktväg. Rubriken ska innehålla «SLA service credit request» och identifieraren EX1550543.
4.  Ange de fem obligatoriska uppgifterna från föregående avsnitt i beskrivningen och bifoga underlagen.

En separat ansökan krävs för varje tjänst och incident. Enligt SLA:t handlägger Microsoft normalt ansökningar inom 45 dagar.

### Är det värt det för små företag

För tio användare är krediten liten: för en svit räknas endast Exchange Online-andelen, och av den 25 eller 50 procent för en månad. Arbetsinsatsen är också liten om du sammanställer uppgifterna nu. Ansökan har ytterligare en fördel: den dokumenterar gentemot partner och företagsledning hur kraftigt din organisation påverkades. Det hjälper senare vid frågan om en reservlösning för e-post behövs.

## Stora företag: direkt med Microsoft, med egna mätdata

Stora organisationer licensierar vanligtvis Microsoft 365 via ett Enterprise Agreement (EA) eller Microsoft Customer Agreement for Enterprise (MCA-E). Ansökan går då direkt till Microsoft, antingen via Admin Center enligt beskrivningen ovan eller via Unified Support-avtalet. Om du har en Customer Success Account Manager (CSAM), informera denne om ansökan; personen kan följa handläggningen. Vid ett EA via en Licensing Solution Partner (LSP) bör du dessutom klargöra på vilken faktura krediten visas.

För flera tusen användare handlar det om betydande belopp, och Microsoft granskar därför uppgifterna särskilt noga. Skillnaden jämfört med en liten klientorganisation ligger i bevisningen:

1.  **Fastställ berörda användare exakt.** Stora klientorganisationer är fördelade över flera databaser och regioner i Microsofts datacenter, och alla postlådor hade inte störningar lika länge. Att schablonmässigt rapportera alla användare som berörda är sårbart för stora klientorganisationer. Mätdata är mer robusta: syntetiska tester mot Outlook på webben per plats, tillgänglighetsdata från egen övervakning, analyser av helpdeskärenden per plats och tid.

2.  **Ange platser.** SLA:t kräver platserna för de berörda användarna. För internationella organisationer är det värt att dela upp efter land, eftersom störningen först drabbade Nord- och Sydamerika och först senare andra regioner.

3.  **Avgränsa hybridmiljöer korrekt.** Fördröjningar i det egna datacentret, exempelvis e-post i kön från lokala Exchange-servrar till klientorganisationen, undantas av SLA:t. Styrk att felet låg inom Microsoft 365. Köerna på de lokala servrarna är ett bra underlag eftersom de visar att Exchange Online inte tog emot e-post eller endast tog emot den fördröjt.

4.  **Välj servicenivå.** Om främst leveranstiderna och i mindre grad postlådeåtkomsten påverkades hos dig, kan servicenivån för leveranstid vara en bättre grund. Endast en ansökan per incident och tjänst är tillåten, därför bör valet göras före ansökan.

5.  **Involvera inköp och licenshantering.** Krediten visas på fakturan, inte på IT-kostnadsstället. Stäm av med inköp vem som lämnar in ansökan, vem som kontrollerar krediten och hur den bokförs.

Stora organisationer bör dessutom förbereda processen inför nästa avbrott: en mall med de fem obligatoriska uppgifterna, övervakning som mäter tillgängligheten för Outlook på webben per plats samt ett tydligt ansvar för SLA-ansökningar.

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

1.  [Service Level Agreement for Microsoft Online Services (översikt över versioner)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): version från den 17 november 2026, definition av driftstopp för Exchange Online, kreditnivåer, obligatoriska uppgifter, tidsfrister och undantag.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): ansökningsväg för CSP-partner, undantag för indirekta återförsäljare, nödvändiga underlag, klassificering av Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet för att hämta en incident för tjänstens hälsa via Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): offentliga statusmeddelanden om EX1550543 och MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): ansökningsväg i Admin Center för novemberavbrottet, information om privatkunder.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): undantag för privat-, gratis- och ideella licenser, en ansökan per tjänst och incident.

7.  [Liverapportering: Exchange Online, globalt avbrott i postlådeåtkomst](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): förloppet för störningen den 12 och 13 november 2026 med Microsofts kronologi.
