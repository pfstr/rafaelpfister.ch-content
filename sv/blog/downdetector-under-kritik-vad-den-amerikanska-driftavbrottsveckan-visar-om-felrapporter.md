---
title: "Downdetector under kritik: Vad den amerikanska driftavbrottsveckan visar om felrapporter"
navTitle: "Kritik mot Downdetector"
description: "Mellan den 13 och 17 november 2026 registrerade Downdetector i USA fler störningar än under någon annan vecka det senaste året. Avbrotten hos FAA, AWS, Chase, United och Apple var verkliga, men hade separata orsaker och var oftast åtgärdade inom några timmar. Downdetectors siffror förstärkte bilden av en sammanhängande kris. Vad plattformen mäter, var kritiken riktas och hur administratörer bättre kan bedöma störningar."
date: "2026-11-18"
kategorie: "Drift"
timeToRead: "7 min lästid"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-under-kritik-vad-den-amerikanska-driftavbrottsveckan-visar-om-felrapporter"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/sv/blog/downdetector-under-kritik-vad-den-amerikanska-driftavbrottsveckan-visar-om-felrapporter
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:45:53.515Z
translationReview: automatic
---

# Downdetector under kritik: Vad den amerikanska driftavbrottsveckan visar om felrapporter

Mellan den 13 och 17 november 2026 drabbades flera stora system i USA av avbrott: flygtrafikledningen FAA med ett landsomfattande startförbud, delar av AWS i regionen us-east-1, banken JPMorgan Chase, markservicen hos United Airlines samt iCloud-inloggningen och andra tjänster från Apple på tisdagen. Jag följde händelseförloppet i en [liverapport](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). När veckan var över stod det klart: Avbrotten hade separata orsaker, det finns inga tecken på en cyberattack och Europa berördes endast marginellt. Ändå uppstod i allmänhetens ögon bilden av ett internet som faller samman överallt samtidigt. En stor del av den bilden kommer från Downdetectors diagram, som citerades i nästan varje medierapport. Sedan i måndags har detta mött bred kritik.

## Vad Downdetector mäter

Downdetector tillhör Ookla och därmed Accenture sedan den 17 juni 2026. Plattformen samlar in problemrapporter från användare som rapporterar ett problem på en tjänsts sida och kompletterar enligt sin egen beskrivning detta med signaler från sociala medier och andra onlinekällor. För varje tjänst beräknar Downdetector en baslinje: det genomsnittliga antalet rapporter vid samma tid på dygnet under det senaste året. Om det aktuella antalet ligger tydligt över detta betraktas det som en störning. Downdetector publicerar inte hur källorna viktas eller vid vilken faktor en topp räknas som en störning.

Detta medför tre egenskaper som ofta förbises i rapporteringen:

1.  **Rapporter är inte detsamma som berörda användare.** En siffra som «20'000 rapporter» säger inget om hur många kunder som faktiskt saknade åtkomst. Den räknar personer som har märkt ett problem och öppnat Downdetector.

2.  **Användare rapporterar den tjänst de ser.** Om ett betalningsnätverk, en molnleverantör eller en identitetstjänst slutar fungera, visas rapporterna hos de banker, appar och butiker som är beroende av dem. Orsaken syns inte i diagrammet.

3.  **Baslinjen avspeglar inte exceptionella veckor.** Om medier löpande rapporterar om avbrott kontrollerar fler personer vid varje fördröjning om en tjänst är störd. Varje ny rubrik ökar antalet rapporter även för tjänster som fungerar normalt.

## Veckans siffror

Enligt Axios tog Downdetector emot nästan 6 miljoner problemrapporter från USA under de sju dagarna fram till den 16 november, det högsta veckovärdet på ett år. Samma analys visar dock också att det under en 30-dagarsperiod förekom tre stora avbrott, jämfört med normalt fem. Rekordantalet rapporter står därmed i kontrast till ett snarare undergenomsnittligt antal faktiska större störningar.

Flera fall under veckan visar hur stort avståndet kan vara mellan rapporter och det faktiska läget:

- **Banker den 16 november:** Medier skrev med hänvisning till Downdetector att användare rapporterade problem hos «nästan alla stora amerikanska banker», däribland Chase, Wells Fargo och Capital One. Chase uppgav självt att transaktioner tog längre tid; kunderna hade fortsatt åtkomst till konton och tillgodohavanden. Enligt New York Times låg orsaken i ett programvarufel i JPMorgan Chases egen hårdvara, vilket kan ha påverkat andra banker via gemensamma transaktioner. De andra bankerna bekräftade inga egna störningar.

- **Gmail den 16 och 17 november:** Downdetector visade över 2'800 rapporter, främst om e-post som inte levererades. Google uppgav att det inte fanns några tecken på en omfattande störning och Workspace Status Dashboard förblev utan notis.

- **Charles Schwab den 17 november:** Enligt Wall Street Journal steg rapporterna omkring klockan 9:50 lokal tid och föll igen en halvtimme senare. Det finns ingen störningsrapport från Schwab om detta.

- **Exchange Online den 12 november:** Reuters angav över 97'000 rapporter för samma störning, medan en lokal tv-station angav nästan 119'000, båda vid olika tidpunkter. Enligt Downdetector började rapporterna först stiga omkring 68 minuter efter den starttid som Microsoft anger i Admin Center.

Avbrotten hos FAA, AWS, Chase, United och Apple var verkliga och bekräftade av operatörerna. Det överdrivna var sammankopplingen: Fem oberoende störningar och en allmän ökning av rapporterna blev berättelsen om en samordnad attack. Vita husets talesperson sade den 17 november att det inte fanns någon kris som amerikaner behövde oroa sig för. Liz Fong-Jones (Honeycomb) sade till Axios att incidenterna hade mycket lite gemensamt.

## Var kritiken riktas

Kritiken kommer från flera håll. Operatörer som Google säger offentligt emot när Downdetector visar en störning som de inte ser i sina egna data. Fackmedier påpekar att Downdetector räknar rapporter i stället för berörda användare, visar störningar med fördröjning och inte alls fångar avbrott i komponenter som slutkunder inte ser. I diskussioner bland utvecklare, till exempel på Hacker News, förekommer dessutom antagandet att redan ett besök på en sida som «Är Chase störd?» räknas som en implicit rapport och att toppar på så sätt överförs från en tjänst till liknande tjänster. Downdetector har inte bekräftat detta antagande; det är endast belagt att plattformen utvärderar ytterligare onlinekällor utöver aktiva rapporter och inte redovisar deras viktning.

Plattformens försvarare invänder att Downdetector ofta visar störningar tidigare än leverantörernas statussidor. Vid Apple-avbrottet den 17 november visade Apples statussida fortfarande normal tillgänglighet när rapporterna redan ökade kraftigt. Tjänsten är användbar som tidig varning. Den är inte ett mått på en störnings omfattning, och det var just så den användes denna vecka: Rubriker som «Tusentals rapporterar problem» stod utan sammanhang bredvid bekräftade avbrott.

## Bedöm störningar bättre

För administratörer som vid en rapporterad störning måste avgöra om de ska informera användare eller vidta åtgärder finns en enkel ordning:

1.  **Kontrollera leverantörens statussida och administrationsportal.** För Microsoft 365 är det Service Health i Admin Center, inte den offentliga statussidan. AWS, Google Cloud, Cloudflare och Apple har egna statussidor med tidsstämplar.

2.  **Använd egna mätningar.** Ett enkelt syntetiskt test (inloggning, e-postutskick, hämtning av en sida) från det egna nätverket visar om den egna organisationen påverkas. Min [M365-anslutningsmonitor](https://rafaelpfister.ch/liveticker) kontrollerar var femte minut tillgängligheten till Microsoft 365-slutpunkter från Europa.

3.  **Läs Downdetector endast som en indikation.** En topp är en anledning att undersöka saken. Det absoluta antalet och toppar hos flera liknande tjänster samtidigt säger lite om orsak och omfattning.

4.  **Beakta regionalt ursprung.** De flesta siffror i medierapporter avser den amerikanska versionen av Downdetector. För användare i Schweiz är de bara relevanta om leverantören också bekräftar störningen för europeiska regioner.

## Liverapporten

Jag avslutade liverapporten om serien av avbrott med denna artikel. Den finns kvar online som en krönika över veckan, med en slutlig bedömning av de enskilda orsakerna och uppgiften att Europa, förutom vid transatlantiska flygningar och den globala iCloud-störningen, inte påverkades.

## Källor

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): nästan 6 miljoner Downdetector-rapporter på sju dagar, tre stora avbrott i stället för de vanliga fem under 30 dagar, bedömning av Liz Fong-Jones.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): funktion, baslinje per tid på dygnet, uppköpet av Accenture, avvikande siffror för Exchange Online-störningen den 12 november.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): Downdetector-rapporter hos flera amerikanska banker, uttalande från Chase.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): citerar New York Times och Wall Street Journal om Apple, Charles Schwab och orsaken hos JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): Downdetector-rapporter om Gmail och Googles dementi.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): översikt över operatörernas förklaringar.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): förloppet vid Apple-avbrottet, statussidan låg efter rapporterna.

8.  [Hacker News: Diskussion om Downdetectors metodik](https://news.ycombinator.com/item?id=50521819): antaganden om implicita rapporter och överföring mellan liknande tjänster, inte bekräftat av Downdetector.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): bedömning från en säkerhetsexpert, inga tecken på ett samband.
