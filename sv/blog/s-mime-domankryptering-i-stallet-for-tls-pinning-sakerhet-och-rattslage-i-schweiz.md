---
title: "S/MIME-domänkryptering i stället för TLS-pinning: säkerhet och rättsläge i Schweiz"
navTitle: "Domän i stället för pinning"
description: "Varför S/MIME-domänkryptering skyddar e-posttrafiken mellan två organisationer bättre än TLS-pinning, vad en angripare kan göra med ett uppfångat S/MIME-meddelande, hur DSG, DSV och tystnadsplikter påverkar valet och varför MTA-STS och DANE tar över transportskyddet."
date: "2026-11-16"
kategorie: "E-postkryptering"
timeToRead: "10 min. lästid"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "tls"
  - "dns"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2099-12-31"
slug: "s-mime-domankryptering-i-stallet-for-tls-pinning-sakerhet-och-rattslage-i-schweiz"
translationId: "article-92db7a4587660dec"
translationOf: domain-verschluesselung-statt-tls-pinning
url: https://rafaelpfister.ch/sv/blog/s-mime-domankryptering-i-stallet-for-tls-pinning-sakerhet-och-rattslage-i-schweiz
translationSourceHash: 8eeb8d5552f8680381710c1a052edd3e62b69b34e1af9fd032c4bd481e8d65a1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T16:00:12.496Z
translationReview: automatic
---

# S/MIME-domänkryptering i stället för TLS-pinning: säkerhet och rättsläge i Schweiz

Kort slutsats: TLS-pinning skyddar anslutningen mellan två e-postservrar, medan S/MIME-domänkryptering skyddar meddelandets innehåll under hela vägen mellan två organisationer. Om man måste välja mellan dem är domänkryptering det bättre valet ur teknisk och rättslig synvinkel. MTA-STS och DANE tar över det transportskydd som pinning hittills har gett.

I ett aktuellt projekt ska en befintlig e-postanslutning mellan två organisationer, som hittills har säkrats med tvingande TLS och certifikatspinning, ställas om till S/MIME-domänkryptering via en Secure Mail-tjänst. Tjänsten kan inte använda båda samtidigt för samma anslutning: antingen gäller pinning eller domänkryptering. Frågan om säkerheten därmed försämras är berättigad. Svaret är tydligt så snart man ser vad de båda metoderna faktiskt skyddar.

**Fokus på Schweiz:** Den rättsliga bedömningen följer schweizisk rätt, särskilt DSG, DSV och StGB. Där EU-reglerna skiljer sig anges detta i en markerad anmärkning. Artikeln ersätter inte juridisk rådgivning i enskilda fall.

## Vad TLS-pinning skyddar

Utan ytterligare åtgärder är TLS mellan e-postservrar opportunistiskt. Detta ger tre angreppsmöjligheter:

- **Nedgradering:** En angripare i överföringsvägen tar bort STARTTLS-erbjudandet från SMTP-sessionen. Den sändande servern levererar då i klartext.
- **Godtyckligt certifikat:** Sändande e-postservrar kontrollerar normalt inte motpartens certifikat. En server med ett självutfärdat certifikat accepteras.
- **Förfalskad MX:** Den som manipulerar DNS-svaret omdirigerar e-postmeddelandet till en egen server som erbjuder korrekt TLS.

Pinning stänger alla tre luckorna för en viss partner. Den sändande gatewayen tvingar fram TLS och accepterar endast ett fastställt certifikat eller en fastställd nyckel. Skyddet upphör dock med anslutningen. På den mottagande servern, i köer, spam- och virusfilter, journalförings- och arkivsystem finns meddelandet i klartext. Om det passerar flera servrar gäller pinning endast för den sträcka som det har konfigurerats för.

Därtill kommer driften. Pinnen är en manuell regel hos avsändaren. Om mottagaren förnyar sitt certifikat eller ändrar sin routning, exempelvis genom en ny filtertjänst eller en backup-MX, stoppas leveransen tills regeln har uppdaterats. Vid störningar lättas regeln därför ofta tillfälligt, och aktiveras inte alltid på nytt.

## Vad S/MIME-domänkryptering skyddar

Vid domänkryptering krypterar avsändarens gateway meddelandet med den offentliga nyckeln från mottagardomänens domäncertifikat. Först mottagarens gateway dekrypterar det igen. Alla system däremellan ser endast det krypterade innehållet, oavsett hur många servrar meddelandet passerar och var det mellanlagras. Avsändarens gateway signerar dessutom meddelandet med sitt eget domäncertifikat.

| Skydd | TLS-pinning | S/MIME-domänkryptering |
|---|---|---|
| E-posttext och bilagor | endast under överföringen på den pinnade sträckan | under hela vägen, även på mellanservrar och i köer |
| Ämne, avsändare, mottagare, datum | krypterade under överföringen | läsbara, om inget headerskydd är aktivt |
| Kuvert, storlek, tidpunkt | krypterade under överföringen | läsbara |
| Mottagande servers äkthet | ja, en felaktig server avvisas | nej, e-postmeddelandet överlämnas till den server som MX-routningen anger |
| Forward Secrecy | ja, vid ECDHE-nyckelutbyte | nej |
| Integritet och ursprung efter leverans | nej | ja, genom signatur |
| Beroende av routning och infrastruktur | högt, pinnen gäller för en server respektive nyckel | lågt, skyddet är knutet till domäncertifikatet |

Pinning har därmed tre egenskaper som domänkryptering inte erbjuder: det skyddar metadata, förhindrar att en felaktig server tar emot e-postmeddelandet och erbjuder Forward Secrecy. För innehållet är domänkryptering klart överlägsen.

## Vad en angripare kan göra med ett uppfångat S/MIME-meddelande

Anta att en angripare får in en egen e-postserver i överföringsvägen, genom förfalskade MX-poster, BGP-kapning eller ett komprometterat relä.

Följande kan angriparen inte göra:

- **Läsa innehållet:** Text och bilagor är krypterade med mottagardomänens offentliga nyckel. Utan mottagargatewayens privata nyckel förblir de oläsliga.
- **Ändra innehållet obemärkt:** Om meddelandet är signerat och mottagargatewayen kräver en giltig signatur upptäcks varje ändring.
- **Skapa meddelanden i avsändarens namn:** För detta saknar angriparen avsändardomänens privata signeringsnyckel.

Följande kan angriparen göra:

- **Utvärdera metadata:** Avsändare, mottagare, tidpunkt, storlek och i regel ämnesraden är läsbara. Ett ämne som ”Utskrivningsrapport Max Muster” innebär redan ett utlämnande av personuppgifter.
- **Spara meddelanden och dekryptera dem senare:** S/MIME erbjuder ingen Forward Secrecy. Om den privata domännyckeln senare komprometteras blir alla sparade meddelanden läsbara. Detsamma gäller på längre sikt om RSA blir angripbart genom kvantdatorer.
- **Undertrycka eller fördröja meddelanden:** Avsändaren märker inte nödvändigtvis detta. Tillgängligheten påverkas, vilket kan vara relevant vid tidsfrister.
- **Ändra oskyddade headers:** Angriparen kan exempelvis skriva om ämnet eller `Reply-To` för att styra om svar. Det krypterade innehållet förblir oförändrat.
- **Manipulera krypterad text på ett riktat sätt:** För meddelanden som endast är krypterade och inte signerade och som använder AES-CBC kan den krypterade texten ändras så att delar av klartexten kan läcka ut efter dekryptering. Detta är principen bakom EFAIL-attackerna från 2018. Obligatorisk signaturkontroll eller autentiserad kryptering med AES-GCM enligt S/MIME 4.0 förhindrar detta.

Uppfångad klartext vid ren transportkryptering innebär ett fullständigt dataläckage. Ett uppfångat S/MIME-meddelande berör i huvudsak metadata och tillgänglighet.

## Rättslig bedömning

Ingen schweizisk författning föreskriver S/MIME, TLS-pinning eller någon annan specifik metod. Kraven är teknikneutrala och definierar skyddsmål. Valet av metod måste dock kunna mätas mot dessa skyddsmål och dokumenteras.

| Regelverk | Innehåll | Betydelse för valet av metod |
|---|---|---|
| DSG art. 8, DSV art. 1–3 | lämplig datasäkerhet, skyddsmål, transportkontroll | skydd vid utlämnande, bedömt utifrån risk och aktuell tekniknivå |
| DSG art. 61 lit. c | böter vid överträdelse av minimikraven för datasäkerhet | personligt straffansvar för den ansvariga personen |
| DSG art. 9 | behandling genom personuppgiftsbiträden | varje system med åtkomst till klartext ska beaktas |
| DSG art. 16 och 17 | utlämnande till utlandet | behandling av klartext i utländska system |
| DSG art. 24 | anmälan av datasäkerhetsincidenter | risken för de berörda beror på om läckta uppgifter är läsbara |
| StGB art. 321, DSG art. 62, ATSG art. 33 | yrkeshemlighet, yrkesmässig tystnadsplikt, tystnadsplikt inom socialförsäkringarna | innehållet ska skyddas mot varje obehörig kännedom |
| ZPO art. 177 | elektroniska filer som handlingar | signaturen stärker beviset för ursprung och integritet |

### Datasäkerhet enligt DSG och DSV

Art. 8 DSG kräver datasäkerhet som är lämplig i förhållande till risken. DSV konkretiserar skyddsmålen konfidentialitet, tillgänglighet, integritet och spårbarhet och nämner i art. 3 bland annat transportkontroll som åtgärd: personuppgifter får inte obehörigen läsas, kopieras, ändras eller raderas vid utlämnande. För särskilt skyddsvärda personuppgifter enligt art. 5 lit. c DSG, exempelvis hälsouppgifter eller uppgifter om åtgärder inom social hjälp, gäller motsvarande högre krav.

TLS-pinning uppfyller transportkontrollen för en del av överföringen. Meddelandet lagras dock på flera system innan det når sitt mål, och där är det läsbart i klartext. Med domänkryptering skyddas innehållet på hela sträckan mellan de båda organisationerna. Signaturen täcker dessutom skyddsmålet integritet, vilket TLS inte längre kan styrka efter leveransen.

### Personligt straffansvar

Enligt art. 61 lit. c DSG straffas den som uppsåtligen inte uppfyller minimikraven för datasäkerhet enligt art. 8 stycke 3 DSG, på begäran, med böter på upp till CHF 250'000. Böterna riktas mot den ansvariga fysiska personen, inte mot företaget. Innehållskryptering enligt aktuell tekniknivå, dokumenterad i en riskbedömning, är för dessa personer ett starkare bevis än en transportregel för enskilda servrar.

**EU-anmärkning:** GDPR kräver i art. 32 också lämpliga tekniska och organisatoriska åtgärder och nämner uttryckligen kryptering. Böter enligt art. 83 GDPR riktas dock mot företaget och uppgår vid överträdelser av art. 32 till högst 10 miljoner euro eller 2 % av den globala årsomsättningen.

### Personuppgiftsbiträden och utlämnande till utlandet

Varje system längs vägen som behandlar klartext måste bedömas rättsligt: filtertjänster, reläer, molngatewayar och arkiveringstjänster. Om tredje parter driver dessa system handlar det i regel om personuppgiftsbiträdesbehandling enligt art. 9 DSG, med motsvarande avtal och kontrollskyldigheter. Finns systemen utomlands tillkommer reglerna om utlämnande till utlandet enligt art. 16 och 17 DSG.

Med domänkryptering behandlar dessa system endast krypterad text. Kretsen av ställen som kan få åtkomst till innehållet begränsas till de båda gatewayarna. Metadata är dock fortfarande personuppgifter, och de nämnda skyldigheterna gäller fortsatt för dem.

### Anmälningsplikt vid datasäkerhetsincidenter

En datasäkerhetsincident som sannolikt medför en hög risk för de berörda personerna måste enligt art. 24 DSG anmälas till EDÖB. Om meddelanden läcker via en komprometterad mellanserver beror bedömningen på om innehållet är läsbart. Är det starkt krypterat och nycklarna inte har påverkats är risken för de berörda normalt låg, förutsatt att inga känsliga uppgifter finns i ämnesraden. Vid ren transportkryptering har angriparen tillgång till innehållet i klartext.

### Yrkeshemligheter och tystnadsplikter

Om uppgifter omfattas av en yrkeshemlighet enligt art. 321 StGB, exempelvis för läkare, advokater eller deras medhjälpare, eller av en lagstadgad tystnadsplikt såsom art. 33 ATSG inom socialförsäkringsområdet, måste innehållet skyddas mot varje obehörig kännedom. Detta omfattar även operatörer av mellansystem. Art. 62 DSG straffbelägger dessutom överträdelse av yrkesmässig tystnadsplikt för personer som inte omfattas av art. 321 StGB. Endast innehållskryptering uppfyller detta krav.

### Signaturens bevisvärde

Enligt art. 177 ZPO är elektroniska filer handlingar vars bevisvärde domstolen fritt bedömer. En giltig S/MIME-signatur från avsändardomänen visar att meddelandet kommer från denna domän och inte har ändrats efter signeringen. Detta stärker bevisningen vid tidsfrister, utfästelser eller instruktioner. Den ersätter inte en underskrift; formkraven i OR uppfylls endast av en kvalificerad elektronisk signatur enligt ZertES. Med TLS-pinning går det efter leveransen varken att bevisa ursprung eller integritet.

## MTA-STS och DANE som ersättning för TLS-pinning

När pinning tas bort saknas de egenskaper som endast transportskydd erbjuder: tvingande TLS, en autentiserad mottagande server och därmed skydd av metadata mot avlyssning och omdirigering. Det är precis dessa egenskaper som MTA-STS och DANE ger, som en standard som mottagaren publicerar en gång för alla avsändare.

**MTA-STS (RFC 8461):** Mottagaren publicerar under `https://mta-sts.<domain>/.well-known/mta-sts.txt` en policy med tillåtna MX-värdnamn och läget `enforce`. En avsändare som stöder MTA-STS hämtar policyn, sparar den och levererar sedan endast via TLS med giltigt certifikat till en av de angivna MX-servrarna. En förfalskad MX eller en server utan giltigt certifikat för MX-namnet avvisas.

**DANE (RFC 7672):** Mottagaren publicerar en TLSA-post med hashen för sitt certifikat eller sin nyckel i sin DNSSEC-signerade zon. Avsändaren kontrollerar MX-serverns certifikat mot denna post. Tekniskt sett är DANE en pinning som mottagaren själv publicerar i DNS och säkrar med DNSSEC.

| | TLS-pinning | MTA-STS | DANE |
|---|---|---|---|
| Underhålls av | avsändaren, manuellt per partner | mottagaren, en gång för alla avsändare | mottagaren, en gång för alla avsändare |
| Gäller för | en konfigurerad partner | alla avsändare med MTA-STS-stöd | alla avsändare med DANE-stöd |
| Certifikatförnyelse | pinnen måste uppdateras hos avsändaren | utan anpassning, så länge det nya certifikatet är giltigt | utan anpassning med samma nyckel, annars måste en ny TLSA-post publiceras i förväg |
| Förtroendebas | exakt detta certifikat respektive denna nyckel | offentliga certifikatutfärdare | DNSSEC |
| Svaghet | driftarbete, lättnader vid störningar | första hämtningen och utgången cache (Trust on First Use), felutfärdande av en CA | kräver DNSSEC |
| Felrapporter | inga | TLS-RPT (RFC 8460) | TLS-RPT (RFC 8460) |

För dokumentationen enligt art. 8 DSG har MTA-STS och DANE ytterligare en fördel: Via TLS-RPT får mottagaren dagliga rapporter om leveranser skedde krypterat eller misslyckades. Denna återkoppling finns inte vid pinning. Artikeln [DNS-poster för e-post](https://rafaelpfister.ch/blog/dns-records-e-mail-stolpersteine) beskriver hur båda metoderna konfigureras i avsnittet om MTA-STS, TLS-RPT och DANE.

## Vem som måste kontrollera MTA-STS och DANE

Det system som direkt överlämnar meddelandet till mottagarens MX måste alltid kontrollera. Operatören av MX-servern måste publicera posterna.

För en Secure Mail-tjänst ser den utgående vägen vanligtvis ut så här:

```text
Eigener Mailserver -> Secure-Mail-Dienst (verschlüsselt, signiert) -> MX des Partners
```

Den sista sträckan har hittills skyddats av pinning. Här måste tjänsten kontrollera partnerns MTA-STS och DANE; den egna e-postservern har inget inflytande över denna sträcka. Sträckan från den egna e-postservern till tjänsten säkrar ni själva med tvingande TLS och certifikatkontroll på connectorn.

På mottagarsidan beror det på MX-servern. Om MX pekar på tjänsten måste tjänsten tillhandahålla lämpliga certifikat, och ni publicerar MTA-STS- och TLSA-poster som passar dess servrar. För DANE förutsätter detta att tjänsten meddelar nyckelbyten i god tid. Om MX pekar på den egna gatewayen, exempelvis Exchange Online, konfigurerar ni MTA-STS och DANE där. Om båda organisationerna använder samma tjänst kan den leverera meddelandet inom sin egen plattform; då finns ingen offentlig SMTP-sträcka mellan de båda organisationerna.

Om tjänsten varken stöder MTA-STS eller DANE vid sändning skyddas innehållet fortfarande av S/MIME. Skyddet av metadata och autentiseringen av den mottagande servern på den sista sträckan förblir då öppna. Denna kvarstående risk ska ingå i riskbedömningen.

## Kvarstående risker och åtgärder

| Kvarstående risk | Åtgärd |
|---|---|
| Personuppgifter i ämnesraden är läsbara | Regel: inga personuppgifter i ämnesraden, skriftligen överenskommet med partnern |
| saknad Forward Secrecy | lagra den privata domännyckeln så att den inte kan exporteras, helst i ett HSM; logga åtkomst |
| Manipulation av osignerade meddelanden | signaturkrav på båda sidor, AES-GCM om båda gatewayarna stöder det |
| Leverans i klartext vid fel | håll kvar meddelanden när kryptering misslyckas i stället för att leverera dem i klartext |
| Metadata på överföringsvägen | MTA-STS i läget `enforce` och, där DNSSEC finns, DANE; utvärdera TLS-RPT |
| Utgångna certifikat | ömsesidig information minst 30 dagar före utgång, jämför fingeravtryck via en andra kanal |

Beslutet om domänkryptering och de kvarstående riskerna ska ingå i datasäkerhetskonceptet respektive i dokumentationen av behandlingsaktiviteterna.

## Frågor till gatewayleverantören

1.  Kontrollerar tjänsten vid sändning MTA-STS-policyer i läget `enforce` och DANE-poster för mottagardomänen?

2.  Från vilka servrar sker leveransen, och går det att se TLS-RPT-rapporter eller leveransloggar för transportkryptering?

3.  Stöder tjänsten som MX MTA-STS och DANE för inkommande meddelanden, och hur meddelas nyckelbyten?

4.  Kan connectorn från den egna e-postservern till tjänsten drivas med tvingande TLS och certifikatkontroll?

5.  Levereras mellan två kunder hos tjänsten via offentlig SMTP eller inom plattformen?

6.  Vilka algoritmer används för kryptering och signatur, och stöds AES-GCM?

## Källor

1.  [Schweiziska federala lagen om dataskydd (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 8 om datasäkerhet, art. 9 om personuppgiftsbiträdesbehandling, art. 16 och 17 om utlämnande till utlandet, art. 24 om anmälningsplikt, art. 61 och 62 om straffbestämmelser.

2.  [Dataskyddsförordningen (DSV)](https://www.fedlex.admin.ch/eli/cc/2022/568/de): art. 1–3 med principer, skyddsmål och tekniska och organisatoriska åtgärder, bland annat transportkontroll.

3.  [Schweiziska strafflagen (StGB)](https://www.fedlex.admin.ch/eli/cc/54/757_781_799/de): art. 321 om överträdelse av yrkeshemlighet.

4.  [Federala lagen om den allmänna delen av socialförsäkringsrätten (ATSG)](https://www.fedlex.admin.ch/eli/cc/2002/510/de): art. 33 om tystnadsplikt för verkställande organ.

5.  [Civilprocesslagen (ZPO)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 177 om handlingsbegreppet, inklusive elektroniska filer.

6.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): specifikation av policyn, hämtningsreglerna och läget `enforce`.

7.  [RFC 7672: SMTP Security via Opportunistic DANE TLS](https://www.rfc-editor.org/rfc/rfc7672): DANE för SMTP, TLSA-poster och kontrollregler för sändande servrar.

8.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): format och leverans av TLS-RPT-rapporter.

9.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): aktuell S/MIME-specifikation med AuthEnvelopedData och AES-GCM.

10.  [EFAIL](https://efail.de/): beskrivning av angreppen från 2018 mot osignerade S/MIME- och OpenPGP-meddelanden i CBC-läge.

11.  [Dataskyddsförordningen (GDPR)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32016R0679): art. 32 om säkerhet vid behandling, art. 83 om administrativa sanktionsavgifter (EU-anmärkning).
