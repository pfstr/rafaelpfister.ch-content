---
title: "Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – Vad som hittills är känt"
navTitle: "Kiteworks-avstängning"
description: "Kiteworks uppmanar sina kunder via e-post att stänga av alla system lördagen den 26.09.2026, från 04:00 till 10:00. Orsaken är en varning från brottsbekämpande myndigheter om en möjlig attack. Totemomail påverkas inte."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min lästid"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-tillverkaren-rekommenderar-avstangning-den-26-september-vad-som-hittills-ar-kant"
featured: "2026-09-27"
warnung: true
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
url: https://rafaelpfister.ch/sv/blog/kiteworks-tillverkaren-rekommenderar-avstangning-den-26-september-vad-som-hittills-ar-kant
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:48:26.357Z
translationReview: automatic
---

# Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – Vad som hittills är känt

Kiteworks uppmanade sina kunder via e-post den 25 september 2026 att stänga av alla Kiteworks-system lördagen den 26 september, från 04:00 till 10:00 (centraleuropeisk tid). Enligt skrivelsen från CISO Frank Balonis har tillverkaren fått information från brottsbekämpande myndigheter om att en attack mot Kiteworks-system kan vara förestående denna helg. Kundsupporten motiverar avstängningen med skydd mot möjliga Zero-Day-attacker. heise online har bekräftat äktheten av meddelandet per telefon med supporten.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Akuthjälp vid omläggning av e-postflödet</p>
<p>Om du behöver hjälp med att dirigera om e-postflödet före avstängningen och återställa det efteråt, använd <a href="https://adeptio.ch/">kontaktformuläret på adeptio.ch</a>. Jag återkommer även med kort varsel.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Uppdatering från 25 september 2026: Uttalande från Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail påverkas inte.</strong> Det är oklart om Kiteworks EPG (Email Protection Gateway) påverkas.</p>
</div>

## Kronologi

Alla tider anges i centraleuropeisk sommartid (CEST). Där ingen tidpunkt anges finns ingen tillförlitlig tidsuppgift.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Meddelande till kunderna</p>
<p>CISO Frank Balonis informerar kunderna via e-post om uppgifter från brottsbekämpande myndigheter om en möjlig attack under helgen och rekommenderar en avstängning i sex timmar. Enligt meddelandet är alla kända sårbarheter åtgärdade i version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Första medierapporterna</p>
<p>heise online rapporterar att Kiteworks support bekräftar meddelandets äkthet och motiverar avstängningen med skydd mot möjliga Zero-Day-attacker. Kort därefter följer TechCrunch, BleepingComputer, Computer Weekly och fler.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september, 17:41</p>
<p class="timeline__titel">BKA uttalar sig inte</p>
<p>heise tillägger: BKA avböjer att uttala sig av utredningstaktiska skäl. BSI svarar inte och FBI avstår från en kommentar till TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Uttalande och pressmeddelande</p>
<p>Kiteworks beskriver avstängningen som en försiktighetsåtgärd utan någon känd kompromettering. Pressmeddelandet anger ”federal intelligence authorities” som källa och räknar upp de dotterbolag som inte påverkas, däribland totemo. Tillverkaren stänger själv av instanser som hostas av Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lör 26 september, 04:00 till 10:00</p>
<p class="timeline__titel">Avstängningsfönster</p>
<p>Fönstret är samtidigt över hela världen: 02:00 till 08:00 UTC, i Sydney 12:00 till 18:00, i New York fredag 22:00 till lördag 04:00.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status lör 26 september</p>
<p class="timeline__titel">Fortfarande oklart</p>
<p>Inget offentligt meddelande, inget CVE-nummer, inga uppgifter om sårbarheten och inga rapporter om en genomförd attack.</p>
</li>
</ol>

## Vad som är känt

Rekommendationen gäller globalt; e-postmeddelandet anger tidsfönstret för alla tidszoner från AEST till PDT. Kiteworks råder att stänga av systemen redan före fönstrets början, även om de inte är åtkomliga från internet.

Nästan allt annat är fortfarande oklart: Det finns inget offentligt säkerhetsmeddelande, inget CVE-nummer, ingen patch och ingen uppgift om vilka produkter eller versioner som påverkas. Pressmeddelandet anger ”federal intelligence authorities” som källa, sannolikt alltså federala amerikanska myndigheter; vilka är okänt. Under Security Updates och i Kiteworks GitHub-meddelanden finns det den 26 september ingen post. Offentliga är uttalandet som citeras ovan och pressmeddelandet från 25 september.

Till TechCrunch har Kiteworks CISO Frank Balonis lämnat uttalandet med samma ordalydelse. BKA har avböjt att uttala sig till heise av utredningstaktiska skäl, medan BSI inte har svarat. FBI ville inte uttala sig till TechCrunch, och CISA hade inte svarat. En kund inom sjukvården tog enligt TechCrunch omedelbart sin server offline, med märkbara driftsbegränsningar som följd.

Pressmeddelandet avviker från kundmejlet på en punkt: Det talar om ett avstängningsfönster på nio timmar, medan meddelandet till kunderna anger sex timmar. Enligt pressmeddelandet gäller rekommendationen endast installationer som drivs av kunden själv (On-Premises, AWS, Azure). Enligt tillverkaren påverkas inte dotterbolagen Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai och 123FormBuilder.

## Meddelandet till kunderna

Kundmejlet från 25 september innehåller, utöver varningen, en tidplan per tidszon och instruktioner för kluster. Om tiderna räknas om till UTC blir det samma fönster på 02:00 till 08:00 UTC för alla regioner.

| Tidszon | Stad | Början | Slut |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Lör 12:00 | Lör 18:00 |
| SGT (UTC+8) | Singapore | Lör 10:00 | Lör 16:00 |
| IDT (UTC+3) | Tel Aviv | Lör 05:00 | Lör 11:00 |
| CEST (UTC+2) | Amsterdam, Zürich | Lör 04:00 | Lör 10:00 |
| BST (UTC+1) | London | Lör 03:00 | Lör 09:00 |
| EDT (UTC−4) | New York | Fre 22:00 | Lör 04:00 |
| CDT (UTC−5) | Chicago | Fre 21:00 | Lör 03:00 |
| MDT (UTC−6) | Denver | Fre 20:00 | Lör 02:00 |
| PDT (UTC−7) | San Francisco | Fre 19:00 | Lör 01:00 |

För kluster med flera servrar anger Kiteworks en fast ordning:

1.  **Aktivera underhållsläge** under System Setup > Maintenance Mode, så att inga användare längre kan komma åt systemet.

2.  **Skapa en säkerhetskopia:** ta en ögonblicksbild av varje nod eller säkerhetskopiera Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Endast en databassäkerhetskopia behålls; varje ny ersätter den föregående.

3.  **Registrera roller:** Under System Setup > Locations visar kolumnen Assigned Roles vilka noder som har rollen Application; den primära Application-noden är markerad med en asterisk. Anteckna noderna och deras IP-adresser, eftersom de behövs vid omstarten.

4.  **Stäng av i denna ordning:** först alla noder utan Application-roll, sedan övriga Application-noder och sist den primära Application-noden. Detta görs via fliken Shut Down för respektive nod eller via hypervisorns konsol (till exempel VMware eller AWS) om Kiteworks-gränssnittet inte längre är åtkomligt.

5.  **Starta om i omvänd ordning** via hypervisorn, eftersom administratörskonsolen först blir åtkomlig när tillräckligt många noder körs (bilaga E i Administrator Guide): först den primära Application-noden, sedan de övriga Application-noderna en i taget och först när den föregående körs fullt ut, så att databasservrarna kan bilda ett kvorum. Därefter Storage-servrarna, sedan de övriga rollerna (Repositories Gateway, Search, SFTP, Antivirus) och sist webbservrarna.

6.  **Inaktivera underhållsläget** så snart alla noder är gröna i Cluster Health Dashboard på administratörskonsolens statussida.

På förfrågan har Kiteworks support dessutom bekräftat att inget av Kiteworks dotterbolag påverkas.

## Möjliga orsaker: teorier

Så länge Kiteworks inte publicerar några detaljer är orsaken oklar. Följande förklaringar är hypoteser som kan härledas från de kända förutsättningarna; några av dem diskuteras även i kommentarerna till heise-artikeln. Ingen av dem är bekräftad.

Tre omständigheter begränsar möjligheterna. För det första anger varningen ett fast tidsfönster i stället för en obegränsad avstängning fram till en patch. För det andra ska även system som inte är åtkomliga från internet tas offline. För det tredje ligger fönstret globalt vid samma tidpunkt (02:00 till 08:00 UTC), i stället för under respektive lokal natt. En klassisk sårbarhet som kan utnyttjas via internet skulle inte förklara de två första punkterna: Det hjälper att koppla bort systemet från internet, och då så länge tills patchen finns tillgänglig.

### 1. Myndigheterna känner till en planerad tidpunkt

Brottsbekämpande myndigheter får ibland i förväg kännedom om tidpunkten för en planerad kampanj, exempelvis genom övervakad kommunikation från en gärningsgrupp eller beslagtagen infrastruktur. Massutnyttjande av produkter för filutbyte genomförs vanligtvis inom ett kort, samordnat tidsfönster, ofta under helger eller helgdagar när mindre personal är i tjänst. Kiteworks är efterföljaren till Accellion, vars File Transfer Appliance attackerades på just detta sätt 2020 och 2021: data exfiltrerades via flera sårbarheter och de drabbade organisationerna utpressades därefter.

Det snävt avgränsade fönstret under helgen talar för detta. Emot talar invändningen som flera kommentatorer på heise framför: Varningen gick till alla kunder, så angriparna torde känna till den och helt enkelt kunna skjuta upp attacken. En uppskjutning skulle dock ge tillverkaren tid att ta fram en patch.

### 2. Tillverkaren känner ännu inte till sårbarheten

Det är också möjligt att Kiteworks utöver myndigheternas tips saknar tekniska detaljer, alltså varken känner till den berörda komponenten eller kan rekommendera en patch eller konfigurationsändring. Då är avstängningen den enda åtgärd som fungerar utan kännedom om sårbarheten, och det fasta slutet är en kompromiss som kunderna lättare accepterar. I heise-kommentarerna framförs antagandet att tillverkaren kan låta enskilda system vara online som lockbete under fönstret för att observera attacken. Det finns inga belägg för detta.

För detta talar att varken ett meddelande eller en åtgärd för begränsning nämns. Emot talar att Kiteworks enligt egen uppgift samarbetar med Mandiant och att det vid en varning från myndigheter normalt åtminstone finns indikatorer tillgängliga.

### 3. En redan placerad bakdörr med tidsutlösare

Rekommendationen att även stänga av interna system passar ett scenario där attacken inte kommer utifrån, utan redan har förberetts på apparaterna: exempelvis en bakdörr från en tidigare kompromettering som aktiveras vid en bestämd tidpunkt eller kontaktar en kontrollserver. Ett avstängt system kan inte utföra något vid den tidpunkten.

För detta talar att åtkomlighet från internet inte spelar någon roll i detta scenario. Emot talar att en tillverkare i detta fall snarare skulle rekommendera en kontroll för kompromettering och en ominstallation än en omstart efter sex timmar.

### 4. Kompromettering hos tillverkaren

En annan väg som når interna system är anslutningar som upprättas från apparaten till tillverkaren, till exempel för uppdateringar, licenskontroll eller fjärrunderhåll. Om en sådan kanal har komprometterats skyddar en brandvägg inte mot inkommande trafik. I detta scenario skulle avstängningen ge tillverkaren ett fönster för att rensa sin egen infrastruktur, byta nycklar eller certifikat och först därefter åter tillåta anslutningar.

Den globt enhetliga tidpunkten, som passar en samordnad åtgärd hos tillverkaren, talar för detta. Emot talar att tillverkaren då snarare skulle rekommendera att blockera utgående anslutningar än att helt stänga av systemen.

### 5. En stödåtgärd till en myndighetsinsats

Slutligen är det tänkbart att myndigheterna agerar mot angriparnas infrastruktur under samma tidsperiod och vill förhindra att de slår till snabbt som reaktion. Det skulle förklara det korta fönstret och brottsbekämpningens roll. Att BKA avböjer att uttala sig av utredningstaktiska skäl tyder på pågående utredningar, men bevisar inte denna variant.

### Kritik mot kommunikationen

I heise-kommentarerna dominerar skepsis, och invändningarna är sakligt begripliga: Utan uppgifter om sårbarheten går det inte att bedöma om det hade räckt att koppla bort systemet från internet med en brandvägg. Ett tidsfönster utan aviserad patch lämnar öppet vad som gäller efter klockan 10:00. Och en varning som endast skickas via e-post till kunder når inte alla operatörer, exempelvis hos partner, tjänsteleverantörer eller efter personalbyten. Oavsett vilken teori som stämmer: Den som driver Kiteworks bör kontrollera loggarna efter återstarten och bevaka tillverkarens kanaler tills ett meddelande finns tillgängligt.

## Källor

1.  [heise online: Förestående Zero-Day-attack: KiteWorks uppmanar kunder att stänga av servrar](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Första rapporten med utdrag ur kundmejlet och tidsfönstret; uppdatering från 25.09., 17:41, med BKA:s svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): engelsk version med CISO:ns ursprungliga ordalydelse.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): tillverkarens officiella kanal, utan post om varningen per 26.09.2026.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): tillverkarens lista över meddelanden, senaste posten från 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): officiella meddelanden, sedan 25.09.2026 med pressmeddelandet om avstängningen.

6.  [heise-forum: Kommentarer till artikeln](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): läsardiskussion med invändningar mot det fasta tidsfönstret och avstängningen av interna system samt teorin om lockbete.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): meddelande om utnyttjandet av Accellion FTA 2020/2021 med efterföljande utpressning; Accellion är Kiteworks tidigare namn.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): tillverkarens uppgift om samarbetet med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): uttalande från CISO:n, tidpunkt för utskicket av varningen, reaktioner från FBI och CISA samt effekter hos en kund.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressmeddelande)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): officiellt meddelande från 25.09.2026 med uppgifter om hostade instanser, version 9.5.1 och dotterbolagen som inte påverkas.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): tidsfönster per region och kontext om tidigare attacker mot produkter för filutbyte.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs bedömning av den ovanliga avstängningsrekommendationen.
