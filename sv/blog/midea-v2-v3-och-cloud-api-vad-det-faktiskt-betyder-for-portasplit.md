---
title: "Midea V2, V3 och Cloud API: Vad det faktiskt betyder för PortaSplit"
navTitle: "Midea V2 Cloud API"
description: "Lokalt enhetsprotokoll, privata appändpunkter och officiellt partner-API använder liknande versionsnamn. Källanalysen skiljer dessa nivåer åt och sätter avvecklingsvarningen i sitt sammanhang."
date: "2026-07-25"
kategorie: "Home Assistant och IoT"
timeToRead: "11 min läsning"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-portasplit-home-assistant-einrichten
draft: false
slug: "midea-v2-v3-och-cloud-api-vad-det-faktiskt-betyder-for-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
url: "https://rafaelpfister.ch/sv/blog/midea-v2-v3-och-cloud-api-vad-det-faktiskt-betyder-for-portasplit"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-07-29T07:27:32.140Z
translationReview: automatic
translationSourceHash: d63bb09234efede35ce75b1f2ef582fc016791b560d7cb6ba826c718ce50c791
---

I samband med Midea PortaSplit syftar «V2» på flera sinsemellan oberoende saker. Det finns ett lokalt V2-enhetsprotokoll, versionsnummer i privata appändpunkter och ett officiellt moln-till-moln-API V2 för partner. Den som likställer dessa nivåer drar oundvikligen fel slutsatser om den lokala styrningen.

Projektet `Midea AC LAN` varnar i sin [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) för att tidigare token-gränssnitt skulle stängas och ersättas av ett molnbaserat V2-API. En granskning av diskussionerna, den aktuella koden och Mideas officiella dokumentation ger en mer nyanserad bild:

> Ett officiellt Midea moln-till-moln-API V2 finns. Men det är inte identiskt med token-gränssnittet som Home Assistant använder, och inte heller med det lokala V2- eller V3-enhetsprotokollet. Någon officiellt annonserad avveckling av lokal PortaSplit-styrning med ett konkret datum är inte dokumenterad. I juni 2026 visades dessutom att det SmartHome-token-API som påstods vara avstängt fortfarande fungerade – den tidigare begäran från community-biblioteket var bara ofullständig.

Den här artikeln är aktuell den 25 juli 2026.

## Varför den tidigare bedömningen måste korrigeras

I [den första artikeln om frågan om molntoken](/blog/midea-portasplit-home-assistant) återgav jag i huvudsak varningen från projektet `Midea AC LAN` som en annonserad avveckling av molngränssnitten. Det motsvarade ordalydelsen i projektets README, men var för starkt formulerat som faktapåstående.

Varningen är fortfarande relevant som en riskindikation. Den är dock inte en publicerad Midea-färdplan. Framför allt finns nu nytt tekniskt material som ifrågasätter en väsentlig del av den tidigare tolkningen.

## Så fungerar den lokala PortaSplit-styrningen

Home Assistant-integrationen `Midea Smart AC` beskriver uttryckligen sin arkitektur som lokal styrning. För nyare V3-enheter används Midea-molnet endast under installationen för att hämta enhetsspecifik token och nyckel. Därefter sparar integrationen båda värdena lokalt och behöver ingen ytterligare molnanslutning för själva styrningen. Projektet dokumenterar detta under [„Note On Cloud Usage"](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

För manuellt konfigurerade V3-enheter kräver `Midea Smart AC` enhets-ID, IP-adress, port, token och nyckel. Den dokumenterade standardporten är `6444/TCP`; token och nyckel anges som 128 respektive 64 hexadecimala tecken. Dessa uppgifter finns i [dokumentationen för manuell konfiguration](https://github.com/mill1000/midea-ac-py#manual-configuration).

En PortaSplit identifierades exempelvis i ärendehanteraren för `Midea AC LAN` som enhetstyp `0xAC`, modell `00000Q1D` och protokollversion 3. Samma användare kunde sedan lägga till den i Home Assistant via NetHome Plus. Det konkreta förloppet dokumenteras i [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

Det avgörande är åtskillnaden:

- Molntjänsten används för att hämta de lokala åtkomstuppgifterna.
- Den senare styrningen sker direkt i LAN.
- Ett avbrott i token-tjänsten förhindrar därför främst nya installationer.
- Det avslutar inte automatiskt en redan konfigurerad lokal anslutning.

Det senare motsvarar också den uttryckliga beskrivningen av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Varifrån avvecklingsvarningen kommer

Den varningstext som syns i dag lades till i dokumentationen den 19 maj 2025 genom [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

Motiveringen kan sammanfattas så här:

- De lokala token-värdena skulle sakna utgångsdatum.
- Olika Home Assistant-projekt använde emulerad eller extraherad appkryptering.
- Detta skulle innebära ett säkerhetsproblem.
- Midea skulle därför stegvis stänga de tidigare token-tjänsterna.
- På lång sikt skulle lokal V1-styrning trängas undan av ett molnbaserat V2-API.

I juli 2025 justerades dokumentationen ännu en gång genom [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). I stället för SmartHome-molnet nämndes nu NetHome Plus som tillfällig token-källa. Själva avvecklingsvarningen fanns kvar.

Den underliggande diskussionen är dock mer försiktigt formulerad än README-filen.

I [kommentaren från Midea-AC-LAN-underhållaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) sägs i huvudsak att NetHome Plus möjligen bara är en tillfällig lösning och att Midea, enligt hans uppfattning, har en ny helt molnbaserad V2-tjänst.

Underhållaren av `midea-msmart` svarade att han också hade misstänkt att ett nytt V2-API fanns, men på grund av brist på egna Midea-enheter bara kunde undersöka det begränsat. Detta står i [det direkta svarskommentaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Därmed är källsituationen tydligare:

- Varningen kommer från erfarna community-utvecklare.
- Den bygger på observerade förändringar och deras tekniska bedömning.
- En av underhållarna beskriver uttryckligen V2-migreringen som sin uppfattning.
- Den andra talar om ett antagande.
- Varken pull requesten eller diskussionen länkar till ett officiellt Midea-meddelande om avveckling eller ett datum.

Det gör inte varningen värdelös. Men det gör den till en riskanalys, inte en bekräftad tillverkarfärdplan.

## Det avgörande nya fyndet från juni 2026

Den 15 juni 2026 infördes en korrigering i biblioteket `midea-local` som väsentligt förändrar den tidigare tolkningen.

Utgångspunkten var felet:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Detta fel uppstod vid hämtning av token och nyckel via SmartHome-molnet. Inloggning och enhetslistan fungerade fortfarande, men anropet till `/v1/iot/secure/getToken` avvisades.

Till en början såg det ut som ett avstängt eller obrukbart gränssnitt. En analys av begäran från den officiella SmartHome-appen visade dock en annan orsak: Appen skickade, utöver `udpid`, även fältet `applianceCodes`. Community-biblioteket hade inte skickat med detta fält.

Den korrigerade begäran innehåller nu:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Utvecklaren testade ändringen med ett verkligt SmartHome-konto och fyra V3-luftkonditioneringsenheter av typen `0xAC`:

- Utan `applianceCodes` svarade servern med fel 3004.
- Med `applianceCodes` levererade den giltiga token-värden och nycklar.
- De returnerade värdena fungerade sedan för lokal V3-autentisering.

Den fullständiga undersökningen, testresultaten och kod-diffen dokumenteras i [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). Den tillhörande oföränderliga committen är [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Även i den aktuella källkoden används fortfarande exakt denna ändpunkt:

```text
/v1/iot/secure/getToken
```

Dessutom skickas nu `applianceCodes` med. Detta går direkt att följa i den aktuella [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Den nuvarande versionen av `Midea AC LAN` inkluderar `midea-local==6.11.0` och deklarerar sig fortfarande som en `local_push`-integration. Båda framgår av det aktuella [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

Det generella påståendet att SmartHome-token-API:t skulle ha stängts är därmed motbevisat, åtminstone för de konton och enheter som testades i juni 2026. Korrekt vore:

> Den tidigare token-hämtningen fungerade inte längre efter en ändring av det förväntade begärandeformatet. Efter anpassning till det format som den officiella appen använder levererade samma V1-ändpunkt åter giltiga lokala åtkomstuppgifter.

Regionala skillnader, avvikande konton eller enhetstyper som inte stöds är därmed inte uteslutna. Men det var uppenbarligen ingen global avveckling.

## Varför „V2" så lätt missförstås här

I Midea-sammanhang används minst tre sinsemellan oberoende versionsbeteckningar.

| Begrepp | Betydelse |
| --- | --- |
| Lokalt V2-/V3-protokoll | Generation för direkt kommunikation mellan integration och enhet |
| V1-/V2-appändpunkt | Versionsnummer för en enskild HTTP-ändpunkt i Midea-apparnas backend |
| Moln-till-moln-API V2 | Officiellt partner-API för auktoriserade tredjepartsföretag |

### Lokalt V2 och V3

I det lokala enhetsprotokollet anger V2 respektive V3 enhetens kommunikationsgeneration. Nyare V3-enheter behöver token och nyckel för lokal autentisering. `Midea Smart AC` dokumenterar detta krav i sin [konfigurationsguide](https://github.com/mill1000/midea-ac-py#manual-configuration).

Denna protokollversion har inget att göra med det officiella moln-till-moln-API:t V2.

### V1 och V2 i app-URL:er

Även inom samma app kan ändpunkter med olika versionsnummer användas samtidigt. Ett `/v2/` i URL-sökvägen betyder därför inte att hela plattformen har ställts om till en ny arkitektur.

Den aktuella `midea-local`-koden använder fortfarande [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) för token och nyckel. Andra funktioner kan ändå ligga på sökvägar med andra versionsnummer.

### Officiellt moln-till-moln-API V2

Midea dokumenterar faktiskt ett [officiellt moln-till-moln-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Det använder bland annat:

- OAuth 2.0
- `client_id` och `client_secret`
- kortlivade access-token och refresh-token
- HMAC-SHA256-signaturer
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- molnbaserade statusförfrågningar och styrkommandon

Detta är ett kontrollerat partnergränssnitt. Det nödvändiga `client_secret` tilldelas en tredjepartsleverantör av Midea. En vanlig PortaSplit-ägare får det inte bara via sitt MSmartHome-konto. Kraven och signaturreglerna beskrivs i [den officiella V2-dokumentationen](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Detta API uppstod dessutom inte först 2025. Dokumentationen innehåller exempel på begäranden med tidsstämplar från 2018 och en Java-kommentar från den 18 april 2019. Partnergränssnittet V2 fanns alltså redan långt före varningen i `Midea AC LAN`.

## Midea ersätter faktiskt ett V1-API – men ett annat

Midea har också ett äldre officiellt moln-till-moln-gränssnitt under `/v1/open/...`. Dess dokumentation har uttryckligen en anmärkning om att det inte längre rekommenderas, kan avvecklas i framtiden och att den nya V2-dokumentationen bör användas. Detta står i Mideas [dokumentation för det gamla moln-till-moln-API:t](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Denna anmärkning är en verklig officiell V1-till-V2-migrering. Men den gäller partnerändpunkterna:

```text
/v1/open/...
           ↓
/v2/open/...
```

Token-hämtningen som används av Home Assistant-biblioteken är däremot:

```text
/v1/iot/secure/getToken
```

Och den lokala PortaSplit-anslutningen sker sedan inte alls via en sådan moln-URL, utan direkt i hemnätverket.

Att likställa de tre gränssnitten bara på grund av versionsnumret „V1" skulle därför inte vara tekniskt motiverat.

## Finns det redan en helt molnbaserad Home Assistant-integration?

Med [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) finns numera en community-integration som styr Midea-enheter via molnet i stället för direkt via LAN.

Inte heller detta är dock ett belägg för att det officiella partner-API:t V2 redan har ersatt lokal styrning. Den aktuella källkoden för `Midea Auto Cloud` använder bland annat:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Dessa ändpunkter kan ses i den aktuella [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Integrationen emulerar alltså privata app- respektive konsumentmolnfunktioner. Den använder inte helt enkelt det dokumenterade `/v2/open/...`-partnergränssnittet.

Det finns alltså redan ett molnbaserat alternativ. Men det medför också de vanliga beroendena för en molnintegration: internetåtkomst, ett fungerande användarkonto, tillgängliga Midea-servrar och fortsatt kompatibla privata ändpunkter.

## Vad betyder detta konkret för PortaSplit-ägare?

### Redan konfigurerad lokal styrning

För en redan konfigurerad PortaSplit är situationen relativt lugn. `Midea Smart AC` sparar token och nyckel lokalt efter installationen och behöver enligt sin egen [molndokumentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) ingen molnanslutning för fortsatt styrning.

En avveckling av ren token-hämtning skulle därför inte automatiskt avsluta den befintliga lokala anslutningen.

### Nyinstallation eller återställning

Risken är större vid:

- en ny Home Assistant-installation
- byte till en annan integration
- en förlorad eller skadad säkerhetskopia
- byte av Wi-Fi-modul
- ändringar i enhetstilldelningen
- en ny parkoppling, om enhetens åtkomstuppgifter då ändras

I sådana fall måste integrationen hämta token och nyckel på nytt, eller så måste användaren ange dem manuellt. Att `Midea Smart AC` stöder manuell konfiguration beskrivs i dess [konfigurationsdokumentation](https://github.com/mill1000/midea-ac-py#manual-configuration).

Om en fabriksåterställning eller ny parkoppling nödvändigtvis skapar nya åtkomstuppgifter för varje PortaSplit är inte officiellt dokumenterat och bör därför inte påstås generellt.

### En verklig avveckling av LAN-styrningen

För att en redan konfigurerad PortaSplit inte längre ska acceptera sina lokalt sparade åtkomstuppgifter måste dessutom enhetens eller Wi-Fi-modulens beteende ändras – till exempel genom ny firmware eller en ändrad autentiseringsmetod.

Enbart en avveckling av molnändpunkten `/v1/iot/secure/getToken` tar inte automatiskt bort åtkomstuppgifterna som redan finns i enheten och Home Assistant. Det följer av den uppdelning mellan engångshämtning från molnet och efterföljande LAN-styrning som dokumenteras av [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

En sådan framtida enhetsändring är tekniskt möjlig. Men jag har inte hittat något konkret tillkännagivande eller avvecklingsdatum specifikt för PortaSplit i Mideas offentligt tillgängliga dokumentation.

## Vad jag fortfarande skulle rekommendera

Trots de relativiserande insikterna är en säkerhetskopia klok.

För V3-enheter rekommenderar `Midea AC LAN` uttryckligen att den skapade JSON-konfigurationen sparas utanför HAOS. Den aktuella rekommendationen finns direkt i [projektets README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Då gäller följande:

- Behandla token och nyckel som lösenord.
- Ladda inte upp JSON-filen till ett offentligt Git-repository.
- Publicera inga osanerade felsökningsloggar.
- Kryptera säkerhetskopian.
- Skapa dessutom en fullständig Home Assistant-säkerhetskopia.
- Kontrollera den aktuella funktionen före firmware- och integrationsuppdateringar.
- Testa den lokala styrningen igen efter uppdateringar.

En säkerhetskopia är ett rimligt skydd mot molnändringar, integrationsproblem och egna misstag. Men den är inget tecken på att en avveckling är nära förestående. Hur en PortaSplit konfigureras korrekt och säkras i hemnätverket beskrivs i [den praktiska delen om installation](/blog/midea-portasplit-home-assistant-einrichten).

## Bedömning utifrån tillgängliga belägg

Varningen från `Midea AC LAN` bör tas på allvar, men sättas i rätt sammanhang.

Den dokumenterar en rimlig långsiktig risk: Midea kan betrakta lokala token-värden utan utgångsdatum som ett säkerhetsproblem, ytterligare begränsa möjligheten att hämta sådana token-värden eller knyta framtida enheter hårdare till molnet.

Däremot är en officiellt annonserad och tidsatt avveckling av lokal PortaSplit-styrning inte belagd.

Det aktuella tekniska läget visar till och med motsatsen till en redan genomförd avveckling: I juni 2026 levererade den fortfarande använda V1-token-ändpunkten giltiga åtkomstuppgifter efter att begäran hade anpassats till den officiella SmartHome-appens format. Den aktuella korrigeringen ingår i dag i biblioteket som används av `Midea AC LAN`.

Även det officiella Midea moln-till-moln-API:t V2 finns. Men det är ett äldre partnergränssnitt med begränsad åtkomst och inte automatiskt efterföljaren till det lokala PortaSplit-protokollet.

Den nyktra slutsatsen är därför:

> Skapa en säkerhetskopia, följ integrationerna och ha molnberoenden i åtanke – men avskriv inte den lokala PortaSplit-styrningen i förtid på grund av ett obekräftat antagande om avveckling.

## Källor

1.  [Midea AC LAN: aktuell README och avvecklingsvarning](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Varningens ordalydelse, rekommendation om säkerhetskopia och åtskillnaden mellan äldre V2- och nyare V3-enheter.

2.  [Midea AC LAN PR #578 från den 19 maj 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Införande av varningen om stegvis avveckling av token-tjänsterna och den påstådda migreringen till ett molnbaserat V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Byte av den dokumenterade token-källan till NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskussion om den felaktiga SmartHome-token-hämtningen och den tillfälliga användningen av NetHome Plus.

5.  [Kommentar från Midea-AC-LAN-underhållaren om den misstänkta V2-migreringen](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Anger uttryckligen påståendet om det nya V2-molnet som den egna uppfattningen.

6.  [Svar från midea-msmart-underhållaren](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beskriver existensen av ett nytt V2-API som ett antagande och hänvisar till begränsade möjligheter till reverse engineering.

7.  [midea-local PR #470 från den 15 juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analys av fel 3004, inspelning av den officiella appbegäran, tillägg av `applianceCodes` och framgångsrikt test med fyra V3-luftkonditioneringsenheter.

8.  [Oföränderlig commit för SmartHome-getToken-korrigeringen](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Exakt kod-diff för den införda korrigeringen.

9.  [Aktuell midea-local-molnkod](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Fortfarande använd ändpunkt `/v1/iot/secure/getToken` och aktuellt begärandefält `applianceCodes`.

10.  [Aktuellt manifest för Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Använd version av `midea-local` och klassificering som lokal push-integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentation av lokal styrning, engångshämtning från molnet för V3-enheter och manuell konfiguration med token och nyckel.

12.  [Midea AC LAN Issue #607 om PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkret PortaSplit-exempel med enhetstyp `0xAC`, modell `00000Q1D`, protokollversion 3 och lyckad installation via NetHome Plus.

13.  [Officiellt Midea moln-till-moln-API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, klient-ID, klienthemlighet, access- och refresh-token, signaturmetod och `/v2/open/...`-ändpunkter.

14.  [Officiellt Midea moln-till-moln-API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Officiell anmärkning om att det gamla `/v1/open/...`-partnergränssnittet inte längre rekommenderas och kan avvecklas i framtiden.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) och [aktuell molnkod](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Community-integration för fullständig molnstyrning och de privata V1-appändpunkter som faktiskt används.
