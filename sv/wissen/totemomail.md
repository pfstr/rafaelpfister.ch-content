---
title: "Totemomail och Kiteworks EPG: e-postgateway och plattformsbyte"
blatt: "totemomail"
description: "Totemomail och dess efterföljare Kiteworks Email Protection Gateway för meddelandeadministratörer: SMTP-inlinearkitektur, policyer, S/MIME, OpenPGP och TLS, WebMail, Java-kryptografistack, nyckelstatus, identitet, licenser, migrering, hög tillgänglighet, övervakning och återställning."
fakten:
  - label: Produktlinje
    wert: totemomail → Kiteworks Email Protection Gateway
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Systemroll
    wert: policystyrd krypteringsgateway för e-post
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Transport
    wert: SMTP i meddelandevägen
    href: https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview
  - label: Metoder
    wert: S/MIME · OpenPGP · TLS
    href: https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf
  - label: Utan mottagarnyckel
    wert: WebMail- eller portaldistribution
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Beslut
    wert: centrala policyer och automatiskt nyckelval
    href: https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf
  - label: Äldre kryptokärna
    wert: Totemo Security Platform · Java-baserad TCM
    href: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500
  - label: EPG-plattform
    wert: Kiteworks Hardened Virtual Appliance
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: EPG-datalagring
    wert: integrerad databas som kunden inte administrerar direkt
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Observerbarhet
    wert: bevis för e-postflöde · Syslog · plattformsgranskning
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: EPG-driftformer
    wert: lokalt · egna molnresurser · Single-Tenant-hosting
    href: https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf
  - label: Ursprung
    wert: totemo AG, Schweiz · grundat 2001 · förvärvat av Kiteworks 2022
    href: https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - totemomail
translationSourceHash: d949453b9a80abb64f76a8ecdee768813548ae2987b18b071cc23d33b0c5df21
translationModel: gpt-5.6-terra
translatedAt: 2026-10-15T08:56:17.763Z
translationReview: automatic
---

# Totemomail och Kiteworks EPG: e-postgateway och plattformsbyte

Totemomail var en produktfamilj från schweiziska totemo AG för central e-postkryptering. Efter förvärvet av Kiteworks integrerades koden enligt tillverkaren i Kiteworks-plattformen och vidareutvecklades som **Email Protection Gateway**, EPG. Namnen betecknar därför inte två helt oberoende produkter, men inte heller samma driftmiljö: en befintlig Totemomail-appliance och ett EPG i Kiteworks Hardened Virtual Appliance har olika gränser för plattform, databas, administration och återställning ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

Funktionellt kvarstår kärnrollen som e-postgateway: den behandlar e-post enligt centrala riktlinjer och använder S/MIME, OpenPGP eller TLS; för mottagare utan lämpliga nycklar finns en WebMail- eller portallösning. Tillverkarens dokumentation beskriver automatisk hantering av nycklar och certifikat samt användning med vanliga e-postklienter utan EPG-specifikt klientinsticksprogram som grundmodell ([Kiteworks: Enterprise packaging and EPG features](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf), [Microsoft Marketplace: totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)).

För meddelandeadministratörer är produktlinjen framför allt en kedja av förtroendegränser. Gatewayen tar emot ett meddelande via SMTP, klassificerar riktning och identiteter, bestämmer en policy, söker efter nycklar eller en reservkanal, ändrar vid behov MIME-innehållet och överlämnar meddelandet till nästa hopp. Med WebMail tillkommer ett separat tillstånd för webb, konto, nycklar och lagring. Påståendet ”meddelandet är krypterat” är ofullständigt utan angiven metod, skyddad sträcka, nyckelinnehavare och tidpunkt ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Förklaringen följer ett meddelande genom EPG-regler, nyckelval och krypteringsmetoder fram till direkt leverans eller WebMail. Bytet från Totemomail till Kiteworks samt drift, diagnos och återställning behandlas därefter separat.

Totemomail och Kiteworks EPG fungerar som inline-gatewayer i SMTP-sökvägen. Ett meddelande tas emot, tilldelas en identitet och policy, skyddas med en lämplig metod och tillhandahålls sedan via SMTP eller portal. Plattformsbytet ändrar implementering och drift, inte denna grunduppgift.

## Arkitekturansats: inline-gateway med central policy

Vid gatewaydrift skickar det interna e-postsystemet utgående meddelanden via [SMTP](/kb/smtp) till Totemomail eller EPG. Inkommande meddelanden når gatewayen direkt via MX eller via ett föregående system för e-postsäkerhet. Gatewayen avslutar SMTP-sessionen, behandlar meddelandet och öppnar en ny session för vidarebefordran. En lyckad TCP- eller TLS-handskakning innebär ännu inte att meddelandet har accepterats; ansvarsöverlämningen sker först med det positiva SMTP-svaret vid slutet av DATA-fasen ([RFC 5321, avsnitt 4.1.1.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

Den offentliga EPG-produktbeskrivningen skiljer mellan flera skyddsvägar. ”Encryption Gateway” omfattar extern gatewaykryptering och -dekryptering. ”Internal Encryption” skyddar meddelanden under transport genom och vid lagring i en molnbaserad e-postplattform. Ytterligare dokumenterade komponenter är Directory Proxy, Application Integration Proxy och en anslutning för RMS respektive Microsoft Information Protection. Vilka av dessa roller som är licensierade och faktiskt aktiverade måste framgå av den egna konfigurationen och avtalet; produktnamnet ensamt styrker dem inte ([Kiteworks: Enterprise packaging and licensing](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

För Microsoft 365 är transportintegrationen fortfarande ett connectorproblem. Exchange Online måste dirigera meddelanden specifikt till gatewayen och acceptera returtrafik baserat på en tillförlitlig motpartsidentitet. En MX framför en tredjepartsgateway ändrar dessutom vilken käll-IP Exchange Online ser vid spam- och spoofingbedömning. Microsoft dokumenterar connectors och Enhanced Filtering för detta; båda är separata från EPG:s kryptopolicy ([Microsoft: Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

## Teknikstack och verifierbara produktgränser

För den historiska Totemomail-plattformen är den kryptografiska kärnan väl offentligt belagd. NIST-CMVP-dokumentationen beskriver **Totemo Cryptographic Module**, TCM, som ett gemensamt Java-bibliotek i Totemo Security Platform, TSP. Valideringen från 2019 nämner Totemo Appliance OS 2.3, JRE 8 och en mjukvarukryptomodul med AES, RSA, ECDSA, SHA-familjen, HMAC och slumptalsgenerator; posten har numera historisk status och är därför inget belägg för en idag tillåten upphandling eller ett konkret installerat algoritmläge ([NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500), [Totemo TCM Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)).

NIST:s säkerhetspolicy avgränsar uttryckligen TCM som en komponent i TSP. Den dokumenterar inte ett fullständigt Totemomail-e-postflöde och inte heller någon generellt giltig implementation av kö, databas eller MTA. Sådana detaljer får därför inte överföras från en främmande Apache James-installation eller en enskild fältobservation till alla Totemomail- eller EPG-versioner. För ett driftokument är exporterad produktkonfiguration, supportmatris, paket- eller tjänsteinventering och tillverkarhandbok för den faktiskt installerade generationen avgörande.

Med EPG flyttas gränsen till Kiteworks Hardened Virtual Appliance. Migrations-FAQ:n beskriver att operativsystem, webbserver och databas ingår i appliance-lösningen och att kundadministratörer inte har direkt åtkomst till operativsystem eller databas. Uppdateringar installeras som en testad appliance-enhet. Detta minskar kundens drift av enskilda komponenter, men gör appliance-processerna för backup, kluster, uppdatering och support till den primära kontrollpunkten ([Kiteworks: FAQ for totemomail customers](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Tillståndsområde | Äldre Totemomail | Kiteworks EPG | Adminbevis |
|---|---|---|---|
| SMTP och routing | Gateway-, MTA- och målkonfiguration för installationen | EPG plus Kiteworks nät- och rollmodell | Testmeddelande, SMTP-svar, kö-/mailflow-ID, nästa hopp |
| Policy | Totemomail-regler och utlösare | integrerad EPG-policy plus plattformskontroller | exporterad policy, prioritet, aktiverat läge, testmatris |
| Nycklar | TSP/TCM, certifikat och OpenPGP-nycklar | migrerat EPG-bestånd av nycklar och certifikat | fingeravtryck, innehavare, utgång, historik, dekrypteringstest |
| Webbleverans | Totemomail WebMail och mottagarkonton | EPG-portal inom Kiteworks-plattformen | registrering, inloggning, bevarande, öppna gammalt meddelande |
| Databas | installationsberoende extern eller lokal databas | integrerad databas utan direkt administratörsåtkomst | backupjobb, återställningsförfarande, tillverkarens supportväg |
| Loggar | produkt-, e-postflödes- och systemloggar | normaliserade plattforms- och EPG-loggar, Syslogexport | korrelerbar meddelande-/händelse-ID, tidsbas, SIEM-ingång |

Jämförelsen följer tillverkarens migrations-FAQ och de publicerade EPG-funktionsbeskrivningarna. Där Totemomail-detaljer är installationsberoende lämnas detta medvetet som en kontrollpunkt i stället för som en påhittad produktegenskap ([Kiteworks: totemomail-to-EPG migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: Top reasons for EPG](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-totemomail.svg?v=20260813" title="Interaktive Infografik: Totemomail- und Kiteworks-EPG-Mailflow, Policy, Schlüssel, WebMail, Plattformmigration, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-totemomail.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## SMTP-e-postflöde och looprisk

En gatewayväg behöver ett tydligt ansvar för varje riktning. Utgående får e-postsystemet inte leverera valfritt direkt eller via EPG om endast gatewayvägen utför den föreskrivna policyn. Inkommande måste det vara fastställt om MX, föregående filter eller EPG autentiserar den ursprungliga avsändaren och vilket hopp som når den interna e-postservern. Flera dubbelriktade connectors utan tydlig mottagar- och domänauktoritet kan få meddelanden att cirkulera mellan system ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3), [Microsoft: Mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

För varje produktionsrutt bör administratörer dokumentera följande värden tillsammans:

- källnät eller certifikatbaserad peer-identitet och förväntat EHLO-namn;
- lokal mottagningsadress, port, STARTTLS- och klientcertifikatkrav;
- ansvar för auktoritativ domän, relä eller split domain;
- policyriktning, användar- och mottagarklassificering;
- nästa hopp, DNS- eller smarthostupplösning och TLS-krav;
- beteende när policy-, nyckel-, katalog- eller portalkomponent inte kan nås;
- maximalt antal hopp, köålder samt ansvar för återförsök och studsmeddelanden.

SMTP-standarden kräver att Received-rader läggs till vid varje vidarebefordran och beskriver loopdetektering med hjälp av spårningsvägen. För diagnos och revision måste gatewayens händelse-ID kunna korreleras med Internet `Message-ID`, envelope, tidsfönster och efterföljande spårning. En synlig ämnesrad är ingen stabil identifierare och kan ändras av policy eller webbleverans ([RFC 5321, avsnitt 4.4](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322#section-3.6.4)).

Efter SMTP-mottagningen beslutar policy engine, baserat på avsändare, mottagare, katalog och tillgängliga nycklar, vilken skyddsväg som är tillåten och möjlig.

## Policy engine och identitetsbeslut

Kiteworks beskriver EPG som ett automatiskt, policystyrt beslutslager. Beroende på licensierad funktionsomfattning kan regler baseras på domäner, användare, platser, innehållsegenskaper eller Microsoft Information Protection-etiketter. Automatisering eliminerar dock inte behovet av att lita på indata: en avsändarheader, en ämnesutlösare eller en etikett som satts utifrån får inte otestat ha samma effekt som en autentiserad intern identitet ([Kiteworks: EPG capability brief](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: totemo and Kiteworks better together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)).

En robust regelmodell skiljer mellan fyra frågor:

1. **Riktning:** internt till externt, externt till internt, internt till internt eller applikation till externt.
2. **Identitet:** SMTP-envelope, synlig `From:`, autentiserad session, connector, katalogobjekt och tenant är inte samma sak.
3. **Åtgärd:** signera, kryptera, omkryptera, portalleverans, vidarebefordra oförändrat, tillfälligt avvisa eller studsa permanent.
4. **Felväg:** saknad nyckel, ogiltig kedja, oåtkomlig katalog, policyfel och oåtkomligt nästa hopp behöver separata resultat.

Före ändringar bör en liten testmatris köras med exakt en avsändare och mottagare per identitetsklass. För varje rad dokumenteras förväntad regel, metod, synlig avsändare, mottagarkanal, loggmarkör och felbeteende. Först när dessa resultat är reproducerbara är policyn inte bara konfigurerad utan även styrkt.

## Skyddsmetoder och omkryptering

[S/MIME](https://datatracker.ietf.org/doc/html/rfc8551) använder CMS och X.509-certifikat. Gatewayen behöver en privat nyckel för signering eller dekryptering och ett lämpligt offentligt certifikat för mottagarkryptering. Certifikatkedja, namn eller e-postadress, Key Usage, giltighet och spärrinformation ingår i kontrollen ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

[OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580) använder en fristående modell för nycklar, signaturer och paket. Fingeravtryck, User ID, kapaciteter, utgång och återkallelse måste inventeras oberoende av X.509. En automatiskt inlärd offentlig nyckel är bara så tillförlitlig som den dokumenterade inlärnings- och bekräftelsevägen. Produktdokumentationen nämner S/MIME och OpenPGP sida vid sida, men gör dem inte till utbytbara trustmodeller ([Kiteworks: Enterprise EPG feature matrix](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)).

[SMTP-TLS](/kb/tls#tls-policy-bei-smtp) skyddar en enskild transportförbindelse. Vid opportunistisk STARTTLS kan leveransen utan ytterligare policy falla tillbaka till klartext. En bindande peer-identitet kan, beroende på miljö, fastställas genom certifikatkontroll, DANE eller MTA-STS; detta skyddar ändå inte meddelandet från de deltagande MTA:erna ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

Vid omkryptering dekrypterar gatewayen en inkommande metod och skapar en ny för nästa avsnitt. Det möjliggör innehållskontroll eller protokollbyte, men skapar ett klartextögonblick inom gatewaygränsen. EPG-funktionskatalogen beskriver uttryckligen scenarier där internt skyddade meddelanden kort dekrypteras vid gatewayen för att kontrolleras eller omvandlas till mottagarens standard ([Kiteworks: Internal Encryption and Encryption Gateway](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). DLP, antivirus, tillfälliga filer, minnesavbildningar, felsökningsloggar och administratörsåtkomst ingår då i kryptogränsen.

Om en direkt användbar mottagarnyckel saknas flyttar portalleveransen klartextåtkomsten till en webbapplikation. Därmed blir webbidentitet, session och lagrat meddelande en del av driftmodellen.

## WebMail och portalleverans

Den officiella migrations-FAQ:n bekräftar att EPG tillhandahåller en funktion som motsvarar Totemomail WebMail för mottagare utan lämplig e-postkryptering. Offentliga operatörsanvisningar för Totemomail visar det typiska användarflödet: avisering, första lösenord eller registreringslänk, lösenordsbyte, inloggning på portalen och val mellan WebMail, S/MIME eller OpenPGP ([Kiteworks: Migration FAQ, fråga 7](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [L-Bank: Säker e-postkommunikation](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)).

Portalleverans är en separat lagrings- och identitetsväg. Aviseringsmeddelandet innehåller inte nödvändigtvis det konfidentiella meddelandet; innehållet kan finnas på portalen eller i ett lösenordsskyddat kuvert. För administratörer är därför minst värdnamn, TLS-certifikat, kontostatus, lösenords- och återställningsförfarande, bevarandetid, meddelandenycklar, leveranshistorik och raderingsprocess relevanta. Den valda varianten måste följa den egna policyn; externa operatörsanvisningar belägger möjliga Totemomail-funktioner, men inga globala standardinställningar.

En portalhälsokontroll får inte stanna vid HTTP 200. Det syntetiska testet måste omfatta registrering eller befintligt testkonto, inloggning, öppning av ett på förhand känt meddelande, hämtning av en bilaga, svar och utloggning. Efter en återställning är det särskilt viktigt att kontrollera om äldre portalmaterial fortfarande kan läsas med sina historiska nycklar.

## Katalog och nycklar med licensräkning

Kiteworks dokumenterar LDAP/AD-integration, roller och centrala policyer på plattformsnivå. För en gateway kan katalogen klassificera avsändare och mottagare, tillhandahålla gruppvillkor och koppla identiteter till certifikat eller nycklar. Så snart en regel beror på en katalogfråga ingår [LDAP](/kb/ldap#bind-sasl-und-tls-vertrauensgrenzen)-endpoint, Search Base, filter, bindkonto, TLS-förtroende, timeout, cache och failover i meddelandevägen ([Kiteworks: security and integration overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511), [RFC 4513](https://datatracker.ietf.org/doc/html/rfc4513)).

Nyckelbeståndet är ingen biprodukt av användarimporten. Privata användar- och domännycklar, offentliga mottagarnycklar, CA- och intermediate-certifikat, förtroendestatus, spärrinformation, Keyserver- respektive MPKI-anslutningar och koppling till historiska meddelanden ska inventeras. En rensning i katalogen får aldrig utan kontroll ta bort en privat nyckel som behövs för gamla meddelanden.

Ett publicerat Kiteworks-dokument om paketering definierade EPG-platser så att en intern användare förbrukar en plats vid sändning eller mottagning av ett krypterat meddelande eller vid sändning av ett signerat meddelande; interna användare som passerade okrypterat och osignerat behandlades där annorlunda. Dokumentet är en daterad avtalsbeskrivning, ingen tidlös licensgaranti ([Kiteworks: Enterprise 8.0 licensing and packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)). För drift måste därför tre värden hållas åtskilda: tekniskt kända identiteter, faktiskt kryptografiskt aktiva identiteter och den avtalsmässigt licensierade metriken för den egna beställningen.

Vid bytet till EPG måste därför inte bara SMTP-rutter utan även regler, identiteter, nycklar, portaltillstånd och bevis övertas eller medvetet byggas upp på nytt.

## Migrering från Totemomail till EPG

Kiteworks beskriver inte migreringen som en in-place-omdöpning. En automatiserad överföring kopierar information från den befintliga Totemomail-databasen till EPG-databasen. Den gamla externa databasen ska avvecklas efter att Totemomail-systemet har tagits ur drift, eftersom den därefter inte längre är synkroniserad. EPG innehåller själv databas, operativsystem och webbtjänster i Hardened Virtual Appliance ([Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

En fullständig migreringsomfattning består minst av:

- domäner, rutter, relä- och connectoridentiteter samt TLS-policyer;
- aktiva regler, prioriteringar, utlösare, undantag och felvägar;
- S/MIME-, OpenPGP-, domän-, TLS- och CA-nycklar inklusive historik;
- WebMail-konton, portalmeddelanden, bevarande, mallar och offentligt namn;
- katalog-, SSO-, SIEM-, Syslog-, DLP-, arkiv- och applikationsintegrationer;
- roller, administratörer samt åtkomst för revision och support;
- licensmetrik, aktivt användarbestånd och avtalsgränser;
- backup, återställning, HA, DNS, övervakning och runbooks.

Acceptanstestet måste utöver nya e-postmeddelanden även kontrollera historiska data. För varje gammal krypteringsmetod öppnas respektive dekrypteras ett meddelande som skapats före migreringen. För varje riktning skickas ett nytt testmeddelande med och utan befintlig mottagarnyckel. Dessutom kontrolleras negativa fall som ogiltigt certifikat, oåtkomligt nästa hopp och otillgänglig katalog. Först därefter kan den gamla gatewayen tas bort från DNS, connectors, allowlistor och övervakning.

## Hög tillgänglighet med backup och återställning

EPG-migrations-FAQ:n nämner integrerad klustring, backup/återställning, katastrofåterställning, hög tillgänglighet och distribution som egenskaper hos Hardened Virtual Appliance. Den säger dock inte att varje enskild EPG-funktion automatiskt får samma RPO och RTO. E-postkö, databas, nycklar, portalinnehåll, revisionslogg, konfiguration och extern objekt- eller fillagring måste tilldelas separat beroende på arkitektur ([Kiteworks: Migration FAQ – Hardened Virtual Appliance](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)).

| Återställningsobjekt | Minimikrav på bevis | Typisk lucka |
|---|---|---|
| Gatewaykonfiguration | exporterad eller säkerhetskopierad målkonfiguration och återställning på ersättningssystem | konfiguration finns, men hemligheter eller certifikat saknas |
| Privata nycklar | fingeravtryckslista före och efter återställning; historisk dekryptering | aktuella nycklar finns, gammal rotation har gått förlorad |
| Policy | aktiverat läge plus positiv och negativ testmatris | exporten motsvarar inte den faktiskt aktiva regeln |
| E-post- och portaltillstånd | i förväg kända kö- och WebMail-testobjekt efter återställning | plattformen startar, men gammalt meddelande går inte att läsa |
| Kluster | failover med SMTP-, HTTPS-, nyckel- och loggtest | VIP växlar, men backendtillstånd eller utgående IP gör det inte |
| Revision och Syslog | extern händelsekopia med tids- och meddelandekorrelation | lokal återställning raderar utredningshistoriken |

För äldre Totemomail är backupförfarandet beroende av version och installation. EPG-uppgiften om integrerad backup får inte retroaktivt överföras till en gammal extern databas eller en separat WebMail-lagring. Tillverkarens supportväg, den installerade dokumentationen och ett isolerat återställningstest är här primärkällorna.

## Övervakning och säkerhetsdrift

Kiteworks bekräftar för EPG export till ett eller flera Syslog-mål samt Splunk-integration. Plattformsdokumentationen beskriver normaliserade loggar och metadata för e-post och andra kommunikationskanaler. För meddelandeadministratörer är den sakliga korrelationen avgörande: plattformsrevision, SMTP-händelse, policybeslut, nyckeloperation, portalåtkomst och efterföljande message trace måste sammanföras via tid, identitet och meddelandeidentifierare ([Kiteworks: Migration FAQ – syslog](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: EPG logging and audit](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)).

En administratörsdashboard bör minst visa mottagningar och avvisningar, köålder per nästa hopp, policyfel, antal och typ av kryptoåtgärder, saknade eller ogiltiga nycklar, certifikatutgång, LDAP-latens, fel vid portalinloggning, disk- och databastillstånd, klusterreplikering, backupålder och Syslog-förlust. En tom kö är bara ett positivt fynd om meddelanden samtidigt tas emot och framgångsrikt överlämnas till nästa hopp.

Syslog via UDP erbjuder ingen garanterad leverans. För skyddsvärda revisionsdata måste TLS-transport, buffring, målautentisering, korrekt tid och ett avbrottslarm planeras ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [RFC 5425](https://datatracker.ietf.org/doc/html/rfc5425)). Loggar kan innehålla avsändare, mottagare, ämnesrad, certifikatinformation eller portalaktiviteter; dataminimering och åtkomstskydd ingår därför i loggningspolicyn.

Historiska säkerhetsmeddelanden visar att även webbgränssnittet är en del av angreppsytan. NVD-poster dokumenterar bland annat tidigare CSRF-, objektåtkomst- och XSS-sårbarheter i Totemomail-webbkomponenter. Detta innebär inget uttalande om en godtycklig, okontrollerad patchnivå, men däremot den bestående skyldigheten att inkludera tillverkaranvisningar, exponering, supportstatus och webbportaltester i patchhanteringen ([NVD: CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563), [NVD: CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918), [NVD: CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)).

Diagnosen följer ett testmeddelande från mottagning via policy- och nyckelbeslut fram till SMTP-utgång eller portalåtkomst.

## Diagnosverktyg

Diagnosen börjar vid den publicerade e-postvägen och följer sedan EPG-händelserna fram till leverans eller WebMail. Plattformsmigrering och licensstatus kontrolleras först när det berörda bearbetningssteget har fastställts.

### DNS och offentliga e-postvägar

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName epg.example.ch -Type A,AAAA
Resolve-DnsName _25._tcp.epg.example.ch -Type TLSA
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A epg.example.ch
dig +short AAAA epg.example.ch
dig +short TLSA _25._tcp.epg.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar offentlig MX, gatewayadress och i förekommande fall DANE-TLSA. Intern och extern resolvervy ska kontrolleras separat. För Microsoft 365 beskriver MX endast det första internethoppet; connectors och transportregler bestämmer den fortsatta vägen.

### TCP-, SMTP- och TLS-åtkomlighet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection epg.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection epg.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://epg.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz epg.example.ch 25
nc -vz epg.example.ch 443
openssl s_client -starttls smtp -connect epg.example.ch:25 \
  -servername epg.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) belägger endast TCP-vägen. Windows-[`curl.exe`](https://curl.se/docs/manpage.html) begär SMTP-STARTTLS; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar funktioner, handskakning, SNI och certifikatkedja. Först en fullständig SMTP-transaktion med positivt DATA-svar belägger mottagning.

### Auktoriserat SMTP-testmeddelande

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://epg.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\epg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server epg.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data epg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) och [`swaks`](https://jetmore.org/john/code/swaks/) skickar här ett kontrollerat testmeddelande. Avsändare, mottagare och mål måste uttryckligen vara auktoriserade för detta. SMTP-slutsvar, gateway-ID, utlöst policy, använd nyckel eller portalväg, nästa hopp och faktisk ankomst ska loggas. Ett `250` på `RCPT TO` innebär ännu inte att meddelandeinnehållet har accepterats.

### WebMail- eller portalväg

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Portaldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
openssl s_client -connect securemail.example.ch:443 \
  -servername securemail.example.ch -showcerts
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest), [`curl`](https://curl.se/docs/manpage.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) kontrollerar HTTP- och TLS-gränssnittet. En lyckad statuskod styrker varken inloggning eller nyckelåtkomst. Ett syntetiskt webbläsartest måste omfatta inloggning, öppning av ett känt meddelande, bilaga och svar.

### Paketväg vid rätt mätpunkt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Totemomail- und EPG-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add EPG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name epg.etl
pktmon stop
pktmon pcapng epg.etl -o epg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w epg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) ser endast trafik vid den valda mätpunkten. En administratörsklient observerar inte automatiskt vägen mellan Exchange Online, Load Balancer och EPG. För analys lämpar sig en auktoriserad värd, port mirror eller cloud flow logs; SMTP före STARTTLS och webbmetadata ska lagras med lämpligt skydd.

## Teknisk historia

Totemo grundades 2001 i Schweiz och utvecklade lösningar för e-postkryptering, Managed File Transfer och mobil kommunikation. NIST-valideringarna från 2012 och 2019 dokumenterar utvecklingen av den Java-baserade Totemo Cryptographic Module och Totemo Security Platform på egna appliance-operativsystem ([Entrust Partner Directory: Totemo](https://www.entrust.com/partners/directory/totemo), [NIST: TCM Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf), [NIST CMVP Certificate 3500](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)).

Kiteworks förvärvade totemo i januari 2022. Tillverkarens dokumentation beskriver integrationen av Totemomail-koden i Kiteworks-plattformen och namnändringen till Email Protection Gateway. EPG kombinerade de befintliga funktionerna för e-postkryptering med Hardened Virtual Appliance, gemensamma roller, normaliserade loggar och ytterligare Kiteworks-kommunikationskanaler ([Kiteworks: acquisition and EPG integration](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf), [Kiteworks: Migration FAQ](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf), [Kiteworks: January 2022 acquisition](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)).

Övergången är tekniskt relevant: den gamla produkten kunde drivas med en extern databas; EPG övertar dess data i en integrerad databas och flyttar underhållet av operativsystem och databas inom appliance-gränsen. Runbooks, övervakning och återställning får därför inte bara ersätta produktbegrepp, utan måste tilldela kontrollpunkter samt lagrade data och konfigurationer på nytt.

## Källor

- [Kiteworks – FAQ for totemomail Customers Updating to EPG](https://www.kiteworks.com/sites/default/files/resources/sp/FAQs%20-%20totemoMAIL%20to%20EPG%20Update%20-%20FINAL%202023-04-03.pdf)
- [Kiteworks – Top 6 Reasons to Add Email Protection Gateway](https://www.kiteworks.com/sites/default/files/resources/kiteworks-brief-top-6-reasons-to-add-email-encryption-with-the-email-protection-gateway-epg-to-kiteworks.pdf)
- [Kiteworks – Enterprise 8.0 Licensing and Packaging](https://www.kiteworks.com/sites/default/files/resources/sp/Packaging%20and%20Licensing%20-%20Kiteworks%20Enterprise%208.0%20-%20FINAL%202023-04-12.pdf)
- [Microsoft Marketplace – totemomail Encryption Gateway](https://marketplace.microsoft.com/en/marketplace/apps/totemo.totemo-azr-tm7?tab=overview)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 8551 – S/MIME 4.0](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Microsoft – Configure mail flow using connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [NIST CMVP – Certificate 3500, Totemo Cryptographic Module](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/3500)
- [NIST – Totemo Cryptographic Module Security Policy 2019](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3500.pdf)
- [Microsoft – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [RFC 5322: Message-ID](https://datatracker.ietf.org/doc/html/rfc5322)
- [Kiteworks – Top 5 Reasons totemo and Kiteworks Are Better Together](https://www.kiteworks.com/sites/default/files/resources/kiteworks-product-brief-top-5-reasons-totemo-kiteworks-better-together.pdf)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [IETF RFC 5280 – X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [L-Bank – Säker e-postkommunikation med Totemomail](https://www.l-bank.info/allgemein/rechtliches/e-secure/sichere-e-mail-kommunikation.html)
- [Kiteworks – Security and Integration Overview](https://www.kiteworks.com/sites/default/files/resources/accellion-brochure-external-file-sharing-security-and-governance-platform_redirected_20191210.pdf)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 4513 – LDAP Authentication Methods and Security Mechanisms](https://datatracker.ietf.org/doc/html/rfc4513)
- [IETF RFC 5424 – Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [IETF RFC 5425 – TLS Transport Mapping for Syslog](https://datatracker.ietf.org/doc/html/rfc5425)
- [NVD – CVE-2018-6563](https://nvd.nist.gov/vuln/detail/CVE-2018-6563)
- [NVD – CVE-2020-7918](https://nvd.nist.gov/vuln/detail/CVE-2020-7918)
- [NVD – CVE-2024-28063](https://nvd.nist.gov/vuln/detail/CVE-2024-28063)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Entrust Partner Directory – Totemo](https://www.entrust.com/partners/directory/totemo)
- [NIST – Totemo Cryptographic Module Security Policy 2012](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp1884.pdf)
- [Kiteworks – The Deal: On the Hunt](https://www.kiteworks.com/sites/default/files/resources/on-the-hunt-kiteworks-secure-content-strategy.pdf)
