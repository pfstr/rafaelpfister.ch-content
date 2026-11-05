---
title: "Exchange Server: Tillat bare utvalgte eksterne avsendere til en distribusjonsliste"
navTitle: "Distribusjonsliste: eksterne avsendere"
description: "En distribusjonsliste i Exchange Server kan konfigureres slik at bare bestemte eksterne avsendere kan nå den. To metoder med PowerShell: leveringsbegrensning med e-postkontakter eller transportregel. I tillegg grensene for avsenderkontrollen og hvordan DMARC lukker dem."
date: "2026-11-05"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min lesetid"
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
slug: "exchange-server-tillat-bare-utvalgte-eksterne-avsendere-til-en-distribusjonsliste"
translationId: "article-b3dc70052f5e8729"
translationOf: exchange-verteiler-externe-absender
url: https://rafaelpfister.ch/no/blog/exchange-server-tillat-bare-utvalgte-eksterne-avsendere-til-en-distribusjonsliste
translationSourceHash: cff4b579945841f2469f23b5e44d834f6be7528cf9817f9d7ade0fd613f1c582
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:23:28.837Z
translationReview: automatic
---

# Exchange Server: Tillat bare utvalgte eksterne avsendere til en distribusjonsliste

En distribusjonsliste i Exchange Server godtar som standard bare meldinger fra autentiserte, altså interne avsendere. Hvis en partner, leverandør eller kunde også skal kunne nå listen uten at den dermed er åpen for hele internett, finnes det to muligheter: en leveringsbegrensning på selve gruppen eller en transportregel. Begge fungerer i Exchange Server 2016, 2019 og Subscription Edition.

## Utgangspunkt: RequireSenderAuthenticationEnabled

Om en gruppe i det hele tatt godtar eksterne meldinger, styres av egenskapen `RequireSenderAuthenticationEnabled`. Hvis den er satt til `$true` (standard for nye distribusjonsgrupper), avviser Exchange alle meldinger fra en ikke-autentisert avsender. Begge variantene nedenfor forutsetter derfor at denne kontrollen er deaktivert:

```powershell
Set-DistributionGroup -Identity "Projektteam" -RequireSenderAuthenticationEnabled $false
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-Identity` | Distribusjonsgruppen (navn, alias eller e-postadresse). |
| `-RequireSenderAuthenticationEnabled $false` | Gruppen godtar også meldinger fra ikke-autentiserte, eksterne avsendere. |

</details>

Uten ytterligere begrensninger kan enhver ekstern avsender skrive til listen etter dette trinnet. Begrensningen til enkeltavsendere følger i neste trinn, helst i samme vedlikeholdsvindu.

## Variant 1: Leveringsbegrensning med e-postkontakter

Listen over tillatte avsendere (`AcceptMessagesOnlyFromSendersOrMembers`) godtar bare objekter fra egen katalog: postbokser, e-postbrukere, e-postkontakter og grupper. En ren SMTP-adresse kan ikke legges inn der. Opprett derfor en e-postkontakt for hver ekstern avsender. For at kontakten ikke skal vises i den globale adresselisten, skjuler du den:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Name` | Kontaktens visningsnavn i Active Directory. Et ensartet prefiks som «Ekstern» gjør senere vedlikehold enklere. |
| `ExternalEmailAddress` | Den eksterne adressen avsenderen bruker. |
| `OrganizationalUnit` | Mål-OU i Active Directory. Uten angivelse havner kontakten i standardbeholderen `Users`. |
| `-HiddenFromAddressListsEnabled $true` | Skjuler kontakten fra den globale adresselisten og alle adresselister. |

</details>

Deretter legger du inn kontakten som tillatt avsender. Hashtable-en med `Add` utvider den eksisterende listen i stedet for å overskrive den:

```powershell
$erlaubt = @{ Add = "Extern Max Muster", "Extern Anna Beispiel" }
Set-DistributionGroup -Identity "Projektteam" -AcceptMessagesOnlyFromSendersOrMembers $erlaubt
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-AcceptMessagesOnlyFromSendersOrMembers` | Liste over avsendere som har lov til å nå gruppen. Enkelte avsendere og grupper er mulig; for grupper kan alle medlemmer sende. |
| `@{ Add = ... }` | Legger til oppføringer uten å fjerne eksisterende. Med `@{ Remove = ... }` fjernes oppføringer; en verdi uten hashtable erstatter hele listen. |

</details>

Så snart denne listen inneholder minst én oppføring, avviser Exchange **alle andre avsendere**, også interne. Hvis ansatte fortsatt skal kunne skrive til listen, legger du i tillegg inn en passende gruppe, for eksempel distribusjonsgruppen selv (da kan alle medlemmer sende) eller en gruppe med alle ansatte. Avviste avsendere mottar en leveringsstatusrapport med status `5.7.1`.

Slik kontrollerer du gjeldende status:

```powershell
Get-DistributionGroup -Identity "Projektteam" |
    Format-List Name, RequireSenderAuthenticationEnabled,
        AcceptMessagesOnlyFrom, AcceptMessagesOnlyFromDLMembers
```

`AcceptMessagesOnlyFromSendersOrMembers` er en samleparameter: Exchange fordeler oppføringene på egenskapene `AcceptMessagesOnlyFrom` (enkeltavsendere) og `AcceptMessagesOnlyFromDLMembers` (grupper). Derfor viser spørringen disse to egenskapene. Den samme innstillingen finner du i Exchange Admin Center under gruppens egenskaper, i leveringsadministrasjon (Delivery management).

Denne varianten passer når det bare skal legges til noen få eksterne avsendere og listen over tillatte avsendere skal være synlig på selve gruppeobjektet. Ulempen er vedlikeholdsarbeidet: Hver nye avsender trenger en kontakt i Active Directory.

## Variant 2: Transportregel uten kontakter

Hvis alle fortsatt skal kunne skrive til listen internt, mens bare eksterne avsendere begrenses, er en transportregel enklere. Den trenger ingen kontakter og kan også brukes på hele domener. `RequireSenderAuthenticationEnabled` må også her være satt til `$false`.

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `Name` | Regelens navn i listen over transportregler. |
| `SentTo` | Betingelse: Meldingen er adressert til denne gruppen. |
| `FromScope NotInOrganization` | Betingelse: Avsenderen tilhører ikke egen organisasjon. Interne meldinger påvirkes dermed ikke. |
| `ExceptIfFromAddressMatchesPatterns` | Unntak: Avsenderadresser som samsvarer med ett av de regulære uttrykkene. `^…$` forankrer én enkelt adresse, `@domain$` tillater et helt domene. Escape punktum med `\.`. |
| `RejectMessageReasonText` | Tekst i leveringsstatusrapporten til den avviste avsenderen. |
| `RejectMessageEnhancedStatusCode` | Utvidet statuskode i leveringsstatusrapporten; `5.7.1` angir manglende tillatelse. |

</details>

Uten forankringen med `^` og `$` ville et mønster som `max\.muster@partner\.example` også passet på `max.muster@partner.example.net`. Nye avsendere legger du til med `Set-TransportRule` og den komplette, utvidede listen over mønstre; parameteren erstatter den eksisterende listen.

## Grensen for begge variantene: Avsenderadressen kan forfalskes

Begge variantene kontrollerer bare avsenderadressen. Exchange fastslår ikke om meldingen faktisk kommer fra den godkjente partneren. Den som kjenner adressen til en godkjent avsender, kan forfalske den og dermed nå listen. For en liste med interne deltakere er dette en inngangsport for phishing som i tillegg gir inntrykk av å komme fra en kjent adresse.

SPF, DKIM og DMARC lukker dette hullet. Hvis partnerens domene har en DMARC-policy med `p=reject` eller `p=quarantine` og e-postgatewayen din håndhever denne policyen, når forfalskede meldinger fra dette domenet aldri Exchange Server. For domener uten DMARC-policy eller med `p=none` mangler denne beskyttelsen.

Hvis gatewayen din skriver kontrollresultatet i headeren `Authentication-Results`, kan en andre transportregel i tillegg avvise godkjente avsendere dersom DMARC ikke er bestått:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `FromAddressMatchesPatterns` | Betingelse: Regelen gjelder bare for de godkjente avsenderne, med de samme mønstrene som i den første regelen. |
| `ExceptIfHeaderMatchesMessageHeader` | Navnet på headeren som unntaket kontrollerer. |
| `ExceptIfHeaderMatchesPatterns` | Unntak: Hvis headeren inneholder `dmarc=pass`, leveres meldingen. Alle øvrige meldinger fra de godkjente avsenderne avvises. |

</details>

Denne regelen fungerer bare pålitelig dersom gatewayen fjerner eller overskriver innkommende `Authentication-Results`-headere fra internett. Ellers kan en avsender selv legge ved headeren med `dmarc=pass`. Hvilke headere gatewayen din setter, og om den fjerner fremmede headere, er beskrevet i dokumentasjonen; Exchange Server evaluerer ikke DMARC selv.

## Hvilken variant passer

Leveringsbegrensning med kontakter passer for lister med en fast, liten krets av eksterne avsendere og begrenser ved behov også interne avsendere. Transportregelen passer når alle internt skal kunne sende, hele partnerdomener skal godkjennes eller det ikke skal opprettes kontakter i Active Directory. Uavhengig av variant bør du kontrollere DMARC-policyen for hvert godkjente domene og dokumentere hvem som vedlikeholder listen over tillatte avsendere.

## Kilder

1.  [Microsoft Learn: Set-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-distributiongroup?view=exchange-ps): Parameterne AcceptMessagesOnlyFromSendersOrMembers og RequireSenderAuthenticationEnabled, tillatte objekttyper.

2.  [Microsoft Learn: Recipient resolution in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/recipient-resolution): Hvordan Categorizer evaluerer leveringsbegrensninger og avsenderautentisering.

3.  [Microsoft Learn: New-MailContact](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailcontact?view=exchange-ps): Oppretting av e-postkontakter for eksterne adresser.

4.  [Microsoft Learn: New-TransportRule](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-transportrule?view=exchange-ps): Betingelser, unntak og handlinger for transportregler, inkludert FromScope og HeaderMatchesPatterns.

5.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Spesifikasjon av DMARC-policyene p=none, quarantine og reject.

6.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Oppbygningen av headeren Authentication-Results og merknaden om at mottakende systemer bør fjerne fremmede instanser.
