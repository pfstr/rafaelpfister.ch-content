---
title: "RustDesk 1.5.1: signerade uppdateringar, Trusted Devices V2 och härdning av filöverföring"
navTitle: "RustDesk 1.5.1"
description: "RustDesk 1.5.1 kontrollerar skrivbordsuppdateringar med hjälp av signerade metadata, introducerar Trusted Devices V2 och härdar filöverföring, Windows-installationsprogram och automatisk inloggning. Dessutom tillkommer omkring 35 felrättningar, främst för DRM- och Wayland-capture. En uppdatering av klienterna rekommenderas, servern förblir oförändrad."
date: "2026-11-02"
kategorie: "Fjärrsupport och support"
timeToRead: "5 min läsning"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
slug: "rustdesk-1-5-1-signerade-uppdateringar-trusted-devices-v2-och-hardning-av-filoverforing"
translationId: article-c95ef39dfcc09875
translationOf: rustdesk-1-5-1
url: https://rafaelpfister.ch/sv/blog/rustdesk-1-5-1-signerade-uppdateringar-trusted-devices-v2-och-hardning-av-filoverforing
translationSourceHash: aa77142cfee63f49b15c18d6d1aa69368ec9e6ac75dee970c2907402fc21fc91
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:35:49.833Z
translationReview: automatic
---

# RustDesk 1.5.1: signerade uppdateringar, Trusted Devices V2 och härdning av filöverföring

RustDesk publicerade klientversion 1.5.1 den 1 november 2026, drygt en månad efter 1.5.0 (30 september 2026). Versionen innehåller fem nya funktioner, däribland kontroll av signerade uppdateringsmetadata och en ny version av Trusted Devices för tvåfaktorsinloggning, tre ändringar och omkring 35 felrättningar. Flera av dem härdar säkerhetsrelevanta flöden: validering av nedladdningssökvägar, skyddad överlämning till installationsprogrammet i Windows, varning för okrypterade API-servrar och en mer begränsad automatisk inloggning. Tillverkaren nämner ingen CVE eller Security Advisory för detta (per den 2 november 2026). Versionen riktar sig till alla som använder RustDesk-klienter på Windows, macOS, Linux, Android eller iOS, särskilt med egen server. Installation och drift av en egen RustDesk-server beskrivs i artikeln [RustDesk: konfigurera alternativet till TeamViewer med öppen källkod](/blog/rustdesk-teamviewer-alternative) beskriven.

## Versionerna i översikt

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 1.5.1 | 01.11.2026 | Klientversion | signerade uppdateringsmetadata, Trusted Devices V2, kortkommandon, validering av nedladdningssökvägar, härdning av installationsprogrammet i Windows, varning för HTTP-API-server, korrigeringar för DRM, Wayland, X11 och QSV |

Open-source-servern (`rustdesk-server`) påverkas inte av denna version. Dess senaste version är 1.1.16 från den 20 juli 2026.

## Signerade uppdateringsmetadata

Med PR #15914 hämtar skrivbordsuppdateraren i Windows och macOS sin information från signerade versionsmetadata. Varje version innehåller tillgångarna `rustdesk-update.json` (version, URL, filnamn, storlek och SHA-256 per artefakt) och `rustdesk-update.json.sig` (Ed25519-signatur). Klienten kontrollerar signaturen innan den utvärderar metadata och accepterar endast nedladdnings-URL:er under versionen från `rustdesk/rustdesk`. Om kontrollen misslyckas avbryts uppdateringen; det finns ingen återgång till osignerade data. Nedladdade installationsprogram hålls i en skyddad katalog fram till start och hashas på nytt före körning med förhöjda rättigheter.

Enligt PR bevisar signaturen äkthet och integritet, men inte aktualitet: skydd mot återinstallation av äldre, korrekt signerade versioner (rollback) ingår inte. Den portabla varianten i Windows packar fortfarande upp och startar från en katalog som användaren kan skriva till; härdningen för detta har enligt PR flyttats till en separat branch.

## Trusted Devices V2

PR #16548 ersätter den tidigare förtroendemodellen via maskinvaru-ID med token som utfärdas av värden. För en ny registrering krävs fortfarande det permanenta lösenordet och TOTP; en Trusted Device-token sparar därefter endast TOTP-inmatningen. Värden lagrar endast SHA-256-hashen av tokenet i krypterad form, och kontrollanten binder sitt token till värdens verifierade offentliga nyckel. Vid direkta IP-anslutningar erbjuds ingen förtroendestatus, eftersom värdens identitet inte verifieras där.

Följande punkter från PR är relevanta för drift:

- **Giltighet:** kan ställas in på 1, 7, 30 eller 90 dagar eller obegränsat, standard är 90 dagar. Förtroendestatus kan förnyas, tas bort eller raderas samlat var för sig i inställningarna.
- **Återkallande:** En ändring av lösenord, 2FA eller Trusted Device-inställningen återkallar berörda token. Värden har högst 128 poster.
- **Blandade versioner:** Befintliga förtroendestatusar som inte har löpt ut förblir giltiga efter uppdatering av värden. Nya registreringar kräver 1.5.1 på båda sidor; i blandade sessioner frågar RustDesk efter TOTP och erbjuder inte ”Trust this device”.
- **Nedgradering:** Enligt PR inte säker, eftersom äldre versioner inte kan läsa de importerade förtroendestatusarna.
- **Kända begränsningar:** Ett återkallande sparas först lokalt och överförs asynkront till rotkonfigurationen; en omstart före synkroniseringen kan återställa en äldre förtroendestatus. Utgångsdatumet baseras på värdens systemklocka.

Planera därför uppdateringen så att värdar och kontrollanter byter till 1.5.1 inom kort, och undvik att återställa till 1.5.0.

## Ytterligare säkerhetsrelevanta korrigeringar

- **Nedladdningssökvägar (PR #16601):** Nedladdningar avvisar osäkra filnamn och sökvägar som leder ut från den valda målkatalogen eller går via symboliska länkar. Fjärrfilistor kontrolleras före nedladdning, och överskrivning av befintliga filer samt återupptagning av avbrutna nedladdningar kräver bekräftelse.
- **Windows-installationsprogram (PR #16554, #16562):** Skripten för överlämning till installationsprogrammet med förhöjda rättigheter ligger inte längre i `System32`, utan i en katalog under Windows Temp med begränsad DACL (Administrators och SYSTEM). Tjänsten styrs via den inbyggda Service Control. Bakgrunden var bland annat att Bitdefender blockerade det tillfälliga batch-skriptet (Issue #16557).
- **Okrypterad API-server (PR #16616):** Serverinställningarna visar en varning när API-servern är tom eller använder en fjärransluten HTTP-adress. HTTPS och lokala adresser utlöser ingen varning. Anslutningen blockeras inte.
- **Automatisk inloggning (PR #16520):** Den automatiska inloggningen matar endast in operativsystemets lösenord på inloggnings- eller låsskärmen.
- **Privacy Mode (PR #16507):** En nekad Privacy Mode-begäran stänger inte längre av Privacy Mode för en annan anslutning.

Om du driver en egen server med en `http://`-adress som API-server visas den nya varningen för dina användare. Byt API-servern till HTTPS innan du distribuerar 1.5.1 för att undvika frågor.

## Nya funktioner och ändringar

Utöver punkterna som redan nämnts innehåller 1.5.1 kortkommandon (PR #14933), en knapp i adressboken för att öppna webbkonsolen samt direkt filöverföring till SAF-mappar i Android. Kodningen av statiskt bildinnehåll efter rullning har ändrats (skarpare visning), liksom uppdateringsbudgetarna för MediaCodec. Hanteringen av `ConfigUpdate` har tagits bort.

## Felrättningar per plattform

- **Linux, DRM-capture:** korrekt storlek för roterade skärmar, ingen uppväckning av vilande skärmar vid start, kassering av dubbla bildrutor och bildfrekvens anpassad till kodaren.
- **Linux, Wayland:** PipeWire-capture utan väntetider, korrekt tilldelning av portalströmmen till utgången, tangentbordslayout vid inmatning och en öppen portalbegäran blockerar inte längre servern.
- **Linux, allmänt:** korrekt aktiv användare under Flatpak, krasch vid stängning av GTK-fönstret, sudo-fråga utan felaktigt ”Sorry, try again.” och med stöd för sudo-rs, xrandr-utgångsnamn, RandR-skärmfråga under X11.
- **Windows:** läsning av `host=` från programsökvägar med icke-ASCII-tecken, explicit namngiven körbar fil för Privacy-fönstret, hängningar i QSV-kodaren.
- **macOS:** länk till inställningarna för Screen Recording, släpp av musknappen.
- **Mobil:** ingen fördröjd Mouse-Down efter att den flytande knappen släppts.
- **Nätverk:** HTTP-API-fel 407 via HTTPS-proxyer, IPv6-ID-server utan hakparenteser vid härledning av API-servern, ett felmeddelande i stället för två när båda WebRTC-anslutningsförsöken misslyckas.
- **Övrigt:** borttagna skrivare behålls, replay-buffer i terminalen, tangentbordsnavigering i fillistan, inläsning av peers utan angiven appkatalog, klick på peer-kort i webbklienten.

## Rekommenderat tillvägagångssätt

Målversionen för alla klienter är 1.5.1. Uppdatera värdar och kontrollanter så nära i tid som möjligt så att nya Trusted Device-registreringar fungerar; en nedgradering till 1.5.0 är enligt PR #16548 inte säker. Kontrollera före utrullningen att din egen API-server är nåbar via HTTPS. Open-source-servern behöver inte uppdateras för denna version. Release Notes innehåller inga uppgifter om Known Issues eller supportperioder (per den 2 november 2026).

## Källor

1.  [GitHub: RustDesk Release 1.5.1](https://github.com/rustdesk/rustdesk/releases/tag/1.5.1): Release Notes med datum, nya funktioner, ändringar och felrättningar.

2.  [GitHub: RustDesk Release 1.5.0](https://github.com/rustdesk/rustdesk/releases/tag/1.5.0): Föregående version från den 30 september 2026.

3.  [GitHub: PR #15914, verify signed update metadata](https://github.com/rustdesk/rustdesk/pull/15914): Förloppet för signaturkontrollen, avsaknat rollback-skydd, portabel variant.

4.  [GitHub: PR #16548, Trusted Devices V2](https://github.com/rustdesk/rustdesk/pull/16548): Token-modell, kompatibilitet med äldre versioner, nedgradering och kända begränsningar.

5.  [GitHub: PR #16601, Download path validation](https://github.com/rustdesk/rustdesk/pull/16601): Kontroll av filnamn, sökvägar och symboliska länkar vid nedladdningar.

6.  [GitHub: PR #16554, installationsskript i skyddade Temp-kataloger](https://github.com/rustdesk/rustdesk/pull/16554): Flytt från System32, DACL och koppling till Bitdefender.

7.  [GitHub: PR #16616, varning vid osäker API-server](https://github.com/rustdesk/rustdesk/pull/16616): Villkor och text för den nya varningen.

8.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): inga publicerade Advisories (per den 2 november 2026).

9.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): senaste serverversionen 1.1.16 från den 20 juli 2026.

10.  [RustDesk: konfigurera alternativet till TeamViewer med öppen källkod](/blog/rustdesk-teamviewer-alternative): Konfiguration av klient och egen server.
