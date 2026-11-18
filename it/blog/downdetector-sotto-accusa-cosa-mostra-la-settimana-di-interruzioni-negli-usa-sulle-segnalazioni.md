---
title: "Downdetector sotto accusa: cosa mostra la settimana di interruzioni negli USA sulle segnalazioni di disservizi"
navTitle: "Critiche a Downdetector"
description: "Tra il 13 e il 17 novembre 2026, Downdetector ha registrato negli USA più disservizi che in qualsiasi altra settimana dell’ultimo anno. Le interruzioni di FAA, AWS, Chase, United e Apple erano reali, ma avevano cause distinte e nella maggior parte dei casi sono state risolte nel giro di poche ore. I numeri di Downdetector hanno rafforzato l’immagine di una crisi interconnessa. Cosa misura la piattaforma, su cosa si concentrano le critiche e come gli amministratori possono interpretare meglio i disservizi."
date: "2026-11-18"
kategorie: "Esercizio"
timeToRead: "7 min di lettura"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-sotto-accusa-cosa-mostra-la-settimana-di-interruzioni-negli-usa-sulle-segnalazioni"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/it/blog/downdetector-sotto-accusa-cosa-mostra-la-settimana-di-interruzioni-negli-usa-sulle-segnalazioni
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:45:01.508Z
translationReview: automatic
---

# Downdetector sotto accusa: cosa mostra la settimana di interruzioni negli USA sulle segnalazioni di disservizi

Dal 13 al 17 novembre 2026, negli USA sono andati in tilt diversi grandi sistemi: il controllo del traffico aereo FAA con un divieto di decollo a livello nazionale, parti di AWS nella regione us-east-1, la banca JPMorgan Chase, le operazioni di assistenza a terra di United Airlines e, martedì, l’accesso a iCloud e altri servizi di Apple. Ho seguito la serie in un [live ticker](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). Alla fine della settimana è chiaro: le interruzioni avevano cause distinte, non ci sono indizi di un attacco informatico e l’Europa è stata coinvolta solo marginalmente. Nell’opinione pubblica si è comunque diffusa l’immagine di un Internet che smette di funzionare contemporaneamente ovunque. Gran parte di questa immagine proviene dai grafici di Downdetector, citati in quasi tutti gli articoli dei media. Da lunedì sono al centro di ampie critiche.

## Cosa misura Downdetector

Downdetector appartiene a Ookla e, dal 17 giugno 2026, quindi ad Accenture. La piattaforma raccoglie le segnalazioni di problemi degli utenti, che indicano un problema sulla pagina di un servizio, e le integra, secondo la propria descrizione, con segnali provenienti dai social media e da altre fonti online. Per ciascun servizio, Downdetector calcola una linea di base: il numero medio di segnalazioni alla stessa ora del giorno nell’ultimo anno. Se il numero attuale è sensibilmente superiore, viene considerato un disservizio. Downdetector non pubblica né la ponderazione delle fonti né il fattore oltre il quale un picco viene classificato come disservizio.

Ne derivano tre caratteristiche che spesso passano inosservate nella copertura mediatica:

1.  **Le segnalazioni non corrispondono agli utenti interessati.** Un numero come «20'000 segnalazioni» non dice nulla sul numero di clienti che non hanno effettivamente avuto accesso. Conta le persone che hanno notato un problema e hanno aperto Downdetector.

2.  **Gli utenti segnalano il servizio che vedono.** Se una rete di pagamenti, un provider cloud o un servizio di identità smette di funzionare, le segnalazioni compaiono presso le banche, le app e i negozi che ne dipendono. La causa non è riconoscibile nel grafico.

3.  **La linea di base non rappresenta settimane eccezionali.** Se i media riferiscono continuamente di interruzioni, più persone verificano a ogni ritardo se un servizio è in panne. Ogni nuovo titolo aumenta il numero delle segnalazioni anche per servizi che funzionano normalmente.

## I numeri della settimana

Secondo Axios, nei sette giorni fino al 16 novembre Downdetector ha ricevuto dagli USA quasi 6 milioni di segnalazioni di problemi, il valore settimanale più alto dell’ultimo anno. La stessa analisi mostra però anche che, considerando 30 giorni, si sono verificati tre grandi disservizi, mentre normalmente sono cinque. Il numero record di segnalazioni si contrappone quindi a un numero di grandi interruzioni effettive piuttosto inferiore alla media.

Diversi casi della settimana mostrano quanto possano divergere le segnalazioni e la situazione reale:

- **Banche il 16 novembre:** i media hanno scritto, citando Downdetector, che gli utenti segnalavano problemi presso «praticamente ogni grande banca statunitense», fra cui Chase, Wells Fargo e Capital One. Chase stessa ha parlato di transazioni più lente; i clienti continuavano ad avere accesso ai conti e ai saldi. Secondo il New York Times, la causa era un errore software nell’hardware proprietario di JPMorgan Chase, che potrebbe aver coinvolto altre banche attraverso transazioni condivise. Le altre banche non hanno confermato propri disservizi.

- **Gmail il 16 e 17 novembre:** Downdetector ha mostrato oltre 2'800 segnalazioni, soprattutto relative a e-mail non recapitate. Google ha dichiarato che non vi erano indizi di un disservizio diffuso; il Workspace Status Dashboard è rimasto senza avvisi.

- **Charles Schwab il 17 novembre:** secondo il Wall Street Journal, le segnalazioni sono aumentate verso le 9:50 ora locale e sono nuovamente diminuite mezz’ora dopo. Non esiste una segnalazione di disservizio di Schwab in merito.

- **Exchange Online il 12 novembre:** per lo stesso disservizio Reuters ha indicato oltre 97'000 segnalazioni, mentre un’emittente locale quasi 119'000, entrambe in momenti diversi. Secondo Downdetector, le segnalazioni sono aumentate solo circa 68 minuti dopo l’inizio indicato da Microsoft nell’Admin Center.

Le interruzioni di FAA, AWS, Chase, United e Apple erano reali e confermate dagli operatori. Esagerato era il collegamento: da cinque disservizi indipendenti e da un aumento generale delle segnalazioni è nata la narrazione di un attacco coordinato. La portavoce della Casa Bianca ha dichiarato il 17 novembre che non c’era alcuna crisi di cui gli americani dovessero preoccuparsi. Liz Fong-Jones (Honeycomb) ha detto ad Axios che gli incidenti avevano ben poco in comune.

## Su cosa si concentrano le critiche

Le critiche provengono da più direzioni. Operatori come Google contraddicono pubblicamente Downdetector quando mostra un disservizio che non rilevano nei propri dati. Le testate specializzate sottolineano che Downdetector conta le segnalazioni anziché gli utenti interessati, indica i disservizi in ritardo e non rileva affatto le interruzioni nei componenti che i clienti finali non vedono. Nelle discussioni fra sviluppatori, per esempio su Hacker News, circola inoltre l’ipotesi che già l’apertura di una pagina come «Chase è in panne?» venga conteggiata come segnalazione implicita e che i picchi possano così trasferirsi da un servizio a servizi simili. Downdetector non ha confermato questa ipotesi; è documentato soltanto che, oltre alle segnalazioni attive, la piattaforma valuta altre fonti online e non ne divulga la ponderazione.

I difensori della piattaforma ribattono che Downdetector spesso mostra i disservizi prima delle pagine di stato dei provider. Durante l’interruzione di Apple del 17 novembre, la pagina di stato di Apple indicava ancora disponibilità normale quando le segnalazioni erano già in forte aumento. Come sistema di allerta precoce, il servizio è utile. Non lo è come misura dell’entità di un disservizio, ed è proprio così che è stato usato quella settimana: titoli come «Migliaia segnalano problemi» apparivano senza contesto accanto a interruzioni confermate.

## Interpretare meglio i disservizi

Per gli amministratori che, a fronte di un disservizio segnalato, devono decidere se informare gli utenti o avviare misure, ne deriva una semplice sequenza:

1.  **Controllare la pagina di stato e il portale di amministrazione del provider.** Per Microsoft 365 si tratta di Service Health nell’Admin Center, non della pagina di stato pubblica. AWS, Google Cloud, Cloudflare e Apple gestiscono pagine di stato proprie con indicazioni orarie.

2.  **Fare riferimento alle proprie misurazioni.** Un semplice test sintetico (accesso, invio di e-mail, recupero di una pagina) dalla propria rete mostra se la propria organizzazione è coinvolta. Il mio [monitor di connettività M365](https://rafaelpfister.ch/liveticker) verifica ogni cinque minuti la raggiungibilità degli endpoint Microsoft 365 dall’Europa.

3.  **Leggere Downdetector solo come indicazione.** Un picco è un motivo per verificare. Il numero assoluto e i picchi simultanei presso più servizi simili dicono poco sulla causa e sull’entità.

4.  **Considerare l’origine regionale.** La maggior parte dei numeri riportati dai media si riferisce alla versione statunitense di Downdetector. Per gli utenti in Svizzera sono rilevanti solo se il provider conferma il disservizio anche per le regioni europee.

## Il live ticker

Ho concluso il live ticker sulla serie di interruzioni con questo articolo. Rimane online come cronaca della settimana, con una conclusione sulle singole cause e l’indicazione che l’Europa non è stata coinvolta, fatta eccezione per i voli transatlantici e il disservizio globale di iCloud.

## Fonti

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): quasi 6 milioni di segnalazioni Downdetector in sette giorni, tre anziché i consueti cinque grandi disservizi in 30 giorni, valutazione di Liz Fong-Jones.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): funzionamento, linea di base per ora del giorno, acquisizione da parte di Accenture, cifre divergenti sul disservizio di Exchange Online del 12 novembre.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): segnalazioni Downdetector presso diverse banche statunitensi, dichiarazione di Chase.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): cita New York Times e Wall Street Journal su Apple, Charles Schwab e la causa presso JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): segnalazioni Downdetector su Gmail e smentita di Google.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): panoramica delle spiegazioni degli operatori.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): andamento dell’interruzione di Apple, pagina di stato in ritardo rispetto alle segnalazioni.

8.  [Hacker News: Discussione sulla metodologia di Downdetector](https://news.ycombinator.com/item?id=50521819): ipotesi sulle segnalazioni implicite e sul trasferimento fra servizi simili, non confermate da Downdetector.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): valutazione di un esperto di sicurezza, nessun indizio di un collegamento.
