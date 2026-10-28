---
title: "Proton Drive på Linux: läget i oktober 2026"
navTitle: "Proton Drive & Linux"
description: "Den officiella Linux-klienten är aviserad men ännu inte tillgänglig. För skript och servrar finns den officiella Proton Drive CLI sedan juni 2026; Proton Drive kan fortfarande bara monteras med Rclone. Det som saknas är maskinåtkomst begränsad till enskilda mappar eller uppgifter."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min. läsning"
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
translatedAt: 2026-10-27T09:19:17.726Z
translationReview: automatic
translationSourceHash: e32f1f479477b6a2eb1ff5ad2fb4e4144953faad34f2ac9302f3b2338ca22047
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/proton-drive-pa-linux-laget-i-juli-2026
---

För Windows och macOS erbjuder Proton Drive egna synkroniseringsklienter sedan 2023. På Linux finns hittills webbgränssnittet, verktyg från communityn och sedan juni 2026 en officiell kommandoradsapplikation, men ännu ingen synkroniseringsklient. På en server är situationen ännu svårare, eftersom varken skrivbordssynkronisering eller interaktiv inloggning passar särskilt bra där.

Denna översikt beskriver läget den 1 oktober 2026. Den bygger på de publicerade färdplanerna, källkoden för Proton Drive CLI och ett praktiskt test av Rclone-backenden [som dokumentlagring för Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Uppdatering från den 1 oktober 2026:** Den första versionen från den 26 juli beskrev kommandoradsapplikationen endast som ett verktyg i SDK-repositoriet. Proton hade dock redan den 9 juni 2026 officiellt publicerat den som **Proton Drive CLI**, med färdiga byggen för Windows, macOS och Linux. Avsnittet om den och rekommendationstabellen har uppdaterats i enlighet med detta; detaljerna finns i den separata artikeln om [Proton Drive CLI](/blog/proton-drive-cli).

## Linux-klienten är aviserad, men fortfarande utan datum

I juni 2026 bekräftade Proton för första gången uttryckligen att en Linux-klient utvecklas. Den byggs på det nya, enhetliga SDK:t och ska använda samma tekniska grund som applikationerna för Windows och macOS. I början av oktober 2026 finns fortfarande varken något datum eller någon offentlig beta.

Viktigt för bedömningen: detta blir en **synkroniseringsklient för skrivbordet**. För skrivbordet löser den problemet. För serverapplikationer är en synkroniseringsklient däremot fel verktyg, eftersom en tjänst ska läsa filer direkt från Proton Drive och skriva dem dit. En synkroniseringsklient håller en fullständig lokal kopia, alltså precis det man vill undvika vid begränsat lagringsutrymme.

## Rclone behövs fortfarande för monteringar och speglingar

På Linux är Rclone med sin `protondrive`-backend för närvarande det mest mångsidiga verktyget. Det kan kopiera och synkronisera filer och är den enda tillgängliga lösningen som kan tillhandahålla Proton Drive som en lokal katalog via en **FUSE-montering**. Två begränsningar är viktiga:

**Det är beta och bygger på ett återskapat API.** Proton dokumenterar inte sitt Drive-API offentligt; backenden bygger på reverse engineering. I testet fungerade den tillförlitligt, men vid snabba anropsföljder stryptes den med inkonsekventa kataloglistor.

**För oövervakad drift frågar Rclone efter TOTP-nyckeln.** Konfigurationsguiden kallar fältet `otp_secret_key`. Det som avses är den permanenta nyckeln från 2FA-konfigurationen, inte den sexsiffriga kod som en autentiseringsapp just nu visar. Rclone lagrar detta värde maskerat och skapar själv en giltig TOTP-kod vid varje inloggning.

Den som av misstag anger en aktuell engångskod kan slutföra den första inloggningen. Nästa återautentisering misslyckas dock med fel 8002, eftersom Rclone inte kan använda samma kod en gång till.

Kontot förblir därmed skyddat mot ett isolerat stulet lösenord. En komprometterad server avslöjar dock både lösenord och TOTP-nyckel. För automatiserade åtkomster rekommenderas därför ett **dedikerat Proton-konto**.

Hur en sådan montering beter sig i Docker-miljöer, inklusive två odokumenterade problem, beskrivs i den [separata artikeln om Rclone i containrar](/blog/rclone-mount-in-docker-container).

## Den officiella CLI:n täcker skript och säkerhetskopior

Den 9 juni 2026 publicerade Proton **Proton Drive CLI**, en enda körbar fil `proton-drive` för Windows, macOS och Linux. Den bygger på samma SDK som de officiella apparna; källkoden finns i det offentliga SDK-repositoriet. Den aktuella versionen är 0.9.0 från den 5 oktober 2026, med byggen för x86-64 (även utan AVX2), ARM64 och musl-distributioner som Alpine.

Inloggningsmodellen är bättre utformad än den i Rclone-backenden:

- `auth login` ger en inloggnings-URL som även kan öppnas **på en annan enhet**; inloggningen sker normalt **inklusive tvåfaktorsautentisering**, alltså även via SSH på en server utan skrivbord
- sessionen hamnar i **operativsystemets nyckelring** (Keychain, Credential Manager, libsecret) eller, sedan version 0.6.0, i lösenordshanteraren `pass`, som är mer praktisk på servrar utan skrivbordssession
- därefter: ladda upp och ner filer, flytta dem, lägga dem i papperskorgen, hantera delningar, inbjudningar och offentliga länkar, använda Proton Photos; med `--json` för maskinläsbar utdata

Lösenord och TOTP-nyckel behöver därmed inte finnas på servern. För säkerhetskopior och byggartefakter är CLI:n därför i dag ett bättre val än Rclone. Två begränsningar kvarstår: CLI:n kan **inte montera ett filsystem** och **inte skapa en spegel med raderingar**; den laddar upp och ner, men synkroniserar inte. Sedan version 0.9.0 från den 5 oktober 2026 har dessutom kommandot `takeout` för en fullständig lokal export publicerats.

Proton bedömer fortfarande själva SDK:t som inte produktionsklart för tredjepartsapplikationer; godkännandet är planerat till slutet av 2026 eller början av 2027. CLI:n påverkas inte av detta, eftersom Proton själva ger ut den.

## Den egentliga luckan: maskinåtkomst

Kärnan i problemet ligger en nivå djupare än klient eller SDK: **Proton känner inte till maskinåtkomst.** Inget applösenord, inget tjänstekonto, ingen token med begränsad omfattning. Varje automatisering, oavsett om det är ett säkerhetskopieringsskript, en servermontering eller ett CI-jobb, måste arbeta med kontots fullständiga inloggningsuppgifter.

Som jämförelse är par med åtkomstnycklar standard för S3-kompatibel lagring, återkalleliga och möjliga att begränsa till buckets eller prefix. Google och Microsoft har applösenord och tjänstekonton. Hos Proton gäller däremot allt eller inget: den som vill ge en server åtkomst till en mapp ger den åtkomst till hela kontot.

För en tjänst med end-to-end-kryptering är detta svårare än för S3, eftersom begränsad åtkomst även skulle behöva innebära begränsat nyckelmaterial. CLI:ns sessioner visar dock att Proton behärskar sådana konstruktioner. En session är redan en härledd, återkallelig åtkomst, bara med kontots fulla omfattning. En officiell ”maskintoken för just denna mapp, endast läsning” vore det största enskilda framsteget för serveranvändning, långt före varje klient.

## Rekommendation efter användningsfall

| Användningsfall | Läget i oktober 2026 |
|---|---|
| Skrivbordssynkronisering på Linux | Vänta på den aviserade klienten; tills dess Rclone-synkronisering eller webbgränssnittet |
| Serversäkerhetskopiering (ladda upp filer) | [Proton Drive CLI](/blog/proton-drive-cli) med `filesystem upload` och konfliktstrategin `create-new-revision`; officiellt stödd, utan lagrat lösenord |
| Spegel med raderingar | Rclone med `sync`; ta hänsyn till betastatus |
| Filsystemsmontering för tjänster | Rclone med `mount`, lagrad TOTP-nyckel och dedikerat konto; den enda [praktiskt beprövade vägen](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering, hantera delningar | Proton Drive CLI med `--json`; versionsstatus 0.x, kommandon kan fortfarande ändras |

På Linux-skrivbordet kan man vänta på den aviserade klienten eller tills vidare använda Rclone. På servrar hanterar den officiella CLI:n numera säkerhetskopior och automatisering; för en montering är Rclone fortfarande den enda praktiska lösningen. En fungerande nödlösning blir dock först en robust plattform när Proton erbjuder begränsad maskinåtkomst och en officiellt stödd montering.

## Ändringshistorik

- **25 oktober 2026:** Uppdaterat till Proton Drive CLI 0.9.0 från den 5 oktober 2026, med den nu publicerade takeout-funktionen för en fullständig lokal export.

## Källor

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekräftelsen från juni 2026 att Linux-klienten är under utveckling, utan datum.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): färdplanen med Linux-klienten utan tidsram och SDK:t som grund för de egna apparna.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentliga SDK-repositoriet med källkod och ändringslogg för CLI:n (bland annat med kommandot `takeout` i version 0.9.0).

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): den officiella publiceringen av CLI:n den 9 juni 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): aktuell version 0.9.0 från den 5 oktober 2026 med alla plattformsbyggen.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons egen bedömning: ännu inte produktionsklart för tredjepartsapplikationer.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backenden med beta-anmärkningen och alternativet `otp_secret_key` för oövervakad inloggning.
