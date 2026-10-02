---
title: "Proton Drive på Linux: Status i oktober 2026"
navTitle: "Proton Drive og Linux"
description: "Den offisielle Linux-klienten er annonsert, men ennå ikke tilgjengelig. For skript og servere har den offisielle Proton Drive CLI vært tilgjengelig siden juni 2026; Proton Drive kan fortsatt bare monteres med Rclone. Det som mangler, er maskintilgang begrenset til enkeltmapper eller oppgaver."
date: "2026-10-01"
kategorie: "Proton Drive"
timeToRead: "8 min. lesetid"
themen:
  - proton-drive
  - rclone
related:
  - proton-drive-cli
  - paperless-dokumente-clouddienst-auslagern
  - rclone-mount-in-docker-container
slug: "proton-drive-pa-linux-status-i-juli-2026"
translationOf: "proton-drive-linux-status"
translationId: article-ca282447e0b9acff
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:03:23.149Z
translationReview: automatic
translationSourceHash: 73500e1be526e5e93bbd7bf789b1cc500976140cb4cb9542404e1e7a56b40ce0
url: https://rafaelpfister.ch/no/blog/proton-drive-pa-linux-status-i-juli-2026
---

For Windows og macOS har Proton Drive tilbudt egne synkroniseringsklienter siden 2023. På Linux finnes foreløpig nettgrensesnittet, fellesskapsverktøy og siden juni 2026 et offisielt kommandolinjeprogram, men fortsatt ingen synkroniseringsklient. På en server er situasjonen enda vanskeligere, fordi verken synkronisering for skrivebord eller interaktiv pålogging passer godt der.

Denne oversikten beskriver status per 1. oktober 2026. Grunnlaget er de publiserte veikartene, kildekoden til Proton Drive CLI og en praktisk test av Rclone-backenden [som dokumentlager for Paperless-ngx](/blog/paperless-dokumente-clouddienst-auslagern).

**Oppdatering 1. oktober 2026:** Den første versjonen fra 26. juli beskrev kommandolinjeprogrammet bare som et verktøy i SDK-repositoriet. Proton hadde imidlertid allerede 9. juni 2026 offisielt lansert det som **Proton Drive CLI**, med ferdige bygg for Windows, macOS og Linux. Avsnittet om dette og anbefalingstabellen er oppdatert tilsvarende; detaljene finnes i den egne artikkelen om [Proton Drive CLI](/blog/proton-drive-cli).

## Linux-klienten er annonsert, men fortsatt uten dato

I juni 2026 bekreftet Proton for første gang uttrykkelig at en Linux-klient er under utvikling. Den bygger på det nye, enhetlige SDK-et og skal bruke samme tekniske grunnlag som programmene for Windows og macOS. I begynnelsen av oktober 2026 finnes det fortsatt verken en dato eller en offentlig beta.

Viktig for vurderingen: Dette blir en **synkroniseringsklient for skrivebordet**. For skrivebordet løser den problemet. For serverprogrammer er en synkroniseringsklient derimot feil verktøy, fordi en tjeneste skal lese filer direkte fra Proton Drive og skrive dem dit. En synkroniseringsklient holder en full lokal kopi, nettopp det man vil unngå ved begrenset lagringsplass.

## Rclone er fortsatt nødvendig for monteringer og speiling

På Linux er Rclone med sin `protondrive`-backend for tiden det mest allsidige verktøyet. Det kan kopiere og synkronisere filer og er den eneste tilgjengelige løsningen som kan gjøre Proton Drive tilgjengelig som en lokal katalog via **FUSE-montering**. To begrensninger er viktige:

**Det er beta på et rekonstruert API.** Proton dokumenterer ikke Drive-API-et offentlig; backenden er basert på omvendt utvikling. I testen fungerte den pålitelig, men ved raske kallsekvenser ble den strupet med inkonsekvente kataloglister.

**For uovervåket drift ber Rclone om TOTP-nøkkelen.** Konfigurasjonsveiviseren kaller feltet `otp_secret_key`. Det menes den permanente nøkkelen fra 2FA-oppsettet, ikke den sekssifrede koden som en autentiseringsapp viser akkurat nå. Rclone lagrer denne verdien tilslørt og genererer selv en gyldig TOTP-kode fra den ved hver pålogging.

Den som ved en feil oppgir en aktuell engangskode, kan fullføre den første påloggingen. Neste reautentisering mislykkes imidlertid med feil 8002, fordi Rclone ikke kan bruke den samme koden én gang til.

Dermed er kontoen fortsatt beskyttet mot et isolert stjålet passord. En kompromittert server avslører imidlertid passord og TOTP-nøkkel. For automatiserte tilganger anbefales derfor en **dedikert Proton-konto**.

Hvordan en slik montering oppfører seg i Docker-miljøer, inkludert to udokumenterte problemer, står i [den egne artikkelen om Rclone i containere](/blog/rclone-mount-in-docker-container).

## Den offisielle CLI-en dekker skript og sikkerhetskopiering

9. juni 2026 lanserte Proton **Proton Drive CLI**, én enkelt kjørbar fil `proton-drive` for Windows, macOS og Linux. Den er basert på samme SDK som de offisielle appene; kildekoden ligger i det offentlige SDK-repositoriet. Nåværende versjon er 0.8.0 fra 13. august 2026, med bygg for x86-64 (også uten AVX2), ARM64 og musl-distribusjoner som Alpine.

Påloggingsmodellen er ryddigere enn den til Rclone-backenden:

- `auth login` gir en påloggings-URL som også kan åpnes **på en annen enhet**; påloggingen skjer normalt **inkludert tofaktorautentisering**, altså også via SSH på en server uten skrivebord
- økten havner i **operativsystemets nøkkellager** (Keychain, Credential Manager, libsecret) eller, siden versjon 0.6.0, i passordbehandleren `pass`, som er mer praktisk på servere uten skrivebordsøkt
- deretter: laste opp og ned filer, flytte dem, legge dem i papirkurven, administrere delinger, invitasjoner og offentlige lenker, bruke Proton Photos; hver gang med `--json` for maskinlesbar utdata

Passord og TOTP-nøkkel trenger dermed ikke ligge på serveren. For sikkerhetskopier og byggartefakter er CLI-en derfor i dag et bedre valg enn Rclone. To begrensninger gjenstår: CLI-en kan **ikke montere et filsystem** og **ikke opprette et speil med slettinger**; den laster opp og ned, men synkroniserer ikke. En kommando `takeout` for en fullstendig lokal eksport finnes allerede i repositoriet, men er ennå ikke lansert.

Proton vurderer fortsatt selve SDK-et som ikke produksjonsklart for tredjepartsprogrammer; lansering er planlagt til slutten av 2026 eller begynnelsen av 2027. CLI-en er ikke berørt av dette, fordi Proton selv gir den ut.

## Det egentlige gapet: maskintilganger

Kjernen i problemet ligger ett nivå dypere enn klient eller SDK: **Proton har ingen maskintilganger.** Ingen app-passord, ingen tjenestekonto, ingen token med begrenset omfang. All automatisering, enten det er sikkerhetskopieringsskript, servermontering eller CI-jobb, må arbeide med kontoens fullverdige tilgangsopplysninger.

Til sammenligning: Med S3-kompatibel lagring er tilgangsnøkkelpar normalen, og de kan tilbakekalles og begrenses til bucketer eller prefikser. Google og Microsoft har app-passord og tjenestekontoer. Hos Proton gjelder derimot alt eller ingenting: Den som vil gi en server tilgang til en mappe, gir den tilgang til hele kontoen.

For en ende-til-ende-kryptert tjeneste er dette vanskeligere enn med S3, fordi begrenset tilgang også måtte bety begrenset nøkkelmateriale. CLI-øktene viser imidlertid at Proton behersker slike konstruksjoner. En økt er allerede en avledet tilgang som kan tilbakekalles, bare med kontoens fulle omfang. En offisiell «maskintoken for akkurat denne mappen, kun lesing» ville vært det største enkeltstående fremskrittet for serverbruk, langt foran enhver klient.

## Anbefaling etter bruksområde

| Bruksområde | Status oktober 2026 |
|---|---|
| Synkronisering på Linux-skrivebord | Vent på den annonserte klienten; inntil da Rclone-synkronisering eller nettgrensesnittet |
| Serversikkerhetskopi (laste opp filer) | [Proton Drive CLI](/blog/proton-drive-cli) med `filesystem upload` og konfliktstrategien `create-new-revision`; offisielt støttet, uten lagret passord |
| Speil med slettinger | Rclone med `sync`; ta høyde for beta-status |
| Filsystemmontering for tjenester | Rclone med `mount`, lagret TOTP-nøkkel og dedikert konto; den eneste [utprøvde løsningen i praksis](/blog/paperless-dokumente-clouddienst-auslagern) |
| Skriptautomatisering, administrere delinger | Proton Drive CLI med `--json`; versjonsnivå 0.x, kommandoer kan fortsatt endres |

På Linux-skrivebordet kan man vente på den annonserte klienten eller foreløpig bruke Rclone. På servere tar den offisielle CLI-en nå hånd om sikkerhetskopiering og automatisering; for en montering er Rclone fortsatt den eneste praktiske løsningen. En fungerende nødløsning blir imidlertid først en robust plattform når Proton tilbyr begrensede maskintilganger og en offisielt støttet montering.

## Kilder

1.  [OMG Ubuntu: Proton Drive client is (finally) coming to Linux](https://www.omgubuntu.co.uk/2026/06/proton-drive-linux-client): bekreftelsen fra juni 2026 på at Linux-klienten er under utvikling, uten dato.

2.  [Proton: Product roadmaps for spring and summer 2026](https://proton.me/blog/2026-spring-summer-roadmaps): veikartet med Linux-klienten uten tidsvindu og SDK-et som grunnlag for egne apper.

3.  [ProtonDriveApps/sdk på GitHub](https://github.com/ProtonDriveApps/sdk): det offentlige SDK-repositoriet med kildekoden og endringsloggen for CLI-en.

4.  [Proton: Introducing Proton Drive CLI](https://proton.me/blog/proton-drive-cli): den offisielle lanseringen av CLI-en 9. juni 2026.

5.  [Proton Drive CLI: Downloads](https://proton.me/download/drive/cli/index.html): nåværende versjon 0.8.0 fra 13. august 2026 med alle plattformbyggene.

6.  [Proton Drive SDK preview](https://proton.me/blog/proton-drive-sdk-preview): Protons egen vurdering: fortsatt ikke produksjonsklart for tredjepartsprogrammer.

7.  [Rclone: Proton Drive](https://rclone.org/protondrive/): backenden med beta-merknad og alternativet `otp_secret_key` for uovervåket pålogging.
