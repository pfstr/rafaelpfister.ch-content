---
title: "Kiteworks: il produttore consiglia lo spegnimento il 26 settembre - Ciò che si sa finora"
navTitle: "Spegnimento di Kiteworks"
description: "Kiteworks ha chiesto ai propri clienti di spegnere tutti i sistemi sabato 26/09/2026, dalle 04:00 alle 10:00. Rapporto finale: durante lo spegnimento, il produttore ha individuato e corretto una vulnerabilità critica senza CVE; il 30/09 sono seguiti 125 advisory, tra cui CVE-2026-54154 (CVSS 10.0) nell'Email Protection Gateway. Totemomail non è interessato."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min di lettura"
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
slug: "kiteworks-il-produttore-raccomanda-lo-spegnimento-il-26-settembre-cosa-si-sa-finora"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:53:06.532Z
translationReview: required
url: https://rafaelpfister.ch/it/blog/kiteworks-il-produttore-raccomanda-lo-spegnimento-il-26-settembre-cosa-si-sa-finora
---

# Kiteworks: il produttore consiglia lo spegnimento il 26 settembre - Ciò che si sa finora

Il 25 settembre 2026, Kiteworks ha chiesto ai propri clienti via e-mail di spegnere tutti i sistemi Kiteworks sabato 26 settembre, dalle 04:00 alle 10:00 (ora dell'Europa centrale). Secondo la comunicazione del CISO Frank Balonis, il produttore aveva ricevuto indicazioni dalle autorità di contrasto secondo cui quel fine settimana avrebbe potuto verificarsi un attacco ai sistemi Kiteworks. L'assistenza clienti ha motivato lo spegnimento con la protezione da possibili attacchi zero-day. heise online ha confermato telefonicamente presso l'assistenza l'autenticità del messaggio.

<div class="update-hinweis">
<p class="update-hinweis__titel">Rapporto finale del 7 ottobre 2026</p>
<p>Dal punto di vista del produttore, l'incidente è concluso. La raccomandazione di spegnimento non è più valida dal 27 settembre e finora non è stato reso noto alcun attacco a sistemi Kiteworks o dei clienti. I risultati principali:</p>
<ul>
<li><strong>Vulnerabilità critica individuata durante lo spegnimento:</strong> Secondo il comunicato stampa del 28 settembre, nell'analisi con le autorità federali Kiteworks si è imbattuta in una vulnerabilità critica finora sconosciuta in una funzione attivata presso meno dell'1% dei clienti. Durante la finestra di spegnimento il produttore ha sviluppato e distribuito una correzione e ha inoltre attivato un ulteriore livello di protezione in tutti gli ambienti. Non è stato pubblicato quale funzione fosse interessata e a tutt'oggi non esiste un numero CVE per essa.</li>
<li><strong>125 advisory il 30 settembre:</strong> Due giorni dopo, Kiteworks ha pubblicato su GitHub 125 security advisory per Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28) e MFT Server (3); 12 sono critici e 49 alti. Tutti sono risolti nelle versioni fino alla 9.5.1 inclusa; la maggior parte è stata segnalata attraverso il programma bug bounty su YesWeHack. Allo stato attuale, non hanno nulla a che fare con la vulnerabilità della finestra di spegnimento.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> La vulnerabilità più grave riguarda l'Email Protection Gateway prima della versione 9.4.1. Un attaccante non autenticato può eseguire codice con privilegi root tramite endpoint pubblicamente accessibili. 10 dei 12 advisory critici riguardano l'Email Protection Gateway.</li>
<li><strong>Nessuno sfruttamento noto:</strong> Non esistono segnalazioni di attacchi per nessuna delle vulnerabilità; al 7 ottobre il catalogo CISA-KEV non contiene alcuna voce Kiteworks del 2026. Secondo BleepingComputer, Shadowserver conta poco meno di 400 istanze Kiteworks raggiungibili da Internet.</li>
<li><strong>Totemomail:</strong> Non compare in nessuno degli advisory e, secondo il produttore, non è stato interessato dallo spegnimento.</li>
</ul>
<p><strong>Interventi necessari:</strong> Chi gestisce Kiteworks autonomamente dovrebbe aggiornare tutti i componenti alla versione 9.5.1; l'Email Protection Gateway ha la priorità. I dettagli sono nella sezione <a href="#abschlussbericht">Rapporto finale</a>.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Aggiornamento del 28 settembre 2026: Kiteworks ritira la raccomandazione di spegnimento</p>
<p>Kiteworks ha integrato il comunicato stampa con un avviso: dal 27 settembre la raccomandazione di spegnimento non è più valida per tutti i clienti.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Chi non ha ancora riavviato i propri sistemi può farlo ora. Chi gestisce autonomamente Advanced Forms deve contattare l'assistenza Kiteworks prima del riavvio. Le istanze ospitate da Kiteworks sono nuovamente operative. Continuano a non esserci un numero CVE, una nuova versione oltre la 9.5.1, indicatori di compromissione né informazioni sul fatto che sia stato tentato un attacco o su cosa abbia motivato l'avvertimento. La pagina degli aggiornamenti di sicurezza e gli advisory su GitHub sono invariati.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Aggiornamento del 25 settembre 2026: dichiarazione di Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail non è interessato.</strong> Resta da chiarire se Kiteworks EPG (Email Protection Gateway) sia interessato.</p>
</div>

## Cronologia

Tutti gli orari sono espressi nell'ora legale dell'Europa centrale (CEST). Laddove non è indicato un orario, non è disponibile un'indicazione temporale affidabile.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Advisory ai clienti</p>
<p>Il CISO Frank Balonis informa i clienti via e-mail delle indicazioni delle autorità di contrasto su un possibile attacco nel fine settimana e raccomanda uno spegnimento di sei ore. Secondo l'advisory, tutte le vulnerabilità note sono risolte nella versione 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Primi resoconti dei media</p>
<p>heise online riferisce che l'assistenza Kiteworks conferma l'autenticità del messaggio e motiva lo spegnimento con la protezione da possibili attacchi zero-day. Poco dopo seguono TechCrunch, BleepingComputer, Computer Weekly e altri.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre, 17:41</p>
<p class="timeline__titel">Il BKA non commenta</p>
<p>heise aggiunge: il BKA rifiuta di commentare per ragioni investigative. Il BSI non risponde, l'FBI rinuncia a commentare nei confronti di TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Ven, 25 settembre</p>
<p class="timeline__titel">Dichiarazione e comunicato stampa</p>
<p>Kiteworks definisce lo spegnimento una misura precauzionale senza compromissioni note. Il comunicato stampa cita le “federal intelligence authorities” come fonte ed elenca le filiali non interessate, tra cui totemo. Il produttore spegne direttamente le istanze ospitate da Kiteworks.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sab, 26 settembre, dalle 04:00 alle 10:00</p>
<p class="timeline__titel">Finestra di spegnimento</p>
<p>La finestra si svolge simultaneamente in tutto il mondo: dalle 02:00 alle 08:00 UTC, a Sydney dalle 12:00 alle 18:00, a New York da venerdì alle 22:00 a sabato alle 04:00.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sab, 26 settembre, 10:00</p>
<p class="timeline__titel">Fine della finestra</p>
<p>Termina la finestra indicata nell'e-mail ai clienti. La revoca formale della raccomandazione segue il 27 settembre.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Dom, 27 settembre</p>
<p class="timeline__titel">Raccomandazione ritirata</p>
<p>Kiteworks integra il comunicato stampa: la raccomandazione di spegnimento è ritirata per tutti i clienti e i sistemi possono tornare operativi. Le istanze ospitate sono nuovamente in funzione. I clienti con Advanced Forms gestiti autonomamente devono contattare l'assistenza.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Lun, 28 settembre</p>
<p class="timeline__titel">Vulnerabilità critica individuata e corretta</p>
<p>In un ulteriore comunicato stampa, Kiteworks riferisce che durante lo spegnimento, nel lavoro con le autorità federali, è stata individuata una vulnerabilità critica finora sconosciuta. Riguarda una funzione attivata presso meno dell'1% dei clienti. La correzione e l'ulteriore livello di protezione sono stati distribuiti; non vi sono indicazioni di una compromissione. Il produttore non indica né la funzione né il numero CVE.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mer, 30 settembre, dalle 18:38</p>
<p class="timeline__titel">125 security advisory su GitHub</p>
<p>Kiteworks pubblica 125 advisory per Core, Email Protection Gateway, Secure Data Forms e MFT Server, tutti risolti fino alla versione 9.5.1. La vulnerabilità più grave è CVE-2026-54154 nell'Email Protection Gateway prima della 9.4.1 (CVSS 10.0, esecuzione di codice con privilegi root senza autenticazione).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Gio, 1 ottobre</p>
<p class="timeline__titel">Resoconti dei media e advisory MS-ISAC</p>
<p>BleepingComputer, SecurityOnline e altri riferiscono degli advisory; l'MS-ISAC (Center for Internet Security) pubblica un proprio advisory su CVE-2026-54154. Non è noto alcuno sfruttamento.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">Situazione a mer, 7 ottobre</p>
<p class="timeline__titel">Conclusione</p>
<p>Nessuna segnalazione di un attacco riuscito o tentato, nessuna voce Kiteworks nel catalogo CISA-KEV. Restano ignoti la funzione interessata, un numero CVE per la vulnerabilità della finestra di spegnimento e il contesto dell'avvertimento delle autorità.</p>
</li>
</ol>

## Ciò che si sa

La raccomandazione vale in tutto il mondo; l'e-mail indica la finestra temporale per tutti i fusi orari, dall'AEST al PDT. Kiteworks consiglia di spegnere i sistemi già prima dell'inizio della finestra, anche se non sono raggiungibili da Internet.

Fino al 28 settembre quasi tutto il resto era ignoto: non esistevano un security advisory pubblico, un numero CVE, una patch né indicazioni sui prodotti o sulle versioni interessati. Il comunicato stampa cita le “federal intelligence authorities” come fonte, presumibilmente autorità federali statunitensi; quali siano resta ignoto a tutt'oggi. Negli advisory GitHub di Kiteworks, fino ad allora l'ultima voce risaliva al 27 maggio 2026; gli advisory del 30 settembre sono riassunti nella sezione [Rapporto finale](#abschlussbericht).

Nei confronti di TechCrunch, il CISO di Kiteworks Frank Balonis ha rilasciato la dichiarazione con lo stesso testo. Il BKA ha rifiutato di commentare a heise per ragioni investigative, mentre il BSI non ha risposto. L'FBI non ha voluto commentare a TechCrunch e un portavoce della CISA non ha voluto esprimersi pubblicamente. Secondo TechCrunch, un cliente del settore sanitario ha immediatamente scollegato il proprio server dalla rete, con limitazioni operative evidenti: i medici hanno potuto raggiungere i pazienti solo con ritardi temporanei. Secondo un ricercatore di sicurezza citato da TechCrunch, almeno 1.000 sistemi Kiteworks sono raggiungibili da Internet; BornCity parla di oltre 1.000 organizzazioni che hanno ricevuto l'avvertimento.

Il comunicato stampa differisce dall'e-mail ai clienti su un punto: parla di una finestra di spegnimento di nove ore, mentre l'advisory ai clienti parla di sei ore. Secondo il comunicato stampa, la raccomandazione riguarda solo le installazioni gestite autonomamente (on-premises, AWS, Azure). Secondo il produttore, non sono interessate le filiali Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai e 123FormBuilder.

## L'advisory ai clienti

L'e-mail ai clienti del 25 settembre contiene, oltre all'avvertimento, un calendario per fuso orario e istruzioni per i cluster. Convertendo gli orari in UTC, per tutte le regioni risulta la stessa finestra dalle 02:00 alle 08:00 UTC.

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

Per i cluster con più server, Kiteworks stabilisce un ordine preciso:

1.  **Attivare la modalità di manutenzione** in System Setup > Maintenance Mode, affinché nessun utente possa più accedere.

2.  **Creare un backup:** uno snapshot di ogni nodo oppure un backup del database Kiteworks (System Setup > Cluster Configuration > System Configuration). Viene conservato un solo backup del database; ogni nuovo backup sostituisce quello precedente.

3.  **Rilevare i ruoli:** in System Setup > Locations, la colonna Assigned Roles mostra quali nodi hanno il ruolo Application; il nodo Application primario è contrassegnato da un asterisco. Annotare i nodi e i relativi indirizzi IP, che servono per il riavvio.

4.  **Spegnere in questo ordine:** prima tutti i nodi senza ruolo Application, poi gli altri nodi Application, infine il nodo Application primario. È possibile farlo tramite la scheda Shut Down del rispettivo nodo oppure dalla console dell'hypervisor (ad esempio VMware o AWS), se l'interfaccia Kiteworks non è più raggiungibile.

5.  **Riavviare nell'ordine inverso** tramite l'hypervisor, poiché la console di amministrazione è raggiungibile solo quando sono in esecuzione abbastanza nodi (Appendice E della Administrator Guide): prima il nodo Application primario, poi gli altri nodi Application singolarmente e solo quando quello precedente è completamente operativo, affinché i server del database possano formare un quorum. Quindi i server di storage, poi gli altri ruoli (Repositories Gateway, Search, SFTP, Antivirus) e infine i server web.

6.  **Disattivare la modalità di manutenzione** non appena tutti i nodi sono verdi nel Cluster Health Dashboard sulla pagina di stato della console di amministrazione.

Su richiesta, l'assistenza Kiteworks ha inoltre confermato che nessuna delle filiali di Kiteworks è interessata.

## Possibili cause: teorie

Questa sezione è stata redatta prima del 28 settembre; la valutazione in base allo stato attuale si trova nel [Rapporto finale](#abschlussbericht). Le spiegazioni seguenti sono ipotesi deducibili dai dati noti; alcune vengono discusse anche nei commenti alla notizia di heise. Nessuna è confermata.

Tre elementi circoscrivono il quadro. In primo luogo, l'avvertimento indica una finestra temporale fissa anziché uno spegnimento a tempo indeterminato fino alla disponibilità della patch. In secondo luogo, devono essere scollegati anche i sistemi non raggiungibili da Internet. In terzo luogo, la finestra cade ovunque nel mondo alla stessa ora (dalle 02:00 alle 08:00 UTC), invece che durante la notte locale. Una classica vulnerabilità sfruttabile via Internet non spiegherebbe i primi due punti: basta scollegare il sistema da Internet finché non è disponibile la patch.

### 1. Le autorità conoscono un momento pianificato

Le autorità di contrasto apprendono occasionalmente in anticipo il momento di una campagna pianificata, ad esempio da comunicazioni monitorate di un gruppo criminale o da infrastrutture sequestrate. Gli sfruttamenti di massa di prodotti per lo scambio di file avvengono tipicamente in una finestra breve e coordinata, spesso nei fine settimana o nei giorni festivi, quando è presente meno personale. Kiteworks è il successore di Accellion, la cui File Transfer Appliance è stata attaccata proprio in questo modo nel 2020 e nel 2021, all'epoca attribuito al gruppo Clop: attraverso diverse vulnerabilità sono stati sottratti dati, quindi le organizzazioni colpite sono state ricattate.

A favore depone la finestra ristretta nel fine settimana. Contro depone l'obiezione sollevata da vari commentatori su heise: l'avvertimento è stato inviato a tutti i clienti, quindi gli attaccanti dovrebbero esserne a conoscenza e potrebbero semplicemente rimandare l'attacco. Un rinvio darebbe però al produttore tempo per una patch.

### 2. Il produttore non conosce ancora la vulnerabilità

È anche possibile che, oltre all'indicazione delle autorità, Kiteworks non disponga di dettagli tecnici, quindi non conosca né il componente interessato né possa consigliare una patch o una modifica di configurazione. In tal caso, lo spegnimento è l'unica misura efficace senza conoscere la vulnerabilità, e la fine fissa è un compromesso che i clienti sono più inclini ad accettare. Nei commenti di heise viene avanzata l'ipotesi che il produttore possa lasciare online alcuni sistemi come esche durante la finestra, per osservare l'attacco. Non esistono prove in tal senso.

A favore depone il fatto che non siano stati indicati né un advisory né una mitigazione. Contro depone il fatto che Kiteworks affermi di collaborare con Mandiant e che, in caso di avvertimento delle autorità, siano di norma disponibili almeno indicatori.

### 3. Una backdoor già installata con attivazione temporizzata

La raccomandazione di spegnere anche i sistemi interni si adatta a uno scenario in cui l'attacco non arriva dall'esterno, ma è già predisposto sulle appliance: ad esempio una backdoor da una compromissione precedente, che si attiva in un momento fisso o contatta un server di controllo. Un sistema spento non può eseguire nulla in quel momento.

A favore depone che, in questo scenario, la raggiungibilità da Internet non svolga alcun ruolo. Contro depone il fatto che in tal caso un produttore raccomanderebbe piuttosto una verifica di compromissione e una reinstallazione, anziché un riavvio dopo sei ore.

### 4. Compromissione presso il produttore

Un'altra via per raggiungere i sistemi interni sono le connessioni stabilite dall'appliance verso il produttore, ad esempio per aggiornamenti, verifica delle licenze o assistenza remota. Se un canale simile è compromesso, un firewall non protegge dal traffico in entrata. In questo scenario, lo spegnimento darebbe al produttore una finestra per bonificare la propria infrastruttura, sostituire chiavi o certificati e consentire nuovamente le connessioni solo in seguito.

A favore depone l'orario uniforme a livello mondiale, compatibile con un'azione coordinata presso il produttore. Contro depone il fatto che il produttore raccomanderebbe piuttosto di bloccare le connessioni in uscita, anziché spegnere del tutto i sistemi.

### 5. Misura di accompagnamento a un'azione delle autorità

Infine, è ipotizzabile che le autorità agiscano nello stesso periodo contro l'infrastruttura degli attaccanti e vogliano impedire che questi colpiscano rapidamente in risposta. Ciò spiegherebbe la finestra breve e il ruolo delle autorità di contrasto. Il fatto che il BKA rifiuti di commentare per ragioni investigative indica indagini in corso, ma non prova questa variante.

### Critiche alla comunicazione

Nei commenti di heise prevale lo scetticismo, e le obiezioni sono comprensibili nel merito: senza informazioni sulla vulnerabilità non è possibile valutare se sarebbe stato sufficiente scollegare Internet tramite firewall. Una finestra temporale senza una patch annunciata lascia aperto cosa valga dopo le 10:00. E un avvertimento inviato solo via e-mail ai clienti non raggiunge tutti gli operatori, ad esempio presso partner, fornitori di servizi o dopo cambiamenti del personale. Indipendentemente dalla teoria corretta: chi gestisce Kiteworks dovrebbe controllare i log dopo il riavvio e monitorare i canali del produttore fino alla pubblicazione di un advisory.

## Rapporto finale

Al 7 ottobre 2026, dal punto di vista del produttore l'incidente è concluso. Gli eventi successivi alla finestra di spegnimento si possono separare in due filoni: la vulnerabilità individuata durante lo spegnimento e la pubblicazione in blocco di advisory due giorni dopo.

### La vulnerabilità della finestra di spegnimento

Il 28 settembre Kiteworks ha pubblicato un secondo comunicato stampa. Secondo il comunicato, il produttore ha collaborato con le autorità federali per tutto il fine settimana; in tale contesto è stata scoperta una vulnerabilità critica finora sconosciuta, limitata a una funzione attivata presso meno dell'1% dei clienti. Durante la finestra di spegnimento Kiteworks ha sviluppato e distribuito una correzione e ha inoltre attivato un ulteriore livello di protezione in tutti gli ambienti. Il monitoraggio continuo non avrebbe mostrato attività sospette e non vi sarebbero indicazioni di compromissione di sistemi Kiteworks o dei clienti. Tutti gli altri prodotti Kiteworks non sarebbero interessati.

Non sono stati pubblicati la funzione interessata, un numero CVE, le versioni che includono la correzione e se le installazioni gestite autonomamente abbiano ricevuto automaticamente la correzione. La revoca del 27 settembre conteneva una sola eccezione: i clienti con Advanced Forms gestiti autonomamente dovevano contattare l'assistenza prima del riavvio. Kiteworks non ha confermato se questa fosse la funzione interessata.

Riguardo alle teorie precedenti: il comunicato stampa descrive una vulnerabilità individuata solo durante la finestra. Ciò è compatibile con la teoria 2 (il produttore non conosceva prima la vulnerabilità) insieme alla teoria 1 (le autorità conoscevano un momento pianificato). Non vi sono conferme per le teorie dalla 3 alla 5. Resta ignoto cosa sapessero concretamente le autorità e se sia stato tentato un attacco.

### 125 security advisory del 30 settembre

Il 30 settembre, dalle 18:38, Kiteworks ha pubblicato su GitHub 125 security advisory in una sola volta. La distribuzione è la seguente:

| Prodotto | Advisory | di cui critici |
|---|---|---|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Totale** | **125** | **12** |

Per gravità, vi sono 12 classificazioni critiche, 49 alte, 52 medie e 12 basse. Tutte le vulnerabilità sono risolte nelle versioni fino alla 9.5.1 inclusa; le voci più vecchie riguardano la versione 9.2.1. Si tratta quindi di una divulgazione successiva di correzioni già distribuite, non di una nuova versione. Gli advisory indicano prevalentemente come segnalanti partecipanti al programma bug bounty su YesWeHack. Kiteworks non stabilisce alcun collegamento con la vulnerabilità della finestra di spegnimento; una settimana dopo la finestra, lo stato degli advisory continua a corrispondere all'affermazione del 25 settembre secondo cui tutte le vulnerabilità note sono risolte nella 9.5.1.

Gli advisory critici:

| CVE | Prodotto | CVSS 3.1 | risolto dalla versione | Impatto |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Esecuzione di codice con privilegi root senza autenticazione |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Acquisizione dell'account |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Acquisizione dell'account |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Acquisizione dell'account tramite reimpostazione della password |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Acquisizione dell'account |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Acquisizione dell'account |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Aggiramento delle funzioni di sicurezza |
| CVE-2026-102095, CVE-2026-102102 fino a 102105 | EPG | 9.1 | 9.5.0 | Accesso a risorse di rete interne (SSRF) |

CVE-2026-54154 è la vulnerabilità più grave: secondo l'advisory, una combinazione di errori nella convalida dell'input in endpoint pubblicamente accessibili dell'Email Protection Gateway consente a un attaccante non autenticato di eseguire codice e, tramite ulteriori debolezze locali, ottenere privilegi root sull'appliance. L'MS-ISAC ha pubblicato un proprio advisory il 1° ottobre. Per gli amministratori di posta, il gateway è la parte rilevante della pubblicazione: tipicamente si trova direttamente nel flusso di posta ed è raggiungibile da Internet.

### Sfruttamento e diffusione

Non vi sono segnalazioni di sfruttamento o exploit pubblici per nessuna delle vulnerabilità. Al 7 ottobre, il catalogo CISA-KEV contiene solo le quattro voci Accellion FTA del 2021. Secondo BleepingComputer, Shadowserver conta poco meno di 400 istanze Kiteworks raggiungibili da Internet; non si sa quante di queste eseguano già la versione 9.5.1. Totemomail non compare in nessuno degli advisory.

## Dopo la finestra: cosa possono fare ora gli operatori

La raccomandazione di spegnimento è stata ritirata e le vulnerabilità note sono risolte nella versione 9.5.1. Per le installazioni gestite autonomamente sono sensati i seguenti passaggi:

1.  **Verificare la versione:** tutti i nodi e tutti i componenti (Core, Email Protection Gateway, Secure Data Forms, MFT Server) eseguono la versione 9.5.1? Un Email Protection Gateway precedente alla 9.4.1 è interessato da CVE-2026-54154 e dovrebbe essere aggiornato per primo.

2.  **Verificare lo stato del cluster:** nel Cluster Health Dashboard tutti i nodi dovrebbero essere verdi e la modalità di manutenzione dovrebbe essere disattivata.

3.  **Analizzare i log:** esaminare accessi, azioni degli amministratori e download insoliti di file intorno alla finestra di spegnimento, in particolare sui sistemi che non sono stati spenti o lo sono stati tardi.

4.  **Limitare la raggiungibilità:** ove possibile, bloccare l'accesso da Internet all'interfaccia di amministrazione e abilitare solo i servizi necessari.

5.  **Advanced Forms:** chi gestisce autonomamente il modulo e non ha ancora contattato l'assistenza chiarisca con Kiteworks se la correzione della finestra di spegnimento è arrivata nella propria installazione.

6.  **Confrontare gli advisory:** gli advisory GitHub possono essere filtrati per prodotto (prefisso `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). Per ogni componente utilizzato, verificare se la versione installata è inferiore alla rispettiva versione corretta indicata.

7.  **Monitorare i canali:** advisory GitHub, Newsroom ed e-mail ai clienti di Kiteworks, nel caso il produttore pubblichi ancora un advisory con numero CVE per la vulnerabilità della finestra di spegnimento. Nuove CVE per Kiteworks Email Protection Gateway e Totemomail sono riportate anche dal [CVE tracker](/cve) di questa pagina; lì è possibile abbonarsi a un'e-mail di avviso.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Supporto per l'aggiornamento</p>
<p>Se avete bisogno di aiuto per aggiornare un gateway Kiteworks o Totemomail, ad esempio per reindirizzare il flusso di posta durante la finestra di manutenzione o per analizzare i log, utilizzate il <a href="https://adeptio.ch/">modulo di contatto su adeptio.ch</a>.</p>
</div>

## Fonti

1.  [heise online: Imminente attacco zero-day: KiteWorks esorta i clienti a spegnere i server](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): prima notizia con estratti dall'e-mail ai clienti e dalla finestra temporale; aggiornamento del 25/09, ore 17:41, con la risposta del BKA.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): versione inglese con il testo originale del CISO.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): precedente pagina degli aggiornamenti del produttore, al 07/10/2026 senza voce sull'avvertimento o sugli advisory del 30/09/2026.

4.  [Kiteworks: Security Advisories su GitHub](https://github.com/kiteworks/security-advisories/security): elenco degli advisory del produttore, fino al 28/09/2026 l'ultima voce del 27/05/2026; il 30/09/2026 125 nuovi advisory per Core, EPG, SDF e MFT. I numeri in questo articolo sono stati conteggiati tramite API GitHub.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): comunicazioni ufficiali, dal 25/09/2026 con il comunicato stampa sullo spegnimento.

6.  [Forum heise: commenti alla notizia](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): discussione dei lettori con le obiezioni alla finestra temporale fissa e allo spegnimento dei sistemi interni, nonché la teoria dell'esca.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): advisory sullo sfruttamento di Accellion FTA nel 2020/2021 con successiva estorsione; Accellion è il precedente nome di Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): indicazione del produttore sulla collaborazione con Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): dichiarazione del CISO, orario di invio dell'avvertimento, reazioni di FBI e CISA (aggiunta), impatti presso un cliente, numero dei sistemi raggiungibili da Internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (comunicato stampa)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): comunicazione ufficiale del 25/09/2026 con informazioni sulle istanze ospitate, la versione 9.5.1 e le filiali non interessate; integrata con l'avviso del 27/09/2026 che la raccomandazione di spegnimento è stata ritirata.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): finestra temporale per regione e contestualizzazione di precedenti attacchi a prodotti per lo scambio di file.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): valutazione di watchTowr sulla insolita raccomandazione di spegnimento.

13.  [BornCity: Kiteworks: oltre 1.000 organizzazioni devono spegnere i server](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): numero delle organizzazioni avvisate e dei settori nell'area di lingua tedesca.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): contestualizzazione degli attacchi di Clop ad Accellion nel 2020/2021 e citazione di watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): articolo del 28/09/2026 sulla revoca della raccomandazione e sul funzionamento delle istanze ospitate.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (comunicato stampa)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): comunicazione del 28/09/2026 sulla vulnerabilità critica individuata durante lo spegnimento, sulla correzione e sull'ulteriore livello di protezione.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): sintesi del secondo comunicato stampa con citazioni del CISO.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): informazioni del produttore sull'esecuzione di codice nell'Email Protection Gateway prima della 9.4.1, CVSS 10.0, segnalazione tramite YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): articolo del 01/10/2026 su CVE-2026-54154 e sul numero di istanze conteggiate da Shadowserver.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): advisory del Center for Internet Security del 01/10/2026 con raccomandazioni.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): contestualizzazione delle vulnerabilità di acquisizione dell'account in Core, inclusa CVE-2026-102115; il conteggio differisce dall'elenco degli advisory.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): al 07/10/2026 solo le quattro voci Accellion FTA del 2021, nessuna voce Kiteworks del 2026.
