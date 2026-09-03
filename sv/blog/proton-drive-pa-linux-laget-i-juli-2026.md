---
title: "Proton Drive på Linux: läget i juli 2026"
navTitle: "Proton Drive och Linux"
description: "Den officiella Linux-klienten har annonserats men är ännu inte tillgänglig. På servrar kan Proton Drive för närvarande monteras med Rclone; det nya SDK:t visar den tekniska riktningen. Det som fortfarande saknas är maskinåtkomst begränsad till enskilda mappar eller uppgifter."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min. lästid"
themen:
  - proton-drive
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-pa-linux-laget-i-juli-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:27:27.377Z
translationReview: automatic
translationSourceHash: dc35729208664efc8971642a0b5e67d38634e3871b66e1a448a41d14cd2d67b3
url: https://rafaelpfister.ch/sv/blog/proton-drive-pa-linux-laget-i-juli-2026
---

För Windows och macOS erbjuder Proton Drive egna synkroniseringsklienter sedan 2023. Under Linux finns hittills endast webbgränssnittet, communityverktyg och ett officiellt SDK i förhandsvisningsstadiet. På en server är situationen ännu svårare, eftersom varken skrivbordssynkronisering eller interaktiv inloggning passar särskilt bra där.

Den här översikten beskriver läget i juli 2026. Utöver de publicerade färdplanerna bygger den på ett praktiskt test av Rclone-backenden [som dokumentlagring för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Linux-klienten har annonserats, men saknar fortfarande datum

I juni 2026 bekräftade Proton för första gången uttryckligen att en Linux-klient utvecklas. Den bygger på det nya, enhetliga SDK:t och ska använda samma tekniska grund som programmen för Windows och macOS. Det finns ännu inget datum eller någon offentlig beta.

Viktigt för bedömningen: Det här blir en **synkroniseringsklient för skrivbordet**. För skrivbordet löser den problemet. För serverprogram är en synkroniseringsklient däremot fel verktyg, eftersom en tjänst ska läsa filer direkt från Proton Drive och skriva dit dem. En synkroniseringsklient håller en fullständig lokal kopia, precis det man vill undvika vid begränsat lagringsutrymme.

## Rclone utför det praktiska arbetet i dag

Under Linux är Rclone med sin `protondrive`-backend för närvarande det mest mångsidiga verktyget. Det kan kopiera och synkronisera filer och är den enda tillgängliga lösningen som kan tillhandahålla Proton Drive som ett lokalt katalogträd via **FUSE-monteringspunkt**. Två begränsningar är viktiga:

**Det är beta på ett rekonstruerat API.** Proton dokumenterar inte sitt Drive-API offentligt; backenden bygger på reverse engineering. I testet fungerade den tillförlitligt, men begränsade snabba anropsföljder med inkonsekventa kataloglistor.

**För oövervakad drift frågar Rclone efter TOTP-nyckeln.** Konfigurationsassistenten benämner fältet `otp_secret_key`. Det avser den permanenta nyckeln från 2FA-konfigurationen, inte den sexsiffriga kod som en autentiseringsapp just visar. Rclone lagrar detta värde fördunklat och genererar själv en giltig TOTP-kod från den vid varje inloggning.

Den som av misstag anger en aktuell engångskod kan slutföra den första inloggningen. Nästa återautentisering misslyckas dock med fel 8002, eftersom Rclone inte kan använda samma kod igen.

Kontot förblir därmed skyddat mot ett isolerat stulet lösenord. En komprometterad server avslöjar däremot både lösenord och TOTP-nyckel. För automatiserade åtkomster rekommenderas därför ett **dedikerat Proton-konto**.

Hur en sådan montering beter sig i Docker-miljöer, inklusive två odokumenterade problem, beskrivs i [den separata artikeln om Rclone i containrar](/blog/rclone-mount-in-docker-container).

## Det officiella SDK:t visar vart utvecklingen är på väg

Parallellt flyttar Proton sina program till ett **officiellt SDK** för JavaScript och C# med bindningar för Swift och Kotlin. Det offentliga kodarkivet innehåller även ett kommandoradsverktyg. Dess inloggningsmodell är renare än Rclone-backendens:

- `auth login` öppnar webbläsaren; inloggningen sker normalt **inklusive tvåfaktorsautentisering**
- sessionen hamnar i **operativsystemets nyckellager** (Keychain, Credential Manager, libsecret), och SDK:t förnyar den självt
- därefter: lista filer med maskinläsbar JSON-utdata, ladda upp och kontrollera delningar

Lösenord och TOTP-nyckel behöver alltså inte finnas i en konfigurationsfil. För serverdrift återstår ändå tre gränser: CLI:t kan **inte montera ett filsystem**, inloggningen öppnar en webbläsare och Proton klassar ännu inte SDK:t som produktionsredo för tredjepartsprogram. Lanseringen är planerad till slutet av 2026 till början av 2027.

## Den egentliga luckan: maskinåtkomst

Problemets kärna ligger en nivå djupare än klient eller SDK: **Proton har ingen maskinåtkomst.** Inget applösenord, inget tjänstkonto, ingen token med begränsad omfattning. Varje automatisering, vare sig det är ett säkerhetskopieringsskript, en servermontering eller ett CI-jobb, måste arbeta med kontots fullständiga inloggningsuppgifter.

Som jämförelse är åtkomstnyckelpar standard för S3-kompatibel lagring, och de kan återkallas samt begränsas till buckets eller prefix. Google och Microsoft har applösenord och tjänstkonton. Hos Proton gäller däremot allt eller inget: Den som vill ge en server åtkomst till en mapp ger den åtkomst till hela kontot.

För en end-to-end-krypterad tjänst är detta svårare än för S3, eftersom begränsad åtkomst också skulle behöva innebära begränsat nyckelmaterial. SDK-sessionerna visar dock att Proton behärskar sådana konstruktioner. En session är redan en härledd åtkomst som kan återkallas. En officiell ”maskintoken för just den här mappen, skrivskyddad” vore det största enskilda framsteget för serveranvändning, långt före någon klient.

## Rekommendation efter användningsfall

| Användningsfall | Läget i juli 2026 |
|---|---|
| Skrivbordssynkronisering under Linux | Vänta på den annonserade klienten; fram till dess Rclone-synkronisering eller webbgränssnitt |
| Serversäkerhetskopiering (ladda upp filer) | Rclone med `copy` eller `sync`; fungerar, räkna med betastatus |
| Filsystemmontering för tjänster | Rclone med `mount`, lagrad TOTP-nyckel och dedikerat konto; den enda [beprövade vägen i praktiken](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering med ren inloggning | Håll koll på SDK-CLI:t; ännu för tidigt för produktion |

På Linux-skrivbordet kan man vänta på den annonserade klienten eller tills vidare använda Rclone. På servrar förblir Rclone den enda praktiska monteringslösningen. Men en fungerande nödlösning blir först en robust plattform när Proton erbjuder begränsad maskinåtkomst och en officiellt stödd montering.

## Källor

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekräftelsen från juni 2026 att Linux-klienten är under utveckling, utan datum.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): färdplanen med Linux-klienten utan tidsram och SDK:t som grund för de egna programmen.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentliga SDK-kodarkivet inklusive CLI med webbläsarinloggning och session i nyckellagret.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons egen bedömning: ännu inte produktionsredo för tredjepartsprogram.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backenden inklusive betaanmärkningen och alternativet `otp_secret_key` för oövervakad inloggning.
