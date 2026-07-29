---
title: "Proton Drive på Linux: Status i juli 2026"
navTitle: "Proton Drive og Linux"
description: "Den offisielle Linux-klienten er annonsert, men ennå ikke tilgjengelig. På servere kan Proton Drive for tiden monteres med Rclone; det nye SDK-et viser den tekniske retningen. Det som fortsatt mangler, er maskintilgang begrenset til enkelte mapper eller oppgaver."
date: "2026-07-26"
kategorie: "Proton Drive"
timeToRead: "8 min lesetid"
themen:
  - "proton-drive"
  - "rclone"
related:
  - "paperless-dokumente-clouddienst-auslagern"
  - "rclone-mount-in-docker-container"
slug: "proton-drive-pa-linux-status-i-juli-2026"
translationOf: "proton-drive-linux-status"
url: "https://rafaelpfister.ch/no/blog/proton-drive-pa-linux-status-i-juli-2026"
---

For Windows og macOS har Proton Drive tilbudt egne synkroniseringsklienter siden 2023. På Linux finnes det foreløpig bare nettgrensesnittet, fellesskapsverktøy og et offisielt SDK i forhåndsvisningsstadiet. På en server er situasjonen enda vanskeligere, fordi verken skrivebordssynkronisering eller interaktiv innlogging passer godt der.

Denne oversikten beskriver statusen i juli 2026. I tillegg til de publiserte veikartene bygger den på en praktisk test av Rclone-backenden [som dokumentlager for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

## Linux-klienten er annonsert, men fortsatt uten dato

I juni 2026 bekreftet Proton for første gang uttrykkelig at en Linux-klient utvikles. Den bygges på det nye, enhetlige SDK-et og skal bruke samme tekniske grunnlag som programmene for Windows og macOS. Det finnes fortsatt ingen dato eller offentlig beta.

Viktig for vurderingen: Dette blir en **synkroniseringsklient for skrivebordet**. For skrivebordet løser den problemet. For serverprogrammer er en synkroniseringsklient derimot feil verktøy, for en tjeneste skal lese filer direkte fra Proton Drive og skrive dem dit. En synkroniseringsklient holder en full lokal kopi, nettopp det man vil unngå når lagringsplassen er begrenset.

## I dag gjør Rclone det praktiske arbeidet

På Linux er Rclone med sin `protondrive`-backend for tiden det mest allsidige verktøyet. Det kan kopiere og synkronisere filer og er den eneste tilgjengelige løsningen som kan gjøre Proton Drive tilgjengelig som en **FUSE-montering** på samme måte som en lokal katalog. To begrensninger er viktige:

**Det er beta på et rekonstruert API.** Proton dokumenterer ikke Drive-API-et offentlig; backenden bygger på omvendt utvikling. I testen fungerte den pålitelig, men begrenset ved raske sekvenser av kall med inkonsekvente kataloglister.

**For uovervåket drift spør Rclone etter TOTP-nøkkelen.** Konfigurasjonsveiviseren kaller feltet `otp_secret_key`. Det menes den permanente nøkkelen fra 2FA-oppsettet, ikke den sekssifrede koden som en autentiseringsapp akkurat viser. Rclone lagrer denne verdien tilslørt og genererer selv en gyldig TOTP-kode ved hver innlogging.

Den som ved et uhell legger inn en aktuell engangskode, kan fullføre den første innloggingen. Neste reautentisering mislykkes imidlertid med feil 8002, fordi Rclone ikke kan bruke samme kode én gang til.

Dermed forblir kontoen beskyttet mot et isolert stjålet passord. En kompromittert server avslører imidlertid både passord og TOTP-nøkkel. For automatisert tilgang anbefales derfor en **dedikert Proton-konto**.

Hvordan en slik montering oppfører seg i Docker-miljøer, inkludert to udokumenterte fallgruver, står i [den egne artikkelen om Rclone i containere](/blog/rclone-mount-in-docker-container).

## Det offisielle SDK-et viser hvor utviklingen går

Parallelt legger Proton programmene sine om til et **offisielt SDK** for JavaScript og C# med bindinger for Swift og Kotlin. Det offentlige repositoriet inneholder også et kommandolinjeverktøy. Innloggingsmodellen er ryddigere enn den i Rclone-backenden:

- `auth login` åpner nettleseren; innloggingen foregår på vanlig måte **inkludert tofaktorautentisering**
- økten havner i **operativsystemets nøkkellager** (Keychain, Credential Manager, libsecret), og SDK-et fornyer den selv
- deretter: liste filer med maskinlesbar JSON-utdata, laste opp og kontrollere delinger

Passord og TOTP-nøkkel trenger dermed ikke å stå i en konfigurasjonsfil. For serverdrift gjenstår likevel tre begrensninger: CLI-en kan **ikke montere et filsystem**, innloggingen åpner en nettleser, og Proton klassifiserer fortsatt ikke SDK-et som produksjonsklart for tredjepartsapplikasjoner. Utgivelsen er planlagt til slutten av 2026 til begynnelsen av 2027.

## Det egentlige hullet: maskintilganger

Kjernen i problemet ligger ett nivå dypere enn klient eller SDK: **Proton har ingen maskintilganger.** Ingen app-passord, ingen tjenestekonto, ingen token med begrenset omfang. Hver automatisering, enten det er et sikkerhetskopieringsskript, en servermontering eller en CI-jobb, må arbeide med kontoens fullverdige påloggingsopplysninger.

Til sammenligning er tilgangsnøkkelpar normalen hos S3-kompatibel lagring, de kan tilbakekalles og begrenses til bøtter eller prefikser. Google og Microsoft har app-passord og tjenestekontoer. Hos Proton gjelder derimot alt eller ingenting: Den som vil gi en server tilgang til én mappe, gir den tilgang til hele kontoen.

Rettferdigvis er dette vanskeligere hos en ende-til-ende-kryptert tjeneste enn hos S3, fordi begrenset tilgang også måtte innebære begrenset nøkkelmateriale. SDK-øktene viser imidlertid at Proton behersker slike konstruksjoner. En økt er allerede en avledet tilgang som kan tilbakekalles. En offisiell «maskintoken for akkurat denne mappen, bare lesing» ville være det største enkeltstående fremskrittet for serverbruk, lenge før enhver klient.

## Anbefaling etter bruksområde

| Bruksområde | Status juli 2026 |
|---|---|
| Skrivebordssynkronisering på Linux | Vent på den annonserte klienten; inntil da Rclone-synkronisering eller nettgrensesnittet |
| Sikkerhetskopiering på server (laste opp filer) | Rclone med `copy` eller `sync`; fungerer, men ta høyde for beta-status |
| Filsystemmontering for tjenester | Rclone med `mount`, lagret TOTP-nøkkel og dedikert konto; den eneste [utprøvde veien i praksis](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering med ryddig innlogging | Følg med på SDK-CLI-en; fortsatt for tidlig for produksjon |

På Linux-skrivebordet kan man vente på den annonserte klienten eller bruke Rclone foreløpig. På servere er Rclone fortsatt den eneste praktiske monteringsløsningen. Et fungerende hjelpemiddel blir imidlertid først en robust plattform når Proton tilbyr begrensede maskintilganger og en offisielt støttet montering.

## Kilder

1.  [OMG Ubuntu: Proton Drive-klient kommer (endelig) til Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekreftelsen fra juni 2026 på at Linux-klienten er under utvikling, uten dato.

2.  [Proton: Produktveikart for våren og sommeren 2026](https://proton.me/blog/2026-spring-summer-roadmaps): veikartet med Linux-klienten uten tidsvindu og SDK-et som grunnlag for Protons egne apper.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentlige SDK-repositoriet med CLI, nettleserinnlogging og økt i nøkkellageret.

4.  [Forhåndsvisning av Proton Drive SDK](https://proton.me/blog/proton-drive-sdk-preview): Protons egen vurdering: fortsatt ikke produksjonsklart for tredjepartsapplikasjoner.

5.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backenden med beta-merknad og alternativet `otp_secret_key` for uovervåket innlogging.
