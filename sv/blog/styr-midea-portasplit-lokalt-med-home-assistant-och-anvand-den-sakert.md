---
title: "Styr Midea PortaSplit lokalt med Home Assistant och använd den säkert"
navTitle: "Konfigurera PortaSplit"
description: "Från rätt community-integration till IoT-VLAN: Så konfigurerar du PortaSplit, skyddar token och key samt begränsar moln- och nätverksåtkomst."
date: "2026-07-24"
kategorie: "Smarta hem & IoT"
timeToRead: "14 min lästid"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert"
translationOf: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/sv/blog/styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert"
---

Midea PortaSplit kan efter konfiguration styras direkt i det lokala nätverket via Home Assistant. För detta behöver community-integrationen två enhetsspecifika åtkomstvärden från Midea-molnet: token och key.

Den här artikeln går igenom val, konfiguration och säkerhet för integrationen. De beskrivna lösningarna kommer från communityn och stöds inte officiellt av vare sig Midea eller Home Assistant. Firmware- eller molnändringar kan därför när som helst påverka deras funktion. Bakgrunden till token-gränssnittet och den tvetydiga varningen om avveckling finns i [analysen av Midea Cloud-API:erna](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Så fungerar den lokala styrningen

De faktiska styrkommandona skickas efter konfigurationen direkt från Home Assistant till PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Ett styrkommando behöver inte gå via en extern Midea-server, svarstiden är kort, ett fel i Midea-molnet behöver inte slå ut den redan konfigurerade lokala styrningen och enheten kan i princip styras även utan internetåtkomst.

På nyare enheter med det så kallade V3-protokollet accepterar PortaSplit dock inte lokala kommandon utan skydd. Home Assistant behöver två enhetsspecifika värden, en token och en key, som används för autentisering och kryptering av den lokala anslutningen. Integrationen hämtar dem en gång via ett Midea-molngränssnitt under den första konfigurationen och lagrar dem sedan lokalt; ingen molnanslutning krävs för fortsatt styrning.

Förenklat ser processen ut så här:

1. PortaSplit ansluts till MSmartHome.
2. Home Assistant loggar in på ett Midea-moln.
3. Home Assistant får enhets-ID, token och key.
4. Token och key lagras lokalt.
5. Home Assistant styr PortaSplit direkt i LAN.

## Vilken integration passar

### Midea Smart AC

Repositoryt <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> fokuserar på Midea-luftkonditioneringsapparater och relaterade OEM-modeller och stöder enhetstyperna `0xAC` och `0xCC`. Det erbjuder lokal styrning, grafisk konfiguration, automatisk upptäckt, manuell konfiguration med token och key samt automatisk avläsning av enhetsfunktioner. PortaSplits ”Out Silent Mode” stöds uttryckligen.

Som indikator på kompatibilitet nämner projektet bland annat apparna Artic King, Midea Air, NetHome Plus, SmartHome respektive MSmartHome, Toshiba AC NA och 美的美居. PortaSplit använder vanligtvis MSmartHome i Europa och passar därmed in i detta ekosystem.

### Midea AC LAN

Repositoryt <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> stöder inte bara luftkonditioneringsapparater utan också många andra Midea-enhetsklasser: avfuktare, fläktar, luftrenare, tvättmaskiner, torktumlare, diskmaskiner, varmvattenberedare, värmepumpar, kylskåp och mycket mer, delvis även under andra varumärken som Carrier eller Electrolux. Det erbjuder också lokal kommunikation, automatisk enhetsupptäckt och ytterligare sensorer och håller enligt projektbeskrivningen en längre TCP-anslutning till enheten öppen för att synkronisera statusändringar snabbt. Minst Home Assistant 2024.4.1 krävs.

Den största nackdelen är för närvarande utvecklarens varning: De moln-token-API:er som används för att lägga till nya enheter avvecklas stegvis. Detta kan göra det omöjligt att senare lägga till nya enheter.

### Rekommendation

För en ren PortaSplit-installation skulle jag börja med `Midea Smart AC` och ha `Midea AC LAN` i åtanke som alternativ. `Midea Smart AC` är mer inriktad på luftkonditioneringsapparater och dokumenterar uttryckligen de aktuella PortaSplit-funktionerna.

Det är inte meningsfullt att köra båda integrationerna samtidigt och permanent med samma enhet. Flera parallella anslutningar leder till statusproblem, onödig nätverkstrafik och svårförutsägbart beteende.

## Vad integrationen ger

Efter konfigurationen visas PortaSplit som en `climate`-entitet i Home Assistant. Beroende på firmware och integration är bland annat följande funktioner tillgängliga:

- Slå på och av
- Ställa in börtemperatur
- Läsa av aktuell rumstemperatur
- Kylning, avfuktning och enbart fläktdrift
- Ställa in fläkthastighet
- Styra swing-funktionen
- Eco- och boostläge
- Läsa av luftfuktighet
- Visa felkoder
- Läsa av energi- och effektvärden
- Visa kompressorvärden
- Aktivera utomhusenhetens tystläge

Vilka entiteter som faktiskt visas beror på modellen, firmwaren, det använda protokollet och respektive integration. `Midea Smart AC` läser av de funktioner som enheten rapporterar och döljer funktioner som modellen inte stöder. `Midea AC LAN` dokumenterar också omfattande klimatentiteter, däribland temperatur, luftfuktighet, aktuell effekt, total energi, kompressorfrekvens, pumpstatus och olika driftlägen, och anger egna metoder för avkodning av energidata för vissa PortaSplit-undertyper.

Alla visade mätvärden behöver inte vara korrekta. Särskilt energiförbrukning och effekt överförs i olika format för olika Midea-modeller. Om Home Assistant visar uppenbart felaktiga värden behöver den använda avkodningsmetoden vanligtvis anpassas, inte enheten repareras.

## Förutsättningar

Du behöver en Midea PortaSplit med Wi-Fi-funktion, ett 2,4 GHz-Wi-Fi, MSmartHome-appen, ett Midea-användarkonto, Home Assistant, HACS och nätverksåtkomst mellan Home Assistant och PortaSplit. PortaSplit bör först anslutas på vanligt sätt med MSmartHome-appen och först därefter läggas till i Home Assistant.

## Steg 1: Anslut PortaSplit till MSmartHome

1. Installera MSmartHome-appen.
2. Skapa ett Midea-konto eller logga in.
3. Sätt PortaSplit i Wi-Fi-parkopplingsläge.
4. Anslut enheten till 2,4 GHz-Wi-Fi.
5. Kontrollera att PortaSplit kan styras via appen.

Många IoT-enheter stöder fortfarande endast 2,4 GHz. Om routern använder samma SSID för 2,4 och 5 GHz fungerar konfigurationen vanligtvis ändå. Vid problem kan det hjälpa att tillfälligt tillhandahålla ett separat 2,4 GHz-Wi-Fi.

## Steg 2: Installera HACS

HACS är Home Assistant Community Store. Där kan du installera community-integrationer som inte ingår i Home Assistant Core. Efter HACS-installationen öppnar du HACS, går till integrationerna, söker efter `Midea Smart AC`, laddar ner integrationen och startar om Home Assistant. Alternativt kan du söka efter `Midea AC LAN`.

HACS förenklar installation och uppdateringar. Det gör dock inte en Custom Integration till en officiellt granskad Home Assistant-komponent. Den skillnaden är viktig ur säkerhetssynpunkt och behandlas längre ned.

## Steg 3: Lägg till Midea Smart AC

Efter omstarten går du via Inställningar, Enheter och tjänster och Lägg till integration till sökningen efter `Midea Smart AC`, och därefter till `Discover devices`. Integrationen kan antingen genomsöka hela det lokala nätverket eller rikta in sig på PortaSplits IP-adress.

När enheten hittas behöver integrationen för nyare V3-enheter region, Midea-konto, lösenord och enhets-ID samt den token och key som härleds från dessa. Molnregionen måste passa det använda kontot. Vid problem rekommenderar projektet att även prova de andra tillgängliga regionerna.

### Manuell konfiguration

Om den automatiska konfigurationen misslyckas kan enheten konfigureras manuellt. För `Midea Smart AC` krävs följande uppgifter:

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

För V3-enheter anger dokumentationen token som en 128-siffrig och key som en 64-siffrig hexadecimal teckensträng. Båda värdena är hemligheter och ska behandlas därefter. Den som inte vill hämta åtkomstuppgifterna via Discovery kan hämta dem med sitt eget konto via CLI `msmart-ng`.

## Använd PortaSplit säkert

Den som styr PortaSplit lokalt återtar en del av kontrollen från tillverkarens moln, men flyttar samtidigt ansvaret till sitt eget nätverk. Följande punkter gör att token och key orsakar liten skada även vid en incident och att enheten förblir korrekt isolerad.

### Token och key är hemligheter

Token och key autentiserar den lokala kommunikationen med enheten och ska behandlas som ett lösenord. För driften är det viktigaste: De hör inte hemma i loggar, okrypterade säkerhetskopior eller ett repository.

### Ingen portvidarebefordran till PortaSplit

Det vanligaste undvikbara misstaget vore att göra den lokala enhetsporten direkt åtkomlig från internet. En regel som denna skulle vara farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finns ingen god anledning att göra PortaSplit direkt åtkomlig från internet. Home Assistant finns redan i det lokala nätverket och fungerar som kontrollerande instans. Routern bör inte ha någon portvidarebefordran till PortaSplit, UPnP bör om möjligt begränsas eller inaktiveras, inkommande anslutningar bör blockeras som standard och ingen DMZ-frisläppning för enheten ska användas.

### Eget IoT-VLAN

Den bästa nätverksarkitekturen är ett separat IoT-nätverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit finns i IoT-VLAN:et. Home Assistant får selektivt komma åt enheten, men PortaSplit får inte godtyckligt komma åt datorer, NAS och andra interna system. En möjlig brandväggslogik:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under den första konfigurationen behöver enheten internetåtkomst till Midea-molnet. Efter lyckad lokal konfiguration kan du testa om utgående internetåtkomst kan blockeras. Sätt inte en permanent spärr direkt. Kontrollera först om den lokala styrningen fortfarande fungerar, om enheten förblir åtkomlig efter en omstart, om den klarar en routeromstart, om den fortfarande reagerar efter flera dagar, om MSmartHome-appen fortfarande behövs och om firmware-uppdateringar fortfarande erbjuds. Den som vill fortsätta använda molnet och firmware-uppdateringar kan tillfälligt tillåta utgående internetåtkomst och därefter blockera den igen.

### Nätverkssegmentering kan förhindra Discovery

Automatisk enhetsupptäckt bygger ofta på broadcast- eller multicast-trafik, och den routas normalt inte över VLAN-gränser. Home Assistant kanske därför inte hittar PortaSplit automatiskt, även om en vanlig IP-anslutning vore tillåten.

Då kan det hjälpa att tillfälligt konfigurera PortaSplit i samma VLAN som Home Assistant, ange enhetens IP-adress manuellt, använda en lämplig broadcast-reläfunktion eller definiera riktade brandväggsregler efter konfigurationen. Manuell konfiguration är ofta till och med det bättre alternativet ur säkerhetssynpunkt, eftersom ingen ytterligare broadcast-trafik behöver tillåtas mellan näten.

### Statisk DHCP-tilldelning

PortaSplit bör få en fast DHCP-tilldelning i routern:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservation är vanligtvis att föredra framför en statisk IP-adress som ställs in på enheten. Home Assistant hittar enheten tillförlitligt, brandväggsregler kan begränsas till en fast adress, felanalysen blir enklare och tilldelningen förblir stabil efter omstarter av router eller enhet. En brandväggsregel kan då formuleras mycket snävt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den port som faktiskt behövs ska verifieras utifrån integrationen och den egna enheten.

### Home Assistant som central förtroendeankare

Den som styr PortaSplit lokalt flyttar delvis förtroendet från Midea-molnet till Home Assistant. Om Home Assistant komprometteras kan en angripare eventuellt inte bara styra luftkonditioneringen utan hela det smarta hemmet.

Home Assistant bör därför uppdateras regelbundet, inte publiceras via oskyddad portvidarebefordran, skyddas med ett starkt och unikt lösenord, använda flerfaktorsautentisering, skapa krypterade säkerhetskopior, endast innehålla nödvändiga tillägg och inte tillåta onödig SSH-åtkomst från internet. För fjärråtkomst är VPN, Home Assistant Cloud eller en korrekt konfigurerad reverse proxy bättre alternativ än en enkel portvidarebefordran till port 8123.

### HACS och risken i leverantörskedjan

`Midea Smart AC` och `Midea AC LAN` är Custom Integrations. De körs i Home Assistant och får därmed omfattande åtkomst till dess körmiljö. En skadlig eller komprometterad integration skulle teoretiskt kunna läsa konfigurationsdata, hämta secrets, upprätta nätverksanslutningar, skanna enheter i det lokala nätverket, läsa tillstånd för andra entiteter, överföra data till externa system och påverka tillgängligheten för Home Assistant.

Det innebär inte att de nämnda integrationerna är skadliga. Båda projekten är offentligt tillgängliga, utvecklas aktivt och har en synlig community. Open source är dock ingen automatisk säkerhetsgaranti. Före installationen är det värt att åtminstone kontrollera om repositoryt underhålls aktivt, om det finns regelbundna releaser, hur många personer som bidrar till koden, om det finns öppna säkerhetsärenden, om maintainers eller repository-ägare nyligen har bytts ut, om HACS hänvisar till det förväntade repositoryt och om en uppdatering innehåller ovanligt stora eller oförklarliga ändringar.

Uppdateringar bör inte installeras blint direkt efter publicering. Särskilt för säkerhetskritiska system i smarta hem är det klokt att vänta några dagar och kontrollera release notes samt rapporterade problem.

### Skydda molnkontot

Så länge Midea-molnet används för konfiguration eller appstyrning förblir även Midea-kontot en del av säkerhetsmodellen. Det bör ha ett unikt lösenord som inte delas med andra tjänster, använda en lösenordshanterare, flerfaktorsautentisering om den erbjuds, gamla smartphones och sessioner ska tas bort, delade konton ska undvikas och det bör regelbundet kontrolleras vilka enheter som är registrerade i kontot.

Om Home Assistant-integrationen begär användarnamn och lösenord under konfigurationen bör du kontrollera om åtkomstuppgifterna endast används för engångshämtning av token eller lagras permanent. Utvecklarna av `Midea Smart AC` skriver att enheter efter konfigurationen inte kopplas till inbyggda integrationskonton och att token och key också kan hämtas manuellt via CLI med det egna kontot. Där det är möjligt är det egna kontot att föredra framför främmande eller integrerade samlingskonton.

### Blockera molnet eller inte?

Efter lyckad konfiguration uppstår frågan om PortaSplits internetåtkomst bör blockeras helt. Argument för en blockering är mindre telemetri, mindre beroende av externa tjänster, en mindre angreppsyta via tillverkarens moln, att enheten inte kan kontakta godtyckliga externa mål och mindre påverkan av molnsidiga ändringar.

Emot talar att MSmartHome-appen utanför hemnätverket kanske inte längre fungerar, att firmware-uppdateringar inte laddas ner, att tids- eller molnfunktioner kan sluta fungera, att ny inloggning eller återställning blir svårare och att vissa enheter reagerar oväntat efter en längre tid offline.

En pragmatisk ordning: Konfigurera enheten normalt, testa Home Assistant och appen, säkerhetskopiera token och konfiguration, blockera internetåtkomsten, starta om enheten och Home Assistant, observera under flera dagar och återaktivera vid behov internetåtkomsten endast tillfälligt.

### Firmware-uppdateringar: säkerhetsvinst eller integrationsrisk?

Firmware-uppdateringar är ett dilemma för IoT-enheter. De kan åtgärda kända sårbarheter, förbättra stabiliteten, modernisera säkerhetsmekanismer och ge nya funktioner. Men de kan också ändra lokala gränssnitt, bryta reverse-engineering-integrationer, göra token ogiltiga, inaktivera det lokala API:et och införa nya molnberoenden.

PortaSplit-firmwaren som levererades i januari 2026 innehöll exempelvis ett nytt tystläge för utomhusenheten som minskar ljudnivån med cirka 6 decibel. Community-integrationerna behövde först analysera och implementera detta, vilket dokumenterades i ett eget GitHub-ärende för PortaSplit.

Slutsatsen är: Förhindra inte firmware-uppdateringar generellt, kontrollera före en uppdatering om andra Home Assistant-användare rapporterar problem, säkerhetskopiera konfiguration och token i förväg, skapa en Home Assistant-säkerhetskopia och testa sedan den lokala styrningen fullständigt. Säkerhet innebär inte ”uppdatera aldrig”. Föråldrad firmware kan vara farligare än en tillfälligt inkompatibel integration.

### Debug-loggar innehåller känsliga data

Vid problem ber open source-projekt ofta om debug-loggar. Dokumentationen för `Midea AC LAN` visar hur loggning för de två relevanta komponenterna aktiveras:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Därefter kan loggarna laddas ner via Inställningar, System och Loggar. Beroende på integration och fel kan sådana loggar innehålla lokala IP-adresser, enhets-ID, serienummer, modellidentifierare, molnsvar, kontoinformation, token eller delar av dem, nätverkspaket samt tidsstämplar och användningsmönster. Innan de laddas upp till ett offentligt GitHub-ärende bör de därför granskas och känsliga värden maskeras.

När felsökningen är klar ska debug-loggningen tas bort igen. Permanent aktiverad debug-loggning ökar inte bara lagringsförbrukningen utan också mängden känslig information i säkerhetskopiorna.

### Vad Midea själva säger om säkerhet

Midea marknadsför sitt SmartHome-ekosystem med inriktning mot flera säkerhets- och dataskyddsstandarder, där EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling och kraven i EU:s Radio Equipment Directive nämns. Det är positiva signaler, men inte ett uttalande om hur varje enskild PortaSplit-firmware, varje molnändpunkt och varje lokalt API faktiskt är implementerat. Certifierings- och marknadsföringspåståenden ersätter inte teknisk granskning av den konkreta enheten.

På samma sätt vore det fel att dra slutsatsen från en community-integrations varning att PortaSplit generellt är osäker. Det beskrivna problemet gäller arkitekturen för långlivade token och hur de används av inofficiella klienter.

### Risk per scenario

| Scenario | Risk | Motivering |
| --- | --- | --- |
| Normalt hemnätverk utan portvidarebefordran | hanterbar | En angripare måste först få åtkomst till Wi-Fi, Home Assistant eller en säkerhetskopia. |
| Platt hemnätverk med många osäkra IoT-enheter | medel | En komprometterad annan IoT-enhet kan nå PortaSplit eller Home Assistant i samma nätverk. |
| PortaSplit direkt åtkomlig från internet | hög | Enheten ska aldrig publiceras via portvidarebefordran. |
| Token och key offentligt på GitHub | hög | Hemligheterna anses komprometterade; det är inte garanterat att de kan återkallas. |
| Separat IoT-VLAN, restriktiv brandvägg, lokal styrning | jämförelsevis låg | Även vid en sårbarhet i enheten är rörelsefriheten i nätverket kraftigt begränsad. |

## Säkerhetskopiera konfigurationen

Att säkerhetskopiera token, key och konfiguration är det viktigaste engångssteget: När moln-token-gränssnitten väl har stängts är en säkerhetskopia det enda sättet att göra en ny konfiguration. `Midea AC LAN` skapar efter lyckad konfiguration en JSON-konfigurationsfil för V3-enheter. Den dokumenterade sökvägen är:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID:t som filnamn:

```text
<device-id>.json
```

Den här filen är ingen vanlig textanteckning. Den kan innehålla enhets-ID, serienummer, IP-adress, token, key, protokollinformation samt moln- och enhetsparametrar. Därför gäller följande:

- Ladda inte upp den till ett offentligt GitHub-repository.
- Publicera den inte i forum.
- Dela den inte som en omaskerad skärmbild.
- Skicka den inte via okrypterad e-post.

Även ett privat Git-repository är inte automatiskt rätt lagringsplats, eftersom hemligheter finns kvar i Git-historiken även om de senare tas bort från den aktuella filen. Bättre alternativ är en krypterad säkerhetskopia, en lösenordshanterare med filbilaga, en krypterad NAS-säkerhetskopia, ett krypterat offlinemedium eller ett krypterat arkiv med separat lagrat lösenord.

För säkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visa filen:

```bash
cat <enhets-id>.json
```

Filen bör inte överföras via en offentlig webbtjänst vid kopiering. Ett krypterat arkiv, som sedan förs över till en krypterad säkerhetskopia, är bättre:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filerna i `.storage` bör inte redigeras manuellt. Utvecklaren rekommenderar uttryckligen att JSON-filen varken raderas eller ändras direkt vid problem, utan att den döps om och säkerhetskopieras före ändringar.

En fullständig Home Assistant-säkerhetskopia innehåller också dessa filer. En separat kopia är ändå klok, eftersom Home Assistant-säkerhetskopior kan skadas, en återställning kan skriva över integrationen, filen kan behövas specifikt för en framtida ny konfiguration och en säkerhetskopia aldrig bara bör finnas på samma system.

## Ta bort secrets från ett publicerat Git-repository

Om en JSON-fil av misstag har publicerats på GitHub räcker det inte att radera den normalt och göra en ny commit. Filen kan fortfarande hämtas från Git-historiken. Minst följande steg krävs:

1. Gör repositoryt privat omedelbart, om möjligt.
2. Ta bort filen från hela Git-historiken.
3. Beakta GitHub-cachar och forks.
4. Behandla token som komprometterad.
5. Ta bort enheten från Midea-kontot och anslut den på nytt, om detta skapar nya nycklar.
6. Konfigurera Home Assistant-integrationen på nytt.
7. Ändra Midea-kontots lösenord om även åtkomstuppgifterna var berörda.

Om ny parkoppling faktiskt skapar en ny token varierar beroende på enhet och molnarkitektur. Man bör inte förlita sig på att ett byte av kontolösenord automatiskt gör den lokala enhetstoken ogiltig.

## Användbara automationer

Efter en lyckad integration kan PortaSplit drivas betydligt smartare. Entity-ID:n måste anpassas till den egna installationen.

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

Den önskade kommunikationsriktningen ser då ut så här:

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

## Rekommenderat driftläge

Midea PortaSplit kan integreras förvånansvärt väl i Home Assistant. Efter lyckad konfiguration kan den styras lokalt och användas i automationer, vilket eliminerar en stor del av molnberoendet i den dagliga driften.

Ur säkerhetssynpunkt är integrationen försvarbar om några grundregler följs: ingen portvidarebefordran, håll token och key hemliga, kryptera säkerhetskopior, granska debug-loggar före publicering, säkra Home Assistant, segmentera IoT-enheter, begränsa utgående internetåtkomst till det nödvändiga och installera inte firmware- och HACS-uppdateringar blint. Med detta upplägg förblir PortaSplit en kraftfull luftkonditioneringsapparat och blir samtidigt en komponent som kan integreras på ett meningsfullt sätt i ett lokalt styrt smart hem.

## Källor

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: enhetstyper som stöds `0xAC` och `0xCC`, PortaSplit med ”Out Silent Mode”, molnanvändning för att hämta token och key för V3-enheter, manuell konfiguration och standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN`: enhetsklasser som stöds, längre TCP-anslutning för statussynkronisering och lägsta version Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentation av klimatentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entiteter och attribut för luftkonditioneringsapparater, däribland effekt, total energi, kompressorfrekvens och avkodningsmetoder för energivärden för enskilda undertyper.

4.  [midea_ac_lan: Anvisningar för debug och konfiguration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): lagring av enhetskonfiguration under `/config/.storage/midea_ac_lan/`, rekommendation att säkerhetskopiera i stället för att radera JSON-filen samt logger-konfigurationen för debug-loggar.

5.  [Issue 779: PortaSplits Out Silent Mode](https://github.com/wuwentao/midea_ac_lan/issues/779): begäran om stöd för utomhusenhetens tystläge som infördes med firmware-uppdateringen i januari 2026 och minskar ljudnivån med cirka 6 decibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): tillverkaruppgifter om säkerhets- och dataskyddsstandarderna EN 303 645, PSTI, NIST, GDPR och RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation och hantering av Custom Integrations som inte ingår i Home Assistant Core.
