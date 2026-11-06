---
title: "Teams-voicemail via e-post: ny avsändare och ny ämnesrad från mars 2027 (MC1511072)"
navTitle: "Voicemail-avsändare"
description: "Microsoft ändrar leveransen av Teams-voicemail via e-post: avsändaren blir framöver «Microsoft Teams» från den egna klientdomänen och uppringaruppgifterna flyttas till ämnesraden. Artikeln visar vem som påverkas, hur du granskar transportregler, tillåtelselistor och automatiseringar samt vad som måste anpassas före mars 2027."
date: "2026-11-02"
kategorie: "Microsoft Teams"
timeToRead: "4 min. lästid"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
  - "exchange-online"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "teams-voicemail-via-e-post-ny-avsandare-och-ny-amnesrad-fran-mars-2027-mc1511072"
translationId: "article-b75c975f086dd0d5"
translationOf: teams-voicemail-mail-absender
url: https://rafaelpfister.ch/sv/blog/teams-voicemail-via-e-post-ny-avsandare-och-ny-amnesrad-fran-mars-2027-mc1511072
translationSourceHash: 32089440ce20d39865ec421317bed6588c48eace59eb8f114015c626da412eba
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:19:05.039Z
translationReview: automatic
---

I meddelandet från Message Center MC1511072 av den 16 oktober 2026 meddelar Microsoft en ny leverans av voicemail-e-postmeddelanden från Teams. Enligt nuvarande plan börjar utrullningen i början av mars 2027 och ska vara slutförd i slutet av juni 2027. Microsoft förbehåller sig rätten att skjuta upp den och avser att återkomma med bindande datum samt eventuella kundåtgärder i ett senare meddelande. Bakgrunden är stöd för egna avsändardomäner (Bring Your Own Domain), så att voicemail-e-postmeddelanden omfattas av organisationens SPF-, DKIM- och DMARC-principer.

## Vad som ändras

Fältet «Från» kommer framöver varken att innehålla uppringarens namn, telefonnummer eller e-postadress. Dessa uppgifter flyttas i stället till ämnesraden och finns fortfarande kvar i meddelandetexten. I Teams förblir visningen oförändrad.

| Egenskap | I dag | Efter utrullningen |
|---|---|---|
| Avsändare (personlig voicemail) | Uppringarens namn eller nummer | `"Microsoft Teams" no-reply@<Standard-Tenant-Domain>` eller egen domän, om den har konfigurerats |
| Avsändardomän | `skype.voicemail.microsoft.com` vid PSTN-samtal, klientdomän vid Teams-till-Teams-samtal | Standardklientdomän eller egen domän |
| Ämnesrad (personlig voicemail) | `Voice mail (22 seconds)` | `Identified: John Smith (+19371817273) - Voicemail (00:22)` respektive `Unidentified: +19371817273 - Voicemail (00:22)` |
| Samtalskö (delad voicemail) | `Shared Voicemail (<CQ-Name>)` | `John Smith (+37251223123) - Shared voicemail in <CQ-Name> (00:05)` |
| Automatisk telefonväxel (delad voicemail) | `Shared Voicemail (<AA-Name>)` | `John Smith - Shared voicemail in <AA-Name> (00:11)` |

För användarna får detta två synliga följder: I inkorgen visas alla voicemail-meddelanden med samma avsändare, «Microsoft Teams», och eftersom adressen är en no-reply-adress når funktionen «Svara» inte längre uppringaren.

## Vem som påverkas

Alla organisationer som använder Teams Calling med voicemail påverkas, alltså Teams Phone med Calling Plan, Operator Connect eller Direct Routing samt samtalsköer och automatiska telefonväxlar med delad voicemail. Meddelandet nämner inga licens- eller molnbegränsningar; utrullningen har aviserats globalt.

Åtgärder krävs där något är kopplat till den nuvarande avsändaren eller ämnesraden:

- Exchange-transportregler som kontrollerar `skype.voicemail.microsoft.com`, uppringaren som avsändare eller ämnesraden `Voice mail` respektive `Shared Voicemail`
- poster i tillåtelselistor: Tenant Allow/Block List, spoofing-tillåtelser, `AllowedSenderDomains` i anti-spam-principer
- inkorgsregler för användare och delade postlådor för samtalsköer som sorterar eller vidarebefordrar voicemail utifrån avsändare eller ämnesrad
- Power Automate-flöden, ärendehanteringssystem och CRM-integreringar som läser uppringarens telefonnummer från avsändaren
- uppströms- eller nedströmsgatewayer (till exempel vid centraliserat e-postflöde i hybriddrift) som innehåller regler eller undantag för voicemail-domänen

## Så kontrollerar du om du påverkas

Följande frågor körs i en PowerShell-session för Exchange Online (`Connect-ExchangeOnline`).

### Sök igenom transportregler

```powershell
$muster = 'skype\.voicemail|voice ?mail|shared voicemail'
Get-TransportRule |
  Where-Object { ($_ | Format-List * | Out-String) -match $muster } |
  Select-Object Name, State, Priority
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `$muster` | Reguljärt uttryck för den nuvarande avsändardomänen och de nuvarande ämnesradstexterna |
| `Format-List *` | Visar regelns alla egenskaper, så att även villkor och undantag genomsöks |
| `Out-String` | Omvandlar utdata till text som `-match` tillämpas på |
| `Select-Object Name, State, Priority` | Visar endast namn, status och ordning för de hittade reglerna |

</details>

Kontrollera varje hittad regel för villkor som `FromAddressContainsWords`, `SenderDomainIs` eller `SubjectContainsWords`.

### Tillåtelselistor och anti-spam-principer

```powershell
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object Value -match 'voicemail'
Get-TenantAllowBlockListSpoofItems |
  Where-Object SpoofedUser -match 'voicemail'
Get-HostedContentFilterPolicy |
  Select-Object Name, AllowedSenderDomains, AllowedSenders
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-ListType Sender` | Läser avsändarposter från Tenant Allow/Block List |
| `Where-Object Value -match 'voicemail'` | Filtrerar poster vars värde innehåller voicemail-domänen |
| `Get-TenantAllowBlockListSpoofItems` | Listar spoofing-tillåtelser och -blockeringar |
| `AllowedSenderDomains, AllowedSenders` | Tillåtelser direkt i anti-spam-principerna |

</details>

### Fastställ faktisk e-postvolym

Meddelandespårningen visar om och vart voicemail-meddelanden levereras i dag:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddDays(-10) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.SenderAddress -like '*@skype.voicemail.microsoft.com' -or $_.Subject -match 'voice ?mail' } |
  Group-Object RecipientAddress | Sort-Object Count -Descending |
  Select-Object Count, Name
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-StartDate` / `-EndDate` | Sökperiod; Exchange Online tillhandahåller högst de senaste 90 dagarna och högst 10 dagar per fråga |
| `-ResultSize 5000` | Maximalt antal träffar per fråga |
| `Where-Object …` | Behåller e-postmeddelanden från PSTN-voicemail-domänen eller med voicemail-ämnesrad |
| `Group-Object RecipientAddress` | Räknar voicemail-meddelanden per mottagare |

</details>

Mottagarlistan visar vilka postlådor och delade postlådor som tar emot voicemail. Där är det värt att kontrollera inkorgsregler (`Get-InboxRule -Mailbox <Adresse>`) och anslutna flöden.

## Vad som behöver göras

1.  Ställ om regler från avsändare till ämnesrad: Rikta nya villkor mot de framtida ämnesradsmönstren `- Voicemail (` och `- Shared voicemail in `. Låt gamla och nya villkor köras parallellt tills utrullningen har nått den egna klientorganisationen.

2.  Ta bort tillåtelser för `skype.voicemail.microsoft.com` efter utrullningen. E-post från den egna klientdomänen behöver inget undantag; en tillåtelse för den egna domänen skulle vara en säkerhetsrisk eftersom den underlättar spoofing.

3.  Anpassa automatiseringar som läser uppringarens nummer från avsändaren. Numret kommer framöver att stå inom parentes i ämnesraden eller, för okända uppringare, direkt efter `Unidentified:`.

4.  Förbered egen avsändardomän: Microsoft aviserar konfigurationsanvisningar för utskick under en egen domän i uppföljningsmeddelandet. Om DMARC för denna domän är satt till `p=reject`, måste SPF och DKIM täcka utskick från Exchange Online, annars hamnar voicemail-meddelandena i karantän.

5.  Informera användare och servicedesk: avsändare «Microsoft Teams», uppringaruppgifter i ämnesraden, det går inte längre att svara uppringaren via e-post.

## Källor

1.  [MC1511072: Microsoft Teams: Upcoming change to voicemail message delivery by email](https://mc.merill.net/message/MC1511072): arkiverad text av Message Center-meddelandet med tidsplan och jämförelse mellan nuvarande och framtida avsändare och ämnesrad.

2.  [Microsoft Learn: Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): parametrar och begränsningar för meddelandespårning i Exchange Online.

3.  [Microsoft Learn: Get-TenantAllowBlockListItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-tenantallowblocklistitems): läsning av Tenant Allow/Block List.

4.  [Microsoft Learn: Set up voicemail for Teams](https://learn.microsoft.com/en-us/microsoftteams/set-up-phone-system-voicemail): grunderna för Teams-voicemail och leverans via e-post.
