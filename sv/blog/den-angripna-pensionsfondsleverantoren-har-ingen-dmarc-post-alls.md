---
title: "Den angripna pensionsfondsleverantören har ingen DMARC-post alls"
navTitle: "DMARC pensionsfonder"
description: "Pensionsfondsleverantören PK Softech, som angreps i slutet av september 2026, har fortfarande ingen DMARC-post. En passiv granskning av 66 pensionsinstitutioner och 30 tjänsteleverantörer visar att detta inte är ett enskilt fall: särskilt tillverkarna av administrationsprogramvara saknar skydd."
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
aiPrompt: |
  Du bist mein Berater für E-Mail-Sicherheit. Prüfe für meine Domain, ob SPF, DKIM, DMARC und MTA-STS korrekt gesetzt sind, erkläre mir den aktuellen Stand in einfachen Worten und zeige mir Schritt für Schritt, wie ich DMARC gefahrlos von p=none über p=quarantine auf p=reject anhebe.
translationOf: dmarc-pensionskassen-schweiz
url: https://rafaelpfister.ch/sv/blog/den-angripna-pensionsfondsleverantoren-har-ingen-dmarc-post-alls
translationSourceHash: 64ec035a358c39b09487102da66cf7aca02df2c567e29e343895906a4490d205
translationModel: gpt-5.6-terra
translatedAt: 2026-10-18T14:54:40.713Z
translationReview: automatic
---

I slutet av september 2026 angreps PK Softech AG, en programvarutillverkare vars pensionslösning används av flera schweiziska pensionsfonder. Data läckte via denna väg. Enligt rapporteringen omfattas bland annat den federala pensionskassan Publica, Migros pensionskassa och staden Zürichs pensionskassa, liksom ytterligare institutioner. Beroende på kassa handlar det om namn, AHV-nummer, civilstånd, adress och IBAN samt löne- och pensionsuppgifter.

Detta läckage är allvarligt i sig. Det berör identifikations- och finansuppgifter för en stor del av den arbetande befolkningen och, via Publica, stora delar av den federala förvaltningen. Data i denna omfattning och kvalitet är inte bara ett problem för den enskilda personen. De möjliggör identitetsmissbruk, riktat bedrägeri och utpressning i stor skala och utgör därmed en fara för den allmänna ordningen och den nationella säkerheten. Själva pensionsmedlen förblev opåverkade, men cyberbrottspolisen varnade redan för bedrägerisamtal med hjälp av de stulna uppgifterna.

Därtill kommer ett konstaterande som är svårt att förklara för en tjänsteleverantör i denna kategori: **PK Softech AG har fortfarande ingen DMARC-post.** Dess domän har en strikt SPF-post, men ingen DMARC-policy (kontrollerat den 18.10.2026). Därmed kan avsändaradressen för den angripna tjänsteleverantören förfalskas utan att en mottagande e-postplattform tillförlitligt avvisar den. Efter ett dataläckage är detta den sämsta tänkbara utgångspunkten. Den som känner till en försäkrad persons namn, adress och bankuppgifter och dessutom kan förfalska ett e-postmeddelande i tjänsteleverantörens eller kassans namn kan skriva ett meddelande som knappast går att skilja från ett äkta, exempelvis en uppmaning att uppdatera IBAN för nästa pensionsutbetalning. Det är just mot denna förfalskning som DMARC skyddar.

PK Softech är inte ett enskilt fall, utan det tydligaste exemplet på ett mönster. Jag har granskat hur väl schweiziska pensionsinstitutioner och deras typiska tjänsteleverantörer är rustade för e-postautentisering.

## Så genomfördes granskningen

Utvärderingen omfattar 66 pensionsinstitutioner inom den andra pelaren, från de stora offentligrättsliga kassorna och försäkringsbolagens samlingsstiftelser till mindre gemensamma institutioner. Endast offentliga DNS-poster granskades: SPF, DKIM, DMARC, MTA-STS, TLS-RPT och MX-poster. Det är samma uppgifter som varje mottagande e-postsystem frågar efter för varje inkommande meddelande.

Granskningen var helt passiv. Endast offentligt DNS efterfrågades, ingen anslutning upprättades till institutionernas system, inga inloggningsförsök gjordes och inga personuppgifter samlades in. Utvärderingen återger endast sådant som ändå är offentligt tillgängligt för alla. Koncernsamlingstiftelser har sammanförts under huvuddomänen för huvudmannen; enskilda anslutna stiftelser kan avvika.

## Resultatet

| Egenskap | Andel av de 66 institutionerna |
|---|---|
| SPF finns | 88 % |
| Strikt SPF (`-all`) | 59 % |
| DMARC finns | 59 % |
| DMARC `p=reject` (fullständigt skydd) | 20 % |
| DMARC `p=quarantine` | 26 % |
| DMARC `p=none` (endast övervakning) | 14 % |
| ingen DMARC | 41 % |
| DKIM kan identifieras | 55 % |
| MTA-STS | 2 % |
| TLS-RPT | 5 % |

Tre punkter sticker ut.

**Endast 20 procent är effektivt skyddade med DMARC `p=reject`.** Först denna policy instruerar en mottagande e-postplattform att faktiskt avvisa förfalskade meddelanden. I alla andra fall hamnar en förfalskning i tveksamma fall i den försäkrade personens inkorg.

**41 procent saknar DMARC.** Deras avsändardomän kan utan vidare användas för bedrägliga e-postmeddelanden, och institutionen får inte ens veta om det eftersom rapporteringsfunktionen också saknas utan DMARC. Anmärkningsvärt är att detta även omfattar kassor som använder Microsoft 365 för sin e-post, där DMARC kan konfigureras med liten arbetsinsats.

**DMARC `p=none` är inget skydd.** De 14 procenten i denna grupp har satt en DMARC-policy, men lämnat den på övervakningsnivå. Det ger rapporter men avvisar inte ett enda förfalskat meddelande. I praktiken är det ett mellansteg som ofta blir kvar i månader eller år.

Därtill kommer ett genomgående konstaterande: med 2 procent är MTA-STS praktiskt taget obefintligt. Utan MTA-STS tvingas inte transportkryptering för inkommande e-post, vilket gynnar en angripare med åtkomst till transportvägen.

## Alla dataposter att klicka igenom

Följande infografik visar den granskade statusen för samtliga 66 pensionsinstitutioner och 30 tjänsteleverantörer. Med filtren går det att begränsa efter DMARC-status, kategori och e-postleverantör, och varje post kan fällas ut för de fullständiga DNS-resultaten.

<div data-dmarc-infographic></div>

## Jämförelse: de största schweiziska företagen

Hur pensionsinstitutionerna står sig jämfört med övriga näringslivet visas av [Mail Security Index Switzerland](/open-data/mail-security-index). Det granskar varje dag över 500 av de största schweiziska företagen med samma passiva metod, med en interaktiv karta över kantoner, jämförelser efter bransch och e-postleverantör samt en detaljsida för varje företag. Bland dem finns även de banker, försäkringsbolag och IT-tjänsteleverantörer som pensionsinstitutionerna samarbetar med.

Skillnaden är tydlig: Av de 518 företagen i indexet använder 38 procent DMARC med `p=reject` den 18.10.2026, medan andelen bland pensionsinstitutionerna är 20 procent. Helt utan DMARC är 5 procent av företagen, jämfört med 41 procent av pensionsinstitutionerna.

## Leverantörerna är den större frågan

Händelsen i slutet av september drabbade inte en kassa, utan en gemensam tjänsteleverantör. Därför har jag granskat 30 typiska leverantörer med samma passiva metod: tillverkare av administrationsprogramvara, företag för outsourcad administration, aktuarietjänster och konsultverksamhet, depåbanker samt IT-tjänsteleverantörer.

I genomsnitt är tjänsteleverantörerna något bättre rustade än kassorna: 40 procent använder DMARC med `p=reject`, jämfört med 20 procent bland institutionerna. Avgörande är dock fördelningen mellan kategorierna.

| Kategori | med DMARC `p=reject` |
|---|---|
| Aktuarietjänster och konsultverksamhet | bra |
| Depåbanker | bra |
| IT-tjänsteleverantörer | blandat |
| Outsourcad administration | svagt |
| Administrationsprogramvara | svagast |

Tillverkarna av administrationsprogramvara är den svagaste gruppen. Ingen av de granskade programvaruleverantörerna använder DMARC med full verkan, och den angripna leverantören PK Softech har ingen DMARC-post alls. Det gäller just den kategori som behandlar pensionsuppgifter och är djupt integrerad med kassorna via gränssnitt, och det är kategorin för den aktuella händelsen.

Därav följer den egentliga poängen för omsorgsplikten: En kassa med `p=reject` är väl skyddad så länge angriparen vill förfalska dess domän. Kan angriparen i stället förfalska domänen hos en dåligt skyddad tjänsteleverantör som ändå kommunicerar med de försäkrade, är e-postmeddelandet lika trovärdigt. Att skydda den egna domänen är därför bara det första steget. Det andra är att ålägga de viktigaste tjänsteleverantörerna att uppnå samma nivå, helst genom avtal.

## Vad detta innebär för stiftelserådet

Ansvaret för informationssäkerheten ligger hos pensionsinstitutionens högsta organ. Ett dataläckage utlöser dessutom konkreta skyldigheter enligt den reviderade dataskyddslagen: En överträdelse av datasäkerheten ska anmälas till EDÖB om den sannolikt leder till en hög risk för de berörda personerna, och de berörda ska informeras i den mån det behövs för deras skydd.

E-postautentisering är i detta läge den mest effektiva enskilda åtgärden med minst arbetsinsats. SPF, DKIM och DMARC kräver varken licens eller ny programvara. De består av DNS-poster och en korrekt konfiguration av den befintliga e-postplattformen. Skyddet fungerar omedelbart för varje försäkrad person, utan att personen själv behöver göra något.

## Vad som behöver göras

För institutioner utan effektivt skydd finns en tydlig väg till målet:

1. **Granska SPF och gör den strikt.** SPF-posten listar de behöriga sändsystemen och avslutas med `-all`, så att alla övriga avsändare betraktas som obehöriga.
2. **Aktivera DKIM.** Varje utgående meddelande signeras kryptografiskt. I Microsoft 365 och i vanliga e-postgateways är detta en inställning, inte ett projekt.
3. **Inför DMARC och skärp den stegvis.** Börja med `p=none` och aktiv rapportering för att registrera den legitima e-posttrafiken. Därefter `p=quarantine`, slutligen `p=reject`. Hela processen tar några veckor beroende på miljön, inte månader.
4. **Utvärdera rapporteringen.** DMARC-rapporterna visar vem som skickar e-post i domänens namn och avslöjar både bortglömda legitima system och missbruk.
5. **Komplettera med MTA-STS och TLS-RPT.** Båda tvingar fram och övervakar transportkrypteringen för inkommande e-post.
6. **Håll tjänsteleverantörerna ansvariga.** Programvaruleverantörer, outsourcad administration och IT-partner bör kunna styrka samma nivå för SPF, DKIM och DMARC. Detta hör hemma i avtalen och i den regelbundna leverantörsgranskningen.

## Slutsats

De senaste årens angrepp riktade sig mot en tjänsteleverantör, inte mot kassorna själva. Följdskadorna uppstår dock där en avsändardomän är oskyddad, både hos kassan och hos dess leverantörer. Fyra av fem granskade pensionsinstitutioner är inte fullt skyddade mot förfalskade avsändare, två av fem har ingen DMARC alls, och just tillverkarna av administrationsprogramvara presterar sämst. Bristen är välkänd, billig att åtgärda och får omedelbar effekt. Efter ett dataläckage som berör namn, adress och bankuppgifter är den inte en teknisk bisak, utan en del av det högsta organets omsorgsplikt, genom hela leveranskedjan.

## Källor

1.  [SRF: Förbundets pensionskassa – hur kunde känsliga data hamna hos ett externt programvaruföretag?](https://www.srf.ch/news/schweiz/pensionskasse-des-bundes-wie-konnten-heikle-daten-bei-externer-softwarefirma-landen): Bakgrund till händelsen, berörda datatyper och programvaruleverantörens roll.

2.  [cash.ch (AWP): Angrepp mot Publicas leverantör, löneuppgifter kan ha berörts](https://www.cash.ch/news/top-news/angriff-auf-publica-lieferanten-lohndaten-konnten-betroffen-sein-975247): Publica uppger för AWP att PK Softech AG är det angripna företaget.

3.  [Netzwoche: Förbundets pensionskassa Publica rapporterar dataläckage](https://www.netzwoche.ch/news/2026-10-08/bundespensionskasse-publica-meldet-datenabfluss): Rapport om angreppet mot programvaruleverantören och dataläckaget.

4.  [Mimikama: Schweizisk polis varnar för möjliga investeringssamtal efter dataläckage](https://www.mimikama.org/?p=436628): Bedömning av varningen för bedrägerisamtal med de läckta uppgifterna.

5.  [EDÖB: Anmälan av överträdelser av datasäkerheten](https://www.edoeb.admin.ch/de/meldung-von-verletzungen-der-datensicherheit): Anmälningsskyldighet enligt revDSG och förfarande vid överträdelser av datasäkerheten.
