---
title: "Kjøre Paperless-ngx med lite lagringsplass: Flytt dokumenter til en skytjeneste"
navTitle: "Paperless med skytjeneste"
description: "Paperless-ngx trenger bare database, søkeindeks og forhåndsvisningsbilder lokalt; selve dokumentene kan ligge i en skytjeneste. Hva den praktiske testen viste, og hvordan oppsettet med den ferdige malen lykkes med tre kommandoer."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min lesetid"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-lite-lagringsplass-dokumenter-i-skyen"
translationOf: "paperless-dokumente-clouddienst-auslagern"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:25:51.480Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/no/blog/paperless-ngx-lite-lagringsplass-dokumenter-i-skyen
---

Paperless-ngx lagrer dokumentene sine i en lokal katalog, og denne katalogen vokser med hver skanning. I praksis trenger Paperless knapt filene: Søket kjører mot databasen, listen viser forhåndsvisningsbilder, og selve filen leses først når den åpnes. Derfor testet jeg om lagringen kan flyttes til en skytjeneste. Verktøyet for dette er Rclone, som Plex-brukere i årevis har brukt til å montere hele mediesamlinger fra skyen.

Resultatet: **Det fungerer begge veier**, og oppsettet er nå redusert til tre kommandoer. Denne artikkelen oppsummerer hva testen viste, og hvordan du kan sette opp løsningen selv. De tekniske detaljene finnes i egne artikler som er lenket nederst: Docker-mount-propagasjon, AppArmor-særegenheter, tofaktorautentisering og målemetodikken.

## Prinsippet: Hot Storage forblir lokalt, Cold Storage ligger i skyen

| Bestanddel | Sted | Hvorfor |
|---|---|---|
| Database (inneholder OCR-teksten) | lokalt | trenger ekte låsing |
| Søkeindeks, forhåndsvisningsbilder | lokalt | kontinuerlig tilgang |
| **Dokumentfiler** | **Sky** | leses sjelden |
| Cache (sist åpnede dokumenter) | lokalt, begrenset | gjentatte tilganger forblir raske |

I Paperless er det nettopp katalognavnet som er misvisende: `archive/` er **ikke Cold Storage**, men inneholder PDF/A-versjonen som leveres ved hver visning. Til tross for navnet hører den til Hot Storage. De sjelden brukte originalene under `originals/` er den egentlige Cold Storage. Hvis du vil spare maksimalt, kan du slå av arkivkopien helt med `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; fulltekstsøket påvirkes ikke av dette, fordi teksten ligger i databasen.

Paperless-ngx har for øvrig ingen egen skyintegrasjon, verken S3 eller `django-storages`. Et filsystem-mount via Rclone er for tiden den eneste løsningen, og den fungerer med hver av de over 70 tjenestene som støttes av Rclone. Proton Drive var mitt testvalg på grunn av ende-til-ende-kryptering; S3-kompatibel lagring er det mer robuste alternativet.

## Hva testen viste

Testet med en isolert Paperless-instans, 40 genererte test-PDF-er (13.9 MB) og en dedikert Proton-konto:

| Handling | Resultat |
|---|---|
| Åpne et dokument for første gang (fra skyen) | ~1.8 s |
| Åpne samme dokument igjen (fra cachen) | ~20 ms |
| Legge til et nytt dokument, til det ligger i skyen | ~20 s |
| Dokumentliste, fulltekstsøk | 39 ms / 272 ms, fungerer også **uten** skyforbindelse |
| Integritetskontroll (sjekksum for hver fil) | bestått, ingen avvik |
| Bortfall av mountet | selvreparasjon uten omstart av Paperless, verifisert |

Det lokale lagringsbehovet er dermed frikoblet fra størrelsen på arkivet: Samlingen kan vokse, disken trenger ikke det.

## Slik setter du det opp

Den komplette konfigurasjonen finnes som mal på GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). På serveren:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

Veiviseren spør etter skytjenesten (Proton, S3, Backblaze B2, WebDAV, SFTP eller «Not in the list» for enhver annen Rclone-tjeneste), tester forbindelsen med en faktisk opplastings- og nedlastingstest og starter Storage-containeren. Deretter:

- **Nyinstallasjon:** `docker compose -f paperless.yml up -d`, ferdig.
- **Eksisterende Paperless-instans:** Database og innstillinger forblir urørt; veiledningen [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) beskriver opplasting av eksisterende dokumenter og den nødvendige endringen i Compose-filen din.

Jeg har bevisst unnlatt et webgrensesnitt. Rclones web-GUI var først i bruk, men SSH-tunneler, CORS og flyktige mount gjorde den verre enn kommandolinjen den skulle erstatte. Tre spørsmål i terminalen går raskere.

## Slik holder mountet seg stabilt i hverdagen

Malen tar seg av fire punkter som du også må ta hensyn til dersom du bygger opp løsningen selv:

1. **`propagation: rslave`** på Paperless-containerens media-bind-mount, ellers overlever ikke containeren en omstart av mountet. Detaljer og AppArmor-problemet bak dette: [Rclone-mount i Docker-container](/blog/rclone-mount-in-docker-container).
2. **Stopp Paperless når mountet mangler.** Ellers skriver det dokumenter til en tom lokal katalog, og mountet som kommer tilbake dekker dem usynlig til. Et watchdog-skript følger med malen.
3. **En konto som kan logge inn uten tilsyn.** For Proton betyr det: lagre TOTP-nøkkelen i Rclone-konfigurasjonen. Hvorfor dette ikke svekker tofaktorautentiseringen, og hvordan Proton generelt står seg under Linux: [Proton Drive under Linux](/blog/proton-drive-linux-status).
4. **Slå av planlagte oppgaver som leser alt** (`PAPERLESS_SANITY_TASK_CRON=disable`), fordi integritetskontrollen ellers regelmessig leser hele bestanden fra skyen.

## Hva du bør vurdere før bruk

Et nylig lagt til dokument ligger i noen sekunder bare i den lokale cachen, til opplastingen er fullført. Hvis maskinen svikter akkurat i dette vinduet, mangler filen. Cache-grensen er myk og kan ved tilgangstopper overskrides betydelig i korte perioder. Og Rclones Proton-backend er offisielt Beta; ved raske API-kall viste den tegn til begrensning. Fordi langtidsdata fra kontinuerlig drift fortsatt mangler, er malen merket som eksperimentell.

Hvordan måleverdiene ble til, hvilke feil som ble simulert og hvordan et slikt oppsett i det hele tatt kan testes på en seriøs måte, står i metodeartikkelen: [Teste sky-mount med genererte PDF-er](/blog/cloud-mount-testen-dummy-pdfs).

## Konklusjon

Paperless-ngx på en liten disk med skylagring er mulig og egnet for hverdagsbruk: knapt to sekunder ved første åpning, deretter cache-hastighet, søk og brukergrensesnitt forblir uavhengige av skyen, og oppsettet reparerer seg selv etter feil. Hvis du bare vil spare noen gigabyte på en normalt dimensjonert server, bør du imidlertid regne på det: I mitt tilfelle tok hele lagringen 71 MB, mens operativsystemet tok flere gigabyte. Gevinsten ligger ikke i plassen som spares umiddelbart, men i at bestanden kan vokse uten at disken må vokse med den.

## Kilder

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): malen fra denne artikkelen: setup.sh, wizard.sh, Compose-filer, watchdog og retrofit-veiledning.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): de over 70 støttede tjenestene og egenskapene deres sammenlignet.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` og de øvrige innstillingene som brukes.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): Sanity Checker, eksport og import samt de planlagte bakgrunnsoppgavene.
