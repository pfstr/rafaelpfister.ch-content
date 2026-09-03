---
title: "Testa Cold Storage med Rclone: en praktisk testplan"
navTitle: "Testa Rclone"
description: "Innan en tjänst läser sina filer från molnet via en Rclone-mount bör du kontrollera mer än katalogåtkomst. Den här testplanen omfattar kalla läsningar, varma läsningar, skrivoperationer, cachebeteende, filintegritet och fel."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min läsning"
themen:
  - rclone
  - testing
related:
  - rclone-mount-in-docker-container
  - paperless-dokumente-clouddienst-auslagern
slug: "testa-cold-storage-med-rclone-en-praktisk-testplan"
translationOf: "cloud-mount-testen-dummy-pdfs"
translationId: article-8592f808b2e93cd4
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:22:48.161Z
translationReview: required
translationSourceHash: 27bc45a50d8e84fc785eaf04ec6814054e327f516587d0f9f9a101c989ce2aa1
url: https://rafaelpfister.ch/sv/blog/testa-cold-storage-med-rclone-en-praktisk-testplan
---

En Rclone-mount är snabbt konfigurerad. Fjärrlagringen visas som en katalog, `ls` visar filer och det första funktionstestet är godkänt. För drift i produktion säger det dock fortfarande mycket lite.

Så snart en tjänst använder mounten tillkommer fler frågor: Hur lång tid tar den första åtkomsten till en fil? Vilka åtkomster hanteras av den lokala cachen? Vad händer med en fil som ännu inte har laddats upp om Rclone kraschar? Ser en container som körs den återupprättade mounten igen? Och hur reagerar tjänsten om molnet tillfälligt inte går att nå?

Den här artikeln ger en generell testplan för detta. Du kan använda den för ett dokumentarkiv, en mediaserver, en fotohantering eller någon annan tjänst som hämtar sällan använda filer via Rclone från Cold Storage.

## De viktigaste alternativen i rclone

Som orientering följer här Rclone-alternativen som förekommer i denna testplan, fritt översatta från dokumentationen:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Alternativ | Betydelse |
|---|---|
| `--seed n` | Startvärde för slumptalsgeneratorn vid `rclone test makefiles`; samma seed ger samma filträd |
| `--files n` | Antal testfiler som ska skapas |
| `--files-per-directory n` | Genomsnittligt antal filer per katalog |
| `--min-file-size grösse` | Minsta skapade filstorlek (suffix som K, M, G tillåts) |
| `--max-file-size grösse` | Största skapade filstorlek |
| `--progress` | Löpande förloppsindikering under överföringen |
| `--stats dauer` | Intervall för utskrift av överföringsstatistik |
| `--log-file datei` | Skriver loggen till den angivna filen |
| `--log-level stufe` | Loggens detaljnivå: DEBUG, INFO, NOTICE eller ERROR |
| `--one-way` | Kontrollerar vid `rclone check` endast om alla källfiler finns i målet och är identiska; extra filer i målet räknas inte som fel |
| `--download` | Hämtar faktiskt data vid jämförelsen i stället för att bara jämföra hashar |
| `--vfs-cache-mode modus` | Cachestrategi för VFS-lagret; `full` buffrar läs- och skrivåtkomster lokalt |
| `--cache-dir verzeichnis` | Katalog för den lokala cachen |
| `--vfs-cache-max-size grösse` | Mjuk gräns för VFS-cachens totala storlek |
| `--vfs-cache-poll-interval dauer` | Intervall då Rclone kontrollerar och rensar cachen |
| `--vfs-write-back dauer` | Väntetid efter att en fil stängts innan uppladdningen till fjärrlagringen startar |
| `--vfs-read-ahead grösse` | Ytterligare datamängd som läses i förväg bortom den begärda positionen vid `full` |
| `--poll-interval dauer` | Intervall då Rclone frågar fjärrlagringen efter ändringar (endast för backends med stöd för polling) |
| `--dir-cache-time dauer` | Giltighetstid för cachade kataloglistor |
| `--allow-other` | Tillåter andra användare än den som monterar att komma åt FUSE-mounten |

</details>

De fullständiga listorna finns i Rclone-dokumentationen, särskilt under [rclone mount](https://rclone.org/commands/rclone_mount/) och i översikten över [globala flaggor](https://rclone.org/flags/).

## Fastställ först vad du vill uppnå

Cold Storage betyder inte automatiskt samma sak för varje tillämpning. En mediaserver läser oftast stora filer sekventiellt. En fotohantering laddar många små förhandsvisningsdata och hoppar mellan olika positioner. Ett dokumentarkiv öppnar relativt små filer, men ofta bara en gång.

Notera de viktigaste egenskaperna hos ditt faktiska bestånd före testet:

- typisk filstorlek samt största förekommande fil
- antal filer per katalog
- fullständig läsning eller slumpmässiga åtkomster till enskilda områden
- förhållandet mellan läs- och skrivåtkomster
- antal samtidiga användare eller processer
- ändringar som sker direkt i fjärrlagringen utanför mounten
- acceptabel väntetid för en kall läsning
- maximalt tillgängligt utrymme för den lokala cachen

Först då uppstår meningsfulla framgångskriterier. Att öppna en enskild fil på 1.2 sekunder kan vara helt acceptabelt för ett arkiv men oanvändbart för en interaktiv tillämpning.

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

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Målmapp där testträdet skapas |
| `--seed 42` | Fast startvärde för slumptalsgeneratorn; varje körning skapar samma bestånd |
| `--files 250` | Totalt 250 testfiler |
| `--files-per-directory 25` | I genomsnitt 25 filer per katalog |
| `--min-file-size 16K` | Minsta fil 16 KiB |
| `--max-file-size 32M` | Största fil 32 MiB |

</details>

Anpassa antal och storlekar till ditt verkliga bestånd. Testa inte bara genomsnittsfiler. Några mycket små filer visar hur kostsamma metadataåtkomster är; några stora filer synliggör genomströmning, read-ahead och cachebeteende.

Lägg dessutom till filnamn som kan orsaka problem i praktiken:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `mkdir -p` | Skapar även saknade överordnade kataloger och ger inget fel om katalogen redan finns |
| `printf '…\n' > datei` | Skriver den angivna texten som filinnehåll; omdirigeringen skapar filen med det problematiska namnet |

</details>

Det sista testet är särskilt viktigt om det lokala filsystemet och molnbackendet hanterar stora och små bokstäver olika.

Om din tjänst endast accepterar vissa format räcker inte godtyckliga binärfiler. Skapa då även syntetiska filer i just dessa format. För Paperless-ngx var det PDF:er med ett riktigt textlager, så att testet inte av misstag mäter OCR-prestanda i stället för lagringsvägen. För en fotohantering bör beståndet innehålla olika bildstorlekar och format, för en mediaserver korta filer med olika codecs.

## En referensmätning utan mount

Innan FUSE och VFS-cachen kommer in i bilden bör du mäta backendet direkt. Kopiera beståndet med Rclone till testfjärrlagringen och spara en detaljerad logg:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Kopians källa: det lokalt skapade testbeståndet |
| `remote:cold-storage-test` | Mål: sökväg i den konfigurerade fjärrlagringen |
| `--progress` | Löpande förloppsindikering i terminalen |
| `--stats 5s` | Överföringsstatistik var femte sekund i stället för standardintervallet |
| `--log-file rclone-copy.log` | Fullständig logg i en fil för senare utvärdering |
| `--log-level INFO` | Loggar överföringar, försök igen och fel utan DEBUG-omfattningen |

</details>

Kontrollera sedan om källa och mål överensstämmer:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Referens: det lokala originalbeståndet |
| `remote:cold-storage-test` | Testobjekt: det nyuppladdade beståndet i fjärrlagringen |
| `--one-way` | Kontrollerar endast om alla källfiler finns i målet och är identiska; extra filer i målet anmärks inte |
| `--download` | Hämtar data och jämför innehållet i stället för att förlita sig på hashar |

</details>

`--download` är avgörande här eftersom vissa backends inte tillhandahåller lämpliga hashar. Jämförelsen tar längre tid, men ger en användbar utgångspunkt för det senare integritetstestet.

Dokumentera uppladdningstid, överföringshastighet, antal försök igen och API-fel. Om den direkta åtkomsten redan är instabil kan mounten inte reparera det.

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

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `remote:cold-storage-test` | Fjärrlagring med sökväg som ska monteras |
| `/mnt/rclone-test` | Mountpunkt på testsystemet |
| `--vfs-cache-mode full` | Buffrar läs- och skrivåtkomster helt i den lokala cachen |
| `--cache-dir /var/cache/rclone-test` | Egen cachekatalog, skild från produktionscachen |
| `--vfs-cache-max-size 10G` | Mjuk gräns på 10 GiB för VFS-cachen |
| `--vfs-cache-poll-interval 1m` | Cachekontroll och rensning varje minut |
| `--allow-other` | Även andra användare än den som monterar får åtkomst; krävs för tjänster och containrar |
| `--log-file /var/log/rclone-test.log` | Logg i en fil för att kunna följa åtkomster under testerna |
| `--log-level INFO` | Medelhög detaljnivå i loggen |

</details>

Värdena är ett exempel och ingen generell rekommendation. Det avgörande är separationen: en tom testcache gör kalla läsningar reproducerbara utan att du behöver radera filer från en aktiv produktionscache.

`--vfs-cache-mode full` är för tillämpningar oftast det mest informativa testläget. Rclone buffrar då läs- och skrivåtkomster lokalt och kan bättre efterlikna filåtkomster som inte skulle vara möjliga med ren objektlagring. Den extra kompatibiliteten kostar lokalt lagringsutrymme.

## Kontrollera alltid från den verkliga tjänstens perspektiv

En mount kan fungera för din användare men ändå vara obrukbar för tjänsten. Vanliga orsaker är ett annat användar-ID, saknat `--allow-other`, containergränser eller felaktig mount-propagation.

Utför därför minst en fullständig läsning med samma identitet som tillämpningen senare körs under:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-u <service-user>` | Kör kommandot som den angivna användaren, inte som root |
| `/mnt/rclone-test/pfad/zur/datei` | Fil som ska läsas; `sha256sum` tvingar fram en fullständig läsning |

</details>

Om tjänsten körs i Docker ska testet göras i containern:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--user <uid>:<gid>` | Kör kommandot i containern med detta användar- och grupp-ID, oberoende av imageens standardanvändare |
| `<app-container>` | Namn eller ID för den körande tillämpningscontainern |
| `sha256sum /pfad/im/container/datei` | Kommando som ska köras; sökvägen är mounten ur containerns perspektiv |

</details>

Ännu bättre är ett verkligt tillämpningstest. Öppna filen via tjänstens webbgränssnitt eller API. Endast då märker du om tillämpningen exempelvis startar flera parallella läsningar, hoppar till filslutet eller förväntar sig ytterligare metadata.

## Mät kalla och varma läsningar separat

Med `--vfs-cache-mode full` finns det tre nivåer mellan tillämpningen och molnet:

| Nivå | Vad som finns där |
|---|---|
| Fjärrlagring | hela filen i molntjänsten |
| VFS-cache | lokalt lagrade områden av redan lästa filer |
| Linux sidcache | nyligen använda data i RAM |

För en kall läsning väljer du en fil vars innehåll aldrig tidigare har lästs via test-mounten. Vid den direkt efterföljande varma läsningen finns den i VFS-cachen och oftast dessutom i RAM.

```bash
measure_read() {
  file="$1"
  label="$2"
  start=$(date +%s%3N)
  cat "$file" > /dev/null
  end=$(date +%s%3N)
  printf '%s: %s ms\n' "$label" "$((end - start))"
}

measure_read "/mnt/rclone-test/grosse-datei.bin" "Cold Read"
measure_read "/mnt/rclone-test/grosse-datei.bin" "Warm Read"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `date +%s%3N` | Tidsstämpel i millisekunder: Unix-sekunder, direkt följda av de första tre siffrorna i nanosekundsdelen (GNU date) |
| `cat "$file" > /dev/null` | Läser filen helt och förkastar utdata; endast lästiden mäts |
| `"$1"`, `"$2"` | Argument till skalfunktionen: filsökväg och etikett för mätraden |

</details>

Mät inte bara en fil. Använd minst tio tidigare olästa filer i olika storlekar och notera medianen, det långsammaste värdet och filstorleken. Ett enskilt bästa värde är inget beslutsunderlag.

En varm läsning är inte ett rent disktest eftersom kärnan kan behålla delar av filen i RAM. För de flesta Cold Storage-scenarier är detta inget problem. Det avgörande är vad en användare upplever vid första och upprepade öppningar. Om du vill bedöma RAM och lokal disk separat måste du dessutom kontrollera och bevisligen tömma sidcachen.

## Testa inte bara fullständiga läsningar

`cat` läser en fil från början till slut. Många tillämpningar beter sig annorlunda:

- En videospelare läser först headers och index, hoppar senare till en annan position och fortsätter sedan läsa sekventiellt.
- En bildhantering läser metadata och skapar därefter en förhandsvisningsbild.
- Ett arkivprogram kan först läsa slutet av filen.
- Flera arbetare kan samtidigt komma åt olika filer.

Testa dessa flöden med den faktiska tillämpningen. Övervaka samtidigt Rclone-loggen och cachen. För stora filer är det intressant hur mycket Rclone faktiskt lagrar lokalt och om `--vfs-read-ahead` passar åtkomstmönstret.

En Rclone-mount är dessutom ingen lämplig lagringsplats för databaser eller andra filer som kräver tillförlitlig låsning och frekventa ändringar inom samma fil. VFS-lagret kompenserar för skillnader mellan filsystem och objektlagring, men gör inte backendet till ett lokalt filsystem.

## Godkänn skrivvägen separat

Om din tjänst endast läser bör du om möjligt montera fjärrlagringen skrivskyddat. Om den måste skriva ska du testa skapande, överskrivning, namnbyte och radering var för sig.

En skriven fil syns inte nödvändigtvis omedelbart i fjärrlagringen. Med aktiv VFS-cache startar uppladdningen först efter att filen stängts och `--vfs-write-back` har löpt ut. Kontrollera därför båda tillstånden:

1. Tillämpningen har stängt filen utan fel.
2. Filen kan därefter läsas i fjärrlagringen genom direkt Rclone-åtkomst.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Efter att --vfs-write-back har löpt ut:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Målfil i mounten; omdirigeringen skriver via VFS-cachen |
| `remote:cold-storage-test/writeback-test.txt` | Direktåtkomst förbi mounten: `rclone cat` läser filen från fjärrlagringen och skriver den till stdout |

</details>

Upprepa testet med en stor fil och avsluta Rclone medan uppladdningen fortfarande pågår. Starta sedan om med samma cachekatalog och kontrollera om uppladdningen fortsätter. Just detta tidsfönster avgör hur mycket data som riskeras vid ett serverfel.

Testa även namnbyte och radering. Många molnbackends hanterar dessa operationer annorlunda än ett lokalt filsystem. Det relevanta är inte bara om kommandot avslutas korrekt, utan när ändringen blir synlig vid direktåtkomst till fjärrlagringen och för andra klienter.

## Testa ändringar utanför mounten

Filer kan ändras via leverantörens webbgränssnitt, en andra Rclone-process eller en annan server. Mounten ser inte alltid sådana ändringar omedelbart eftersom kataloginformation är cachad.

Skapa därför en fil direkt i fjärrlagringen med ett andra Rclone-anrop:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `external-change.txt` | Källa: den lokalt skapade filen |
| `remote:cold-storage-test/external-change.txt` | Mål med exakt filnamn; `copyto` kopierar en enskild fil under exakt detta namn, i stället för att som `copy` kopiera till en katalog |

</details>

Mät när filen visas i mounten. Upprepa testet för ändring och radering. Resultatet beror på backendet, dess stöd för polling samt `--poll-interval` och `--dir-cache-time`. Om tillämpningen måste se aktuella ändringar omedelbart ska detta beteende uttryckligen ingå i godkännandekriterierna.

Med aktiverat Remote Control-gränssnitt kan du tömma katalogcachen specifikt:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `vfs/forget` | Remote Control-kommando som ska köras: tömmer VFS:ens cachade katalogträd så att nästa åtkomst frågar fjärrlagringen på nytt |

</details>

Det är användbart för ett manuellt test, men ersätter inte en lämplig driftstrategi.

## Sätt cachen under press

En nästan tom cache är det enklaste fallet. Sätt `--vfs-cache-max-size` medvetet lågt i en andra testomgång och läs mer data än vad som ryms.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `du -s` | Sammanfattar diskutrymmesanvändningen till en summarad i stället för att lista varje underkatalog |
| `du -h` | Utdata i läsbara enheter (K, M, G) |
| `du --apparent-size` | Visar den logiska filstorleken i stället för faktiskt upptaget diskutrymme |
| `find … -type f` | Tar endast hänsyn till vanliga filer, inte kataloger |
| `wc -l` | Räknar raderna i utdata, alltså här antalet cachefiler |

</details>

De två storlekarna kan skilja sig kraftigt. I full-läget använder Rclone glesa filer: en fil visar sin fullständiga logiska storlek trots att endast lästa områden upptar lokalt utrymme.

Cachegränsen är dessutom mjuk. Rclone kontrollerar den i takt med `--vfs-cache-poll-interval`, och öppna filer kan inte tas bort. Cachen kan därför tillfälligt överskrida gränsen. Efter att filerna stängts och nästa rensningskörning genomförts bör den dock minska igen.

Logga toppvärdet, värdet efter rensningen och tiden det tar. På så sätt kan det nödvändiga lokala lagringsutrymmet dimensioneras rimligt.

## Simulera två olika fel

Ett moln som inte går att nå och en kraschad Rclone-process är två olika fel:

| Fel | Vad du kontrollerar |
|---|---|
| Backend eller nätverk kan inte nås, Rclone fortsätter köra | Beteende vid försök igen, timeouts och redan cachade filer |
| Rclone-processen avslutas | FUSE-mountens beteende och återställning av mountpunkten |

Simulera båda endast i testmiljön. Du kan tvångsavsluta en Rclone-container i det andra fallet:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--signal KILL` | Skickar SIGKILL i stället för standardsignalen SIGTERM; processen får ingen möjlighet att städa upp |
| `<rclone-container>` | Namn eller ID för Rclone-containern |

</details>

Kontrollera tillämpningen under felet, inte bara mountpunkten:

- Vilka funktioner förblir tillgängliga?
- Hur länge väntar en åtkomst innan ett fel visas?
- Kan redan helt cachade filer fortfarande nås?
- Stoppar tillämpningen nya skrivoperationer?
- Uppstår ett begripligt felmeddelande eller bara en hängande process?
- Utlöses din övervakning?

En skrivtjänst får inte obemärkt skriva till den underliggande lokala katalogen när mounten saknas. När mounten återkommer skulle dessa filer döljas. Ett enkelt skydd före varje skrivjobb är:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-q` | Ingen utdata; resultatet finns endast i exitkoden |
| `/mnt/rclone-test` | Sökväg som ska kontrolleras; exitkod 0 endast om en mount faktiskt är aktiv där |
| `\|\| exit 1` | Avbryter skriptet om sökvägen inte är en mountpunkt |

</details>

Efter omstart av Rclone kontrollerar du mounten på värden och från varje konsumerande container. En återupprättad mount når en redan körande container endast med korrekt mount-propagation. För Docker krävs oftast `rslave` på den konsumerande sidan. Detaljerna finns i artikeln [Driv Rclone-mountar i Docker på ett tillförlitligt sätt](/blog/rclone-mount-in-docker-container).

## Ett konkret exempel från Paperless-ngx

För mitt Paperless-test skapade jag 40 PDF:er på totalt 13.9 MB. Ett tidigare oöppnat dokument tog ungefär 1.8 sekunder, medan en direkt upprepad åtkomst tog 19 till 24 millisekunder. En VFS-cache begränsad till 4 MB steg tillfälligt till 12.7 MiB och rensades igen vid nästa körning.

Medan fjärrlagringen inte gick att nå fortsatte dokumentlistan, fulltextsökningen och förhandsvisningsbilderna att fungera eftersom dessa data fanns lokalt. Endast originalet kunde inte öppnas. Efter att mounten återupprättats kunde den körande Paperless-containern åter komma åt filerna utan att själv behöva startas om.

Dessa siffror är inget riktmärke för Rclone eller Proton Drive. Det intressanta är beteendet: Hot Storage förblev tillgängligt lokalt, kalla läsningar var långsammare men förutsägbara, och tjänsten återhämtade sig efter felet.

## Vad som bör ingå i testprotokollet

Ett resultat som går att följa upp senare innehåller minst:

- Rclone-version och använt backend
- operativsystem, FUSE-variant och filsystem för cachekatalogen
- fullständigt mount-kommando utan inloggningsuppgifter
- antal, storleksfördelning och struktur för testfilerna
- värden för kalla och varma läsningar för flera filer
- skrivtid tills synlighet i fjärrlagringen
- cachetoppvärde och rensningens varaktighet
- resultat av `rclone check --download`
- beteende vid backendfel och avslutad Rclone-process
- återställningstid ur tillämpningens perspektiv
- försök igen, timeouts, begränsningar och autentiseringsfel från loggen

Definiera ett gränsvärde för varje punkt i förväg. Då avslutas testet med ett beslut och inte bara en samling intressanta siffror.

## När lösningen är klar

En Cold Storage-mount är klar att tas i bruk om du kan svara ja på dessa frågor:

- Är kalla läsningar tillräckligt snabba för den avsedda tjänsten?
- Snabbar cachen upp upprepade åtkomster som förväntat?
- Förblir det lokala utrymmesbehovet hanterbart även under belastning?
- Stämmer alla filer efter en fullständig nedladdning?
- Fungerar alla nödvändiga filoperationer med det valda backendet?
- Beter sig tillämpningen kontrollerat vid ett molnfel?
- Stoppas skrivoperationer säkert när mounten saknas?
- Når en återupprättad mount alla körande konsumenter?
- Visar övervakningen felet innan en användare rapporterar det?

Om ett svar saknas vet du åtminstone exakt vad du behöver fortsätta arbeta med. Det är betydligt mer hjälpsamt än en mount som såg bra ut vid första `ls` men visar sina begränsningar först i drift.

## Källor

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducerbara testfiler och katalogstrukturer med konfigurerbara storlekar.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachelägen, writeback, glesa filer, cachegränser och katalogcache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): jämförelse av källa och mål, inklusive fullständig kontroll med `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): specifik tömning av VFS-katalogcachen med `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): fullständig referens för de globala alternativen, inklusive loggning, statistik och VFS-parametrar.
