---
title: "Köra Paperless-ngx med lite lagringsutrymme: flytta dokument till en molntjänst"
navTitle: "Paperless med molntjänst"
description: "Paperless-ngx behöver bara databasen, sökindexet och förhandsvisningsbilderna lokalt; själva dokumenten kan ligga i en molntjänst. Vad det praktiska testet visade och hur installationen med den färdiga mallen lyckas med tre kommandon."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min lästid"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-begransat-lagringsutrymme-dokument-i-molnet"
translationOf: "paperless-dokumente-clouddienst-auslagern"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:25:28.021Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/sv/blog/paperless-ngx-begransat-lagringsutrymme-dokument-i-molnet
---

Paperless-ngx lagrar sina dokument i en lokal katalog, och den katalogen växer med varje inskanning. I vardagen behöver Paperless dock knappt filerna: sökningen körs mot databasen, listan visar förhandsvisningsbilder och själva filen läses först när den öppnas. Därför testade jag om lagringen kan flyttas till en molntjänst. Verktyget för detta är Rclone, som Plex-användare sedan åratal använder för att montera hela mediesamlingar från molnet.

Resultatet: **Det fungerar åt båda hållen**, och installationen har nu krympt till tre kommandon. Den här artikeln sammanfattar vad testet visade och hur du kan sätta upp lösningen själv. De tekniska detaljerna finns i separata artiklar som länkas i slutet: Docker-mount-propagation, AppArmor-särdrag, tvåfaktorsautentisering och testmetodik.

## Principen: Hot Storage stannar lokalt, Cold Storage ligger i molnet

| Beståndsdel | Plats | Varför |
|---|---|---|
| Databas (innehåller OCR-texten) | lokalt | behöver riktig låsning |
| Sökindex, förhandsvisningsbilder | lokalt | ständiga åtkomster |
| **Dokumentfiler** | **Moln** | läses sällan |
| Cache (senast öppnade dokument) | lokalt, begränsad | upprepade åtkomster förblir snabba |

I Paperless är just katalognamnet missvisande: `archive/` är **inte Cold Storage**, utan innehåller PDF/A-versionen som levereras vid varje visning. Trots namnet hör den till Hot Storage. Originalen under `originals/` som sällan behövs är den egentliga Cold Storage. Om du vill spara maximalt kan du stänga av arkivkopian helt med `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; fulltextsökningen påverkas inte, eftersom texten finns i databasen.

Paperless-ngx har för övrigt ingen egen molnanslutning, varken S3 eller `django-storages`. En filsystemsmount via Rclone är för närvarande den enda vägen, och den fungerar med var och en av de över 70 tjänster som Rclone stöder. Proton Drive var mitt testval på grund av end-to-end-krypteringen; S3-kompatibel lagring är det robustare alternativet.

## Vad testet visade

Testat med en isolerad Paperless-instans, 40 genererade test-PDF:er (13.9 MB) och ett dedikerat Proton-konto:

| Åtgärd | Resultat |
|---|---|
| Öppna dokument för första gången (från molnet) | ~1.8 s |
| Öppna samma dokument igen (från cachen) | ~20 ms |
| Läsa in ett nytt dokument tills det finns i molnet | ~20 s |
| Dokumentlista, fulltextsökning | 39 ms / 272 ms, fungerar även **utan** molnanslutning |
| Integritetskontroll (kontrollsumma för varje fil) | godkänd, inga avvikelser |
| Mounten faller bort | självläkning utan omstart av Paperless, verifierat |

Det lokala lagringsbehovet är därmed frikopplat från arkivets storlek: samlingen får växa, disken behöver inte göra det.

## Så ställer du in det

Den kompletta konfigurationen finns som en mall på GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). På servern:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

Guiden frågar efter molntjänsten (Proton, S3, Backblaze B2, WebDAV, SFTP eller ”Not in the list” för alla andra Rclone-tjänster), kontrollerar anslutningen med ett verkligt upp- och nedladdningstest och startar lagringscontainern. Därefter:

- **Nyinstallation:** `docker compose -f paperless.yml up -d`, klart.
- **Befintlig Paperless-instans:** Databas och inställningar förblir orörda; instruktionen [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) beskriver uppladdningen av befintliga dokument och den nödvändiga ändringen i din Compose-fil.

Jag har medvetet avstått från ett webbgränssnitt. Rclones webb-GUI användes först, men SSH-tunnlar, CORS och tillfälliga mountar gjorde det sämre än kommandoraden som det skulle ersätta. Tre frågor i terminalen går snabbare.

## Så förblir mounten stabil i vardagen

Mallen hanterar fyra punkter som du också måste beakta vid en egen uppsättning:

1. **`propagation: rslave`** på Paperless-containerns media-bind-mount, annars överlever containern inte en omstart av mounten. Detaljer och AppArmor-problemet bakom: [Rclone-mount i Docker-containern](/blog/rclone-mount-in-docker-container).
2. **Stoppa Paperless när mounten saknas.** Annars skriver den dokument till en tom lokal katalog, och den återvändande mounten döljer dem osynligt. Ett watchdog-skript ingår i mallen.
3. **Ett konto som kan logga in utan övervakning.** För Proton innebär det att TOTP-nyckeln måste sparas i Rclone-konfigurationen. Varför det inte urholkar tvåfaktorsautentiseringen och hur läget för Proton under Linux ser ut i stort: [Proton Drive under Linux](/blog/proton-drive-linux-status).
4. **Stäng av schemalagda uppgifter som läser hela innehållet** (`PAPERLESS_SANITY_TASK_CRON=disable`), eftersom integritetskontrollen annars regelbundet läser hela beståndet från molnet.

## Vad du bör överväga före användning

Ett nyinläst dokument finns bara i den lokala cachen under några sekunder tills uppladdningen är klar. Om maskinen fallerar just i det fönstret saknas filen. Cachegränsen är mjuk och kan tillfälligt överskridas betydligt vid åtkomsttoppar. Dessutom är Rclones Proton-backend officiellt i beta; vid snabba API-anrop visade den tecken på strypning. Eftersom långtidsdata från kontinuerlig drift ännu saknas är mallen märkt som experimentell.

Hur mätvärdena togs fram, vilka fel som simulerades och hur en sådan uppsättning över huvud taget kan testas seriöst beskrivs i metodartikeln: [Testa molnmountar med genererade PDF:er](/blog/cloud-mount-testen-dummy-pdfs).

## Slutsats

Paperless-ngx på en liten disk med molnlagring är genomförbart och användbart i vardagen: knappt två sekunder vid första öppningen, därefter cachehastighet, sökning och gränssnitt förblir oberoende av molnet, och uppsättningen självläker efter avbrott. Om du på en normalt dimensionerad server bara vill spara några gigabyte bör du dock räkna efter: i mitt fall tog hela lagringen upp 71 MB, medan operativsystemet tog flera gigabyte. Vinsten ligger inte i det omedelbart sparade utrymmet, utan i att beståndet får växa utan att disken måste växa med det.

## Källor

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): mallen från den här artikeln: setup.sh, wizard.sh, Compose-filer, watchdog och eftermonteringsinstruktion.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): de över 70 tjänster som stöds och deras funktioner i jämförelse.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` och övriga använda inställningar.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): Sanity Checker, export och import samt de schemalagda bakgrundsuppgifterna.
