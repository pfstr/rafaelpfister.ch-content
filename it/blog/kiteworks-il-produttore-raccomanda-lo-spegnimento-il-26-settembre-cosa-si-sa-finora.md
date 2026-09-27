---
title: "Kiteworks: il produttore raccomanda lo spegnimento il 26 settembre - Cosa si sa finora"
navTitle: "Spegnimento di Kiteworks"
description: "Kiteworks invita i propri clienti via e-mail a spegnere tutti i sistemi sabato 26.09.2026, dalle 04:00 alle 10:00. Il motivo è un avviso delle autorità di contrasto su un possibile attacco. Totemomail non è interessata."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min di lettura"
themen:
  - totemomail
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-il-produttore-raccomanda-lo-spegnimento-il-26-settembre-cosa-si-sa-finora"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
url: https://rafaelpfister.ch/it/blog/kiteworks-il-produttore-raccomanda-lo-spegnimento-il-26-settembre-cosa-si-sa-finora
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:45:36.714Z
translationReview: automatic
---

# Kiteworks: il produttore raccomanda lo spegnimento il 26 settembre - Cosa si sa finora

Il 25 settembre 2026, Kiteworks ha invitato i propri clienti via e-mail a spegnere tutti i sistemi Kiteworks sabato 26 settembre, dalle 04:00 alle 10:00 (ora dell’Europa centrale). Secondo la comunicazione del CISO Frank Balonis, il produttore dispone di indicazioni da parte delle autorità di contrasto secondo cui questo fine settimana potrebbe verificarsi un attacco ai sistemi Kiteworks. L’assistenza clienti motiva lo spegnimento con la protezione da possibili attacchi zero-day. heise online ha confermato telefonicamente l’autenticità del messaggio con il supporto.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Assistenza d’emergenza per il reindirizzamento del flusso di posta</p>
<p>Se avete bisogno di aiuto per reindirizzare il flusso di posta prima dello spegnimento e ripristinarlo successivamente, utilizzate il <a href="https://adeptio.ch/">modulo di contatto su adeptio.ch</a>. Risponderò anche con breve preavviso.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Aggiornamento del 25 settembre 2026: dichiarazione di Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail non è interessata.</strong> Resta da chiarire se Kiteworks EPG (Email Protection Gateway) sia interessato.</p>
</div>

## Cronologia

Tutti gli orari sono espressi nell’ora legale dell’Europa centrale (CEST). Laddove non è indicato un orario, non sono disponibili informazioni temporali affidabili.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Avviso ai clienti</p>
<p>Il CISO Frank Balonis informa i clienti via e-mail delle indicazioni delle autorità di contrasto su un possibile attacco durante il fine settimana e raccomanda uno spegnimento di sei ore. Secondo l’avviso, tutte le vulnerabilità note sono state risolte nella versione 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Prime notizie sui media</p>
<p>heise online riferisce che il supporto Kiteworks conferma l’autenticità del messaggio e motiva lo spegnimento con la protezione da possibili attacchi zero-day. Poco dopo seguono TechCrunch, BleepingComputer, Computer Weekly e altri.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre, 17:41</p>
<p class="timeline__titel">La BKA non commenta</p>
<p>heise aggiunge: la BKA rifiuta di rilasciare una dichiarazione per ragioni investigative. La BSI non risponde, l’FBI rinuncia a commentare nei confronti di TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Dichiarazione e comunicato stampa</p>
<p>Kiteworks definisce lo spegnimento una misura precauzionale senza compromissioni note. Il comunicato stampa indica le «federal intelligence authorities» come fonte ed elenca le controllate non interessate, tra cui totemo. Il produttore spegne direttamente le istanze ospitate da Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sab, 26 settembre, dalle 04:00 alle 10:00</p>
<p class="timeline__titel">Finestra di spegnimento</p>
<p>La finestra è simultanea in tutto il mondo: dalle 02:00 alle 08:00 UTC, a Sydney dalle 12:00 alle 18:00, a New York da venerdì alle 22:00 a sabato alle 04:00.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situazione a sab, 26 settembre</p>
<p class="timeline__titel">Ancora aperto</p>
<p>Nessun avviso pubblico, nessun numero CVE, nessuna indicazione sulla falla e nessuna segnalazione di un attacco avvenuto.</p>
</li>
</ol>

## Cosa si sa

La raccomandazione vale a livello mondiale; l’e-mail indica la finestra per tutti i fusi orari da AEST a PDT. Kiteworks consiglia di spegnere i sistemi già prima dell’inizio della finestra, anche se non sono raggiungibili da Internet.

Per ora, quasi tutto il resto resta aperto: non esistono un Security Advisory pubblico, un numero CVE, una patch né indicazioni sui prodotti o sulle versioni interessati. Il comunicato stampa cita le «federal intelligence authorities» come fonte, presumibilmente autorità federali statunitensi; non è noto quali. Alla data del 26 settembre non risulta alcuna voce negli Security Updates né negli advisory GitHub di Kiteworks. Sono pubblici la dichiarazione citata sopra e il comunicato stampa del 25 settembre.

Nei confronti di TechCrunch, il CISO di Kiteworks Frank Balonis ha rilasciato la dichiarazione con lo stesso testo. La BKA ha rifiutato di commentare nei confronti di heise per ragioni investigative, mentre la BSI non ha risposto. L’FBI non ha voluto esprimersi nei confronti di TechCrunch e non è giunta alcuna risposta dalla CISA. Secondo TechCrunch, un cliente del settore sanitario ha immediatamente scollegato il proprio server dalla rete, con limitazioni operative percepibili.

Il comunicato stampa diverge dall’e-mail ai clienti su un punto: parla di una finestra di spegnimento di nove ore, mentre l’avviso ai clienti parla di sei ore. Secondo il comunicato stampa, la raccomandazione riguarda solo le installazioni gestite autonomamente (On-Premises, AWS, Azure). Secondo il produttore, non sono interessate le controllate Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai e 123FormBuilder.

## L’avviso ai clienti

L’e-mail ai clienti del 25 settembre contiene, oltre all’avvertimento, un calendario per fuso orario e istruzioni per i cluster. Convertendo gli orari in UTC, per tutte le regioni risulta la medesima finestra dalle 02:00 alle 08:00 UTC.

| Fuso orario | Città | Inizio | Fine |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Sab, 12:00 | Sab, 18:00 |
| SGT (UTC+8) | Singapore | Sab, 10:00 | Sab, 16:00 |
| IDT (UTC+3) | Tel Aviv | Sab, 05:00 | Sab, 11:00 |
| CEST (UTC+2) | Amsterdam, Zurigo | Sab, 04:00 | Sab, 10:00 |
| BST (UTC+1) | Londra | Sab, 03:00 | Sab, 09:00 |
| EDT (UTC−4) | New York | Ven, 22:00 | Sab, 04:00 |
| CDT (UTC−5) | Chicago | Ven, 21:00 | Sab, 03:00 |
| MDT (UTC−6) | Denver | Ven, 20:00 | Sab, 02:00 |
| PDT (UTC−7) | San Francisco | Ven, 19:00 | Sab, 01:00 |

Per i cluster con più server, Kiteworks prescrive un ordine preciso:

1.  **Attivare la modalità di manutenzione** in System Setup > Maintenance Mode, affinché gli utenti non possano più accedere.

2.  **Creare un backup:** uno snapshot di ciascun nodo o un backup del database Kiteworks (System Setup > Cluster Configuration > System Configuration). Viene conservato un solo backup del database; ogni nuovo backup sostituisce il precedente.

3.  **Rilevare i ruoli:** in System Setup > Locations, la colonna Assigned Roles mostra quali nodi hanno il ruolo Application; il nodo Application primario è contrassegnato da un asterisco. Annotare i nodi e i loro indirizzi IP, poiché saranno necessari per il riavvio.

4.  **Spegnere in questo ordine:** prima tutti i nodi senza ruolo Application, quindi gli altri nodi Application e infine il nodo Application primario. È possibile farlo tramite la scheda Shut Down del rispettivo nodo oppure tramite la console dell’hypervisor (ad esempio VMware o AWS), se l’interfaccia Kiteworks non è più raggiungibile.

5.  **Riavviare in ordine inverso** tramite l’hypervisor, poiché la console di amministrazione è raggiungibile soltanto quando è in esecuzione un numero sufficiente di nodi (appendice E dell’Administrator Guide): prima il nodo Application primario, quindi gli altri nodi Application, uno alla volta e solo quando il precedente è completamente operativo, affinché i server del database possano formare un quorum. Poi i server di storage, quindi gli altri ruoli (Repositories Gateway, Search, SFTP, Antivirus) e infine i server web.

6.  **Disattivare la modalità di manutenzione** non appena tutti i nodi nel Cluster Health Dashboard, nella pagina di stato della console di amministrazione, sono verdi.

Su richiesta, il supporto Kiteworks ha inoltre confermato che nessuna delle controllate di Kiteworks è interessata.

## Possibili cause: teorie

Finché Kiteworks non pubblicherà dettagli, la causa rimarrà aperta. Le spiegazioni seguenti sono ipotesi dedotte dagli elementi noti; alcune sono discusse anche nei commenti all’articolo di heise. Nessuna è confermata.

Tre elementi delimitano il campo. In primo luogo, l’avviso indica una finestra temporale fissa anziché uno spegnimento a tempo indeterminato fino alla disponibilità della patch. In secondo luogo, devono essere scollegati anche i sistemi non raggiungibili da Internet. In terzo luogo, la finestra ha lo stesso orario in tutto il mondo (dalle 02:00 alle 08:00 UTC), anziché coincidere ogni volta con la notte locale. Una classica vulnerabilità sfruttabile via Internet non spiegherebbe i primi due punti: per contrastarla basta scollegare il sistema da Internet, fino all’arrivo della patch.

### 1. Le autorità conoscono un momento pianificato

Le autorità di contrasto talvolta apprendono in anticipo il momento di una campagna pianificata, ad esempio da comunicazioni monitorate di un gruppo criminale o da infrastrutture sequestrate. Gli sfruttamenti su larga scala di prodotti per lo scambio di file avvengono tipicamente in una breve finestra coordinata, spesso nei fine settimana o nei giorni festivi, quando è in servizio meno personale. Kiteworks è il successore di Accellion, la cui File Transfer Appliance è stata attaccata proprio in questo modo nel 2020 e nel 2021: tramite diverse vulnerabilità sono stati sottratti dati, quindi le organizzazioni colpite sono state ricattate.

A favore di questa ipotesi parla la finestra ristretta nel fine settimana. Contro vi è l’obiezione sollevata da vari commentatori su heise: l’avviso è stato inviato a tutti i clienti, quindi gli aggressori dovrebbero esserne al corrente e potrebbero semplicemente rimandare l’attacco. Un rinvio darebbe però al produttore il tempo di preparare una patch.

### 2. Il produttore non conosce ancora la vulnerabilità

È anche possibile che Kiteworks, oltre all’indicazione delle autorità, non disponga di dettagli tecnici, dunque non conosca né il componente interessato né possa raccomandare una patch o una modifica di configurazione. In tal caso, lo spegnimento è l’unica misura efficace senza conoscere la vulnerabilità, e la fine fissata è un compromesso che i clienti sono più propensi ad accettare. Nei commenti di heise viene avanzata l’ipotesi che il produttore possa lasciare online singoli sistemi come esca durante la finestra, per osservare l’attacco. Non esistono prove al riguardo.

A favore parla il fatto che non sono indicati né un advisory né una mitigazione. Contro parla il fatto che Kiteworks dichiara di collaborare con Mandiant e che, in caso di un avviso delle autorità, di norma sono disponibili almeno degli indicatori.

### 3. Una backdoor già installata con attivazione temporizzata

La raccomandazione di spegnere anche i sistemi interni è coerente con uno scenario in cui l’attacco non proviene dall’esterno, ma è già stato predisposto sulle appliance: ad esempio una backdoor derivante da una precedente compromissione, che si attiva a un orario stabilito o contatta un server di controllo. Un sistema spento non può eseguire nulla in quel momento.

A favore parla il fatto che, in questo scenario, la raggiungibilità da Internet non ha alcun ruolo. Contro parla il fatto che in questo caso un produttore raccomanderebbe più probabilmente una verifica della compromissione e una reinstallazione, anziché un riavvio dopo sei ore.

### 4. Compromissione lato produttore

Un’altra via per raggiungere sistemi interni sono le connessioni instaurate dall’appliance verso il produttore, ad esempio per aggiornamenti, verifica delle licenze o manutenzione remota. Se uno di questi canali è compromesso, un firewall non protegge dal traffico in entrata. In questo scenario, lo spegnimento darebbe al produttore una finestra per bonificare la propria infrastruttura, sostituire chiavi o certificati e consentire nuovamente le connessioni solo in seguito.

A favore parla l’orario uniforme a livello mondiale, compatibile con un’azione coordinata lato produttore. Contro parla il fatto che il produttore raccomanderebbe probabilmente di bloccare le connessioni in uscita anziché spegnere completamente i sistemi.

### 5. Misura complementare a un’operazione delle autorità

Infine, è ipotizzabile che nello stesso periodo le autorità intervengano contro l’infrastruttura degli aggressori e vogliano impedire loro di colpire rapidamente in reazione. Ciò spiegherebbe la breve finestra e il ruolo delle forze dell’ordine. Il fatto che la BKA rifiuti di commentare per ragioni investigative suggerisce indagini in corso, ma non dimostra questa variante.

### Critiche alla comunicazione

Nei commenti di heise prevale lo scetticismo e le obiezioni sono oggettivamente comprensibili: senza indicazioni sulla vulnerabilità, non è possibile valutare se sarebbe bastato scollegare da Internet tramite firewall. Una finestra temporale senza una patch annunciata lascia aperto cosa valga dopo le 10:00. E un avviso inviato solo via e-mail ai clienti non raggiunge tutti gli operatori, ad esempio presso partner, fornitori di servizi o dopo cambi di personale. Indipendentemente dalla teoria corretta: chi gestisce Kiteworks dovrebbe controllare i log dopo il riavvio e monitorare i canali del produttore finché non sarà disponibile un advisory.

## Fonti

1.  [heise online: Imminente attacco zero-day: KiteWorks esorta i clienti a spegnere i server](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): prima notizia con estratti dall’e-mail ai clienti e dalla finestra temporale; aggiornamento del 25.09., ore 17:41, con la risposta della BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versione inglese con il testo originale del CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): canale ufficiale del produttore, senza voce relativa all’avviso alla data del 26.09.2026.

4.  [Kiteworks: Security Advisories su GitHub](https://github.com/kiteworks/security-advisories/security): elenco degli advisory del produttore, ultima voce del 27.05.2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): comunicazioni ufficiali, dal 25.09.2026 con il comunicato stampa sullo spegnimento.

6.  [Forum heise: commenti all’articolo](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussione dei lettori con le obiezioni sulla finestra temporale fissa e sullo spegnimento dei sistemi interni, nonché la teoria dell’esca.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): advisory sullo sfruttamento di Accellion FTA nel 2020/2021 con successiva estorsione; Accellion è il nome precedente di Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): indicazione del produttore sulla collaborazione con Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): dichiarazione del CISO, orario di invio dell’avviso, reazioni di FBI e CISA, conseguenze per un cliente.

10.  [Kiteworks: Precautionary Shutdown Advisory (comunicato stampa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicazione ufficiale del 25.09.2026 con indicazioni sulle istanze ospitate, sulla versione 9.5.1 e sulle controllate non interessate.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): finestra temporale per regioni e inquadramento dei precedenti attacchi a prodotti per lo scambio di file.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valutazione di watchTowr sull’insolita raccomandazione di spegnimento.
