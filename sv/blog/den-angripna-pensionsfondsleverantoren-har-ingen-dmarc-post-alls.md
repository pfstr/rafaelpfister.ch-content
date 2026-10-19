---
title: "Den angripna tjänsteleverantören för pensionskassor har ingen DMARC-post alls"
navTitle: "DMARC pensionskassor"
description: "PK Softech, tjänsteleverantören för pensionskassor som attackerades i slutet av september 2026, har fortfarande ingen DMARC-post. En passiv granskning av 66 pensionsinstitut och 30 tjänsteleverantörer visar att detta inte är ett enskilt fall: särskilt tillverkarna av administrationsprogramvara är oskyddade."
date: "2026-10-18"
kategorie: "SMTP och e-postflöde"
timeToRead: "9 min läsning"
themen:
  - smtp-mailflow
produkte: []
protokolle:
  - "mail-auth"
  - "dns"
slug: "den-angripna-pensionsfondsleverantoren-har-ingen-dmarc-post-alls"
translationId: "article-a34017b39fecf271"
zielgruppe: "entscheider"
translationOf: dmarc-pensionskassen-schweiz
translationSourceHash: 56251be16ac00d877c2536fc6384a564114736e38301637d9006f8e21da77d9a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T08:34:09.246Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/den-angripna-pensionsfondsleverantoren-har-ingen-dmarc-post-alls
---

I slutet av september 2026 attackerades PK Softech AG, en programvarutillverkare vars pensionslösning används av flera schweiziska pensionskassor. Data läckte ut denna väg. Enligt rapporteringen berörs bland annat den federala pensionskassan Publica, Migros pensionskassa och pensionskassan för staden Zürich samt ytterligare institutioner. Beroende på pensionskassa rör det sig om namn, AHV-nummer, civilstånd, adress och IBAN samt löne- och pensionsuppgifter.

Detta läckage är i sig allvarligt. Det berör identitets- och finansuppgifter för en stor del av den arbetande befolkningen och, genom Publica, stora delar av den federala förvaltningen. Data i denna omfattning och kvalitet är inte bara ett problem för den enskilda personen. De möjliggör identitetsmissbruk, riktat bedrägeri och utpressning i stor skala och utgör därmed en fara för den allmänna ordningen och den nationella säkerheten. Pensionsmedlen i sig förblev orörda, men informationsplattformen Cybercrimepolice.ch varnade redan för bedrägerisamtal med de stulna uppgifterna.

Därtill kommer ett konstaterande som är svårt att förklara för en tjänsteleverantör i denna kategori: **PK Softech AG har fortfarande ingen DMARC-post.** Dess domän har en strikt SPF-post, men ingen DMARC-policy (kontrollerat den 18.10.2026). Därmed kan avsändaradressen för den angripna tjänsteleverantören förfalskas utan att en mottagande e-postplattform tillförlitligt avvisar detta. Efter ett dataläckage är detta det sämsta tänkbara utgångsläget. Den som känner till en försäkrad persons namn, adress och bankuppgifter och dessutom kan förfalska ett e-postmeddelande i tjänsteleverantörens eller pensionskassans namn, kan skriva ett meddelande som knappt går att skilja från ett äkta, exempelvis en uppmaning att uppdatera IBAN för nästa pensionsutbetalning. Det är just mot denna förfalskning som DMARC skyddar.

PK Softech är inte ett enskilt fall, utan det tydligaste exemplet på ett mönster. Jag har granskat hur väl schweiziska pensionsinstitut och deras typiska tjänsteleverantörer är rustade för e-postautentisering.

## Hur granskningen genomfördes

Utvärderingen omfattar 66 pensionsinstitut inom den andra pelaren, från stora offentligrättsliga kassor och försäkringsbolagens samlingsstiftelser till mindre gemensamma institutioner. Endast offentliga DNS-poster granskades: SPF, DKIM, DMARC, MX-poster samt DNSSEC och DANE (TLSA-poster för e-postservrar, kontrollerade den 19.10.2026). Det är samma uppgifter som varje mottagande e-postsystem frågar efter för varje inkommande meddelande.

Granskningen var helt passiv. Endast offentliga DNS-poster efterfrågades, ingen anslutning upprättades till institutionernas system, inga inloggningsförsök gjordes och inga personuppgifter samlades in. Utvärderingen återger endast det som redan är offentligt tillgängligt för alla. Koncernanknutna samlingsstiftelser har sammanförts under huvuddomänen för huvudmannen; enskilda anslutna stiftelser kan avvika.

## Resultatet

| Egenskap | Andel av de 66 institutionerna |
|---|---|
| SPF finns | 88 % |
| SPF strikt (`-all`) | 59 % |
| DMARC finns | 59 % |
| DMARC `p=reject` (fullständigt skydd) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (endast övervakning) | 14 % |
| ingen DMARC | 41 % |
| DKIM kan identifieras | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punkter sticker ut.

**Endast 20 procent är effektivt skyddade med DMARC `p=reject`.** Först denna policy instruerar en mottagande e-postplattform att faktiskt avvisa förfalskade meddelanden. I alla andra fall hamnar en förfalskning i tveksamma fall i den försäkrade personens inkorg.

**41 procent har ingen DMARC.** Deras avsändardomän kan utan vidare användas för bedrägliga e-postmeddelanden, och institutionen får inte ens veta det, eftersom rapporteringsfunktionen också saknas utan DMARC. Anmärkningsvärt är att detta även omfattar kassor som använder Microsoft 365 för sin e-post, där DMARC kan konfigureras med liten arbetsinsats.

**DMARC `p=none` är inget skydd.** De 14 procent i denna grupp har angett en DMARC-policy, men lämnat den på övervakningsnivån. Det ger rapporter men avvisar inte ett enda förfalskat meddelande. I praktiken är det ett mellansteg som ofta blir kvar i månader eller år.

Även för transportkryptering ser bilden liknande ut: endast 6 av de 66 institutionerna (9 procent) skyddar sina e-postservrar med DANE. Därmed kontrollerar en sändande e-postserver, med hjälp av en DNSSEC-signerad post, om den är krypterat ansluten till rätt mottagare. Förutsättningen är DNSSEC, som 38 procent av institutionerna redan har aktiverat. För dem är DANE nästa naturliga steg.

## Alla dataset att utforska

Följande infografik visar den granskade statusen för alla 66 pensionsinstitut och 30 tjänsteleverantörer. Med filtren kan urvalet begränsas efter DMARC-status, kategori och e-postleverantör, och varje post kan expanderas för fullständiga DNS-resultat.

<div data-dmarc-infographic></div>

## Jämförelse: de största schweiziska företagen

Hur pensionsinstituten står sig jämfört med övriga näringslivet framgår av [Mail Security Index Switzerland](/open-data/mail-security-index). Den granskar dagligen över 500 av Schweiz största företag med samma passiva metod, med en interaktiv karta per kanton, jämförelser per bransch och e-postleverantör samt en detaljsida för varje företag. Där ingår även bankerna, försäkringsbolagen och IT-tjänsteleverantörerna som pensionsinstituten samarbetar med.

Skillnaden är tydlig: Av de 518 företagen i indexet använder 38 procent DMARC med `p=reject` den 18.10.2026, medan motsvarande andel bland pensionsinstituten är 20 procent. Helt utan DMARC är 5 procent av företagen, jämfört med 41 procent av pensionsinstituten.

## Leverantörerna är den större frågan

Händelsen i slutet av september drabbade inte en pensionskassa utan en gemensam tjänsteleverantör. Därför har jag med samma passiva metod granskat 30 typiska leverantörer: tillverkare av administrationsprogramvara, företag för outsourcad administration, aktuarietjänster och rådgivning, depåbanker och IT-tjänsteleverantörer.

I genomsnitt står tjänsteleverantörerna något bättre rustade än kassorna: 40 procent använder DMARC med `p=reject`, jämfört med 20 procent bland institutionerna. Avgörande är dock fördelningen mellan kategorierna.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarietjänster och rådgivning | bra |
| Depåbanker | bra |
| IT-tjänsteleverantörer | blandat |
| Outsourcad administration | svagt |
| Administrationsprogramvara | svagast |

Tillverkarna av administrationsprogramvara är den svagaste gruppen. Ingen av de granskade programvaruleverantörerna använder DMARC med full effekt, och den angripna leverantören PK Softech har ingen DMARC-post alls. Det är just den kategori som behandlar pensionsuppgifter och är djupt integrerad med kassorna via gränssnitt, och det är kategorin i den aktuella händelsen.

Därav följer den egentliga poängen för aktsamhetsplikten: En kassa med `p=reject` är väl skyddad så länge angriparen vill förfalska dess domän. Kan angriparen i stället förfalska domänen för en dåligt skyddad tjänsteleverantör genom vilken kommunikation med de försäkrade ändå sker, är e-postmeddelandet lika trovärdigt. Att säkra den egna domänen är därför bara det första steget. Det andra är att kräva samma nivå av de viktigaste tjänsteleverantörerna, helst genom avtal.

## Vad detta innebär för stiftelserådet

Ansvaret för informationssäkerheten ligger hos pensionsinstitutionens högsta organ. Ett dataläckage medför dessutom konkreta skyldigheter enligt den reviderade dataskyddslagen: En överträdelse av datasäkerheten ska anmälas till EDÖB om den sannolikt medför en hög risk för de berörda personerna, och de berörda ska informeras i den mån det är nödvändigt för deras skydd.

E-postautentisering är i denna situation den mest effektiva enskilda åtgärden med minst arbetsinsats. SPF, DKIM och DMARC kräver varken licenser eller ny programvara. De består av DNS-poster och en korrekt konfiguration av den befintliga e-postplattformen. Skyddet fungerar omedelbart för varje försäkrad person, utan att personen själv behöver göra något.

## Vad som behöver göras

För institutioner utan effektivt skydd finns en tydlig väg till målet:

1. **Kontrollera SPF och gör den strikt.** SPF-posten listar de behöriga sändsystemen och avslutas med `-all`, så att alla övriga avsändare betraktas som obehöriga.
2. **Aktivera DKIM.** Varje utgående meddelande signeras kryptografiskt. I Microsoft 365 och i vanliga e-postgateways är detta en inställning, inte ett projekt.
3. **Inför DMARC och skärp den stegvis.** Börja med `p=none` och aktiv rapportering för att registrera legitim e-posttrafik. Därefter `p=quarantine`, och slutligen `p=reject`. Hela processen tar några veckor beroende på miljön, inte månader.
4. **Utvärdera rapporteringen.** DMARC-rapporterna visar vem som skickar e-post i domänens namn och avslöjar både bortglömda legitima system och missbruk.
5. **Utvärdera åtminstone DANE och se över e-postkrypteringen.** DANE skyddar transportkrypteringen mellan e-postservrar via DNSSEC. Om den egna e-postplattformen och DNS-leverantören har stöd för detta kan klarläggas med liten arbetsinsats. För meddelanden med löne- och pensionsuppgifter bör det dessutom regleras hur de skickas end-to-end-krypterat, exempelvis via en e-postgateway med S/MIME eller säker leverans via en webbportal.
6. **Håll tjänsteleverantörerna ansvariga.** Programvaruleverantörer, outsourcad administration och IT-partner bör kunna visa att de uppfyller samma nivå för SPF, DKIM och DMARC. Det hör hemma i avtalen och i den regelbundna leverantörsgranskningen.

## Slutsats

De senaste attackerna riktades mot en tjänsteleverantör, inte mot kassorna själva. Följdskadorna uppstår dock där en avsändardomän är oskyddad, både hos pensionskassan och dess leverantörer. Fyra av fem granskade pensionsinstitut är inte helt skyddade mot förfalskade avsändare, två av fem har ingen DMARC alls, och just tillverkarna av administrationsprogramvara presterar sämst. Bristen är känd, billig att åtgärda och får omedelbar effekt. Efter ett dataläckage som berör namn, adress och bankuppgifter är den inte en teknisk bisak, utan en del av aktsamhetsplikten för det högsta organet, genom hela leveranskedjan.

## Källor

1.  [SRF: Förbundets pensionskassa – hur kunde känsliga data hamna hos ett externt programvaruföretag?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrund till händelsen, berörda datatyper och programvaruleverantörens roll.

2.  [cash.ch (AWP): Attack mot Publica-leverantör, löneuppgifter kan ha berörts](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica anger PK Softech AG som det angripna företaget för AWP.

3.  [Netzwoche: Förbundets pensionskassa Publica rapporterar dataläckage](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Rapport om attacken mot programvaruleverantören och dataläckaget.

4.  [Mimikama: Schweizisk polis varnar för möjliga investeringssamtal efter dataläckage](https://www.mimikama.org/?p=436628): Kontext till varningen för bedrägerisamtal med de läckta uppgifterna.

5.  [EDÖB: Anmälan av överträdelser av datasäkerheten](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Anmälningsskyldighet enligt revDSG och förfarande vid överträdelser av datasäkerheten.
