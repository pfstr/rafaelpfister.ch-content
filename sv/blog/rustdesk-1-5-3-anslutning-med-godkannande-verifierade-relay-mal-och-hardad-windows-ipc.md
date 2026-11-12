---
title: "RustDesk 1.5.3: anslutning med godkännande, verifierade relay-mål och härdad Windows-IPC"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 ger anslutningar utan lösenord med godkännande från motparten, accepterar relay-begäranden endast för erbjudna adresser och autentiserar IPC-pipes i Windows. Serverstrategier får inte längre ändra känsliga inställningar. En uppdatering av alla klienter rekommenderas."
date: "2026-11-11"
kategorie: "Fjärrunderhåll och support"
timeToRead: "5 min lästid"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-anslutning-med-godkannande-verifierade-relay-mal-och-hardad-windows-ipc"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/sv/blog/rustdesk-1-5-3-anslutning-med-godkannande-verifierade-relay-mal-och-hardad-windows-ipc
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:34:47.972Z
translationReview: automatic
---

# RustDesk 1.5.3: anslutning med godkännande, verifierade relay-mål och härdad Windows-IPC

RustDesk släppte version 1.5.3 av klienten den 9 november 2026, åtta dagar efter 1.5.1. Version 1.5.2 finns inte listad på releasesidan. Utgåvan innehåller en ny funktion (anslutning med godkännande från motparten), en ändring av relay-logiken och 22 felrättningar. Flera av dem berör säkerhetsrelevanta processer: kontroll av relay-adresser och relay-UUID:er, autentisering av Named Pipes i Windows, låsta konfigurationsnycklar i serverstrategier och återigen installeraröverlämningen i Windows. Tillverkaren anger ingen CVE eller Security Advisory för detta (status 11 november 2026). Utgåvan riktar sig till alla som driver RustDesk-klienter, särskilt i Windows och med egen server. Föregående status med signerade uppdateringar och Trusted Devices V2 beskrivs i artikeln [RustDesk 1.5.1: signerade uppdateringar, Trusted Devices V2 och härdning av filöverföring](/blog/rustdesk-1-5-1).

## Versionerna i korthet

| Version | Datum | Typ | Huvudpunkter |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Klientutgåva | Request approval, relay endast till erbjudna adresser, IPC-autentisering i Windows, låsta nycklar i strategier, installerarfixar (F-Secure, Junction), korrigeringar för macOS, DRM, Wayland, Android och iOS |

Open-source-servern (`rustdesk-server`) har inte fått någon ny utgåva; den senaste versionen är fortfarande 1.1.16 från den 20 juli 2026.

## Ny funktion: Request approval

Med PR #16784 kan en anslutning upprättas utan RustDesk-lösenord. Motparten måste godkänna den i en dialogruta. Enligt PR skickar klienten i detta läge inget lösenord från någon källa, inte heller från lokal lagring eller adressboken. Ett uttryckligen angivet lösenord ignoreras. Valet visas som undermenyn ”Request approval” i ID-rullgardinsmenyn och i peer-kortens menyer, i varje fall med de anslutningstyper som plattformen och motparten stöder (anslutning, filöverföring och fler).

För skript och genvägar anger PR kommandoradsparametern `--request-approval` och URI-parametern `request-approval=true`. Läget behålls vid återanslutning, när en session flyttas till ett annat fönster och vid ytterligare anslutningar från fjärrverktygsfältet. Normala anslutningar behåller sitt tidigare lösenordsbeteende.

Enligt PR gäller följande begränsningar:

- Tillgängliga autentiseringsmetoder förhandlas inte med motparten. Om endast lösenordsinloggning är tillåten där eller om begränsningar gäller på inloggningsskärmen, gäller dessa som tidigare.
- Om det redan finns ett fönster för samma peer och samma anslutningstyp aktiverar RustDesk detta utan att ändra inloggningssättet. Stäng först den befintliga sessionen för en ny anslutning med godkännande.
- Vid tunnelmultiplexering kan en redan godkänd tunnel återanvändas utan förnyad förfrågan.

PR innehåller inga uppgifter om kompatibilitet med äldre versioner hos motparten.

## Relay endast till erbjudna adresser

PR #16858 stänger en lucka i relay-förhandlingen. Tidigare tog den kontrollerade sidan okontrollerat över relay-adressen och UUID:n från ett vidarebefordrat `RequestRelay`-meddelande och anslöt till den angivna adressen. Nu sparar den kontrollerade sidan de relay-adresser som har erbjudits den, samt det relay som den själv skulle välja (högst 32 poster, giltiga i tio minuter från det senaste meddelandet). Begäranden med ett annat relay avvisas före namnuppslagningen. Båda sidor avvisar dessutom UUID:er som är tomma eller längre än 128 byte eller innehåller andra tecken än ASCII-bokstäver, siffror, `-` och `_`. Enligt PR motsvarar detta kontrollen som hbbs redan utför.

En avvisad begäran får inget svar, och controllerns relay-försök går därför ut i timeout i stället för att ge ett felmeddelande. Ett relay som du tar bort från konfigurationen accepteras senast tio minuter efter dess senaste meddelande inte längre.

För operatörer av egen server är en begränsning relevant: Om hbbs körs med den som föråldrad markerade flaggan `--mask`, avvisas återgången till ett relay efter ett misslyckat WebRTC-försök när hbbs har skrivit om relay-adressen för LAN. Denna punkt var öppen vid mergningen. Kontrollera om din hbbs-instans använder `--mask` och testa relay-anslutningar efter uppdateringen.

## Ytterligare säkerhetsrelevanta korrigeringar

- **Windows-IPC (PR #16755):** Enligt PR kunde andra användare tidigare skapa ytterligare serverinstanser av Named Pipes för huvud- och tjänste-IPC och på så sätt fånga upp anslutningar till RustDesk, läsa IPC-data eller förfalska konfigurationssvar. Klienter öppnar nu pipes utan denna rättighet och autentiserar servern före datautbytet. Tjänste-pipes måste ägas av SYSTEM. Enligt PR är en Denial of Service fortfarande möjlig om en angripare först reserverar pipe-namnet, och äldre versioner med öppnare åtkomstregler kan inte upprätta anslutning förrän efter uppdateringen.
- **Strategier (PR #16696):** Strategier som distribueras från servern via heartbeat får inte längre ändra känsliga inställningar. Bland annat är `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` och `websocket-port` låsta. Fortsatt möjliga att ange är exempelvis `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` och `allow-auto-update`. Avvisade nycklar loggas utan sina värden. Författaren beskriver uttryckligen ändringen som begränsad härdning: strategisvar är fortfarande inte autentiserade.
- **Windows-installeraren (PR #16889, #16908, #16871):** Överlämningen via Named Pipe som infördes med 1.5.1 har tagits bort igen, eftersom den enligt PR förhindrade installation och ”Stop service” på system med F-Secure DeepGuard (fel `0x800700E8`). Skyddet av de förberedda filerna bygger nu på de restriktiva DACL:erna från 1.5.1 och på egna staging-kataloger vars handles hålls öppna under överlämningen. Om `C:\Windows\Temp` omdirigeras via en Junction använder installeraren i stället systemets ProgramData-mapp, i stället för att avbryta med ett fel.
- **macOS (PR #16729, #16879):** Privacy Mode stängs av vid sessionens slut före låsning, och tjänstedaemonen startar direkt i stället för via `sh`.
- **IP-vitlista (PR #16692):** Mellanslag runt poster i IP-vitlistan ignoreras.

Om installationen i Windows avbryts med F-Secure bör du testa 1.5.3 på en berörd enhet före bred utrullning. Enligt PR #16889 saknades fortfarande bekräftelse från rapportören på F-Secure-fixen vid mergningen. Om du hittills distribuerar konfigurationsvärden som API-, ID- eller relay-servrar via strategier måste du framöver ange dem på annat sätt.

## Felrättningar efter plattform

- **Windows:** Avinstallationen fortsätter efter fel vid konfigurationsavstämningen, tjänste-IPC försöks igen innan GUI:t avslutas, Clipboard-Owner-markören skrivs först.
- **Linux:** Inloggningsskärmen under greetd med Wayland tillhandahålls via DRM, oförändrat skärminnehåll kodas inte på nytt under DRM, oassocierade Wayland-capture-fel visas med sitt faktiska meddelande i stället för ”x11 expected”.
- **Android:** inga fler krascher vid urklippsinnehåll över 1 MB, frames kopieras innan bilden frigörs, hebreiska och indonesiska mappas korrekt från äldre locale-koder.
- **iOS:** Startskärmen följer systemets Dark Mode.
- **Övrigt:** Port-forwarding håller KCP-handtaget öppet och skickar en avslutsorsak, sidofälten i godkännandefönstret stannar på den aktuella bildskärmen i Windows, ljudinmatningsenheter söks endast under inmatningsenheter, minsta storlek för Sciters huvudfönster skalas inte längre dubbelt.

## Kända problem

- **Rollback-skydd för signerade uppdateringar (från 1.5.1):** öppet. Release Notes för 1.5.3 nämner ingen ändring av detta.
- **Portabel variant i Windows (från 1.5.1):** Härdningen hade enligt PR #15914 flyttats till en separat branch. Release Notes för 1.5.3 nämner ingen ändring av detta.
- **Trusted Devices V2 (från 1.5.1):** De kända begränsningarna (asynkront återkallande, utgång enligt värdens systemklocka) behandlas inte i Release Notes för 1.5.3.
- **Relay-fallback med `--mask` (nytt):** öppet enligt PR #16858.
- **Pipe-namn reserverat i förväg (nytt):** Denial of Service är fortfarande möjlig enligt PR #16755.
- **Avbruten UAC-förfrågan (nytt):** De förberedda filerna ligger enligt PR #16889 kvar i den skyddade katalogen och kan tas bort av en administratör.
- **Strategisvar (nytt):** fortfarande inte autentiserade. Vid deployment med HTTP i stället för HTTPS kan de fortfarande tillåtna alternativen enligt PR #16696 manipuleras.

## Rekommenderat tillvägagångssätt

Målversionen för alla klienter är 1.5.3. Eftersom äldre versioner enligt PR #16755 inte kan ansluta till de härdade IPC-pipes förrän efter uppdateringen, ska du i Windows uppdatera alla RustDesk-komponenter på en enhet samtidigt. Kontrollera före utrullningen om du distribuerar serveradresser eller nycklar via strategier, om hbbs körs med `--mask` och om F-Secure används på målsystemen. API-servern bör vara tillgänglig via HTTPS, vilket redan rekommenderades för 1.5.1. Open-source-servern behöver inte uppdateras för denna utgåva. Release Notes innehåller inga uppgifter om supportperioder (status 11 november 2026).

## Källor

1.  [GitHub: RustDesk Release 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): Release Notes med datum, ny funktion, ändring och felrättningar.

2.  [GitHub: RustDesk Releases](https://github.com/rustdesk/rustdesk/releases): versionslista utan release 1.5.2.

3.  [GitHub: PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784): förlopp, parametrar och begränsningar för anslutning med godkännande.

4.  [GitHub: PR #16858, relay endast till erbjudna adresser](https://github.com/rustdesk/rustdesk/pull/16858): allowlist för relay-adresser, UUID-kontroll och begränsning med `--mask`.

5.  [GitHub: PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755): skydd av Named Pipes, kompatibilitet och återstående risk.

6.  [GitHub: PR #16696, låsta nycklar i strategier](https://github.com/rustdesk/rustdesk/pull/16696): låsta och fortsatt tillåtna alternativ, avsaknad av autentisering av svaren.

7.  [GitHub: PR #16889, borttagning av pipe-överlämningen i installeraren](https://github.com/rustdesk/rustdesk/pull/16889): koppling till F-Secure DeepGuard och beteende vid avbruten UAC-förfrågan.

8.  [GitHub: PR #16908, skydd av installerarkatalogen](https://github.com/rustdesk/rustdesk/pull/16908): egna staging-kataloger med öppna handles.

9.  [GitHub: PR #16871, installerar-staging vid Junction](https://github.com/rustdesk/rustdesk/pull/16871): användning av ProgramData vid omdirigerad Windows Temp.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): inga publicerade advisories (status 11 november 2026).

11.  [GitHub: rustdesk-server Releases](https://github.com/rustdesk/rustdesk-server/releases): senaste serverversionen 1.1.16 från den 20 juli 2026.

12.  [RustDesk 1.5.1: signerade uppdateringar, Trusted Devices V2 och härdning av filöverföring](/blog/rustdesk-1-5-1): föregående release-status.
