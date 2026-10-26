---
title: "Den attackerade pensionskassetjänsteleverantören har ingen DMARC-post alls"
navTitle: "DMARC pensionskassor"
description: "Pensionskassetjänsteleverantören PK Softech, som attackerades i slutet av september 2026, har fortfarande ingen DMARC-post. En passiv granskning av 66 pensionsinstitut och 30 tjänsteleverantörer visar att detta inte är ett enskilt fall: särskilt tillverkarna av administrationsprogramvara är oskyddade."
date: "2026-10-18"
kategorie: "SMTP och e-postflöde"
timeToRead: "9 min lästid"
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
translationSourceHash: a673909df1426a76c58ced5aaa46d839e480ae867b2ffad468c903d65919e0c3
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:16:29.910Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/den-angripna-pensionsfondsleverantoren-har-ingen-dmarc-post-alls
---

I slutet av september 2026 attackerades PK Softech AG, en programvarutillverkare vars pensionslösning används av flera schweiziska pensionskassor. Data läckte ut via denna väg. Enligt rapporteringen drabbades bland annat den federala pensionskassan Publica, Migros pensionskassa och pensionskassan för staden Zürich, liksom ytterligare institutioner. Beroende på kassa handlar det om namn, AHV-nummer, civilstånd, adress och IBAN samt löne- och pensionsuppgifter.

Läckaget är i sig allvarligt. Det berör identifierings- och ekonomiska uppgifter för en stor del av den arbetande befolkningen och, genom Publica, stora delar av den federala förvaltningen. Data i denna mängd och kvalitet är inte bara ett problem för den enskilda personen. De möjliggör identitetsmissbruk, riktade bedrägerier och utpressning i stor skala och utgör därmed en fara för den allmänna ordningen och den nationella säkerheten. Själva pensionsmedlen förblev opåverkade, men informationsplattformen Cybercrimepolice.ch varnade redan för bedrägerisamtal med de stulna uppgifterna.

Därtill kommer ett konstaterande som är svårt att förklara för en tjänsteleverantör i denna kategori: **PK Softech AG har fortfarande ingen DMARC-post.** Dess domän har en strikt SPF-post, men ingen DMARC-policy (kontrollerat 18.10.2026). Därmed kan avsändaradressen till den attackerade tjänsteleverantören förfalskas utan att en mottagande e-postplattform tillförlitligt avvisar den. Efter ett dataläckage är detta sämsta tänkbara utgångsläge. Den som känner till en försäkrad persons namn, adress och bankuppgifter och dessutom kan förfalska ett e-postmeddelande i tjänsteleverantörens eller kassans namn kan skriva ett meddelande som knappt går att skilja från ett äkta, till exempel en uppmaning att uppdatera IBAN för nästa pensionsutbetalning. Det är just mot denna förfalskning som DMARC skyddar.

PK Softech är inte ett enskilt fall, utan det tydligaste exemplet på ett mönster. Jag har granskat hur väl schweiziska pensionsinstitut och deras typiska tjänsteleverantörer är rustade när det gäller e-postautentisering.

## Hur granskningen genomfördes

Utvärderingen omfattar 66 pensionsinstitut i den andra pelaren, från stora offentligrättsliga kassor och försäkringsbolagens samlingsstiftelser till mindre gemensamma institutioner. Endast offentliga DNS-poster granskades: SPF, DKIM, DMARC, MX-poster samt DNSSEC och DANE (TLSA-poster för e-postservrarna, kontrollerat 19.10.2026). Det är samma uppgifter som varje mottagande e-postsystem frågar efter för varje inkommande meddelande.

Granskningen var helt passiv. Endast offentliga DNS-uppgifter efterfrågades, ingen anslutning upprättades till institutionernas system, inga inloggningsförsök gjordes och inga personuppgifter samlades in. Utvärderingen återger endast det som ändå är offentligt tillgängligt för alla. Koncernanknutna samlingsstiftelser har sammanfattats under huvudmannens huvuddomän; enskilda anslutna stiftelser kan avvika.

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

**Endast 20 procent skyddas effektivt med DMARC `p=reject`.** Det är först denna policy som instruerar en mottagande e-postplattform att faktiskt avvisa förfalskade meddelanden. I alla andra fall hamnar en förfalskning i tveksamma fall i den försäkrade personens inkorg.

**41 procent har ingen DMARC.** Deras avsändardomän kan utan vidare användas för bedrägliga e-postmeddelanden, och institutionen får inte ens veta det eftersom rapporteringsfunktionen också saknas utan DMARC. Anmärkningsvärt är att detta även omfattar kassor som hanterar sin e-post med Microsoft 365, där DMARC kan konfigureras med liten insats.

**DMARC `p=none` är inget skydd.** De 14 procenten i denna grupp har angett en DMARC-policy men lämnat den på övervakningsnivå. Det ger rapporter, men avvisar inte ett enda förfalskat meddelande. I praktiken är detta ett mellansteg som ofta blir kvar i månader eller år.

När det gäller transportkryptering framträder en liknande bild: Endast 6 av de 66 institutionerna (9 procent) skyddar sina e-postservrar med DANE. Därmed kontrollerar en sändande e-postserver, utifrån en post signerad med DNSSEC, om den är krypterat ansluten till rätt mottagare. Förutsättningen är DNSSEC, som 38 procent av institutionerna redan har aktiverat. För dessa är DANE nästa naturliga steg.

## Alla poster att klicka igenom

Följande infografik visar den granskade statusen för samtliga 66 pensionsinstitut och 30 tjänsteleverantörer. Med filtren kan urvalet begränsas efter DMARC-status, kategori och e-postleverantör, och varje post kan expanderas för de fullständiga DNS-resultaten.

<div data-dmarc-infographic></div>

## Som jämförelse: de största schweiziska företagen

Hur pensionsinstituten står sig jämfört med övriga näringslivet visas av [Mail-Sicherheitsindex Schweiz](/open-data/mail-security-index). Med samma passiva metod granskar det dagligen över 1000 av Schweiz största företag samt förbund, kantoner och kommuner, med en interaktiv karta efter kantoner, jämförelser per bransch och e-postleverantör samt en detaljsida för varje organisation. Alla pensionsinstitut och tjänsteleverantörer som granskas här ingår där med en egen profil, sammanfattade i branschen [Pensionsinstitut](/open-data/mail-security-index/branche/vorsorgeeinrichtungen) och uppdateras dagligen. I infografiken ovan leder länken i den expanderade posten direkt till respektive profil.

Skillnaden är tydlig: Av de 518 företagen i indexet använder 38 procent DMARC med `p=reject` den 18.10.2026, jämfört med 20 procent bland pensionsinstituten. Helt utan DMARC är 5 procent av företagen, jämfört med 41 procent bland pensionsinstituten.

## Leverantörerna är den större frågan

Incidenten i slutet av september drabbade inte en kassa utan en gemensam tjänsteleverantör. Därför har jag med samma passiva metod granskat 30 typiska leverantörer: tillverkare av administrationsprogramvara, företag för outsourcad företagsledning, aktuariat och konsultverksamhet, depåbanker och IT-tjänsteleverantörer.

I genomsnitt står tjänsteleverantörerna något bättre rustade än kassorna: 40 procent använder DMARC med `p=reject`, jämfört med 20 procent bland institutionerna. Avgörande är dock fördelningen mellan kategorierna.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuariat och konsultverksamhet | bra |
| Depåbanker | bra |
| IT-tjänsteleverantörer | blandat |
| Outsourcad företagsledning | svagt |
| Administrationsprogramvara | svagast |

Tillverkarna av administrationsprogramvara är den svagaste gruppen. Ingen av de granskade programvaruleverantörerna använder DMARC med full verkan, och den attackerade leverantören PK Softech har ingen DMARC-post alls. Det är just denna kategori som behandlar pensionsuppgifter och är djupt integrerad med kassorna via gränssnitt, och det är kategorin för den aktuella incidenten.

Detta leder till den egentliga poängen för aktsamheten: En kassa med `p=reject` är väl skyddad så länge angriparen vill förfalska dess domän. Om angriparen i stället kan förfalska domänen för en dåligt skyddad tjänsteleverantör, genom vilken kommunikation med de försäkrade ändå sker, är e-postmeddelandet lika trovärdigt. Att skydda den egna domänen är därför bara det första steget. Det andra är att kräva att de viktigaste tjänsteleverantörerna uppnår samma nivå, helst avtalsmässigt.

## Vad detta innebär för stiftelserådet

Ansvaret för informationssäkerheten ligger hos pensionsinstitutets högsta organ. Ett dataläckage utlöser dessutom konkreta skyldigheter enligt den reviderade dataskyddslagen: En överträdelse av datasäkerheten ska anmälas till EDÖB om den sannolikt medför en hög risk för de berörda personerna, och de berörda ska informeras i den mån det är nödvändigt för deras skydd.

E-postautentisering är i detta läge den effektivaste enskilda åtgärden med minst arbetsinsats. SPF, DKIM och DMARC kräver varken licens eller ny programvara. De består av DNS-poster och en korrekt konfiguration av den befintliga e-postplattformen. Skyddet verkar omedelbart för varje försäkrad person, utan att denne själv behöver göra något.

## Vad som behöver göras

För institutioner utan effektivt skydd finns en tydlig väg till målet:

1. **Kontrollera SPF och gör den strikt.** SPF-posten listar de behöriga sändningssystemen och avslutas med `-all`, så att alla övriga avsändare betraktas som obehöriga.
2. **Aktivera DKIM.** Varje utgående meddelande signeras kryptografiskt. I Microsoft 365 och vanliga e-postgateways är detta en inställning, inte ett projekt.
3. **Inför DMARC och skärp den stegvis.** Börja med `p=none` och aktiv rapportering för att kartlägga legitim e-posttrafik. Därefter `p=quarantine`, slutligen `p=reject`. Hela processen tar, beroende på miljön, några veckor, inte månader.
4. **Utvärdera rapporteringen.** DMARC-rapporterna visar vem som skickar e-post i domänens namn och avslöjar både bortglömda legitima system och missbruk.
5. **Granska åtminstone DANE och se över e-postkrypteringen.** DANE skyddar transportkrypteringen mellan e-postservrar via DNSSEC. Det går att klarlägga med liten insats om den egna e-postplattformen och DNS-leverantören stöder detta. För meddelanden med löne- och pensionsuppgifter bör det dessutom regleras hur de skickas end-to-end-krypterat, exempelvis via en e-postgateway med S/MIME eller säker leverans via en webbportal.
6. **Ställ krav på tjänsteleverantörerna.** Programvaruleverantörer, outsourcad företagsledning och IT-partner bör kunna visa att de uppfyller samma nivå för SPF, DKIM och DMARC. Detta hör hemma i avtalen och i den regelbundna leverantörsgranskningen.

## Slutsats

De senaste attackerna riktades mot en tjänsteleverantör, inte mot kassorna själva. Följdskadorna uppstår dock där en avsändardomän är oskyddad, både hos kassan och dess leverantörer. Fyra av fem granskade pensionsinstitut är inte fullt skyddade mot förfalskade avsändare, två av fem saknar DMARC helt, och just tillverkarna av administrationsprogramvara presterar sämst. Bristen är känd, billig att åtgärda och får omedelbar effekt. Efter ett dataläckage som berör namn, adress och bankuppgifter är den inte en teknisk bisak utan en del av det högsta organets aktsamhetsplikt, genom hela leveranskedjan.

## Ändringshistorik

- **25 oktober 2026:** Källänken till EDÖB:s nya adress uppdaterades.

## Källor

1.  [SRF: Förbundets pensionskassa – hur kunde känsliga uppgifter hamna hos ett externt programvaruföretag](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrund till incidenten, berörda datatyper och programvaruleverantörens roll.

2.  [cash.ch (AWP): Attack mot Publicas leverantör, löneuppgifter kan ha berörts](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica uppger för AWP att PK Softech AG är det attackerade företaget.

3.  [Netzwoche: Förbundets pensionskassa Publica rapporterar dataläckage](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Rapport om attacken mot programvaruleverantören och dataläckaget.

4.  [Mimikama: Schweizisk polis varnar efter dataläckage för möjliga investeringssamtal](https://www.mimikama.org/?p=436628): Kontext till varningen för bedrägerisamtal med de läckta uppgifterna.

5.  [EDÖB: DataBreach](https://www.edoeb.admin.ch/de/databreach): Anmälningsplikt enligt art. 24 DSG, anmälningsportal och vägledning för hantering av överträdelser av datasäkerheten.
