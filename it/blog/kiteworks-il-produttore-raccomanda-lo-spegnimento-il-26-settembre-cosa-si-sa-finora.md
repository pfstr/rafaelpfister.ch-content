---
title: "Kiteworks: il produttore raccomanda lo spegnimento il 26 settembre - Cosa si sa finora"
navTitle: "Spegnimento di Kiteworks"
description: "Kiteworks chiede ai propri clienti via e-mail di spegnere tutti i sistemi sabato 26/09/2026, dalle 04:00 alle 10:00. Il motivo è un avvertimento delle autorità di contrasto su un possibile attacco. Dal 27/09 la raccomandazione è stata revocata; non esistono né una CVE né una nuova patch. Totemomail non è interessato."
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
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T09:58:57.857Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/kiteworks-il-produttore-raccomanda-lo-spegnimento-il-26-settembre-cosa-si-sa-finora
---

# Kiteworks: il produttore raccomanda lo spegnimento il 26 settembre - Cosa si sa finora

Il 25 settembre 2026, Kiteworks ha invitato i propri clienti via e-mail a spegnere tutti i sistemi Kiteworks sabato 26 settembre, dalle 04:00 alle 10:00 (ora dell'Europa centrale). Secondo la comunicazione del CISO Frank Balonis, il produttore ha ricevuto indicazioni dalle autorità di contrasto secondo cui potrebbe verificarsi un attacco ai sistemi Kiteworks durante quel fine settimana. Il supporto clienti giustifica lo spegnimento con la protezione da possibili attacchi zero-day. heise online ha confermato telefonicamente con il supporto l'autenticità del messaggio.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Supporto d'emergenza per la commutazione del flusso di posta</p>
<p>Se avete bisogno di aiuto per reindirizzare il flusso di posta prima dello spegnimento e ripristinarlo in seguito, utilizzate il <a href="https://adeptio.ch/">modulo di contatto su adeptio.ch</a>. Risponderò anche con breve preavviso.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Aggiornamento del 28 settembre 2026: Kiteworks revoca la raccomandazione di spegnimento</p>
<p>Kiteworks ha integrato il comunicato stampa con una nota: dal 27 settembre la raccomandazione di spegnimento non è più valida per tutti i clienti.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Chi non ha ancora riavviato i propri sistemi può farlo ora. Chi gestisce autonomamente Advanced Forms dovrebbe contattare il supporto Kiteworks prima del riavvio. Le istanze ospitate da Kiteworks sono di nuovo operative. Continuano a non esserci un numero CVE, una nuova versione oltre la 9.5.1, indicatori di compromissione né informazioni sul fatto che sia stato tentato un attacco o su cosa abbia motivato l'avvertimento. La pagina Security Updates e gli advisory su GitHub restano invariati.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Aggiornamento del 25 settembre 2026: dichiarazione di Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail non è interessato.</strong> Resta da chiarire se sia interessato Kiteworks EPG (Email Protection Gateway).</p>
</div>

## Cronologia

Tutti gli orari sono espressi nell'ora legale dell'Europa centrale (CEST). Dove non è indicato un orario, non sono disponibili indicazioni temporali affidabili.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Advisory ai clienti</p>
<p>Il CISO Frank Balonis informa i clienti via e-mail delle indicazioni delle autorità di contrasto su un possibile attacco nel fine settimana e raccomanda uno spegnimento di sei ore. Secondo l'advisory, tutte le vulnerabilità note sono risolte nella versione 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Primi resoconti dei media</p>
<p>heise online riporta che il supporto Kiteworks ha confermato l'autenticità del messaggio e giustifica lo spegnimento con la protezione da possibili attacchi zero-day. Poco dopo seguono TechCrunch, BleepingComputer, Computer Weekly e altri.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre, 17:41</p>
<p class="timeline__titel">Il BKA non rilascia dichiarazioni</p>
<p>heise aggiunge: il BKA rifiuta di rilasciare dichiarazioni per ragioni investigative. Il BSI non risponde, l'FBI non commenta a TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Dichiarazione e comunicato stampa</p>
<p>Kiteworks definisce lo spegnimento una misura precauzionale senza compromissioni note. Il comunicato stampa cita le «federal intelligence authorities» come fonte e elenca le filiali non interessate, tra cui totemo. Le istanze ospitate da Kiteworks vengono spente direttamente dal produttore.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sab, 26 settembre, dalle 04:00 alle 10:00</p>
<p class="timeline__titel">Finestra di spegnimento</p>
<p>La finestra è simultanea in tutto il mondo: dalle 02:00 alle 08:00 UTC, a Sydney dalle 12:00 alle 18:00, a New York da venerdì alle 22:00 a sabato alle 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sab, 26 settembre, 10:00</p>
<p class="timeline__titel">Fine della finestra</p>
<p>Termina la finestra indicata nell'e-mail ai clienti. La revoca formale della raccomandazione segue il 27 settembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dom, 27 settembre</p>
<p class="timeline__titel">Raccomandazione revocata</p>
<p>Kiteworks integra il comunicato stampa: la raccomandazione di spegnimento è revocata per tutti i clienti, i sistemi possono tornare a funzionare. Le istanze ospitate sono di nuovo in servizio. I clienti che gestiscono autonomamente Advanced Forms devono contattare il supporto.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situazione a lun, 28 settembre</p>
<p class="timeline__titel">Ancora irrisolto</p>
<p>Nessun advisory pubblico, nessun numero CVE, nessuna nuova versione, nessun indicatore, nessuna informazione sulla vulnerabilità e nessun resoconto di un attacco riuscito o tentato.</p>
</li>
</ol>

## Cosa si sa

La raccomandazione vale in tutto il mondo; l'e-mail indica la finestra per tutti i fusi orari, da AEST a PDT. Kiteworks consiglia di spegnere i sistemi già prima dell'inizio della finestra, anche se non sono raggiungibili da Internet.

Quasi tutto il resto resta finora aperto: non esiste un Security Advisory pubblico, nessun numero CVE, nessuna patch né indicazione su quali prodotti o versioni siano interessati. Il comunicato stampa cita le «federal intelligence authorities» come fonte, presumibilmente autorità federali statunitensi; non si sa quali. Al 28 settembre non vi sono voci in Security Updates né negli advisory GitHub di Kiteworks; l'ultima voce su GitHub risale al 27 maggio 2026. Sono pubbliche la dichiarazione citata sopra e il comunicato stampa del 25 settembre.

A TechCrunch, il CISO di Kiteworks Frank Balonis ha rilasciato la dichiarazione con lo stesso testo. Il BKA ha rifiutato di rilasciare dichiarazioni a heise per ragioni investigative, il BSI non ha risposto. L'FBI non ha voluto commentare a TechCrunch e un portavoce della CISA non ha voluto esprimersi pubblicamente. Secondo TechCrunch, un cliente del settore sanitario ha scollegato immediatamente il proprio server dalla rete, con limitazioni operative percepibili: i medici hanno potuto raggiungere i loro pazienti solo con ritardi temporanei. Secondo un ricercatore di sicurezza citato da TechCrunch, almeno 1000 sistemi Kiteworks sono raggiungibili da Internet; BornCity parla di oltre 1000 organizzazioni che hanno ricevuto l'avvertimento.

Il comunicato stampa differisce dall'e-mail ai clienti su un punto: parla di una finestra di spegnimento di nove ore, mentre l'advisory ai clienti di sei ore. Secondo il comunicato stampa, la raccomandazione riguarda solo installazioni gestite autonomamente (On-Premises, AWS, Azure). Secondo il produttore, non sono interessate le filiali Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai e 123FormBuilder.

## L'advisory ai clienti

L'e-mail ai clienti del 25 settembre contiene, oltre all'avvertimento, un calendario per fuso orario e istruzioni per i cluster. Convertendo gli orari in UTC, tutte le regioni hanno la stessa finestra dalle 02:00 alle 08:00 UTC.

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

1.  **Attivare la modalità manutenzione** in System Setup > Maintenance Mode, affinché nessun utente possa più accedere.

2.  **Creare un backup:** uno snapshot di ogni nodo oppure un backup del database Kiteworks (System Setup > Cluster Configuration > System Configuration). Viene mantenuto un solo backup del database; ogni nuovo backup sostituisce il precedente.

3.  **Registrare i ruoli:** in System Setup > Locations, la colonna Assigned Roles mostra quali nodi hanno il ruolo Application; il nodo Application primario è contrassegnato da un asterisco. Annotare i nodi e i relativi indirizzi IP, poiché serviranno per il riavvio.

4.  **Spegnere in questo ordine:** prima tutti i nodi senza ruolo Application, poi gli altri nodi Application e infine il nodo Application primario. È possibile farlo tramite la scheda Shut Down del rispettivo nodo oppure tramite la console dell'hypervisor (ad esempio VMware o AWS), se l'interfaccia Kiteworks non è più raggiungibile.

5.  **Riavviare nell'ordine inverso** tramite l'hypervisor, poiché la console di amministrazione è raggiungibile solo quando sono in esecuzione nodi sufficienti (Appendice E della Administrator Guide): prima il nodo Application primario, poi gli altri nodi Application uno alla volta e solo dopo che il precedente è completamente operativo, affinché i server di database possano formare un quorum. Quindi i server Storage, poi gli altri ruoli (Repositories Gateway, Search, SFTP, Antivirus) e infine i server Web.

6.  **Disattivare la modalità manutenzione** non appena tutti i nodi nel Cluster Health Dashboard della pagina di stato della console di amministrazione sono verdi.

Su richiesta, il supporto Kiteworks ha inoltre confermato che nessuna delle filiali di Kiteworks è interessata.

## Possibili cause: teorie

Finché Kiteworks non pubblicherà dettagli, la causa resta aperta. Le spiegazioni seguenti sono ipotesi deducibili dagli elementi noti; alcune sono discusse anche nei commenti all'articolo di heise. Nessuna è confermata.

Tre elementi delimitano lo scenario. Primo, l'avvertimento indica una finestra temporale fissa invece di uno spegnimento a tempo indeterminato fino alla patch. Secondo, devono essere scollegati anche i sistemi non raggiungibili da Internet. Terzo, la finestra è collocata alla stessa ora in tutto il mondo (dalle 02:00 alle 08:00 UTC), anziché durante la notte locale. Una classica vulnerabilità sfruttabile via Internet non spiegherebbe i primi due punti: basterebbe scollegare il sistema da Internet fino alla disponibilità della patch.

### 1. Le autorità conoscono un momento pianificato

Le autorità di contrasto talvolta vengono a conoscenza in anticipo del momento di una campagna pianificata, ad esempio tramite comunicazioni monitorate di un gruppo criminale o infrastrutture sequestrate. Gli sfruttamenti di massa di prodotti per lo scambio di file avvengono tipicamente in una finestra breve e coordinata, spesso durante fine settimana o festività, quando è presente meno personale. Kiteworks è il successore di Accellion, la cui File Transfer Appliance è stata attaccata proprio in questo modo nel 2020 e nel 2021, allora attribuito al gruppo Clop: i dati venivano sottratti tramite diverse vulnerabilità e le organizzazioni colpite venivano poi ricattate.

A favore depone la finestra ristretta nel fine settimana. Contro vi è l'obiezione sollevata da diversi commentatori su heise: l'avvertimento è stato inviato a tutti i clienti, quindi gli aggressori probabilmente ne sono al corrente e possono semplicemente rinviare l'attacco. Un rinvio darebbe però al produttore tempo per una patch.

### 2. Il produttore non conosce ancora la vulnerabilità

È anche possibile che Kiteworks, oltre alla segnalazione delle autorità, non disponga di dettagli tecnici e quindi non conosca né il componente interessato né una patch o una modifica di configurazione da raccomandare. In tal caso, lo spegnimento è l'unica misura efficace senza conoscere la vulnerabilità, e la fine fissata rappresenta un compromesso che i clienti sono più propensi ad accettare. Nei commenti di heise viene avanzata l'ipotesi che il produttore possa lasciare online singoli sistemi come esca durante la finestra per osservare l'attacco. Non vi sono prove a sostegno.

A favore depone il fatto che non siano indicati né un advisory né una mitigazione. Contro vi è il fatto che, secondo le proprie dichiarazioni, Kiteworks collabora con Mandiant e in caso di avvertimento delle autorità siano di norma disponibili almeno indicatori.

### 3. Una backdoor già installata con attivazione temporizzata

La raccomandazione di spegnere anche i sistemi interni si adatta a uno scenario in cui l'attacco non arriva dall'esterno, ma è già predisposto sugli appliance: ad esempio una backdoor derivante da una precedente compromissione, che si attiva a un orario prestabilito o contatta un server di controllo. Un sistema spento non può eseguire nulla in quel momento.

A favore depone il fatto che, in questo scenario, la raggiungibilità da Internet non svolga alcun ruolo. Contro vi è il fatto che, in questo caso, un produttore raccomanderebbe piuttosto una verifica della compromissione e una reinstallazione anziché un riavvio dopo sei ore.

### 4. Compromissione lato produttore

Un'altra via che raggiunge i sistemi interni è costituita dalle connessioni stabilite dall'appliance verso il produttore, ad esempio per aggiornamenti, verifica delle licenze o assistenza remota. Se tale canale è compromesso, un firewall non protegge dal traffico in entrata. In questo scenario, lo spegnimento darebbe al produttore una finestra per ripulire la propria infrastruttura, sostituire chiavi o certificati e consentire nuovamente le connessioni solo in seguito.

A favore depone l'orario uniforme a livello mondiale, compatibile con un'azione coordinata dal lato del produttore. Contro vi è il fatto che il produttore raccomanderebbe piuttosto di bloccare le connessioni in uscita anziché spegnere completamente i sistemi.

### 5. Misura complementare a un'azione delle autorità

Infine, è concepibile che le autorità intervengano nello stesso periodo contro l'infrastruttura degli aggressori e vogliano evitare che questi colpiscano rapidamente in risposta. Ciò spiegherebbe la breve finestra e il ruolo delle autorità di contrasto. Il fatto che il BKA rifiuti una dichiarazione per ragioni investigative indica indagini in corso, ma non dimostra questa variante.

### Critica alla comunicazione

Nei commenti di heise prevale lo scetticismo e le obiezioni sono oggettivamente comprensibili: senza informazioni sulla vulnerabilità non è possibile valutare se sarebbe bastato isolare Internet tramite firewall. Una finestra temporale senza una patch annunciata lascia aperto cosa valga dopo le 10:00. E un avvertimento inviato solo via e-mail ai clienti non raggiunge tutti gli operatori, ad esempio presso partner, fornitori di servizi o dopo cambi di personale. Indipendentemente dalla teoria corretta: chi gestisce Kiteworks dovrebbe controllare i log dopo il riavvio e monitorare i canali del produttore finché non sarà disponibile un advisory.

## Dopo la finestra: cosa possono fare ora gli operatori

Kiteworks ha revocato la raccomandazione di spegnimento il 27 settembre, ma non ha pubblicato dettagli tecnici. Non è quindi possibile valutare se e come il pericolo sia stato eliminato. Durante il riavvio e successivamente, sono opportuni i seguenti passi:

1.  **Verificare la versione:** tutti i nodi eseguono la versione 9.5.1? Secondo il produttore, tutte le vulnerabilità note sono risolte in questa versione.

2.  **Verificare lo stato del cluster:** nel Cluster Health Dashboard tutti i nodi dovrebbero essere verdi e la modalità manutenzione disattivata.

3.  **Analizzare i log:** esaminare accessi, azioni amministrative e download insoliti di file intorno alla finestra di spegnimento, in particolare sui sistemi che non sono stati spenti o lo sono stati solo tardi.

4.  **Limitare la raggiungibilità:** dove possibile, bloccare l'accesso da Internet all'interfaccia di amministrazione e abilitare solo i servizi necessari.

5.  **Advanced Forms:** chi gestisce autonomamente il modulo chiarisca il riavvio preventivamente con il supporto Kiteworks.

6.  **Monitorare i canali:** Security Updates, advisory GitHub, Newsroom ed e-mail ai clienti di Kiteworks, finché non sarà disponibile un advisory con dettagli tecnici.

## Fonti

1.  [heise online: Imminente attacco zero-day: KiteWorks sollecita i clienti a spegnere i server](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): prima notizia con estratti dall'e-mail ai clienti e la finestra temporale; aggiornamento del 25/09, ore 17:41, con la risposta del BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versione inglese con il testo originale del CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): canale ufficiale del produttore, al 28/09/2026 senza voci relative all'avvertimento.

4.  [Kiteworks: Security Advisories su GitHub](https://github.com/kiteworks/security-advisories/security): elenco degli advisory del produttore, al 28/09/2026 ultima voce del 27/05/2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): comunicazioni ufficiali, dal 25/09/2026 con il comunicato stampa sullo spegnimento.

6.  [Forum heise: commenti all'articolo](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussione dei lettori con le obiezioni sulla finestra temporale fissa e sullo spegnimento dei sistemi interni, nonché la teoria dell'esca.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): advisory sullo sfruttamento di Accellion FTA nel 2020/2021 con successiva estorsione; Accellion è il precedente nome di Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): dichiarazione del produttore sulla collaborazione con Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): dichiarazione del CISO, orario di invio dell'avvertimento, reazioni di FBI e CISA (aggiunta), conseguenze presso un cliente, numero di sistemi raggiungibili da Internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (comunicato stampa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicazione ufficiale del 25/09/2026 con informazioni sulle istanze ospitate, la versione 9.5.1 e le filiali non interessate; integrata con la nota del 27/09/2026 secondo cui la raccomandazione di spegnimento è stata revocata.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): finestra temporale per regione e contesto dei precedenti attacchi a prodotti per lo scambio di file.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valutazione di watchTowr sull'insolita raccomandazione di spegnimento.

13.  [BornCity: Kiteworks: oltre 1.000 organizzazioni devono spegnere i server](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): numero delle organizzazioni avvisate e settori nell'area di lingua tedesca.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): contesto degli attacchi Clop ad Accellion nel 2020/2021 e citazione di watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): articolo del 28/09/2026 sulla revoca della raccomandazione e sul funzionamento delle istanze ospitate.
