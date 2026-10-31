---
title: "Testa Cold Storage med Rclone: en praktisk testplan"
navTitle: "Testa Rclone"
description: "Innan en tjänst läser sina filer från molnet via en Rclone-mount bör du kontrollera mer än katalogåtkomsten. Den här testplanen omfattar Cold Reads, Warm Reads, skrivoperationer, cachebeteende, filintegritet och avbrott."
date: "2026-07-26"
kategorie: "Rclone"
timeToRead: "11 min lästid"
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
translatedAt: 2026-10-31T03:28:13.388Z
translationReview: automatic
translationSourceHash: 820965b014837c0d0b5b5bf629a02842f2ebbbb26f62d343d8f1630278cd3bec
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/testa-cold-storage-med-rclone-en-praktisk-testplan
---

En Rclone-mount är snabbt konfigurerad. Remoten visas som en katalog, `ls` visar filer och det första funktionstestet är godkänt. För produktionsdrift säger det dock fortfarande väldigt lite.

Så snart en tjänst använder mounten uppstår fler frågor: Hur lång tid tar den första åtkomsten till en fil? Vilka åtkomster hanteras av den lokala cachen? Vad händer med en fil som ännu inte har laddats upp om Rclone kraschar? Ser en körande container den återupprättade mounten igen? Och hur reagerar tjänsten om molnet tillfälligt inte kan nås?

Den här artikeln ger en generell testplan för detta. Du kan använda den för ett dokumentarkiv, en mediaserver, en fotohantering eller vilken annan tjänst som helst som hämtar sällan använda filer via Rclone från Cold Storage.

## De viktigaste alternativen i rclone

Som orientering följer här Rclone-alternativen som förekommer i denna testplan, fritt översatta från dokumentationen:

<details class="options-details">
<summary>Översikt över alternativen</summary>

| Alternativ | Betydelse |
|---|---|
| `--seed n` | Startvärde för slumpgeneratorn i `rclone test makefiles`; samma seed ger samma filträd |
| `--files n` | Antal testfiler som ska skapas |
| `--files-per-directory n` | Genomsnittligt antal filer per katalog |
| `--min-file-size grösse` | Minsta skapade filstorlek (suffix som K, M och G tillåts) |
| `--max-file-size grösse` | Största skapade filstorlek |
| `--progress` | Löpande förloppsvisning under överföringen |
| `--stats dauer` | Intervall för utskrift av överföringsstatistik |
| `--log-file datei` | Skriver loggen till den angivna filen |
| `--log-level stufe` | Loggens detaljnivå: DEBUG, INFO, NOTICE eller ERROR |
| `--one-way` | Kontrollerar med `rclone check` endast om alla källfiler finns i målet och är identiska; extra filer i målet räknas inte som fel |
| `--download` | Laddar faktiskt ned data vid jämförelsen i stället för att bara jämföra hashvärden |
| `--vfs-cache-mode modus` | Cachestrategi för VFS-lagret; `full` buffrar läs- och skrivåtkomster lokalt |
| `--cache-dir verzeichnis` | Katalog för den lokala cachen |
| `--vfs-cache-max-size grösse` | Mjuk gräns för VFS-cachens totala storlek |
| `--vfs-cache-poll-interval dauer` | Intervall då Rclone kontrollerar och rensar cachen |
| `--vfs-write-back dauer` | Väntetid efter att en fil stängts innan uppladdningen till remoten startar |
| `--vfs-read-ahead grösse` | Extra datamängd som läses i förväg bortom den begärda positionen med `full` |
| `--poll-interval dauer` | Intervall då Rclone frågar remoten efter ändringar (endast för backends med stöd för polling) |
| `--dir-cache-time dauer` | Giltighetstid för cachade kataloglistor |
| `--allow-other` | Tillåter andra användare än den som monterar att komma åt FUSE-mounten |

</details>

De fullständiga listorna finns i Rclone-dokumentationen, särskilt under [rclone mount](https://rclone.org/commands/rclone_mount/) och i översikten över [globala flaggor](https://rclone.org/flags/).

## Bestäm först vad du vill uppnå

Cold Storage betyder inte automatiskt samma sak för varje applikation. En mediaserver läser oftast stora filer sekventiellt. En fotohantering laddar många små förhandsvisningsdata och hoppar till olika positioner. Ett dokumentarkiv öppnar relativt små filer, men ofta bara en gång.

Anteckna före testet de viktigaste egenskaperna hos ditt verkliga bestånd:

- typisk filstorlek samt största förekommande fil
- antal filer per katalog
- fullständig läsning eller slumpmässiga åtkomster till enskilda områden
- förhållandet mellan läs- och skrivåtkomster
- antal samtidiga användare eller processer
- ändringar som sker direkt i remoten utanför mounten
- acceptabel väntetid för en Cold Read
- maximalt tillgängligt utrymme för den lokala cachen

Först då uppstår meningsfulla framgångskriterier. Att öppna en enskild fil på 1.2 sekunder kan vara helt acceptabelt för ett arkiv och oanvändbart för en interaktiv applikation.

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
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Målkatalog där testträdet skapas |
| `--seed 42` | Fast startvärde för slumpgeneratorn; varje körning skapar samma bestånd |
| `--files 250` | Totalt 250 testfiler |
| `--files-per-directory 25` | I genomsnitt 25 filer per katalog |
| `--min-file-size 16K` | Minsta fil 16 KiB |
| `--max-file-size 32M` | Största fil 32 MiB |

</details>

Anpassa antal och storlekar efter ditt verkliga bestånd. Testa inte bara genomsnittliga filer. Några mycket små filer visar hur kostsamma metadataåtkomster är; några stora filer synliggör genomströmning, Read-Ahead och cachebeteende.

Lägg även till filnamn som kan orsaka problem i praktiken:

```bash
mkdir -p "testdata/Sonderfälle/Unterordner"
printf 'Leerzeichen\n' > "testdata/Sonderfälle/Datei mit Leerzeichen.txt"
printf 'Umlaute\n' > "testdata/Sonderfälle/Grösse und Änderung.txt"
printf 'Grossschreibung\n' > "testdata/Sonderfälle/Test.txt"
printf 'Kleinschreibung\n' > "testdata/Sonderfälle/test.txt"
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `mkdir -p` | Skapar även saknade överordnade kataloger och rapporterar inget fel om katalogen redan finns |
| `printf '…\n' > datei` | Skriver den angivna texten som filinnehåll; omdirigeringen skapar filen med det problematiska namnet |

</details>

Det sista testet är särskilt viktigt om det lokala filsystemet och moln-backenden behandlar stora och små bokstäver olika.

Om din tjänst bara accepterar vissa format räcker inte godtyckliga binärfiler. Skapa då dessutom syntetiska filer i just dessa format. För Paperless-ngx var det PDF:er med ett riktigt textlager, så att testet inte av misstag mäter OCR-prestandan i stället för lagringssökvägen. För en fotohantering ska beståndet innehålla olika bildstorlekar och format, och för en mediaserver korta filer med olika codecs.

## En referensmätning utan mount

Innan FUSE och VFS-cachen kommer in i bilden bör du mäta backenden direkt. Kopiera beståndet med Rclone till test-remoten och spara en detaljerad logg:

```bash
rclone copy ./testdata remote:cold-storage-test \
  --progress \
  --stats 5s \
  --log-file rclone-copy.log \
  --log-level INFO
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Källa för kopian: det lokalt skapade testbeståndet |
| `remote:cold-storage-test` | Mål: sökväg i den konfigurerade remoten |
| `--progress` | Löpande förloppsvisning i terminalen |
| `--stats 5s` | Överföringsstatistik var 5:e sekund i stället för standardintervallet |
| `--log-file rclone-copy.log` | Fullständig logg i en fil för senare utvärdering |
| `--log-level INFO` | Loggar överföringar, återförsök och fel utan DEBUG-omfattningen |

</details>

Kontrollera sedan att källa och mål stämmer överens:

```bash
rclone check ./testdata remote:cold-storage-test \
  --one-way \
  --download
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `./testdata` | Referens: det lokala originalbeståndet |
| `remote:cold-storage-test` | Kontrollobjekt: det nyuppladdade beståndet i remoten |
| `--one-way` | Kontrollerar bara om alla källfiler finns i målet och är identiska; extra filer i målet anmärks inte på |
| `--download` | Laddar ned data och jämför innehållet i stället för att förlita sig på hashvärden |

</details>

`--download` är avgörande här eftersom vissa backends inte tillhandahåller lämpliga hashvärden. Jämförelsen tar längre tid, men ger en användbar utgångspunkt för det senare integritetstestet.

Dokumentera uppladdningstid, överföringshastighet, antal återförsök och API-fel. Om direktåtkomsten redan är instabil kan mounten inte reparera det.

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
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `remote:cold-storage-test` | Remote inklusive sökväg som ska monteras |
| `/mnt/rclone-test` | Mountpunkt på testsystemet |
| `--vfs-cache-mode full` | Buffrar läs- och skrivåtkomster helt i den lokala cachen |
| `--cache-dir /var/cache/rclone-test` | Egen cachekatalog, separerad från produktionscachen |
| `--vfs-cache-max-size 10G` | Mjuk gräns på 10 GiB för VFS-cachen |
| `--vfs-cache-poll-interval 1m` | Cachekontroll och rensning varje minut |
| `--allow-other` | Även andra användare än den som monterar får åtkomst; krävs för tjänster och containrar |
| `--log-file /var/log/rclone-test.log` | Logg till en fil för att kunna följa åtkomster under testerna |
| `--log-level INFO` | Medelhög detaljnivå i loggen |

</details>

Värdena är ett exempel och ingen generell rekommendation. Det avgörande är separationen: En tom testcache gör Cold Reads reproducerbara utan att du måste radera filer från en aktiv produktionscache.

`--vfs-cache-mode full` är för applikationer oftast det mest informativa testläget. Rclone buffrar då läs- och skrivåtkomster lokalt och kan bättre efterlikna filåtkomster som inte vore möjliga med ren objektlagring. Den extra kompatibiliteten kostar lokalt lagringsutrymme.

## Kontrollera alltid ur den verkliga tjänstens perspektiv

En mount kan fungera för din användare men ändå vara oanvändbar för tjänsten. Vanliga orsaker är ett annat användar-ID, saknad `--allow-other`, containergränser eller felaktig mount-propagation.

Utför därför minst en fullständig läsning med samma identitet som applikationen senare kommer att köras under:

```bash
sudo -u <service-user> sha256sum /mnt/rclone-test/pfad/zur/datei
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-u <service-user>` | Kör kommandot som angiven användare, inte som root |
| `/mnt/rclone-test/pfad/zur/datei` | Fil som ska läsas; `sha256sum` tvingar fram en fullständig läsning |

</details>

Om tjänsten körs i Docker ska testet utföras i containern:

```bash
docker exec --user <uid>:<gid> <app-container> \
  sha256sum /pfad/im/container/datei
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `--user <uid>:<gid>` | Kör kommandot i containern med detta användar- och grupp-ID, oberoende av imagets standardanvändare |
| `<app-container>` | Namn eller ID för den körande applikationscontainern |
| `sha256sum /pfad/im/container/datei` | Kommando som ska köras; sökvägen är mounten ur containerns perspektiv |

</details>

Ännu bättre är ett riktigt applikationstest. Öppna filen via tjänstens webbgränssnitt eller API. Bara då märker du om applikationen exempelvis startar flera parallella läsningar, hoppar till slutet av filen eller förväntar sig ytterligare metadata.

## Mät Cold Reads och Warm Reads separat

Med `--vfs-cache-mode full` finns tre nivåer mellan applikationen och molnet:

| Nivå | Vad som finns där |
|---|---|
| Remote | den fullständiga filen i molntjänsten |
| VFS-cache | lokalt lagrade områden av redan lästa filer |
| Linux sidcache | nyligen använda data i RAM |

För en Cold Read väljer du en fil vars innehåll ännu aldrig har lästs via test-mounten. Vid den direkt efterföljande Warm Read ligger den i VFS-cachen och oftast dessutom i RAM.

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
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `date +%s%3N` | Tidsstämpel i millisekunder: Unix-sekunder, direkt följt av de tre första siffrorna i nanosekundsdelen (GNU date) |
| `cat "$file" > /dev/null` | Läser filen fullständigt och kastar bort utdata; endast lästiden mäts |
| `"$1"`, `"$2"` | Argument till shellfunktionen: filsökväg och etikett för mätraden |

</details>

Mät inte bara en fil. Använd minst tio tidigare olästa filer av olika storlek och anteckna median, långsammaste värde och filstorlek. Ett enskilt toppresultat är ingen grund för beslut.

En Warm Read är inte ett rent disktest eftersom kärnan kan behålla delar av filen i RAM. För de flesta Cold Storage-scenarier är det inget problem. Avgörande är vad en användare upplever vid första och upprepade öppningar. Om du vill bedöma RAM och lokal disk var för sig måste du dessutom kontrollera och bevisligen rensa sidcachen.

## Testa inte bara fullständiga läsningar

`cat` läser en fil från början till slut. Många applikationer beter sig annorlunda:

- En videospelare läser först header och index, hoppar senare till en annan position och fortsätter sedan att läsa sekventiellt.
- En bildhantering läser metadata och skapar sedan en förhandsvisningsbild.
- Ett arkivprogram kan först läsa slutet av filen.
- Flera workers kan samtidigt komma åt olika filer.

Testa dessa förlopp med den faktiska applikationen. Observera parallellt Rclone-loggen och cachen. För stora filer är det intressant hur mycket Rclone verkligen lagrar lokalt och om `--vfs-read-ahead` passar åtkomstmönstret.

En Rclone-mount är dessutom ingen lämplig lagringsplats för databaser eller andra filer som behöver tillförlitlig låsning och frekventa ändringar inom samma fil. VFS-lagret utjämnar skillnader mellan filsystem och objektlagring, men gör inte backenden till ett lokalt filsystem.

## Godkänn skrivsökvägen separat

Om din tjänst bara läser bör du om möjligt montera remoten skrivskyddat. Om den måste skriva ska du testa skapande, överskrivning, namnbyte och radering var för sig.

En skriven fil visas inte nödvändigtvis omedelbart i remoten. Med aktiv VFS-cache startar uppladdningen först efter att filen har stängts och `--vfs-write-back` har löpt ut. Kontrollera därför båda tillstånden:

1. Applikationen har stängt filen utan fel.
2. Filen kan därefter läsas i remoten via en direkt Rclone-åtkomst.

```bash
printf 'writeback-test\n' > /mnt/rclone-test/writeback-test.txt

# Efter att --vfs-write-back har löpt ut:
rclone cat remote:cold-storage-test/writeback-test.txt
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `/mnt/rclone-test/writeback-test.txt` | Målfil i mounten; omdirigeringen skriver via VFS-cachen |
| `remote:cold-storage-test/writeback-test.txt` | Direktåtkomst förbi mounten: `rclone cat` läser filen från remoten och skriver ut den till stdout |

</details>

Upprepa testet med en stor fil och avsluta Rclone medan uppladdningen fortfarande pågår. Starta sedan om med samma cachekatalog och kontrollera om uppladdningen fortsätter. Just detta tidsfönster avgör hur mycket data som är i fara vid ett serveravbrott.

Testa även namnbyte och radering. Många moln-backends hanterar dessa operationer annorlunda än ett lokalt filsystem. Det relevanta är inte bara om kommandot avslutas framgångsrikt, utan när ändringen blir synlig vid direktåtkomst till remoten och för andra klienter.

## Testa ändringar utanför mounten

Filer kan ändras via leverantörens webbgränssnitt, en andra Rclone-process eller en annan server. Mounten ser inte alltid sådana ändringar direkt eftersom kataloginformation cachas.

Skapa därför en fil direkt i remoten med ett andra Rclone-anrop:

```bash
printf 'external-change\n' > external-change.txt
rclone copyto external-change.txt \
  remote:cold-storage-test/external-change.txt
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `external-change.txt` | Källa: den lokalt skapade filen |
| `remote:cold-storage-test/external-change.txt` | Mål med exakt filnamn; `copyto` kopierar en enskild fil med exakt detta namn, i stället för att som `copy` kopiera den till en katalog |

</details>

Mät när filen visas i mounten. Upprepa testet för ändring och radering. Resultatet beror på backenden, dess stöd för polling samt `--poll-interval` och `--dir-cache-time`. Om applikationen måste se aktuella ändringar direkt ska detta beteende uttryckligen ingå i acceptanskriterierna.

Med aktiverat Remote Control-gränssnitt kan du specifikt tömma katalogcachen:

```bash
rclone rc vfs/forget
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `vfs/forget` | Remote Control-kommando som ska köras: tömmer VFS:ens cachade katalogträd så att nästa åtkomst frågar remoten på nytt |

</details>

Det är användbart för ett manuellt test, men ersätter ingen lämplig driftstrategi.

## Sätt cachen under press

En nästan tom cache är det enklaste fallet. Sätt `--vfs-cache-max-size` medvetet lågt i en andra testomgång och läs mer data än vad som ryms.

```bash
du -sh /var/cache/rclone-test/vfs
du -sh --apparent-size /var/cache/rclone-test/vfs
find /var/cache/rclone-test/vfs -type f | wc -l
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `du -s` | Sammanfattar utrymmesanvändningen på en rad i stället för att lista varje underkatalog |
| `du -h` | Utdata i läsbara enheter (K, M, G) |
| `du --apparent-size` | Visar den logiska filstorleken i stället för faktiskt använt lagringsutrymme |
| `find … -type f` | Tar bara hänsyn till vanliga filer, inte kataloger |
| `wc -l` | Räknar raderna i utdata, alltså här antalet cachefiler |

</details>

De båda storlekarna kan skilja sig kraftigt. I Full-läge använder Rclone Sparse Files: En fil visar sin fullständiga logiska storlek trots att bara de lästa områdena tar lokal plats.

Cachegränsen är dessutom mjuk. Rclone kontrollerar den med intervallet `--vfs-cache-poll-interval`, och öppna filer kan inte tas bort. Cachen får därför tillfälligt överskrida gränsen. Den bör dock sjunka igen efter att filerna stängts och nästa rensning har körts.

Logga toppvärdet, värdet efter rensningen och tiden som behövdes. På så sätt kan det nödvändiga lokala lagringsutrymmet dimensioneras förnuftigt.

## Simulera två olika avbrott

Ett moln som inte kan nås och en kraschad Rclone-process är två olika fel:

| Avbrott | Vad du kontrollerar |
|---|---|
| Backend eller nätverk kan inte nås, Rclone fortsätter att köra | Beteende vid återförsök, timeouter och redan cachade filer |
| Rclone-processen avslutas | FUSE-mountens beteende och återställning av mountpunkten |

Simulera båda endast i testmiljön. Du kan hårt avsluta en Rclone-container för det andra fallet:

```bash
docker kill --signal KILL <rclone-container>
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `--signal KILL` | Skickar SIGKILL (vilket redan är standard för `docker kill`, men anges här för tydlighetens skull); processen får ingen möjlighet att städa upp |
| `<rclone-container>` | Namn eller ID för Rclone-containern |

</details>

Kontrollera applikationen under avbrottet, inte bara mountpunkten:

- Vilka funktioner förblir tillgängliga?
- Hur länge väntar en åtkomst innan ett fel visas?
- Kan redan helt cachade filer fortfarande nås?
- Stoppar applikationen nya skrivoperationer?
- Uppstår ett begripligt felmeddelande eller bara en hängande process?
- Utlöses övervakningen?

En skrivtjänst får inte obemärkt skriva till den underliggande lokala katalogen när mounten saknas. När mounten kommer tillbaka skulle dessa filer döljas. Ett enkelt skydd före varje skrivjobb är:

```bash
mountpoint -q /mnt/rclone-test || exit 1
```

<details class="options-details">
<summary>Alternativen förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-q` | Ingen utdata; resultatet finns bara i exitkoden |
| `/mnt/rclone-test` | Sökväg som ska kontrolleras; exitkod 0 endast om en mount verkligen är aktiv där |
| `\|\| exit 1` | Avbryter skriptet om sökvägen inte är en mountpunkt |

</details>

Efter omstart av Rclone ska du kontrollera mounten på värden och från varje konsumerande container. En återupprättad mount når en redan körande container endast med korrekt mount-propagation. För Docker behövs vanligtvis `rslave` på den konsumerande sidan. Detaljerna finns i artikeln [Driftsätt Rclone-mountar i Docker på ett tillförlitligt sätt](/blog/rclone-mount-in-docker-container).

## Ett konkret exempel från Paperless-ngx

För mitt Paperless-test skapade jag 40 PDF:er med totalt 13.9 MB. Ett tidigare oöppnat dokument behövde cirka 1.8 sekunder, medan en direkt upprepad åtkomst tog 19 till 24 millisekunder. En VFS-cache begränsad till 4 MB steg tillfälligt till 12.7 MiB och rensades igen vid nästa körning.

Medan remoten inte kunde nås fungerade dokumentlistan, fulltextsökningen och förhandsvisningsbilderna vidare eftersom dessa data fanns lokalt. Endast originalet kunde inte öppnas. Efter att mounten återupprättats kunde den körande Paperless-containern åter få åtkomst till filerna utan att själv startas om.

Dessa siffror är inget benchmark för Rclone eller Proton Drive. Intressant är beteendet: Hot Storage förblev tillgängligt lokalt, Cold Reads var långsammare men förutsägbara, och tjänsten återhämtade sig efter avbrottet.

## Vad som ska ingå i testprotokollet

Ett resultat som senare går att följa upp innehåller minst:

- Rclone-version och använd backend
- operativsystem, FUSE-variant och filsystemet för cachekatalogen
- fullständigt mount-kommando utan inloggningsuppgifter
- antal, storleksfördelning och struktur för testfilerna
- Cold Read- och Warm Read-värden för flera filer
- skrivtid till synlighet i remoten
- cachens toppvärde och rensningens varaktighet
- resultat av `rclone check --download`
- beteende vid backendavbrott och avslutad Rclone-process
- återställningstid ur applikationens perspektiv
- återförsök, timeouter, begränsningar och autentiseringsfel från loggen

Definiera ett gränsvärde för varje punkt i förväg. Då avslutas testet med ett beslut och inte bara med en samling intressanta siffror.

## När uppsättningen är klar

En Cold Storage-mount är klar för användning när du kan svara ja på följande frågor:

- Är Cold Reads tillräckligt snabba för den avsedda tjänsten?
- Snabbar cachen upp upprepade åtkomster som förväntat?
- Förblir det lokala utrymmesbehovet hanterbart även under belastning?
- Stämmer alla filer efter en fullständig nedladdning?
- Fungerar alla nödvändiga filoperationer med vald backend?
- Beter sig applikationen kontrollerat vid ett molnavbrott?
- Stoppas skrivoperationer säkert när mounten saknas?
- Når en återupprättad mount alla körande konsumenter?
- Visar övervakningen avbrottet innan en användare rapporterar det?

Om ett svar saknas vet du åtminstone exakt vad du behöver arbeta vidare med. Det är betydligt mer hjälpsamt än en mount som såg bra ut vid första `ls` och först visar sina begränsningar i drift.

## Ändringshistorik

- **25 oktober 2026:** Rättat: För docker kill är SIGKILL redan standardsignalen; alternativet --signal KILL används bara för tydlighetens skull.

## Källor

1.  [Rclone test makefiles](https://rclone.org/commands/rclone_test_makefiles/): reproducerbara testfiler och katalogstrukturer med konfigurerbara storlekar.

2.  [Rclone mount](https://rclone.org/commands/rclone_mount/): VFS-cachelägen, Writeback, Sparse Files, cachegränser och katalogcache.

3.  [Rclone check](https://rclone.org/commands/rclone_check/): jämförelse av källa och mål, inklusive fullständig kontroll med `--download`.

4.  [Rclone Remote Control](https://rclone.org/rc/): riktad tömning av VFS-katalogcachen med `vfs/forget`.

5.  [Rclone Global Flags](https://rclone.org/flags/): fullständig referens för globala alternativ, inklusive loggning, statistik och VFS-parametrar.
