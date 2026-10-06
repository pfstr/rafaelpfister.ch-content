---
title: "Kiteworks: Produsenten anbefaler nedstenging 26. september – dette vet vi så langt"
navTitle: "Kiteworks-nedstenging"
description: "Kiteworks ber kundene sine via e-post om å slå av alle systemer lørdag 26.09.2026 fra kl. 04.00 til 10.00. Årsaken er en advarsel fra politimyndigheter om et mulig angrep. Siden 27.09. er anbefalingen opphevet; det finnes ingen CVE eller ny oppdatering. Totemomail er ikke berørt."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min lesetid"
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
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T10:03:39.279Z
translationReview: required
url: https://rafaelpfister.ch/no/blog/kiteworks-produsenten-anbefaler-nedstenging-26-september-dette-vet-vi-sa-langt
---

# Kiteworks: Produsenten anbefaler nedstenging 26. september – dette vet vi så langt

Kiteworks oppfordret kundene sine per e-post 25. september 2026 til å slå av alle Kiteworks-systemer lørdag 26. september fra kl. 04.00 til 10.00 (sentraleuropeisk tid). Ifølge brevet fra CISO Frank Balonis har produsenten informasjon fra politimyndigheter om at et angrep mot Kiteworks-systemer kan være nært forestående denne helgen. Kundestøtten begrunner nedstengingen med beskyttelse mot mulige zero-day-angrep. heise online har bekreftet ektheten av meldingen med kundestøtten per telefon.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Beredskapshjelp ved omlegging av e-postflyten</p>
<p>Hvis du trenger hjelp til å omdirigere e-postflyten før nedstengingen og tilbakeføre den etterpå, bruk gjerne <a href="https://adeptio.ch/">kontaktskjemaet på adeptio.ch</a>. Jeg er også tilgjengelig på kort varsel.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Oppdatering 28. september 2026: Kiteworks opphever anbefalingen om nedstenging</p>
<p>Kiteworks har lagt til en merknad i pressemeldingen: Siden 27. september gjelder ikke anbefalingen om nedstenging lenger for noen kunder.</p>
<blockquote lang="en">
<p>Per 27. september er anbefalingen om nedstenging nå opphevet for alle kunder. Hvis du ikke allerede har startet på nytt, kan du bringe Kiteworks-systemet ditt på nett igjen. Kunder med selvhostede Advanced Forms bør kontakte kundestøtten for hjelp. Alle systemer som Kiteworks drifter på vegne av kunder, er startet opp igjen og fungerer normalt.</p>
</blockquote>
<p>De som ennå ikke har startet systemene sine igjen, kan gjøre det nå. De som drifter Advanced Forms selv, bør kontakte Kiteworks-kundestøtten før omstart. Instansene som hostes av Kiteworks, kjører igjen. Det finnes fortsatt ikke noe CVE-nummer, ingen ny versjon utover 9.5.1, ingen indikatorer på kompromittering og ingen opplysning om et angrep ble forsøkt eller hva som lå bak advarselen. Siden Security Updates og GitHub-advisories er uendret.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Oppdatering 25. september 2026: Uttalelse fra Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks mottok troverdig trusselinformasjon fra politimyndigheter som indikerer at en trusselaktør kan forsøke å rette seg mot enkelte Kiteworks-systemer hos kunder. Av overflod av forsiktighet varslet vi kundene direkte og anbefalte et forebyggende nedstengingsvindu mens vi og våre samarbeidspartnere i politiet arbeider med saken. Vi kjenner ikke til noen kompromittering av Kiteworks-systemer, og denne advarselen er forebyggende og ikke en reaksjon på et bekreftet sikkerhetsbrudd. Alle kjente sårbarheter er utbedret i vår nåværende versjon, 9.5.1, og vi anbefaler fortsatt kundene å bruke den nyeste versjonen.</p>
<p>totemomail er ikke berørt av dette.</p>
</blockquote>
<p><strong>Totemomail er ikke berørt.</strong> Det er uavklart om Kiteworks EPG (Email Protection Gateway) er berørt.</p>
</div>

## Kronologi

Alle tider er i sentraleuropeisk sommertid (CEST). Der klokkeslett ikke er oppgitt, foreligger det ingen pålitelig tidsangivelse.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Advarsel til kundene</p>
<p>CISO Frank Balonis informerer kundene per e-post om opplysninger fra politimyndigheter om et mulig angrep denne helgen og anbefaler nedstenging i seks timer. Ifølge advarselen er alle kjente sårbarheter rettet i versjon 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Første medieoppslag</p>
<p>heise online rapporterer at Kiteworks-kundestøtten bekrefter ektheten av meldingen og begrunner nedstengingen med beskyttelse mot mulige zero-day-angrep. Kort etter følger TechCrunch, BleepingComputer, Computer Weekly og andre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september, 17.41</p>
<p class="timeline__titel">BKA uttaler seg ikke</p>
<p>heise legger til: BKA avslår å kommentere av etterforskningsmessige grunner. BSI svarer ikke, og FBI avstår fra å kommentere overfor TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Uttalelse og pressemelding</p>
<p>Kiteworks beskriver nedstengingen som et føre-var-tiltak uten kjent kompromittering. Pressemeldingen oppgir «federal intelligence authorities» som kilde og lister opp datterselskapene som ikke er berørt, blant annet totemo. Instanser som hostes av Kiteworks, stenger produsenten selv ned.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lør. 26. september, 04.00 til 10.00</p>
<p class="timeline__titel">Nedstengingsvindu</p>
<p>Vinduet gjelder samtidig over hele verden: 02.00 til 08.00 UTC, i Sydney 12.00 til 18.00, i New York fredag 22.00 til lørdag 04.00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lør. 26. september, 10.00</p>
<p class="timeline__titel">Slutt på vinduet</p>
<p>Vinduet som er angitt i kunde-e-posten, utløper. Den formelle opphevingen av anbefalingen følger 27. september.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Søn. 27. september</p>
<p class="timeline__titel">Anbefalingen opphevet</p>
<p>Kiteworks føyer til i pressemeldingen: Anbefalingen om nedstenging er opphevet for alle kunder, og systemene kan tas i bruk igjen. De hostede instansene er i drift igjen. Kunder med selvdriftede Advanced Forms bør kontakte kundestøtten.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status man. 28. september</p>
<p class="timeline__titel">Fortsatt uavklart</p>
<p>Ingen offentlig advarsel, intet CVE-nummer, ingen ny versjon, ingen indikatorer, ingen opplysninger om sårbarheten og ingen rapporter om et gjennomført eller forsøkt angrep.</p>
</li>
</ol>

## Det som er kjent

Anbefalingen gjelder globalt; e-posten oppgir tidsvinduet for alle tidssoner fra AEST til PDT. Kiteworks råder til å slå av systemene allerede før vinduet begynner, også dersom de ikke er tilgjengelige fra internett.

Nesten alt annet er foreløpig uavklart: Det finnes ingen offentlig sikkerhetsadvarsel, intet CVE-nummer, ingen oppdatering og ingen opplysninger om hvilke produkter eller versjoner som er berørt. Pressemeldingen oppgir «federal intelligence authorities» som kilde, altså antakelig amerikanske føderale myndigheter; hvilke er ukjent. Per 28. september finnes det ingen oppføring under Security Updates eller i GitHub-advarslene fra Kiteworks; den siste GitHub-oppføringen er fra 27. mai 2026. Offentlig tilgjengelig er uttalelsen sitert ovenfor og pressemeldingen fra 25. september.

Overfor TechCrunch ga Kiteworks-CISO Frank Balonis uttalelsen med samme ordlyd. BKA avslo å uttale seg overfor heise av etterforskningsmessige grunner, og BSI svarte ikke. FBI ønsket ikke å uttale seg overfor TechCrunch, og en talsperson for CISA ønsket ikke å uttale seg offentlig. Ifølge TechCrunch tok en kunde i helsesektoren serveren sin umiddelbart av nett, med merkbare driftsbegrensninger: Leger kunne bare nå pasientene sine med forsinkelser i en periode. Ifølge en sikkerhetsforsker sitert av TechCrunch er minst 1000 Kiteworks-systemer tilgjengelige fra internett; BornCity omtaler mer enn 1000 organisasjoner som har mottatt advarselen.

Pressemeldingen avviker på ett punkt fra kunde-e-posten: Den omtaler et nedstengingsvindu på ni timer, mens advarselen til kundene sier seks timer. Ifølge pressemeldingen gjelder anbefalingen bare selvdriftede installasjoner (On-Premises, AWS, Azure). Ifølge produsenten er datterselskapene Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai og 123FormBuilder ikke berørt.

## Advarselen til kundene

Kunde-e-posten fra 25. september inneholder, i tillegg til advarselen, en tidsplan per tidssone og en veiledning for klynger. Omregnet til UTC gir tidene det samme vinduet for alle regioner, fra 02.00 til 08.00 UTC.

| Tidssone | By | Start | Slutt |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Lør. 12.00 | Lør. 18.00 |
| SGT (UTC+8) | Singapore | Lør. 10.00 | Lør. 16.00 |
| IDT (UTC+3) | Tel Aviv | Lør. 05.00 | Lør. 11.00 |
| CEST (UTC+2) | Amsterdam, Zürich | Lør. 04.00 | Lør. 10.00 |
| BST (UTC+1) | London | Lør. 03.00 | Lør. 09.00 |
| EDT (UTC−4) | New York | Fre. 22.00 | Lør. 04.00 |
| CDT (UTC−5) | Chicago | Fre. 21.00 | Lør. 03.00 |
| MDT (UTC−6) | Denver | Fre. 20.00 | Lør. 02.00 |
| PDT (UTC−7) | San Francisco | Fre. 19.00 | Lør. 01.00 |

For klynger med flere servere angir Kiteworks en fast rekkefølge:

1.  **Slå på vedlikeholdsmodus** under System Setup > Maintenance Mode, slik at brukere ikke lenger får tilgang.

2.  **Opprett sikkerhetskopi:** Ta et øyeblikksbilde av hver node eller en sikkerhetskopi av Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Det oppbevares bare én databasesikkerhetskopi; hver ny erstatter den forrige.

3.  **Registrer roller:** Under System Setup > Locations viser kolonnen Assigned Roles hvilke noder som har Application-rollen; den primære Application-noden er merket med en stjerne. Noter nodene og IP-adressene deres, siden de trengs ved omstart.

4.  **Slå av i denne rekkefølgen:** først alle noder uten Application-rolle, deretter de øvrige Application-nodene og til slutt den primære Application-noden. Dette gjøres via fanen Shut Down for den aktuelle noden eller via hypervisorens konsoll (for eksempel VMware eller AWS) dersom Kiteworks-grensesnittet ikke lenger er tilgjengelig.

5.  **Start på nytt i omvendt rekkefølge** via hypervisoren, siden administrasjonskonsollen først blir tilgjengelig når tilstrekkelig mange noder kjører (vedlegg E i Administrator Guide): først den primære Application-noden, deretter de øvrige Application-nodene enkeltvis og først når den foregående kjører fullt, slik at databaseserverne kan danne et quorum. Deretter Storage-serverne, så de øvrige rollene (Repositories Gateway, Search, SFTP, Antivirus), og til slutt webserverne.

6.  **Slå av vedlikeholdsmodus** så snart alle noder i Cluster Health Dashboard på statussiden i administrasjonskonsollen er grønne.

På forespørsel har Kiteworks-kundestøtten dessuten bekreftet at ingen av datterselskapene til Kiteworks er berørt.

## Mulige årsaker: teorier

Så lenge Kiteworks ikke publiserer detaljer, forblir årsaken uavklart. Følgende forklaringer er hypoteser som kan utledes av de kjente faktaene; noen av dem diskuteres også i kommentarene til heise-meldingen. Ingen av dem er bekreftet.

Tre fakta snevrer inn mulighetsrommet. For det første angir advarselen et fast tidsvindu i stedet for en tidsubegrenset nedstenging frem til en oppdatering. For det andre skal også systemer som ikke er tilgjengelige fra internett, tas av nett. For det tredje ligger vinduet globalt på samme klokkeslett (02.00 til 08.00 UTC), fremfor å ligge i den lokale natten. En klassisk sårbarhet som kan utnyttes via internett, ville ikke forklart de to første punktene: Det hjelper å koble systemet fra internett, og det så lenge til oppdateringen foreligger.

### 1. Myndighetene kjenner et planlagt tidspunkt

Politimyndigheter får av og til kjennskap til tidspunktet for en planlagt kampanje på forhånd, for eksempel fra overvåket kommunikasjon i en gjerningsgruppe eller beslaglagt infrastruktur. Omfattende utnyttelser av produkter for filutveksling skjer typisk i et kort, koordinert tidsvindu, ofte i helger eller på helligdager når færre ansatte er på jobb. Kiteworks er etterfølgeren til Accellion, hvis File Transfer Appliance ble angrepet på nettopp denne måten i 2020 og 2021, den gang tilskrevet gruppen Clop: Data ble hentet ut gjennom flere sårbarheter, og de berørte organisasjonene ble deretter utsatt for utpressing.

Det snevre vinduet i helgen taler for dette. Imot taler innvendingen flere kommentatorer hos heise fremmer: Advarselen gikk til alle kunder, så angriperne vil trolig vite om den og kan ganske enkelt utsette angrepet. En utsettelse ville imidlertid gi produsenten tid til en oppdatering.

### 2. Produsenten kjenner ennå ikke selve sårbarheten

Det er også mulig at Kiteworks ikke har tekniske detaljer utover myndighetenes varsel, altså verken kjenner den berørte komponenten eller kan anbefale en oppdatering eller konfigurasjonsendring. Da er nedstenging det eneste tiltaket som virker uten kjennskap til sårbarheten, og den faste slutten et kompromiss kundene lettere aksepterer. I heise-kommentarene fremsettes antakelsen om at produsenten kan la enkelte systemer være på nett som lokkemat under vinduet for å observere angrepet. Det finnes ingen bevis for dette.

Det taler for dette at verken en advarsel eller et avbøtende tiltak er oppgitt. Imot taler at Kiteworks ifølge egne opplysninger samarbeider med Mandiant, og at det ved en advarsel fra myndigheter vanligvis i det minste foreligger indikatorer.

### 3. En allerede plassert bakdør med tidsutløser

Anbefalingen om å slå av også interne systemer passer med et scenario der angrepet ikke kommer utenfra, men allerede er forberedt på enhetene: for eksempel en bakdør fra en tidligere kompromittering som aktiveres på et fast tidspunkt eller tar kontakt med en kontrollserver. Et avslått system kan ikke utføre noe på dette tidspunktet.

Det taler for dette at tilgjengelighet fra internett ikke spiller noen rolle i dette scenarioet. Imot taler at en produsent i så fall heller ville anbefale kontroll for kompromittering og nyinstallasjon enn omstart etter seks timer.

### 4. Kompromittering hos produsenten

En annen vei til interne systemer er forbindelser som opprettes fra enheten til produsenten, for eksempel for oppdateringer, lisenskontroll eller fjernvedlikehold. Hvis en slik kanal er kompromittert, beskytter ikke en brannmur mot innkommende trafikk. I dette scenarioet ville nedstengingen gi produsenten et vindu til å rydde opp i egen infrastruktur, bytte nøkler eller sertifikater og først deretter tillate forbindelser igjen.

Det globale, ensartede tidspunktet taler for dette, siden det passer med en koordinert handling hos produsenten. Imot taler at produsenten da heller ville anbefale å blokkere utgående forbindelser fremfor å slå systemene helt av.

### 5. Følgetiltak til en myndighetsaksjon

Til slutt er det tenkbart at myndighetene i samme tidsrom aksjonerer mot angripernes infrastruktur og vil hindre at de slår til raskt som reaksjon. Det ville forklare det korte vinduet og politimyndighetenes rolle. At BKA avviser å kommentere av etterforskningsmessige grunner, tyder på pågående etterforskning, men beviser ikke denne varianten.

### Kritikk av kommunikasjonen

I heise-kommentarene dominerer skepsis, og innvendingene er saklig forståelige: Uten opplysninger om sårbarheten kan det ikke vurderes om frakobling fra internett med brannmur ville vært tilstrekkelig. Et tidsvindu uten varslet oppdatering lar det stå åpent hva som gjelder etter kl. 10.00. Og en advarsel som bare sendes per e-post til kunder, når ikke alle operatører, for eksempel hos partnere, tjenesteleverandører eller etter personalskifter. Uavhengig av hvilken teori som stemmer: De som drifter Kiteworks, bør kontrollere loggene etter oppstart og følge produsentens kanaler til en advarsel foreligger.

## Etter vinduet: Hva operatører kan gjøre nå

Kiteworks opphevet anbefalingen om nedstenging 27. september, men har ikke publisert tekniske detaljer. Det kan derfor ikke vurderes om eller hvordan faren ble fjernet. Ved oppstart og etterpå er følgende tiltak hensiktsmessige:

1.  **Kontroller versjon:** Kjører versjon 9.5.1 på alle noder? Ifølge produsenten er alle kjente sårbarheter rettet der.

2.  **Kontroller klyngetilstanden:** Alle noder bør være grønne i Cluster Health Dashboard, og vedlikeholdsmodus bør være slått av.

3.  **Evaluer logger:** Gå gjennom pålogginger, administratorhandlinger og uvanlige filnedlastinger rundt nedstengingsvinduet, særlig for systemer som ikke ble slått av eller ble slått av sent.

4.  **Begrens tilgjengelighet:** Der det er mulig, blokker tilgang fra internett til administrasjonsgrensesnittet og åpne bare nødvendige tjenester.

5.  **Advanced Forms:** De som drifter modulen selv, bør avklare omstarten med Kiteworks-kundestøtten på forhånd.

6.  **Følg kanalene:** Security Updates, GitHub-advisories, Newsroom og kunde-e-poster fra Kiteworks, til det foreligger en advarsel med tekniske detaljer.

## Kilder

1.  [heise online: Nært forestående zero-day-angrep: KiteWorks oppfordrer kunder til å slå av servere](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Første melding med utdrag fra kunde-e-posten og tidsvinduet; oppdatering fra 25.09. kl. 17.41 med BKAs svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): Engelsk versjon med CISO-ens opprinnelige ordlyd.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): Produsentens offisielle kanal, per 28.09.2026 uten oppføring om advarselen.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): Produsentens liste over advarsler, per 28.09.2026 med siste oppføring fra 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): Offisielle meldinger, siden 25.09.2026 med pressemeldingen om nedstengingen.

6.  [heise-forum: Kommentarer til meldingen](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Leserdiskusjon med innvendingene om det faste tidsvinduet og nedstengingen av interne systemer samt lokkematteorien.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advarsel om utnyttelsen av Accellion FTA i 2020/2021 med påfølgende utpressing; Accellion er det tidligere navnet på Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Produsentens opplysning om samarbeidet med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): Uttalelse fra CISO-en, utsendelsestidspunktet for advarselen, reaksjoner fra FBI og CISA (tillegg), konsekvenser hos en kunde, antall systemer tilgjengelige fra internett.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressemelding)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Offisiell melding fra 25.09.2026 med opplysninger om hostede instanser, versjon 9.5.1 og datterselskapene som ikke er berørt; supplert med merknaden fra 27.09.2026 om at anbefalingen om nedstenging er opphevet.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Tidsvindu etter regioner og vurdering av tidligere angrep mot produkter for filutveksling.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs vurdering av den uvanlige anbefalingen om nedstenging.

13.  [BornCity: Kiteworks: Mer enn 1 000 organisasjoner skal slå av servere](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): Antall varslede organisasjoner og bransjer i det tyskspråklige området.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): Vurdering av Accellion-angrepene fra Clop i 2020/2021 og sitat fra watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): Rapport fra 28.09.2026 om opphevingen av anbefalingen og driften av de hostede instansene.
