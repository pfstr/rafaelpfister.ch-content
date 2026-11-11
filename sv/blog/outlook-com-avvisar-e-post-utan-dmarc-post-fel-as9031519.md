---
title: "Outlook.com avvisar e-post utan DMARC-post: fel AS9031519"
navTitle: "Outlook.com: AS9031519"
description: "Sedan den 5 november 2026 avvisar Outlook.com, Hotmail och Live meddelanden från domäner utan DMARC-post med 550 5.7.1 och tillägget AS9031519, även vid små sändningsvolymer. Vad Microsoft dokumenterar, vad operatörer har mätt, hur du kontrollerar din egen domän och vad e-postsäkerhetsindexet för schweiziska kommuner visar."
date: "2026-11-10"
kategorie: "SMTP och e-postflöde"
timeToRead: "8 min läsning"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
hauptthema: "smtp-mailflow"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "dns"
  - "smtp"
  - "troubleshooting"
related:
  - dns-records-e-mail-stolpersteine
  - microsoft-365-compauth-reason-codes
slug: "outlook-com-avvisar-e-post-utan-dmarc-post-fel-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/sv/blog/outlook-com-avvisar-e-post-utan-dmarc-post-fel-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:21:09.572Z
translationReview: automatic
---

# Outlook.com avvisar e-post utan DMARC-post: fel AS9031519

Sedan den 5 november 2026 kommer meddelanden till inkorgar hos Outlook.com, Hotmail, Live och MSN tillbaka med ett nytt fel när avsändardomänen inte har publicerat någon DMARC-post:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Uppgifterna gäller per den 10 november 2026.

## Vad Microsoft dokumenterar

Outlook.coms postmaster-riktlinjer innehåller fortfarande endast regeln från den 5 maj 2025: Domäner som skickar fler än 5000 meddelanden per dag till Outlook.com-konton måste klara SPF och DKIM samt publicera en DMARC-post med minst `p=none`, som är anpassad via SPF eller DKIM. Meddelanden som inte uppfyller detta hamnar först i skräppostmappen och kan senare avvisas med följande fel:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

För alla andra avsändare formulerar samma sida endast en rekommendation. Tillägget `AS9031519` och feltexten `Message lacks SPF/DKIM/DMARC Auth Records` förekommer varken i postmaster-riktlinjerna eller i supportartikeln om 5.7.515.

Microsoft har endast uttalat sig om den aktuella situationen i Microsoft Q&A-forumen. En moderator märkt som ”Microsoft External Staff” skrev den 9 november att det finns en pågående störning i Exchange-tjänsten vid leverans till Outlook.com, Hotmail.com och Live.com. Den är klassad som Service Degradation och Microsoft undersöker orsaken. Inlägget nämner varken incidentnummer, orsak eller någon tidpunkt för åtgärd. Det finns inget blogginlägg, ingen post i postmaster-riktlinjerna och inget offentligt Service Health-meddelande.

## Vad operatörer har mätt

De mest tillförlitliga uppgifterna kommer från e-postleverantörer som har analyserat sina loggar och publicerat dem på sändlistan mailop samt i egna blogginlägg (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Observationerna stämmer överens:

- **Utlösare:** Avsändardomänen saknar DMARC-post. SPF och DKIM var giltiga och anpassade i de dokumenterade fallen. Feltexten nämner visserligen alla tre mekanismerna, men avvisningen beror på den saknade DMARC-posten.
- **Volym:** Även domäner med få meddelanden per vecka påverkas, långt under gränsen på 5000 per dag.
- **Mottagare:** Endast konsumentinkorgarna (outlook.com, hotmail.com, live.com, msn.com och landsvarianter som hotmail.de eller outlook.be). Enligt tillgängliga rapporter avvisar Exchange Online-tenanter som mottagare inte med detta fel.
- **Oregelbundet:** Endast en del av meddelandena avvisas. Att skicka samma meddelande på nytt kan fungera.
- **Åtgärd:** En post med `v=DMARC1; p=none` räckte i de dokumenterade fallen. Efter publicering tog det mellan några minuter och ungefär en och en halv timme innan Outlook.com återigen accepterade meddelanden.

Två analyser begränsar utlösaren ytterligare: Hos one.com avvisades endast domäner vars uppslagning av `_dmarc.<domain>` slutar med `NXDOMAIN`. Domäner där namnet finns men endast saknar en TXT-post (`NOERROR` utan svar) påverkades inte där. MXroute hittade i en logg 367 avvisade avsändardomäner, varav 286 med `NXDOMAIN` på `_dmarc`. Denna avgränsning bygger på data från enskilda operatörer och är inte slutgiltigt klarlagd.

Sedan middagen den 9 november har enskilda berörda rapporterat att avvisningarna har upphört. Microsoft har inte bekräftat detta.

## Särskilt fall: Microsoft 365 som avsändare

Tenanter i Microsoft 365 är som avsändare oproportionerligt ofta berörda. Microsoft 365 skapar ingen DMARC-post för en egen domän, och så länge DKIM inte har aktiverats för domänen signerar Exchange Online med tenantens `onmicrosoft.com`-domän. Rapporten om utebliven leverans kapslar dessutom in avvisningen i en andra statuskod som hänvisar till SPF:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

SPF-posten är i dessa fall vanligtvis korrekt. Det avgörande är det inre felet efter pilen.

## Kontrollera din egen domän

TXT-posten under `_dmarc.<domain>` efterfrågas:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Förklaringar av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `+short` | `dig` visar endast postens innehåll |
| `TXT` respektive `-Type TXT` | Typ av post för frågan |
| `_dmarc.example.com` respektive `-Name` | Efterfrågat namn: etiketten `_dmarc` före avsändardomänen |

</details>

Tom utdata vid `dig +short` eller felet `DNS name does not exist` vid `Resolve-DnsName` betyder att ingen DMARC-post finns. För underdomäner gäller: Om `news.example.com` skickar och själv saknar en post, räknas enligt RFC 7489 posten för organisationsdomänen `example.com`.

[Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) på den här sidan kontrollerar DMARC tillsammans med SPF, DKIM och anpassning i en enda körning.

## Omedelbar åtgärd och vägen vidare

Om posten saknas återställer denna post leveransen till Outlook.com:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` ändrar inget i hanteringen av din e-post hos andra mottagare och kan därför publiceras utan förberedelser. Taggen `rua` krävs inte för acceptans hos Outlook.com. Utan `rua` får du dock inga aggregerade rapporter, och utan dessa rapporter saknas grunden för varje nästa steg. Adressen måste nå en inkorg eller en analystjänst som behandlar XML-rapporterna.

`p=none` skyddar inte domänen mot missbruk. Posten uppfyller endast acceptansvillkoret. För ett effektivt skydd följer dessa steg:

1. Kartlägg alla system som skickar med domänen (e-postservrar, nyhetsbrevstjänst, ERP, webbutik, skannrar).
2. Håll SPF komplett och giltigt för dessa system, med högst tio DNS-uppslagningar.
3. Aktivera DKIM med den egna domänen, i Microsoft 365 per domän i Defender-portalen.
4. Analysera de aggregerade rapporterna under några veckor och korrigera källor som inte är anpassade.
5. Höj policyn till `p=quarantine` och därefter till `p=reject`.

De vanliga felen vid publicering av dessa poster (dubbla SPF-poster, för långa DKIM-nycklar, automatiskt tillagt zonnamn) beskrivs i [Guiden för DNS-administratörer](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine).

## Effekten i Schweiz: 152 kommuner på fem dagar

[Mail-säkerhetsindex Schweiz](https://rafaelpfister.ch/open-data/mail-security-index) kontrollerar varje natt DNS-posterna för 3364 schweiziska organisationer, däribland 2078 kommuner. [Ändringsloggen](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) visar sedan den 6 november en våg av nypublicerade DMARC-poster. Datumet är alltid datumet för den nattliga avstämningen.

| Avstämning | Organisationer som nyligen fått DMARC-post | varav kommuner |
|---|---|---|
| 19 oktober till 5 november | högst 5 per dag | högst 4 per dag |
| 6 november | 4 | 4 |
| 7 november | 22 | 21 |
| 8 november | 1 | 1 |
| 9 november | 2 | 2 |
| 10 november | 140 | 124 |

På fem dagar har 169 organisationer publicerat en DMARC-post för första gången, varav 152 är kommuner. Antalet organisationer utan DMARC i indexet sjönk från 817 till 650. Tidpunkten sammanfaller med avvisningarna hos Outlook.com. Jag har inte frågat kommunerna själva om orsakerna.

Posterna har följande egenskaper:

- **Policy:** 139 av de 152 kommunerna har angett `p=none`, fem `p=quarantine`, tre `p=reject`. För fem består posten endast av `v=DMARC1` utan taggen `p=` och är därför ogiltig.
- **Rapporter:** 114 poster innehåller enbart `v=DMARC1; p=none`. Totalt har 124 av de 152 kommunerna ingen `rua`-adress angiven och får därför inga rapporter.
- **Centrala omläggningar:** Ändringarna sker samlat. Den 10 november tillkom 32 kommuner i Genève, vars e-postservrar alla körs under `acg-geneve.ch`, samt 26 kommuner i Zürich och 19 i Fribourg. Den 7 november var det åtta kommuner i Nidwalden och sju i Ticino. Samlingen efter e-postserver och kanton tyder på att IT-tjänsteleverantörer har satt posterna för alla kommuner de förvaltar i en enda omgång.

För leverans till Outlook.com räcker dessa poster. De skyddar ännu inte kommunernas domäner mot förfalskade avsändare: Utan `rua` saknas data för att senare höja policyn. 537 av de 2078 kommunerna i indexet saknar fortfarande DMARC-post.

## Bedömning

Det är oklart om Microsoft återtar avvisningen eller bekräftar den som regel. För planeringen spelar det liten roll: Google och Yahoo har krävt DMARC av massavsändare sedan februari 2024, Microsoft sedan maj 2025, och Outlook.coms postmaster-riktlinjer rekommenderar det för alla avsändare. En domän utan DMARC-post kunde de senaste dagarna inte längre levereras tillförlitligt hos en av de största e-postleverantörerna. Posten med `p=none` och `rua` publiceras på några minuter och har inga bieffekter för leveransen.

## Källor

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Officiella krav för avsändare; nämner gränsen på 5000 meddelanden per dag, felet 5.7.515 och rekommendationen till alla avsändare.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Supportartikel om det dokumenterade felet för avsändare med hög volym.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Forumtråd med moderatorns svar från den 9 november 2026 om den pågående störningen; foruminlägg, ingen officiell Service Health-rapport.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Analys av en e-postleverantörs utgående loggar från den 5 november 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Rapport från ett webbhotell med över 500 avvisningar den första dagen och åtgärden via `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Tråd på operatörernas sändlista med mätningar från MXroute, Fastmail och one.com, bland annat avgränsningen till `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Reproducerat test med giltig och anpassad SPF och DKIM, utan DMARC-post.

8.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): Tyskspråkig sammanfattning av rapporterna från den 9 november 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Specifikation, bland annat om hur posten fastställs via organisationsdomänen.

10.  [Mail-säkerhetsindex Schweiz: Ändringslogg](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Dagliga ändringar av DNS-posterna för 3364 schweiziska organisationer; grund för siffrorna om kommunerna.
