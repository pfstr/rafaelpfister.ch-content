---
title: "Ghost Sender in Exchange Online: un record MX non è un firewall"
navTitle: "Ghost Sender"
description: "La consegna diretta a Exchange Online aggira un gateway a monte se il tenant non la blocca esplicitamente. Il rischio è reale, ma la causa è una configurazione incompleta del flusso di posta."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min di lettura"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-in-exchange-online-un-record-mx-non-e-un-firewall"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: da918db49e8e0b5243e6a3c354324682ddb6bda54ec743be474f3011df3f9dc6
translatedAt: 2026-10-14T08:54:51.878Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/it/blog/ghost-sender-in-exchange-online-un-record-mx-non-e-un-firewall
---

# Ghost Sender in Exchange Online: un record MX non è un firewall

![Un amministratore fantasma tiene aperta nel data center la porta accanto al security gate, mentre le e-mail passano direttamente nella casella di posta aggirando il filtro.](../images/ghost-admin.png)

La possibilità di attacco descritta da InfoGuard Labs come «Ghost Sender» è reale: un aggressore può aggirare un gateway e-mail a monte e consegnare direttamente a Exchange Online. Il presupposto, tuttavia, è che il tenant continui ad accettare questo percorso diretto. Non si tratta di una vulnerabilità universale di Exchange Online, bensì di una topologia del flusso di posta protetta in modo incompleto.

Un Mail Transfer Agent che gestisce caselle di posta per un dominio accetta fondamentalmente connessioni SMTP da Internet. Il record MX indica ai mittenti legittimi il percorso di consegna desiderato. Non è né una regola firewall né una lista di controllo degli accessi e non impedisce a nessuno di contattare direttamente un endpoint noto di Exchange Online.

## Cosa mostra davvero «Ghost Sender»

Lo scenario [descritto da InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) è il seguente:

1. Un'organizzazione ospita le proprie caselle di posta in Exchange Online.
2. Il record MX pubblico punta a un Secure Email Gateway a monte.
3. L'endpoint di Exchange Online in `*.mail.protection.outlook.com` rimane direttamente raggiungibile da Internet.
4. L'amministratore non ha limitato Exchange Online affinché soltanto il gateway a monte possa consegnarvi messaggi.
5. Un aggressore ignora il record MX e recapita il proprio messaggio direttamente a Exchange Online.

Il percorso previsto è quindi:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Tuttavia, rimane aperto questo percorso:

```text
Angreifer -> Exchange Online -> Postfach
```

Si tratta di una configurazione errata da prendere seriamente. Il filtro a monte può essere aggirato lungo questo percorso; mittenti contraffatti, phishing e CEO fraud risultano così notevolmente più semplici. Va riconosciuto a InfoGuard il merito di aver reso visibile il problema, averne analizzato la diffusione e pubblicato un test semplice da usare.

Ma dove sarebbe esattamente il difetto del prodotto?

Anche l'enfasi mediatica aiuta poco a inquadrare la questione. [Heise titola che Exchange Online lascia passare e-mail contraffatte «senza problemi»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), sebbene siano interessate solo determinate configurazioni di terze parti e ibride non completamente rafforzate. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) lo formula in modo molto più preciso: non una falla di sicurezza in senso stretto, ma un problema di progettazione e configurazione.

## «An MTA is doing MTA-Things»

Ogni tenant di Exchange Online dispone di un endpoint SMTP pubblico. Questo endpoint non è un segreto, né deve esserlo. Microsoft stessa spiega che Exchange Online accetta per impostazione predefinita i messaggi indirizzati direttamente alle caselle di posta ospitate lì: [è semplicemente il funzionamento dell'e-mail](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Anche [SMTP stesso descrive il record MX come il meccanismo per individuare il sistema di destinazione regolare](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Da ciò non deriva alcun obbligo per il server di destinazione di rifiutare connessioni attraverso qualsiasi altro host raggiungibile. Un aggressore non deve seguire il percorso indicato. Se un altro MTA è raggiungibile, conosce il dominio del destinatario e accetta il messaggio, verrà provato, proprio come gli spammer tentano da decenni di contattare sistemi MX di backup meno protetti.

Chi pone un filtro di terze parti a monte modifica la topologia standard. Da «Exchange Online è il mio gateway di posta Internet» si passa a «solo il gateway del mio fornitore terzo può inoltrare posta Internet a Exchange Online». Questo nuovo `Trust-Border` non nasce da una voce DNS. Deve essere imposto esplicitamente sul sistema ricevente.

Microsoft documenta esattamente questo: con un MX esterno va creato un connettore in ingresso di tipo `Partner` che, per `SenderDomains *`, accetti solo il certificato o gli indirizzi IP sorgente del servizio a monte. I messaggi consegnati direttamente aggirando il gateway vengono così rifiutati. È scritto nero su bianco nella guida Microsoft [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Anche Frank Carius descrive in dettaglio questo «ingresso secondario» nella [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM e DMARC non sono buttafuori

InfoGuard mostra messaggi per i quali SPF, DKIM e DMARC falliscono e che arrivano comunque nella casella di posta. Sembra spettacolare, ma non è un «bypass» crittografico di questi meccanismi. Le e-mail non superano affatto i controlli con successo. Forniscono `fail`. È decisiva l'azione locale che il sistema ricevente deduce da questo risultato.

SPF verifica se un sistema è autorizzato a inviare per il mittente dell'envelope. DKIM verifica una firma. DMARC collega questi risultati al dominio del mittente visibile e pubblica il trattamento desiderato. Persino l'attuale [standard DMARC RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) afferma esplicitamente che il destinatario può considerare questo trattamento desiderato, ma non è obbligato a farlo. DMARC è un segnale importante, ma non un controllo di accesso alla rete.

Con un gateway a monte si aggiunge il fatto che Exchange Online vede inizialmente l'indirizzo IP del gateway e non quello del mittente originale. A questo scopo esiste [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): ricostruisce l'origine iniziale e migliora le valutazioni di SPF, DKIM, DMARC, anti-spoofing e anti-phishing. Anche Enhanced Filtering, però, non è una serratura. Non sostituisce il connettore partner restrittivo.

La configurazione errata diventa particolarmente evidente quando un amministratore attenua il controllo EOP tramite bypass SCL o lo rimuove completamente, poiché dovrebbe già filtrare il prodotto a monte, lasciando al contempo aperta la consegna diretta da Internet. In tal caso non ha subito l'«aggiramento» di un meccanismo di protezione, ma ha deliberatamente previsto che uno dei due ingressi non disponga più di una protezione efficace.

Si può certamente criticare Microsoft quando un messaggio con un errore di autenticazione chiaramente visibile arriva nella posta in arrivo senza avvertimento. Si possono criticare la semantica dei tipi di connettore, la documentazione e l'assenza di avvisi nel Configuration Analyzer. Sono tutti punti legittimi. L'esistenza di un endpoint SMTP pubblicamente raggiungibile, tuttavia, non è una falla di sicurezza.

## «Direct Send» non equivale a «consegna diretta»

Nella discussione vengono confuse due cose:

- **Direct Send** indica in Microsoft messaggi anonimi il cui mittente dell'envelope (`5321.MailFrom`) utilizza un Accepted Domain proprio del tenant.
- **Consegna diretta a Exchange Online** indica in generale un messaggio SMTP che ignora l'MX di terze parti pubblicato e viene recapitato direttamente all'endpoint Exchange. Il mittente può anche utilizzare un dominio esterno qualsiasi.

Per Direct Send esiste un'apposita impostazione:

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-RejectDirectSend $true` | Rifiuta le consegne dirette anonime il cui mittente dell'envelope utilizza un Accepted Domain del tenant |

</details>

L'impostazione è utile se Direct Send non è necessario. Impedisce lo spoofing del dominio interno attraverso questo percorso. Tuttavia, non chiude l'intero ingresso secondario per mittenti esterni arbitrari. Microsoft descrive l'esatto ambito di applicazione nella [documentazione del cmdlet per `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Chi vuole impedire completamente «Ghost Sender» continua ad aver bisogno della restrizione degli accessi tramite connettore partner o di una regola di flusso di posta adeguata.

## Microsoft deve davvero fare tutto al posto dell'amministratore?

No. Chi aggiunge un filtro di posta supplementare a una catena di trasporto produttiva si assume la responsabilità di quella catena di trasporto.

Il fornitore non può stabilire in modo affidabile se, oltre all'MX esterno, scanner, dispositivi multifunzione, servizi SaaS, server ibridi, relay partner o altri sistemi legittimi debbano inviare direttamente a Exchange Online. Un blocco automatico del tipo «l'MX punta altrove, quindi blocca tutto il resto» interromperebbe flussi di posta desiderati in numerosi ambienti reali. Per questo l'amministratore deve definire esplicitamente il confine di fiducia desiderato.

Ciononostante, Microsoft dovrebbe rendere il compito più semplice ai responsabili. Un buon Configuration Analyzer dovrebbe rilevare un MX esterno senza connettore partner restrittivo e visualizzare un avviso chiaro. La procedura guidata di configurazione potrebbe spiegare che un connettore del tipo «La tua organizzazione» identifica sì le connessioni appropriate, ma non rifiuta automaticamente quelle inappropriate. Sarebbero inoltre benvenute impostazioni secure-by-default e migliori report operativi.

Questo sarebbe un utile rafforzamento del prodotto. Ma non cambia la valutazione tecnica: una topologia speciale insicura resta una configurazione insicura e non diventa uno zero-day solo perché è diffusa.

## Come chiudere l'ingresso secondario

Per gli ambienti con filtro a monte, almeno questi punti devono figurare nella checklist:

1. **Documentare completamente il flusso di posta.** Quali sistemi sono effettivamente autorizzati a consegnare a Exchange Online? Ne fanno parte anche percorsi ibridi, applicativi e di emergenza.
2. **Configurare un connettore partner restrittivo.** Utilizzare `SenderDomains *` e limitare la consegna a un certificato (preferibile) o a intervalli di IP sorgente gestiti. Un connettore di tipo `OnPremises` ovvero «La tua organizzazione» non impone questo effetto di default deny (si veda ad esempio anche: [Routing della posta tra Apache James e Exchange Online](/blog/totemomail-m365)).
3. **Configurare correttamente Enhanced Filtering.** Se EOP deve continuare a filtrare, IP originale e informazioni del mittente devono essere ricostruiti correttamente. I bypass SCL-`-1` generalizzati vanno esaminati criticamente.
4. **Disattivare Direct Send se inutilizzato.** Prima verificare con Message Trace o con i report disponibili se scanner o applicazioni ne dipendono.
5. **Non effettuare modifiche alla cieca.** Testare e quindi monitorare gli intervalli IP del gateway, i cambi di certificato, il flusso di posta ibrido nonché i percorsi speciali di `onmicrosoft.com`, Teams e altri.

Un esempio semplificato della variante basata su IP è il seguente:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Name` | Nome visualizzato del nuovo connettore in ingresso |
| `-ConnectorType Partner` | Classe di connettore per sistemi partner esterni; solo questo tipo impone il rifiuto delle connessioni inappropriate |
| `-SenderDomains *` | Il connettore si applica alla posta proveniente da tutti i domini mittente |
| `-RestrictDomainsToIPAddresses $true` | Attiva il blocco: la posta dei domini indicati viene accettata solo dagli indirizzi in `-SenderIpAddresses` |
| `-SenderIpAddresses` | Gli indirizzi o intervalli IP sorgente consentiti del gateway a monte |
| `-RequireTls $true` | Richiede la crittografia TLS per le connessioni tramite questo connettore |

</details>

Dove possibile, il vincolo al certificato è preferibile a un elenco di IP consentiti. Le modifiche vanno prima sottoposte a un test controllato, poiché una allowlist errata trasforma molto rapidamente l'ingresso secondario aperto in un'interruzione completa della posta.

## Il semplice autotest

Il test mostrato da InfoGuard (e dalla MSXFAQ) è utile:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-SmtpServer` | Host di destinazione: l'endpoint pubblico di Exchange Online del tenant, intenzionalmente oltre l'MX |
| `-To` | Indirizzo del destinatario nel tenant da testare |
| `-From` | Indirizzo mittente esterno qualsiasi; è proprio questo che l'ingresso secondario non dovrebbe più accettare |
| `-Subject` | Oggetto, per ritrovarlo nel Message Trace |
| `-Body` | Testo del messaggio |

</details>

Con un connettore partner correttamente limitato ci si deve aspettare un rifiuto SMTP come `5.7.51 TenantInboundAttribution; Rejecting`. Una regola di trasporto alternativa può prima accettare il messaggio e poi spostarlo in quarantena; perciò, oltre alla risposta SMTP, devono essere controllati anche Message Trace, quarantena e casella di posta. `Send-MailMessage` (deprecato) serve qui soltanto come illustrazione facilmente comprensibile. Qualsiasi strumento di test SMTP controllato svolge lo stesso scopo.

## Un test utile con un'etichetta fuorviante

«Ghost Sender» non è un nuovo exploit SMTP. È un nome efficace per un ingresso secondario aperto, la cui protezione Microsoft documenta da tempo e che l'amministratore ha lasciato aperto.

L'ironia è che InfoGuard definisce il problema nel proprio contributo come «widespread and systematic misconfiguration» e conclude con la frase «Ghost-Sender is a misconfiguration». Anche il Microsoft Security Response Center inizialmente non ha classificato la segnalazione come una vulnerabilità di sicurezza. I fatti sono dunque presenti nell'articolo: solo il titolo, l'e-mail di test e il branding «Vulnerability» suggeriscono purtroppo un'interpretazione più drammatica.

La parte utile della pubblicazione è il campanello d'allarme: molte aziende apparentemente non hanno protetto correttamente il proprio flusso di posta. La parte problematica è l'affermazione secondo cui Exchange Online avrebbe una falla di sicurezza universale. No: Exchange Online qui si comporta anzitutto come un MTA. Diventa insicuro a causa di un confine di fiducia non configurato fino in fondo.

Bisogna davvero fare tutto al posto dell'amministratore? No. Ma evidentemente bisogna ricordare continuamente che il routing DNS non sostituisce il controllo degli accessi.

## Fonti

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): L'indagine originale, con analisi della diffusione e la conclusione degli stessi autori: «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): Il test online pubblicato da InfoGuard per verificare se il proprio tenant presenta un ingresso secondario aperto.

3.  [MSXFAQ: Exchange Online come ingresso secondario per la ricezione della posta](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): La valutazione di Frank Carius: non un errore in Exchange Online, ma una configurazione errata dell'amministratore.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft spiega che l'accettazione diretta della posta per le caselle ospitate è il funzionamento dell'e-mail e distingue Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): La guida ufficiale con il proprio passaggio dedicato al connettore partner restrittivo in caso di MX esterno.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Ricostruisce la sorgente originale del mittente dietro un gateway; migliora la valutazione, ma non sostituisce il connettore.

7.  [Heise: Ghost-Sender – Exchange Online lascia passare e-mail contraffatte senza problemi](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Esempio di cronaca enfatizzata che generalizza solo determinate configurazioni errate.

8.  [Crow in the Cloud: I fantasmi che non ho evocato](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Corretta valutazione come problema di progettazione e configurazione, con misure di protezione.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Descrive il record MX come meccanismo per individuare il sistema di destinazione regolare, non come controllo degli accessi.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Stabilisce che il destinatario può considerare il trattamento DMARC pubblicato, ma non è obbligato a farlo.

---

## Il vostro flusso di posta è sicuro?

Non siete sicuri che anche il vostro tenant Exchange Online abbia un ingresso secondario aperto? **adeptio** verifica l'intero flusso di posta: dai record MX, connettori e gateway di terze parti fino a EOP, SPF, DKIM, DMARC e Direct Send. In modo pratico, indipendente e con raccomandazioni concrete.

Chi desidera far verificare o proteggere correttamente il proprio flusso di posta può fissare senza impegno un colloquio di consulenza:

**[Prenota un colloquio di consulenza con adeptio](/kontakt)**  
[adeptio.ch](https://adeptio.ch/)
