---
title: "Midea PortaSplit i Home Assistant: varför token och nyckel är avgörande"
navTitle: "PortaSplit och token"
description: "Lokal styrning kräver två värden från Midea-molnet. Så hämtar du token och nyckel, varför det är problematiskt att förlora dem och hur ägare säkrar sin befintliga installation."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "9 min. lästid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-einrichten
  - serverloser-newsletter-cloudflare-workers-d1
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande"
translationOf: "midea-portasplit-home-assistant"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: e2e3c42704dc7a3f4618f688790356c5a0ccfa18e0796789bd48cf9841bed1a8
translatedAt: 2026-08-08T14:16:41.513Z
url: https://rafaelpfister.ch/sv/blog/midea-portasplit-i-home-assistant-varfor-token-och-nyckel-ar-avgorande
translationModel: gpt-5.6-terra
---

<aside class="article-update">
  <p class="article-update__label">Vad PortaSplit-ägare bör göra nu</p>
  <p>Via privata molngränssnitt hämtar Home Assistant den enhetsspecifika token och nyckeln vid installationen. Projektet Midea AC LAN har varnat för möjliga ändringar sedan den 19 maj 2025. Något konkret datum för tillverkarens avveckling är dock inte dokumenterat. För ägare innebär det följande:</p>
  <ol>
    <li><strong>Ta inte bort en befintlig installation i onödan.</strong> Endast hämtningen av åtkomstuppgifterna kräver Midea-molnet. Framtida ändringar av den privata slutpunkten kan försvåra en ny installation.</li>
    <li><strong>Säkerhetskopiera token, nyckel och konfiguration krypterat.</strong> Om hämtningen senare inte längre fungerar är säkerhetskopian det tillförlitligaste sättet att återställa.</li>
    <li><strong>Bryt inte parkopplingen utan behov.</strong> Fabriksåterställning, borttagning från Midea-kontot eller byte av Wi-Fi-modul kräver en ny tokenhämtning, som kan misslyckas i framtiden.</li>
  </ol>
  <p>Redan konfigurerade enheter styrs lokalt. Ändringar av molngränssnittet påverkar därför först tillägg och återställning, inte varje pågående styrkommando. De konkreta stegen finns i <a href="/blog/midea-portasplit-home-assistant-einrichten">praktikartikeln om integrering och skydd</a>.</p>
</aside>

![Exempel på en Home Assistant-instrumentpanel för en Midea PortaSplit med rums- och börtemperatur, luftfuktighet, effektförbrukning, energiförbrukning och kompressorns drifttider under de senaste 24 timmarna.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

Den lokala styrningen av Midea PortaSplit bygger på två enhetsspecifika värden: token och nyckel. Home Assistant-integrationen hämtar båda via en privat Midea-molnslutpunkt under installationen. Därefter skickar den styrkommandon direkt i det lokala nätverket.

Projektet Midea AC LAN varnar för möjliga ändringar av dessa molngränssnitt. Nyare analyser visar dock att ingen bekräftad tillverkarplan eller något konkret avvecklingsdatum kan härledas från detta. Den här artikeln förklarar det tekniska beroendeförhållandet; den [detaljerade API-analysen](/blog/midea-v2-cloud-api-portasplit-home-assistant) sätter de olika ”V2”-benämningarna och det aktuella läget i sitt sammanhang.

## Token-frågan i detalj

### Varför har Home Assistant hittills kunnat hämta token?

Det intressanta är att communityn aldrig har beräknat token. I stället har den analyserat nätverkstrafiken från den officiella appen och konstaterat att appen inte själv skapar token, utan hämtar den från molnet:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

Home Assistant-integrationen har återskapat just detta molnanrop. Den loggar in i molnet med samma slutpunkter och samma förlopp som appen och får därmed samma token och nyckel. Grunden är alltså inte en smart beräkning, utan en återskapad hämtning. Försvinner slutpunkten, försvinner även möjligheten att hämta dem.

### Skulle man kunna läsa ut token från den officiella appen?

Teoretiskt sett ja. Appen måste känna till token vid något tillfälle, annars skulle den inte kunna kommunicera lokalt med enheten. I princip tänkbara vägar är:

- reverse engineering av appen,
- avlyssning av nätverkstrafiken, om den inte är extra skyddad,
- instrumentering av appen under körning, exempelvis med Frida eller Objection,
- hookning av funktionerna som bearbetar token.

Det är just detta som utvecklaren av Midea AC LAN syftar på när han säger att den tidigare designen ur Mideas perspektiv är ett säkerhetsproblem: En långlivad hemlighet som med rimlig insats kan extraheras ur en brett distribuerad app är svår att kontrollera. För den enskilda användaren är dessa metoder dock krävande och ersätter inte den bekväma molnhämtningen.

### Skulle man kunna få token direkt från enheten?

Det vore den elegantaste lösningen. Om enheten vid första lokala parkopplingen utbytte en publik nyckel eller använde en engångskod för parkoppling via Bluetooth, skulle inget moln alls behövas. Många moderna IoT-enheter fungerar just så.

Midea har dock utformat det ursprungliga LAN-protokollet annorlunda: Enheten accepterar lokala kommandon först med rätt molnrelaterade åtkomstuppgifter. Det finns ingen dokumenterad lokal parkopplingsmekanism som skulle lämna ut token utan omvägen via molnet. Molnet är därmed inte bara en bekvämlighet, utan arkitektoniskt den enda avsedda vägen till token.

### Kan communityn kringgå ändringar av tokenslutpunkten?

Det vore bara möjligt om något av följande alternativ hittas:

- ett nytt moln-API som fortsatt levererar token,
- en hittills okänd lokal parkopplingsmetod,
- en sårbarhet i enheten,
- eller om Midea någon gång själv publicerar ett officiellt lokalt API.

Att bara ”räkna fram” token kommer däremot med stor sannolikhet inte att fungera. Om det vore möjligt hade communityn sannolikt redan implementerat det och aldrig varit beroende av moln-API:t. Att omvägen via molnet över huvud taget byggdes är den starkaste indikationen på att det inte finns någon enklare lokal väg.

## Varningen från Midea AC LAN

Repositoryt för `Midea AC LAN` innehåller ett framträdande ”Important Notice”. Enligt utvecklaren har Midea redan stängt de serverbaserade token-API:erna i Meiju- och SmartHome-molnen. Integrationen använder därför för närvarande tokengränssnitt från NetHome Plus-molnet, och även dessa ska stängas stegvis. Följden skulle vara att redan konfigurerade enheter fortsätter att fungera lokalt, men att nya enheter inte längre kan läggas till. Utvecklaren går ännu längre och skriver att Midea på sikt vill gå över till ett nytt Cloud-Control-API och därmed göra det tidigare V1-LAN-API:t obrukbart.

Varningen har en kort historia. Det framträdande ”Important Notice” lades till i README den 19 maj 2025 (Pull Request #578) och angav då SmartHome-molnet som reservlösning för att lägga till nya enheter. Den 14 juli 2025 (#639) uppdaterades den; sedan dess hänvisar den till NetHome Plus-molnet, eftersom Midea hade stängt ytterligare slutpunkter. Kärnan var oförändrad i båda versionerna: tokengränssnitten försvinner stegvis, och endast det moln som fortfarande går att använda ändras.

Detta måste ses nyanserat. Det handlar om bedömningen från ett open source-projekt, inte om en bindande plan från Midea, och tidsplanen är okänd. En framtida firmwareuppdatering kan förändra lokala funktioner; en redan sparad token kan fortsätta fungera, men behöver inte göra det för alltid. En fabriksåterställning, byte av Wi-Fi-modul eller en ny enhet kan göra det nödvändigt att hämta token på nytt.

Av detta följer de tre stegen i rutan i början av artikeln, med respektive motivering:

- **Byt inte ut en fungerande installation utan anledning.** Hämtning av token är det enda steg som nödvändigtvis går via Midea-molnet. Ändringar av den privata slutpunkten kan främst påverka en senare ny installation.
- **Säkerhetskopiera åtkomstuppgifterna.** Home Assistant lagrar token och nyckel lokalt. Ett trasigt system, en misslyckad återställning eller en oavsiktligt borttagen integration kan ändå göra den lokala styrningen obrukbar om ingen extern säkerhetskopia finns.
- **Bryt inte parkopplingen lättvindigt.** Om en fabriksåterställning eller borttagning från Midea-kontot kräver nya åtkomstuppgifter för varje modell är inte fullständigt dokumenterat. En säkerhetskopia före sådana ändringar är därför nödvändig.

Den löpande driften påverkas till en början inte av detta: Den lokala styrningen använder de redan sparade värdena och behöver inte längre tokenslutpunkten. En kvarvarande risk finns om en senare firmware ändrar det lokala protokollet eller autentiseringen. Hur token, nyckel och konfiguration säkerhetskopieras beskrivs i [praktikartikeln om installation](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Vad detta innebär för säkerheten

Varningen har, utöver tillgängligheten, en säkerhetsmässig kärna. Enligt `Midea AC LAN` bygger den äldre LAN-arkitekturen på ett problematiskt antagande: Klientkommunikationen ansågs ursprungligen vara tillräckligt skyddad, vilket innebar att de token som utfärdades av molnet inte fick någon utgångstid.

En token som inte löper ut är i sig ännu ingen sårbarhet. Det blir problematiskt om den hamnar i protokoll eller oskyddade säkerhetskopior, hamnar hos tredje part eller varken kan återkallas eller roteras. Utvecklaren av `Midea AC LAN` misstänker att Midea reagerar på dessa risker med ändringar av token-tjänsterna och en mer molnbaserad arkitektur. Något motsvarande tillkännagivande från tillverkaren med tidsplan finns dock inte belagt.

Språklig precision är viktig här. Community-integrationen ”hackar” inte luftkonditioneringsenheten. Den implementerar ett proprietärt protokoll som har återskapats genom reverse engineering. Säkerhetsproblemet uppstår genom att långlivade hemligheter kan användas och lagras utanför den ursprungligen avsedda appen.

För drift i det egna nätverket är det framför allt relevant vad token och nyckel möjliggör. Båda autentiserar den lokala kommunikationen med enheten. Om de hamnar i fel händer kan en angripare, beroende på protokollet och sin nätverksposition, identifiera enheten, autentisera sig mot den, läsa ut statusinformation, ändra inställningar, slå på eller stänga av luftkonditioneringen, byta driftläge och ändra börtemperaturen. Angriparen måste vanligtvis ändå kunna upprätta en nätverksanslutning till enheten; innehav av enbart token och nyckel möjliggör ingen attack från hela internet. Token och nyckel bör därför behandlas som ett lösenord. Hur enheten kan integreras nätverksmässigt så att dessa värden orsakar liten skada även vid en incident är ämnet för [del två](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Vad som återstår i praktiken

Den lokala styrningen av PortaSplit står och faller med token och nyckel, som för närvarande endast kan hämtas via Midea-molnet. Denna omväg är en del av protokolldesignen: Lokala kommandon är knutna till molnrelaterade åtkomstuppgifter. Eftersom slutpunkten är privat och odokumenterad är den inofficiella integrationens långsiktiga tillgänglighet osäker.

I praktiken innebär det: säkerhetskopiera åtkomstuppgifter och konfiguration, bryt inte en fungerande parkoppling i onödan och följ ändringar i integrationen och firmware. Redan konfigurerade enheter fortsätter att köras lokalt. Installation, säkerhetskopiering och nätverksskydd beskrivs i [praktikartikeln om PortaSplit](/blog/midea-portasplit-home-assistant-einrichten).

## Källor

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` med ”Important Notice” (sedan 19 maj 2025, uppdaterat den 14 juli 2025), motiveringen om token som inte löper ut och rekonstruerad klientkryptering samt beskrivningen av den molnbaserade tokenhämtningen.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: Beskrivning av den molnbaserade hämtningen av token och nyckel för V3-enheter samt den lokala lagringen av värdena.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): Tillverkarens uppgifter om SmartHome-ekosystemet och de refererade säkerhets- och dataskyddsstandarderna.
