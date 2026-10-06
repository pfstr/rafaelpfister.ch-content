---
title: "Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – vad som hittills är känt"
navTitle: "Kiteworks-avstängning"
description: "Kiteworks uppmanar sina kunder via e-post att stänga ned alla system lördagen den 26.09.2026 från 04:00 till 10:00. Orsaken är en varning från brottsbekämpande myndigheter om en möjlig attack. Sedan den 27.09. har rekommendationen upphävts; det finns ingen CVE eller ny patch. Totemomail påverkas inte."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min. läsning"
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
slug: "kiteworks-tillverkaren-rekommenderar-avstangning-den-26-september-vad-som-hittills-ar-kant"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T10:02:21.357Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/kiteworks-tillverkaren-rekommenderar-avstangning-den-26-september-vad-som-hittills-ar-kant
---

# Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – vad som hittills är känt

Kiteworks uppmanade sina kunder via e-post den 25 september 2026 att stänga ned alla Kiteworks-system lördagen den 26 september från 04:00 till 10:00 (centraleuropeisk tid). Enligt skrivelsen från CISO Frank Balonis har tillverkaren fått information från brottsbekämpande myndigheter om att en attack mot Kiteworks-system kan vara nära förestående under denna helg. Kundsupporten motiverar avstängningen med skydd mot möjliga Zero-Day-attacker. heise online har telefonledes bekräftat med supporten att meddelandet är äkta.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Akuthjälp vid omläggning av e-postflödet</p>
<p>Om du behöver hjälp med att dirigera om e-postflödet före avstängningen och återställa det efteråt, använd <a href="https://adeptio.ch/">kontaktformuläret på adeptio.ch</a>. Jag återkommer även med kort varsel.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Uppdatering den 28 september 2026: Kiteworks upphäver rekommendationen om avstängning</p>
<p>Kiteworks har lagt till en notis i pressmeddelandet: Sedan den 27 september gäller rekommendationen om avstängning inte längre för några kunder.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Den som ännu inte har startat sina system igen kan nu göra det. Den som själv driver Advanced Forms ska kontakta Kiteworks-supporten före omstarten. De instanser som hostas av Kiteworks är åter i drift. Det finns fortfarande inget CVE-nummer, ingen ny version utöver 9.5.1, inga indikatorer på en kompromettering och ingen uppgift om huruvida ett angrepp har försökt genomföras eller vad som låg bakom varningen. Sidan för säkerhetsuppdateringar och GitHub-advisories är oförändrade.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Uppdatering den 25 september 2026: Uttalande från Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail påverkas inte.</strong> Det är oklart om Kiteworks EPG (Email Protection Gateway) påverkas.</p>
</div>

## Kronologi

Alla tider anges i centraleuropeisk sommartid (CEST). Där ingen tid anges finns ingen tillförlitlig tidsuppgift.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre, 25 september</p>
<p class="timeline__titel">Meddelande till kunderna</p>
<p>CISO Frank Balonis informerar kunderna via e-post om uppgifter från brottsbekämpande myndigheter om en möjlig attack denna helg och rekommenderar avstängning i sex timmar. Enligt meddelandet är alla kända sårbarheter åtgärdade i version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre, 25 september</p>
<p class="timeline__titel">Första medierapporterna</p>
<p>heise online rapporterar att Kiteworks-supporten bekräftar att meddelandet är äkta och motiverar avstängningen med skydd mot möjliga Zero-Day-attacker. Kort därefter följer TechCrunch, BleepingComputer, Computer Weekly och fler.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre, 25 september, 17:41</p>
<p class="timeline__titel">BKA uttalar sig inte</p>
<p>heise tillägger: BKA avböjer ett uttalande av utredningstaktiska skäl. BSI svarar inte och FBI avstår från en kommentar till TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre, 25 september</p>
<p class="timeline__titel">Uttalande och pressmeddelande</p>
<p>Kiteworks beskriver avstängningen som en försiktighetsåtgärd utan någon känd kompromettering. Pressmeddelandet anger ”federal intelligence authorities” som källa och listar de dotterbolag som inte påverkas, däribland totemo. Instanser som hostas av Kiteworks stänger tillverkaren själv ned.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lör, 26 september, 04:00 till 10:00</p>
<p class="timeline__titel">Avstängningsfönster</p>
<p>Fönstret är samtidigt över hela världen: 02:00 till 08:00 UTC, i Sydney 12:00 till 18:00 och i New York fredag 22:00 till lördag 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lör, 26 september, 10:00</p>
<p class="timeline__titel">Slutet på fönstret</p>
<p>Fönstret som anges i kundmejlet upphör. Det formella upphävandet av rekommendationen följer den 27 september.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sön, 27 september</p>
<p class="timeline__titel">Rekommendationen upphävs</p>
<p>Kiteworks lägger till i pressmeddelandet: Rekommendationen om avstängning är upphävd för alla kunder och systemen får köras igen. De hostade instanserna är åter i drift. Kunder med egen drift av Advanced Forms ska kontakta supporten.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status mån, 28 september</p>
<p class="timeline__titel">Fortfarande oklart</p>
<p>Inget offentligt meddelande, inget CVE-nummer, ingen ny version, inga indikatorer, inga uppgifter om sårbarheten och inga rapporter om en genomförd eller försökt attack.</p>
</li>
</ol>

## Vad som är känt

Rekommendationen gäller globalt; e-postmeddelandet anger tidsfönstret för alla tidszoner från AEST till PDT. Kiteworks rekommenderar att systemen stängs ned redan före fönstrets början, även om de inte är åtkomliga från internet.

Nästan allt annat är fortfarande oklart: Det finns inget offentligt säkerhetsmeddelande, inget CVE-nummer, ingen patch och ingen uppgift om vilka produkter eller versioner som påverkas. Pressmeddelandet anger ”federal intelligence authorities” som källa, sannolikt alltså amerikanska federala myndigheter; vilka är okänt. Under Security Updates och i Kiteworks GitHub-advisories finns per den 28 september ingen post; den senaste GitHub-posten är från den 27 maj 2026. Offentliga är uttalandet som citeras ovan och pressmeddelandet från den 25 september.

Till TechCrunch har Kiteworks CISO Frank Balonis lämnat uttalandet med samma ordalydelse. BKA har avböjt att uttala sig till heise av utredningstaktiska skäl, och BSI har inte svarat. FBI ville inte uttala sig till TechCrunch och en talesperson för CISA ville inte uttala sig offentligt. En kund inom hälso- och sjukvården tog enligt TechCrunch omedelbart sin server offline, med märkbara begränsningar i verksamheten: läkare kunde tidvis bara nå sina patienter med fördröjning. Enligt en säkerhetsforskare som TechCrunch citerar är minst 1 000 Kiteworks-system åtkomliga från internet; BornCity talar om fler än 1 000 organisationer som fått varningen.

Pressmeddelandet skiljer sig från kundmejlet på en punkt: Det talar om ett avstängningsfönster på nio timmar, medan kundmeddelandet anger sex timmar. Enligt pressmeddelandet gäller rekommendationen endast installationer som drivs i egen regi (on-premises, AWS, Azure). Enligt tillverkaren påverkas inte dotterbolagen Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai och 123FormBuilder.

## Meddelandet till kunderna

Kundmejlet från den 25 september innehåller, utöver varningen, ett schema per tidszon och instruktioner för kluster. Om tiderna räknas om till UTC blir resultatet samma fönster för alla regioner: 02:00 till 08:00 UTC.

| Tidszon | Stad | Start | Slut |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Lör, 12:00 | Lör, 18:00 |
| SGT (UTC+8) | Singapore | Lör, 10:00 | Lör, 16:00 |
| IDT (UTC+3) | Tel Aviv | Lör, 05:00 | Lör, 11:00 |
| CEST (UTC+2) | Amsterdam, Zürich | Lör, 04:00 | Lör, 10:00 |
| BST (UTC+1) | London | Lör, 03:00 | Lör, 09:00 |
| EDT (UTC−4) | New York | Fre, 22:00 | Lör, 04:00 |
| CDT (UTC−5) | Chicago | Fre, 21:00 | Lör, 03:00 |
| MDT (UTC−6) | Denver | Fre, 20:00 | Lör, 02:00 |
| PDT (UTC−7) | San Francisco | Fre, 19:00 | Lör, 01:00 |

För kluster med flera servrar anger Kiteworks en fast ordning:

1.  **Aktivera underhållsläge** under System Setup > Maintenance Mode, så att inga användare längre kan komma åt systemet.

2.  **Skapa en säkerhetskopia:** ta en snapshot av varje nod eller en säkerhetskopia av Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Endast en databassäkerhetskopia sparas; varje ny ersätter den föregående.

3.  **Dokumentera roller:** Under System Setup > Locations visar kolumnen Assigned Roles vilka noder som har Application-rollen; den primära Application-noden är markerad med en stjärna. Notera noderna och deras IP-adresser, eftersom de behövs för omstarten.

4.  **Stäng ned i denna ordning:** först alla noder utan Application-roll, sedan de övriga Application-noderna och sist den primära Application-noden. Det görs via fliken Shut Down för respektive nod eller via hypervisorns konsol (till exempel VMware eller AWS) om Kiteworks-gränssnittet inte längre är åtkomligt.

5.  **Starta om i omvänd ordning** via hypervisorn, eftersom administratörskonsolen först blir åtkomlig när tillräckligt många noder körs (bilaga E i Administrator Guide): först den primära Application-noden, sedan de övriga Application-noderna en i taget och först när den föregående körs fullt ut, så att databasservrarna kan bilda ett kvorum. Därefter Storage-servrarna, sedan övriga roller (Repositories Gateway, Search, SFTP, Antivirus) och sist webbservrarna.

6.  **Inaktivera underhållsläge** när alla noder i Cluster Health Dashboard på administratörskonsolens statussida är gröna.

På förfrågan har Kiteworks-supporten dessutom bekräftat att inget av Kiteworks dotterbolag påverkas.

## Möjliga orsaker: teorier

Så länge Kiteworks inte publicerar några detaljer är orsaken oklar. Följande förklaringar är hypoteser som kan härledas från de kända omständigheterna; några av dem diskuteras också i kommentarerna till heise-artikeln. Ingen av dem är bekräftad.

Tre omständigheter begränsar utrymmet. För det första anger varningen ett fast tidsfönster i stället för en obegränsad avstängning fram till en patch. För det andra ska även system som inte är åtkomliga från internet kopplas ned. För det tredje infaller fönstret vid samma tidpunkt globalt (02:00 till 08:00 UTC), i stället för under den lokala natten. En klassisk sårbarhet som kan utnyttjas via internet skulle inte förklara de två första punkterna: Det hjälper att koppla bort systemet från internet, och det tills en patch finns tillgänglig.

### 1. Myndigheterna känner till en planerad tidpunkt

Brottsbekämpande myndigheter får ibland kännedom om tidpunkten för en planerad kampanj i förväg, till exempel genom övervakad kommunikation från en gärningsgrupp eller från beslagtagen infrastruktur. Massutnyttjanden av filöverföringsprodukter sker typiskt i ett kort, samordnat fönster, ofta på helger eller helgdagar när färre medarbetare är i tjänst. Kiteworks är efterföljaren till Accellion, vars File Transfer Appliance attackerades på just detta sätt 2020 och 2021, då tillskrivet gruppen Clop: Data hämtades ut genom flera sårbarheter, varefter de drabbade organisationerna utpressades.

Det snävt avgränsade fönstret under helgen talar för detta. Emot talar invändningen som flera kommentatorer på heise framför: Varningen gick till alla kunder, så angriparna torde känna till den och kan helt enkelt skjuta upp attacken. En uppskjutning skulle dock ge tillverkaren tid att ta fram en patch.

### 2. Tillverkaren känner ännu inte själv till sårbarheten

Det är också möjligt att Kiteworks, utöver myndigheternas uppgifter, inte har några tekniska detaljer, alltså varken känner till den berörda komponenten eller kan rekommendera en patch eller konfigurationsändring. Då är avstängning den enda åtgärd som fungerar utan kännedom om sårbarheten, och det fasta slutet en kompromiss som kunderna lättare accepterar. I heise-kommentarerna framförs misstanken att tillverkaren under fönstret kan låta enskilda system vara online som lockbete för att observera attacken. Det finns inga belägg för detta.

För detta talar att varken ett meddelande eller en åtgärd för riskminskning nämns. Emot talar att Kiteworks enligt egna uppgifter samarbetar med Mandiant och att indikatorer normalt åtminstone finns tillgängliga vid en varning från myndigheter.

### 3. En redan installerad bakdörr med tidsutlösare

Rekommendationen att även stänga ned interna system passar ett scenario där attacken inte kommer utifrån utan redan har förberetts på apparaterna: exempelvis en bakdörr från en tidigare kompromettering som aktiveras vid en bestämd tidpunkt eller tar kontakt med en kontrollserver. Ett avstängt system kan inte utföra något vid den tidpunkten.

För detta talar att åtkomlighet från internet inte spelar någon roll i detta scenario. Emot talar att en tillverkare i ett sådant fall snarare skulle rekommendera en kontroll av kompromettering och ominstallation än en omstart efter sex timmar.

### 4. Kompromettering hos tillverkaren

En annan väg som når interna system är anslutningar som upprättas från apparaten till tillverkaren, till exempel för uppdateringar, licenskontroll eller fjärrunderhåll. Om en sådan kanal är komprometterad skyddar en brandvägg inte mot inkommande trafik. I detta scenario skulle avstängningen ge tillverkaren ett fönster för att sanera sin egen infrastruktur, byta nycklar eller certifikat och först därefter tillåta anslutningar igen.

Den enhetliga tidpunkten globalt talar för detta, eftersom den passar en samordnad åtgärd hos tillverkaren. Emot talar att tillverkaren då snarare skulle rekommendera att blockera utgående anslutningar än att stänga av systemen helt.

### 5. Kompletterande åtgärd till en myndighetsinsats

Slutligen är det tänkbart att myndigheterna agerar mot angriparnas infrastruktur under samma tidsperiod och vill förhindra att de slår till snabbt som reaktion. Det skulle förklara det korta fönstret och de brottsbekämpande myndigheternas roll. Att BKA avböjer ett uttalande av utredningstaktiska skäl tyder på pågående utredningar, men bevisar inte denna variant.

### Kritik mot kommunikationen

I heise-kommentarerna dominerar skepsis, och invändningarna är sakligt förståeliga: Utan uppgifter om sårbarheten går det inte att bedöma om det hade räckt att koppla bort internet via brandvägg. Ett tidsfönster utan en aviserad patch lämnar öppet vad som gäller efter 10:00. Och en varning som endast skickas via e-post till kunder når inte alla operatörer, till exempel hos partner, tjänsteleverantörer eller efter personalbyten. Oavsett vilken teori som stämmer bör den som driver Kiteworks kontrollera loggarna efter omstarten och följa tillverkarens kanaler tills ett meddelande finns tillgängligt.

## Efter fönstret: Vad operatörer kan göra nu

Kiteworks upphävde rekommendationen om avstängning den 27 september, men har inte publicerat några tekniska detaljer. Det går därför inte att bedöma om eller hur risken har undanröjts. Vid omstarten och därefter är följande steg lämpliga:

1.  **Kontrollera versionen:** Körs version 9.5.1 på alla noder? Enligt tillverkaren är alla kända sårbarheter åtgärdade i den.

2.  **Kontrollera klustrets status:** I Cluster Health Dashboard ska alla noder vara gröna och underhållsläget vara inaktiverat.

3.  **Utvärdera loggar:** Granska inloggningar, administratörsåtgärder och ovanliga filnedladdningar kring avstängningsfönstret, särskilt för system som inte stängdes ned eller stängdes ned sent.

4.  **Begränsa åtkomligheten:** Där det är möjligt, blockera åtkomst från internet till administrationsgränssnittet och tillåt endast nödvändiga tjänster.

5.  **Advanced Forms:** Den som driver modulen själv bör stämma av omstarten med Kiteworks-supporten i förväg.

6.  **Följ kanalerna:** Kiteworks Security Updates, GitHub-advisories, Newsroom och kundmejl tills ett meddelande med tekniska detaljer finns tillgängligt.

## Källor

1.  [heise online: Förestående Zero-Day-attack: KiteWorks uppmanar kunder att stänga ned servrar](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Första rapporten med utdrag ur kundmejlet och tidsfönstret; uppdatering från 25.09. kl. 17:41 med BKA:s svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): engelsk version med CISO:ns ursprungliga ordalydelse.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): tillverkarens officiella kanal, utan någon post om varningen per 28.09.2026.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): tillverkarens lista över advisories, där den senaste posten per 28.09.2026 är från 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): officiella meddelanden, med pressmeddelandet om avstängningen sedan 25.09.2026.

6.  [heise-forum: Kommentarer till artikeln](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): läsardiskussion med invändningarna mot det fasta tidsfönstret och avstängningen av interna system samt lockbeteorin.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): meddelande om utnyttjandet av Accellion FTA 2020/2021 med efterföljande utpressning; Accellion är Kiteworks tidigare namn.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): tillverkarens uppgift om samarbetet med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO:ns uttalande, tidpunkten då varningen skickades, reaktioner från FBI och CISA (tillägg), konsekvenser för en kund och antalet system som är åtkomliga från internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressmeddelande)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): officiellt meddelande från 25.09.2026 med uppgifter om hostade instanser, version 9.5.1 och de dotterbolag som inte påverkas; kompletterat med notisen från 27.09.2026 om att rekommendationen om avstängning har upphävts.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): tidsfönster per region och inordning av tidigare attacker mot filöverföringsprodukter.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs bedömning av den ovanliga rekommendationen om avstängning.

13.  [BornCity: Kiteworks: Fler än 1 000 organisationer ska stänga ned servrar](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): antal underrättade organisationer och branscher i den tyskspråkiga regionen.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): inordning av Clops Accellion-attacker 2020/2021 och citat från watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): rapport från 28.09.2026 om att rekommendationen upphävts och driften av de hostade instanserna.
