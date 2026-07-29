---
title: "Köra Paperless-ngx med begränsat lagringsutrymme: lagra dokument i en molntjänst"
navTitle: "Paperless med molntjänst"
description: "Paperless-ngx behöver bara databasen, sökindexet och miniatyrbilderna lokalt; själva dokumenten kan ligga i en molntjänst. Vad det praktiska testet visade och hur installationen med den färdiga mallen lyckas med tre kommandon."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min läsning"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-begransat-lagringsutrymme-dokument-i-molnet"
translationOf: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/sv/blog/paperless-ngx-begransat-lagringsutrymme-dokument-i-molnet"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:26:21.452Z
translationReview: automatic
translationSourceHash: 81212f097221ec6213025dc5de54f583369799181f72747549102e2b4246e021
---

Paperless-ngx lagrar sina dokument i en lokal katalog, och den katalogen växer med varje skanning. I vardagen behöver Paperless dock knappt filerna: sökningen körs mot databasen, listan visar miniatyrbilder och själva filen läses först när den öppnas. Därför testade jag om lagringen kan flyttas till en molntjänst. Verktyget för detta är Rclone, som Plex-användare i flera år har använt för att ansluta hela mediesamlingar från molnet.

Resultatet: **Det fungerar åt båda hållen**, och installationen har numera krympt till tre kommandon. Den här artikeln sammanfattar vad testet visade och hur du själv sätter upp lösningen. De tekniska detaljerna finns i separata artiklar som länkas i slutet: Docker-mount-propagation, AppArmor-fällor, tvåfaktorsautentisering och mätmetodik.

## Principen: Hot Storage stannar lokalt, Cold Storage ligger i molnet

| Beståndsdel | Plats | Varför |
|---|---|---|
| Databas (innehåller OCR-texten) | lokalt | behöver riktig låsning |
| Sökindex, miniatyrbilder | lokalt | ständiga åtkomster |
| **Dokumentfiler** | **molnet** | läses sällan |
| Cache (senast öppnade dokument) | lokalt, begränsad | upprepade åtkomster förblir snabba |

I Paperless är det just katalognamnet som vilseleder: `archive/` är **inte Cold Storage**, utan innehåller PDF/A-versionen som levereras vid varje visning. Trots namnet hör den till Hot Storage. Originalen som sällan behövs under `originals/` är den egentliga Cold Storage. Om du vill spara maximalt stänger du av arkivkopian helt med `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; fulltextsökningen påverkas inte av detta, eftersom texten finns i databasen.

Paperless-ngx har för övrigt ingen egen molnanslutning, varken S3 eller `django-storages`. En filsystemsmount via Rclone är för närvarande den enda vägen, och den fungerar med alla de över 70 tjänster som Rclone stöder. Proton Drive var mitt testval på grund av end-to-end-krypteringen; S3-kompatibel lagring är det robustare alternativet.

## Vad testet visade

Testat med en isolerad Paperless-instans, 40 genererade test-PDF:er (13,9 MB) och ett dedikerat Proton-konto:

| Åtgärd | Resultat |
|---|---|
| Öppna dokument för första gången (från molnet) | ~1,8 s |
| Öppna samma dokument igen (från cachen) | ~20 ms |
| Ta in nytt dokument tills det finns i molnet | ~20 s |
| Dokumentlista, fulltextsökning | 39 ms / 272 ms, fungerar även **utan** molnanslutning |
| Integritetskontroll (kontrollsumma för varje fil) | godkänd, inga avvikelser |
| Mounten slutar fungera | självläkning utan omstart av Paperless, verifierad |

Det lokala lagringsbehovet är därmed frikopplat från arkivets storlek: samlingen får växa, disken behöver inte göra det.

## Så ställer du in det

Den kompletta konfigurationen finns som mall på GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). På servern:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # körs en gång, förbereder värden (enda steget som kräver root)
./wizard.sh          # guidad: välj leverantör, inloggningsuppgifter, runtest
```

Guiden frågar efter molntjänsten (Proton, S3, Backblaze B2, WebDAV, SFTP eller ”Inte i listan” för alla andra Rclone-tjänster), testar anslutningen med ett verkligt upp- och nedladdningstest och startar lagringscontainern. Därefter:

- **Nyinstallation:** `docker compose -f paperless.yml up -d`, klart.
- **Befintlig Paperless-instans:** Databas och inställningar förblir orörda; instruktionen [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) beskriver uppladdningen av befintliga dokument och den nödvändiga ändringen i din Compose-fil.

Jag har medvetet avstått från ett webbgränssnitt. Rclones webb-GUI användes först, men SSH-tunnlar, CORS och tillfälliga mountar gjorde det värre än kommandoraden som det skulle ersätta. Tre frågor i terminalen går snabbare.

## Så förblir mounten stabil i vardagen

Mallen hanterar fyra punkter som du också måste ta hänsyn till vid en egen lösning:

1. **`propagation: rslave`** på Paperless-containerns media-bind-mount, annars överlever containern inte en omstart av mounten. Detaljer och AppArmor-fällan bakom detta: [Rclone-mount i Docker-container](/blog/rclone-mount-in-docker-container).
2. **Stoppa Paperless när mounten saknas.** Annars skriver den dokument till en tom lokal katalog, och den återvändande mounten döljer dem osynligt. Ett watchdog-skript medföljer mallen.
3. **Ett konto som kan logga in utan tillsyn.** För Proton innebär det att TOTP-nyckeln sparas i Rclone-konfigurationen. Varför detta inte urholkar tvåfaktorsautentiseringen och hur Proton fungerar under Linux i stort: [Proton Drive under Linux](/blog/proton-drive-linux-status).
4. **Stäng av schemalagda uppgifter som läser allt** (`PAPERLESS_SANITY_TASK_CRON=disable`), eftersom integritetskontrollen annars regelbundet läser hela beståndet från molnet.

## Vad du bör väga in före användning

Ett nyinskannat dokument finns bara i den lokala cachen under några sekunder tills uppladdningen är klar. Om maskinen dör just under detta fönster saknas filen. Cachegränsen är mjuk och kan tillfälligt överskridas betydligt vid åtkomsttoppar. Rclones Proton-backend är dessutom officiellt Beta; vid snabba API-anrop visade den tecken på begränsning. Eftersom långtidsdata från kontinuerlig drift fortfarande saknas är mallen märkt som experimentell.

Hur mätvärdena togs fram, vilka fel som simulerades och hur en sådan lösning över huvud taget kan testas seriöst står i metodartikeln: [Testa molnmountar med genererade PDF:er](/blog/cloud-mount-testen-dummy-pdfs).

## Slutsats

Paperless-ngx på en liten disk med molnlagring är genomförbart och praktiskt i vardagen: knappt två sekunder vid första öppningen, därefter cachehastighet, sökning och gränssnitt förblir oberoende av molnet, och lösningen läker sig själv efter fel. Om du bara vill spara några gigabyte på en normalt dimensionerad server bör du dock räkna efter: i mitt fall tog hela lagringen upp 71 MB, operativsystemet flera gigabyte. Vinsten ligger inte i det omedelbart sparade utrymmet, utan i att beståndet kan växa utan att disken behöver växa med det.

## Källor

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): mallen från den här artikeln: setup.sh, wizard.sh, Compose-filer, watchdog och retrofit-instruktion.

2.  [Rclone: Översikt över molnlagringssystem](https://rclone.org/overview/): de över 70 tjänsterna som stöds och en jämförelse av deras funktioner.

3.  [Paperless-ngx: Konfiguration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` och övriga använda inställningar.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): sanity-kontroll, export och import samt de schemalagda bakgrundsuppgifterna.
