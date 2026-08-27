---
title: "F5 BIG-IP come proxy in uscita per l'invio massivo di e-mail: persistenza, SNAT, timeout e risoluzione DNS"
navTitle: "F5 invio massivo"
description: "Un invio massivo di 1000 e-mail al minuto passa attraverso una BIG-IP come proxy in uscita verso il relay del provider. L'articolo chiarisce perché le sessioni sticky non servono, come risolvere correttamente il nome host del provider tramite un nodo FQDN e quali impostazioni di SNAT, timeout e limiti di connessione determinano realmente il throughput."
date: "2026-08-26"
kategorie: "Load balancer"
timeToRead: "9 min di lettura"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-come-proxy-in-uscita-per-l-invio-massivo-di-e-mail-persistenza-snat-timeout-e"
featured: true
translationId: "article-ee5e63e82ffd2604"
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/it/blog/f5-big-ip-come-proxy-in-uscita-per-l-invio-massivo-di-e-mail-persistenza-snat-timeout-e
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:29:01.588Z
translationReview: required
---

Un ciclo di fatturazione o l'invio di una newsletter con circa 1000 e-mail al minuto esce dalla rete aziendale passando attraverso una F5 BIG-IP come proxy in uscita verso il punto di consegna del provider. La BIG-IP non distribuisce il traffico su più destinazioni, ma lo inoltra. Proprio questa configurazione determina quali impostazioni siano sensate e quali presunte ottimizzazioni non producano alcun effetto.

## L'architettura in una frase

I sistemi di invio usano come smarthost un indirizzo interno di Virtual Server sulla BIG-IP; la BIG-IP traduce gli indirizzi sorgente tramite SNAT in un IP pubblico fisso e inoltra ogni connessione al nome host del provider. Sulla BIG-IP non avviene un vero load balancing, poiché il pool ha un solo membro. Può sembrare una configurazione banale, ma le decisioni di dettaglio (persistenza, timeout, tipo di SNAT, risoluzione DNS) determinano se l'invio funziona stabilmente o se presenta interruzioni inspiegabili sotto carico.

## Le sessioni sticky sono migliori? No, per due motivi

La questione della persistenza di sessione deriva dal mondo HTTP, dove un utente con carrello o sessione di login deve sempre raggiungere lo stesso backend. Applicato a SMTP, il concetto non ha senso.

In primo luogo, SMTP si conclude senza stato per connessione: ogni connessione esegue una o più transazioni complete (MAIL FROM, RCPT TO, DATA) e termina con QUIT. Non esiste alcuno stato che debba risiedere sullo stesso sistema di destinazione tra connessioni diverse. Quale sistema lato provider accetti la connessione successiva è irrilevante per la consegna.

In secondo luogo, su questa BIG-IP non c'è letteralmente nulla da rendere persistente: il pool contiene esattamente un membro, l'unico indirizzo IP del provider. Un profilo di persistenza consumerebbe solo memoria per una tabella di persistenza e richiederebbe una ricerca a ogni connessione, fornendo sempre lo stesso risultato. L'impostazione corretta è quindi: Default Persistence Profile su None. Anche se in futuro il provider dovesse pubblicare più indirizzi IP dietro il nome host, la persistenza sarebbe controproducente, perché impedirebbe la distribuzione su tali indirizzi e caricherebbe unilateralmente singole destinazioni.

Per il throughput dell'invio massivo è decisivo il profilo di connessione del mittente: poche connessioni di lunga durata con molti messaggi per connessione invece di una nuova connessione per ogni e-mail; maggiori dettagli più avanti.

## Virtual Server: FastL4 invece di Full Proxy

Per il semplice inoltro di SMTP, la scelta giusta è un Virtual Server Performance (Layer 4) con profilo FastL4. La BIG-IP elabora così la connessione in larga parte nell'hardware o nel percorso accelerato, senza terminare completamente la connessione TCP. Un Virtual Server standard in modalità Full Proxy offre un vantaggio solo se si intende effettivamente intervenire nel flusso di dati sulla BIG-IP, ad esempio con un profilo di sicurezza SMTP o con iRules a livello di protocollo. Per un proxy in uscita verso il proprio provider contrattuale è superfluo e crea soltanto ulteriori fonti di errore.

Importante in entrambi i casi: non attivare alcun profilo che scriva nella connessione. I sistemi di invio negoziano STARTTLS direttamente con il relay del provider; qualsiasi istanza che modifichi o filtri byte mette a rischio l'instaurazione di TLS.

## Risoluzione DNS: il nome host del provider deve essere un nodo FQDN nel pool

Il provider ha fornito un nome host, non un indirizzo IP. Il riflesso più immediato, ovvero risolvere l'IP una volta e registrarlo staticamente come nodo, è la variante peggiore: se il provider cambia indirizzo (manutenzione, migrazione, caso DR), l'invio si blocca finché qualcuno non adatta la configurazione BIG-IP. Per questo esistono i nodi FQDN.

Un nodo FQDN memorizza il nome host invece dell'indirizzo. La BIG-IP risolve autonomamente il nome, crea un cosiddetto nodo ephemeral per ogni indirizzo restituito e li aggiorna automaticamente quando cambia la risposta DNS. Per impostazione predefinita, interroga nuovamente il nome alla scadenza del TTL DNS; in alternativa è possibile impostare un intervallo di interrogazione fisso. Con Autopopulate attivato, il pool acquisisce automaticamente come membri anche più record A: se in futuro il provider estendesse la consegna a più indirizzi, la BIG-IP lo seguirebbe senza modifiche alla configurazione.

Due requisiti vengono spesso dimenticati. Innanzitutto, la BIG-IP necessita di server DNS funzionanti nella configurazione di sistema (System, Configuration, Device, DNS); i nodi FQDN usano i resolver di sistema, non una cache DNS di un profilo listener. In secondo luogo, questi resolver devono essere effettivamente raggiungibili dal contesto di gestione o TMM, altrimenti il nodo rimane nello stato unresolved e il pool resta vuoto.

La configurazione in tmsh appare così (indirizzi e nomi sono esempi):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

I sistemi di invio configurano quindi 10.0.5.10 come smarthost. Se si utilizza la porta 25 o 587 dipende dal provider; la configurazione BIG-IP è identica in entrambi i casi, cambia solo la porta.

## SNAT: indirizzo fisso invece di Automap

Per il traffico e-mail in uscita, l'indirizzo sorgente deve essere sotto controllo. SNAT Automap utilizza il Floating Self-IP della VLAN in uscita, che può cambiare inosservato in caso di modifiche di rete o ristrutturazioni del failover. Tuttavia, i provider spesso associano la consegna a un allowlist IP e, anche senza un allowlist formale, la reputazione è legata all'indirizzo sorgente. Un pool SNAT dedicato con un indirizzo assegnato in modo fisso rende l'IP sorgente un oggetto di configurazione documentato e stabile.

Per quanto riguarda la capacità: un singolo indirizzo SNAT offre circa 64'000 traduzioni simultanee verso una singola destinazione (un IP, una porta), poiché ogni connessione riceve una propria porta sorgente effimera. Con il profilo di carico qui descritto, costituito da poche decine di connessioni simultanee, è sufficiente di molti ordini di grandezza. L'esaurimento delle porte diventa un problema solo quando un mittente configurato erroneamente apre una nuova connessione per ogni e-mail e non la chiude correttamente; in tal caso le traduzioni si accumulano in uno stato simile a TIME-WAIT. Un comportamento simile va corretto sul mittente, non con un secondo indirizzo SNAT.

## Timeout: la causa più frequente delle interruzioni di connessione sotto carico

Un mittente bulk mantiene aperte le connessioni e invia un messaggio dopo l'altro. Tra due messaggi possono verificarsi pause: il mittente genera il blocco successivo, il relay ritarda l'accettazione (tarpitting, residui di greylisting, code interne). Il timeout di inattività del profilo FastL4 è impostato per default a 300 secondi. Se una pausa supera tale valore, la BIG-IP chiude la connessione e il mittente scrive su una connessione che non esiste più.

Due impostazioni attenuano il problema. Innanzitutto, impostare il timeout di inattività su un valore superiore alle pause realistiche; per l'invio massivo, 600 secondi sono un valore iniziale ragionevole. Il valore non dovrebbe essere arbitrariamente elevato, altrimenti le connessioni orfane si accumulano nella tabella delle connessioni. In secondo luogo, mantenere attivo Reset on Timeout nel profilo: la BIG-IP segnala la chiusura con un reset TCP e l'MTA mittente riconosce subito che la connessione è interrotta, invece di andare in timeout e ripianificare il messaggio solo dopo minuti.

Non si ha alcuna influenza sui timeout della controparte, ma devono essere considerati: se il relay del provider chiude le connessioni dopo 120 secondi di inattività, un timeout BIG-IP generoso non serve a nulla. È determinante il valore di timeout più basso lungo l'intero percorso; in caso di dubbio, chiedere al provider e usare tale valore come base di pianificazione.

## Strategia di connessione: poche connessioni, molti messaggi

In assenza di specifiche di consegna del provider, vale la pena fare un breve calcolo. 1000 e-mail al minuto corrispondono a circa 17 al secondo. Una transazione SMTP su una connessione già stabilita richiede, con latenza normale, nettamente meno di mezzo secondo. Con 10-20 connessioni parallele e, ad esempio, 100 messaggi per connessione prima che il mittente le rinnovi, la velocità desiderata viene raggiunta comodamente. Lato provider è generalmente disponibile una capacità di connessione molto maggiore, ma è condivisa con tutti gli altri clienti. Poche connessioni di lunga durata con molte transazioni non sono quindi solo efficienti (si evita l'instaurazione TCP e TLS per ogni messaggio), ma anche il modo più compatibile di usare infrastrutture altrui.

Le impostazioni per questo si trovano nel sistema di invio, non sulla BIG-IP: massimo numero di messaggi per connessione, massimo numero di connessioni parallele allo smarthost, riutilizzo delle connessioni esistenti. Sulla BIG-IP è possibile proteggere il tutto con un Connection Limit sul pool member, ad esempio 200 connessioni simultanee: nel normale esercizio il valore non viene mai raggiunto, ma un mittente configurato erroneamente che improvvisamente apra una connessione per ogni e-mail non inonda senza limiti il relay del provider. Il limite è una rete di sicurezza, non uno strumento di controllo.

La misurazione mostra se il profilo di connessione configurato viene effettivamente applicato: connessioni al minuto e messaggi per connessione possono essere valutati tramite Message Tracking o Connector Logs, come descritto nell'articolo [Determinare il profilo di carico di un mail server](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Per un test di carico con un profilo bulk realistico (poche sessioni, molti messaggi per sessione), smtp-source del pacchetto Postfix è più adatto degli strumenti di carico orientati a HTTP, perché genera esattamente questo profilo di connessione.

## Monitoraggio: non sovraccaricare il provider con health check

Un monitor sul pool member è utile affinché la BIG-IP riconosca un guasto lato provider e lo segnali correttamente. Vale però quanto segue: ogni health check è una connessione reale verso il provider e conta contro gli stessi limiti del traffico utente. È pienamente sufficiente un semplice monitor TCP con intervallo moderato (30 secondi o più). Un monitor SMTP completo, che verifichi fino al banner o a EHLO, fornisce pochi elementi aggiuntivi, ma genera voci di log lato provider e, nel caso peggiore, richieste di chiarimento sul perché ogni 5 secondi arrivi una connessione senza e-mail.

## Checklist

| Impostazione | Raccomandazione |
|---|---|
| Profilo di persistenza | None; le sessioni sticky non portano nulla con SMTP e ancor meno con un pool a un solo membro |
| Tipo di Virtual Server | Performance (Layer 4) con profilo FastL4, nessun intervento nel flusso di dati |
| Nodo di destinazione | Nodo FQDN con Autopopulate invece di IP statico; server DNS configurati sulla BIG-IP |
| SNAT | pool SNAT dedicato con indirizzo fisso noto al provider; nessun Automap |
| Timeout di inattività | superiore alle pause di invio reali, valore iniziale 600 s; Reset on Timeout attivo |
| Connection Limit | come rete di sicurezza sul pool member, ad es. 200 |
| Monitor | TCP, intervallo di 30 s o più; nessun monitor SMTP aggressivo |
| Configurazione del mittente | poche connessioni parallele, molti messaggi per connessione; riutilizzo attivo |

La risposta breve alla domanda iniziale è quindi: no, le sessioni sticky non sono migliori; in questa configurazione sono inefficaci o dannose. La qualità della soluzione dipende dalla risoluzione DNS del nome host del provider, da un indirizzo SNAT stabile, da timeout adatti al profilo di carico e dal fatto che i sistemi di invio consegnino le loro 1000 e-mail al minuto attraverso poche connessioni persistenti anziché mille singole connessioni.

## Fonti

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): la sezione 4.5.4 e il modello transazionale mostrano che più transazioni e-mail su una connessione sono il normale caso previsto.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): articolo introduttivo F5 su SNAT, pool SNAT e traduzione delle porte per destinazione.

3.  [Riferimento tmsh: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documenta le opzioni FQDN (name, autopopulate, interval) per i nodi e quindi per i pool member.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): generatore di carico che riproduce il profilo di connessione di un mittente bulk (poche sessioni, molti messaggi).

5.  [Determinare il profilo di carico di un mail server](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): guida interna su come valutare connessioni al minuto e messaggi per connessione dal Message Tracking.
