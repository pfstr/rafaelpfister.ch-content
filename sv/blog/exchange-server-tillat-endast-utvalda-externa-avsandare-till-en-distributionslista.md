---
title: "Exchange: Tillåt endast utvalda externa avsändare till en distributionslista"
navTitle: "Distributionslista: externa avsändare"
description: "En distributionslista i Exchange kan konfigureras så att endast vissa externa avsändare kan nå den. Två sätt med PowerShell: leveransbegränsning med e-postkontakter eller transportregel. Dessutom begränsningarna i avsändarkontrollen och hur DMARC täpper till dem."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min lästid"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-on-premises"
protokolle:
  - "smtp"
  - "mail-auth"
  - "powershell"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-message-tracking-und-receive-connectoren-analysieren
slug: "exchange-server-tillat-endast-utvalda-externa-avsandare-till-en-distributionslista"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
translationSourceHash: 14114b684164aa90e68aac9d9ca4f75ed59c1ce77cf6479a167f154e66788d45
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T15:43:28.342Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/exchange-server-tillat-endast-utvalda-externa-avsandare-till-en-distributionslista
---

# Exchange: Tillåt endast utvalda externa avsändare till en distributionslista

En distributionslista i Exchange accepterar som standard endast meddelanden från autentiserade, alltså interna, avsändare. Om en partner, leverantör eller kund också ska kunna nå listan utan att den därmed är öppen för hela internet, finns det två möjligheter: en leveransbegränsning på själva gruppen eller en transportregel. Båda fungerar i Exchange Server 2016, 2019 och Subscription Edition.

## Utgångsläge: RequireSenderAuthenticationEnabled

Om en grupp över huvud taget accepterar externa meddelanden styrs av egenskapen `RequireSenderAuthenticationEnabled`. Om den är satt till `$true` (standard för nya distributionsgrupper) avvisar Exchange alla meddelanden från en icke-autentiserad avsändare. Båda följande varianter förutsätter därför att denna kontroll är avstängd:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Identity` | Distributionsgruppen (namn, alias eller e-postadress). |
| `-RequireSenderAuthenticationEnabled $false` | Gruppen accepterar även meddelanden från icke-autentiserade, externa avsändare. |

</details>

Utan ytterligare begränsning kan varje extern avsändare skriva till listan efter detta steg. Begränsningen till enskilda avsändare följer i nästa steg, helst inom samma underhållsfönster.

## Variant 1: Leveransbegränsning med e-postkontakter

Listan över tillåtna avsändare (`AcceptMessagesOnlyFromSendersOrMembers`) accepterar endast objekt från den egna katalogen: postlådor, e-postanvändare, e-postkontakter och grupper. Det går inte att ange en ren SMTP-adress där. Skapa därför en e-postkontakt för varje extern avsändare. Dölj kontakten så att den inte visas i den globala adresslistan:

```powershell
$kontakt = @{
    Name                 = "Extern Max Muster"
    ExternalEmailAddress = "max.muster@partner.example"
    OrganizationalUnit   = "example.local/Kontakte/Extern"
}
New-MailContact @kontakt
Set-MailContact -Identity "Extern Max Muster" -HiddenFromAddressListsEnabled $true
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Name` | Kontaktens visningsnamn i Active Directory. Ett enhetligt prefix som ”Extern” underlättar senare administration. |
| `ExternalEmailAddress` | Den externa adress som avsändaren skriver från. |
| `OrganizationalUnit` | Mål-OU i Active Directory. Utan angivelse hamnar kontakten i standardcontainern `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Döljer kontakten från den globala adresslistan och alla adresslistor. |

</details>

Lägg sedan till kontakten som tillåten avsändare. Hashtable-tabellen med `Add` utökar den befintliga listan i stället för att skriva över den:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Lista över avsändare som får nå gruppen. Enskilda avsändare och grupper är möjliga; för grupper får alla medlemmar skicka. |
| `@{ Add = ... }` | Lägger till poster utan att ta bort befintliga. Med `@{ Remove = ... }` tas poster bort; ett värde utan hashtable ersätter hela listan. |

</details>

Så snart denna lista innehåller minst en post avvisar Exchange **alla andra avsändare**, även interna. Om medarbetare fortfarande ska kunna skriva till listan lägger du dessutom till en lämplig grupp, till exempel distributionsgruppen själv (då får alla medlemmar skicka) eller en grupp med alla medarbetare. Avvisade avsändare får en leveransstatusrapport med statusen `5.7.1`.

Kontrollera aktuellt läge så här:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` är en samlingsparameter: Exchange fördelar posterna på egenskaperna `AcceptMessagesOnlyFrom` (enskilda avsändare) och `AcceptMessagesOnlyFromDLMembers` (grupper). Därför visar frågan dessa båda egenskaper. Samma inställning finns i Exchange Admin Center, i gruppens egenskaper under leveranshantering (Delivery management).

Denna variant passar när endast ett fåtal externa avsändare tillkommer och listan över tillåtna avsändare ska vara synlig på själva gruppobjektet. Nackdelen är administrationsarbetet: varje ny avsändare behöver en kontakt i Active Directory.

## Variant 2: Transportregel utan kontakter

Om alla internt fortfarande ska kunna skriva till listan och endast externa avsändare ska begränsas är en transportregel enklare. Den kräver inga kontakter och kan även tillämpas på hela domäner. `RequireSenderAuthenticationEnabled` måste även här vara satt till `$false`.

```powershell
$regel = @{
    Name                              = "Projektteam: nur freigegebene externe Absender"
    SentTo                            = "projektteam@example.com"
    FromScope                         = "NotInOrganization"
    ExceptIfFromAddressMatchesPatterns = '^max\.muster@partner\.example$',
                                         '@lieferant\.example$'
    RejectMessageReasonText           = "Sie sind nicht berechtigt, an diese Adresse zu senden."
    RejectMessageEnhancedStatusCode   = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `Name` | Regelns namn i listan över transportregler. |
| `SentTo` | Villkor: Meddelandet är adresserat till denna grupp. |
| `FromScope NotInOrganization` | Villkor: Avsändaren tillhör inte den egna organisationen. Interna meddelanden påverkas därmed inte. |
| `ExceptIfFromAddressMatchesPatterns` | Undantag: Avsändaradresser som motsvarar något av de reguljära uttrycken. `^…$` förankrar en enskild adress, `@domain$` tillåter en hel domän. Escapa punkter med `\.`. |
| `RejectMessageReasonText` | Text i leveransstatusrapporten till den avvisade avsändaren. |
| `RejectMessageEnhancedStatusCode` | Utökad statuskod i leveransstatusrapporten; `5.7.1` står för saknad behörighet. |

</details>

Utan förankringen med `^` och `$` skulle ett mönster som `max\.muster@partner\.example` även passa `max.muster@partner.example.net`. Lägg till nya avsändare med `Set-TransportRule` och den fullständiga, utökade listan med mönster; parametern ersätter den befintliga listan.

## Begränsning i båda varianterna: Avsändaradressen kan förfalskas

Båda varianterna kontrollerar endast avsändaradressen. Exchange fastställer inte om meddelandet verkligen kommer från den godkända partnern. Den som känner till adressen till en godkänd avsändare kan förfalska den och därmed nå listan. För en lista med interna deltagare är detta en ingångspunkt för nätfiske, som dessutom ger intryck av att komma från en välkänd adress.

SPF, DKIM och DMARC täpper till luckan. Om partnerns domän har en DMARC-policy med `p=reject` eller `p=quarantine` och din e-postgateway tillämpar den policyn, når förfalskade meddelanden från denna domän inte ens Exchange. För domäner utan DMARC-policy eller med `p=none` saknas detta skydd.

Om din gateway skriver kontrollresultatet i huvudet `Authentication-Results`, kan en andra transportregel dessutom avvisa godkända avsändare om DMARC inte har godkänts:

```powershell
$regel = @{
    Name                            = "Projektteam: freigegebene Absender nur mit DMARC pass"
    SentTo                          = "projektteam@example.com"
    FromScope                       = "NotInOrganization"
    FromAddressMatchesPatterns      = '^max\.muster@partner\.example$',
                                      '@lieferant\.example$'
    ExceptIfHeaderMatchesMessageHeader = "Authentication-Results"
    ExceptIfHeaderMatchesPatterns   = 'dmarc=pass'
    RejectMessageReasonText         = "Absender konnte nicht verifiziert werden."
    RejectMessageEnhancedStatusCode = "5.7.1"
}
New-TransportRule @regel
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `FromAddressMatchesPatterns` | Villkor: Regeln gäller endast för de godkända avsändarna, med samma mönster som i den första regeln. |
| `ExceptIfHeaderMatchesMessageHeader` | Namnet på det huvud som undantaget kontrollerar. |
| `ExceptIfHeaderMatchesPatterns` | Undantag: Om huvudet innehåller `dmarc=pass` levereras meddelandet. Alla övriga meddelanden från de godkända avsändarna avvisas. |

</details>

Denna regel fungerar bara tillförlitligt om gatewayen tar bort eller skriver över inkommande `Authentication-Results`-huvuden från internet. Annars kan en avsändare själv inkludera huvudet med `dmarc=pass`. Vilka huvuden din gateway sätter och om den tar bort externa huvuden beskrivs i dess dokumentation; Exchange utvärderar inte själv DMARC.

## Vilken variant passar

Leveransbegränsning med kontakter passar för listor med en fast, liten grupp externa avsändare och begränsar vid behov även interna avsändare. Transportregeln passar när alla internt får skicka, hela partnerdomäner ska godkännas eller inga kontakter ska skapas i Active Directory. Oavsett variant bör du kontrollera DMARC-policyn för varje godkänd domän och dokumentera vem som underhåller listan över tillåtna avsändare.

## Källor

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parametrarna AcceptMessagesOnlyFromSendersOrMembers och RequireSenderAuthenticationEnabled, tillåtna objekttyper.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): Hur Categorizer utvärderar leveransbegränsningar och avsändarautentisering.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Skapa e-postkontakter för externa adresser.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Villkor, undantag och åtgärder för transportregler, inklusive FromScope och HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specifikation av DMARC-policyerna p=none, quarantine och reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Strukturen för huvudet Authentication-Results och informationen om att mottagande system bör ta bort externa instanser.
