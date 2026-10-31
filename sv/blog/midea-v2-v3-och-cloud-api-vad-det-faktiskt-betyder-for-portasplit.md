---
title: "Midea V2, V3 och Cloud API: Vad det egentligen innebär för PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "Lokalt enhetsprotokoll, privata appslutpunkter och officiellt partner-API använder liknande versionsnamn. Källanalysen skiljer mellan dessa nivåer och sätter avvecklingsvarningen i sitt sammanhang."
date: "2026-07-25"
kategorie: "Home Assistant och IoT"
timeToRead: "11 min läsning"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
slug: "midea-v2-v3-och-cloud-api-vad-det-faktiskt-betyder-for-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:35:16.280Z
translationReview: required
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/midea-v2-v3-och-cloud-api-vad-det-faktiskt-betyder-for-portasplit
---

I Midea PortaSplit-sammanhang avser ”V2” flera sinsemellan oberoende saker. Det finns ett lokalt V2-enhetsprotokoll, versionsnummer i privata appslutpunkter och ett officiellt Cloud-to-Cloud API V2 för partner. Den som likställer dessa nivåer drar oundvikligen fel slutsatser om den lokala styrningen.

Projektet `Midea AC LAN` varnar i sin [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) för att tidigare tokensnittställen skulle stängas och ersättas av ett molnbaserat V2-API. En granskning av diskussionerna, den aktuella koden och Mideas officiella dokumentation ger en mer nyanserad bild:

> Ett officiellt Midea Cloud-to-Cloud API V2 finns. Men det är inte identiskt med tokensnittstället som används av Home Assistant och inte heller med det lokala V2- eller V3-enhetsprotokollet. Någon officiellt annonserad avveckling av lokal PortaSplit-styrning med ett konkret datum är inte dokumenterad. I juni 2026 visades dessutom att det förment avstängda SmartHome-token-API:t fortfarande fungerade – den tidigare begäran från community-biblioteket var bara ofullständig.

Detta är del 3 i serien; [del 1](/blog/midea-portasplit-home-assistant) beskriver installationen fram till instrumentpanelen, [del 2](/blog/midea-portasplit-home-assistant-absichern) skyddet av token, key och hemnätverk. Artikeln är uppdaterad den 25 oktober 2026.

![Home Assistant-instrumentpanel för Midea PortaSplit i kyldrift: nyckeltal högst upp, termostat på 22 °C, förlopp för rumstemperatur, effektförbrukning, dagsenergi, kompressorfrekvens, kompressordrift och fläkthastighet, samt tekniska värden och status nedanför.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Varför den tidigare bedömningen behöver korrigeras

I en tidigare version av [artikeln om token och key](/blog/midea-portasplit-home-assistant-absichern) återgav jag varningen från projektet `Midea AC LAN` ungefär som en annonserad avveckling av molnsnittställena. Det motsvarade ordalydelsen i projektets README, men var för starkt formulerat som faktapåstående.

Varningen är fortfarande relevant som riskindikering. Den är dock ingen publicerad Midea-färdplan. Framför allt finns nu nytt tekniskt material som ifrågasätter en väsentlig del av den tidigare tolkningen.

## Så fungerar den lokala PortaSplit-styrningen

Home Assistant-integrationen `Midea Smart AC` beskriver uttryckligen sin arkitektur som lokal styrning. För nyare V3-enheter används Midea-molnet endast under installationen för att hämta en enhetsspecifik token och key. Därefter lagrar integrationen båda värdena lokalt och behöver ingen ytterligare molnanslutning för själva styrningen. Projektet dokumenterar detta under [”Note On Cloud Usage”](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Förenklat ser flödet ut så här:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

För manuellt konfigurerade V3-enheter kräver `Midea Smart AC` enhets-ID, IP-adress, port, token och key. Den dokumenterade standardporten är `6444/TCP`; token och key anges som 128 respektive 64 hexadecimala tecken. Dessa uppgifter finns i [dokumentationen för manuell konfiguration](https://github.com/mill1000/midea-ac-py#manual-configuration).

En PortaSplit identifierades exempelvis i issue-trackern för `Midea AC LAN` som enhetstyp `0xAC`, modell `00000Q1D` och protokollversion 3. Samma användare kunde sedan lägga till den via NetHome Plus i Home Assistant. Det konkreta förloppet dokumenteras i [issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

Avgörande är uppdelningen:

- Molntjänsten används för att hämta lokala åtkomstuppgifter.
- Den senare styrningen sker direkt i LAN.
- Ett fel i token-tjänsten hindrar därför främst nya installationer.
- Det avslutar inte automatiskt en redan konfigurerad lokal anslutning.

Det senare överensstämmer även med den uttryckliga beskrivningen från [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Varifrån avvecklingsvarningen kommer

Den varningstext som syns i dag lades till i dokumentationen den 19 maj 2025 genom [pull request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Motiveringen kan sammanfattas så här:

- De lokala tokenvärdena saknade utgångsdatum.
- Olika Home Assistant-projekt använde efterbildad eller extraherad appkryptering.
- Detta medförde ett säkerhetsproblem.
- Midea skulle därför stegvis stänga de tidigare token-tjänsterna.
- På lång sikt skulle lokal V1-styrning trängas undan av ett molnbaserat V2-API.

I juli 2025 anpassades dokumentationen på nytt genom [pull request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). I stället för SmartHome-molnet angavs nu NetHome Plus som tillfälligt använd tokenkälla. Själva avvecklingsvarningen fanns kvar.

Den bakomliggande diskussionen är dock mer försiktigt formulerad än README-filen.

I [kommentaren från Midea-AC-LAN-underhållaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) sägs ungefär att NetHome Plus möjligen bara är en tillfällig lösning och att Midea, enligt hans uppfattning, har en ny helt molnbaserad V2-tjänst.

Underhållaren av `midea-msmart` svarade att även han hade misstänkt att ett nytt V2-API fanns, men att han bara i begränsad omfattning kunde undersöka det eftersom han saknade egna Midea-enheter. Detta framgår av [det direkta svarskommentaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Därmed är källäget tydligare:

- Varningen kommer från erfarna community-utvecklare.
- Den bygger på observerade förändringar och deras tekniska bedömning.
- En av underhållarna beskriver uttryckligen V2-migreringen som sin uppfattning.
- Den andra talar om en misstanke.
- Varken pull requesten eller diskussionen länkar till någon officiell Midea-annonsering om avveckling eller något datum.

Det gör inte varningen värdelös. Men det gör den till en riskanalys, inte en bekräftad färdplan från tillverkaren.

## Det avgörande nya fyndet från juni 2026

Den 15 juni 2026 infördes en korrigering i biblioteket `midea-local`, vilket väsentligt förändrar den tidigare tolkningen.

Utgångspunkten var felet:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Felet hade uppstått vid hämtning av token och key via SmartHome-molnet. Inloggning och enhetslista fungerade fortfarande, men anropet till `/v1/iot/secure/getToken` avvisades.

Till en början såg detta ut som ett avstängt eller obrukbart gränssnitt. En analys av begäran från den officiella SmartHome-appen visade dock en annan orsak: Appen skickade, utöver `udpid`, även fältet `applianceCodes`. Community-biblioteket hade inte skickat med detta fält.

Den korrigerade begäran innehåller nu:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Utvecklaren testade ändringen med ett verkligt SmartHome-konto och fyra V3-luftkonditioneringsenheter av typen `0xAC`:

- Utan `applianceCodes` svarade servern med fel 3004.
- Med `applianceCodes` levererade den giltiga tokenvärden och keys.
- De returnerade värdena fungerade sedan för lokal V3-autentisering.

Den fullständiga undersökningen, testresultaten och kod-diffen dokumenteras i [`midea-local` pull request #470](https://github.com/midea-lan/midea-local/pull/470). Den tillhörande oföränderliga committen finns i [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Även den aktuella källkoden använder fortfarande exakt denna slutpunkt:

```text
/v1/iot/secure/getToken
```

Dessutom skickas nu `applianceCodes` med. Det går att direkt kontrollera i den aktuella [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Sedan den 12 augusti 2026 använder `Midea AC LAN` sitt eget bibliotek `midea-lan` i stället för `midea-local`; den aktuella versionen inkluderar `midea-lan==2026.10.0` och deklarerar sig fortfarande som en `local_push`-integration. Båda framgår av det aktuella [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json). Även `midea-lan` hämtar token och key från SmartHome-molnet via `/v1/iot/secure/getToken` med `applianceCodes`, vilket framgår av [`midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

Det generella påståendet att SmartHome-token-API:t hade stängts motbevisas därmed, åtminstone för de konton och enheter som testades i juni 2026. Rätt formulering är:

> Den tidigare tokenförfrågan fungerade inte längre efter en ändring i det förväntade begäranseformatet. Efter anpassning till det format som används av den officiella appen levererade samma V1-slutpunkt åter giltiga lokala åtkomstuppgifter.

Regionala skillnader, avvikande konton eller enhetstyper som inte stöds kan därmed inte uteslutas. Men det var uppenbarligen ingen global avstängning.

I andra Midea-moln har läget förändrats sedan dess. Det kinesiska Meiju-molnet svarar sedan augusti 2026 på `/v1/iot/secure/getToken` med felet `40404`; token och key levereras där nu bara av appslutpunkten `/v2/iot/secure/getToken`, som dessutom kräver `homegroupId`. Biblioteken anpassades i slutet av augusti 2026 i enlighet med detta, dokumenterat i [`midea-local` pull request #717](https://github.com/midea-lan/midea-local/pull/717). Via NetHome Plus-konton går det enligt [README för `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice) för närvarande inte längre att hämta tokenvärden; bara personliga Meiju- och SmartHome-konton som enheten är knuten till kan användas. Även denna förändring berör en privat appslutpunkt, inte det officiella Cloud-to-Cloud API V2.

## Varför ”V2” så lätt missförstås här

I Midea-sammanhang används minst tre sinsemellan oberoende versionsbeteckningar.

| Begrepp | Betydelse |
| --- | --- |
| Lokalt V2-/V3-protokoll | Generation av den direkta kommunikationen mellan integration och enhet |
| V1-/V2-appslutpunkt | Versionsnummer för en enskild HTTP-slutpunkt i Midea-apparnas backend |
| Cloud-to-Cloud API V2 | Officiellt partner-API för auktoriserade tredjepartsföretag |

### Lokalt V2 och V3

I det lokala enhetsprotokollet avser V2 respektive V3 enhetens kommunikationsgeneration. Nyare V3-enheter behöver token och key för lokal autentisering. `Midea Smart AC` dokumenterar detta krav i sin [konfigurationsguide](https://github.com/mill1000/midea-ac-py#manual-configuration).

Denna protokollversion har inget att göra med det officiella Cloud-to-Cloud API V2.

### V1 och V2 i app-URL:er

Även inom samma app kan slutpunkter med olika versionsnummer användas samtidigt. Ett `/v2/` i URL-sökvägen betyder därför inte att hela plattformen har lagts om till en ny arkitektur.

Den aktuella koden för `midea-local` använder fortfarande [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) för token och key. Andra funktioner kan ändå ligga under sökvägar med andra versionsnummer.

### Officiellt Cloud-to-Cloud API V2

Midea dokumenterar faktiskt ett [officiellt Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Det använder bland annat:

- OAuth 2.0
- `client_id` och `client_secret`
- kortlivade access tokens och refresh tokens
- HMAC-SHA256-signaturer
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- molnbaserade statusförfrågningar och styrkommandon

Detta är ett kontrollerat partnersnittställe. Den nödvändiga `client_secret` tilldelas en tredjepartsleverantör av Midea. En vanlig ägare av en PortaSplit får den inte bara via sitt MSmartHome-konto. Kraven och signaturreglerna beskrivs i den [officiella V2-dokumentationen](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Detta API uppstod dessutom inte först 2025. Dokumentationen innehåller exempel på begäranden med tidsstämplar från 2018 och en Java-kommentar från den 18 april 2019. V2-partnersnittstället fanns alltså långt före varningen i `Midea AC LAN`.

## Midea ersätter faktiskt ett V1-API – men ett annat

Midea har också ett äldre officiellt Cloud-to-Cloud-gränssnitt under `/v1/open/...`. Dess dokumentation innehåller uttryckligen en upplysning om att det inte längre rekommenderas, kan stängas av i framtiden och att den nya V2-dokumentationen bör användas. Detta står i Mideas [dokumentation för det gamla Cloud-to-Cloud API:t](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Denna upplysning är en verklig officiell V1-till-V2-migrering. Men den gäller partnerslutpunkterna:

```text
/v1/open/...
           ↓
/v2/open/...
```

Tokenförfrågan som används av Home Assistant-biblioteken är däremot:

```text
/v1/iot/secure/getToken
```

Och den lokala PortaSplit-anslutningen går sedan inte alls via en sådan moln-URL, utan direkt i hemnätverket.

Att likställa de tre gränssnitten bara på grund av versionsnumret ”V1” vore därför tekniskt obefogat.

## Finns det redan en helt molnbaserad Home Assistant-integration?

Med [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) finns nu en community-integration som styr Midea-enheter via molnet i stället för direkt via LAN.

Men inte heller detta bevisar att det officiella partner-V2-API:t redan skulle ha ersatt den lokala styrningen. Den aktuella källkoden för `Midea Auto Cloud` använder bland annat:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Dessa slutpunkter kan ses i den aktuella [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Integrationen efterbildar alltså privata app- respektive konsumentmolnfunktioner. Den använder inte bara det dokumenterade `/v2/open/...`-partnersnittstället.

Det finns alltså redan ett molnbaserat alternativ. Men det medför också de vanliga beroendena för en molnintegration: internetanslutning, fungerande användarkonto, tillgängliga Midea-servrar och fortsatt kompatibla privata slutpunkter.

## Vad innebär detta konkret för PortaSplit-ägare?

### Redan konfigurerad lokal styrning

För en redan konfigurerad PortaSplit är läget relativt okritiskt. `Midea Smart AC` lagrar token och key lokalt efter installationen och behöver enligt sin egen [molndokumentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) ingen molnanslutning för fortsatt styrning.

En avstängning av enbart tokenhämtningen skulle därför inte automatiskt avsluta den befintliga lokala anslutningen.

### Ny installation eller återställning

Risken är större vid:

- en ny Home Assistant-installation
- byte till en annan integration
- en förlorad eller skadad säkerhetskopia
- byte av WLAN-modulen
- ändringar i enhetstilldelningen
- en ny parkoppling, om enhetens åtkomstuppgifter ändras då

I sådana fall måste integrationen hämta token och key på nytt, eller så måste användaren ange dem manuellt. Att `Midea Smart AC` stöder manuell konfiguration beskrivs i dess [konfigurationsdokumentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Huruvida en fabriksåterställning eller ny parkoppling alltid genererar nya åtkomstuppgifter för varje PortaSplit är inte officiellt dokumenterat och bör därför inte påstås generellt.

### En verklig avstängning av LAN-styrningen

För att en redan konfigurerad PortaSplit inte längre ska acceptera sina lokalt lagrade åtkomstuppgifter måste även enhetens eller WLAN-modulens beteende förändras, exempelvis genom ny firmware eller ett ändrat autentiseringsförfarande.

En ren avstängning av molnslutpunkten `/v1/iot/secure/getToken` tar inte automatiskt bort åtkomstuppgifterna som redan finns i enheten och i Home Assistant. Detta följer av uppdelningen mellan engångshämtning från molnet och efterföljande LAN-styrning som dokumenteras av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

En sådan framtida enhetsändring är tekniskt möjlig. Men jag har inte hittat någon konkret annonsering eller något avstängningsdatum specifikt för PortaSplit i Mideas offentligt tillgängliga dokumentation.

## Vad jag fortfarande skulle rekommendera

Trots de relativiserande insikterna är en säkerhetskopia fortfarande klok.

För V3-enheter rekommenderar `Midea AC LAN` uttryckligen att den genererade JSON-konfigurationen sparas utanför HAOS. Den aktuella rekommendationen finns direkt i [projektets README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

En säkerhetskopia är ett rimligt skydd mot molnförändringar, integrationsproblem och egna misstag, men inte en indikation på att en avstängning är nära förestående. Del 2 beskriver hur token, key och konfiguration säkras [del 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern).

## Bedömning utifrån tillgängliga belägg

Varningen från `Midea AC LAN` bör tas på allvar, men sättas i rätt sammanhang.

Den dokumenterar en rimlig långsiktig risk: Midea skulle kunna betrakta lokala tokenvärden utan utgångsdatum som ett säkerhetsproblem, ytterligare begränsa hämtningen av sådana tokenvärden eller knyta framtida enheter starkare till molnet.

Däremot finns inga belägg för en officiellt annonserad och tidsatt avveckling av lokal PortaSplit-styrning.

Det aktuella tekniska läget visar till och med motsatsen till en redan genomförd avstängning: I juni 2026 levererade den fortfarande använda V1-tokenslutpunkten giltiga åtkomstuppgifter efter att begäran hade anpassats till formatet i den officiella SmartHome-appen. Den aktuella korrigeringen ingår nu i biblioteket som används av `Midea AC LAN`.

Det officiella Midea Cloud-to-Cloud API V2 finns också. Men det är ett äldre partnersnittställe med begränsad åtkomst och inte automatiskt efterföljaren till det lokala PortaSplit-protokollet.

Den sakliga slutsatsen är därför:

> Skapa en säkerhetskopia, håll koll på integrationerna och tänk på molnberoenden – men avskriv inte den lokala PortaSplit-styrningen i förtid på grund av ett obekräftat antagande om avveckling.

## Ändringshistorik

- **25 oktober 2026:** Tillägg: Midea AC LAN använder sedan augusti 2026 biblioteket midea-lan (för närvarande 2026.10.0), och Meiju-molnet levererar token och key endast via slutpunkten /v2/iot/secure/getToken med homegroupId. Artikelns datum har uppdaterats till den 25 oktober 2026 och källor har lagts till.

## Källor

1.  [Midea AC LAN: aktuell README och avvecklingsvarning](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Varningens ordalydelse, rekommendation om säkerhetskopia och åtskillnad mellan äldre V2- och nyare V3-enheter.

2.  [Midea AC LAN PR #578 från 19 maj 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Införande av varningen om stegvis avveckling av token-tjänsterna och den påstådda migreringen till ett molnbaserat V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Byte av den dokumenterade tokenkällan till NetHome Plus.

4.  [midea-msmart issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskussion om den felaktiga SmartHome-tokenförfrågan och den tillfälliga användningen av NetHome Plus.

5.  [Kommentar från Midea-AC-LAN-underhållaren om den misstänkta V2-migreringen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Markerar uttryckligen uttalandet om det nya V2-molnet som sin egen uppfattning.

6.  [Svar från midea-msmart-underhållaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beskriver förekomsten av ett nytt V2-API som en misstanke och hänvisar till de begränsade möjligheterna till reverse engineering.

7.  [midea-local PR #470 från 15 juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analys av fel 3004, inspelning av den officiella appens begäran, tillägg av `applianceCodes` och lyckat test med fyra V3-luftkonditioneringsenheter.

8.  [Oföränderlig commit för SmartHome-getToken-korrigeringen](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Exakt kod-diff för den införda korrigeringen.

9.  [Aktuell midea-local-molnkod](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Fortsatt använd slutpunkt `/v1/iot/secure/getToken` och aktuellt begäransefält `applianceCodes`.

10.  [Aktuellt manifest för Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Använd version av `midea-lan` och klassificering som lokal push-integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentation av lokal styrning, engångshämtning från molnet för V3-enheter och manuell konfiguration med token och key.

12.  [Midea AC LAN issue #607 om PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkret PortaSplit-exempel med enhetstyp `0xAC`, modell `00000Q1D`, protokollversion 3 och lyckad installation via NetHome Plus.

13.  [Officiellt Midea Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, Client-ID, Client-Secret, access- och refresh tokens, signaturförfarande och `/v2/open/...`-slutpunkter.

14.  [Officiellt Midea Cloud-to-Cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Officiell information om att det gamla `/v1/open/...`-partnersnittstället inte längre rekommenderas och kan avvecklas i framtiden.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) och [aktuell molnkod](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Community-integration för fullständig molnstyrning och de privata V1-appslutpunkter som faktiskt används.

16.  [midea-local PR #717](https://github.com/midea-lan/midea-local/pull/717): Meiju-molnets byte från `/v1/iot/secure/getToken` (fel 40404) till `/v2/iot/secure/getToken` med `homegroupId`.

17.  [Aktuell midea-lan-molnkod](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py): Bibliotek som Midea AC LAN använder sedan augusti 2026, SmartHome-tokenförfrågan via `/v1/iot/secure/getToken` med `applianceCodes`.
