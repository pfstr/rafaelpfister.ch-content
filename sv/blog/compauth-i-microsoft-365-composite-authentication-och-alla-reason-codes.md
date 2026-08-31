---
title: "compauth i Microsoft 365: Composite Authentication och alla Reason Codes"
navTitle: "compauth-koder"
description: "Microsoft 365 kompletterar SPF, DKIM och DMARC med en egen bedömning: compauth. Vad Composite Authentication kontrollerar, vad pass, softpass, fail och none betyder samt vilken orsak som ligger bakom varje Reason Code, från 000 till 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min. läsning"
themen:
  - microsoft-365-exchange
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - exchange-hybrid-header-intern-extern
  - dns-records-e-mail-stolpersteine
slug: "compauth-i-microsoft-365-composite-authentication-och-alla-reason-codes"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/sv/blog/compauth-i-microsoft-365-composite-authentication-och-alla-reason-codes
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:20:11.752Z
translationReview: automatic
---

# compauth i Microsoft 365: Composite Authentication och alla Reason Codes

I `Authentication-Results`-huvudet för ett e-postmeddelande som tagits emot i Microsoft 365 finns, utöver standardresultaten för SPF, DKIM och DMARC, ett Microsoft-eget fält:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` står för Composite Authentication: Microsoft 365 kombinerar resultaten från SPF, DKIM och DMARC med ytterligare signaler från meddelandet till en samlad bedömning av om den synliga From-adressen är trovärdig. Bedömningen utgår från From-domänen, alltså den adress som mottagare ser i e-postklienten. Därmed täpper Microsoft till luckan som uppstår när en avsändardomän inte har publicerat några eller endast ofullständiga autentiseringsposter: Även utan en DMARC-policy kontrolleras implicit om e-postmeddelandet passar den påstådda domänen.

## De fyra resultaten

- `compauth=pass`: Meddelandet har klarat den explicita (DMARC) eller implicita autentiseringen.
- `compauth=softpass`: Den implicita kontrollen klarades med lägre säkerhet.
- `compauth=fail`: Meddelandet underkändes av den explicita eller implicita kontrollen.
- `compauth=none`: Ingen Composite-kontroll genomfördes eller så hoppades den över.

Ett `compauth=fail` leder inte automatiskt till karantän eller skräppostmappen. Det är en insignal till filterbeslutet; avgörande för den faktiska hanteringen är `CAT` och ytterligare fält i `X-Forefront-Antispam-Report`. Omvänt gäller: Den som vill veta varför compauth fattade sitt beslut behöver `reason`-koden direkt efter resultatet.

## Översikt över Reason Codes

Den tresiffriga koden anger regeln som ledde till resultatet. Den första siffran grupperar: 0xx och 6xx är underkännanden, 1xx och 7xx är godkända kontroller, 2xx är softpass, medan 3xx, 4xx och 9xx betyder att ingen kontroll utfördes eller att den hoppades över.

| Kod | Betydelse |
|---|---|
| `000` | Explicit underkänd: DMARC-fel med en policy `p=quarantine` eller `p=reject`. |
| `001` | Implicit underkänd: Domänen publicerar inga autentiseringsposter eller endast svaga sådana (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | Organisationen har uttryckligen förbjudit detta avsändar-/domänpar att skicka förfalskade e-postmeddelanden (manuellt underhållen post). |
| `010` | DMARC-fel med `p=reject`/`p=quarantine`, och den sändande domänen är en egen Accepted Domain (spoofing av den egna organisationen). |
| `100` | SPF eller DKIM godkändes, och MAIL FROM- samt From-domänen är aligned. |
| `101` | Meddelandet är DKIM-signerat av From-domänen. |
| `102` | MAIL FROM- och From-domänen är aligned, SPF godkändes. |
| `103` / `104` | From-domänen matchar PTR-posten (omvänd uppslagning) för den levererande IP-adressen. |
| `108` | DKIM-fel på grund av en ändring i meddelandekroppen vid tidigare legitima led, exempelvis i den egna OnPrem-miljön. |
| `109` | Domänen har ingen DMARC-post, men kontrollen skulle ha godkänts. |
| `111` | Trots DMARC-temp- eller permerror är SPF- eller DKIM-domänen aligned med From-domänen. |
| `112` | En DNS-timeout förhindrade hämtningen av DMARC-posten. |
| `115` | E-postmeddelandet kommer från en Microsoft 365-organisation där From-domänen är konfigurerad som Accepted Domain. |
| `116` | MX-posten för From-domänen matchar PTR-posten för den levererande IP-adressen. |
| `130` | En ARC-sealer som konfigurerats som betrodd har åsidosatt DMARC-felet. |
| `201` / `202` | Softpass: From-domänen matchar PTR-posten respektive dess subnät. |
| `3xx` / `4xx` / `9xx` | Ingen Composite-kontroll genomfördes respektive den hoppades över. |
| `501` / `502` | DMARC tillämpades inte eftersom det rör sig om ett giltigt NDR. |
| `601` | Implicit underkänd: Den sändande domänen är en egen Accepted Domain (spoofing av den egna domänen, vanligt vid Direct Send). |
| `701`–`704` | DMARC tillämpades inte eftersom organisationen bevisligen tar emot legitima e-postmeddelanden från denna infrastruktur. |
| `905` | DMARC tillämpades inte på grund av komplex routning, exempelvis e-post från internet via OnPrem-Exchange eller en tredjepartstjänst före Microsoft 365. |

## De vanligaste fallen i praktiken

**`compauth=fail reason=001`** är standardfallet för domäner utan eller med svag autentisering. Åtgärden ligger hos avsändaren: publicera SPF med `-all`, DKIM-signering och en DMARC-policy. Så länge posterna saknas är levererbarheten beroende av reputationssignaler.

**`compauth=fail reason=601`** förekommer när e-postmeddelanden med den egna domänen som avsändare kommer utifrån, klassiskt vid Direct Send: Multifunktionsenheter, applikationer eller tjänsteleverantörer levererar direkt till MX utan en autentiserad connector. Åtgärden är en korrekt konfigurerad Inbound Connector eller att lägga till källan i den egna SPF-posten.

**`compauth=fail reason=000` eller `010`** betyder att DMARC tillämpades normalt. Om `action=oreject` står bredvid har Microsoft 365 översatt avsändarens Reject-policy till leverans i karantän. Här finns inget att åtgärda, såvida inte avsändaren är legitim och dess autentisering är defekt.

**`reason=108`** och **`reason=130`** gäller vidarebefordrings- och gatewayscenarier: En mellanliggande station har ändrat e-postmeddelandet eller en betrodd ARC-sealer har bevarat de ursprungliga kontrollresultaten. Den som driver en gateway framför Microsoft 365 bör ange dess ARC-sealing som betrodd i antispamkonfigurationen, annars fastnar legitima e-postmeddelanden på DMARC.

## Läsa compauth i huvudet

I praktiken står `compauth` sällan ensamt: Först samspelet med de enskilda SPF-, DKIM- och DMARC-resultaten, aligneringen av de berörda domänerna och `Received`-kedjan ger hela bilden. [Mail Header Analyzer](/tools/header-analyzer) på denna webbplats avkodar `compauth` med Reason Code direkt i webbläsaren och visar de tillhörande domänerna (From, Envelope-From, `d=`) sida vid sida för aligneringsbedömningen; det inklistrade huvudet lämnar inte webbläsaren.

## Källor

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Officiell referens för Authentication-Results-fälten och den fullständiga tabellen över compauth Reason Codes.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Tillvägagångssätt vid autentiseringsfel ur ett SecOps-perspektiv.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Konfiguration av betrodda ARC-sealers för gateway- och vidarebefordringsscenarier (Reason Code 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Avgränsning mellan compauth-signalen och det faktiska filterbeslutet.
