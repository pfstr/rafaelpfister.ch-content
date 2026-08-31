---
title: "Analysera e-posthuvuden utan att ladda upp e-postmeddelandet: lokalt i webbläsaren i stället för i ett webbverktyg"
navTitle: "Analysera huvuden lokalt"
description: "E-posthuvuden innehåller interna värdnamn, IP-adresser och personuppgifter. Den som klistrar in dem i ett onlineverktyg överför denna information till en extern server. Varför analysen inte behöver någon server och vad ett verktyg som körs lokalt i webbläsaren kan göra."
date: "2026-08-26"
kategorie: "SMTP & Mailflow"
timeToRead: "7 min läsning"
themen:
  - smtp-mailflow
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "mail-auth"
  - "troubleshooting"
related:
  - microsoft-365-compauth-reason-codes
  - exchange-hybrid-header-intern-extern
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "analysera-e-posthuvuden-utan-att-ladda-upp-e-postmeddelandet-lokalt-i-webblasaren-i-stallet-for"
translationId: "article-cad792e705cee24e"
translationOf: e-mail-header-analysieren-ohne-upload
url: https://rafaelpfister.ch/sv/blog/analysera-e-posthuvuden-utan-att-ladda-upp-e-postmeddelandet-lokalt-i-webblasaren-i-stallet-for
translationSourceHash: 11c4e7d120ea34ca557f0136b93120e5e8e9d72dc7350fd2df7880b23ff46649
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:17:01.156Z
translationReview: automatic
---

# Analysera e-posthuvuden utan att ladda upp e-postmeddelandet: lokalt i webbläsaren i stället för i ett webbverktyg

Det vanliga sättet att analysera ett e-posthuvud ser ut så här: kopiera huvudet från e-postklienten, klistra in det i ett onlineverktyg och låt det analyseras. Det är praktiskt, men då skickas hela huvudet till verktygsoperatörens server. Vad man exakt överför på det sättet är få medvetna om.

## Vad som faktiskt står i ett huvud

Ett fullständigt huvud från ett e-postmeddelande i en företagsmiljö innehåller vanligtvis:

- **Interna värdnamn och IP-adresser:** Varje `Received`-rad dokumenterar en server i leveransvägen, inklusive interna Exchange-servrar, gateways och lastbalanserare med FQDN och ofta privat IP-adress. Tillsammans ger de en skiss över e-postinfrastrukturen.
- **Personuppgifter:** Avsändar- och mottagaradresser, visningsnamn, ämnesrad, Message-ID:n och, beroende på klient, IP-adressen till den ursprungliga avsändaren.
- **Programvara och versioner:** Received-rader och produktspecifika huvuden anger de använda produkterna, ibland med versionsnummer.
- **Intern organisationsbedömning:** I Microsoft 365 exempelvis den kompletta spam- och autentiseringsbedömningen, tenant-identifierare och den interna klassificeringen av meddelandet.

För en angripare är detta användbart material för förberedelser, och ur dataskyddssynpunkt är det personuppgifter: avsändare, mottagare och ämnesrad för ett specifikt meddelande. Enligt den reviderade dataskyddslagen är behandling via ett utländskt onlineverktyg fortfarande ett utlämnande till tredje part, i tveksamma fall till utlandet. För ett huvud från en kunds supportärende blir frågan ännu känsligare: att klistra in kundens data i ett externt webbverktyg är svårt att motivera utan rättslig grund eller samtycke.

## Analysen behöver ingen server

Den avgörande punkten: ett huvud är ren text och analysen är ren parsning. Att ordna Received-kedjan kronologiskt, beräkna tidsskillnader, avkoda `Authentication-Results`, jämföra domäner: inget av detta kräver någon serverkomponent. Allt körs i JavaScript i webbläsaren utan att huvudet lämnar enheten.

Ett verktyg som är byggt på detta sätt skiljer sig ur dataskyddssynpunkt i grunden från en klassisk onlineanalysator: ingen överföring, ingen lagring hos operatören, inga loggfiler med externa huvuden. Analysen av ett kundhuvud stannar därmed på samma nivå som att öppna filen i en lokal textredigerare, men är mer lättläst.

## Vad ett lokalt verktyg kan göra

[Mail-Header-Analyzer](/tools/header-analyzer) på denna webbplats är byggd enligt denna princip. Det inklistrade huvudet analyseras uteslutande lokalt i webbläsaren. Funktionaliteten visar att inget går förlorat:

- **Leveransväg med transittider:** `Received`-kedjan ordnas kronologiskt, uppehållstiden per station beräknas och den längsta delen markeras. På så sätt syns var en långsam leverans faktiskt fastnade. Klockavvikelser mellan servrar identifieras och redovisas.
- **Transportkryptering per hopp:** TLS-version och chiffer läses från Received-raderna där den mottagande servern loggar dem; Microsoft, Postfix och Exim använder olika format.
- **Autentisering:** SPF-, DKIM- och DMARC-resultat från `Authentication-Results` (RFC 8601), inklusive detaljer som `header.d`, `smtp.mailfrom` och Microsofts `compauth` med Reason Code.
- **DMARC-alignment:** From-domän, Envelope-From och DKIM-domän visas bredvid varandra och bedöms enligt strict och relaxed alignment.
- **ARC- och DKIM-integritet:** Egna spår i flödesgrafiken visar varifrån till var DKIM-hashen var intakt och från vilken station ARC-kedjan bevarar verifieringsresultaten.
- **Microsoft-miljöer:** Spamfilterfälten (`X-Forefront-Antispam-Report`, SCL, CAT) avkodas, tenant-övergångar och hybridklassificeringen i leveransvägen markeras.

En begränsning gäller för alla huvudverktyg, lokala eller inte: de visar den dokumenterade bedömningen från mottagarservern, inte någon egen omverifiering. Huruvida en SPF-post fortfarande ser likadan ut i dag som vid mottagandet kan inte besvaras av huvudet.

## Bedömning av övriga verktyg

Även vissa andra leverantörer analyserar numera på klientsidan; en titt på integritetspolicyn och webbläsarens nätverkskonsol klargör om ingen begäran med huvudets innehåll faktiskt skickas när det klistras in. För klassiska serverbaserade analysatorer gäller den enkla regeln: klistra inte in huvuden från produktionsmiljöer eller från tredje part, utan högst anonymiserade exempel.

För regelbundna analyser av incident- eller supporthuvuden är ett lokalt körande verktyg därför det självklara valet: frågan om var data har hamnat uppstår inte.

## Källor

1.  [RFC 8601: Message Header Field for Indicating Message Authentication Status](https://datatracker.ietf.org/doc/html/rfc8601): Standard för Authentication-Results-huvudfältet, som ligger till grund för autentiseringsanalysen.

2.  [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321): Definition av Received-raderna (Trace Information), som gör det möjligt att rekonstruera leveransväg och transittider.

3.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Referens för de Microsoft 365-specifika huvudfälten som en analysator avkodar.
