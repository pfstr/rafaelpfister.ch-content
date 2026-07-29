---
title: "Midea PortaSplit i Home Assistant: Varför token och nyckel är avgörande"
navTitle: "PortaSplit och token"
description: "Lokal styrning kräver två värden från Midea-molnet. Så hämtar du token och nyckel, varför det är problematiskt att förlora dem och hur ägare säkrar sin befintliga installation."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "9 min. lästid"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-einrichten"
  - "serverloser-newsletter-cloudflare-workers-d1"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande"
translationOf: "midea-portasplit-home-assistant"
url: "https://rafaelpfister.ch/sv/blog/midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande"
---

<aside class="article-update">
  <p class="article-update__label">Vad PortaSplit-ägare bör göra nu</p>
  <p>Via privata molngränssnitt hämtar Home Assistant den enhetsspecifika token och nyckeln vid installationen. Projektet Midea AC LAN har varnat för möjliga ändringar sedan den 19 maj 2025. Något konkret datum då tillverkaren stänger ned tjänsten är dock inte dokumenterat. För ägare innebär det:</p>
  <ol>
    <li><strong>Ta inte bort en befintlig installation i onödan.</strong> Endast hämtningen av åtkomstuppgifterna kräver Midea-molnet. Framtida ändringar av den privata ändpunkten kan försvåra en ny installation.</li>
    <li><strong>Säkerhetskopiera token, nyckel och konfiguration krypterat.</strong> Om hämtningen inte längre fungerar senare är säkerhetskopian det mest tillförlitliga sättet att återställa.</li>
    <li><strong>Ta inte bort parkopplingen utan behov.</strong> Fabriksåterställning, borttagning från Midea-kontot eller byte av Wi-Fi-modul kräver en ny tokenhämtning, som kan misslyckas i framtiden.</li>
  </ol>
  <p>Redan installerade enheter styrs lokalt. Ändringar av molngränssnittet påverkar därför i första hand tillägg och återställning, inte varje pågående styrkommando. De konkreta stegen finns i <a href="/blog/midea-portasplit-home-assistant-einrichten">praktikartikeln om integrering och säkerhetskopiering</a>.</p>
</aside>

![Exempel på Home Assistant-instrumentpanel för en Midea PortaSplit med rums- och börtemperatur, luftfuktighet, effektförbrukning, energiförbrukning och kompressorns drifttider under de senaste 24 timmarna.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Den lokala styrningen av Midea PortaSplit bygger på två enhetsspecifika värden: token och nyckel. Home Assistant-integrationen hämtar båda via en privat Midea-molnändpunkt under installationen. Därefter skickar den styrkommandon direkt i det lokala nätverket.

Projektet Midea AC LAN varnar för möjliga ändringar i dessa molngränssnitt. Nyare analyser visar dock att ingen bekräftad färdplan från tillverkaren eller något konkret nedstängningsdatum kan härledas av detta. Den här artikeln förklarar det tekniska beroendeförhållandet; den [detaljerade API-analysen](/blog/midea-v2-cloud-api-portasplit-home-assistant) sätter de olika ”V2”-benämningarna och den aktuella situationen i sitt sammanhang.

## Tokenfrågan i detalj

### Varför har Home Assistant hittills kunnat hämta token?

Det intressanta är att communityn aldrig har beräknat token. Den har i stället analyserat nätverkstrafiken från den officiella appen och då konstaterat att appen inte själv skapar token, utan hämtar den från molnet:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

Home Assistant-integrationen har implementerat just detta molnanrop på nytt. Den loggar in i molnet med samma ändpunkter och samma process som appen och får därmed samma token och nyckel. Den egentliga grunden är alltså inte en smart beräkning, utan en återskapad hämtning. Försvinner ändpunkten, försvinner också möjligheten att hämta uppgifterna.

### Skulle man kunna läsa ut token från den officiella appen?

Teoretiskt sett ja. Appen måste känna till token vid någon tidpunkt, annars skulle den inte kunna kommunicera lokalt med enheten. I princip tänkbara sätt är:

- reverse engineering av appen,
- avlyssning av nätverkstrafiken, om den inte är ytterligare skyddad,
- instrumentering av appen under körning, exempelvis med Frida eller Objection,
- hooking av funktionerna som behandlar token.

Det är just detta utvecklaren av Midea AC LAN syftar på när han säger att den tidigare designen är ett säkerhetsproblem ur Mideas perspektiv: En långlivad hemlighet som med rimlig ansträngning går att extrahera ur en brett distribuerad app är svår att kontrollera. För den enskilda användaren är dessa metoder dock krävande och ersätter inte den bekväma molnhämtningen.

### Skulle man kunna få token direkt från enheten?

Det vore den elegantaste lösningen. Om enheten vid den första lokala parkopplingen utbytte en offentlig nyckel eller använde en engångskod via Bluetooth, skulle inget moln behövas. Många moderna IoT-enheter fungerar precis så.

Midea har dock utformat det ursprungliga LAN-protokollet annorlunda: Enheten accepterar lokala kommandon först med rätt molnrelaterade åtkomstuppgifter. Det finns ingen dokumenterad lokal parkopplingsmekanism som skulle lämna ut token utan omvägen via molnet. Molnet är därmed inte bara en bekvämlighet, utan arkitektoniskt den enda avsedda vägen till token.

### Skulle communityn kunna kringgå ändringar av tokenändpunkten?

Det vore bara möjligt om något av följande alternativ hittas:

- ett nytt moln-API som fortfarande levererar token,
- en hittills okänd lokal parkopplingsmetod,
- en sårbarhet i enheten,
- eller att Midea någon gång själv publicerar ett officiellt lokalt API.

Att helt enkelt ”räkna ut” token i efterhand kommer däremot med stor sannolikhet inte att fungera. Om det vore möjligt hade communityn förmodligen redan implementerat det och aldrig varit beroende av moln-API:t. Att omvägen via molnet över huvud taget byggdes är den starkaste indikationen på att det inte finns någon enklare lokal väg.

## Varningen från Midea AC LAN

Repositoryt för `Midea AC LAN` innehåller en tydligt placerad ”Important Notice”. Enligt utvecklaren har Midea redan stängt de serverbaserade token-API:erna i Meiju- och SmartHome-molnen. Integrationen använder därför för närvarande tokengränssnitt från NetHome Plus-molnet, och även dessa ska stängas stegvis. Följden vore att redan installerade enheter fortsätter fungera lokalt, men att nya enheter inte längre kan läggas till. Utvecklaren går ännu längre och skriver att Midea på sikt vill övergå till ett nytt Cloud-Control-API och därmed göra det tidigare V1-LAN-API:t oanvändbart.

Varningen har en kort historia. Den tydliga ”Important Notice” lades till i README den 19 maj 2025 (pull request #578) och angav då SmartHome-molnet som reservnivå för att lägga till nya enheter. Den 14 juli 2025 (#639) uppdaterades den; sedan dess hänvisar den till NetHome Plus-molnet, eftersom Midea hade stängt ytterligare ändpunkter. Kärnan förblev oförändrad i båda versionerna: Tokengränssnitten försvinner steg för steg, och endast det moln som för tillfället fortfarande går att använda ändras.

Detta bör bedömas nyanserat. Det handlar om bedömningen från ett open source-projekt, inte om en bindande färdplan från Midea, och tidplanen är okänd. En framtida firmwareuppdatering kan ändra lokala funktioner, och en redan sparad token kan fortsätta fungera, men behöver inte göra det för alltid. En fabriksåterställning, ett byte av Wi-Fi-modul eller en ny enhet kan göra det nödvändigt att hämta token på nytt.

Detta leder till de tre stegen i rutan i början av artikeln, var och en med sin motivering:

- **Ersätt inte en fungerande installation utan anledning.** Tokenhämtningen är det enda steg som ovillkorligen går via Midea-molnet. Ändringar av den privata ändpunkten kan framför allt drabba en senare nyinstallation.
- **Säkerhetskopiera åtkomstuppgifterna.** Home Assistant lagrar token och nyckel lokalt. Ett trasigt system, en misslyckad återställning eller en oavsiktligt raderad integration kan ändå göra den lokala styrningen obrukbar om ingen extern säkerhetskopia finns.
- **Ta inte bort parkopplingen lättvindigt.** Om en fabriksåterställning eller borttagning från Midea-kontot kräver nya åtkomstuppgifter för varje modell är inte fullständigt dokumenterat. Därför är en säkerhetskopia före sådana ändringar nödvändig.

Den löpande driften påverkas inte i första hand av detta: Den lokala styrningen använder de redan sparade värdena och behöver inte längre tokenändpunkten. En kvarvarande risk finns om en senare firmware ändrar det lokala protokollet eller autentiseringen. Hur token, nyckel och konfiguration säkerhetskopieras beskrivs i [praktikartikeln om installationen](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Vad det innebär för säkerheten

Varningen har, utöver tillgänglighet, en säkerhetsmässig kärna. Enligt `Midea AC LAN` bygger den äldre LAN-arkitekturen på ett problematiskt antagande: Klientkommunikationen ansågs ursprungligen vara tillräckligt skyddad, vilket innebar att de tokens som molnet utfärdade inte fick någon utgångstid.

En token som inte löper ut är i sig ännu ingen sårbarhet. Den blir problematisk om den hamnar i loggar eller oskyddade säkerhetskopior, kommer i tredje parts händer eller varken kan återkallas eller roteras. Utvecklaren av `Midea AC LAN` antar att Midea reagerar på dessa risker med ändringar av tokentjänsterna och en mer molnbaserad arkitektur. Något motsvarande tillkännagivande från tillverkaren med tidplan är dock inte belagt.

Språklig precision är viktig här. Community-integrationen ”hackar” inte luftkonditioneringsenheten. Den implementerar ett proprietärt protokoll som har kartlagts genom reverse engineering. Säkerhetsproblemet uppstår genom att långlivade hemligheter kan användas och lagras utanför den ursprungligen avsedda appen.

För drift i det egna nätverket är det framför allt relevant vad token och nyckel möjliggör. Båda autentiserar den lokala kommunikationen med enheten. Om de hamnar i fel händer kan en angripare, beroende på protokollet och sin nätverksposition, identifiera enheten, autentisera sig mot den, läsa ut statusinformation, ändra inställningar, slå på eller av luftkonditioneringen, byta driftläge och ändra börtemperaturen. Angriparen måste dock vanligtvis fortfarande kunna upprätta en nätverksanslutning till enheten; enbart innehav av token och nyckel möjliggör ingen attack från hela internet. Token och nyckel bör därför behandlas som ett lösenord. Hur enheten kan integreras i nätverket så att dessa värden orsakar liten skada även vid en incident är ämnet för [den andra delen](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Vad som återstår i praktiken

Den lokala styrningen av PortaSplit är helt beroende av token och nyckel, som för närvarande endast kan hämtas via Midea-molnet. Denna omväg är en del av protokolldesignen: Lokala kommandon är bundna till molnrelaterade åtkomstuppgifter. Eftersom ändpunkten är privat och odokumenterad är den långsiktiga tillgängligheten för den inofficiella integrationen osäker.

I praktiken betyder det: säkerhetskopiera åtkomstuppgifter och konfiguration, ta inte bort en fungerande parkoppling i onödan och följ ändringar av integration och firmware. Redan installerade enheter fortsätter fungera lokalt. Installation, säkerhetskopiering och nätverksskydd beskrivs i [praktikartikeln om PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Källor

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` med ”Important Notice” (sedan 19 maj 2025, uppdaterad den 14 juli 2025), motiveringen om tokens utan utgångstid och rekonstruerad klientkryptering samt beskrivningen av den molnbaserade tokenhämtningen.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: Beskrivning av den molnbaserade hämtningen av token och nyckel för V3-enheter och den lokala lagringen av värdena.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Tillverkaruppgifter om SmartHome-ekosystemet och de säkerhets- och dataskyddsstandarder som det hänvisas till.
