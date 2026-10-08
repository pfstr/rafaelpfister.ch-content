---
title: "Kiteworks: Produsenten anbefaler nedstenging 26. september – det som er kjent så langt"
navTitle: "Kiteworks-nedstenging"
description: "Kiteworks ba kundene sine om å stenge ned alle systemer lørdag 26.09.2026 fra kl. 04:00 til 10:00. Sluttrapport: Under nedstengingen fant og lukket produsenten en kritisk sårbarhet uten CVE; 30.09. fulgte 125 advisories, deriblant CVE-2026-54154 (CVSS 10.0) i Email Protection Gateway. Totemomail er ikke berørt."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min lesetid"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-produsenten-anbefaler-nedstenging-26-september-dette-vet-vi-sa-langt"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:58:35.493Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/kiteworks-produsenten-anbefaler-nedstenging-26-september-dette-vet-vi-sa-langt
---

# Kiteworks: Produsenten anbefaler nedstenging 26. september – det som er kjent så langt

Kiteworks ba kundene sine per e-post 25. september 2026 om å stenge ned alle Kiteworks-systemer lørdag 26. september fra kl. 04:00 til 10:00 (sentraleuropeisk tid). Ifølge brevet fra CISO Frank Balonis hadde produsenten fått indikasjoner fra politimyndigheter om at et angrep mot Kiteworks-systemer kunne være nært forestående denne helgen. Kundestøtten begrunnet nedstengingen med beskyttelse mot mulige zero-day-angrep. heise online har telefonisk bekreftet meldingens ekthet med kundestøtten.

<div class="update-hinweis">
<p class="update-hinweis__titel">Sluttrapport av 7. oktober 2026</p>
<p>Fra produsentens perspektiv er hendelsen avsluttet. Anbefalingen om nedstenging gjelder ikke lenger siden 27. september, og det er til nå ikke blitt kjent noe angrep mot Kiteworks- eller kundesystemer. De viktigste resultatene:</p>
<ul>
<li><strong>Kritisk sårbarhet funnet under nedstengingen:</strong> Ifølge pressemeldingen fra 28. september oppdaget Kiteworks, under analysen sammen med føderale myndigheter, en tidligere ukjent kritisk sårbarhet i en funksjon som er aktivert hos færre enn 1 % av kundene. Produsenten utviklet og rullet ut en rettelse i nedstengingsvinduet, og aktiverte dessuten et ekstra beskyttelseslag i alle miljøer. Hvilken funksjon som var berørt, er ikke offentliggjort; det finnes fortsatt ikke noe CVE-nummer for dette.</li>
<li><strong>125 advisories 30. september:</strong> To dager senere publiserte Kiteworks på GitHub 125 Security Advisories for Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) og MFT Server (3); 12 av dem kritiske, 49 høye. Alle er rettet i versjoner opptil og med 9.5.1, og de fleste ble rapportert via bug-bounty-programmet på YesWeHack. Slik situasjonen er kjent i dag, har de ingenting å gjøre med sårbarheten fra nedstengingsvinduet.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> Den mest alvorlige sårbarheten gjelder Email Protection Gateway før versjon 9.4.1. En uautentisert angriper kan kjøre kode med root-rettigheter via offentlig tilgjengelige endepunkter. 10 av de 12 kritiske advisoryene gjelder Email Protection Gateway.</li>
<li><strong>Ingen kjent utnyttelse:</strong> Det foreligger ingen rapporter om angrep for noen av sårbarhetene; per 7. oktober har CISA-KEV-katalogen ingen Kiteworks-oppføring fra 2026. Ifølge BleepingComputer teller Shadowserver nesten 400 Kiteworks-instanser som er tilgjengelige fra internett.</li>
<li><strong>Totemomail:</strong> Er ikke nevnt i noen av advisoryene og var ifølge produsenten ikke berørt av nedstengingen.</li>
</ul>
<p><strong>Tiltak nødvendig:</strong> De som drifter Kiteworks selv, bør oppdatere alle komponenter til versjon 9.5.1; Email Protection Gateway har høyest prioritet. Detaljene finnes i avsnittet <a href="#abschlussbericht">Sluttrapport</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Oppdatering fra 28. september 2026: Kiteworks opphever anbefalingen om nedstenging</p>
<p>Kiteworks har supplert pressemeldingen med en merknad: Anbefalingen om nedstenging gjelder ikke lenger for noen kunder siden 27. september.</p>
<blockquote lang="en">
<p>Fra 27. september er anbefalingen om nedstenging nå opphevet for alle kunder. Hvis du ikke allerede har startet på nytt, kan du sette Kiteworks-systemet ditt tilbake på nett. Kunder med selvdriftede Advanced Forms bør kontakte kundestøtten for hjelp. Alle systemer som Kiteworks drifter på vegne av kunder, er startet opp igjen og fungerer normalt.</p>
</blockquote>
<p>De som ennå ikke har startet systemene sine opp igjen, kan gjøre det nå. De som drifter Advanced Forms selv, skal kontakte Kiteworks-støtten før omstart. Instansene som driftes av Kiteworks, kjører igjen. Det finnes fortsatt ikke noe CVE-nummer, ingen ny versjon utover 9.5.1, ingen indikatorer på kompromittering og ingen opplysning om det ble forsøkt et angrep eller hva som lå bak advarselen. Siden for sikkerhetsoppdateringer og GitHub-advisoryene er uendret.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Oppdatering fra 25. september 2026: Uttalelse fra Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks mottok troverdig trusselinformasjon fra politimyndigheter som indikerte at en trusselaktør kunne forsøke å angripe enkelte Kiteworks-systemer hos kunder. Av stor forsiktighet varslet vi kundene direkte og anbefalte et forebyggende nedstengingsvindu mens vi og våre samarbeidspartnere i politiet arbeider med saken. Vi kjenner ikke til at Kiteworks-systemer er kompromittert, og dette varselet er forebyggende og ikke en reaksjon på et bekreftet innbrudd. Alle kjente sårbarheter er håndtert i vår nåværende versjon, 9.5.1, og vi fortsetter å anbefale at kundene kjører den nyeste versjonen.</p>
<p>totemomail er ikke berørt av dette.</p>
</blockquote>
<p><strong>Totemomail er ikke berørt.</strong> Det er uavklart om Kiteworks EPG (Email Protection Gateway) er berørt.</p>
</div>

## Kronologi

Alle tider er i sentraleuropeisk sommertid (CEST). Der klokkeslett ikke er oppgitt, finnes det ingen pålitelig tidsangivelse.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Advisory til kundene</p>
<p>CISO Frank Balonis informerer kundene per e-post om indikasjoner fra politimyndigheter på et mulig angrep denne helgen og anbefaler nedstenging i seks timer. Ifølge advisoryet er alle kjente sårbarheter rettet i versjon 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Første medieoppslag</p>
<p>heise online rapporterer at Kiteworks-støtten bekrefter meldingens ekthet og begrunner nedstengingen med beskyttelse mot mulige zero-day-angrep. Kort etter følger TechCrunch, BleepingComputer, Computer Weekly og flere andre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september, 17:41</p>
<p class="timeline__titel">BKA uttaler seg ikke</p>
<p>heise legger til: BKA avviser å uttale seg av etterforskningstaktiske grunner. BSI svarer ikke, og FBI avstår fra kommentar overfor TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Uttalelse og pressemelding</p>
<p>Kiteworks beskriver nedstengingen som et forsiktighetstiltak uten kjent kompromittering. Pressemeldingen oppgir «federal intelligence authorities» som kilde og lister opp de ikke-berørte datterselskapene, deriblant totemo. Produsenten stenger selv ned instanser som driftes av Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lør. 26. september, 04:00 til 10:00</p>
<p class="timeline__titel">Nedstengingsvindu</p>
<p>Vinduet er samtidig verden over: 02:00 til 08:00 UTC, i Sydney 12:00 til 18:00, i New York fredag 22:00 til lørdag 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lør. 26. september, 10:00</p>
<p class="timeline__titel">Slutt på vinduet</p>
<p>Vinduet som er oppgitt i kunde-e-posten, avsluttes. Den formelle opphevingen av anbefalingen følger 27. september.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Søn. 27. september</p>
<p class="timeline__titel">Anbefalingen opphevet</p>
<p>Kiteworks supplerer pressemeldingen: Anbefalingen om nedstenging er opphevet for alle kunder, og systemene kan kjøre igjen. De driftede instansene er tilbake i drift. Kunder med selvdriftede Advanced Forms skal kontakte støtten.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Man. 28. september</p>
<p class="timeline__titel">Kritisk sårbarhet funnet og lukket</p>
<p>Kiteworks melder i en ny pressemelding at det under arbeidet med føderale myndigheter under nedstengingen ble funnet en tidligere ukjent kritisk sårbarhet. Den berører en funksjon som er aktivert hos færre enn 1 % av kundene. Rettelsen og et ekstra beskyttelseslag er rullet ut, og det finnes ingen tegn på kompromittering. Produsenten oppgir ikke funksjonen eller CVE-nummeret.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ons. 30. september, fra 18:38</p>
<p class="timeline__titel">125 Security Advisories på GitHub</p>
<p>Kiteworks publiserer 125 advisories for Core, Email Protection Gateway, Secure Data Forms og MFT Server, alle rettet innen versjon 9.5.1. Den mest alvorlige sårbarheten er CVE-2026-54154 i Email Protection Gateway før 9.4.1 (CVSS 10.0, kjøring av kode med root-rettigheter uten innlogging).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Tor. 1. oktober</p>
<p class="timeline__titel">Medieoppslag og MS-ISAC-advisory</p>
<p>BleepingComputer, SecurityOnline og flere andre rapporterer om advisoryene; MS-ISAC (Center for Internet Security) utgir et eget advisory om CVE-2026-54154. Ingen utnyttelse er kjent.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status ons. 7. oktober</p>
<p class="timeline__titel">Avslutning</p>
<p>Ingen rapporter om et gjennomført eller forsøkt angrep, og ingen Kiteworks-oppføring i CISA-KEV-katalogen. Det som fortsatt er uavklart, er den berørte funksjonen, et CVE-nummer for sårbarheten fra nedstengingsvinduet og bakgrunnen for myndighetenes advarsel.</p>
</li>
</ol>

## Det som er kjent

Anbefalingen gjelder globalt; e-posten oppgir tidsvinduet for alle tidssoner fra AEST til PDT. Kiteworks råder til å stenge systemene ned allerede før vinduet begynner, også dersom de ikke er tilgjengelige fra internett.

Frem til 28. september var nesten alt annet uavklart: Det fantes ikke noe offentlig Security Advisory, intet CVE-nummer, ingen oppdatering og ingen opplysning om hvilke produkter eller versjoner som er berørt. Pressemeldingen oppgir «federal intelligence authorities» som kilde, trolig amerikanske føderale myndigheter; hvilke, er fortsatt ikke kjent. I GitHub-advisoryene fra Kiteworks stammet den siste oppføringen frem til da fra 27. mai 2026; advisoryene fra 30. september er oppsummert i avsnittet [Sluttrapport](#abschlussbericht).

Overfor TechCrunch ga Kiteworks-CISO Frank Balonis uttalelsen med samme ordlyd. BKA avslo å uttale seg overfor heise av etterforskningstaktiske grunner, og BSI svarte ikke. FBI ønsket ikke å uttale seg overfor TechCrunch, og en talsperson for CISA ønsket ikke å uttale seg offentlig. Ifølge TechCrunch tok en kunde i helsevesenet umiddelbart serveren sin av nettet, med merkbare driftsforstyrrelser: Leger kunne i perioder bare nå pasientene sine forsinket. Ifølge en sikkerhetsforsker sitert av TechCrunch er minst 1000 Kiteworks-systemer tilgjengelige fra internett; BornCity snakker om mer enn 1000 organisasjoner som mottok advarselen.

Pressemeldingen avviker på ett punkt fra kunde-e-posten: Den snakker om et nedstengingsvindu på ni timer, mens advisoryet til kundene angir seks timer. Ifølge pressemeldingen gjelder anbefalingen bare selvdriftede installasjoner (On-Premises, AWS, Azure). Ifølge produsenten er datterselskapene Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai og 123FormBuilder ikke berørt.

## Advisoryet til kundene

Kunde-e-posten fra 25. september inneholder, i tillegg til advarselen, en tidsplan per tidssone og en veiledning for klynger. Regnes tidene om til UTC, får alle regioner det samme vinduet fra 02:00 til 08:00 UTC.

| Tidssone | By | Start | Slutt |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Lør. 12:00 | Lør. 18:00 |
| SGT (UTC+8) | Singapore | Lør. 10:00 | Lør. 16:00 |
| IDT (UTC+3) | Tel Aviv | Lør. 05:00 | Lør. 11:00 |
| CEST (UTC+2) | Amsterdam, Zürich | Lør. 04:00 | Lør. 10:00 |
| BST (UTC+1) | London | Lør. 03:00 | Lør. 09:00 |
| EDT (UTC−4) | New York | Fre. 22:00 | Lør. 04:00 |
| CDT (UTC−5) | Chicago | Fre. 21:00 | Lør. 03:00 |
| MDT (UTC−6) | Denver | Fre. 20:00 | Lør. 02:00 |
| PDT (UTC−7) | San Francisco | Fre. 19:00 | Lør. 01:00 |

For klynger med flere servere angir Kiteworks en fast rekkefølge:

1.  **Slå på vedlikeholdsmodus** under System Setup > Maintenance Mode, slik at brukere ikke lenger får tilgang.

2.  **Opprett sikkerhetskopi:** Ta et øyeblikksbilde av hver node eller en sikkerhetskopi av Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Det beholdes bare én databasesikkerhetskopi; hver ny erstatter den forrige.

3.  **Registrer roller:** Under System Setup > Locations viser kolonnen Assigned Roles hvilke noder som har Application-rollen; den primære Application-noden er markert med en stjerne. Noter nodene og IP-adressene deres, da de trengs for omstart.

4.  **Steng ned i denne rekkefølgen:** Først alle noder uten Application-rolle, deretter de øvrige Application-nodene og til slutt den primære Application-noden. Dette gjøres via fanen Shut Down for den aktuelle noden eller via hypervisorens konsoll (for eksempel VMware eller AWS) dersom Kiteworks-grensesnittet ikke lenger er tilgjengelig.

5.  **Start på nytt i omvendt rekkefølge** via hypervisoren, siden administratorkonsollen først blir tilgjengelig når nok noder kjører (vedlegg E i Administrator Guide): Først den primære Application-noden, deretter de øvrige Application-nodene én etter én og først når den foregående kjører fullt ut, slik at databaseserverne kan danne et quorum. Deretter Storage-serverne, så de øvrige rollene (Repositories Gateway, Search, SFTP, Antivirus) og til slutt webserverne.

6.  **Slå av vedlikeholdsmodus** så snart alle noder i Cluster Health Dashboard på statussiden i administratorkonsollen er grønne.

På forespørsel har Kiteworks-støtten dessuten bekreftet at ingen av Kiteworks' datterselskaper er berørt.

## Mulige årsaker: teorier

Dette avsnittet ble til før 28. september; vurderingen basert på dagens status finnes i [Sluttrapport](#abschlussbericht). Forklaringene nedenfor er hypoteser som kan utledes av de kjente hovedfaktaene; noen av dem diskuteres også i kommentarene til heise-saken. Ingen av dem er bekreftet.

Tre hovedfakta avgrenser mulighetene. For det første nevner advarselen et fast tidsvindu i stedet for en ubegrenset nedstenging frem til en oppdatering foreligger. For det andre skal også systemer som ikke er tilgjengelige fra internett, tas av nettet. For det tredje ligger vinduet på samme tidspunkt verden over (02:00 til 08:00 UTC), i stedet for i den lokale natten. En klassisk sårbarhet som kan utnyttes over internett, ville ikke forklare de to første punktene: Det hjelper å koble systemet fra internett, og det så lenge til oppdateringen er klar.

### 1. Myndighetene kjenner et planlagt tidspunkt

Politimyndigheter får iblant vite tidspunktet for en planlagt kampanje på forhånd, for eksempel fra overvåket kommunikasjon i en gjerningsgruppe eller fra beslaglagt infrastruktur. Masseutnyttelse av produkter for filutveksling foregår vanligvis i et kort, koordinert vindu, ofte i helger eller på helligdager når færre ansatte er på jobb. Kiteworks er etterfølgeren til Accellion, hvis File Transfer Appliance ble angrepet nettopp slik i 2020 og 2021, den gang tilskrevet gruppen Clop: Data ble hentet ut via flere sårbarheter, og deretter ble de berørte organisasjonene utpresset.

Det avgrensede helgevinduet taler for dette. Imot taler innvendingen som flere kommentatorer hos heise fremmer: Advarselen gikk til alle kundene, så angriperne må antas å vite om den og kan ganske enkelt utsette angrepet. En utsettelse ville imidlertid gi produsenten tid til å lage en oppdatering.

### 2. Produsenten kjenner ennå ikke sårbarheten selv

Det er også mulig at Kiteworks, utover myndighetenes varsel, ikke har tekniske detaljer, altså verken kjenner den berørte komponenten eller kan anbefale en oppdatering eller konfigurasjonsendring. Da er nedstengingen det eneste tiltaket som virker uten kjennskap til sårbarheten, og den faste slutten et kompromiss som kundene lettere aksepterer. I heise-kommentarene uttrykkes formodningen om at produsenten kunne la enkelte systemer stå på nett som lokkemat under vinduet for å observere angrepet. Det finnes ingen bevis for dette.

Dette støttes av at verken et advisory eller en mitigation er oppgitt. Imot taler at Kiteworks ifølge egne opplysninger samarbeider med Mandiant, og ved en advarsel fra myndigheter er indikatorer som regel i det minste tilgjengelige.

### 3. En allerede plassert bakdør med tidsutløser

Anbefalingen om å stenge ned også interne systemer passer med et scenario der angrepet ikke kommer utenfra, men allerede er forberedt på enhetene: for eksempel en bakdør fra en tidligere kompromittering som aktiveres på et bestemt tidspunkt eller tar kontakt med en kontrollserver. Et avslått system kan ikke utføre noe på dette tidspunktet.

Dette støttes av at tilgjengelighet fra internett ikke spiller noen rolle i dette scenariet. Imot taler at en produsent i et slikt tilfelle snarere ville anbefale kontroll for kompromittering og nyinstallasjon enn omstart etter seks timer.

### 4. Kompromittering hos produsenten

En annen vei som når interne systemer, er forbindelser som opprettes fra enheten til produsenten, for eksempel for oppdateringer, lisenskontroll eller fjernvedlikehold. Hvis en slik kanal er kompromittert, beskytter ikke en brannmur mot innkommende trafikk. I dette scenariet ville nedstengingen gi produsenten et vindu til å rydde opp i egen infrastruktur, bytte nøkler eller sertifikater og først deretter tillate forbindelser igjen.

Det verdensomspennende, ensartede tidspunktet taler for dette, da det passer med en koordinert handling hos produsenten. Imot taler at produsenten da snarere ville anbefale å blokkere utgående forbindelser enn å stenge systemene helt ned.

### 5. Følgetiltak til en myndighetsaksjon

Til slutt er det tenkelig at myndighetene i samme tidsrom går til aksjon mot angripernes infrastruktur og vil hindre at disse slår til raskt som reaksjon. Det ville forklare det korte vinduet og politimyndighetenes rolle. At BKA avviser å uttale seg av etterforskningstaktiske grunner, tyder på pågående etterforskning, men beviser ikke denne varianten.

### Kritikk av kommunikasjonen

I heise-kommentarene dominerer skepsis, og innvendingene er saklig forståelige: Uten opplysninger om sårbarheten kan det ikke vurderes om frakobling fra internett med brannmur ville vært tilstrekkelig. Et tidsvindu uten varslet oppdatering lar det stå åpent hva som gjelder etter kl. 10:00. Og en advarsel som kun sendes per e-post til kunder, når ikke alle driftsansvarlige, for eksempel hos partnere, tjenesteleverandører eller etter personellskifter. Uavhengig av hvilken teori som stemmer: De som drifter Kiteworks, bør kontrollere loggene etter oppstart og følge produsentens kanaler til et advisory foreligger.

## Sluttrapport

Per 7. oktober 2026 er hendelsen avsluttet fra produsentens perspektiv. Hendelsene etter nedstengingsvinduet kan skilles i to spor: sårbarheten som ble funnet under nedstengingen, og samlepubliseringen av advisories to dager senere.

### Sårbarheten fra nedstengingsvinduet

28. september publiserte Kiteworks en andre pressemelding. Ifølge denne samarbeidet produsenten med føderale myndigheter gjennom helgen; da ble det oppdaget en tidligere ukjent kritisk sårbarhet begrenset til en funksjon som er aktivert hos færre enn 1 % av kundene. Kiteworks utviklet og rullet ut en rettelse i nedstengingsvinduet, og aktiverte dessuten et ekstra beskyttelseslag i alle miljøer. Den kontinuerlige overvåkingen hadde ikke vist mistenkelig aktivitet, og det fantes ingen tegn på kompromittering av Kiteworks- eller kundesystemer. Alle øvrige Kiteworks-produkter var ikke berørt.

Det som ikke er offentliggjort, er den berørte funksjonen, et CVE-nummer, versjonene med rettelsen og om selvdriftede installasjoner fikk rettelsen automatisk. Opphevingen 27. september inneholdt bare ett unntak: Kunder med selvdriftede Advanced Forms skulle kontakte støtten før omstart. Kiteworks har ikke bekreftet om denne funksjonen var den berørte.

Når det gjelder teoriene ovenfor: Pressemeldingen beskriver en sårbarhet som først ble funnet under vinduet. Det passer med teori 2 (produsenten kjente ikke sårbarheten på forhånd) i kombinasjon med teori 1 (myndighetene kjente et planlagt tidspunkt). Det finnes ingen bekreftelse på teori 3 til 5. Hva myndighetene konkret visste, og om et angrep ble forsøkt, er fortsatt ikke kjent.

### 125 Security Advisories fra 30. september

30. september fra kl. 18:38 publiserte Kiteworks 125 Security Advisories på GitHub på én gang. De fordeler seg slik:

| Produkt | Advisories | hvorav kritiske |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Totalt** | **125** | **12** |

Etter alvorlighetsgrad er det 12 kritiske, 49 høye, 52 middels og 12 lave vurderinger. Alle sårbarhetene er rettet i versjoner opptil og med 9.5.1; de eldste oppføringene gjelder versjon 9.2.1. Det dreier seg altså om en etterfølgende offentliggjøring av rettelser som allerede var levert, ikke en ny versjon. Advisoryene oppgir for det meste deltakere i bug-bounty-programmet på YesWeHack som rapportører. Kiteworks knytter ingen forbindelse til sårbarheten fra nedstengingsvinduet; én uke etter vinduet er advisoryenes status fortsatt i tråd med uttalelsen fra 25. september om at alle kjente sårbarheter er rettet i 9.5.1.

De kritiske advisoryene:

| CVE | Produkt | CVSS 3.1 | rettet fra | Konsekvens |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Kjøring av kode med root-rettigheter uten innlogging |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Konto-overtakelse |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Konto-overtakelse |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Konto-overtakelse via tilbakestilling av passord |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Konto-overtakelse |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Konto-overtakelse |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Omgåelse av sikkerhetsfunksjoner |
| CVE-2026-102095, CVE-2026-102102 til 102105 | EPG | 9.1 | 9.5.0 | Tilgang til interne nettverksressurser (SSRF) |

CVE-2026-54154 er den mest alvorlige sårbarheten: Ifølge advisoryet gjør en kombinasjon av feil i inndatavalidering på offentlig tilgjengelige endepunkter i Email Protection Gateway det mulig for en ikke-innlogget angriper å kjøre kode og, gjennom ytterligere lokale svakheter, få root-rettigheter på enheten. MS-ISAC utga et eget advisory om dette 1. oktober. For e-postadministratorer er gatewayen den relevante delen av publiseringen: Den står vanligvis direkte i e-postflyten og er tilgjengelig fra internett.

### Utnyttelse og utbredelse

Det foreligger ingen rapporter om utnyttelse eller offentlige exploiter for noen av sårbarhetene. Per 7. oktober inneholder CISA-KEV-katalogen kun de fire Accellion-FTA-oppføringene fra 2021. Ifølge BleepingComputer teller Shadowserver nesten 400 Kiteworks-instanser som er tilgjengelige fra internett; hvor mange av dem som allerede kjører 9.5.1, er ikke kjent. Totemomail er ikke nevnt i noen av advisoryene.

## Etter vinduet: Hva driftsansvarlige kan gjøre nå

Anbefalingen om nedstenging er opphevet, og de kjente sårbarhetene er rettet i versjon 9.5.1. For selvdriftede installasjoner er følgende tiltak fornuftige:

1.  **Kontroller versjon:** Kjører alle noder og alle komponenter (Core, Email Protection Gateway, Secure Data Forms, MFT Server) versjon 9.5.1? En Email Protection Gateway før 9.4.1 er berørt av CVE-2026-54154 og bør oppdateres først.

2.  **Kontroller klyngetilstand:** I Cluster Health Dashboard skal alle noder være grønne, og vedlikeholdsmodus skal være slått av.

3.  **Evaluer logger:** Gå gjennom innlogginger, administratorhandlinger og uvanlige filnedlastinger rundt nedstengingsvinduet, særlig på systemer som ikke ble stengt ned eller ble stengt ned sent.

4.  **Begrens tilgjengelighet:** Der det er mulig, blokker internettilgang til administrasjonsgrensesnittet og tillat bare nødvendige tjenester.

5.  **Advanced Forms:** De som drifter modulen selv og ennå ikke har kontaktet støtten, avklarer med Kiteworks om rettelsen fra nedstengingsvinduet er kommet til deres egen installasjon.

6.  **Sammenlign advisories:** GitHub-advisoryene kan filtreres etter produkt (prefiks `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Kontroller for hver komponent som brukes om den installerte versjonen ligger under den oppgitte rettelsesversjonen.

7.  **Følg kanalene:** GitHub-advisoryene, Newsroom og kunde-e-postene fra Kiteworks, i tilfelle produsenten likevel publiserer et advisory med CVE-nummer for sårbarheten fra nedstengingsvinduet. Nye CVE-er for Kiteworks Email Protection Gateway og Totemomail finnes også i [CVE-trackeren](/cve) på denne siden; der kan du abonnere på e-postvarsler.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Hjelp med oppdateringen</p>
<p>Hvis du trenger hjelp med oppdatering av en Kiteworks- eller Totemomail-gateway, for eksempel med omdirigering av e-postflyten under vedlikeholdsvinduet eller med evaluering av logger, kan du bruke <a href="https://adeptio.ch/">kontaktskjemaet på adeptio.ch</a>.</p>
</div>

## Kilder

1.  [heise online: Forestående zero-day-angrep: KiteWorks presser kunder til å stenge servere](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Første melding med utdrag fra kunde-e-posten og tidsvinduet; oppdatering fra 25.09. kl. 17:41 med BKAs svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): Engelsk versjon med CISO-ens opprinnelige ordlyd.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): Produsentens eldre oppdateringsside, per 07.10.2026 uten oppføring om advarselen eller advisoryene fra 30.09.2026.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): Produsentens advisoryliste, frem til 28.09.2026 var siste oppføring fra 27.05.2026; 30.09.2026 kom 125 nye advisories for Core, EPG, SDF og MFT. Tallene i denne artikkelen er talt via GitHub-API-et.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): Offisielle meldinger, med pressemeldingen om nedstengingen siden 25.09.2026.

6.  [heise-forum: Kommentarer til saken](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Leserdebatt med innvendingene om det faste tidsvinduet og nedstengingen av interne systemer, samt lokkemat-teorien.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory om utnyttelse av Accellion FTA i 2020/2021 med påfølgende utpressing; Accellion er det tidligere navnet på Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Produsentens opplysninger om samarbeid med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): Uttalelse fra CISO-en, tidspunktet da advarselen ble sendt, reaksjoner fra FBI og CISA (tillegg), konsekvenser hos en kunde og antall systemer tilgjengelige fra internett.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressemelding)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Offisiell melding fra 25.09.2026 med opplysninger om driftede instanser, versjon 9.5.1 og de ikke-berørte datterselskapene; supplert med merknaden fra 27.09.2026 om at anbefalingen om nedstenging er opphevet.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Tidsvindu etter regioner og vurdering av tidligere angrep mot produkter for filutveksling.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs vurdering av den uvanlige anbefalingen om nedstenging.

13.  [BornCity: Kiteworks: Mer enn 1 000 organisasjoner skal stenge servere](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): Antall varslede organisasjoner og bransjer i tyskspråklige områder.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): Vurdering av Accellion-angrepene fra Clop i 2020/2021 og sitat fra watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): Rapport fra 28.09.2026 om opphevingen av anbefalingen og driften av de driftede instansene.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (pressemelding)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): Melding fra 28.09.2026 om den kritiske sårbarheten som ble funnet under nedstengingen, rettelsen og det ekstra beskyttelseslaget.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): Sammendrag av den andre pressemeldingen med sitater fra CISO-en.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Produsentens opplysninger om kodekjøring i Email Protection Gateway før 9.4.1, CVSS 10.0, rapportering via YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): Rapport fra 01.10.2026 om CVE-2026-54154 og antall instanser som Shadowserver har telt.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): Advisory fra Center for Internet Security fra 01.10.2026 med anbefalinger.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): Vurdering av sårbarhetene for konto-overtakelse i Core, deriblant CVE-2026-102115; opptellingen avviker fra advisorylisten.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): Per 07.10.2026 kun de fire Accellion-FTA-oppføringene fra 2021, ingen Kiteworks-oppføring fra 2026.
