---
title: "Kjør Paperless-ngx med lite lagringsplass: Flytt dokumenter til en skytjeneste"
navTitle: "Paperless med skytjeneste"
description: "Paperless-ngx trenger bare databasen, søkeindeksen og forhåndsvisningsbildene lokalt; selve dokumentene kan ligge i en skytjeneste. Hva praktisk testing viste, og hvordan oppsettet lykkes med den ferdige malen på tre kommandoer."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min lesetid"
themen:
  - "paperless-ngx"
related:
  - "rclone-mount-in-docker-container"
  - "proton-drive-linux-status"
  - "cloud-mount-testen-dummy-pdfs"
slug: "paperless-ngx-lite-lagringsplass-dokumenter-i-skyen"
translationOf: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/no/blog/paperless-ngx-lite-lagringsplass-dokumenter-i-skyen"
---

Paperless-ngx lagrer dokumentene sine i en lokal katalog, og denne katalogen vokser med hver skanning. I praksis trenger Paperless knapt filene: Søket går mot databasen, listen viser forhåndsvisningsbilder, og selve filen leses først når den åpnes. Derfor testet jeg om lagringen kan flyttes til en skytjeneste. Verktøyet for dette er Rclone, som Plex-brukere i årevis har brukt til å koble hele mediesamlinger fra skyen.

Resultatet: **Det fungerer begge veier**, og oppsettet er nå redusert til tre kommandoer. Denne artikkelen oppsummerer hva testen viste, og hvordan du setter opp løsningen selv. De tekniske detaljene finnes i egne artikler som er lenket nederst: Docker-mount propagation, AppArmor-feller, tofaktorautentisering og målemetodikk.

## Prinsippet: Hot storage forblir lokalt, cold storage ligger i skyen

| Bestanddel | Sted | Hvorfor |
|---|---|---|
| Database (inneholder OCR-teksten) | lokalt | trenger ekte låsing |
| Søkeindeks, forhåndsvisningsbilder | lokalt | kontinuerlige tilganger |
| **Dokumentfiler** | **Sky** | leses sjelden |
| Cache (sist åpnede dokumenter) | lokalt, begrenset | gjentatte tilganger forblir raske |

I Paperless er det nettopp katalognavnet som er misvisende: `archive/` er **ikke cold storage**, men inneholder PDF/A-versjonen som leveres ved hver visning. Til tross for navnet hører den til hot storage. Originalene som sjelden trengs, under `originals/`, er den egentlige cold storage. Hvis du vil spare mest mulig, kan du slå av arkivkopien helt med `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; fulltekstsøket påvirkes ikke av dette, fordi teksten ligger i databasen.

Paperless-ngx har for øvrig ingen egen skytilkobling, verken S3 eller `django-storages`. Et filsystem-mount via Rclone er for øyeblikket den eneste veien, og det fungerer med hver av de over 70 tjenestene som støttes av Rclone. Proton Drive var mitt testvalg på grunn av ende-til-ende-krypteringen; S3-kompatibel lagring er det mer robuste alternativet.

## Hva testen viste

Testet med en isolert Paperless-instans, 40 genererte test-PDF-er (13,9 MB) og en dedikert Proton-konto:

| Handling | Resultat |
|---|---|
| Åpne dokument for første gang (fra skyen) | ~1,8 s |
| Åpne samme dokument på nytt (fra cachen) | ~20 ms |
| Legge til nytt dokument til det ligger i skyen | ~20 s |
| Dokumentliste, fulltekstsøk | 39 ms / 272 ms, fungerer også **uten** skyforbindelse |
| Integritetskontroll (sjekksum for hver fil) | bestått, ingen avvik |
| Mount-feil | selvreparasjon uten omstart av Paperless, verifisert |

Det lokale lagringsbehovet er dermed frikoblet fra arkivets størrelse: Samlingen kan vokse, men disken trenger ikke gjøre det.

## Slik setter du det opp

Den komplette konfigurasjonen finnes som mal på GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). På serveren:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # én gang, klargjør verten (det eneste root-trinnet)
./wizard.sh          # veiledet: velg leverandør, innloggingsdata, rundturstest
```

Veiviseren spør etter skytjenesten (Proton, S3, Backblaze B2, WebDAV, SFTP eller «Not in the list» for enhver annen Rclone-tjeneste), kontrollerer forbindelsen med en reell opplastings- og nedlastingstest og starter lagringscontaineren. Deretter:

- **Nyinstallasjon:** `docker compose -f paperless.yml up -d`, ferdig.
- **Eksisterende Paperless-instans:** Database og innstillinger forblir urørt; veiledningen [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) beskriver opplasting av eksisterende dokumenter og den nødvendige endringen i Compose-filen din.

Jeg har bevisst valgt bort et webgrensesnitt. Rclones web-GUI var først i bruk, men SSH-tunneler, CORS og flyktige mounts gjorde den verre enn kommandolinjen den skulle erstatte. Tre spørsmål i terminalen går raskere.

## Slik holder mountet seg stabilt i hverdagen

Malen tar seg av fire punkter som du også må ta hensyn til ved et eget oppsett:

1. **`propagation: rslave`** på media-bind-mountet til Paperless-containeren, ellers overlever ikke containeren en omstart av mountet. Detaljer og AppArmor-fellen bak dette: [Rclone-mount i Docker-container](/blog/rclone-mount-in-docker-container).
2. **Stopp Paperless når mountet mangler.** Ellers skriver det dokumenter til en tom lokal katalog, og mountet som kommer tilbake dekker dem usynlig. Et watchdog-skript følger med i malen.
3. **En konto som kan logge inn uten tilsyn.** For Proton betyr det å lagre TOTP-nøkkelen i Rclone-konfigurasjonen. Hvorfor dette ikke svekker tofaktorautentiseringen og hvordan Proton generelt står på Linux: [Proton Drive på Linux](/blog/proton-drive-linux-status).
4. **Slå av planlagte full-lese-oppgaver** (`PAPERLESS_SANITY_TASK_CRON=disable`), fordi integritetskontrollen ellers regelmessig leser hele beholdningen fra skyen.

## Hva du bør vurdere før bruk

Et nylig lagt til dokument ligger i noen sekunder bare i den lokale cachen, til opplastingen er ferdig. Hvis maskinen dør akkurat i dette vinduet, mangler filen. Cache-grensen er myk og kan ved tilgangstopper kortvarig overskrides betydelig. Rclones Proton-backend er dessuten offisielt beta; ved raske API-kall viste den symptomer på struping. Fordi langtidsdata fra kontinuerlig drift fortsatt mangler, er malen merket som eksperimentell.

Hvordan måleverdiene ble til, hvilke feil som ble simulert og hvordan et slikt oppsett i det hele tatt kan testes seriøst, står i metodeartikkelen: [Teste sky-mounts med genererte PDF-er](/blog/cloud-mount-testen-dummy-pdfs).

## Konklusjon

Paperless-ngx på en liten disk med skylagring er gjennomførbart og egnet i hverdagen: knapt to sekunder ved første åpning, deretter cache-hastighet, søk og grensesnitt forblir uavhengige av skyen, og oppsettet reparerer seg selv etter feil. Hvis du bare vil spare noen gigabyte på en normalt dimensjonert server, bør du likevel regne på det: I mitt tilfelle brukte hele lagringen 71 MB, mens operativsystemet brukte flere gigabyte. Gevinsten ligger ikke i plassen du sparer umiddelbart, men i at beholdningen kan vokse uten at disken må vokse med den.

## Kilder

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): malen fra denne artikkelen: setup.sh, wizard.sh, Compose-filer, watchdog og retrofit-veiledning.

2.  [Rclone: Oversikt over skylagringssystemer](https://rclone.org/overview/): de over 70 støttede tjenestene og funksjonene deres sammenlignet.

3.  [Paperless-ngx: Konfigurasjon](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` og de øvrige innstillingene som brukes.

4.  [Paperless-ngx: Administrasjon](https://docs.paperless-ngx.com/administration/): sanity checker, eksport og import samt de planlagte bakgrunnsoppgavene.
