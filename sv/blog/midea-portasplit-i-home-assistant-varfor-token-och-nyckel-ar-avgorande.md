---
title: "Skydda Midea PortaSplit i Home Assistant: token, nyckel och hemnätverk"
navTitle: "Skydda PortaSplit"
description: "PortaSplits token och nyckel kommer från Midea-molnet och löper aldrig ut. Så skyddar du värdena, isolerar enheten i hemnätverket och håller Home Assistant, integrationen och firmware kontrollerat uppdaterade."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "14 min lästid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-v2-cloud-api-portasplit-home-assistant
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard.png"
slug: "midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande"
translationOf: "midea-portasplit-home-assistant-absichern"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: c72a9e3147727e1ec8bb37ab078eb3a73c3cc5a4c92a38fc4b3f845966e2f405
translatedAt: 2026-10-09T10:59:01.886Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/sv/blog/midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande
---

<aside class="article-update">
  <p class="article-update__label">Vad PortaSplit-ägare bör göra nu</p>
  <p>Home Assistant hämtar PortaSplits token och nyckel via privata molngränssnitt vid konfigurationen. Projektet Midea AC LAN har varnat för möjliga ändringar sedan den 19 maj 2025; något avvecklingsdatum från tillverkaren har inte dokumenterats. För ägare innebär detta:</p>
  <ol>
    <li><strong>Säkerhetskopiera token, nyckel och konfiguration krypterat.</strong> Om hämtningen inte längre fungerar senare är säkerhetskopian den enda vägen till återställning.</li>
    <li><strong>Koppla inte från enheten i onödan.</strong> Fabriksåterställning, borttagning från Midea-kontot eller byte av WLAN-modul kräver att token hämtas på nytt.</li>
    <li><strong>Isolera PortaSplit i hemnätverket.</strong> Ingen portvidarebefordran, eget IoT-VLAN, åtkomst endast från Home Assistant.</li>
  </ol>
</aside>

Den lokala styrningen av Midea PortaSplit bygger på två enhetsspecifika värden: token och nyckel. De autentiserar anslutningen mellan Home Assistant och enheten och kan för närvarande endast hämtas via Midea-molnet. Detta medför två uppgifter: att säkra värdena så att en ny konfiguration förblir möjlig utan molnet, och att driva både enheten och Home Assistant på ett sätt som begränsar skadorna även vid en incident.

Serien består av tre delar: [Del 1](/blog/midea-portasplit-home-assistant) beskriver konfigurationen fram till instrumentpanelen, denna del behandlar skyddet och [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) bakgrunden till varningarna om moln-API:et.

![Home Assistant-instrumentpanel för Midea PortaSplit i kyldrift: nyckeltal högst upp, termostat på 22 °C, grafer för rumstemperatur, effektförbrukning, dagsenergi, kompressorfrekvens, kompressordrift och fläkthastighet, samt tekniska värden och status nedanför.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Var token och nyckel kommer från

För enheter med V3-protokollet accepterar PortaSplit lokala kommandon endast med token och nyckel. Värdena skapas inte av enheten utan av Midea-molnet; även den officiella appen hämtar dem därifrån. Community-integrationerna har återskapat detta molnanrop: de loggar in via samma slutpunkter som appen, får token och nyckel och sparar båda lokalt. Därefter krävs ingen molnanslutning för den löpande driften.

Det finns ingen dokumenterad lokal parningsmekanism som lämnar ut värdena utan molnet. Teoretiskt kan de läsas ut ur appen, exempelvis genom reverse engineering eller instrumentering under körning, men för en enskild användare är det omständligt och ingen ersättning för molnhämtningen. Om slutpunkten försvinner försvinner därför också möjligheten att hämta dem.

Projektet `Midea AC LAN` varnar i sin README för att Midea stegvis stänger token-gränssnitten; integrationen växlar därför mellan moln. Redan konfigurerade enheter fortsätter att fungera lokalt, medan nya enheter och nya konfigurationer skulle påverkas. Detta är inte någon bindande färdplan från Midea. I juni 2026 visade det sig dessutom att det förment stängda SmartHome-token-API:et fortfarande fungerade; community-bibliotekets begäran var bara ofullständig. Bedömningen av varningen och de olika beteckningarna ”V2” finns i [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Vad token och nyckel möjliggör

Token och nyckel har ingen utgångstid. Enligt `Midea AC LAN` ansågs klientkommunikationen ursprungligen vara tillräckligt skyddad, vilket är anledningen till att molnet utfärdade token som aldrig löper ut. I sig är detta ingen sårbarhet; det blir problematiskt om värdena hamnar i loggar eller oskyddade säkerhetskopior, hamnar hos tredje part eller varken kan återkallas eller roteras.

Den som har token och nyckel och kan nå enheten i nätverket kan autentisera sig mot PortaSplit, läsa statusinformation, slå på och av den, byta driftläge och ändra börtemperaturen. Värdena i sig möjliggör ingen attack från internet; angriparen behöver dessutom en nätverksanslutning till enheten. Token och nyckel ska därför behandlas som ett lösenord, och nätverket bör om möjligt endast tillåta denna anslutning för Home Assistant.

Community-integrationen angriper inte luftkonditioneringsenheten. Den implementerar ett proprietärt protokoll som har kartlagts genom reverse engineering. Risken uppstår genom att långlivade hemligheter lagras utanför den avsedda appen.

## Säkerhetskopiera token, nyckel och konfiguration

Säkerhetskopieringen av token, nyckel och konfiguration är det viktigaste engångssteget: när molnets token-gränssnitt väl har stängts är en säkerhetskopia den enda vägen till en ny konfiguration. `Midea AC LAN` sparar en JSON-konfigurationsfil för V3-enheter efter en lyckad konfiguration. Den dokumenterade sökvägen är:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhetens ID som filnamn:

```text
<device-id>.json
```

Den här filen är ingen vanlig textanteckning. Den kan innehålla enhets-ID, serienummer, IP-adress, token, nyckel, protokollinformation samt moln- och enhetsparametrar. Därför gäller följande:

- Ladda inte upp den till ett offentligt GitHub-repository.
- Publicera den inte i forum.
- Dela den inte som en ocensurerad skärmbild.
- Skicka den inte via okrypterad e-post.

Inte heller ett privat Git-repository är automatiskt rätt lagringsplats, eftersom hemligheter finns kvar i Git-historiken även om de senare raderas ur den aktuella filen. Lämpligare alternativ är en krypterad säkerhetskopia, en lösenordshanterare med filbilaga, en krypterad NAS-säkerhetskopia, ett krypterat offlinemedium eller ett krypterat arkiv med lösenordet lagrat separat.

För säkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visa filen:

```bash
cat <device-id>.json
```

Vid kopiering bör filen inte överföras via en offentlig webbtjänst. Ett bättre alternativ är ett krypterat arkiv som sedan förs över till en krypterad säkerhetskopia:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filerna i `.storage` bör inte redigeras manuellt. Utvecklaren rekommenderar uttryckligen att JSON-filen varken raderas eller ändras direkt vid problem, utan att den byter namn och säkerhetskopieras före ändringar.

En fullständig Home Assistant-säkerhetskopia innehåller också dessa filer. En separat kopia är ändå klok, eftersom Home Assistant-säkerhetskopior kan skadas, en återställning kan skriva över integrationen, filen kan behövas specifikt för en senare ny konfiguration och en säkerhetskopia aldrig bör finnas enbart på samma system.

### Ta bort hemligheter från ett publicerat Git-repository

Om en JSON-fil av misstag har publicerats på GitHub räcker det inte att radera den normalt och göra en ny commit. Filen kan fortfarande hämtas från Git-historiken. Minst följande steg krävs:

1. Gör omedelbart repositoryt privat, om möjligt.
2. Ta bort filen ur hela Git-historiken.
3. Ta hänsyn till GitHub-cachar och forkade repositoryn.
4. Behandla token som komprometterad.
5. Ta bort enheten från Midea-kontot och anslut den på nytt, om detta skapar nya nycklar.
6. Konfigurera Home Assistant-integrationen på nytt.
7. Byt lösenordet för Midea-kontot om även inloggningsuppgifterna berördes.

Huruvida den förnyade parkopplingen faktiskt skapar en ny token varierar beroende på enhet och molnarkitektur. Man bör inte förlita sig på att ett byte av kontolösenord automatiskt gör den lokala enhetstoken ogiltig.

## Isolera PortaSplit i nätverket

### Ingen portvidarebefordran till PortaSplit

Det vanligaste undvikbara misstaget vore att göra den lokala enhetsporten direkt nåbar från internet. En regel som denna skulle vara farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finns ingen god anledning att göra PortaSplit direkt nåbar från internet. Home Assistant finns redan i det lokala nätverket och fungerar som en kontrollerande instans. Routern bör inte ha någon portvidarebefordran till PortaSplit, UPnP bör begränsas eller avaktiveras där det är möjligt, inkommande anslutningar bör blockeras som standard och ingen DMZ-regel bör användas för enheten.

### Eget IoT-VLAN

Den bästa nätverksarkitekturen är ett separat IoT-nätverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit finns i IoT-VLAN:et. Home Assistant får ha riktad åtkomst till enheten, men PortaSplit får inte ha godtycklig åtkomst till datorer, NAS och andra interna system. En möjlig brandväggslogik:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Under den första konfigurationen behöver enheten internetåtkomst till Midea-molnet. Efter en lyckad lokal konfiguration kan man testa om utgående internetåtkomst kan blockeras. En permanent blockering bör dock inte sättas direkt. Kontrollera först om den lokala styrningen fortfarande fungerar, om enheten förblir nåbar efter en omstart, om den klarar en routeromstart, om den fortfarande reagerar efter flera dagar, om MSmartHome-appen fortfarande behövs och om firmware-uppdateringar fortfarande erbjuds. Den som vill fortsätta använda molnet och firmware-uppdateringar kan tillfälligt tillåta utgående internetåtkomst och sedan blockera den igen.

### Nätverkssegmentering kan hindra upptäckt

Automatisk enhetsupptäckt bygger ofta på broadcast- eller multicast-trafik, och den routas normalt inte över VLAN-gränser. Home Assistant kanske därför inte hittar PortaSplit automatiskt, även om en vanlig IP-anslutning skulle vara tillåten.

Då kan man tillfälligt konfigurera PortaSplit i samma VLAN som Home Assistant, ange enhetens IP-adress manuellt, använda en lämplig broadcast-reläfunktion eller definiera riktade brandväggsregler efter konfigurationen. Manuell konfiguration är ofta till och med det bättre alternativet ur säkerhetssynpunkt, eftersom ingen ytterligare broadcast-trafik mellan näten behöver tillåtas.

### Statisk DHCP-tilldelning

PortaSplit bör få en fast DHCP-tilldelning i routern:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservation är oftast att föredra framför en statisk IP-adress som ställs in i enheten. Home Assistant hittar enheten tillförlitligt, brandväggsregler kan begränsas till en fast adress, felsökningen blir enklare och tilldelningen förblir stabil efter omstarter av router eller enhet. En brandväggsregel kan då formuleras mycket snävt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den port som faktiskt behövs måste verifieras utifrån integrationen och den egna enheten.

## Skydda Home Assistant och integrationer

### Home Assistant som central förtroendeankare

Den som styr PortaSplit lokalt flyttar delvis förtroendet från Midea-molnet till Home Assistant. Om Home Assistant komprometteras kan en angripare i vissa fall inte bara kontrollera luftkonditioneringen utan hela smarta hemmet.

Home Assistant bör därför uppdateras regelbundet, inte publiceras via oskyddad portvidarebefordran, skyddas med ett starkt och unikt lösenord, använda flerfaktorsautentisering, skapa krypterade säkerhetskopior, endast innehålla nödvändiga tillägg och inte tillåta onödig SSH-åtkomst från internet. För fjärråtkomst är ett VPN, Home Assistant Cloud eller en korrekt konfigurerad reverse proxy bättre alternativ än en enkel portvidarebefordran till port 8123.

### HACS och risken i leveranskedjan

`Midea Smart AC` och `Midea AC LAN` är anpassade integrationer. De körs inom Home Assistant och får därmed omfattande åtkomst till dess körmiljö. En skadlig eller komprometterad integration skulle teoretiskt kunna läsa konfigurationsdata, hämta hemligheter, upprätta nätverksanslutningar, skanna enheter i det lokala nätverket, läsa tillstånd för andra entiteter, överföra data till externa system och påverka tillgängligheten för Home Assistant.

Det betyder inte att de nämnda integrationerna är skadliga. Båda projekten är offentligt granskbara, utvecklas aktivt och har en synlig community. Open source är dock ingen automatisk säkerhetsgaranti. Före installationen är det minst värt att kontrollera om repositoryt underhålls aktivt, om det finns regelbundna releaser, hur många personer som bidrar till koden, om det finns öppna säkerhetsärenden, om underhållare eller repositoryägare nyligen har bytts, om HACS pekar på det förväntade repositoryt och om en uppdatering innehåller ovanligt stora eller oförklarliga ändringar.

Uppdateringar bör inte installeras blint direkt efter publicering. Särskilt för säkerhetskritiska smarta hemsystem är det klokt att vänta några dagar och granska release notes samt rapporterade problem.

### Debuggloggar innehåller känsliga data

Vid problem begär open source-projekt ofta debug-loggar. Dokumentationen för `Midea AC LAN` visar hur loggning aktiveras för de två relevanta komponenterna:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Därefter kan loggarna laddas ner via Inställningar, System och Loggar. Beroende på integration och fel kan sådana loggar innehålla lokala IP-adresser, enhets-ID, serienummer, modellidentifierare, molnsvar, kontoinformation, token eller delar av dem, nätverkspaket samt tidsstämplar och användningsmönster. De bör därför granskas och känsliga värden maskeras innan de laddas upp till ett offentligt GitHub-ärende.

När felsökningen är klar ska debug-loggningen tas bort igen. Permanent aktiverad debug-loggning ökar inte bara lagringsförbrukningen, utan även mängden känslig information i säkerhetskopiorna.

## Moln och firmware

### Skydda molnkontot

Så länge Midea-molnet används för konfiguration eller appstyrning förblir även Midea-kontot en del av säkerhetsmodellen. Det ska ha ett unikt lösenord som inte delas med andra tjänster, en lösenordshanterare, flerfaktorsautentisering om den erbjuds, gamla smarttelefoner och sessioner ska tas bort, delade konton ska undvikas och man bör regelbundet kontrollera vilka enheter som är registrerade på kontot.

Om Home Assistant-integrationen begär användarnamn och lösenord under konfigurationen bör det kontrolleras om inloggningsuppgifterna bara används för engångshämtningen av token eller lagras permanent. Utvecklarna av `Midea Smart AC` skriver att enheter efter konfigurationen inte länkas till inbyggda integrationskonton och att token och nyckel även kan hämtas manuellt via det egna kontot med CLI. Där det är möjligt är det egna kontot att föredra framför främmande eller integrerade samlingskonton.

### Blockera molnet eller inte?

Efter en lyckad konfiguration uppstår frågan om PortaSplits internetåtkomst bör blockeras helt. Argument för en blockering är mindre telemetri, lägre beroende av externa tjänster, en mindre attackyta via tillverkarens moln, att enheten inte kan kontakta godtyckliga externa mål och att molnbaserade ändringar får mindre effekt.

Argument emot är att MSmartHome-appen kanske inte längre fungerar utanför hemnätverket, att firmware-uppdateringar inte kan laddas ner, att tids- eller molnfunktioner kan sluta fungera, att ny inloggning eller återställning blir svårare och att vissa enheter reagerar oväntat efter lång tid offline.

En pragmatisk ordning: konfigurera enheten normalt, testa Home Assistant och appen, säkerhetskopiera token och konfiguration, blockera internetåtkomsten, starta om enheten och Home Assistant, observera under flera dagar och återaktivera internetåtkomsten endast tillfälligt vid behov.

### Firmware-uppdateringar: säkerhetsvinst eller integrationsrisk?

Firmware-uppdateringar är ett dilemma för IoT-enheter. De kan stänga kända sårbarheter, förbättra stabiliteten, modernisera säkerhetsmekanismer och ge nya funktioner. Men de kan också ändra lokala gränssnitt, bryta reverse-engineerade integrationer, ogiltigförklara token, avaktivera det lokala API:et och införa nya molnberoenden.

PortaSplit-firmware som levererades i januari 2026 gav exempelvis ett nytt tyst läge för utomhusenheten, som minskar ljudnivån med cirka 6 decibel. Community-integrationerna behövde först kartlägga och implementera detta, vilket dokumenterades i ett eget GitHub-ärende för PortaSplit.

Slutsatsen är: förhindra inte firmware-uppdateringar generellt, utan kontrollera före en uppdatering om andra Home Assistant-användare rapporterar problem, säkerhetskopiera konfiguration och token i förväg, skapa en Home Assistant-säkerhetskopia och testa den lokala styrningen fullständigt efter uppdateringen. Säkerhet betyder inte ”uppdatera aldrig”. Föråldrad firmware kan vara farligare än en tillfälligt inkompatibel integration.

### Vad Midea själv säger om säkerhet

Midea marknadsför sitt SmartHome-ekosystem med en inriktning mot flera säkerhets- och dataskyddsstandarder, bland annat EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling och kraven i EU:s Radio Equipment Directive. Det är positiva signaler, men säger inget om hur varje enskild PortaSplit-firmware, varje molnslutpunkt och varje lokalt API faktiskt är implementerat. Certifierings- och marknadsföringspåståenden ersätter inte en teknisk granskning av den konkreta enheten.

Det vore lika fel att dra slutsatsen från en community-integrations varning att PortaSplit generellt är osäker. Det beskrivna problemet rör arkitekturen för långlivade token och hur de används av inofficiella klienter.

## Risk efter scenario

| Scenario | Risk | Motivering |
| --- | --- | --- |
| Normalt hemnätverk utan portvidarebefordran | hanterbar | En angripare behöver först få åtkomst till WLAN, Home Assistant eller en säkerhetskopia. |
| Platt hemnätverk med många osäkra IoT-enheter | medel | En komprometterad annan IoT-enhet kan nå PortaSplit eller Home Assistant i samma nätverk. |
| PortaSplit direkt nåbar från internet | hög | Enheten bör aldrig publiceras via portvidarebefordran. |
| Token och nyckel offentligt på GitHub | hög | Hemligheterna betraktas som komprometterade; det är inte garanterat att de kan återkallas. |
| Separat IoT-VLAN, restriktiv brandvägg, lokal styrning | jämförelsevis låg | Även vid en sårbarhet i enheten är rörelsefriheten i nätverket kraftigt begränsad. |

## Checklista

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

Önskad kommunikationsriktning:

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

Med denna drift är den lokala styrningen försvarbar ur säkerhetssynpunkt: token och nyckel förblir hemliga och säkerhetskopierade, enheten är endast nåbar för Home Assistant och uppdateringar av firmware och integration installeras kontrollerat.

## Källor

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` med ”Important Notice” (sedan 19 maj 2025, uppdaterad 14 juli 2025), motiveringen om token som inte löper ut och beskrivningen av den molnbaserade token-hämtningen.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: molnbaserad hämtning av token och nyckel för V3-enheter, lokal lagring av värdena, standardport 6444.

3.  [midea_ac_lan: anvisningar för debug och konfiguration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): lagring av enhetskonfigurationen under `/config/.storage/midea_ac_lan/`, rekommendation att säkerhetskopiera i stället för att radera JSON-filen samt logger-konfigurationen för debug-loggar.

4.  [Issue 779: PortaSplits tysta utomhusläge](https://github.com/wuwentao/midea_ac_lan/issues/779): begäran om stöd för det tysta läge för utomhusenheten som infördes med firmware-uppdateringen i januari 2026 och som minskar ljudnivån med cirka 6 decibel.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): tillverkarens information om säkerhets- och dataskyddsstandarderna EN 303 645, PSTI, NIST, GDPR och RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation och hantering av anpassade integrationer som inte ingår i Home Assistant Core.
