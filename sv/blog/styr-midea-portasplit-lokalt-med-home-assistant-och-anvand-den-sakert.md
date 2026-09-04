---
title: "Styr Midea PortaSplit lokalt med Home Assistant och använd den säkert"
navTitle: "Konfigurera PortaSplit"
description: "Från rätt community-integration till IoT-VLAN: Så konfigurerar du PortaSplit, skyddar Token och Key och begränsar moln- och nätverksåtkomst."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "14 min läsning"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - serverloser-newsletter-cloudflare-workers-d1
slug: "styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert"
translationOf: "midea-portasplit-home-assistant-einrichten"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: bbe70b67dd255184cf0db69f7308c756937dc961c3c83e152268ee668f93dd07
translatedAt: 2026-09-04T08:35:50.856Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/sv/blog/styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert
---

Midea PortaSplit kan efter konfiguration styras direkt i det lokala nätverket via Home Assistant. För detta behöver community-integrationen två enhetsspecifika åtkomstvärden från Midea-molnet: Token och Key.

Den här artikeln vägleder dig genom val, konfiguration och säkring av integrationen. Lösningarna som beskrivs kommer från communityn och stöds inte officiellt av vare sig Midea eller Home Assistant. Ändringar i firmware eller molntjänsten kan därför när som helst påverka deras funktion. Bakgrunden till Token-gränssnittet och den tvetydiga varningen om avveckling finns i [analysen av Midea-moln-API:erna](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Så fungerar lokal styrning

De faktiska styrkommandona går efter konfigurationen direkt från Home Assistant till PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Ett styrkommando behöver inte gå via en extern Midea-server, svarstiden är kort, ett fel i Midea-molnet behöver inte avbryta den redan konfigurerade lokala styrningen och enheten kan i princip även styras utan internetåtkomst.

På nyare enheter med det så kallade V3-protokollet accepterar PortaSplit dock inte lokala kommandon oskyddat. Home Assistant behöver två enhetsspecifika värden, en Token och en Key, som används för autentisering och kryptering av den lokala anslutningen. Integrationen hämtar dem en gång via ett Midea-molngränssnitt under den första konfigurationen och lagrar dem sedan lokalt; ingen molnanslutning krävs för fortsatt styrning.

Förenklat ser förloppet ut så här:

1. PortaSplit ansluts till MSmartHome.
2. Home Assistant loggar in i ett Midea-moln.
3. Home Assistant får enhets-ID, Token och Key.
4. Token och Key lagras lokalt.
5. Home Assistant styr PortaSplit direkt i LAN:et.

## Vilken integration passar

### Midea Smart AC

Repositoryt <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> fokuserar på Midea-luftkonditioneringsenheter och relaterade OEM-modeller samt stöder enhetstyperna `0xAC` och `0xCC`. Det erbjuder lokal styrning, grafisk konfiguration, automatisk upptäckt, manuell konfiguration med Token och Key samt automatisk avläsning av enhetens funktioner. PortaSplits ”Out Silent Mode” stöds uttryckligen.

Projektet anger bland annat apparna Artic King, Midea Air, NetHome Plus, SmartHome respektive MSmartHome, Toshiba AC NA och 美的美居 som indikationer på kompatibilitet. PortaSplit använder vanligtvis MSmartHome i Europa och passar därmed in i detta ekosystem.

### Midea AC LAN

Repositoryt <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> stöder inte bara luftkonditioneringsanläggningar utan även många andra Midea-enhetsklasser: avfuktare, fläktar, luftrenare, tvättmaskiner, torktumlare, diskmaskiner, varmvattenberedare, värmepumpar, kylskåp och mer, delvis även under andra varumärken som Carrier eller Electrolux. Det erbjuder också lokal kommunikation, automatisk enhetsupptäckt och ytterligare sensorer och håller enligt projektbeskrivningen en längre TCP-anslutning till enheten öppen för att synkronisera statusändringar snabbt. Minst Home Assistant 2024.4.1 krävs.

Den största nackdelen är för närvarande utvecklarens varning: moln-Token-API:erna som används för att lägga till nya enheter avvecklas stegvis. Därmed kan det bli omöjligt att senare lägga till nya enheter.

### Rekommendation

För en ren PortaSplit-installation skulle jag börja med `Midea Smart AC` och känna till `Midea AC LAN` som ett alternativ. `Midea Smart AC` är mer fokuserad på luftkonditioneringsenheter och dokumenterar uttryckligen aktuella PortaSplit-funktioner.

Det är inte meningsfullt att köra båda integrationerna samtidigt och permanent med samma enhet. Flera parallella anslutningar leder till statusproblem, onödig nätverkstrafik och svårtolkat beteende.

## Vad integrationen ger

Efter konfigurationen visas PortaSplit som en `climate`-entitet i Home Assistant. Beroende på firmware och integration finns bland annat följande funktioner tillgängliga:

- Slå på och av
- Ställa in önskad temperatur
- Läsa av aktuell rumstemperatur
- Kylning, avfuktning och rent fläktläge
- Ställa in fläkthastighet
- Styra Swing-funktionen
- Eco- och Boost-läge
- Läsa av luftfuktighet
- Visa felkoder
- Läsa av energi- och effektvärden
- Visa kompressorvärden
- Aktivera tyst läge för utomhusenheten

Vilka entiteter som faktiskt visas beror på modellen, firmwaren, det använda protokollet och respektive integration. `Midea Smart AC` frågar efter funktionerna som rapporteras av enheten och döljer funktioner som modellen inte stöder. `Midea AC LAN` dokumenterar också omfattande klimatentiteter, bland annat temperatur, luftfuktighet, aktuell effekt, total energi, kompressorfrekvens, pumpstatus och olika driftlägen, och nämner egna metoder för avkodning av energidata för vissa PortaSplit-undertyper.

Inte alla visade mätvärden behöver vara korrekta. Särskilt energiförbrukning och effekt överförs i olika format för olika Midea-modeller. Om Home Assistant visar uppenbart felaktiga värden behöver vanligen den använda avkodningsmetoden justeras, inte enheten repareras.

## Förutsättningar

Du behöver en Midea PortaSplit med WLAN-funktion, ett 2,4 GHz-WLAN, MSmartHome-appen, ett Midea-användarkonto, Home Assistant, HACS och nätverksåtkomst mellan Home Assistant och PortaSplit. PortaSplit bör först anslutas på normalt sätt med MSmartHome-appen, och först därefter läggas till i Home Assistant.

## Steg 1: Anslut PortaSplit till MSmartHome

1. Installera MSmartHome-appen.
2. Skapa ett Midea-konto eller logga in.
3. Sätt PortaSplit i WLAN-parkopplingsläge.
4. Anslut enheten till 2,4 GHz-WLAN:et.
5. Kontrollera att PortaSplit kan styras via appen.

Många IoT-enheter stöder fortfarande endast 2,4 GHz. Om routern använder samma SSID för 2,4 och 5 GHz fungerar konfigurationen oftast ändå. Vid problem hjälper det att tillfälligt tillhandahålla ett separat 2,4 GHz-WLAN.

## Steg 2: Installera HACS

HACS är Home Assistant Community Store. Med den kan community-integrationer installeras som inte ingår i Home Assistant Core. Efter HACS-installationen öppnar du HACS, går till integrationerna, söker efter `Midea Smart AC`, laddar ner integrationen och startar om Home Assistant. Alternativt kan du söka efter `Midea AC LAN`.

HACS förenklar installation och uppdateringar. Det gör dock inte en Custom Integration till en officiellt granskad Home Assistant-komponent. Denna skillnad är viktig ur säkerhetssynpunkt och tas upp längre ned.

## Steg 3: Lägg till Midea Smart AC

Efter omstarten går du via Inställningar, Enheter och tjänster och Lägg till integration till sökningen efter `Midea Smart AC`, och sedan till `Discover devices`. Integrationen kan antingen söka igenom hela det lokala nätverket eller fråga efter PortaSplits IP-adress direkt.

Om enheten hittas behöver integrationen för nyare V3-enheter region, Midea-konto, lösenord och enhets-ID samt den Token och Key som härleds från dessa. Molnregionen måste passa det använda kontot. Vid problem rekommenderar projektet att även prova de andra tillgängliga regionerna.

### Manuell konfiguration

Om den automatiska konfigurationen misslyckas kan enheten konfigureras manuellt. För `Midea Smart AC` behövs följande uppgifter:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

Den dokumenterade standardporten är:

```text
6444/TCP
```

För V3-enheter anger dokumentationen Token som en hexadecimal teckensträng med 128 tecken och Key som en med 64 tecken. Båda värdena är hemligheter och ska behandlas därefter. Den som inte vill hämta inloggningsuppgifterna via Discovery kan hämta dem med sitt eget konto via CLI `msmart-ng`.

## Använd PortaSplit säkert

Den som styr PortaSplit lokalt tar tillbaka en del av kontrollen från tillverkarens moln, men flyttar därmed också ansvaret till sitt eget nätverk. Följande punkter säkerställer att Token och Key orsakar liten skada även vid en incident och att enheten förblir ordentligt isolerad.

### Token och Key är hemligheter

Token och Key autentiserar den lokala kommunikationen med enheten och ska behandlas som ett lösenord. För driften gäller framför allt: de hör inte hemma i loggar, okrypterade säkerhetskopior eller ett repository.

### Ingen portvidarebefordran till PortaSplit

Det vanligaste undvikbara misstaget vore att göra den lokala enhetsporten direkt åtkomlig från internet. En regel som denna skulle vara farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finns ingen bra anledning att göra PortaSplit direkt åtkomlig från internet. Home Assistant finns redan i det lokala nätverket och fungerar som kontrollerande instans. Routern bör inte ha någon portvidarebefordran till PortaSplit, UPnP bör begränsas eller avaktiveras där det är möjligt, inkommande anslutningar bör blockeras som standard och ingen DMZ-öppning bör användas för enheten.

### Eget IoT-VLAN

Den bästa nätverksarkitekturen är ett separat IoT-nätverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit finns i IoT-VLAN:et. Home Assistant får specifikt åtkomst till enheten, men PortaSplit får inte fritt komma åt datorer, NAS och andra interna system. En möjlig brandväggslogik:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under den första konfigurationen behöver enheten internetåtkomst till Midea-molnet. Efter att den lokala konfigurationen lyckats kan du testa om utgående internetåtkomst kan blockeras. En permanent spärr bör dock inte sättas direkt. Kontrollera först om lokal styrning fortfarande fungerar, om enheten förblir åtkomlig efter omstart, om den klarar en omstart av routern, om den fortfarande svarar efter flera dagar, om MSmartHome-appen fortfarande behövs och om firmwareuppdateringar fortfarande erbjuds. Den som vill fortsätta använda molnet och firmwareuppdateringar kan tillfälligt tillåta utgående internetåtkomst och sedan blockera den igen.

### Nätverkssegmentering kan förhindra Discovery

Automatisk enhetsupptäckt bygger ofta på broadcast- eller multicast-trafik, som normalt inte routas över VLAN-gränser. Därför kanske Home Assistant inte automatiskt hittar PortaSplit, även om en vanlig IP-anslutning skulle vara tillåten.

Då hjälper det att tillfälligt konfigurera PortaSplit i samma VLAN som Home Assistant, ange enhetens IP-adress manuellt, använda en lämplig broadcast-reläfunktion eller definiera specifika brandväggsregler efter konfigurationen. Manuell konfiguration är ofta till och med det bättre alternativet ur säkerhetssynpunkt, eftersom ingen ytterligare broadcast-trafik mellan näten behöver tillåtas.

### Statisk DHCP-tilldelning

PortaSplit bör få en fast DHCP-tilldelning i routern:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservation är vanligtvis att föredra framför en statisk IP som ställs in i enheten. Home Assistant hittar enheten tillförlitligt, brandväggsregler kan begränsas till en fast adress, felsökningen blir enklare och tilldelningen förblir stabil efter omstart av router eller enhet. En brandväggsregel kan därmed formuleras mycket snävt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Vilken port som faktiskt behövs måste verifieras utifrån integrationen och den egna enheten.

### Home Assistant som central förtroendeankare

Den som styr PortaSplit lokalt flyttar delvis förtroendet från Midea-molnet till Home Assistant. Om Home Assistant komprometteras kan en angripare eventuellt kontrollera inte bara luftkonditioneringen utan hela det smarta hemmet.

Home Assistant bör därför uppdateras regelbundet, inte publiceras via oskyddad portvidarebefordran, skyddas med ett starkt och unikt lösenord, använda flerfaktorsautentisering, skapa krypterade säkerhetskopior, endast innehålla nödvändiga tillägg och inte tillåta onödig SSH-åtkomst från internet. För fjärråtkomst är ett VPN, Home Assistant Cloud eller en korrekt konfigurerad omvänd proxy bättre alternativ än enkel portvidarebefordran till port 8123.

### HACS och risken i leveranskedjan

`Midea Smart AC` och `Midea AC LAN` är Custom Integrations. De körs inom Home Assistant och får därmed omfattande åtkomst till dess körmiljö. En skadlig eller komprometterad integration skulle teoretiskt kunna läsa konfigurationsdata, hämta hemligheter, upprätta nätverksanslutningar, skanna enheter i det lokala nätverket, läsa tillstånd hos andra entiteter, överföra data till externa system och påverka tillgängligheten för Home Assistant.

Det betyder inte att de nämnda integrationerna är skadliga. Båda projekten är offentligt tillgängliga, utvecklas aktivt och har en synlig community. Open source är dock ingen automatisk säkerhetsgaranti. Före installation är det värt att åtminstone kontrollera om repositoryt underhålls aktivt, om det finns regelbundna releaser, hur många personer som bidrar till koden, om det finns öppna säkerhetsfrågor, om underhållare eller repository-ägare nyligen har bytts ut, om HACS pekar på det förväntade repositoryt och om en uppdatering innehåller ovanligt stora eller oförklarliga ändringar.

Uppdateringar bör inte installeras blint direkt efter publicering. Särskilt för säkerhetskritiska smarta hemsystem är det klokt att vänta några dagar och kontrollera versionsanteckningar och rapporterade problem.

### Skydda molnkontot

Så länge Midea-molnet används för konfiguration eller appstyrning förblir Midea-kontot en del av säkerhetsmodellen. Det ska ha ett unikt lösenord som inte delas med andra tjänster, en lösenordshanterare, flerfaktorsautentisering om den erbjuds, borttagning av gamla smarttelefoner och sessioner, inga delade konton samt regelbunden kontroll av vilka enheter som är registrerade på kontot.

Om Home Assistant-integrationen begär användarnamn och lösenord under konfigurationen bör du kontrollera om inloggningsuppgifterna endast används för att hämta Token en gång eller lagras permanent. Utvecklarna av `Midea Smart AC` skriver att enheter efter konfigurationen inte kopplas till inbyggda integrationskonton och att Token och Key även kan hämtas manuellt med det egna kontot via CLI. Där det är möjligt bör det egna kontot föredras framför främmande eller integrerade samlingskonton.

### Blockera molnet eller inte?

Efter en lyckad konfiguration uppstår frågan om PortaSplits internetåtkomst ska blockeras helt. Argument för en blockering är mindre telemetri, mindre beroende av externa tjänster, en mindre attackyta via tillverkarens moln, att enheten inte kan kontakta godtyckliga externa mål och mindre påverkan av ändringar på molnsidan.

Emot talar att MSmartHome-appen kanske inte längre fungerar utanför hemnätverket, att firmwareuppdateringar inte laddas ner, att klock- eller molnfunktioner kan sluta fungera, att ny inloggning eller återställning blir svårare och att vissa enheter reagerar oväntat efter en längre tid offline.

En pragmatisk ordning är: konfigurera enheten normalt, testa Home Assistant och appen, säkerhetskopiera Token och konfiguration, blockera internetåtkomsten, starta om enheten och Home Assistant, observera under flera dagar och vid behov endast tillfälligt återaktivera internetåtkomsten.

### Firmwareuppdateringar: säkerhetsvinst eller integrationsrisk?

Firmwareuppdateringar är ett dilemma för IoT-enheter. De kan åtgärda kända sårbarheter, förbättra stabiliteten, modernisera säkerhetsmekanismer och ge nya funktioner. Men de kan också ändra lokala gränssnitt, bryta integrationer baserade på reverse engineering, göra Token ogiltiga, avaktivera det lokala API:et och skapa nya molnberoenden.

PortaSplit-firmwaren som levererades i januari 2026 gav till exempel ett nytt tyst läge för utomhusenheten, som minskar ljudnivån med cirka 6 decibel. Community-integrationerna behövde först förstå och implementera detta, vilket dokumenterats i ett eget GitHub-ärende för PortaSplit.

Slutsatsen är: förhindra inte firmwareuppdateringar generellt, kontrollera före en uppdatering om andra Home Assistant-användare rapporterar problem, säkerhetskopiera konfiguration och Token i förväg, skapa en Home Assistant-säkerhetskopia och testa den lokala styrningen fullständigt efter uppdateringen. Säkerhet betyder inte ”uppdatera aldrig”. Föråldrad firmware kan vara farligare än en tillfälligt inkompatibel integration.

### Debug-loggar innehåller känsliga data

Vid problem begär open source-projekt ofta debug-loggar. Dokumentationen för `Midea AC LAN` visar hur loggning aktiveras för de två relevanta komponenterna:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Därefter kan loggarna laddas ner via Inställningar, System och Loggar. Sådana loggar kan, beroende på integration och fel, innehålla lokala IP-adresser, enhets-ID, serienummer, modellidentifiering, molnsvar, kontoinformation, Token eller delar av den, nätverkspaket samt tidsstämplar och användningsbeteende. Innan de laddas upp till ett offentligt GitHub-ärende bör de därför granskas och känsliga värden maskeras.

När felsökningen är klar ska debug-loggningen tas bort igen. Permanent aktiverad debug-loggning ökar inte bara lagringsförbrukningen utan även mängden känslig information i säkerhetskopiorna.

### Vad Midea själva säger om säkerhet

Midea marknadsför sitt SmartHome-ekosystem med inriktning på flera säkerhets- och dataskyddsstandarder, där EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling och kraven i EU:s Radio Equipment Directive nämns. Det är positiva signaler, men inget uttalande om hur varje enskild PortaSplit-firmware, varje molnslutpunkt och varje lokalt API faktiskt har implementerats. Certifierings- och marknadsföringspåståenden ersätter inte en teknisk granskning av den konkreta enheten.

På samma sätt vore det fel att dra slutsatsen att PortaSplit generellt är osäker utifrån varningen från en community-integration. Det beskrivna problemet gäller arkitekturen för långlivade Tokens och deras användning av inofficiella klienter.

### Risk per scenario

| Scenario | Risk | Motivering |
| --- | --- | --- |
| Normalt hemnätverk utan portvidarebefordran | hanterbar | En angripare måste först få åtkomst till WLAN, Home Assistant eller en säkerhetskopia. |
| Platt hemnätverk med många osäkra IoT-enheter | medel | En komprometterad annan IoT-enhet kan nå PortaSplit eller Home Assistant i samma nätverk. |
| PortaSplit direkt åtkomlig från internet | hög | Enheten bör aldrig publiceras via portvidarebefordran. |
| Token och Key offentliga på GitHub | hög | Hemligheterna betraktas som komprometterade; det är inte garanterat att de kan återkallas. |
| Separat IoT-VLAN, restriktiv brandvägg, lokal styrning | jämförelsevis låg | Även vid en sårbarhet i enheten är rörelsefriheten i nätverket kraftigt begränsad. |

## Säkerhetskopia av konfigurationen

Säkerhetskopiering av Token, Key och konfiguration är det viktigaste engångssteget: när molnets Token-gränssnitt väl har stängts är en säkerhetskopia den enda vägen till en ny konfiguration. `Midea AC LAN` sparar efter lyckad konfiguration en JSON-konfigurationsfil för V3-enheter. Den dokumenterade sökvägen är:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID:t som filnamn:

```text
<device-id>.json
```

Den här filen är inte en vanlig textanteckning. Den kan innehålla enhets-ID, serienummer, IP-adress, Token, Key, protokollinformation samt moln- och enhetsparametrar. Följaktligen gäller:

- Ladda inte upp den till ett offentligt GitHub-repository.
- Publicera den inte i forum.
- Dela den inte som en omaskerad skärmbild.
- Skicka den inte via okrypterad e-post.

Inte heller ett privat Git-repository är automatiskt rätt lagringsplats, eftersom hemligheter blir kvar i Git-historiken även om de senare tas bort från den aktuella filen. Mer lämpliga är en krypterad säkerhetskopia, en lösenordshanterare med filbilaga, en krypterad NAS-säkerhetskopia, ett krypterat offlinemedium eller ett krypterat arkiv med lösenordet lagrat separat.

För säkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visa fil:

```bash
cat <device-id>.json
```

Vid kopiering bör filen inte överföras via en offentlig webbtjänst. Bättre är ett krypterat arkiv som sedan förs över till en krypterad säkerhetskopia:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filerna i `.storage` bör inte redigeras manuellt. Utvecklaren rekommenderar uttryckligen att JSON-filen vid problem varken raderas eller ändras direkt, utan att den byter namn och säkerhetskopieras före ändringar.

En fullständig Home Assistant-säkerhetskopia innehåller också dessa filer. En separat kopia är ändå meningsfull, eftersom Home Assistant-säkerhetskopior kan skadas, en återställning kan skriva över integrationen, filen kan behövas särskilt för en senare ny konfiguration och en säkerhetskopia aldrig bara bör ligga på samma system.

## Ta bort hemligheter från ett publicerat Git-repository

Om en JSON-fil av misstag har publicerats på GitHub räcker det inte att radera den normalt och göra en ny commit. Filen förblir åtkomlig i Git-historiken. Minst följande steg krävs:

1. Gör omedelbart repositoryt privat, om möjligt.
2. Ta bort filen från hela Git-historiken.
3. Ta hänsyn till GitHub-cachar och forks.
4. Behandla Token som komprometterad.
5. Ta bort enheten från Midea-kontot och anslut den på nytt, om detta skapar nya nycklar.
6. Konfigurera Home Assistant-integrationen på nytt.
7. Byt lösenordet till Midea-kontot om även inloggningsuppgifter berördes.

Om omparkopplingen faktiskt skapar en ny Token varierar beroende på enhet och molnarkitektur. Du bör inte förlita dig på att ett byte av kontolösenordet automatiskt gör den lokala enhets-Token ogiltig.

## Användbara automationer

Efter en lyckad integration kan PortaSplit användas betydligt smartare. Entitets-ID:na måste anpassas till den egna installationen.

Kyl endast när fönstren är stängda:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Slå på vid hög rumstemperatur:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Förkyl före läggdags:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Stäng av när ingen är hemma:

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Rekommenderad konfiguration i korthet

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

Önskad kommunikationsriktning ser därmed ut så här:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

## Rekommenderat driftsläge

Midea PortaSplit kan integreras väl i Home Assistant. Efter lyckad konfiguration kan den styras lokalt och användas i automationer, vilket eliminerar en stor del av molnberoendet i den dagliga driften.

Ur säkerhetssynpunkt är integrationen försvarbar om några grundregler följs: ingen portvidarebefordran, håll Token och Key hemliga, kryptera säkerhetskopior, granska debug-loggar före publicering, säkra Home Assistant, segmentera IoT-enheter, begränsa utgående internetåtkomst till det nödvändiga och installera inte firmware- och HACS-uppdateringar blint. Använd på detta sätt förblir PortaSplit en kraftfull luftkonditioneringsanläggning och blir samtidigt en meningsfullt integrerbar del av ett lokalt styrt smart hem.

## Källor

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: enhetstyper som stöds `0xAC` och `0xCC`, PortaSplit med ”Out Silent Mode”, molnanvändning för att hämta Token och Key för V3-enheter, manuell konfiguration och standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN`: enhetsklasser som stöds, längre TCP-anslutning för statussynkronisering och lägsta versionen Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentation av klimatentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entiteter och attribut för luftkonditioneringsenheter, bland annat effekt, total energi, kompressorfrekvens och avkodningsmetoder för energivärden för enskilda undertyper.

4.  [midea_ac_lan: Debug- och konfigurationsanvisningar](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): lagring av enhetskonfigurationen under `/config/.storage/midea_ac_lan/`, rekommendation att säkerhetskopiera i stället för att radera JSON-filen och logger-konfigurationen för debug-loggar.

5.  [Issue 779: PortaSplits Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): begäran om stöd för det tysta läget för utomhusenheten som infördes med firmwareuppdateringen i januari 2026 och som minskar ljudnivån med cirka 6 decibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): tillverkarens uppgifter om säkerhets- och dataskyddsstandarderna EN 303 645, PSTI, NIST, GDPR och RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation och hantering av Custom Integrations som inte ingår i Home Assistant Core.
