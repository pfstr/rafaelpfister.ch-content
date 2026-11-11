---
title: "Outlook.com avviser e-post uten DMARC-post: Feil AS9031519"
navTitle: "Outlook.com: AS9031519"
description: "Siden 5. november 2026 har Outlook.com, Hotmail og Live avvist meldinger fra domener uten DMARC-post med 550 5.7.1 og tillegget AS9031519, også ved lavt sendevolum. Hva Microsoft dokumenterer, hva operatører har målt, hvordan du kontrollerer ditt eget domene, og hva e-postsikkerhetsindeksen viser for sveitsiske kommuner."
date: "2026-11-10"
kategorie: "SMTP og e-postflyt"
timeToRead: "8 min. lesetid"
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
slug: "outlook-com-avviser-e-post-uten-dmarc-post-feil-as9031519"
translationId: "article-84656f0bf3da2006"
translationOf: outlook-com-dmarc-ablehnung-as9031519
url: https://rafaelpfister.ch/no/blog/outlook-com-avviser-e-post-uten-dmarc-post-feil-as9031519
translationSourceHash: afec1593a28f2ec6ab14b2af5b064bae9b03ef23d0d091f8b659713eff887222
translationModel: gpt-5.6-terra
translatedAt: 2026-11-11T03:21:36.144Z
translationReview: automatic
---

# Outlook.com avviser e-post uten DMARC-post: Feil AS9031519

Siden 5. november 2026 har meldinger til postkasser hos Outlook.com, Hotmail, Live og MSN kommet tilbake med en ny feil når avsenderdomenet ikke har publisert en DMARC-post:

```text
550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC Auth Records
(AS9031519)
```

Opplysningene gjelder per 10. november 2026.

## Hva Microsoft dokumenterer

Postmaster-retningslinjene for Outlook.com inneholder fortsatt bare regelen fra 5. mai 2025: Domener som sender mer enn 5000 meldinger per dag til Outlook.com-kontoer, må bestå SPF og DKIM og publisere en DMARC-post med minst `p=none`, som er justert via SPF eller DKIM. Meldinger som ikke oppfyller dette, havner først i søppelpostmappen og kan senere avvises med følgende feil:

```text
550; 5.7.515 Access denied, sending domain [SendingDomain] does not meet
the required authentication level.
```

For alle andre avsendere formulerer samme side bare en anbefaling. Tillegget `AS9031519` og feilmeldingen `Message lacks SPF/DKIM/DMARC Auth Records` forekommer verken i Postmaster-retningslinjene eller i støtteartikkelen om 5.7.515.

Microsoft har bare uttalt seg om den aktuelle situasjonen i Microsoft Q&A-forumene. En moderator merket «Microsoft External Staff» skrev 9. november at det pågikk en feil i Exchange-tjenesten ved levering til Outlook.com, Hotmail.com og Live.com. Den var klassifisert som Service Degradation, og Microsoft undersøkte årsaken. Innlegget oppgir verken et hendelsesnummer, en årsak eller en dato for utbedring. Det foreligger ikke noe blogginnlegg, noen oppføring i Postmaster-retningslinjene eller noen offentlig Service Health-melding.

## Hva operatører har målt

De mest pålitelige dataene kommer fra e-postleverandører som har analysert loggene sine og publisert dem på e-postlisten mailop samt i egne blogginnlegg (MailRoute, Mythic Beasts, MXroute, Fastmail, one.com, EasyDMARC). Observasjonene stemmer overens:

- **Utløser:** Avsenderdomenet har ingen DMARC-post. SPF og DKIM var gyldige og justerte i de dokumenterte tilfellene. Feilmeldingen nevner riktignok alle tre metodene, men avvisningen skyldes den manglende DMARC-posten.
- **Volum:** Også domener med få meldinger per uke er berørt, langt under grensen på 5000 per dag.
- **Mottakere:** Bare forbrukerpostkassene (outlook.com, hotmail.com, live.com, msn.com og landsvarianter som hotmail.de eller outlook.be). Exchange Online-tenanter som mottakere avviser ifølge de foreliggende rapportene ikke med denne feilen.
- **Uregelmessig:** Bare en del av meldingene blir avvist. En ny sending av samme melding kan gå gjennom.
- **Utbedring:** En post med `v=DMARC1; p=none` var tilstrekkelig i de dokumenterte tilfellene. Etter publisering tok det mellom noen minutter og rundt halvannen time før Outlook.com godtok igjen.

To analyser avgrenser utløseren ytterligere: Hos one.com ble bare domener avvist der oppslaget av `_dmarc.<domain>` ender med `NXDOMAIN`. Domener der navnet eksisterer, men det ikke finnes noen TXT-post (`NOERROR` uten svar), var der ikke berørt. MXroute fant i én logg 367 avviste avsenderdomener, hvorav 286 med `NXDOMAIN` på `_dmarc`. Denne avgrensningen bygger på data fra enkeltoperatører og er ikke endelig avklart.

Siden middag 9. november har enkelte berørte meldt at avvisningene har opphørt. Microsoft har ikke bekreftet dette.

## Særtilfelle Microsoft 365 som avsender

Tenanter i Microsoft 365 er uforholdsmessig ofte berørt som avsendere. Microsoft 365 oppretter ingen DMARC-post for et eget domene, og så lenge DKIM ikke er aktivert for domenet, signerer Exchange Online med tenantens `onmicrosoft.com`-domene. Leveringsstatusrapporten pakker i tillegg avvisningen inn i en annen statuskode som viser til SPF:

```text
550 5.7.23 The message was rejected because of Sender Policy Framework
violation -> 550 5.7.1 Message rejected: Message lacks SPF/DKIM/DMARC
Auth Records (AS9031519)
```

SPF-posten er i disse tilfellene som regel korrekt. Det er den indre feilen etter pilen som er avgjørende.

## Kontroller ditt eget domene

TXT-posten under `_dmarc.<domain>` slås opp:

```bash
dig +short TXT _dmarc.example.com
```

```powershell
Resolve-DnsName -Type TXT -Name _dmarc.example.com
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `+short` | `dig` viser bare innholdet i posten |
| `TXT` eller `-Type TXT` | Posttype for oppslaget |
| `_dmarc.example.com` eller `-Name` | Navnet som slås opp: etiketten `_dmarc` foran avsenderdomenet |

</details>

Tom utdata ved `dig +short` eller feilen `DNS name does not exist` ved `Resolve-DnsName` betyr at ingen DMARC-post finnes. For underdomener gjelder følgende: Hvis `news.example.com` sender og ikke selv har en post, gjelder posten for organisasjonsdomenet `example.com` i henhold til RFC 7489.

[Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) på denne siden kontrollerer DMARC sammen med SPF, DKIM og justering i én operasjon.

## Umiddelbart tiltak og veien videre

Hvis posten mangler, gjenoppretter denne oppføringen levering til Outlook.com:

```text
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"
```

`p=none` endrer ingenting i behandlingen av e-postene dine hos andre mottakere og kan derfor publiseres uten forarbeid. `rua`-taggen er ikke nødvendig for at Outlook.com skal godta meldinger. Uten `rua` mottar du imidlertid ingen aggregatrapporter, og uten disse rapportene mangler grunnlaget for ethvert videre steg. Adressen må nå en postkasse eller en analysetjeneste som behandler XML-rapportene.

`p=none` beskytter ikke domenet mot misbruk. Posten oppfyller kun vilkåret for godkjenning. For effektiv beskyttelse følger disse trinnene:

1. Kartlegg alle systemer som sender med domenet (e-postserver, nyhetsbrevstjeneste, ERP, nettbutikk, skanner).
2. Hold SPF for disse systemene komplett og gyldig, med maksimalt ti DNS-oppslag.
3. Aktiver DKIM med eget domene, i Microsoft 365 per domene i Defender-portalen.
4. Analyser aggregatrapportene i noen uker og korriger kilder som ikke er justert.
5. Hev policyen til `p=quarantine` og deretter til `p=reject`.

De typiske feilene ved publisering av disse postene (doble SPF-poster, for lange DKIM-nøkler, automatisk tilføyd sonenavn) beskrives i [veiledningen for DNS-administratorer](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine).

## Virkningen i Sveits: 152 kommuner på fem dager

[Mail-sikkerhetsindeksen for Sveits](https://rafaelpfister.ch/open-data/mail-security-index) kontrollerer hver natt DNS-postene for 3364 sveitsiske organisasjoner, deriblant 2078 kommuner. [Endringsloggen](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen) viser siden 6. november en bølge av nyopprettede DMARC-poster. Datoen er alltid datoen for den nattlige sammenligningen.

| Sammenligning | Organisasjoner med ny DMARC-post | hvorav kommuner |
|---|---|---|
| 19. oktober til 5. november | maksimalt 5 per dag | maksimalt 4 per dag |
| 6. november | 4 | 4 |
| 7. november | 22 | 21 |
| 8. november | 1 | 1 |
| 9. november | 2 | 2 |
| 10. november | 140 | 124 |

På fem dager har 169 organisasjoner publisert en DMARC-post for første gang, hvorav 152 er kommuner. Antallet organisasjoner uten DMARC i indeksen sank fra 817 til 650. Tidspunktet sammenfaller med avvisningene hos Outlook.com. Jeg har ikke undersøkt årsakene hos kommunene selv.

Postene har følgende kjennetegn:

- **Policy:** 139 av de 152 kommunene har satt `p=none`, fem `p=quarantine`, tre `p=reject`. Hos fem består posten bare av `v=DMARC1` uten `p=`-tagg og er dermed ugyldig.
- **Rapporter:** 114 poster inneholder utelukkende `v=DMARC1; p=none`. Til sammen har 124 av de 152 kommunene ingen `rua`-adresse oppført og mottar dermed ingen rapporter.
- **Sentrale omlegginger:** Endringene opptrer samlet. Den 10. november kom 32 kommuner i Genève til, med e-postservere som alle kjører under `acg-geneve.ch`, i tillegg til 26 kommuner i Zürich og 19 i Fribourg. Den 7. november var det åtte kommuner i Nidwalden og sju i Ticino. Samlingen etter e-postserver og kanton tyder på at IT-tjenesteleverandører har satt postene for alle kommunene de forvalter i én operasjon.

For levering til Outlook.com er disse postene tilstrekkelige. De beskytter ennå ikke kommunedomenene mot forfalskede avsendere: Uten `rua` mangler dataene for senere å kunne heve policyen. 537 av de 2078 kommunene i indeksen har fortsatt ingen DMARC-post.

## Vurdering

Det er åpent om Microsoft trekker tilbake avvisningen eller bekrefter den som regel. Dette spiller liten rolle for planleggingen: Google og Yahoo har krevd DMARC fra masseavsendere siden februar 2024, Microsoft siden mai 2025, og Postmaster-retningslinjene for Outlook.com anbefaler det for alle avsendere. Et domene uten DMARC-post kunne de siste dagene ikke lenger levere pålitelig til en av de største e-postleverandørene. Posten med `p=none` og `rua` kan publiseres på få minutter og har ingen bivirkninger for leveringen.

## Kilder

1.  [Outlook.com Postmaster: Policies, Practices, and Guidelines](https://substrate.office.com/ip-domain-management-snds/postmaster/Policies): Offisielle krav til avsendere; nevner grensen på 5000 meldinger per dag, feilen 5.7.515 og anbefalingen til alle avsendere.

2.  [Microsoft Support: Fix NDR error 550 5.7.515 in Outlook.com](https://support.microsoft.com/en-us/outlook/fix-ndr-error-550-5-7-515-in-outlook-com): Støtteartikkel om den dokumenterte feilen for avsendere med høyt volum.

3.  [Microsoft Q&A: SPF Validation Error](https://learn.microsoft.com/en-us/answers/questions/6063210/spf-validation-error): Forumtråd med moderatorens svar fra 9. november 2026 om den pågående feilen; foruminnlegg, ingen offisiell Service Health-melding.

4.  [MailRoute: Outlook.com Is Quietly Rejecting Email From Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc): Analyse av utgående logger hos en e-postleverandør fra 5. november 2026.

5.  [Mythic Beasts: Microsoft are rejecting your mail](https://www.mythic-beasts.com/blog/2026/11/09/microsoft-are-rejecting-your-mail/): Rapport fra en hostingleverandør med over 500 avvisninger første dag og utbedring via `p=none`.

6.  [mailop: Microsoft requiring DMARC now or just us?](https://www.mail-archive.com/mailop@mailop.org/msg27569.html): Tråd fra operatørens e-postliste med målingene fra MXroute, Fastmail og one.com, blant annet avgrensningen til `NXDOMAIN`.

7.  [EasyDMARC: Microsoft Rejecting Emails Without DMARC](https://easydmarc.com/blog/microsoft-rejecting-emails-without-dmarc/): Reprodusert test med gyldig og justert SPF og DKIM, uten DMARC-post.

8.  [heise online: Outlook.com: Die Mail ist echt und sie kommt trotzdem nicht an](https://www.heise.de/news/Outlook-com-Die-Mail-ist-echt-und-sie-kommt-trotzdem-nicht-an-11517776.html): Tyskspråklig oppsummering av rapportene fra 9. november 2026.

9.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489): Spesifikasjon, blant annet om fastsetting av posten via organisasjonsdomenet.

10.  [Mail-sikkerhetsindeksen for Sveits: Endringslogg](https://rafaelpfister.ch/open-data/mail-security-index/aenderungen): Daglige endringer i DNS-postene til 3364 sveitsiske organisasjoner; grunnlag for tallene om kommunene.
