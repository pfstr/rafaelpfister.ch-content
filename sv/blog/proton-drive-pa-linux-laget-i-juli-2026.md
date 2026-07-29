---
title: "Proton Drive på Linux: läget i juli 2026"
navTitle: "Proton Drive & Linux"
description: "Den officiella Linux-klienten är annonserad men ännu inte tillgänglig. På servrar kan Proton Drive för närvarande monteras med Rclone; det nya SDK:t visar den tekniska riktningen. Det som fortfarande saknas är maskinåtkomst begränsad till enskilda mappar eller uppgifter."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min lästid"
themen:
  - proton-drive
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-pa-linux-laget-i-juli-2026"
translationOf: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/sv/blog/proton-drive-pa-linux-laget-i-juli-2026"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:55:54.414Z
translationReview: automatic
translationSourceHash: 1b0af572e102121912376d523c1785ed1563e4ca5c17eee8d605c5000096b57e
---

För Windows och macOS erbjuder Proton Drive egna synkroniseringsklienter sedan 2023. Under Linux finns hittills bara webbgränssnittet, communityverktyg och ett officiellt SDK i förhandsversion. På en server är situationen ännu svårare, eftersom varken skrivbordssynkronisering eller interaktiv inloggning passar bra där.

Den här översikten beskriver läget i juli 2026. Förutom de publicerade färdplanerna bygger den på ett praktiskt test av Rclone-backendprogrammet [som dokumentlagring för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Linux-klienten är annonserad, men har ännu inget datum

I juni 2026 bekräftade Proton för första gången uttryckligen att en Linux-klient utvecklas. Den byggs på det nya, enhetliga SDK:t och ska använda samma tekniska grund som programmen för Windows och macOS. Det finns ännu inget datum eller någon offentlig beta.

Viktigt för sammanhanget: Det blir en **synkroniseringsklient för skrivbordet**. För skrivbordet löser den problemet. För serverprogram är en synkroniseringsklient däremot fel verktyg, eftersom en tjänst ska läsa filer direkt från Proton Drive och skriva dit dem. En synkroniseringsklient behåller en fullständig lokal kopia, precis det man vill undvika när lagringsutrymmet är begränsat.

## I dag gör Rclone det praktiska arbetet

Under Linux är Rclone med sin `protondrive`-backend för närvarande det mest mångsidiga verktyget. Det kan kopiera och synkronisera filer och är den enda tillgängliga lösningen som kan tillhandahålla Proton Drive som en lokal katalog via **FUSE-montering**. Två begränsningar är viktiga:

**Det är beta och bygger på ett återskapat API.** Proton dokumenterar inte sitt Drive-API offentligt; backendprogrammet bygger på reverse engineering. I testet fungerade det tillförlitligt, men vid snabba anropsföljder ströps det med inkonsekventa kataloglistor.

**För oövervakad drift frågar Rclone efter TOTP-nyckeln.** Konfigurationsguiden betecknar fältet som `otp_secret_key`. Det som avses är den permanenta nyckeln från 2FA-konfigurationen, inte den sexsiffriga kod som en autentiseringsapp just nu visar. Rclone lagrar detta värde maskerat och skapar själv en giltig TOTP-kod vid varje inloggning.

Den som av misstag anger en aktuell engångskod kan slutföra den första inloggningen. Nästa återautentisering misslyckas dock med fel 8002, eftersom Rclone inte kan använda samma kod en gång till.

Kontot förblir därmed skyddat mot ett isolerat stulet lösenord. En komprometterad server avslöjar dock både lösenord och TOTP-nyckel. För automatiserade åtkomster rekommenderas därför ett **dedikerat Proton-konto**.

Hur en sådan montering beter sig i Docker-miljöer, inklusive två odokumenterade fallgropar, beskrivs i [den separata artikeln om Rclone i containrar](/blog/rclone-mount-in-docker-container).

## Det officiella SDK:t visar vart utvecklingen är på väg

Parallellt övergår Proton sina program till ett **officiellt SDK** för JavaScript och C# med bindningar för Swift och Kotlin. Det offentliga kodförrådet innehåller också ett kommandoradsverktyg. Dess inloggningsmodell är renare än Rclone-backendens:

- `auth login` öppnar webbläsaren; inloggningen sker normalt **inklusive tvåfaktorsautentisering**
- sessionen hamnar i **operativsystemets nyckelring** (Keychain, Credential Manager, libsecret), och SDK:t förnyar den självt
- därefter: lista filer, ladda upp och kontrollera delningar med maskinläsbar JSON-utdata

Lösenord och TOTP-nyckel behöver då inte finnas i en konfigurationsfil. För serverdrift återstår ändå tre begränsningar: CLI:t kan **inte montera ett filsystem**, inloggningen öppnar en webbläsare och Proton klassar ännu inte SDK:t som produktionsmoget för tredjepartsprogram. Lanseringen är planerad till slutet av 2026 eller början av 2027.

## Den egentliga luckan: maskinåtkomster

Problemets kärna ligger en nivå djupare än klient eller SDK: **Proton har inga maskinåtkomster.** Inget applösenord, inget tjänstkonto, ingen token med begränsad omfattning. Varje automatisering, oavsett om det är ett säkerhetskopieringsskript, en servermontering eller ett CI-jobb, måste arbeta med kontots fullständiga inloggningsuppgifter.

Som jämförelse: Hos S3-kompatibla lagringstjänster är åtkomstnyckelpar standard, kan återkallas och begränsas till buckets eller prefix. Google och Microsoft har applösenord och tjänstkonton. Hos Proton gäller däremot allt eller inget: Den som vill ge en server åtkomst till en mapp ger den åtkomst till hela kontot.

Rättvist nog är detta svårare i en tjänst med end-to-end-kryptering än i S3, eftersom en begränsad åtkomst också skulle kräva begränsat nyckelmaterial. SDK-sessionerna visar dock att Proton behärskar sådana konstruktioner. En session är redan en härledd åtkomst som kan återkallas. En officiell ”maskintoken för just denna mapp, skrivskyddad” vore det enskilt största framsteget för serveranvändning, långt före vilken klient som helst.

## Rekommendation per användningsfall

| Användningsfall | Läget i juli 2026 |
|---|---|
| Skrivbordssynkronisering under Linux | Vänta på den annonserade klienten; använd tills dess Rclone-synkronisering eller webbgränssnittet |
| Serversäkerhetskopiering (ladda upp filer) | Rclone med `copy` eller `sync`; fungerar, men räkna med betastatus |
| Filsystemmontering för tjänster | Rclone med `mount`, lagrad TOTP-nyckel och dedikerat konto; den enda [beprövade vägen](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering med ren inloggning | Håll SDK-CLI:t under uppsikt; ännu för tidigt för produktion |

På Linux-skrivbordet kan man vänta på den annonserade klienten eller tills vidare använda Rclone. På servrar förblir Rclone den enda praktiska monteringslösningen. En fungerande nödlösning blir dock först en robust plattform när Proton erbjuder begränsade maskinåtkomster och en officiellt stödd montering.

## Källor

1.  [OMG Ubuntu: Proton Drive-klienten kommer (äntligen) till Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekräftelsen från juni 2026 att Linux-klienten är under utveckling, utan datum.

2.  [Proton: Produktfärdplaner för våren och sommaren 2026](https://proton.me/blog/2026-spring-summer-roadmaps): färdplanen med Linux-klienten utan tidsram och SDK:t som grund för de egna apparna.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentliga SDK-kodförrådet inklusive CLI med webbläsarinloggning och session i nyckelringen.

4.  [Proton Drive SDK-förhandsversion](https://proton.me/blog/proton-drive-sdk-preview): Protons egen bedömning: ännu inte produktionsmoget för tredjepartsprogram.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backendprogrammet inklusive beta-anmärkning och alternativet `otp_secret_key` för oövervakad inloggning.
