---
title: "E-postkryptering: metoder, nycklar och driftmodeller"
blatt: "verschluesselung"
description: "E-postkryptering för meddelandeadministratörer: SMTP-TLS, MTA-STS, DANE och REQUIRETLS, S/MIME och OpenPGP, klient-, gateway- och portalmodeller, nyckel- och certifikatlivscykel, metadata, övervakning, återställning och diagnostik."
fakten:
  - label: Grundfråga
    wert: Vilka data skyddas mellan vilka förtroendegränser?
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Transport
    wert: SMTP STARTTLS skyddar exakt ett hopp
    href: https://datatracker.ietf.org/doc/html/rfc3207
  - label: Transportpolicy
    wert: DANE · MTA-STS · REQUIRETLS
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf
  - label: S/MIME
    wert: MIME + CMS + X.509/PKIX
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: OpenPGP
    wert: Paketformat, nycklar, signaturer och kryptering
    href: https://datatracker.ietf.org/doc/html/rfc9580
  - label: Signatur
    wert: Autenticitet och integritet; ingen konfidentialitet
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Klientmodell
    wert: privata nycklar vid ändpunkten; e-postservern lagrar chiffertext
    href: https://www.nccoe.nist.gov/publication/1800-6/VolC/
  - label: Gatewaymodell
    wert: klartext inom den kontrollerade zonen och vid gatewayen
    href: https://www.seppmail.com/products/
  - label: Portalmodell
    wert: avisering via e-post; innehåll i webblagring med egen identitet
    href: https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm
  - label: Nyckelhantering
    wert: Inventering · skydd · distribution · rotation · återkallelse · återställning
    href: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
  - label: Synliga metadata
    wert: SMTP-kuvert och yttre rubriker förblir synliga beroende på modell
    href: https://datatracker.ietf.org/doc/html/rfc8551
  - label: Återställningsregel
    wert: Spara krypteringsnycklar vid behov; återskapa inte signaturnycklar
    href: https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - e-mail-verschluesselung
  - seppmail
translationSourceHash: 915d86b5c19b97fc5564faf33053144b692f76fd3d6c732fd955f466d25c4983
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T09:08:44.775Z
translationReview: required
---

# E-postkryptering: metoder, nycklar och driftmodeller

Frågan «Är detta e-postmeddelande krypterat?» är tekniskt ofullständig. Ett meddelande kan vara läsbart i klienten, lagrat som chiffertext på e-postservern, dekrypterat på en gateway, skyddat med [TLS](/kb/tls) mellan två MTA:er och åter lagrat okrypterat hos mottagaren. Ett tillförlitligt svar anger därför **skyddsobjekt, ändpunkter, nyckelinnehavare och klartextställen**. NIST behandlar av detta skäl transport-, domän- och innehållsmekanismer som samverkande men separata kontroller ([NIST SP 800-177 Rev. 1](https://www.nist.gov/publications/trustworthy-email-0)).

**Transportkryptering** skyddar en anslutning. SMTP STARTTLS förhandlas om på nytt för varje hopp och avslutas vid respektive MTA. **Innehållskryptering** skyddar en MIME-entitet med S/MIME eller OpenPGP och kan överleva flera transport- och lagringshopp. Om den verkligen fungerar från ände till ände beror inte enbart på formatet: om en företagsgateway dekrypterar ställföreträdande ligger de kryptografiska ändpunkterna vid gatewayarna och inte hos användarklienterna ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

En **digital signatur** är en ytterligare, annorlunda säkerhetsfunktion. Den binder data till en privat signeringsnyckel och möjliggör kontroll av integritet och identitet; den håller inte innehållet hemligt. Omvänt säger ett framgångsrikt dekrypterat meddelande inget om vem som skapade det. I S/MIME skiljs därför CMS `SignedData` och `EnvelopedData` respektive `AuthEnvelopedData` åt. OpenPGP separerar också paket för signaturer, sessionsnycklar och krypterade data ([RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 8551, S/MIME Content Types](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 9580, Packet Syntax](https://datatracker.ietf.org/doc/html/rfc9580)).

Förklaringen följer ett meddelande och frågar vid varje hopp var klartext finns och vem som kontrollerar nycklarna. Transportkryptering, metoder från ände till ände, gatewayar, portaler, metadata, diagnostik och återställning jämförs utifrån detta.

E-postkryptering kan skydda transporthoppet, meddelandeinnehållet eller portalåtkomst. Dessa modeller har olika nyckelinnehavare och olika ställen där klartext förekommer. Artikeln följer därför först skyddsvägen och därefter de enskilda produkterna och metoderna.

## Arkitekturansats: skyddsmodeller och klartextgränser

E-postkryptering skiljer sig framför allt åt genom vilka två punkter som krypterar och dekrypterar. Tabellen börjar därför med den normala meddelandevägen och visar sedan var klartext tekniskt förblir synlig.

| Modell | Kryptografiska ändpunkter | Klartext finns minst hos | Primärt drifttillstånd |
|---|---|---|---|
| opportunistisk SMTP-TLS | två angränsande MTA:er | varje deltagande MTA | certifikat, TLS-funktioner, kö och återförsök |
| tvingad transport-TLS | två MTA:er plus publicerad eller lokal policy | varje deltagande MTA | MTA-STS/DANE/REQUIRETLS, DNS/HTTPS och rapporter |
| klient-till-klient S/MIME/OpenPGP | användarens MUA eller kryptomodul | före kryptering och efter dekryptering vid ändpunkten | privata nycklar, certifikat/nycklar, klientkompatibilitet |
| gateway-till-gateway | organisationernas e-postgatewayar | internt e-postsystem, gatewaybearbetning, målzon | policy, katalog, gateway-nyckellager, kö och mappning |
| webbportal | avsändare/gateway och portalens webbapplikation | gateway/portal, webbläsare efter inloggning | meddelandeobjekt, mottagarkonto, OTP, länk, bevarande |

Ingen av dessa modeller är generellt «starkare». Klient-till-klient minskar klartextställen i infrastrukturen, men försvårar skadlig programvarukontroll, journalföring, eDiscovery och återställning. NCCoE:s praktikexempel låter därför S/MIME-kontrollen utföras i användarklienten med dess Trust Anchors, efter att transport- och domänbaserade kontroller har slutförts vid MTA:n ([NCCoE: DNS-Based Email Security](https://www.nccoe.nist.gov/publication/1800-6/VolC/)). Gatewayar centraliserar policy och nycklar, men blir därmed själva högprivilegierade klartext- och nyckelställen. Portaler når mottagare utan e-postkryptografi, men skapar ett ytterligare förfarande för webb, identitet och lagring. Valet följer dataklassificering, hotmodell, bevarande, rättsliga krav och båda kommunikationssidornas möjligheter – inte produktnamnet.

<iframe class="kb-infographic kb-infographic--compact" style="aspect-ratio: 1400 / 1076; max-width: 720px; margin-inline: auto" src="/images/kb-interaktiv-verschluesselung.svg?v=20260813" title="Interaktive Infografik: E-Mail-Verschlüsselung mit Transport-, Client-, Gateway- und Portalmodell, Klartextgrenzen, Schlüsselzustand, Metadaten und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-verschluesselung.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

## SMTP-TLS: skydd per transporthopp

[SMTP](/kb/smtp) utformades ursprungligen utan obligatorisk konfidentialitet. STARTTLS utökar en befintlig SMTP-session: efter en lyckad TLS-handshake förhandlas funktioner på nytt med EHLO. I opportunistiskt läge kan den sändande MTA:n återgå till klartext om STARTTLS saknas eller ett certifikatfel uppstår. Det förbättrar skyddet mot passiv avlyssning, men ger utan ytterligare policy ingen tillförlitlig garanti mot nedgradering eller en aktiv angripare ([RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

**MTA-STS** publicerar först en DNS-TXT-indikation och hämtar den egentliga policyn via HTTPS. I läget `enforce` måste de angivna MX-namnen valideras med PKIX; fel leder till fördröjd leverans i stället för klartext. **DANE för SMTP** binder certifikat- eller nyckelinformation till DNSSEC-validerade TLSA-poster. **REQUIRETLS** är en SMTP- och meddelandemarkering med vilken en avsändare kräver TLS för fortsatt överföring. Dessa metoder har olika beroenden och räckvidder och får inte sammanfattas i en enda kryssruta «TLS required» ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

SMTP TLS Reporting, **TLS-RPT**, levererar aggregerade rapporter om lyckade och misslyckade TLS-policykontroller. Det bevisar ingen enskild leverans och innehåller inget meddelande; det hjälper till att upptäcka certifikat-, DNS- och policyfel innan köer växer länge. NIST rekommenderar att mottagardomäner signalerar DANE, MTA-STS eller båda, och beskriver de olika DNSSEC-, HTTPS- och PKIX-beroendena ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [NIST SP 800-177 Rev. 1, avsnitt 5.2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)).

SMTP-TLS avslutas vid nästa e-postserver. Om meddelandeinnehållet ska förbli skyddat över flera hopp används meddelandebaserade metoder som S/MIME eller OpenPGP.

## S/MIME: CMS, X.509 och MIME

S/MIME är inte ett enskilt krypteringsbibliotek, utan en kombination av flera standarder. MIME strukturerar meddelandedelar; CMS definierar signerade och krypterade behållare; X.509/PKIX knyter publika nycklar till identiteter och förtroendeankare via certifikat. För varje mottagare krypterar avsändaren den symmetriska innehållsnyckeln med mottagarens publika nyckel eller en överenskommen nyckelhanteringsmetod. För läsning krävs motsvarande privata nyckel ([RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

Administratören måste skilja mellan två nyckelroller. En **signeringsnyckel** ska endast vara tillgänglig för undertecknaren; en central kopia skulle tekniskt göra det möjligt för tredje parter att skapa nya signaturer under dennes identitet. En **dekrypteringsnyckel** kan säkras eller deponeras enligt organisationens policy, så att äldre affärsmeddelanden förblir läsbara efter enhetsförlust, rollbyte eller certifikatförnyelse. RFC 8551 påpekar uttryckligen att en person kan ha flera aktiva nyckelpar för olika syften; NIST skiljer mellan skydd och återställning per nyckeltyp ([RFC 8551, Multiple Key Pairs](https://datatracker.ietf.org/doc/html/rfc8551), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Före sändning behöver avsändaren ett lämpligt och giltigt krypteringscertifikat för **varje** mottagare. Distribution kan ske via en katalog, signerade meddelanden, en certifikatplattform eller manuellt. Distributionsgrupper är särskilt krävande, eftersom expansion, synlighet och certifikattillgänglighet för alla medlemmar måste passa ihop. Microsoft dokumenterar för Exchange attributen `userSMIMECertificate` och `userCertificate` samt certifikatsamlingen för förtroendekontroll ([Microsoft: S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime), [Microsoft: S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)).

## OpenPGP: format är inte en förtroendemodell

RFC 9580 definierar OpenPGP-paket, nycklar, fingeravtryck, signaturer, sessionsnycklar och krypterade data. En nyckelring är enligt standarden endast en samling nycklar; hur en organisation hittar nycklar, verifierar ägare och avgör förtroende ligger utanför detta dataformat. Det historiskt utbredda **Web of Trust** är därför en möjlig driftform, men inte en synonym definition av OpenPGP. Andra miljöer använder manuellt verifierade fingeravtryck, organisationssignerade nycklar, WKD, katalogtjänster eller centralt hanterade gatewayar ([RFC 9580, Keyrings och Fingerprints](https://datatracker.ietf.org/doc/html/rfc9580)).

För e-post kapslar PGP/MIME in signaturer och chiffertext i MIME-strukturer. Inline-PGP skyddar däremot endast utvalda textdelar och leder lätt till interoperabilitetsproblem med HTML, bilagor, teckenuppsättningar och radkanonisering. RFC 3156 standardiserar `multipart/encrypted` och `multipart/signed` för OpenPGP. Om en klient fullt ut stöder moderna RFC-9580-nycklar, AEAD, använda kurvor och PGP/MIME måste testas med de faktiska motparterna ([RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156), [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580)).

Ett kort Key ID är inte ett tillräckligt identitetsbevis. RFC 9580 varnar för att Key IDs inte ska antas vara unika; fullständigt fingeravtryck används för kontroller. Återkallelsecertifikat, utgång, subkeys, nya fingeravtryck och säker distribution av en rotation hör därför till samma livscykel som den första nyckelutgivningen.

S/MIME och OpenPGP kan användas på slutenheten eller centralt vid gatewayen. Gatewayen förenklar policy och nyckeldrift, men får därmed tillgång till klartexten.

## Gatewaymodell: central policy och central klartextmakt

En e-postgateway tar emot vanliga interna meddelanden, klassificerar avsändare, mottagare och data, söker efter lämpliga S/MIME- eller OpenPGP-nycklar och väljer kryptering, signatur, TLS eller portalleverans. Inkommande dekrypterar den vid behov före överlämning till det interna e-postsystemet. Produkter som [SEPPmail](/kb/seppmail), [Totemomail respektive Kiteworks EPG](/kb/totemomail) och [HIN](/kb/hin) använder olika plattformar, men samma grundläggande förtroendeförskjutning: organisationen delegerar kryptografi från användarens ändpunkt till en infrastrukturkomponent ([SEPPmail: Produkte](https://www.seppmail.com/products/), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Detta löser inte nyckeldistributionen automatiskt. Gatewayen behöver fortfarande en styrkt koppling mellan mottagaridentitet och publik nyckel, en prioritering av flera metoder, regler för utgångna eller återkallade nycklar och en definierad reservlösning. Automatiskt nyckelutbyte mellan gatewayar kan förenkla dessa steg, men är själv ett förtroendeprotokoll med motpartsidentitet, cache, utgång och feltillstånd.

Policyordningen avgör vilka kontroller som ser klartext. Om en inkommande gateway dekrypterar kan antimalware, DLP och arkivering arbeta därefter. Om en utgående gateway krypterar för tidigt är efterföljande skannrar blinda. Om en gateway ändrar en redan signerad MIME-entitet kan innehålls- eller [DKIM](/kb/mail-auth)-signaturen bli ogiltig. RFC 6376 gör de kanoniserade rubrikerna och brödtexten till signaturobjektet; därför måste signering, kryptering, disclaimers, omskrivning och DKIM ordnas i en dokumenterad pipeline ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)).

Om ingen lämplig nyckel finns tillgänglig hos den externa mottagaren kan en portal lagra meddelandet och styra åtkomsten via webbinloggning. Därmed övergår problemet från e-postkryptografi till webbidentitet och plattformsdrift.

## Portalleverans: webbapplikation i stället för mottagarnyckel

Vid portalleverans får mottagaren ett vanligt aviseringsmejl och hämtar det skyddade innehållet via en webbapplikation. Den kryptografiska ändpunkten är portalen, inte mottagarens e-postklient. Därmed flyttas den tekniska uppgiften från distribution av publika nycklar till webbidentitet, sessionshantering, lagring av meddelanden och bilagor, aviseringslänk, bevarande och säker svarskanal.

HIN dokumenterar som ett konkret exempel registrering, mobilnummer och SMS-kod samt möjligheten att komma ihåg en enhet för senare åtkomst. Detta exempel belägger driftmodellen, inte en allmängiltig portalimplementering. Andra produkter använder lösenord, engångskod via separat kanal, federerad identitet eller en befintlig kundinloggning ([HIN: Mail an Nichtmitglieder](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

För administratörer behöver en portal minst följande underlag:

- Vilka data finns i den oskyddade aviseringen och i URL-token?
- Hur säkras första registrering, identitetsbindning, misslyckade försök, att komma ihåg enhet och kontoåterställning?
- Var lagras meddelanden, bilagor, nycklar och revisionsdata, och när raderas de?
- Förblir ett meddelande återställbart efter utgång, användarradering eller tenantbyte?
- Hur loggas säkert svar, vidarebefordran, nedladdning och läsbekräftelse?
- Vad sker om portalen inte kan nås, SMS-tjänsten fallerar eller mottagarkontot komprometteras?

En HTTP-status `200` bevisar endast att ett webbgränssnitt svarar. En syntetisk portalgenomgång måste omfatta avisering, registrering respektive inloggning, andra faktor, öppning, bilaga, svar och den förväntade revisionsposten.

## Metadata, rubriker och arkivering

Innehållskryptering skyddar inte automatiskt SMTP-kuvertet. Levererande MTA:er behöver minst mottagaradresser; system genererar `Received`-rader och behandlar storleks- och routningsinformation. Även med S/MIME förblir yttre rubriker normalt synliga. RFC 8551 tillåter att ett fullständigt meddelande kapslas som `message/rfc822` inuti den skyddade MIME-entiteten, men kräver en särskiljbar presentation av inre och yttre rubriker. Om ämne, avsändarvisning och mottagarlista faktiskt skyddas är en funktion i klienten och gatewayen, inte en säker följd av låssymbolen ([RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), [RFC 8551, Header Protection](https://datatracker.ietf.org/doc/html/rfc8551)).

Arkivering måste fastställa **vilken representation** som bevaras: ursprunglig klartext, signerad originalentitet, krypterat original, dekrypterad arbetskopia eller portalobjekt. Enbart klartext förenklar eDiscovery, men förlorar det kryptografiska originalbeviset. Enbart chiffertext kan bli oläsbar efter nyckelförlust. Signaturvalidering vid en senare tidpunkt kräver originalbytes, certifikatkedja, dåvarande förtroende- och spärrinformation samt en tidsgrund; ett certifikat som har gått ut i dag gör inte automatiskt en då giltig signatur värdelös.

## Nyckel- och certifikatlivscykel

En kryptotjänst i produktion behöver ett inventarium som går längre än utgångsdatum. NIST SP 800-57 behandlar generering, aktivering, skydd, inventering, distribution, användning, rotation, spärr, arkiv, återställning och förstöring som sammanhängande hanteringsuppgifter ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

| Fas | Administratörsfråga | Bevis |
|---|---|---|
| Identitet | Vem eller vilken roll knyts till nyckeln? | kontrollerad enrollmentprocess, Subject/SAN/User ID, godkännande |
| Generering | Var skapas den privata nyckeln? | klient, HSM, gateway eller CA; algoritm och exporterbarhet |
| Distribution | Hur får avsändaren rätt publik nyckel? | katalog, signerad e-post, WKD, certifikattjänst, fingeravtryckskontroll |
| Användning | Får nyckeln signera, dekryptera eller båda? | Key Usage/EKU, OpenPGP-flaggor, policy och faktisk klient |
| Rotation | Hur överlappar gammal och ny nyckel? | publicering före övergång, test mot alla målplattformar |
| Återkallelse | Hur får granskare och avsändare kännedom om kompromettering? | CRL/OCSP eller OpenPGP-Revocation, cache- och felpolicy |
| Återställning | Vem får dekryptera historiskt innehåll? | escrowförfarande, fyrögonsgodkännande, revision, återställningstest |
| Förstöring | När upphör dekrypterbarhet avsiktligt? | bevarande- och raderingsbevis för nycklar och säkerhetskopior |

S/MIME-sökvägsvalidering omfattar förtroendeankare, mellanliggande certifikat, giltighetstid, namn respektive e-postidentitet, Key Usage/Extended Key Usage, algoritm och spärrstatus. OCSP och CRL:er är i sin tur beroenden av nätverk och cache. Vid en soft-fail kan en kontroll fortsätta trots att spärrinformation inte kan nås; vid hard-fail kan ett infrastrukturproblem blockera alla signerade eller krypterade meddelanden. Detta beslut hör hemma i policyn och övervakningen ([RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280), [RFC 6960](https://datatracker.ietf.org/doc/html/rfc6960)).

Diagnostiken börjar med den valda skyddsmodellen: TLS-hopp, innehållets signatur och kryptering eller portalåtkomst. Först därefter kontrolleras certifikat, nyckel, policy och leveransväg.

## Övervakning och felanalys

Övervakning måste delas upp efter skyddsmodell. För transport-TLS räknas erbjudna och förhandlade protokoll, policyfel, certifikatvalidering, TLS-RPT och köålder. För S/MIME/OpenPGP räknas saknade mottagarnycklar, ogiltiga sökvägar eller signaturer, reservlösningar, rotation och dekrypteringsfel. För gatewayar tillkommer policybeslut, Message-ID, Key-Fingerprint, klartextbearbetning och nästa hopp. För portaler är registrering, MFA/OTP, lagring, länkutgång, inloggning, meddelandeöppning och svarsväg separata signaler.

Ett ändamålsenligt incidentprotokoll svarar på:

1. Vilket meddelande och vilken mottagarväg berörs?
2. Vilken skyddsmodell och policy valdes?
3. Vilken publik nyckel eller vilket certifikat användes, med vilket fingeravtryck och vilken status?
4. Var var meddelandet senast läsbart, krypterat, lagrat eller köat?
5. Blev resultatet leverans, portalarkivering, karantän, reservlösning, återförsök eller permanent fel?
6. Kan resultatet reproduceras med en kontrollerad testidentitet utan att produktionsdata exponeras?

## Säkerhetskopiering och återställning

Krypterade data och nycklar måste planeras tillsammans, men med separata skyddsåtgärder. En säkerhetskopia av krypterade meddelanden utan de privata dekrypteringsnycklar som krävs långsiktigt kan vara korrekt säkrad men ändå inte fackmässigt återställbar. Omvänt omvandlar en otillräckligt skyddad säkerhetskopia för key escrow många enskilda hemligheter till en central angreppspunkt. NIST behandlar tillåten användning av återställda privata nycklar beroende på nyckeltyp och kryptoperiod ([NIST SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)).

Minimiomfattningen för återställning omfattar:

- krypterade originalmeddelanden och vid behov bevisäkra signerade MIME-entiteter;
- privata **dekrypterings**nycklar med metadata, lösenfraser, roller och godkännandeförfaranden;
- publika nycklar, certifikatkedjor, Trust Anchors, spärr- och historiska valideringsdata;
- gateway- och portalpolicy, routning, nyckelmappning, användardata och meddelandeobjekt;
- revisions- och tidskällor, så att tidigare beslut förblir spårbara;
- ett återställningstest med gamla och nya nycklar, flera mottagare, bilagor och signaturkontroll.

Privata signeringsnycklar bör inte kopieras som en allmän återställningsmekanism. Om en organisation måste signera automatiskt eller ställföreträdande är en uttrycklig organisationsidentitet med egen policy bättre än en obemärkt kopia av en personlig nyckel. Återställningsförmåga och icke-förnekbarhet står annars i konflikt med varandra.

## Diagnostikverktyg

Felsökningen följer den valda skyddsmodellen. Vid transportkryptering kontrolleras DNS och TLS; vid metoder från ände till ände följer certifikat, nycklar och klientbearbetning; vid gatewayar tillkommer deras policy- och köstatus.

### MTA-STS, TLS-RPT och DANE i DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für E-Mail-Transportpolicy-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type TXT _mta-sts.example.ch
Resolve-DnsName -Type TXT _smtp._tls.example.ch
Resolve-DnsName -Type TLSA _25._tcp.mx.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short TXT _mta-sts.example.ch
dig +short TXT _smtp._tls.example.ch
dig +dnssec TLSA _25._tcp.mx.example.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar publicerade transportpolicyer. För DANE måste dessutom DNSSEC-valideringsstatusen beläggas; en enbart synlig TLSA-text är ännu ingen autentiserad post.

### STARTTLS och certifikatkedja

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar TCP-vägen. [`curl`](https://curl.se/docs/manpage.html) och [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) visar STARTTLS, handshake och kedja. Om en produktiv MTA återgår vid fel eller köar framgår först av dess policy och loggar.

### S/MIME-certifikat och användningsändamål

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für S-MIME-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
certutil.exe -dump .\recipient-smime.cer
certutil.exe -verify -urlfetch .\recipient-smime.cer
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl x509 -in recipient-smime.pem -noout \
  -subject -issuer -serial -fingerprint -dates -purpose -ext keyUsage -ext extendedKeyUsage
openssl verify -show_chain -CAfile trust-chain.pem recipient-smime.pem
```

  </div>
</div>

[`certutil`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil) och [`openssl x509`](https://docs.openssl.org/master/man1/openssl-x509/) visar identitet, fingeravtryck, giltighet och användningsändamål; [`openssl verify`](https://docs.openssl.org/master/man1/openssl-verify/) kontrollerar en angiven Truststore. Testet måste upprepas med samma Trust Anchors och spärrregler som den produktiva klienten eller gatewayen.

### OpenPGP-fingeravtryck och nyckelstruktur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für OpenPGP-Schlüsselprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
gpg.exe --show-keys --with-fingerprint --with-subkey-fingerprint .\recipient.asc
gpg.exe --list-packets .\message.pgp
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
gpg --show-keys --with-fingerprint --with-subkey-fingerprint recipient.asc
gpg --list-packets message.pgp
```

  </div>
</div>

[`gpg`](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html) visar fullständiga fingeravtryck, subkeys, funktioner och paketstruktur. `--list-packets` analyserar endast formatet; det bekräftar varken ägare eller förtroende. Fingeravtryck ska kontrolleras via en autentiserad andra kanal eller ett definierat organisationsförfarande.

### Portalväg och paketinsamling

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Portal- und Paketpfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://securemail.example.ch/
pktmon filter remove
pktmon filter add SecureMail -p 25
pktmon start --capture --pkt-size 0 --file-name securemail.etl
pktmon stop
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://securemail.example.ch/
tcpdump -ni any -s 0 -w securemail.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar endast HTTP/TLS-åtkomlighet. [`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) och [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) samlar endast in vid den valda mätpunkten. Paketdata kan innehålla SMTP-klartext, adresser och portalmetadata och måste skyddas därefter.

## Teknisk historia

PEM, Privacy Enhanced Mail, standardiserade kryptografiskt skyddad internetmail i början av 1990-talet, men uppnådde ingen bred interoperabilitet. S/MIME förenade senare den från PKCS härledda CMS-strukturen med MIME och X.509. S/MIME-standarderna utvecklades över flera generationer fram till version 4.0 som publicerades 2019; CMS självt är standardiserat i RFC 5652 ([RFC 1421](https://datatracker.ietf.org/doc/html/rfc1421), [RFC 8551](https://datatracker.ietf.org/doc/html/rfc8551), [RFC 5652](https://datatracker.ietf.org/doc/html/rfc5652)).

OpenPGP uppstod ur PGP-paketformatet och standardiserades först i RFC 2440, senare i RFC 4880. RFC 9580 ersatte RFC 4880 år 2024 och moderniserade bland annat nyckelversioner, fingeravtryck och metoder för autentiserad kryptering. PGP/MIME enligt RFC 3156 förblir e-postinkapslingen; formatutvecklingen eliminerar dock inte den organisatoriska uppgiften att reglera nyckelidentitet och återställning ([RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580), [RFC 3156](https://datatracker.ietf.org/doc/html/rfc3156)).

SMTP STARTTLS standardiserades 1999 i RFC 2487 och uppdaterades 2002 genom RFC 3207. DANE för SMTP, MTA-STS, TLS-RPT och REQUIRETLS lade senare till policy- och rapportlager mot nedgradering och felkonfiguration. Parallellt etablerade företag gatewayar och webbportaler, eftersom externa mottagare inte tillförlitligt har kompatibla S/MIME- eller OpenPGP-nycklar. Därigenom blev e-postkryptering mindre en enskild klientfunktion och mer ett distribuerat system av SMTP, DNS, HTTPS, PKI, katalog, policy, webbidentitet och återställning ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461), [RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460), [RFC 8689](https://datatracker.ietf.org/doc/html/rfc8689)).

## Källor

- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 8551 – S/MIME 4.0 Message Specification](https://datatracker.ietf.org/doc/html/rfc8551)
- [IETF RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [IETF RFC 5652 – Cryptographic Message Syntax](https://datatracker.ietf.org/doc/html/rfc5652)
- [NCCoE – DNS-Based Email Security Practice Guide](https://www.nccoe.nist.gov/publication/1800-6/VolC/)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8689 – SMTP REQUIRETLS](https://datatracker.ietf.org/doc/html/rfc8689)
- [IETF RFC 8460 – SMTP TLS Reporting](https://datatracker.ietf.org/doc/html/rfc8460)
- [NIST – SP 800-177 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-177r1.pdf)
- [IETF RFC 5280 – Internet X.509 PKI Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – S/MIME in Exchange Server](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/smime/smime)
- [Microsoft Learn – S/MIME in Exchange Online](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo)
- [IETF RFC 3156 – MIME Security with OpenPGP](https://datatracker.ietf.org/doc/html/rfc3156)
- [SEPPmail – Produkte](https://www.seppmail.com/products/)
- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [IETF RFC 6376 – DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [HIN Support – Mail an Nichtmitglieder](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [IETF RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [IETF RFC 6960 – Online Certificate Status Protocol](https://datatracker.ietf.org/doc/html/rfc6960)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt1r5.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – certutil](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)
- [OpenSSL – x509](https://docs.openssl.org/master/man1/openssl-x509/)
- [OpenSSL – verify](https://docs.openssl.org/master/man1/openssl-verify/)
- [GnuPG – gpg manual](https://www.gnupg.org/documentation/manuals/gnupg24/gpg.1.html)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [IETF RFC 1421 – Privacy Enhancement for Internet Electronic Mail](https://datatracker.ietf.org/doc/html/rfc1421)
