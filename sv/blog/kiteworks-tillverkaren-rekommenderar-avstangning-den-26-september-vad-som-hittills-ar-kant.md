---
title: "Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – vad som hittills är känt"
navTitle: "Avstängning av Kiteworks"
description: "Kiteworks uppmanade sina kunder att stänga av alla system lördagen den 26.09.2026 från 04:00 till 10:00. Slutrapport: Under avstängningen hittade och åtgärdade tillverkaren en kritisk sårbarhet utan CVE; den 30.09. följde 125 advisories, däribland CVE-2026-54154 (CVSS 10.0) i Email Protection Gateway. Totemomail påverkas inte."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min. lästid"
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
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:57:07.964Z
translationReview: required
url: https://rafaelpfister.ch/sv/blog/kiteworks-tillverkaren-rekommenderar-avstangning-den-26-september-vad-som-hittills-ar-kant
---

# Kiteworks: Tillverkaren rekommenderar avstängning den 26 september – vad som hittills är känt

Kiteworks uppmanade sina kunder via e-post den 25 september 2026 att stänga av alla Kiteworks-system lördagen den 26 september mellan 04:00 och 10:00 (centraleuropeisk tid). Enligt brevet från CISO Frank Balonis hade tillverkaren fått information från brottsbekämpande myndigheter om att en attack mot Kiteworks-system kunde vara förestående den här helgen. Kundsupporten motiverade avstängningen med skydd mot möjliga zero-day-attacker. heise online har per telefon bekräftat med supporten att meddelandet är äkta.

<div class="update-hinweis">
<p class="update-hinweis__titel">Slutrapport från den 7 oktober 2026</p>
<p>Ur tillverkarens perspektiv är incidenten avslutad. Rekommendationen om avstängning gäller inte längre sedan den 27 september och någon attack mot Kiteworks- eller kundsystem har fortfarande inte blivit känd. De viktigaste resultaten:</p>
<ul>
<li><strong>Kritisk sårbarhet hittades under avstängningen:</strong> Enligt pressmeddelandet från den 28 september upptäckte Kiteworks vid analysen tillsammans med federala myndigheter en tidigare okänd kritisk sårbarhet i en funktion som är aktiverad hos mindre än 1 % av kunderna. Tillverkaren utvecklade och rullade ut en fix under avstängningsfönstret och aktiverade dessutom ett skyddslager i alla miljöer. Vilken funktion som berördes har inte offentliggjorts; någon CVE-nummer finns fortfarande inte.</li>
<li><strong>125 advisories den 30 september:</strong> Två dagar senare publicerade Kiteworks 125 säkerhetsadvisories på GitHub för Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) och MFT Server (3); 12 av dem var kritiska och 49 höga. Alla är åtgärdade i versioner upp till och med 9.5.1, och de flesta rapporterades via bug bounty-programmet på YesWeHack. Enligt nuvarande uppgifter har de inget att göra med sårbarheten från avstängningsfönstret.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> Den allvarligaste sårbarheten berör Email Protection Gateway före version 9.4.1. En oautentiserad angripare kan köra kod med root-rättigheter via publikt åtkomliga ändpunkter. 10 av de 12 kritiska advisories berör Email Protection Gateway.</li>
<li><strong>Ingen känd exploatering:</strong> Det finns inga rapporter om attacker för någon av sårbarheterna; i CISA KEV-katalogen fanns per den 7 oktober ingen Kiteworks-post från 2026. Enligt BleepingComputer räknar Shadowserver knappt 400 Kiteworks-instanser som är åtkomliga från internet.</li>
<li><strong>Totemomail:</strong> Förekommer inte i något av advisories och påverkades enligt tillverkaren inte av avstängningen.</li>
</ul>
<p><strong>Åtgärdsbehov:</strong> Den som driver Kiteworks själv bör uppdatera alla komponenter till version 9.5.1; Email Protection Gateway har prioritet. Detaljerna finns i avsnittet <a href="#abschlussbericht">Slutrapport</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Uppdatering från den 28 september 2026: Kiteworks upphäver rekommendationen om avstängning</p>
<p>Kiteworks har kompletterat pressmeddelandet med en upplysning: Sedan den 27 september gäller rekommendationen om avstängning inte längre för några kunder.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Den som ännu inte har startat sina system igen kan nu göra det. Den som driver Advanced Forms själv ska kontakta Kiteworks-supporten före omstarten. Instanserna som hostas av Kiteworks körs igen. Det finns fortfarande inget CVE-nummer, ingen ny version utöver 9.5.1, inga indikatorer på kompromettering och ingen uppgift om huruvida en attack försöktes eller vad som låg bakom varningen. Sidan med säkerhetsuppdateringar och GitHub-advisories är oförändrade.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Uppdatering från den 25 september 2026: Uttalande från Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail påverkas inte.</strong> Det är oklart om Kiteworks EPG (Email Protection Gateway) påverkas.</p>
</div>

## Kronologi

Alla tider är centraleuropeisk sommartid (CEST). Där ingen tid anges finns ingen tillförlitlig tidsuppgift.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Advisory till kunderna</p>
<p>CISO Frank Balonis informerar kunderna via e-post om information från brottsbekämpande myndigheter om en möjlig attack denna helg och rekommenderar en avstängning i sex timmar. Enligt advisory är alla kända sårbarheter åtgärdade i version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Första medierapporterna</p>
<p>heise online rapporterar, Kiteworks-supporten bekräftar meddelandets äkthet och motiverar avstängningen med skydd mot möjliga zero-day-attacker. Kort därefter följer TechCrunch, BleepingComputer, Computer Weekly och andra.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september, 17:41</p>
<p class="timeline__titel">BKA uttalar sig inte</p>
<p>heise tillägger: BKA avböjer ett uttalande av utredningstaktiska skäl. BSI svarar inte och FBI avstår från en kommentar till TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fre 25 september</p>
<p class="timeline__titel">Uttalande och pressmeddelande</p>
<p>Kiteworks beskriver avstängningen som en försiktighetsåtgärd utan känd kompromettering. Pressmeddelandet anger ”federal intelligence authorities” som källa och listar de opåverkade dotterbolagen, däribland totemo. Tillverkaren stänger själv av instanser som hostas av Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Lör 26 september, 04:00 till 10:00</p>
<p class="timeline__titel">Avstängningsfönster</p>
<p>Fönstret infaller samtidigt över hela världen: 02:00 till 08:00 UTC, i Sydney 12:00 till 18:00 och i New York från fredag 22:00 till lördag 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lör 26 september, 10:00</p>
<p class="timeline__titel">Fönstrets slut</p>
<p>Fönstret som anges i kundmejlet upphör. Det formella upphävandet av rekommendationen följer den 27 september.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sön 27 september</p>
<p class="timeline__titel">Rekommendationen upphävs</p>
<p>Kiteworks kompletterar pressmeddelandet: Rekommendationen om avstängning upphävs för alla kunder och systemen får köras igen. De hostade instanserna är åter i drift. Kunder med egen drift av Advanced Forms ska kontakta supporten.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mån 28 september</p>
<p class="timeline__titel">Kritisk sårbarhet hittad och åtgärdad</p>
<p>Kiteworks meddelar i ett ytterligare pressmeddelande att en tidigare okänd kritisk sårbarhet hittades under avstängningen i arbetet med federala myndigheter. Den berör en funktion som är aktiverad hos mindre än 1 % av kunderna. Fixen och ett ytterligare skyddslager har rullats ut, och det finns inga tecken på kompromettering. Tillverkaren anger inte funktionen eller något CVE-nummer.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ons 30 september, från 18:38</p>
<p class="timeline__titel">125 säkerhetsadvisories på GitHub</p>
<p>Kiteworks publicerar 125 advisories för Core, Email Protection Gateway, Secure Data Forms och MFT Server, samtliga åtgärdade upp till version 9.5.1. Den allvarligaste sårbarheten är CVE-2026-54154 i Email Protection Gateway före 9.4.1 (CVSS 10.0, kodexekvering med root-rättigheter utan inloggning).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Tors 1 oktober</p>
<p class="timeline__titel">Medierapporter och MS-ISAC-advisory</p>
<p>BleepingComputer, SecurityOnline och andra rapporterar om advisories; MS-ISAC (Center for Internet Security) utfärdar ett eget advisory om CVE-2026-54154. Någon exploatering är inte känd.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Status ons 7 oktober</p>
<p class="timeline__titel">Avslutning</p>
<p>Inga rapporter om en genomförd eller försökt attack och ingen Kiteworks-post i CISA KEV-katalogen. Det är fortfarande oklart vilken funktion som berördes, vilket CVE-nummer sårbarheten från avstängningsfönstret har och bakgrunden till myndighetsvarningen.</p>
</li>
</ol>

## Vad som är känt

Rekommendationen gäller globalt; e-postmeddelandet anger tidsfönstret för alla tidszoner från AEST till PDT. Kiteworks rekommenderar att systemen stängs av redan före fönstrets början, även om de inte är åtkomliga från internet.

Fram till den 28 september var nästan allt annat oklart: Det fanns inget offentligt säkerhetsadvisory, inget CVE-nummer, ingen patch och inga uppgifter om vilka produkter eller versioner som berördes. Pressmeddelandet anger ”federal intelligence authorities” som källa, förmodligen alltså amerikanska federala myndigheter; vilka är fortfarande okänt. I Kiteworks GitHub-advisories kom den senaste posten dittills den 27 maj 2026; advisories från den 30 september sammanfattas i avsnittet [Slutrapport](#abschlussbericht).

Till TechCrunch lämnade Kiteworks CISO Frank Balonis samma uttalande ordagrant. BKA avböjde ett uttalande till heise av utredningstaktiska skäl och BSI svarade inte. FBI ville inte uttala sig till TechCrunch och en talesperson för CISA ville inte uttala sig offentligt. En kund inom hälso- och sjukvården tog enligt TechCrunch omedelbart sin server offline, med märkbara driftbegränsningar: Läkare kunde under en period bara nå sina patienter med fördröjning. Enligt en säkerhetsforskare som citeras av TechCrunch är minst 1 000 Kiteworks-system åtkomliga från internet; BornCity talar om mer än 1 000 organisationer som fick varningen.

Pressmeddelandet avviker på en punkt från kundmejlet: Det talar om ett avstängningsfönster på nio timmar, medan advisory till kunderna anger sex timmar. Enligt pressmeddelandet gäller rekommendationen endast installationer som drivs av kunden själv (on-premises, AWS, Azure). Enligt tillverkaren berörs inte dotterbolagen Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai och 123FormBuilder.

## Advisory till kunderna

Kundmejlet från den 25 september innehåller utöver varningen en tidsplan per tidszon och en instruktion för kluster. Om tiderna räknas om till UTC blir fönstret detsamma i alla regioner: 02:00 till 08:00 UTC.

| Tidszon | Stad | Start | Slut |
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

1.  **Aktivera underhållsläge** under System Setup > Maintenance Mode så att inga användare längre kan komma åt systemet.

2.  **Skapa säkerhetskopia:** ta en snapshot av varje nod eller säkerhetskopiera Kiteworks-databasen (System Setup > Cluster Configuration > System Configuration). Endast en säkerhetskopia av databasen sparas; varje ny ersätter den föregående.

3.  **Registrera roller:** Under System Setup > Locations visar kolumnen Assigned Roles vilka noder som har Application-rollen; den primära Application-noden är markerad med en stjärna. Notera noderna och deras IP-adresser, de behövs för omstarten.

4.  **Stäng av i denna ordning:** först alla noder utan Application-roll, därefter övriga Application-noder och sist den primära Application-noden. Detta görs via fliken Shut Down för respektive nod eller via hypervisorns konsol (till exempel VMware eller AWS) om Kiteworks-gränssnittet inte längre är åtkomligt.

5.  **Starta om i omvänd ordning** via hypervisorn, eftersom administratörskonsolen blir åtkomlig först när tillräckligt många noder körs (bilaga E i Administrator Guide): först den primära Application-noden, sedan de övriga Application-noderna en i taget och först när föregående körs helt, så att databasservrarna kan bilda ett kvorum. Därefter Storage-servrarna, sedan övriga roller (Repositories Gateway, Search, SFTP, Antivirus) och sist webbservrarna.

6.  **Inaktivera underhållsläget** så snart alla noder är gröna i Cluster Health Dashboard på administratörskonsolens statussida.

På förfrågan bekräftade Kiteworks-supporten dessutom att inget av Kiteworks dotterbolag påverkas.

## Möjliga orsaker: teorier

Detta avsnitt skrevs före den 28 september; bedömningen enligt dagens läge finns i [Slutrapport](#abschlussbericht). Följande förklaringar är hypoteser som kan härledas ur de kända omständigheterna; vissa diskuteras även i kommentarerna till heise-rapporten. Ingen av dem är bekräftad.

Tre omständigheter begränsar utrymmet. För det första nämner varningen ett fast tidsfönster i stället för en obestämd avstängning fram till en patch. För det andra ska även system som inte är åtkomliga från internet tas offline. För det tredje infaller fönstret över hela världen vid samma tidpunkt (02:00 till 08:00 UTC) i stället för respektive lokal natt. En klassisk sårbarhet som kan utnyttjas via internet skulle inte förklara de två första punkterna: Då räcker det att koppla bort systemet från internet, och göra det tills patchen finns tillgänglig.

### 1. Myndigheterna känner till en planerad tidpunkt

Brottsbekämpande myndigheter får ibland i förväg kännedom om tidpunkten för en planerad kampanj, exempelvis från övervakad kommunikation inom en gärningsmannagrupp eller från beslagtagen infrastruktur. Massutnyttjanden av filutbytesprodukter sker vanligtvis under ett kort, samordnat tidsfönster, ofta på helger eller helgdagar när mindre personal är i tjänst. Kiteworks är efterföljaren till Accellion, vars File Transfer Appliance attackerades på precis detta sätt 2020 och 2021, då tillskrivet gruppen Clop: Data exfiltrerades via flera sårbarheter och de berörda organisationerna utpressades sedan.

Det talar för det snävt avgränsade fönstret under helgen. Emot talar invändningen som flera kommentatorer på heise framför: Varningen gick ut till alla kunder, så angriparna bör känna till den och kan helt enkelt skjuta upp attacken. En förskjutning skulle dock ge tillverkaren tid att ta fram en patch.

### 2. Tillverkaren känner ännu inte till sårbarheten

Det är också möjligt att Kiteworks, utöver informationen från myndigheterna, saknar tekniska detaljer och alltså varken känner till den berörda komponenten eller kan rekommendera en patch eller konfigurationsändring. Då är avstängning den enda åtgärd som fungerar utan kunskap om sårbarheten, och det fasta slutet är en kompromiss som kunderna lättare kan acceptera. I heise-kommentarerna framförs misstanken att tillverkaren kan låta vissa system vara online som lockbete under fönstret för att observera attacken. Det finns inga belägg för detta.

Det talar för att varken ett advisory eller en mitigation nämns. Emot talar att Kiteworks enligt egna uppgifter samarbetar med Mandiant och att indikatorer normalt åtminstone finns tillgängliga vid en varning från myndigheter.

### 3. En redan placerad bakdörr med tidsutlösare

Rekommendationen att även stänga av interna system passar ett scenario där attacken inte kommer utifrån utan redan är förberedd på apparaterna: exempelvis en bakdörr från en tidigare kompromettering som aktiveras vid en bestämd tidpunkt eller tar kontakt med en kontrollserver. Ett avstängt system kan inte exekvera något vid den tidpunkten.

Det talar för att åtkomlighet från internet inte spelar någon roll i detta scenario. Emot talar att en tillverkare i ett sådant fall snarare skulle rekommendera kontroll av kompromettering och ominstallation än en omstart efter sex timmar.

### 4. Kompromettering hos tillverkaren

En annan väg som når interna system är anslutningar från apparaten till tillverkaren, exempelvis för uppdateringar, licenskontroll eller fjärrunderhåll. Om en sådan kanal är komprometterad skyddar en brandvägg inte mot inkommande trafik. I detta scenario skulle avstängningen ge tillverkaren ett fönster för att sanera sin egen infrastruktur, byta nycklar eller certifikat och först därefter tillåta anslutningar igen.

Det talar för den globalt enhetliga tidpunkten, som passar en samordnad åtgärd hos tillverkaren. Emot talar att tillverkaren då snarare skulle rekommendera att blockera utgående anslutningar än att stänga av systemen helt.

### 5. Åtgärd som följer en myndighetsinsats

Slutligen är det tänkbart att myndigheterna samtidigt ingriper mot angriparnas infrastruktur och vill förhindra att de snabbt slår till som reaktion. Det skulle förklara det korta fönstret och brottsbekämpningens roll. Att BKA avböjer ett uttalande av utredningstaktiska skäl tyder på pågående utredningar, men bevisar inte denna variant.

### Kritik mot kommunikationen

I heise-kommentarerna dominerar skepsis, och invändningarna är sakligt begripliga: Utan uppgifter om sårbarheten går det inte att bedöma om en frånkoppling från internet via brandvägg hade räckt. Ett tidsfönster utan utannonserad patch lämnar öppet vad som gäller efter 10:00. Och en varning som bara går per e-post till kunder når inte alla operatörer, till exempel hos partner, tjänsteleverantörer eller efter personalbyten. Oavsett vilken teori som stämmer bör den som driver Kiteworks granska loggarna efter omstarten och bevaka tillverkarens kanaler tills ett advisory finns.

## Slutrapport

Per den 7 oktober 2026 är incidenten avslutad ur tillverkarens perspektiv. Händelserna efter avstängningsfönstret kan delas in i två spår: sårbarheten som hittades under avstängningen och sampubliceringen av advisories två dagar senare.

### Sårbarheten från avstängningsfönstret

Den 28 september publicerade Kiteworks ett andra pressmeddelande. Enligt detta samarbetade tillverkaren med federala myndigheter under helgen; då upptäcktes en tidigare okänd kritisk sårbarhet som är begränsad till en funktion som är aktiverad hos mindre än 1 % av kunderna. Kiteworks utvecklade och rullade ut en fix under avstängningsfönstret och aktiverade dessutom ett skyddslager i alla miljöer. Den kontinuerliga övervakningen visade inga misstänkta aktiviteter, och det finns inga indikationer på kompromettering av Kiteworks- eller kundsystem. Alla andra Kiteworks-produkter berörs inte.

Den berörda funktionen, ett CVE-nummer, versionerna med fixen och frågan om installationer som drivs av kunden själv fick fixen automatiskt har inte offentliggjorts. Upphävandet den 27 september innehöll bara ett undantag: Kunder med egen drift av Advanced Forms skulle kontakta supporten före omstarten. Kiteworks har inte bekräftat om denna funktion var den berörda.

Om teorierna ovan: Pressmeddelandet beskriver en sårbarhet som först hittades under fönstret. Det passar teori 2 (tillverkaren kände inte till sårbarheten tidigare) i kombination med teori 1 (myndigheterna kände till en planerad tidpunkt). Det finns ingen bekräftelse för teorierna 3 till 5. Vad myndigheterna konkret visste och om en attack försöktes är fortfarande inte känt.

### 125 säkerhetsadvisories från den 30 september

Den 30 september från 18:38 publicerade Kiteworks 125 säkerhetsadvisories på GitHub på en gång. De fördelar sig enligt följande:

| Produkt | Advisories | därav kritiska |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Totalt** | **125** | **12** |

Efter allvarlighetsgrad är 12 kritiska, 49 höga, 52 medelhöga och 12 låga. Alla sårbarheter är åtgärdade i versioner upp till och med 9.5.1; de äldsta posterna berör version 9.2.1. Det handlar alltså om ett efterhandsavslöjande av redan levererade fixar, inte om en ny version. Advisories anger främst deltagare i bug bounty-programmet på YesWeHack som rapportörer. Kiteworks kopplar dem inte till sårbarheten från avstängningsfönstret; en vecka efter fönstret överensstämmer advisoryläget fortfarande med uttalandet från den 25 september att alla kända sårbarheter är åtgärdade i 9.5.1.

De kritiska advisories:

| CVE | Produkt | CVSS 3.1 | åtgärdad från | Effekt |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Kodexekvering med root-rättigheter utan inloggning |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Kontoövertagande |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Kontoövertagande |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Kontoövertagande via lösenordsåterställningen |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Kontoövertagande |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Kontoövertagande |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Kringgående av säkerhetsfunktioner |
| CVE-2026-102095, CVE-2026-102102 till 102105 | EPG | 9.1 | 9.5.0 | Åtkomst till interna nätverksresurser (SSRF) |

CVE-2026-54154 är den allvarligaste sårbarheten: Enligt advisory gör en kombination av fel i inmatningsvalideringen i publikt åtkomliga ändpunkter i Email Protection Gateway det möjligt för en oinloggad angripare att exekvera kod och, via ytterligare lokala svagheter, få root-rättigheter på apparaten. MS-ISAC utfärdade ett eget advisory om detta den 1 oktober. För e-postadministratörer är gatewayen den relevanta delen av publiceringen: Den sitter vanligtvis direkt i e-postflödet och är åtkomlig från internet.

### Exploatering och utbredning

Det finns inga rapporter om exploatering eller offentliga exploits för någon av sårbarheterna. CISA KEV-katalogen innehåller per den 7 oktober endast de fyra Accellion FTA-posterna från 2021. Enligt BleepingComputer räknar Shadowserver knappt 400 Kiteworks-instanser som är åtkomliga från internet; hur många av dem som redan kör 9.5.1 är inte känt. Totemomail förekommer inte i något av advisories.

## Efter fönstret: Vad operatörer kan göra nu

Rekommendationen om avstängning har upphävts och de kända sårbarheterna är åtgärdade i version 9.5.1. För installationer som drivs av kunden själv är följande steg lämpliga:

1.  **Kontrollera version:** Körs version 9.5.1 på alla noder och alla komponenter (Core, Email Protection Gateway, Secure Data Forms, MFT Server)? En Email Protection Gateway före 9.4.1 berörs av CVE-2026-54154 och bör uppdateras först.

2.  **Kontrollera klusterstatus:** I Cluster Health Dashboard bör alla noder vara gröna och underhållsläget vara avstängt.

3.  **Utvärdera loggar:** Granska inloggningar, administratörsåtgärder och ovanliga filnedladdningar kring avstängningsfönstret, särskilt på system som inte stängdes av eller stängdes av sent.

4.  **Begränsa åtkomlighet:** Där det är möjligt, blockera åtkomst till administrationsgränssnittet från internet och tillåt endast nödvändiga tjänster.

5.  **Advanced Forms:** Den som driver modulen själv och ännu inte har kontaktat supporten bör klargöra med Kiteworks om fixen från avstängningsfönstret har nått den egna installationen.

6.  **Jämför advisories:** GitHub-advisories kan filtreras efter produkt (prefix `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Kontrollera för varje använd komponent om den installerade versionen ligger under den angivna fixversionen.

7.  **Bevaka kanaler:** GitHub-advisories, Newsroom och kundmejl från Kiteworks, ifall tillverkaren trots allt publicerar ett advisory med CVE-nummer om sårbarheten från avstängningsfönstret. Nya CVE:er för Kiteworks Email Protection Gateway och Totemomail listas även i denna sidas [CVE-tracker](/cve); där går det att prenumerera på varningsmejl.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Hjälp med uppdateringen</p>
<p>Om du behöver hjälp med att uppdatera en Kiteworks- eller Totemomail-gateway, till exempel med att omdirigera e-postflödet under underhållsfönstret eller utvärdera loggarna, använd <a href="https://adeptio.ch/">kontaktformuläret på adeptio.ch</a>.</p>
</div>

## Källor

1.  [heise online: Förestående zero-day-attack: KiteWorks uppmanar kunder att stänga av servrar](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Första rapporten med utdrag ur kundmejlet och tidsfönstret; uppdatering från 25.09. kl. 17:41 med BKA:s svar.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): engelsk version med CISO:ns ursprungliga ordalydelse.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): tillverkarens äldre uppdateringssida, per 07.10.2026 utan post om varningen eller advisories från 30.09.2026.

4.  [Kiteworks: Security Advisories på GitHub](https://github.com/kiteworks/security-advisories/security): tillverkarens advisorylista, fram till 28.09.2026 var senaste posten från 27.05.2026; den 30.09.2026 tillkom 125 nya advisories för Core, EPG, SDF och MFT. Siffrorna i denna artikel räknades via GitHub API.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): officiella meddelanden, sedan 25.09.2026 med pressmeddelandet om avstängningen.

6.  [heise-forum: Kommentarer till rapporten](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): läsardiskussion med invändningarna mot det fasta tidsfönstret och avstängningen av interna system samt lockbetesteorin.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): advisory om exploateringen av Accellion FTA 2020/2021 med efterföljande utpressning; Accellion är Kiteworks tidigare namn.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): tillverkaruppgift om samarbetet med Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO:ns uttalande, varningens utskickstid, reaktioner från FBI och CISA (tillägg), följder hos en kund och antal system som är åtkomliga från internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (pressmeddelande)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): officiellt meddelande från 25.09.2026 med information om hostade instanser, version 9.5.1 och de opåverkade dotterbolagen; kompletterat med upplysningen från 27.09.2026 att rekommendationen om avstängning har upphävts.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): tidsfönster per region och bedömning av tidigare attacker mot filutbytesprodukter.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowrs bedömning av den ovanliga rekommendationen om avstängning.

13.  [BornCity: Kiteworks: Mer än 1 000 organisationer ska stänga av servrar](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): antal underrättade organisationer och branscher i den tyskspråkiga regionen.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): bedömning av Clops Accellion-attacker 2020/2021 och citat från watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): rapport från 28.09.2026 om upphävandet av rekommendationen och drift av de hostade instanserna.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (pressmeddelande)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): meddelande från 28.09.2026 om den kritiska sårbarhet som hittades under avstängningen, fixen och det ytterligare skyddslagret.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): sammanfattning av det andra pressmeddelandet med citat från CISO:n.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): tillverkaruppgifter om kodexekvering i Email Protection Gateway före 9.4.1, CVSS 10.0, rapporterad via YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): rapport från 01.10.2026 om CVE-2026-54154 och antal instanser som Shadowserver räknar.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): advisory från Center for Internet Security från 01.10.2026 med rekommendationer.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): bedömning av sårbarheterna för kontoövertagande i Core, bland annat CVE-2026-102115; räkningen avviker från advisorylistan.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): per 07.10.2026 endast de fyra Accellion FTA-posterna från 2021, ingen Kiteworks-post från 2026.
