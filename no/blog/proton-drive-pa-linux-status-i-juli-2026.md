---
title: "Proton Drive på Linux: Status i juli 2026"
navTitle: "Proton Drive og Linux"
description: "Den offisielle Linux-klienten er annonsert, men ennå ikke tilgjengelig. På servere kan Proton Drive for tiden monteres med Rclone; det nye SDK-et viser den tekniske retningen. Det som fortsatt mangler, er maskintilgang begrenset til enkelte mapper eller oppgaver."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min lesetid"
themen:
  - proton-drive
  - rclone
related:
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-pa-linux-status-i-juli-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:27:52.628Z
translationReview: automatic
translationSourceHash: dc35729208664efc8971642a0b5e67d38634e3871b66e1a448a41d14cd2d67b3
url: https://rafaelpfister.ch/no/blog/proton-drive-pa-linux-status-i-juli-2026
---

For Windows og macOS har Proton Drive tilbudt egne synkroniseringsklienter siden 2023. Under Linux finnes det foreløpig bare webgrensesnittet, fellesskapsverktøy og et offisielt SDK i forhåndsvisning. På en server er situasjonen enda vanskeligere, fordi verken skrivebordssynkronisering eller interaktiv innlogging passer godt der.

Denne oversikten beskriver status i juli 2026. Grunnlaget er, i tillegg til de publiserte veikartene, en praktisk test av Rclone-backenden [som dokumentlagring for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Linux-klienten er annonsert, men fortsatt uten dato

I juni 2026 bekreftet Proton for første gang uttrykkelig at en Linux-klient er under utvikling. Den bygges på det nye, enhetlige SDK-et og skal bruke samme tekniske grunnlag som programmene for Windows og macOS. Det finnes foreløpig ingen dato eller offentlig beta.

Viktig for vurderingen: Dette blir en **synkroniseringsklient for skrivebordet**. For skrivebordet løser den problemet. For serverapplikasjoner er en synkroniseringsklient derimot feil verktøy, fordi en tjeneste skal lese filer direkte fra Proton Drive og skrive til det. En synkroniseringsklient holder en fullstendig lokal kopi, nettopp det man vil unngå ved begrenset lagringsplass.

## I dag gjør Rclone det praktiske arbeidet

Under Linux er Rclone med sin `protondrive`-backend for tiden det mest allsidige verktøyet. Det kan kopiere og synkronisere filer og er den eneste tilgjengelige løsningen som kan gjøre Proton Drive tilgjengelig som en lokal katalog via **FUSE-montering**. To begrensninger er viktige:

**Det er beta basert på et rekonstruert API.** Proton dokumenterer ikke Drive-API-et offentlig; backenden er basert på reverse engineering. I testen fungerte den pålitelig, men ved raske kallsekvenser strupet den med inkonsekvente kataloglister.

**For uovervåket drift spør Rclone etter TOTP-nøkkelen.** Konfigurasjonsveiviseren omtaler feltet som `otp_secret_key`. Det menes den permanente nøkkelen fra 2FA-oppsettet, ikke den sekssifrede koden som en autentiseringsapp viser akkurat nå. Rclone lagrer denne verdien tilslørt og genererer selv en gyldig TOTP-kode fra den ved hver innlogging.

Den som ved en feil skriver inn en aktuell engangskode, kan fullføre den første innloggingen. Neste reautentisering mislykkes imidlertid med feil 8002, fordi Rclone ikke kan bruke samme kode én gang til.

Dermed forblir kontoen beskyttet mot et isolert stjålet passord. En kompromittert server avslører imidlertid både passord og TOTP-nøkkel. For automatisert tilgang anbefales derfor en **dedikert Proton-konto**.

Hvordan en slik montering oppfører seg i Docker-miljøer, inkludert to udokumenterte problemer, beskrives i [den separate artikkelen om Rclone i containere](/blog/rclone-mount-in-docker-container).

## Det offisielle SDK-et viser hvor utviklingen går

Parallelt flytter Proton programmene sine over til et **offisielt SDK** for JavaScript og C# med bindinger for Swift og Kotlin. Det offentlige repositoriet inneholder også et kommandolinjeverktøy. Innloggingsmodellen er renere enn den i Rclone-backenden:

- `auth login` åpner nettleseren; innloggingen skjer på vanlig måte **inkludert tofaktorautentisering**
- økten havner i **operativsystemets nøkkellager** (Keychain, Credential Manager, libsecret), og SDK-et fornyer den selv
- deretter: list opp filer, last opp og kontroller delinger med maskinlesbar JSON-utdata

Passord og TOTP-nøkkel trenger dermed ikke å stå i en konfigurasjonsfil. For serverdrift gjenstår likevel tre grenser: CLI-en kan **ikke montere et filsystem**, innloggingen åpner en nettleser, og Proton vurderer fortsatt ikke SDK-et som produksjonsklart for tredjepartsapplikasjoner. Lanseringen er planlagt for slutten av 2026 til begynnelsen av 2027.

## Det egentlige gapet: maskintilganger

Kjernen i problemet ligger ett nivå dypere enn klient eller SDK: **Proton har ingen maskintilganger.** Ingen app-passord, ingen tjenestekonto, ingen token med begrenset omfang. All automatisering, enten det er et sikkerhetskopieringsskript, en servermontering eller en CI-jobb, må arbeide med kontoens fullverdige innloggingsopplysninger.

Til sammenligning er tilgangsnøkkelpar normen i S3-kompatibel lagring; de kan tilbakekalles og begrenses til bucketer eller prefikser. Google og Microsoft har app-passord og tjenestekontoer. Hos Proton gjelder derimot alt eller ingenting: Den som vil gi en server tilgang til én mappe, gir den tilgang til hele kontoen.

For en ende-til-ende-kryptert tjeneste er dette vanskeligere enn for S3, fordi begrenset tilgang også måtte bety begrenset nøkkelmateriale. SDK-øktene viser imidlertid at Proton behersker slike konstruksjoner. En økt er allerede en avledet tilgang som kan tilbakekalles. En offisiell «maskintoken for akkurat denne mappen, kun lesing» ville være det største enkeltfremskrittet for serverbruk, lenge før enhver klient.

## Anbefaling etter bruksområde

| Bruksområde | Status juli 2026 |
|---|---|
| Skrivebordssynkronisering under Linux | Vent på den annonserte klienten; inntil da Rclone-synkronisering eller webgrensesnittet |
| Serversikkerhetskopi (laste opp filer) | Rclone med `copy` eller `sync`; fungerer, ta høyde for beta-status |
| Filsystemmontering for tjenester | Rclone med `mount`, lagret TOTP-nøkkel og dedikert konto; den eneste [utprøvde løsningen](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering med ryddig innlogging | Følg med på SDK-CLI-en; fortsatt for tidlig for produksjon |

På Linux-skrivebordet kan man vente på den annonserte klienten eller foreløpig bruke Rclone. På servere er Rclone fortsatt den eneste praktiske monteringsløsningen. Et fungerende hjelpemiddel blir imidlertid først en robust plattform når Proton tilbyr begrensede maskintilganger og en offisielt støttet montering.

## Kilder

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekreftelsen fra juni 2026 på at Linux-klienten er under utvikling, uten dato.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): veikartet med Linux-klienten uten tidsvindu og SDK-et som grunnlag for egne apper.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentlige SDK-repositoriet med CLI, nettleserinnlogging og økt i nøkkellageret.

4.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons egen vurdering: fortsatt ikke produksjonsklart for tredjepartsapplikasjoner.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backenden med beta-merknad og valget `otp_secret_key` for uovervåket innlogging.
