---
title: "Ghost Sender in Exchange Online: un record MX non è un firewall"
navTitle: "Ghost Sender"
description: "La consegna diretta a Exchange Online aggira un gateway a monte se il tenant non la blocca esplicitamente. Il rischio è reale, ma la causa è una configurazione incompleta del flusso di posta."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min di lettura"
themen:
  - "microsoft-365-exchange"
slug: "ghost-sender-in-exchange-online-un-record-mx-non-e-un-firewall"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/it/blog/ghost-sender-in-exchange-online-un-record-mx-non-e-un-firewall"
---

# Ghost Sender in Exchange Online: un record MX non è un firewall

![Un amministratore fantasma tiene aperta nel data center la porta accanto al gateway di sicurezza, mentre le e-mail passano direttamente nella casella di posta aggirando il filtro.](../images/ghost-admin.png)

La possibilità di attacco descritta da InfoGuard Labs come «Ghost Sender» è reale: un aggressore può aggirare un gateway e-mail a monte e consegnare direttamente a Exchange Online. Il presupposto, tuttavia, è che il tenant continui ad accettare questo percorso diretto. Non si tratta di una vulnerabilità universale di Exchange Online, bensì di una topologia del flusso di posta protetta in modo incompleto.

Un Mail Transfer Agent che gestisce caselle di posta per un dominio accetta in linea di principio connessioni SMTP da Internet. Il record MX indica ai mittenti regolari il percorso di consegna desiderato. Non è né una regola firewall né una lista di accesso e non impedisce a nessuno di contattare direttamente un endpoint noto di Exchange Online.

## Cosa mostra effettivamente «Ghost Sender»

Lo scenario descritto da [InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) è il seguente:

1. Un'organizzazione gestisce le proprie caselle di posta in Exchange Online.
2. Il record MX pubblico punta a un Secure Email Gateway a monte.
3. L'endpoint Exchange Online in `*.mail.protection.outlook.com` rimane direttamente raggiungibile da Internet.
4. L'amministratore non ha limitato Exchange Online in modo che solo il gateway a monte possa consegnarvi messaggi.
5. Un aggressore ignora il record MX e recapita il proprio messaggio direttamente a Exchange Online.

Il percorso previsto è quindi:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Tuttavia, è rimasto aperto questo percorso:

```text
Angreifer -> Exchange Online -> Postfach
```

Si tratta di una configurazione errata da prendere sul serio. Il filtro a monte può essere aggirato tramite questo percorso; mittenti contraffatti, phishing e frodi del CEO risultano così notevolmente più facili. Va riconosciuto a InfoGuard il merito di aver reso visibile il problema, averne studiato la diffusione e pubblicato un test facile da usare.

Ma dov'è esattamente il difetto del prodotto?

Anche l'enfasi mediatica aiuta poco a inquadrare la questione. [Heise titola che Exchange Online lascia passare e-mail contraffatte «senza problemi»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), sebbene siano interessate solo determinate configurazioni di terze parti e ibride non completamente rafforzate. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) lo formula in modo molto più preciso: non una falla di sicurezza in senso stretto, bensì un problema di progettazione e configurazione.

## «An MTA is doing MTA-Things»

Ogni tenant Exchange Online dispone di un endpoint SMTP pubblico. Questo endpoint non è un segreto e non deve esserlo. Microsoft stessa spiega che Exchange Online, per impostazione predefinita, accetta messaggi indirizzati direttamente alle caselle di posta ospitate: [è semplicemente il funzionamento dell'e-mail](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Anche [SMTP stesso descrive il record MX come un meccanismo per individuare il sistema di destinazione regolare](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Da ciò non deriva alcun obbligo per il server di destinazione di rifiutare connessioni attraverso qualsiasi altro host raggiungibile. Un aggressore non deve attenersi al percorso indicato. Se un ulteriore MTA è raggiungibile, conosce il dominio destinatario e accetta il messaggio, verrà provato, in modo molto simile a come gli spammer tentano da decenni di contattare sistemi MX di backup meno protetti.

Chi antepone un filtro di terze parti modifica la topologia standard. Da «Exchange Online è il mio gateway e-mail Internet» si passa a «solo il mio gateway di terze parti può trasferire e-mail Internet a Exchange Online». Questa nuova `Trust-Border` non nasce da una voce DNS. Deve essere applicata esplicitamente sul sistema ricevente.

Microsoft documenta esattamente questo: con un MX esterno deve essere creato un connettore in entrata di tipo `Partner`, che per `SenderDomains *` accetta solo il certificato o gli indirizzi IP di origine del servizio a monte. I messaggi consegnati direttamente aggirando il gateway vengono quindi rifiutati. È scritto testualmente nella guida Microsoft [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Anche Frank Carius descrive dettagliatamente questo «ingresso laterale» nella [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM e DMARC non sono buttafuori

InfoGuard mostra messaggi per i quali SPF, DKIM e DMARC falliscono e che arrivano comunque nella casella di posta. Sembra spettacolare, ma non è un «bypass» crittografico di questi meccanismi. Le e-mail non superano affatto i controlli con successo. Forniscono `fail`. Ciò che conta è quale azione locale il sistema ricevente deduce da questo risultato.

SPF verifica se un sistema è autorizzato a inviare per il mittente dell'envelope. DKIM verifica una firma. DMARC collega questi risultati al dominio del mittente visibile e pubblica un trattamento desiderato. Persino l'attuale [standard DMARC RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) afferma esplicitamente che il destinatario può prendere in considerazione questo trattamento desiderato, ma non è obbligato a farlo. DMARC è un segnale importante, ma non un controllo di accesso alla rete.

Con un gateway a monte si aggiunge il fatto che Exchange Online vede innanzitutto l'indirizzo IP di tale gateway e non quello del mittente originale. A questo serve [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): ricostruisce la fonte originale e migliora le valutazioni SPF, DKIM, DMARC, anti-spoofing e anti-phishing. Tuttavia, anche Enhanced Filtering non è una serratura. Non sostituisce il connettore partner restrittivo.

La configurazione errata diventa particolarmente evidente quando un amministratore indebolisce o elimina il controllo EOP tramite un bypass SCL, perché il prodotto a monte dovrebbe già filtrare, lasciando contemporaneamente aperta la consegna diretta da Internet. In tal caso non gli è stato «aggirato» un meccanismo di protezione, ma ha consapevolmente previsto che uno dei due ingressi non disponga più di una protezione efficace.

Si può certamente criticare Microsoft se un messaggio con un errore di autenticazione chiaramente visibile finisce nella posta in arrivo senza avvisi. Si possono criticare la semantica dei tipi di connettore, la documentazione e gli avvisi mancanti nel Configuration Analyzer. Sono tutti punti legittimi. L'esistenza di un endpoint SMTP pubblicamente raggiungibile, tuttavia, non è una falla di sicurezza.

## «Direct Send» non equivale a «consegna diretta»

Nella discussione vengono confuse due cose:

- **Direct Send** indica in Microsoft messaggi anonimi il cui mittente dell'envelope (`5321.MailFrom`) utilizza un proprio Accepted Domain del tenant.
- **Consegna diretta a Exchange Online** indica in generale un messaggio SMTP che ignora l'MX di terze parti pubblicato e viene consegnato direttamente all'endpoint Exchange. Il mittente può anche utilizzare un dominio esterno qualsiasi.

L'opzione

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

è utile se Direct Send non è necessario. Impedisce lo spoofing del dominio interno attraverso questo percorso. Tuttavia, non chiude l'intero ingresso laterale per mittenti esterni arbitrari. Microsoft descrive l'esatto ambito di applicazione nella [documentazione del cmdlet `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Chi vuole impedire completamente «Ghost Sender» ha comunque bisogno della limitazione di accesso tramite connettore partner o di una regola di flusso di posta adeguata.

## Microsoft deve davvero fare tutto al posto dell'amministratore?

No. Chi integra un filtro e-mail aggiuntivo in una catena di trasporto produttiva si assume la responsabilità di tale catena di trasporto.

Il fornitore non può indovinare in modo affidabile se, oltre all'MX esterno, scanner, dispositivi multifunzione, servizi SaaS, server ibridi, relay partner o altri sistemi legittimi debbano inviare direttamente a Exchange Online. Un blocco automatico del tipo «l'MX punta altrove, quindi blocca tutto il resto» interromperebbe flussi di posta desiderati in numerosi ambienti reali. Per questo l'amministratore deve definire esplicitamente il confine di fiducia desiderato.

Ciononostante, Microsoft può semplificare il compito ai responsabili. Un buon Configuration Analyzer dovrebbe rilevare un MX esterno senza un connettore partner restrittivo e avvisare chiaramente. La procedura di configurazione potrebbe spiegare che un connettore di tipo «La tua organizzazione» identifica le connessioni appropriate, ma non rifiuta automaticamente quelle non appropriate. Sarebbero inoltre benvenuti interruttori secure-by-default e report operativi migliori.

Questo sarebbe un utile rafforzamento del prodotto. Ma non cambia l'inquadramento tecnico: una topologia speciale insicura rimane una configurazione insicura e non diventa uno zero-day solo per la sua ampia diffusione.

## Come chiudere l'ingresso laterale

Per gli ambienti con filtro a monte, nella checklist dovrebbero figurare almeno questi punti:

1. **Documentare completamente il flusso di posta.** Quali sistemi possono effettivamente consegnare a Exchange Online? Vi rientrano anche percorsi ibridi, applicativi e di emergenza.
2. **Configurare un connettore partner restrittivo.** Utilizzare `SenderDomains *` e limitare la consegna a un certificato (preferibile) o a intervalli di IP di origine mantenuti. Un connettore di tipo `OnPremises` o «La tua organizzazione» non impone questo effetto di default deny (vedi ad esempio: [Routing della posta tra Apache James e Exchange Online](/blog/totemomail-m365)).
3. **Configurare correttamente Enhanced Filtering.** Se EOP deve continuare a filtrare, IP originale e informazioni del mittente devono essere ricostruiti correttamente. I bypass SCL `-1` indiscriminati vanno valutati criticamente.
4. **Disabilitare Direct Send se non utilizzato.** Prima verificare con Message Trace o con i report disponibili se scanner o applicazioni dipendono da esso.
5. **Non effettuare modifiche alla cieca.** Testare e poi monitorare intervalli IP del gateway, cambi di certificato, flusso di posta ibrido nonché percorsi speciali `onmicrosoft.com`, Teams e altri.

Un esempio semplificato per la variante basata su IP è:

```powershell
New-InboundConnector `
  -Name "Solo dal gateway di posta a monte" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <Intervalli-IP-del-gateway> `
  -RequireTls $true
```

Dove possibile, il vincolo tramite certificato è preferibile a una allowlist IP. Le modifiche vanno prima effettuate in un test controllato, poiché una allowlist errata trasforma molto rapidamente l'ingresso laterale aperto in un'interruzione completa della posta.

## Il semplice test autonomo

Il test mostrato da InfoGuard (e da MSXFAQ) è utile:

```powershell
Send-MailMessage `
  -SmtpServer <nomedeltenant>.mail.protection.outlook.com `
  -To admin@<dominiodeltenant> `
  -From noreply@example.com `
  -Subject "Ingresso laterale EXO" `
  -Body "E-mail di test direttamente al tenant"
```

Con un connettore partner correttamente limitato è previsto un rifiuto SMTP come `5.7.51 TenantInboundAttribution; Rejecting`. Una regola di trasporto alternativa può prima accettare il messaggio e poi spostarlo in quarantena; pertanto, oltre alla risposta SMTP, devono essere controllati anche Message Trace, quarantena e casella di posta. `Send-MailMessage` (deprecato) serve qui solo come illustrazione facilmente comprensibile. Qualsiasi strumento di test SMTP controllato soddisfa lo stesso scopo.

## Un test utile con un'etichetta fuorviante

«Ghost Sender» non è un nuovo exploit SMTP. È un nome efficace per un ingresso laterale aperto, la cui protezione Microsoft documenta da tempo e che l'amministratore ha lasciato aperto.

L'ironia è che InfoGuard definisce il problema nel proprio articolo come «widespread and systematic misconfiguration» e conclude con la frase «Ghost-Sender is a misconfiguration». Anche il Security Response Center di Microsoft inizialmente non ha classificato la segnalazione come vulnerabilità di sicurezza. I fatti sono quindi presenti nell'articolo: purtroppo, solo il titolo, l'e-mail di test e il branding «Vulnerability» raccontano una storia più drammatica.

La parte utile della pubblicazione è il campanello d'allarme: molte aziende apparentemente non hanno protetto correttamente il proprio flusso di posta. La parte problematica è l'affermazione che Exchange Online presenti una falla di sicurezza universale. No: Exchange Online si comporta qui innanzitutto come un MTA. Diventa insicuro a causa di un confine di fiducia non configurato fino in fondo.

Bisogna davvero fare tutto al posto dell'amministratore? No. Ma evidentemente bisogna ricordare più volte che il routing DNS non sostituisce il controllo degli accessi.

## Fonti

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): L'indagine originale, con l'analisi della diffusione e la conclusione formulata dagli stessi autori: «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Il test online pubblicato da InfoGuard per verificare se il proprio tenant presenta un ingresso laterale aperto.

3.  [MSXFAQ: Exchange Online come ingresso laterale per la ricezione della posta](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): L'inquadramento di Frank Carius: non un errore in Exchange Online, bensì una configurazione errata dell'amministratore.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft spiega che l'accettazione diretta di posta per le caselle ospitate fa parte del funzionamento dell'e-mail e distingue Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): La guida ufficiale con il passaggio specifico per il connettore partner restrittivo in caso di MX esterno.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Ricostruisce la fonte del mittente originale dietro un gateway; migliora la valutazione, ma non sostituisce il connettore.

7.  [Heise: Ghost-Sender – Exchange Online lascia passare e-mail contraffatte senza problemi](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Esempio di una copertura giornalistica enfatica che generalizza solo determinate configurazioni errate.

8.  [Crow in the Cloud: I fantasmi che non ho evocato](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Un appropriato inquadramento come problema di progettazione e configurazione, comprese le misure di protezione.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Descrive il record MX come meccanismo per individuare il sistema di destinazione regolare, non come controllo degli accessi.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Stabilisce che il destinatario può considerare il trattamento DMARC pubblicato, ma non è obbligato a farlo.

---

## Il vostro flusso di posta è sicuro?

Non siete sicuri che anche il vostro tenant Exchange Online abbia un ingresso laterale aperto? **adeptio** verifica l'intero flusso di posta: dai record MX, connettori e gateway di terze parti fino a EOP, SPF, DKIM, DMARC e Direct Send. In modo pratico, indipendente e con raccomandazioni concrete.

Chi desidera far verificare o proteggere correttamente il proprio flusso di posta può fissare senza impegno un colloquio di consulenza:

**[Prenotare un colloquio di consulenza con adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
