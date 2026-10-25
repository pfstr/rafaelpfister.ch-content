---
title: "Den attackerade tjänsteleverantören för pensionskassor har ingen DMARC-post alls"
navTitle: "DMARC pensionskassor"
description: "PK Softech, tjänsteleverantören för pensionskassor som attackerades i slutet av september 2026, har fortfarande ingen DMARC-post. En passiv granskning av 66 pensionsinstitutioner och 30 tjänsteleverantörer visar att detta inte är ett enskilt fall: särskilt tillverkarna av administrationsprogramvara är oskyddade."
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
translationSourceHash: 57506056bfd8dcc96818df43dc71d22a158978bcd7432c91c56eb92698e52fe1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-20T08:43:14.530Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/den-angripna-pensionsfondsleverantoren-har-ingen-dmarc-post-alls
---

I slutet av september 2026 attackerades PK Softech AG, en programvarutillverkare vars pensionslösning används av flera schweiziska pensionskassor. Via denna väg läckte data ut. Enligt rapporteringen berörs bland annat den federala pensionskassan Publica, Migros pensionskassa och pensionskassan i staden Zürich samt ytterligare institutioner. Beroende på kassa handlar det om namn, AHV-nummer, civilstånd, adress och IBAN samt löne- och pensionsuppgifter.

Detta läckage är i sig allvarligt. Det berör identifierings- och finansiella uppgifter för en stor del av den arbetande befolkningen och, via Publica, stora delar av den federala förvaltningen. Data i denna omfattning och kvalitet är inte bara ett problem för den enskilda personen. De möjliggör identitetsmissbruk, riktat bedrägeri och utpressning i stor skala och utgör därmed en fara för den allmänna ordningen och den nationella säkerheten. Pensionsmedlen förblev opåverkade, men informationsplattformen Cybercrimepolice.ch varnade redan för bedrägerisamtal med de stulna uppgifterna.

Till detta kommer ett fynd som är svårt att förklara för en tjänsteleverantör i denna kategori: **PK Softech AG har fortfarande ingen DMARC-post.** Dess domän har en strikt SPF-post, men ingen DMARC-policy (kontrollerad den 18.10.2026). Därmed kan avsändaradressen för den attackerade tjänsteleverantören förfalskas utan att en mottagande e-postplattform tillförlitligt avvisar den. Efter ett dataläckage är detta sämsta tänkbara utgångsläge. Den som känner till en försäkrad persons namn, adress och bankuppgifter och dessutom kan förfalska ett e-postmeddelande i tjänsteleverantörens eller kassans namn kan skriva ett meddelande som knappt går att skilja från ett äkta, exempelvis en uppmaning att uppdatera IBAN för nästa pensionsutbetalning. Det är just mot denna förfalskning som DMARC skyddar.

PK Softech är inte ett enskilt fall, utan det tydligaste exemplet på ett mönster. Jag har granskat hur väl schweiziska pensionsinstitutioner och deras typiska tjänsteleverantörer är rustade för e-postautentisering.

## Hur granskningen genomfördes

Utvärderingen omfattar 66 pensionsinstitutioner inom den andra pelaren, från stora offentligrättsliga kassor och försäkringsbolagens samlingsstiftelser till mindre gemensamma institutioner. Endast offentliga DNS-poster granskades: SPF, DKIM, DMARC, MX-poster samt DNSSEC och DANE (TLSA-poster för e-postservrarna, kontrollerade den 19.10.2026). Det är samma uppgifter som varje mottagande e-postsystem frågar efter för varje inkommande meddelande.

Granskningen var helt passiv. Endast offentligt DNS efterfrågades, ingen anslutning upprättades till institutionernas system, inga inloggningsförsök gjordes och inga personuppgifter samlades in. Utvärderingen återger endast sådant som ändå är offentligt tillgängligt för alla. Koncernens samlingsstiftelser har sammanfattats under huvuddomänen för huvudmannen; enskilda anslutna stiftelser kan avvika.

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
| DKIM identifierbart | 55 % |
| DNSSEC | 38 % |
| DANE | 9 % |

Tre punkter sticker ut.

**Endast 20 procent är effektivt skyddade med DMARC `p=reject`.** Först denna policy instruerar en mottagande e-postplattform att faktiskt avvisa förfalskade meddelanden. I alla andra fall hamnar en förfalskning i tveksamma fall i den försäkrade personens inkorg.

**41 procent saknar DMARC.** Deras avsändardomän kan utan vidare användas för bedrägliga e-postmeddelanden, och institutionen får inte ens veta det eftersom rapporteringsfunktionen också saknas utan DMARC. Anmärkningsvärt är att detta även omfattar kassor som använder Microsoft 365 för sin e-post, där DMARC kan konfigureras med liten insats.

**DMARC `p=none` är inget skydd.** De 14 procenten i denna grupp har angett en DMARC-policy men lämnat den på övervakningsnivån. Det ger rapporter men avvisar inte ett enda förfalskat meddelande. I praktiken är detta ett mellansteg som ofta blir kvar i månader eller år.

Ett liknande mönster framträder för transportkryptering: endast 6 av de 66 institutionerna (9 procent) skyddar sina e-postservrar med DANE. Då kontrollerar en sändande e-postserver, med hjälp av en post signerad med DNSSEC, att den är krypterat ansluten till rätt mottagare. Förutsättningen är DNSSEC, som 38 procent av institutionerna redan har aktiverat. För dessa är DANE nästa naturliga steg.

## Alla datauppsättningar att klicka igenom

Följande infografik visar den granskade statusen för samtliga 66 pensionsinstitutioner och 30 tjänsteleverantörer. Med filtren kan resultatet begränsas efter DMARC-status, kategori och e-postleverantör, och varje post kan fällas ut för de fullständiga DNS-resultaten.

<div data-dmarc-infographic></div>

## Jämförelse: de största schweiziska företagen

Hur pensionsinstitutionerna står sig jämfört med resten av näringslivet visas av [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Den granskar dagligen över 1 000 av Schweiz största företag samt förbund, kantoner och kommuner med samma passiva metod, med en interaktiv karta per kanton, jämförelser per bransch och e-postleverantör samt en detaljsida för varje organisation. Alla pensionsinstitutioner och tjänsteleverantörer som granskas här finns där med en egen profil, sammanfattade i branschen [Vorsorgeeinrichtungen](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) och uppdateras dagligen. I infografiken ovan leder länken i den utfällda posten direkt till respektive profil.

Skillnaden är tydlig: Av de 518 företagen i indexet använder 38 procent DMARC med `p=reject` den 18.10.2026, jämfört med 20 procent bland pensionsinstitutionerna. Helt utan DMARC är 5 procent av företagen, jämfört med 41 procent av pensionsinstitutionerna.

## Leverantörerna är den större frågan

Incidenten i slutet av september drabbade inte en kassa utan en gemensam tjänsteleverantör. Därför granskade jag med samma passiva metod 30 typiska leverantörer: tillverkare av administrationsprogramvara, företag för outsourcad administration, aktuarietjänster och rådgivning, depåbanker och IT-tjänsteleverantörer.

I genomsnitt står tjänsteleverantörerna något bättre rustade än kassorna: 40 procent använder DMARC med `p=reject`, jämfört med 20 procent av institutionerna. Avgörande är dock fördelningen mellan kategorierna.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarietjänster och rådgivning | bra |
| Depåbanker | bra |
| IT-tjänsteleverantörer | blandat |
| Outsourcad administration | svagt |
| Administrationsprogramvara | svagast |

Tillverkarna av administrationsprogramvara är den svagaste gruppen. Ingen av de granskade programvaruleverantörerna använder DMARC med full effekt, och den attackerade leverantören PK Softech har ingen DMARC-post alls. Det är just den kategori som behandlar pensionsuppgifter och är djupt integrerad med kassorna via gränssnitt, och det är kategorin för den aktuella incidenten.

Härav följer den egentliga poängen för omsorgsplikten: En kassa med `p=reject` är väl skyddad så länge angriparen vill förfalska dess domän. Om angriparen i stället kan förfalska domänen för en dåligt skyddad tjänsteleverantör, genom vilken kommunikationen med de försäkrade ändå sker, är e-postmeddelandet lika trovärdigt. Att skydda den egna domänen är därför bara första steget. Det andra är att kräva samma nivå av de viktigaste tjänsteleverantörerna, helst avtalsmässigt.

## Vad detta innebär för stiftelserådet

Ansvaret för informationssäkerheten ligger hos pensionsinstitutionens högsta organ. Ett dataläckage medför dessutom konkreta skyldigheter enligt den reviderade dataskyddslagen: en överträdelse av datasäkerheten ska anmälas till EDÖB om den sannolikt medför en hög risk för de berörda personerna, och de berörda ska informeras i den mån det behövs för deras skydd.

E-postautentisering är i detta läge den mest effektiva enskilda åtgärden med minst arbetsinsats. SPF, DKIM och DMARC kräver varken licenser eller ny programvara. De består av DNS-poster och en korrekt konfiguration av den befintliga e-postplattformen. Skyddet fungerar omedelbart för varje försäkrad person, utan att personen själv behöver göra något.

## Vad som behöver göras

För institutioner utan effektivt skydd finns en tydlig väg till målet:

1. **Kontrollera SPF och gör det strikt.** SPF-posten listar de behöriga sändsystemen och avslutas med `-all`, så att alla övriga avsändare betraktas som obehöriga.
2. **Aktivera DKIM.** Varje utgående meddelande signeras kryptografiskt. I Microsoft 365 och vanliga e-postgatewayar är detta en inställning, inte ett projekt.
3. **Inför DMARC och skärp det stegvis.** Börja med `p=none` och aktiv rapportering för att kartlägga legitim e-posttrafik. Därefter `p=quarantine`, slutligen `p=reject`. Hela processen tar, beroende på miljö, några veckor, inte månader.
4. **Utvärdera rapporteringen.** DMARC-rapporterna visar vem som skickar e-post i domänens namn och avslöjar både bortglömda legitima system och missbruk.
5. **Granska åtminstone DANE och se över e-postkrypteringen.** DANE skyddar transportkrypteringen mellan e-postservrar med DNSSEC. Om den egna e-postplattformen och DNS-leverantören stöder detta kan klargöras med liten insats. För meddelanden med löne- och pensionsuppgifter bör det dessutom regleras hur de skickas med end-to-end-kryptering, exempelvis via en e-postgateway med S/MIME eller säker leverans via en webbportal.
6. **Ställ krav på tjänsteleverantörerna.** Programvaruleverantörer, outsourcad administration och IT-partner bör kunna visa samma nivå för SPF, DKIM och DMARC. Detta hör hemma i avtalen och i den regelbundna leverantörsgranskningen.

## Slutsats

De senaste angreppen riktades mot en tjänsteleverantör, inte mot kassorna själva. Följdskadorna uppstår dock där en avsändardomän är oskyddad, både hos kassan och dess leverantörer. Fyra av fem granskade pensionsinstitutioner är inte fullt skyddade mot förfalskade avsändare, två av fem saknar DMARC helt, och just tillverkarna av administrationsprogramvara presterar sämst. Bristen är välkänd, billig att åtgärda och ger omedelbar effekt. Efter ett dataläckage som omfattar namn, adress och bankuppgifter är den inte en teknisk bisak, utan en del av det högsta organets omsorgsplikt längs hela leveranskedjan.

## Källor

1.  [SRF: Förbundets pensionskassa – hur kunde känsliga data hamna hos ett externt programvaruföretag?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrund till incidenten, berörda datatyper och programvaruleverantörens roll.

2.  [cash.ch (AWP): Angrepp mot Publicas leverantör, löneuppgifter kan ha berörts](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica identifierar PK Softech AG som det attackerade företaget inför AWP.

3.  [Netzwoche: Förbundets pensionskassa Publica rapporterar dataläckage](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Rapport om angreppet mot programvaruleverantören och dataläckaget.

4.  [Mimikama: Schweizisk polis varnar efter dataläckage för möjliga investeringsbedrägerisamtal](https://www.mimikama.org/?p=436628): Kontext för varningen om bedrägerisamtal med de läckta uppgifterna.

5.  [EDÖB: Anmälan av överträdelser av datasäkerheten](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Anmälningsskyldighet enligt revDSG och förfarande vid överträdelser av datasäkerheten.
