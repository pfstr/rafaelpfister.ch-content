---
title: "Kiteworks: Produsenten anbefaler nedstenging 26. september – dette vet vi så langt"
navTitle: "Kiteworks-nedstenging"
description: "Kiteworks ber kundene sine på e-post om å stenge ned alle systemer lørdag 26.09.2026 fra kl. 04:00 til 10:00. Årsaken er en advarsel fra politimyndigheter om et mulig angrep. Totemomail er ikke berørt."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min lesetid"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-produsenten-anbefaler-nedstenging-26-september-dette-vet-vi-sa-langt"
featured: "2026-09-27"
warnung: true
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
url: https://rafaelpfister.ch/no/blog/kiteworks-produsenten-anbefaler-nedstenging-26-september-dette-vet-vi-sa-langt
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-27T09:23:18.225Z
translationReview: automatic
---

# Kiteworks: Produsenten anbefaler nedstenging 26. september – dette vet vi så langt

Kiteworks ba kundene sine på e-post 25. september 2026 om å stenge ned alle Kiteworks-systemer lørdag 26. september fra kl. 04:00 til 10:00 (sentraleuropeisk tid). Ifølge brevet fra CISO Frank Balonis har produsenten fått indikasjoner fra politimyndigheter på at et angrep på Kiteworks-systemer kan være nært forestående denne helgen. Kundestøtten begrunner nedstengingen med beskyttelse mot mulige zero-day-angrep. heise online har bekreftet meldingens ekthet per telefon med kundestøtten.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Bistand i nødstilfeller ved omlegging av e-postflyten</p>
<p>Hvis du trenger hjelp til å omdirigere e-postflyten før nedstengingen og sette den tilbake etterpå, kan du bruke <a href="https://adeptio.ch/">kontaktskjemaet på adeptio.ch</a>. Jeg kan også komme tilbake til deg på kort varsel.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Oppdatering fra 25. september 2026: Uttalelse fra Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail er ikke berørt.</strong> Det er fortsatt uavklart om Kiteworks EPG (Email Protection Gateway) er berørt.</p>
</div>

## Kronologi

Alle tider er i sentraleuropeisk sommertid (CEST). Der klokkeslett ikke er oppgitt, foreligger det ingen pålitelig tidsangivelse.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Advisory til kundene</p>
<p>CISO Frank Balonis informerer kundene per e-post om indikasjoner fra politimyndigheter på et mulig angrep denne helgen og anbefaler nedstenging i seks timer. Ifølge advisoryet er alle kjente sårbarheter rettet i versjon 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Første medieoppslag</p>
<p>heise online rapporterer at Kiteworks-støtten bekrefter meldingens ekthet og begrunner nedstengingen med beskyttelse mot mulige zero-day-angrep. Kort tid etter følger TechCrunch, BleepingComputer, Computer Weekly og flere andre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september, 17:41</p>
<p class="timeline__titel">BKA uttaler seg ikke</p>
<p>heise legger til: BKA avviser å uttale seg av etterforskningstaktiske grunner. BSI svarer ikke, og FBI avstår fra en kommentar overfor TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre. 25. september</p>
<p class="timeline__titel">Uttalelse og pressemelding</p>
<p>Kiteworks beskriver nedstengingen som et føre-var-tiltak uten kjent kompromittering. Pressemeldingen oppgir «federal intelligence authorities» som kilde og lister opp datterselskapene som ikke er berørt, deriblant totemo. Produsenten stenger selv ned instanser som driftes av Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lør. 26. september, 04:00 til 10:00</p>
<p class="timeline__titel">Nedstengingsvindu</p>
<p>Vinduet gjelder samtidig over hele verden: 02:00 til 08:00 UTC, i Sydney 12:00 til 18:00, i New York fredag 22:00 til lørdag 04:00.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status lør. 26. september</p>
<p class="timeline__titel">Fortsatt uavklart</p>
<p>Ingen offentlig advisory, ingen CVE-nummer, ingen opplysninger om sårbarheten og ingen rapporter om et gjennomført angrep.</p>
</li>
</ol>

## Det som er kjent

Anbefalingen gjelder globalt; e-posten oppgir tidsvinduet for alle tidssoner fra AEST til PDT. Kiteworks råder til å stenge ned systemene allerede før vinduet begynner, også dersom de ikke er tilgjengelige fra internett.

Nesten alt annet er foreløpig uavklart: Det finnes ingen offentlig sikkerhetsadvisory, ingen CVE-nummer, ingen oppdatering og ingen opplysninger om hvilke produkter eller versjoner som er berørt. Pressemeldingen oppgir «federal intelligence authorities» som kilde, antakelig amerikanske føderale myndigheter; hvilke er ukjent. Per 26. september finnes det ingen oppføring under Security Updates eller i Kiteworks' GitHub-advisorier. Offentlig tilgjengelig er uttalelsen sitert ovenfor og pressemeldingen fra 25. september.

Overfor TechCrunch har Kiteworks-CISO Frank Balonis gitt uttalelsen med samme ordlyd. BKA har avvist å uttale seg overfor heise av etterforskningstaktiske grunner, mens BSI ikke har svart. FBI ønsket ikke å uttale seg overfor TechCrunch, og det forelå ikke svar fra CISA. Ifølge TechCrunch tok en kunde i helsevesenet serveren sin umiddelbart av nettet, med merkbare driftsbegrensninger.

Pressemeldingen avviker fra kunde-e-posten på ett punkt: Den omtaler et nedstengingsvindu på ni timer, mens advisoryet til kundene omtaler seks timer. Ifølge pressemeldingen gjelder anbefalingen bare installasjoner som driftes av kundene selv (On-Premises, AWS, Azure). Ifølge produsenten er datterselskapene Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai og 123FormBuilder ikke berørt.

## Advisoryet til kundene

Kunde-e-posten fra 25. september inneholder, i tillegg til advarselen, en tidsplan per tidssone og en veiledning for klynger. Omregnet til UTC gir tidene det samme vinduet for alle regioner, fra 02:00 til 08:00 UTC.

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

2.  **Opprett sikkerhetskopi:** Ta et øyeblikksbilde av hver node eller en sikkerhetskopi av Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Bare én databasesikkerhetskopi beholdes; hver nye erstatter den forrige.

3.  **Registrer roller:** Under System Setup > Locations viser kolonnen Assigned Roles hvilke noder som har Application-rollen; den primære Application-noden er markert med en stjerne. Noter nodene og IP-adressene deres, ettersom de trengs ved omstart.

4.  **Steng ned i denne rekkefølgen:** først alle noder uten Application-rolle, deretter de øvrige Application-nodene, og til slutt den primære Application-noden. Dette gjøres via fanen Shut Down for den aktuelle noden eller via hypervisorens konsoll (for eksempel VMware eller AWS), dersom Kiteworks-grensesnittet ikke lenger er tilgjengelig.

5.  **Start på nytt i omvendt rekkefølge** via hypervisoren, siden administrasjonskonsollen først er tilgjengelig når nok noder kjører (vedlegg E i Administrator Guide): først den primære Application-noden, deretter de øvrige Application-nodene én om gangen og først når den foregående kjører fullt ut, slik at databaseserverne kan danne et quorum. Deretter Storage-serverne, så de øvrige rollene (Repositories Gateway, Search, SFTP, Antivirus) og til slutt Web-serverne.

6.  **Slå av vedlikeholdsmodus** så snart alle noder i Cluster Health Dashboard på statussiden i administrasjonskonsollen er grønne.

På forespørsel har Kiteworks-støtten dessuten bekreftet at ingen av Kiteworks' datterselskaper er berørt.

## Mulige årsaker: teorier

Så lenge Kiteworks ikke offentliggjør detaljer, forblir årsaken uavklart. Forklaringene nedenfor er hypoteser som kan utledes av de kjente omstendighetene; noen av dem diskuteres også i kommentarene til heise-artikkelen. Ingen av dem er bekreftet.

Tre omstendigheter avgrenser mulighetsrommet. For det første angir advarselen et fast tidsvindu i stedet for en ubestemt nedstenging frem til en oppdatering er tilgjengelig. For det andre skal også systemer som ikke er tilgjengelige fra internett, tas av nettet. For det tredje ligger vinduet på samme tidspunkt over hele verden (02:00 til 08:00 UTC), i stedet for i den lokale natten. En klassisk sårbarhet som kan utnyttes over internett, ville ikke forklart de to første punktene: Det hjelper å koble systemet fra internett, og det helt til oppdateringen foreligger.

### 1. Myndighetene kjenner et planlagt tidspunkt

Politimyndigheter får iblant kjennskap til tidspunktet for en planlagt kampanje på forhånd, for eksempel fra overvåket kommunikasjon i en gjerningsgruppe eller beslaglagt infrastruktur. Masseutnyttelse av produkter for filutveksling skjer typisk i et kort, koordinert tidsvindu, ofte i helger eller på helligdager når færre ansatte er på jobb. Kiteworks er etterfølgeren til Accellion, hvis File Transfer Appliance ble angrepet på nettopp denne måten i 2020 og 2021: Data ble hentet ut gjennom flere sårbarheter, og de berørte organisasjonene ble deretter utsatt for utpressing.

Det avgrensede helgevinduet taler for dette. Mot det taler innvendingen som flere kommentatorer hos heise fremmer: Advarselen gikk til alle kunder, så angriperne vil trolig kjenne til den og kan ganske enkelt utsette angrepet. En utsettelse ville imidlertid gi produsenten tid til å lage en oppdatering.

### 2. Produsenten kjenner ennå ikke sårbarheten selv

Det er også mulig at Kiteworks, utover myndighetenes varsel, ikke har tekniske detaljer og dermed verken kjenner den berørte komponenten eller kan anbefale en oppdatering eller konfigurasjonsendring. Da er nedstengingen det eneste tiltaket som virker uten kjennskap til sårbarheten, og den faste sluttiden et kompromiss som kundene lettere kan akseptere. I heise-kommentarene fremsettes formodningen om at produsenten kan la enkelte systemer være på nett som lokkemat under vinduet for å observere angrepet. Det finnes ingen bevis for dette.

Det støttes av at verken et advisory eller en avbøtende tiltak er nevnt. Mot det taler at Kiteworks ifølge egne opplysninger samarbeider med Mandiant, og at det ved en advarsel fra myndigheter normalt i det minste foreligger indikatorer.

### 3. En allerede plassert bakdør med tidsutløser

Anbefalingen om også å stenge ned interne systemer passer med et scenario der angrepet ikke kommer utenfra, men allerede er forberedt på appliance-enhetene: for eksempel en bakdør fra en tidligere kompromittering som aktiveres på et fast tidspunkt eller oppretter kontakt med en kontrollserver. Et avslått system kan ikke utføre noe på dette tidspunktet.

At tilgjengelighet fra internett ikke spiller noen rolle i dette scenariet, taler for det. Mot det taler at en produsent i et slikt tilfelle heller ville anbefale kontroll for kompromittering og nyinstallasjon enn en omstart etter seks timer.

### 4. Kompromittering hos produsenten

En annen vei til interne systemer er forbindelser som appliance-enheten oppretter til produsenten, for eksempel for oppdateringer, lisenskontroll eller fjernvedlikehold. Hvis en slik kanal er kompromittert, beskytter ikke en brannmur mot innkommende trafikk. I dette scenariet ville nedstengingen gi produsenten et tidsvindu til å rydde opp i egen infrastruktur, bytte nøkler eller sertifikater og først deretter tillate forbindelser igjen.

Det verdensomspennende, ensartede tidspunktet taler for dette, siden det passer med en koordinert handling hos produsenten. Mot det taler at produsenten da heller ville anbefale å blokkere utgående forbindelser enn å stenge systemene helt ned.

### 5. Ledsagende tiltak til en myndighetsaksjon

Til slutt kan det tenkes at myndighetene i samme tidsrom går mot angripernes infrastruktur og vil hindre at de slår til raskt som reaksjon. Det ville forklare det korte tidsvinduet og politimyndighetenes rolle. At BKA avviser en uttalelse av etterforskningstaktiske grunner, tyder på pågående etterforskning, men beviser ikke denne varianten.

### Kritikk av kommunikasjonen

I heise-kommentarene dominerer skepsis, og innvendingene er saklig forståelige: Uten opplysninger om sårbarheten kan man ikke vurdere om frakobling fra internett via brannmur ville vært tilstrekkelig. Et tidsvindu uten varslet oppdatering lar det stå åpent hva som gjelder etter kl. 10:00. Og en advarsel som bare sendes per e-post til kunder, når ikke alle operatører, for eksempel hos partnere, tjenesteleverandører eller etter personalskifter. Uavhengig av hvilken teori som stemmer: Den som drifter Kiteworks, bør kontrollere loggene etter oppstart og følge produsentens kanaler til det foreligger et advisory.

## Kilder

1.  [heise online: Nært forestående zero-day-angrep: KiteWorks presser kunder til å stenge ned servere](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Første melding med utdrag fra kunde-e-posten og tidsvinduet; oppdatering fra 25.09., kl. 17:41, med BKAs svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): engelsk versjon med CISO-ens opprinnelige ordlyd.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): produsentens offisielle kanal, per 26.09.2026 uten oppføring om advarselen.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): produsentens advisory-liste, siste oppføring fra 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): offisielle kunngjøringer, med pressemeldingen om nedstengingen siden 25.09.2026.

6.  [heise-forum: Kommentarer til meldingen](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): leserdiskusjon med innvendingene mot det faste tidsvinduet og nedstengingen av interne systemer, samt teorien om lokkemat.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): advisory om utnyttelsen av Accellion FTA i 2020/2021 med påfølgende utpressing; Accellion er det tidligere navnet på Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): produsentopplysninger om samarbeidet med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO-uttalelse, tidspunktet advarselen ble sendt, reaksjoner fra FBI og CISA samt virkninger hos en kunde.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressemelding)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): offisiell melding fra 25.09.2026 med opplysninger om driftede instanser, versjon 9.5.1 og datterselskapene som ikke er berørt.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): tidsvindu etter regioner og vurdering av tidligere angrep på produkter for filutveksling.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs vurdering av den uvanlige anbefalingen om nedstenging.
