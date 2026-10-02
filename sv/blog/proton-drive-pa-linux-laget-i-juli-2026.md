---
title: "Proton Drive på Linux: läget i oktober 2026"
navTitle: "Proton Drive & Linux"
description: "Den officiella Linux-klienten är annonserad men ännu inte tillgänglig. Sedan juni 2026 finns den officiella Proton Drive CLI för skript och servrar; Proton Drive kan fortfarande bara monteras med Rclone. Det som saknas är maskinåtkomst begränsad till enskilda mappar eller uppgifter."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min lästid"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-pa-linux-laget-i-juli-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:02:56.467Z
translationReview: automatic
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/sv/blog/proton-drive-pa-linux-laget-i-juli-2026
---

För Windows och macOS har Proton Drive erbjudit egna synkroniseringsklienter sedan 2023. På Linux finns hittills webbgränssnittet, communityverktyg och sedan juni 2026 en officiell kommandoradsapplikation, men ännu ingen synkroniseringsklient. På en server är situationen ännu svårare, eftersom varken skrivbordssynkronisering eller interaktiv inloggning passar särskilt bra där.

Den här översikten beskriver läget den 1 oktober 2026. Den bygger på de publicerade färdplanerna, källkoden för Proton Drive CLI och ett praktiskt test av Rclone-backendens [användning som dokumentarkiv för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Uppdatering från 1 oktober 2026:** Den första versionen från 26 juli beskrev kommandoradsapplikationen bara som ett verktyg i SDK-repositoriet. Proton hade dock redan den 9 juni 2026 officiellt publicerat den som **Proton Drive CLI**, med färdiga byggen för Windows, macOS och Linux. Avsnittet om den och rekommendationstabellen har uppdaterats i enlighet med detta; detaljerna finns i den separata artikeln om [Proton Drive CLI](/blog/proton-drive-cli).

## Linux-klienten är annonserad, men har ännu inget datum

I juni 2026 bekräftade Proton för första gången uttryckligen att en Linux-klient utvecklas. Den bygger på det nya, enhetliga SDK:t och ska använda samma tekniska grund som applikationerna för Windows och macOS. I början av oktober 2026 finns fortfarande varken något datum eller någon offentlig beta.

Viktigt för sammanhanget: Det blir en **synkroniseringsklient för skrivbordet**. För skrivbordet löser den problemet. För serverapplikationer är en synkroniseringsklient däremot fel verktyg, eftersom en tjänst ska läsa filer direkt från Proton Drive och skriva dit. En synkroniseringsklient håller en fullständig lokal kopia, precis det man vill undvika när lagringsutrymmet är begränsat.

## Rclone behövs fortfarande för monteringar och speglingar

På Linux är Rclone med sin `protondrive`-backend för närvarande det mest mångsidiga verktyget. Det kan kopiera och synkronisera filer och är den enda tillgängliga lösningen som kan tillhandahålla Proton Drive som en **FUSE-montering** likt en lokal katalog. Två begränsningar är viktiga:

**Det är beta och bygger på ett återskapat API.** Proton dokumenterar inte sitt Drive-API offentligt; backend-programmet bygger på reverse engineering. I testet fungerade det tillförlitligt, men begränsade snabba anropsföljder med inkonsekventa kataloglistningar.

**För oövervakad drift frågar Rclone efter TOTP-nyckeln.** Konfigurationsguiden benämner fältet `otp_secret_key`. Det avser den permanenta nyckeln från 2FA-konfigurationen, inte den sexsiffriga kod som en autentiseringsapp just nu visar. Rclone lagrar detta värde maskerat och skapar själv en giltig TOTP-kod vid varje inloggning.

Den som av misstag anger en aktuell engångskod kan slutföra den första inloggningen. Nästa förnyade autentisering misslyckas dock med fel 8002, eftersom Rclone inte kan använda samma kod en gång till.

Därmed förblir kontot skyddat mot ett isolerat stulet lösenord. En komprometterad server avslöjar dock både lösenord och TOTP-nyckel. För automatiserad åtkomst rekommenderas därför ett **dedikerat Proton-konto**.

Hur en sådan montering beter sig i Docker-miljöer, inklusive två odokumenterade problem, beskrivs i [den separata artikeln om Rclone i containrar](/blog/rclone-mount-in-docker-container).

## Den officiella CLI:n täcker skript och säkerhetskopiering

Den 9 juni 2026 publicerade Proton **Proton Drive CLI**, en enda körbar fil `proton-drive` för Windows, macOS och Linux. Den bygger på samma SDK som de officiella apparna; källkoden finns i det offentliga SDK-repositoriet. Den aktuella versionen är 0.8.0 från 13 augusti 2026, med byggen för x86-64 (även utan AVX2), ARM64 och musl-distributioner som Alpine.

Inloggningsmodellen är renare än för Rclone-backenden:

- `auth login` ger en inloggnings-URL som också kan öppnas **på en annan enhet**; inloggningen sker normalt **inklusive tvåfaktorsautentisering**, alltså även via SSH på en server utan skrivbord
- sessionen hamnar i **operativsystemets nyckelring** (Keychain, Credential Manager, libsecret) eller, sedan version 0.6.0, i lösenordshanteraren `pass`, vilket är mer praktiskt på servrar utan skrivbordssession
- därefter: ladda upp och ner filer, flytta dem, lägga dem i papperskorgen, hantera delningar, inbjudningar och offentliga länkar, använda Proton Photos; alltid med `--json` för maskinläsbar utdata

Lösenord och TOTP-nyckel behöver alltså inte finnas på servern. För säkerhetskopior och byggartefakter är CLI:n därför i dag ett bättre val än Rclone. Två begränsningar kvarstår: CLI:n kan **inte montera ett filsystem** och **inte skapa en spegel med raderingar**; den laddar upp och ner, men synkroniserar inte. Ett kommando `takeout` för en fullständig lokal export finns redan i repositoriet, men har ännu inte publicerats.

Proton bedömer fortfarande själva SDK:t som inte produktionsklart för tredjepartsapplikationer; lanseringen är planerad till slutet av 2026 eller början av 2027. CLI:n påverkas inte av detta, eftersom Proton själva ger ut den.

## Den egentliga luckan: maskinåtkomst

Kärnan i problemet ligger ett lager djupare än klient eller SDK: **Proton har ingen maskinåtkomst.** Inget applösenord, inget tjänstekonto, ingen token med begränsad omfattning. Varje automatisering, oavsett om det är ett säkerhetskopieringsskript, en servermontering eller ett CI-jobb, måste arbeta med kontots fullständiga inloggningsuppgifter.

Som jämförelse är par av åtkomstnycklar standard för S3-kompatibel lagring, de kan återkallas och begränsas till buckets eller prefix. Google och Microsoft har applösenord och tjänstekonton. Hos Proton gäller däremot allt eller inget: den som vill ge en server åtkomst till en mapp ger den åtkomst till hela kontot.

För en heltäckande krypterad tjänst är det svårare än med S3, eftersom begränsad åtkomst också skulle behöva innebära begränsat nyckelmaterial. CLI:ns sessioner visar dock att Proton behärskar sådana konstruktioner. En session är redan en härledd, återkallelig åtkomst, bara med kontots fulla omfattning. En officiell ”maskintoken för just denna mapp, endast läsning” skulle vara det enskilt största framsteget för serveranvändning, långt före varje klient.

## Rekommendation efter användningsfall

| Användningsfall | Läget i oktober 2026 |
|---|---|
| Skrivbordssynkronisering på Linux | Vänta på den annonserade klienten; använd tills dess Rclone-synkronisering eller webbgränssnittet |
| Säkerhetskopiering från server (ladda upp filer) | [Proton Drive CLI](/blog/proton-drive-cli) med `filesystem upload` och konfliktstrategin `create-new-revision`; officiellt stödd, utan lagrat lösenord |
| Spegel med raderingar | Rclone med `sync`; räkna med betastatus |
| Filsystemmontering för tjänster | Rclone med `mount`, lagrad TOTP-nyckel och dedikerat konto; den enda [praktiskt beprövade vägen](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering, hantera delningar | Proton Drive CLI med `--json`; versionsnivå 0.x, kommandon kan fortfarande ändras |

På Linux-skrivbordet kan man vänta på den annonserade klienten eller tills vidare använda Rclone. På servrar hanterar den officiella CLI:n numera säkerhetskopiering och automatisering; för en montering förblir Rclone den enda praktiska lösningen. En fungerande nödlösning blir dock först en robust plattform när Proton erbjuder begränsad maskinåtkomst och en officiellt stödd montering.

## Källor

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekräftelsen från juni 2026 att Linux-klienten utvecklas, utan något datum.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): färdplanen med Linux-klienten utan tidsram och SDK:t som grund för de egna apparna.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentliga SDK-repositoriet med CLI:ns källkod och ändringslogg.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): den officiella publiceringen av CLI:n den 9 juni 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuell version 0.8.0 från 13 augusti 2026 med alla plattformsbyggen.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons egen bedömning: ännu inte produktionsklart för tredjepartsapplikationer.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backend-programmet med betaanmärkningen och alternativet `otp_secret_key` för oövervakad inloggning.
