---
title: "F5 BIG-IP come proxy in uscita per l’invio massivo di e-mail: persistenza, SNAT, timeout e risoluzione DNS"
navTitle: "F5 invio massivo"
description: "Un invio massivo di 1000 e-mail al minuto passa attraverso una BIG-IP come proxy in uscita verso il relay del provider. L’articolo spiega perché le Sticky Sessions non servono in questo caso, come risolvere correttamente il nome host del provider tramite un nodo FQDN e quali impostazioni di SNAT, timeout e limiti di connessione determinano davvero la velocità effettiva."
date: "2026-08-26"
kategorie: "Bilanciatore di carico"
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
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:23:16.288Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/f5-big-ip-come-proxy-in-uscita-per-l-invio-massivo-di-e-mail-persistenza-snat-timeout-e
---

Un ciclo di fatturazione o l’invio di una newsletter con circa 1000 e-mail al minuto esce dalla rete aziendale, con una F5 BIG-IP che funge da proxy in uscita verso il punto di consegna del provider. La BIG-IP non distribuisce il traffico su più destinazioni, ma lo inoltra. È proprio questa configurazione a determinare quali impostazioni siano sensate e quali presunte ottimizzazioni non producano alcun effetto.

## L’architettura in una frase

I sistemi di invio usano come smarthost un indirizzo interno di Virtual Server sulla BIG-IP; la BIG-IP traduce gli indirizzi sorgente tramite SNAT in un IP pubblico fisso e inoltra ogni connessione al nome host del provider. Sulla BIG-IP non avviene alcun bilanciamento del carico in senso proprio, perché il pool ha un solo membro. Può sembrare una configurazione banale, ma le decisioni di dettaglio (persistenza, timeout, tipo di SNAT, risoluzione DNS) determinano se l’invio resta stabile o se mostra interruzioni inspiegabili sotto carico.

## Le Sticky Sessions sono migliori? No, per due motivi

La domanda sulla persistenza di sessione proviene dal mondo HTTP, dove un utente con carrello o sessione di login deve raggiungere sempre lo stesso backend. Applicato a SMTP, il concetto non ha senso.

Primo: SMTP termina senza stato per ogni connessione. Ogni connessione gestisce una o più transazioni complete (MAIL FROM, RCPT TO, DATA) e termina con QUIT. Non esiste uno stato che debba risiedere sullo stesso sistema di destinazione tra connessioni diverse. Quale sistema sul lato provider accetti la connessione successiva è irrilevante per la consegna.

Secondo: su questa BIG-IP non c’è semplicemente nulla da rendere persistente. Il pool contiene esattamente un membro, l’unico indirizzo IP del provider. Un profilo di persistenza consumerebbe soltanto memoria per una tabella di persistenza e imporrebbe una ricerca a ogni connessione, con un risultato sempre identico. L’impostazione corretta è quindi: Default Persistence Profile su None. Anche se il provider dovesse in futuro pubblicare più indirizzi IP dietro il nome host, la persistenza sarebbe controproducente, perché impedirebbe la distribuzione verso questi indirizzi e caricherebbe unilateralmente singole destinazioni.

Per la velocità effettiva nell’invio massivo, ciò che conta è il profilo di connessione del mittente: poche connessioni durature con molti messaggi per connessione anziché una nuova connessione per ogni e-mail; maggiori dettagli più avanti.

## Virtual Server: FastL4 invece di Full Proxy

Per il semplice inoltro di SMTP, la scelta corretta è un Virtual Server Performance (Layer 4) con profilo FastL4. La BIG-IP elabora quindi la connessione in gran parte nell’hardware o nel percorso accelerato, senza terminare completamente la connessione TCP. Un Virtual Server standard in modalità Full Proxy offre vantaggi solo se si intende intervenire realmente sul flusso di dati nella BIG-IP, ad esempio con un profilo di sicurezza SMTP o iRules a livello di protocollo. Per un proxy in uscita verso il proprio provider contrattuale questo è superfluo e introduce solo ulteriori fonti di errore.

Importante in entrambi i casi: non attivare alcun profilo che scriva nella connessione. I sistemi di invio negoziano STARTTLS direttamente con il relay del provider; ogni istanza che modifica o filtra byte mette a rischio l’instaurazione di TLS.

## Risoluzione DNS: il nome host del provider deve essere inserito nel pool come nodo FQDN

Il provider ha fornito un nome host, non un indirizzo IP. L’impulso naturale di risolvere una volta l’IP e inserirlo come nodo statico è la variante peggiore: se il provider cambia indirizzo (manutenzione, migrazione, caso di DR), l’invio si interrompe finché qualcuno non adatta la configurazione della BIG-IP. È esattamente per questo che esistono i nodi FQDN.

Un nodo FQDN memorizza il nome host anziché l’indirizzo. La BIG-IP risolve autonomamente il nome, crea per ogni indirizzo restituito un cosiddetto nodo effimero e lo aggiorna automaticamente quando cambia la risposta DNS. Per impostazione predefinita interroga nuovamente il nome ogni 3600 secondi (un’ora); con `interval ttl` segue invece il TTL della risposta DNS, oppure è possibile impostare un diverso intervallo di interrogazione fisso. Con Autopopulate attivato, il pool acquisisce automaticamente anche più record A come membri: se il provider dovesse in futuro estendere la consegna a più indirizzi, la BIG-IP si adeguerà senza modifiche alla configurazione.

Due prerequisiti vengono spesso dimenticati. Primo, la BIG-IP necessita di server DNS funzionanti nella configurazione di sistema (System, Configuration, Device, DNS); i nodi FQDN usano i resolver di sistema, non una cache DNS di un profilo listener. Secondo, questi resolver devono essere effettivamente raggiungibili dal contesto di gestione o TMM, altrimenti il nodo resta nello stato unresolved e il pool rimane vuoto.

La configurazione in tmsh è la seguente (indirizzi e nomi sono esempi):

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

I sistemi di invio configurano quindi 10.0.5.10 come smarthost. Se usare la porta 25 o 587 lo stabilisce il provider; la configurazione della BIG-IP è identica in entrambi i casi, cambia solo la porta.

## SNAT: indirizzo fisso invece di Automap

Per il traffico e-mail in uscita, l’indirizzo sorgente deve essere sotto controllo. SNAT Automap utilizza il Floating Self-IP della VLAN in uscita, che può cambiare inosservato in caso di modifiche di rete o riconfigurazioni del failover. Tuttavia, i provider spesso vincolano la consegna a un allowlist di IP e, anche senza un allowlist formale, la reputazione dipende dall’indirizzo sorgente. Un pool SNAT dedicato con un indirizzo assegnato in modo fisso trasforma l’IP sorgente in un oggetto di configurazione documentato e stabile.

Per quanto riguarda la capacità: un singolo indirizzo SNAT offre circa 64'000 traduzioni simultanee verso una singola destinazione (un IP, una porta), poiché ogni connessione riceve una propria porta sorgente effimera. Con il profilo di carico qui descritto, caratterizzato da poche decine di connessioni simultanee, questo è ampiamente sufficiente. L’esaurimento delle porte diventa un problema solo quando un mittente configurato in modo errato apre una nuova connessione per ogni e-mail e non la chiude correttamente; in tal caso le traduzioni si accumulano in uno stato simile a TIME-WAIT. Questo comportamento va corretto sul mittente, non con un secondo indirizzo SNAT.

## Timeout: la causa più frequente delle interruzioni di connessione sotto carico

Un mittente bulk mantiene aperte le connessioni e invia un messaggio dopo l’altro. Tra due messaggi possono verificarsi pause: il mittente genera il blocco successivo, il relay ritarda l’accettazione (tarpitting, residui di greylisting, code interne). Il timeout di inattività del profilo FastL4 è impostato per default a 300 secondi. Se una pausa supera questo valore, la BIG-IP chiude la connessione e il mittente scrive su una connessione che non esiste più.

Due impostazioni attenuano il problema. Primo, impostare il timeout di inattività a un valore superiore alle pause realistiche; per l’invio massivo, 600 secondi sono un valore iniziale ragionevole. Il valore non dovrebbe essere arbitrariamente elevato, altrimenti le connessioni orfane si accumulano nella tabella delle connessioni. Secondo, mantenere attivo Reset on Timeout nel profilo: la BIG-IP conferma allora la chiusura con un reset TCP e l’MTA mittente riconosce immediatamente che la connessione non è più disponibile, invece di arrivare a un timeout e ripianificare il messaggio solo dopo minuti.

Non è possibile influire sui timeout della controparte, ma fanno parte del quadro: se il relay del provider chiude le connessioni dopo 120 secondi di inattività, un timeout generoso della BIG-IP non serve a nulla. Conta il valore di timeout più basso dell’intero percorso; in caso di dubbio, chiedete al provider e utilizzate tale valore come base di pianificazione.

## Strategia di connessione: poche connessioni, molti messaggi

In assenza di requisiti di consegna del provider, vale la pena fare un breve calcolo. 1000 e-mail al minuto corrispondono a circa 17 al secondo. Una transazione SMTP tramite una connessione già aperta richiede, con una latenza normale, ben meno di mezzo secondo. Con 10-20 connessioni parallele e, ad esempio, 100 messaggi per connessione prima che il mittente le rinnovi, la frequenza obiettivo viene raggiunta comodamente. Sul lato provider è di norma disponibile una capacità di connessione molto maggiore, ma è condivisa con tutti gli altri clienti. Poche connessioni durature con molte transazioni non sono quindi soltanto efficienti (non occorre instaurare TCP e TLS per ogni messaggio), ma anche il modo più rispettoso di utilizzare l’infrastruttura altrui.

Le leve per questo sono nel sistema di invio, non sulla BIG-IP: numero massimo di messaggi per connessione, numero massimo di connessioni parallele verso lo smarthost, riutilizzo delle connessioni esistenti. Sulla BIG-IP è possibile proteggere il tutto con un Connection Limit sul pool member, ad esempio 200 connessioni simultanee: nel normale funzionamento questo valore non viene mai raggiunto, ma un mittente configurato erroneamente che improvvisamente apre una connessione per ogni e-mail non inonda senza limiti il relay del provider. Il limite è una rete di sicurezza, non uno strumento di controllo.

La misurazione mostra se il profilo di connessione impostato si concretizza anche nella pratica: connessioni al minuto e messaggi per connessione possono essere analizzati dal Message Tracking o dai log del connettore, come descritto nell’articolo [Determinare il profilo di carico di un server di posta](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Per un test di carico con un profilo bulk realistico (poche sessioni, molti messaggi per sessione), smtp-source del pacchetto Postfix è più adatto degli strumenti di carico orientati a HTTP, perché genera esattamente questo profilo di connessione.

## Monitoraggio: non gravare il provider con health check

Un monitor sul pool member è utile affinché la BIG-IP rilevi e segnali correttamente un guasto sul lato provider. Vale però quanto segue: ogni health check è una connessione reale al provider e conta rispetto agli stessi limiti del traffico utile. Un semplice monitor TCP con un intervallo moderato (30 secondi o più) è più che sufficiente. Un monitor SMTP completo, che controlli fino al banner o a EHLO, offre pochi vantaggi informativi aggiuntivi, ma genera voci di log sul lato provider e, nel peggiore dei casi, richieste sul perché ogni 5 secondi arrivi una connessione senza e-mail.

## Checklist

| Impostazione | Raccomandazione |
|---|---|
| Profilo di persistenza | None; le Sticky Sessions non apportano nulla con SMTP e ancor meno con un pool a un solo membro |
| Tipo di Virtual Server | Performance (Layer 4) con profilo FastL4, nessun intervento sul flusso di dati |
| Nodo di destinazione | Nodo FQDN con Autopopulate invece di IP statico; server DNS configurati sulla BIG-IP |
| SNAT | pool SNAT dedicato con indirizzo fisso noto al provider; nessun Automap |
| Timeout di inattività | superiore alle reali pause di invio, valore iniziale 600 s; Reset on Timeout attivo |
| Connection Limit | come rete di sicurezza sul pool member, ad es. 200 |
| Monitor | TCP, intervallo di 30 s o più; nessun monitor SMTP aggressivo |
| Configurazione del mittente | poche connessioni parallele, molti messaggi per connessione; riutilizzo attivo |

La risposta breve alla domanda iniziale è quindi: no, le Sticky Sessions non sono migliori; in questa configurazione sono inefficaci o addirittura dannose. La qualità della soluzione dipende dalla risoluzione DNS del nome host del provider, da un indirizzo SNAT stabile, da timeout adatti al profilo di carico e dal fatto che i sistemi di invio consegnino le loro 1000 e-mail al minuto tramite poche connessioni aperte anziché mille singole.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: l’intervallo standard per la risoluzione dei nomi dei nodi FQDN è di 3600 secondi, non il DNS TTL; la BIG-IP segue il TTL solo con l’impostazione interval ttl.

## Fonti

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): la sezione 4.5.4 e il modello transazionale mostrano che più transazioni e-mail su una connessione costituiscono il normale caso previsto.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): articolo introduttivo di F5 su SNAT, pool SNAT e traduzione delle porte per destinazione.

3.  [Riferimento tmsh: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documenta le opzioni FQDN (name, autopopulate, interval) per i nodi e quindi per i pool member, incluso l’intervallo standard di 3600 secondi e il valore `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): generatore di carico che riproduce il profilo di connessione del mittente bulk (poche sessioni, molti messaggi).

5.  [Determinare il profilo di carico di un server di posta](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): guida interna su come analizzare connessioni al minuto e messaggi per connessione dal Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html): conferma 3600 secondi come valore predefinito effettivo dell’intervallo di interrogazione FQDN; la precedente indicazione TTL nella guida era errata.
