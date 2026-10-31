---
title: "Säkra Midea PortaSplit i Home Assistant: token, nyckel och hemnätverk"
navTitle: "Säkra PortaSplit"
description: "PortaSplits token och nyckel kommer från Midea-molnet och upphör aldrig att gälla. Så säkrar du värdena, isolerar enheten i hemnätverket och håller Home Assistant, integration och firmware kontrollerat uppdaterade."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "14 min läsning"
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
translationSourceHash: d9e59ab5647df261814123d662df93c68c9f6cd35c20cec518de0a9cf54d6e22
translatedAt: 2026-10-31T03:39:43.762Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande
---

<aside class="article-update">
  <p class="article-update__label">Vad PortaSplit-ägare bör göra nu</p>
  <p>Home Assistant hämtar PortaSplits token och nyckel via privata molngränssnitt vid installationen. Projektet Midea AC LAN har varnat för möjliga förändringar sedan den 19 maj 2025; något datum för tillverkarens avstängning är inte dokumenterat. För ägare innebär det:</p>
  <ol>
    <li><strong>Säkerhetskopiera token, nyckel och konfiguration krypterat.</strong> Om hämtningen inte längre fungerar senare är säkerhetskopian den enda vägen till återställning.</li>
    <li><strong>Koppla inte från enheten i onödan.</strong> Fabriksåterställning, borttagning från Midea-kontot eller byte av Wi-Fi-modul kräver att en ny token hämtas.</li>
    <li><strong>Isolera PortaSplit i hemnätverket.</strong> Ingen portvidarebefordran, eget IoT-VLAN, åtkomst endast från Home Assistant.</li>
  </ol>
</aside>

Den lokala styrningen av Midea PortaSplit bygger på två enhetsspecifika värden: token och nyckel. De autentiserar anslutningen mellan Home Assistant och enheten och kan för närvarande endast hämtas via Midea-molnet. Det leder till två uppgifter: att säkra värdena så att ominstallation förblir möjlig utan molnet och att driva enheten samt Home Assistant så att värdena orsakar liten skada även vid en incident.

Serien består av tre delar: [Del 1](/blog/midea-portasplit-home-assistant) beskriver installationen fram till instrumentpanelen, denna del säkerheten och [Del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) bakgrunden till varningarna om moln-API:et.

![Home Assistant-instrumentpanel för Midea PortaSplit i kyldrift: nyckeltal överst, termostat på 22 °C, historik för rumstemperatur, effektförbrukning, energi per dag, kompressorfrekvens, kompressordrift och fläkthastighet samt tekniska värden och status nedanför.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Var token och nyckel kommer ifrån

På enheter med V3-protokollet accepterar PortaSplit lokala kommandon endast med token och nyckel. Värdena skapas inte av enheten utan av Midea-molnet; även den officiella appen hämtar dem därifrån. Community-integrationerna har återskapat detta molnanrop: de loggar in via samma ändpunkter som appen, får token och nyckel och lagrar båda lokalt. Därefter behövs ingen molnanslutning för den löpande driften.

Det finns ingen dokumenterad lokal parkopplingsmekanism som lämnar ut värdena utan molnet. Teoretiskt kan de läsas ut ur appen, exempelvis genom reverse engineering eller instrumentering vid körning; för den enskilda användaren är det omständligt och ingen ersättning för molnhämtningen. Om ändpunkten försvinner försvinner därför även möjligheten att hämta dem.

Projektet `Midea AC LAN` varnar i sin README för att Midea i framtiden stegvis kan stänga token-gränssnitten. Sedan README-filen reviderades den 3 september 2026 hämtar integrationen token och nyckel via Meiju- och SmartHome-molnen. Enligt README fungerar detta endast med ett personligt konto som enheten är kopplad till; via NetHome Plus går det för närvarande inte att hämta tokens. Sedan den 9 september 2026 påpekar README-filen dessutom att ett SmartHome-konto som migrerats från en annan app inte levererar några tokens; det krävs ett nyregistrerat konto med nyligen tillagda enheter. Redan konfigurerade enheter fortsätter att fungera lokalt, medan nya enheter och ominstallationer skulle påverkas. Detta är inte en bindande färdplan från Midea. I juni 2026 visade det sig dessutom att det förment stängda SmartHome-token-API:et fortfarande fungerade; community-bibliotekets begäran var bara ofullständig. En bedömning av varningen och de olika beteckningarna ”V2” finns i [Del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Vad token och nyckel möjliggör

Token och nyckel har ingen utgångstid. Enligt `Midea AC LAN` ansågs klientkommunikationen ursprungligen vara tillräckligt skyddad, vilket gjorde att molnet utfärdade tokens som aldrig upphör att gälla. I sig är detta ingen sårbarhet; det blir problematiskt om värdena hamnar i loggar eller oskyddade säkerhetskopior, kommer i tredje parts händer eller varken kan återkallas eller roteras.

Den som har token och nyckel och når enheten i nätverket kan autentisera sig mot PortaSplit, läsa statusinformation, slå på och av den, växla driftlägen och ändra börtemperaturen. Värdena ensamma möjliggör ingen attack från internet; angriparen behöver dessutom en nätverksanslutning till enheten. Token och nyckel ska därför behandlas som ett lösenord, och nätverket bör om möjligt endast tillåta denna anslutning för Home Assistant.

Community-integrationen angriper inte luftkonditioneringsenheten. Den implementerar ett proprietärt protokoll som har kartlagts genom reverse engineering. Risken uppstår genom att långlivade hemligheter lagras utanför den avsedda appen.

## Säkerhetskopiera token, nyckel och konfiguration

Säkerhetskopieringen av token, nyckel och konfiguration är det viktigaste engångssteget: när molnets token-gränssnitt väl har stängts är en säkerhetskopia den enda vägen till ominstallation. `Midea AC LAN` sparar en JSON-konfigurationsfil för V3-enheter efter lyckad installation. Den dokumenterade sökvägen är:

```text
/config/.storage/midea_ac_lan/
```

Filen har enhets-ID:t som filnamn:

```text
<device-id>.json
```

Denna fil är ingen vanlig textanteckning. Den kan innehålla enhets-ID, serienummer, IP-adress, token, nyckel, protokollinformation samt moln- och enhetsparametrar. Därför gäller följande:

- Ladda inte upp den till ett offentligt GitHub-repository.
- Publicera den inte i forum.
- Dela den inte som en ocensurerad skärmbild.
- Skicka den inte via okrypterad e-post.

Inte heller ett privat Git-repository är automatiskt rätt lagringsplats, eftersom hemligheter blir kvar i Git-historiken även om de senare tas bort ur den aktuella filen. Bättre alternativ är en krypterad säkerhetskopia, en lösenordshanterare med filbilaga, en krypterad NAS-säkerhetskopia, ett krypterat offline-medium eller ett krypterat arkiv med lösenordet lagrat separat.

För säkerhetskopiering via Home Assistant-terminalen:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visa filen:

```bash
cat <device-id>.json
```

Vid kopiering bör filen inte överföras via en offentlig webbtjänst. Ett krypterat arkiv som därefter förs över till en krypterad säkerhetskopia är bättre:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Filerna i `.storage` bör inte redigeras manuellt. Utvecklaren rekommenderar uttryckligen att JSON-filen varken raderas eller ändras direkt vid problem, utan att den byter namn och säkerhetskopieras före ändringar.

En fullständig Home Assistant-säkerhetskopia innehåller också dessa filer. En separat kopia är ändå meningsfull, eftersom Home Assistant-säkerhetskopior kan skadas, en återställning kan skriva över integrationen, filen kan behövas specifikt för en senare ominstallation och en säkerhetskopia aldrig bara bör finnas på samma system.

### Ta bort hemligheter från ett publicerat Git-repository

Om en JSON-fil av misstag har publicerats på GitHub räcker det inte att radera den normalt och göra en ny commit. Filen går fortfarande att hämta från Git-historiken. Minst följande steg krävs:

1. Gör omedelbart repositoryt privat, om möjligt.
2. Ta bort filen från hela Git-historiken.
3. Ta hänsyn till GitHub-cachar och forkningar.
4. Behandla token som komprometterad.
5. Ta bort enheten från Midea-kontot och anslut den på nytt, om detta genererar nya nycklar.
6. Konfigurera Home Assistant-integrationen på nytt.
7. Ändra lösenordet för Midea-kontot om även inloggningsuppgifter berördes.

Huruvida en ny parkoppling faktiskt skapar en ny token varierar beroende på enhet och molnarkitektur. Man bör inte förlita sig på att ett ändrat kontolösenord automatiskt ogiltigförklarar den lokala enhetstoken.

## Isolera PortaSplit i nätverket

### Ingen portvidarebefordran till PortaSplit

Det vanligaste undvikbara misstaget vore att göra den lokala enhetsporten direkt åtkomlig från internet. En regel som denna vore farlig:

```text
Internet → TCP 6444 → PortaSplit
```

Det finns ingen god anledning att göra PortaSplit direkt åtkomlig från internet. Home Assistant finns redan i det lokala nätverket och fungerar som kontrollerande instans. Routern bör inte ha någon portvidarebefordran till PortaSplit, begränsa eller inaktivera UPnP där det är möjligt, blockera inkommande anslutningar som standard och inte använda någon DMZ-frisläppning för enheten.

### Eget IoT-VLAN

Den bästa nätverksarkitekturen är ett separat IoT-nätverk:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit finns i IoT-VLAN:et. Home Assistant får riktat åtkomst till enheten, men PortaSplit får inte godtyckligt komma åt datorer, NAS och andra interna system. En möjlig brandväggslogik:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Vid den första installationen behöver enheten internetåtkomst till Midea-molnet. Efter en lyckad lokal installation kan man testa om utgående internetåtkomst kan blockeras. En permanent blockering bör dock inte sättas omedelbart. Kontrollera först om den lokala styrningen fortfarande fungerar, om enheten förblir nåbar efter en omstart, om den klarar en routeromstart, om den fortfarande svarar efter flera dagar, om MSmartHome-appen fortfarande behövs och om firmwareuppdateringar fortfarande erbjuds. Den som vill fortsätta använda molnet och firmwareuppdateringar kan tillfälligt tillåta utgående internetåtkomst och sedan blockera den igen.

### Nätverkssegmentering kan förhindra discovery

Automatisk enhetssökning bygger ofta på broadcast- eller multicast-trafik, som normalt inte routas över VLAN-gränser. Home Assistant kanske därför inte hittar PortaSplit automatiskt trots att en vanlig IP-anslutning vore tillåten.

Då hjälper det att tillfälligt konfigurera PortaSplit i samma VLAN som Home Assistant, ange enhetens IP-adress manuellt, använda en lämplig broadcast-reläfunktion eller definiera riktade brandväggsregler efter installationen. Manuell konfiguration är ur säkerhetssynpunkt ofta till och med det bättre alternativet, eftersom ingen ytterligare broadcast-trafik mellan nätverken behöver tillåtas.

### Statisk DHCP-tilldelning

PortaSplit bör få en fast DHCP-tilldelning i routern:

```text
PortaSplit → 192.168.30.25
```

En DHCP-reservation är oftast att föredra framför en statisk IP-adress som anges i enheten. Home Assistant hittar enheten tillförlitligt, brandväggsregler kan begränsas till en fast adress, felsökningen blir enklare och tilldelningen förblir stabil efter omstarter av router eller enhet. En brandväggsregel kan därmed formuleras mycket snävt:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Den port som faktiskt behövs måste verifieras utifrån integrationen och den egna enheten.

## Säkra Home Assistant och integrationer

### Home Assistant som centralt förtroendeankare

Den som styr PortaSplit lokalt flyttar delvis förtroendet från Midea-molnet till Home Assistant. Om Home Assistant komprometteras kan en angripare under vissa omständigheter kontrollera inte bara luftkonditioneringen utan hela det smarta hemmet.

Home Assistant bör därför uppdateras regelbundet, inte publiceras via oskyddad portvidarebefordran, skyddas med ett starkt och unikt lösenord, använda flerfaktorsautentisering, skapa krypterade säkerhetskopior, endast innehålla nödvändiga tillägg och inte tillåta onödig SSH-åtkomst från internet. För fjärråtkomst är VPN, Home Assistant Cloud eller en korrekt konfigurerad reverse proxy bättre alternativ än enkel portvidarebefordran till port 8123.

### HACS och risker i leveranskedjan

`Midea Smart AC` och `Midea AC LAN` är Custom Integrations. De körs i Home Assistant och får därmed omfattande åtkomst till dess körmiljö. En skadlig eller komprometterad integration skulle teoretiskt kunna läsa konfigurationsdata, hämta hemligheter, upprätta nätverksanslutningar, skanna enheter i det lokala nätverket, läsa tillstånd för andra entiteter, överföra data till externa system och försämra tillgängligheten för Home Assistant.

Det betyder inte att de nämnda integrationerna är skadliga. Båda projekten är offentligt granskbara, utvecklas aktivt och har en synlig community. Open source är dock ingen automatisk säkerhetsgaranti. Före installationen är det åtminstone värt att kontrollera om repositoryt underhålls aktivt, om det finns regelbundna releaser, hur många personer som bidrar till koden, om det finns öppna säkerhetsärenden, om underhållare eller repositoryägare nyligen har bytts, om HACS hänvisar till det förväntade repositoryt och om en uppdatering innehåller ovanligt stora eller oförklarliga ändringar.

Uppdateringar bör inte installeras blint direkt efter publicering. Särskilt för säkerhetskritiska smarta hem-system är det klokt att vänta några dagar och granska release notes samt rapporterade problem.

### Debuggloggar innehåller känsliga data

Vid problem begär open source-projekt ofta debuggloggar. Dokumentationen för `Midea AC LAN` visar hur loggning aktiveras för de två relevanta komponenterna:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Därefter kan loggarna hämtas via Inställningar, System och Loggar. Beroende på integration och fel kan sådana loggar innehålla lokala IP-adresser, enhets-ID, serienummer, modellidentifierare, molnsvar, kontoinformation, token eller delar av den, nätverkspaket samt tidsstämplar och användningsmönster. Innan de laddas upp till ett offentligt GitHub-ärende bör de därför granskas och känsliga värden maskeras.

När felsökningen är klar ska debug-loggningen tas bort igen. Permanent aktiverad debug-loggning ökar inte bara lagringsanvändningen utan också mängden känslig information i säkerhetskopiorna.

## Moln och firmware

### Säkra molnkontot

Så länge Midea-molnet används för installation eller appstyrning förblir även Midea-kontot en del av säkerhetsmodellen. Det bör ha ett unikt lösenord som inte delas med andra tjänster, en lösenordshanterare, flerfaktorsautentisering om den erbjuds, borttagning av gamla smarttelefoner och sessioner, inga delade konton samt regelbunden kontroll av vilka enheter som är registrerade på kontot.

Om Home Assistant-integrationen begär användarnamn och lösenord under installationen bör det kontrolleras om inloggningsuppgifterna endast används för engångshämtning av token eller lagras permanent. Utvecklarna av `Midea Smart AC` skriver att enheter efter installationen inte länkas till inbyggda integrationskonton och att token och nyckel också kan hämtas manuellt via det egna kontot med CLI. Där det är möjligt bör det egna kontot föredras framför främmande eller integrerade samlingskonton. För `Midea AC LAN` är ett personligt Meiju- eller SmartHome-konto som enheten är kopplad till en förutsättning för tokenhämtning sedan september 2026.

### Blockera molnet eller inte?

Efter en lyckad installation uppstår frågan om PortaSplits internetåtkomst bör blockeras helt. För en blockering talar mindre telemetri, mindre beroende av externa tjänster, en mindre angreppsyta via tillverkarens moln, att enheten inte kan kontakta godtyckliga externa mål och mindre påverkan av ändringar på molnsidan.

Emot talar att MSmartHome-appen kanske inte längre fungerar utanför hemnätverket, att firmwareuppdateringar inte längre hämtas, att tids- eller molnfunktioner kan sluta fungera, att ny inloggning eller återställning blir svårare och att vissa enheter reagerar oväntat efter längre tid offline.

En pragmatisk ordning: konfigurera enheten normalt, testa Home Assistant och appen, säkerhetskopiera token och konfiguration, blockera internetåtkomst, starta om enheten och Home Assistant, observera under flera dagar och öppna vid behov endast tillfälligt internetåtkomsten igen.

### Firmwareuppdateringar: säkerhetsvinst eller integrationsrisk?

Firmwareuppdateringar är ett dilemma för IoT-enheter. De kan åtgärda kända sårbarheter, förbättra stabiliteten, modernisera säkerhetsmekanismer och ge nya funktioner. Men de kan också ändra lokala gränssnitt, bryta integrationer byggda med reverse engineering, ogiltigförklara token, inaktivera det lokala API:et och införa nya molnberoenden.

PortaSplit-firmware som levererades i januari 2026 gav exempelvis ett nytt tyst läge för utomhusenheten som sänker ljudnivån med omkring 6 decibel. Community-integrationerna behövde först kartlägga och implementera detta, dokumenterat i ett eget GitHub-ärende för PortaSplit.

Slutsatsen är: förhindra inte firmwareuppdateringar generellt, kontrollera före en uppdatering om andra Home Assistant-användare rapporterar problem, säkerhetskopiera konfiguration och token i förväg, skapa en Home Assistant-säkerhetskopia och testa den lokala styrningen fullständigt efter uppdateringen. Säkerhet betyder inte ”uppdatera aldrig”. Föråldrad firmware kan vara farligare än en tillfälligt inkompatibel integration.

### Vad Midea själv säger om säkerhet

Midea marknadsför sitt SmartHome-ekosystem med inriktning på flera standarder för säkerhet och dataskydd, där EN 303 645, UK PSTI, NIST, GDPR-kompatibel databehandling och kraven i EU:s Radio Equipment Directive nämns. Det är positiva signaler, men säger inget om hur varje enskild PortaSplit-firmware, varje molnändpunkt och varje lokalt API faktiskt är implementerat. Certifierings- och marknadsföringspåståenden ersätter ingen teknisk granskning av den konkreta enheten.

På samma sätt vore det fel att dra slutsatsen av en community-integrations varning att PortaSplit generellt är osäker. Problemet som beskrivs gäller arkitekturen för långlivade tokens och deras användning av inofficiella klienter.

## Risk per scenario

| Scenario | Risk | Motivering |
| --- | --- | --- |
| Normalt hemnätverk utan portvidarebefordran | hanterbar | En angripare måste först få åtkomst till Wi-Fi, Home Assistant eller en säkerhetskopia. |
| Platt hemnätverk med många osäkra IoT-enheter | medel | En komprometterad annan IoT-enhet kan nå PortaSplit eller Home Assistant i samma nätverk. |
| PortaSplit direkt åtkomlig från internet | hög | Enheten bör aldrig publiceras via portvidarebefordran. |
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

Med denna drift är den lokala styrningen försvarbar ur säkerhetssynpunkt: token och nyckel förblir hemliga och säkrade, enheten är endast åtkomlig för Home Assistant och uppdateringar av firmware och integration installeras kontrollerat.

## Ändringslogg

- **25 oktober 2026:** Tillägg: Sedan README-revideringen i september 2026 kräver Midea AC LAN för tokenhämtning ett personligt Meiju- eller SmartHome-konto som enheten är kopplad till; NetHome Plus och migrerade SmartHome-konton levererar inga tokens.

## Källor

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` med ”Important Notice” (sedan den 19 maj 2025, reviderad den 3 och 9 september 2026), motiveringen om tokens som aldrig upphör att gälla och beskrivningen av molnbaserad tokenhämtning via ett personligt Meiju- eller SmartHome-konto.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: molnbaserad hämtning av token och nyckel för V3-enheter, lokal lagring av värdena, standardport 6444.

3.  [midea_ac_lan: anvisningar för debug och konfiguration](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): lagring av enhetskonfigurationen under `/config/.storage/midea_ac_lan/`, rekommendation att säkerhetskopiera i stället för att radera JSON-filen och logger-konfigurationen för debuggloggar.

4.  [Issue 779: PortaSplits utomhus-tystläge](https://github.com/wuwentao/midea_ac_lan/issues/779): begäran om stöd för utomhusenhetens tysta läge, som infördes med firmwareuppdateringen i januari 2026 och sänker ljudnivån med omkring 6 decibel.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): tillverkarens uppgifter om säkerhets- och dataskyddsstandarderna EN 303 645, PSTI, NIST, GDPR och RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation och hantering av Custom Integrations som inte ingår i Home Assistant Core.
