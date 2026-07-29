---
title: "Testa Cold Storage med Rclone: en praktisk testplan"
navTitle: "Testa Rclone"
description: "Innan en tjänst läser sina filer från molnet via en Rclone-mount bör du kontrollera mer än katalogåtkomst. Den här testplanen omfattar Cold Reads, Warm Reads, skrivoperationer, cachebeteende, filintegritet och fel."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min lästid"
themen:
  - "rclone"
related:
  - "rclone-mount-in-docker-container"
  - "paperless-dokumente-clouddienst-auslagern"
slug: "testa-cold-storage-med-rclone-en-praktisk-testplan"
translationOf: "cloud-mount-testen-dummy-pdfs"
url: "https://rafaelpfister.ch/sv/blog/testa-cold-storage-med-rclone-en-praktisk-testplan"
---

En Rclone-mount är snabbt konfigurerad. Fjärrlagringen visas som en katalog, `ls` visar filer och det första funktionstestet är godkänt. Det säger dock inte mycket om produktionsdrift.

Så snart en tjänst använder mounten tillkommer fler frågor: Hur lång tid tar den första åtkomsten till en fil? Vilka åtkomster hanteras av den lokala cachen? Vad händer med en fil som ännu inte har laddats upp om Rclone kraschar? Ser en redan körande container den återupprättade mounten? Och hur reagerar tjänsten om molnet tillfälligt inte går att nå?

Den här artikeln ger dig en allmän testplan för detta. Du kan använda den för ett dokumentarkiv, en mediaserver, en fotohanterare eller vilken annan tjänst som helst som hämtar sällan använda filer via Rclone från Cold Storage.

## Bestäm först vad du vill uppnå

Cold Storage betyder inte automatiskt samma sak för varje applikation. En mediaserver läser vanligtvis stora filer sekventiellt. En fotohanterare laddar många små förhandsvisningsdata och hoppar mellan olika positioner. Ett dokumentarkiv öppnar jämförelsevis små filer, men ofta bara en gång.

Anteckna de viktigaste egenskaperna hos ditt verkliga bestånd före testet:

- typisk filstorlek och den största förekommande filen
- antal filer per katalog
- fullständig läsning eller slumpmässiga åtkomster till enskilda områden
- förhållandet mellan läs- och skrivåtkomster
- antal samtidiga användare eller processer
- ändringar som sker direkt i fjärrlagringen utanför mounten
- acceptabel väntetid för en Cold Read
- maximalt tillgängligt utrymme för den lokala cachen

Först då får du meningsfulla framgångskriterier. Att öppna en enskild fil på 1,2 sekunder kan vara helt acceptabelt för ett arkiv och oanvändbart för en interaktiv applikation.

## Skapa ett reproducerbart testbestånd

Rclone har redan ett lämpligt verktyg för detta. `rclone test makefiles` skapar samma filträd varje gång med ett fast seed:

```bash
rclone test makefiles ./testdata \
  --seed 42 \
  --files 250 \
  --files-per-directory 25 \
  --min-file-size 16K \
  --max-file-size 32M
```

Anpassa antal och storlekar till ditt verkliga bestånd. Testa inte bara genomsnittliga filer. Några mycket små filer visar hur kostsamma metadataåtkomster är; några stora filer synliggör genomströmning, Read-Ahead och cachebeteende.

Lägg dessutom till filnamn som kan orsaka problem i praktiken:

```bash
mkdir -p "testdata/Specialfall/Underkatalog"
printf 'Mellanslag\n' > "testdata/Specialfall/Fil med mellanslag.txt"
printf 'Umlauter\n' > "testdata/Specialfall/Storlek och ändring.txt"
printf 'Versaler\n' > "testdata/Specialfall/Test.txt"
printf 'Gemener\n' > "testdata/Specialfall/test.txt"
```

Det sista testet är särskilt viktigt om det lokala filsystemet och molnbackend hanterar versaler och gemener olika.

Om din tjänst bara accepterar vissa format räcker det inte med godtyckliga binärfiler. Skapa då dessutom syntetiska filer i exakt dessa format. För Paperless-ngx var det PDF:er med ett riktigt textlager, så att testet inte av misstag mäter OCR-prestanda i stället för lagringssökvägen. I en fotohanterare bör beståndet innehålla olika bildstorlekar och format, och i en mediaserver korta filer med olika kodekar.

## En referensmätning utan mount

Innan FUSE och VFS-cache kommer in i bilden bör du mäta backend direkt. Kopiera beståndet till testfjärrlagringen med Rclone och spara en detaljerad logg:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

Kontrollera sedan att källa och mål stämmer överens:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

Med `--download` läser Rclone faktiskt data och jämför dem, även om backend inte tillhandahåller lämpliga hashar. Det tar längre tid, men ger en användbar utgångspunkt för det senare integritetstestet.

Dokumentera uppladdningstid, överföringshastighet, antal återförsök och API-fel. Om den direkta åtkomsten redan är instabil kan mounten inte reparera det.

## Separera test-mounten från produktionscachen

Använd en egen mountpunkt och en egen cachekatalog för mätningen:

```bash
rclone mount remote:cold-storage-test /mnt/rclone-test \
  --vfs-cache-mode full \
  --cache-dir /var/cache/rclone-test \
  --vfs-cache-max-size 10G \
  --vfs-cache-poll-interval 1m \
  --allow-other \
  --log-file /var/log/rclone-test.log \
  --log-level INFO
```

Värdena är ett exempel och ingen allmän rekommendation. Separationen är avgörande: En tom testcache gör Cold Reads reproducerbara utan att du behöver radera filer ur en aktiv produktionscache.

`--vfs-cache-mode full` är vanligtvis det mest informativa testläget för applikationer. Rclone buffrar då läs- och skrivåtkomster lokalt och kan bättre efterlikna filåtkomster som inte skulle vara möjliga med en ren objektlagring. Den extra kompatibiliteten kostar lokalt lagringsutrymme.

## Kontrollera alltid ur den verkliga tjänstens perspektiv

En mount kan fungera för din användare men ändå vara oanvändbar för tjänsten. Vanliga orsaker är ett annat användar-ID, saknat `--allow-other`, containergränser eller felaktig mount-propagation.

Utför därför minst en fullständig läsåtkomst med samma identitet som applikationen senare körs under:

```bash
sudo -u <tjänstanvändare> sha256sum /mnt/rclone-test/sökväg/till/fil
```

Om tjänsten körs i Docker ska testet utföras i containern:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /sökväg/i/containern/fil
```

Ännu bättre är ett riktigt applikationstest. Öppna filen via tjänstens webbgränssnitt eller API. Bara då märker du om applikationen exempelvis startar flera parallella läsningar, hoppar till slutet av filen eller förväntar sig ytterligare metadata.

## Mät Cold Reads och Warm Reads separat

Med `--vfs-cache-mode full` finns tre nivåer mellan applikationen och molnet:

| Nivå | Vad som finns där |
|---|---|
| Fjärrlagring | den fullständiga filen i molntjänsten |
| VFS-cache | lokalt sparade områden av redan lästa filer |
| Linux Page Cache | nyligen använda data i RAM |

För en Cold Read väljer du en fil vars innehåll aldrig tidigare har lästs via test-mounten. Vid den direkt efterföljande Warm Read ligger den i VFS-cachen och oftast även i RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/stor-fil.bin" "Cold Read"
measure_read "/mnt/rclone-test/stor-fil.bin" "Warm Read"
```

Mät inte bara en fil. Använd minst tio tidigare olästa filer av olika storlek och anteckna medianen, det långsammaste värdet och filstorleken. Ett enskilt toppresultat är inget beslutsunderlag.

En Warm Read är inte ett rent disktest, eftersom kärnan kan hålla delar av filen i RAM. För de flesta Cold Storage-scenarier är det inget problem. Det avgörande är vad en användare upplever vid första och upprepade öppningar. Om du vill bedöma RAM och lokal disk separat måste du dessutom kontrollera och bevisligen rensa Page Cache.

## Testa inte bara fullständiga läsåtkomster

`cat` läser en fil från början till slut. Många applikationer beter sig annorlunda:

- En videospelare läser först header och index, hoppar senare till en annan position och fortsätter sedan läsa sekventiellt.
- En bildhanterare läser metadata och skapar därefter en förhandsvisningsbild.
- Ett arkivprogram kan först läsa slutet av filen.
- Flera workers kan samtidigt komma åt olika filer.

Testa dessa förlopp med den faktiska applikationen. Övervaka Rclone-loggen och cachen parallellt. För stora filer är det intressant hur mycket Rclone faktiskt lagrar lokalt och om `--vfs-read-ahead` passar åtkomstmönstret.

En Rclone-mount är dessutom ingen lämplig lagringsplats för databaser eller andra filer som behöver tillförlitlig låsning och frekventa ändringar inom samma fil. VFS-lagret utjämnar skillnader mellan filsystem och objektlagring, men gör inte backend till ett lokalt filsystem.

## Godkänn skrivsökvägen separat

Om din tjänst bara läser ska du om möjligt montera fjärrlagringen skrivskyddad. Måste den skriva, testa skapa, skriva över, byta namn och radera var för sig.

En skriven fil visas inte nödvändigtvis direkt i fjärrlagringen. Med aktiv VFS-cache börjar uppladdningen först efter att filen har stängts och `--vfs-write-back` har löpt ut. Kontrollera därför båda tillstånden:

1. Applikationen har stängt filen utan fel.
2. Filen kan därefter läsas i fjärrlagringen via direkt Rclone-åtkomst.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Efter att --vfs-write-back har löpt ut:
rclone cat remote:cold-storage-test/writeback-test.txt
```

Upprepa testet med en stor fil och avsluta Rclone medan uppladdningen fortfarande pågår. Starta sedan om med samma cachekatalog och kontrollera om uppladdningen fortsätter. Just detta tidsfönster avgör hur mycket data som riskeras vid ett serverfel.

Testa även namnbyte och radering. Många molnbackend hanterar dessa operationer annorlunda än ett lokalt filsystem. Det relevanta är inte bara om kommandot avslutas framgångsrikt, utan när ändringen syns vid direkt åtkomst till fjärrlagringen och för andra klienter.

## Testa ändringar utanför mounten

Filer kan ändras via leverantörens webbgränssnitt, en andra Rclone-process eller en annan server. Mounten ser inte alltid sådana ändringar direkt, eftersom kataloginformation cachelagras.

Skapa därför en fil direkt i fjärrlagringen med ett andra Rclone-anrop:

```bash
printf 'extern-ändring\n' > extern-andring.txt
rclone copyto extern-andring.txt \
  remote:cold-storage-test/extern-andring.txt
```

Mät när filen visas i mounten. Upprepa testet för ändring och radering. Resultatet beror på backend, dess stöd för polling samt `--poll-interval` och `--dir-cache-time`. Om applikationen måste se aktuella ändringar direkt ska detta beteende uttryckligen ingå i godkännandekriterierna.

Med aktiverat Remote Control-gränssnitt kan du specifikt tömma katalogcachen:

```bash
rclone rc vfs/forget
```

Det är användbart för ett manuellt test, men ersätter inte en lämplig driftstrategi.

## Sätt cachen under tryck

En nästan tom cache är det enklaste fallet. Sätt `--vfs-cache-max-size` medvetet litet i en andra testomgång och läs mer data än vad som ryms.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

De båda storlekarna kan skilja sig kraftigt. I Full-läge använder Rclone sparse files: En fil visar sin fullständiga logiska storlek, trots att endast de lästa områdena upptar lokalt utrymme.

Cachegränsen är dessutom mjuk. Rclone kontrollerar den i intervall enligt `--vfs-cache-poll-interval`, och öppna filer kan inte tas bort. Cachen kan därför tillfälligt överskrida gränsen. Efter att filerna stängts och nästa städkörning bör den dock minska igen.

Dokumentera toppvärdet, värdet efter rensningen och tiden som krävs. På så sätt kan nödvändigt lokalt lagringsutrymme dimensioneras på ett rimligt sätt.

## Simulera två olika fel

Ett moln som inte går att nå och en kraschad Rclone-process är två olika fel:

| Fel | Vad du kontrollerar |
|---|---|
| Backend eller nätverk går inte att nå, Rclone fortsätter köra | Beteende vid återförsök, timeouts och redan cachelagrade filer |
| Rclone-processen avslutas | Beteende för FUSE-mounten och återställning av mountpunkten |

Simulera båda endast i testmiljön. Du kan tvångsavsluta en Rclone-container för det andra fallet:

```bash
docker kill --signal KILL <rclone-container>
```

Kontrollera applikationen under felet, inte bara mountpunkten:

- Vilka funktioner förblir tillgängliga?
- Hur länge väntar en åtkomst innan ett fel visas?
- Går redan helt cachelagrade filer fortfarande att nå?
- Stoppar applikationen nya skrivoperationer?
- Uppstår ett begripligt felmeddelande eller bara en hängande process?
- Utlöses din övervakning?

En skrivtjänst får inte obemärkt skriva till den underliggande lokala katalogen när mounten saknas. När mounten återkommer skulle dessa filer döljas. Ett enkelt skydd före varje skrivjobb är:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

Efter omstart av Rclone kontrollerar du mounten på värden och från varje konsumerande container. En återupprättad mount når en redan körande container endast med korrekt mount-propagation. För Docker behövs vanligtvis `rslave` på den konsumerande sidan. Detaljerna finns i artikeln [Driv Rclone-mountar tillförlitligt i Docker](/blog/rclone-mount-in-docker-container).

## Ett konkret exempel från Paperless-ngx

För mitt Paperless-test skapade jag 40 PDF:er på sammanlagt 13,9 MB. Ett tidigare oöppnat dokument behövde cirka 1,8 sekunder, medan en direkt upprepad åtkomst tog 19 till 24 millisekunder. En VFS-cache begränsad till 4 MB steg tillfälligt till 12,7 MiB och rensades igen vid nästa körning.

Medan fjärrlagringen inte gick att nå fungerade dokumentlistan, fulltextsökningen och förhandsvisningsbilderna fortfarande, eftersom dessa data fanns lokalt. Endast originalet gick inte att öppna. Efter att mounten återupprättats kunde den körande Paperless-containern åter komma åt filerna utan att själv startas om.

Dessa siffror är inget benchmark för Rclone eller Proton Drive. Intressant är beteendet: Hot Storage förblev lokalt tillgängligt, Cold Reads var långsammare men förutsägbara, och tjänsten återhämtade sig efter felet.

## Vad som ska ingå i testprotokollet

Ett resultat som kan följas upp senare innehåller minst:

- Rclone-version och använd backend
- operativsystem, FUSE-variant och filsystem för cachekatalogen
- fullständigt mount-kommando utan inloggningsuppgifter
- antal, storleksfördelning och struktur för testfilerna
- Cold Read- och Warm Read-värden för flera filer
- skrivtid tills synlighet i fjärrlagringen
- cachetoppvärde och rensningens varaktighet
- resultat av `rclone check --download`
- beteende vid backendfel och avslutad Rclone-process
- återställningstid ur applikationens perspektiv
- återförsök, timeouts, begränsningar och autentiseringsfel från loggen

Definiera ett gränsvärde i förväg för varje punkt. Då avslutas testet med ett beslut och inte bara med en samling intressanta siffror.

## När lösningen är redo

En Cold Storage-mount är redo att användas när du kan svara ja på dessa frågor:

- Är Cold Reads tillräckligt snabba för den avsedda tjänsten?
- Snabbar cachen upp upprepade åtkomster som förväntat?
- Förblir det lokala utrymmesbehovet kontrollerbart även under belastning?
- Stämmer alla filer efter en fullständig nedladdning?
- Fungerar alla nödvändiga filoperationer med vald backend?
- Beter sig applikationen kontrollerat vid molnavbrott?
- Stoppas skrivoperationer säkert när mounten saknas?
- När en återupprättad mount alla körande konsumenter?
- Visar övervakningen felet innan en användare rapporterar det?

Om ett svar saknas vet du åtminstone exakt vad du behöver fortsätta arbeta med. Det är betydligt mer användbart än en mount som såg bra ut vid första `ls` och först visar sina begränsningar i drift.

## Källor

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducerbara testfiler och katalogstrukturer med konfigurerbara storlekar.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachelägen, Writeback, sparse files, cachegränser och katalogcache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): jämförelse av källa och mål, inklusive fullständig kontroll med `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): specifik tömning av VFS-katalogcachen med `vfs/forget`.
