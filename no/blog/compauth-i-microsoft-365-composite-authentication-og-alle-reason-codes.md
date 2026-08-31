---
title: "compauth i Microsoft 365: Composite Authentication og alle Reason Codes"
navTitle: "compauth-koder"
description: "Microsoft 365 supplerer SPF, DKIM og DMARC med sin egen vurdering: compauth. Hva Composite Authentication kontrollerer, hva pass, softpass, fail og none betyr, og hvilken årsak som ligger bak hver Reason Code, fra 000 til 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min lesetid"
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
slug: "compauth-i-microsoft-365-composite-authentication-og-alle-reason-codes"
translationId: "article-a9dceac9ee095bbd"
translationOf: microsoft-365-compauth-reason-codes
url: https://rafaelpfister.ch/no/blog/compauth-i-microsoft-365-composite-authentication-og-alle-reason-codes
translationSourceHash: a37557eaef3ea6605e72281d81c56154d6062ae726ef646baa906c2d7d9927a4
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:22:51.765Z
translationReview: automatic
---

# compauth i Microsoft 365: Composite Authentication og alle Reason Codes

I `Authentication-Results`-headeren til en e-post mottatt i Microsoft 365 står det, ved siden av standardresultatene for SPF, DKIM og DMARC, et Microsoft-eget felt:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` står for Composite Authentication: Microsoft 365 kombinerer resultatene fra SPF, DKIM og DMARC med ytterligere signaler i meldingen til en samlet vurdering av om den synlige From-adressen er troverdig. Vurderingen baseres på From-domenet, altså adressen mottakerne ser i e-postklienten. Dermed lukker Microsoft gapet som oppstår når et avsenderdomene ikke har publisert noen eller ufullstendige autentiseringsposter: Selv uten en DMARC-policy kontrolleres det implisitt om e-posten passer til det påståtte domenet.

## De fire resultatene

- `compauth=pass`: Meldingen besto den eksplisitte (DMARC) eller implisitte autentiseringen.
- `compauth=softpass`: Den implisitte kontrollen ble bestått med lavere sikkerhet.
- `compauth=fail`: Meldingen strøk på den eksplisitte eller implisitte kontrollen.
- `compauth=none`: Ingen Composite-kontroll ble utført, eller den ble hoppet over.

En `compauth=fail` fører ikke automatisk til karantene eller søppelpostmappen. Det er et inngangssignal for filteravgjørelsen; avgjørende for den faktiske behandlingen er `CAT` og andre felt i `X-Forefront-Antispam-Report`. Omvendt gjelder følgende: Vil du vite hvorfor compauth tok denne avgjørelsen, trenger du `reason`-koden rett etter resultatet.

## Oversikt over Reason Codes

Den tresifrede koden angir regelen som førte til resultatet. Det første sifferet grupperer: 0xx og 6xx er feil, 1xx og 7xx er beståtte kontroller, 2xx er softpass, mens 3xx, 4xx og 9xx betyr at ingen kontroll ble utført eller at den ble hoppet over.

| Kode | Betydning |
|---|---|
| `000` | Eksplisitt mislykket: DMARC-fail med policy `p=quarantine` eller `p=reject`. |
| `001` | Implisitt mislykket: Domenet publiserer ingen autentiseringsposter eller bare svake poster (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | Organisasjonen har eksplisitt forbudt dette avsender-/domeneparet å sende spoofede e-poster (manuelt vedlikeholdt oppføring). |
| `010` | DMARC-fail med `p=reject`/`p=quarantine`, og det sendende domenet er et eget Accepted Domain (spoofing av egen organisasjon). |
| `100` | SPF eller DKIM bestått, MAIL FROM- og From-domenet er aligned. |
| `101` | Meldingen er DKIM-signert av From-domenet. |
| `102` | MAIL FROM- og From-domenet er aligned, SPF bestått. |
| `103` / `104` | From-domenet samsvarer med PTR-posten (reverse lookup) til den innleverende IP-adressen. |
| `108` | DKIM-fail på grunn av en endring i meldingskroppen på tidligere legitime ledd, for eksempel i eget OnPrem-miljø. |
| `109` | Domenet har ingen DMARC-post, men kontrollen ville ha bestått. |
| `111` | Til tross for DMARC-temp- eller permerror er SPF- eller DKIM-domenet aligned med From-domenet. |
| `112` | En DNS-timeout forhindret henting av DMARC-posten. |
| `115` | E-posten kommer fra en Microsoft 365-organisasjon der From-domenet er konfigurert som Accepted Domain. |
| `116` | MX-posten til From-domenet samsvarer med PTR-posten til den innleverende IP-adressen. |
| `130` | En ARC-sealer konfigurert som klarert overstyrte DMARC-failen. |
| `201` / `202` | Softpass: From-domenet samsvarer med PTR-posten eller dets subnett. |
| `3xx` / `4xx` / `9xx` | Ingen Composite-kontroll utført eller den ble hoppet over. |
| `501` / `502` | DMARC ikke håndhevet fordi det er en gyldig NDR. |
| `601` | Implisitt mislykket: Det sendende domenet er et eget Accepted Domain (selv-spoofing, ofte ved Direct Send). |
| `701`–`704` | DMARC ikke håndhevet fordi organisasjonen beviselig mottar legitime e-poster fra denne infrastrukturen. |
| `905` | DMARC ikke håndhevet på grunn av kompleks ruting, for eksempel internett-e-post via OnPrem-Exchange eller en tredjepartstjeneste før Microsoft 365. |

## De vanligste tilfellene i praksis

**`compauth=fail reason=001`** er standardtilfellet for domener uten eller med svak autentisering. Løsningen ligger hos avsenderen: Publiser SPF med `-all`, DKIM-signering og en DMARC-policy. Så lenge postene mangler, avhenger leveringen av omdømmesignaler.

**`compauth=fail reason=601`** oppstår når e-poster med eget domene som avsender kommer utenfra, klassisk ved Direct Send: Multifunksjonsenheter, applikasjoner eller tjenesteleverandører leverer direkte til MX uten en autentisert connector. Løsningen er en korrekt konfigurert Inbound Connector eller å legge kilden til i egen SPF.

**`compauth=fail reason=000` eller `010`** betyr: DMARC ble håndhevet på vanlig måte. Hvis `action=oreject` står ved siden av, har Microsoft 365 oversatt avsenderens reject-policy til levering i karantene. Her er det ingenting å reparere, med mindre avsenderen er legitim og autentiseringen er defekt.

**`reason=108`** og **`reason=130`** gjelder videresendings- og gateway-scenarioer: Et mellomledd har endret e-posten, eller en klarert ARC-sealer har bevart de opprinnelige kontrollresultatene. Den som drifter en gateway foran Microsoft 365, bør konfigurere ARC-sealingen som klarert i antispam-konfigurasjonen, ellers vil legitime e-poster fortsatt henge på DMARC.

## Lese compauth i headeren

I praksis står `compauth` sjelden alene: Først samspillet med de enkelte SPF-, DKIM- og DMARC-resultatene, alignment av de involverte domenene og `Received`-kjeden gir det komplette bildet. [Mail Header Analyzer](/tools/header-analyzer) på dette nettstedet dekoder `compauth` med Reason Code direkte i nettleseren og viser de tilhørende domenene (From, Envelope-From, `d=`) ved siden av hverandre for alignment-vurderingen; den innsatte headeren forlater ikke nettleseren.

## Kilder

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Offisiell referanse for Authentication-Results-feltene og den fullstendige tabellen over compauth-Reason Codes.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Fremgangsmåte ved autentiseringsfeil fra et SecOps-perspektiv.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Konfigurering av klarerte ARC-sealere for gateway- og videresendingsscenarioer (Reason Code 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Skillet mellom compauth-signalet og den faktiske filteravgjørelsen.
