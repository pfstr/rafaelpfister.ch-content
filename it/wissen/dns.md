---
title: "DNS: risoluzione, delega e gestione"
blatt: "dns"
description: "Il Domain Name System dal punto di vista dell’amministratore: spazio dei nomi, zone e delega, risoluzione ricorsiva, RRset, cache e TTL, risposte negative, UDP e TCP, EDNS, DNSSEC, replica delle zone e diagnostica per infrastrutture di messaggistica."
fakten:
  - label: Nome completo
    wert: Domain Name System
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Modello di base
    wert: spazio dei nomi e base dati gerarchici distribuiti
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-2
  - label: Standard principali
    wert: RFC 1034 · RFC 1035 · STD 13
    href: https://www.rfc-editor.org/info/std13
  - label: Chiave di interrogazione
    wert: QNAME · QTYPE · QCLASS
    href: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2
  - label: Unità dati
    wert: Resource Record Set (RRset)
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-5
  - label: Ruoli dei server
    wert: autoritativo · ricorsivo · Forwarder
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-6
  - label: Trasporto
    wert: UDP e TCP · Port 53
    href: https://datatracker.ietf.org/doc/html/rfc7766
  - label: Estensioni
    wert: EDNS(0) tramite OPT
    href: https://datatracker.ietf.org/doc/html/rfc6891
  - label: Coerenza
    wert: cache positive e negative controllate dal TTL
    href: https://datatracker.ietf.org/doc/html/rfc2308
  - label: Integrità
    wert: "DNSSEC: DNSKEY · DS · RRSIG · NSEC"
    href: https://datatracker.ietf.org/doc/html/rfc4034
  - label: Sincronizzazione delle zone
    wert: NOTIFY · AXFR · IXFR
    href: https://datatracker.ietf.org/doc/html/rfc1996
  - label: Riferimento alla posta
    wert: MX · PTR · TXT e nomi di destinazione derivati
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-5
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f68d5b35d0c022538eb216baafcdf1c277fffbe2c2db0ed4a3b519c01ba63062
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:36:31.306Z
translationReview: required
---

# DNS: risoluzione, delega e gestione

DNS risponde alla domanda su quali informazioni siano pubblicate per un determinato nome e chi ne sia responsabile. Il sistema è al tempo stesso uno spazio dei nomi gerarchico, una base dati distribuita e un protocollo binario di interrogazione. Non fornisce solo indirizzi IP, ma anche nameserver, destinazioni di posta, endpoint di servizio, chiavi e policy. Chi considera DNS soltanto come «risoluzione dei nomi» trascura quindi proprio quei record da cui dipendono messaggistica e identità ([RFC 9499](https://datatracker.ietf.org/doc/html/rfc9499)).

Il percorso seguente parte dallo stub resolver di un’applicazione, segue cache e deleghe fino al server autoritativo e riporta quindi la risposta indietro. Questo flusso permette di inquadrare TTL, Glue, fallback TCP, DNSSEC, gestione delle zone e i tipici scenari di errore.

Per gli amministratori di messaggistica, DNS è un sistema di controllo a monte. Un MTA determina tramite esso la successiva destinazione di posta, l’autenticazione del mittente legge policy e chiavi, le procedure per i certificati possono includere dati protetti da DNSSEC e i client di directory o Kerberos cercano servizi tramite record SRV. DNS non verifica tuttavia se il servizio trovato sia operativo. Una risposta MX sintatticamente corretta può puntare a un listener SMTP non raggiungibile; una ricerca A riuscita non dice nulla su TLS, autenticazione o stato dell’applicazione.

## Architettura e ruoli

La responsabilità dei nomi è organizzata come un albero. Alla radice senza nome `.` iniziano domini di primo livello come `ch.`, seguiti da domini delegati e ulteriori label. Un punto finale rende completo un nome e impedisce l’aggiunta di suffissi di ricerca locali. Questa piccola differenza di notazione è rilevante operativamente: `mail.example.ch` può essere completato da un client con un Search Domain, `mail.example.ch.` no.

Un **dominio** è una parte dello spazio dei nomi. Una **zona**, invece, è un insieme di dati amministrativamente coerente per il quale un server autoritativo ha responsabilità locale. Una delega separa una zona figlia dalla zona padre. A questo scopo, il padre pubblica un RRset NS e, se un nome di nameserver si trova all’interno della zona figlia delegata, i dati A o AAAA necessari per raggiungerlo come **Glue**. La distinzione fondamentale tra spazio dei nomi, zone e delega risale a [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034#section-4.2); i termini attuali sono riassunti da [RFC 9499, sezione 7](https://datatracker.ietf.org/doc/html/rfc9499#section-7).

Quattro ruoli logici spiegano il percorso di risoluzione:

| Ruolo | Conoscenza e compito | Importante confine operativo |
|---|---|---|
| Stub Resolver | riceve la richiesta dell’applicazione e la inoltra a un resolver configurato | suffissi di ricerca, file hosts locale e cache del client possono influenzare il risultato prima del DNS vero e proprio |
| Resolver ricorsivo | fornisce una risposta finale dalla cache o tramite interrogazioni iterative | è il confine di fiducia per cache, filtraggio, logging e validazione DNSSEC |
| Forwarder | gestisce richieste ricorsive provenienti da un altro resolver | sposta risoluzione e osservabilità verso un ulteriore gestore |
| Server autoritativo | risponde dalle zone caricate localmente e imposta il bit AA nelle risposte autoritative | non conosce lo stato di salute del servizio e non dovrebbe offrire ricorsione aperta per nomi esterni |

Un prodotto può implementare più ruoli, ma operativamente andrebbero comunque considerati separatamente. Un errore nel servizio autoritativo interessa la pubblicazione delle proprie zone; un errore nel servizio ricorsivo interessa la risoluzione dei nomi dei propri client. Processi, indirizzi o domini di guasto condivisi rendono più difficile questa distinzione.

## Risoluzione dei nomi passo dopo passo

Normalmente un’applicazione non interroga direttamente i server root e autoritativi. Il suo stub resolver passa una richiesta ricorsiva a un resolver. Se non vi è una voce di cache utilizzabile, quest’ultimo segue le deleghe dalla root attraverso il dominio di primo livello fino alla zona competente. Ogni referral indica il successivo RRset NS e, se necessario, indirizzi Glue. Il resolver compone quindi la risposta finale, eventualmente valida DNSSEC e memorizza nella cache il risultato ([RFC 1034, sezione 4.3](https://datatracker.ietf.org/doc/html/rfc1034#section-4.3)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1016" src="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813" title="Interaktive Infografik: DNS-Auflösung von Stub Resolver über Cache, Root und Delegationen bis zur autoritativen Antwort" loading="lazy">
  <a href="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813">Apri l’infografica sulla risoluzione DNS</a>
</iframe>

Il percorso lineare illustrato è un modello di avvio a freddo. In una cache del resolver già popolata, le deleghe root e TLD sono generalmente già presenti, quindi è necessaria solo una parte dei passaggi. Anche minimizzazione QNAME, forwarding, zone locali o cache DNSSEC aggressive possono modificare il percorso dei pacchetti visibile. Rimane fondamentale attribuire ogni osservazione a un ruolo: una risposta dalla cache non autoritativa non dimostra ciò che il server autoritativo competente stia fornendo in quel momento.

## Struttura tecnica di un messaggio DNS

Sul wire, un messaggio DNS è composto da Header, Question, Answer, Authority e Additional Section. La domanda indica QNAME, QTYPE e QCLASS. Risposte e riferimenti compaiono come Resource Record nelle altre sezioni. I flag indicano tra l’altro autorità, richiesta di ricorsione e troncamento; il Response Code descrive il risultato. Per l’analisi non conta quindi solo il testo dell’Answer, ma anche da quale server e con quali flag sia arrivato ([RFC 1035, sezione 4.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1)).

Per una diagnosi sono particolarmente rilevanti:

| Segnale | Significato | Tipica domanda dell’amministratore |
|---|---|---|
| `AA` | La risposta è autoritativa per il nome risposto | È stata interrogata direttamente la zona competente o solo una cache? |
| `TC` | La risposta è stata troncata per il trasporto utilizzato | Il tentativo tramite TCP funziona e il firewall consente TCP/53? |
| `RD` / `RA` | Ricorsione richiesta / offerta dal server | Un server autoritativo è stato usato per errore come resolver? |
| `AD` | Il validatore rispondente considera i dati autenticati | Il resolver è affidabile e il trasporto verso di esso è protetto? |
| `CD` | Il client richiede di non scartare gli errori di validazione nel resolver | Si sta validando DNSSEC o si stanno esaminando solo dati grezzi? |
| `RCODE` | Stato del risultato come NOERROR, NXDOMAIN, SERVFAIL o REFUSED | Il nome è errato, il tipo non esiste, la risoluzione è disturbata o la richiesta è stata rifiutata dalla policy? |

EDNS(0) aggiunge tramite uno pseudo **record OPT** flag, opzioni e un payload UDP annunciato più grande, senza sostituire il formato di base ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)). Il bit DNSSEC DO si trova in questo campo di flag esteso, non nell’header DNS originario.

## Resource Record e RRset

Il contenuto tecnico è costituito da Resource Record. Owner Name, tipo e classe determinano a cosa appartengano i dati; TTL e RDATA forniscono durata della cache e valore specifico del tipo. Tutti i record con lo stesso owner, tipo e classe formano un RRset e condividono un TTL. Più valori MX, A o AAAA sono quindi un insieme memorizzato nella cache congiuntamente, non singoli oggetti controllabili indipendentemente ([RFC 2181, sezione 5](https://datatracker.ietf.org/doc/html/rfc2181#section-5)).

| Tipo | Funzione | Limite importante |
|---|---|---|
| `SOA` | metadati della zona, Serial, Refresh/Retry/Expire e parametro di cache negativo | un RR SOA per zona all’apice; la modifica del Serial controlla la sincronizzazione con i secondary |
| `NS` | server autoritativi di una zona o delega | la destinazione di un NS non può essere un alias |
| `A` / [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596#section-2.1) | indirizzo IPv4 o IPv6 di un nome | non dice nulla sulla porta di servizio o sulla raggiungibilità |
| `CNAME` | alias di un nome verso un nome canonico | in linea di principio non può coesistere sullo stesso owner con altri dati |
| `MX` | Mail Exchanger con Preference | la destinazione deve risolversi in A/AAAA e non può essere un CNAME |
| `PTR` | mappatura inversa, in genere sotto `in-addr.arpa.` o `ip6.arpa.` | la zona inversa appartiene generalmente al titolare dell’indirizzo, non al gestore della zona diretta |
| `TXT` | una o più stringhe di caratteri prive di semantica tra protocolli | l’interpretazione deriva solo da SPF, DKIM, DMARC o un’altra procedura |
| [`SRV`](https://datatracker.ietf.org/doc/html/rfc2782) | servizio, trasporto, priorità, peso, porta e destinazione | il client deve implementare la semantica SRV del relativo servizio |
| [`CAA`](https://datatracker.ietf.org/doc/html/rfc8659) | policy delle autorità di certificazione per i nomi di dominio | non è crittografia del trasporto né un certificato del server |
| `DS`, `DNSKEY`, `RRSIG`, `NSEC` | catena di fiducia DNSSEC, chiavi, firme e inesistenza autenticata | protegge l’integrità dei dati DNS, non la loro riservatezza |

Il testo in un file di zona è soltanto la **Presentation Form**. In rete, i nomi sono trasmessi per label, i numeri in binario e alcuni nomi opzionalmente compressi. Chi copia una stringa da una UI dovrebbe quindi distinguere se l’interfaccia abbia già composto virgolette, sequenze di escape o più stringhe TXT in un payload logico.

## Delega, Glue e autorità

Con una delega, una zona trasferisce la responsabilità di un sottoalbero. Il padre pubblica l’RRset NS della zona figlia; la zona figlia indica nuovamente autonomamente i propri server autoritativi. Se le due parti differiscono, i resolver possono percorrere strade diverse. Gli indirizzi Glue nel padre risolvono solo il problema dell’uovo e della gallina della raggiungibilità. I valori A/AAAA autoritativi sul nome del nameserver restano dati indipendenti, con TTL e manutenzione propri.

Particolarmente critico è il **Glue in-bailiwick**: se `example.ch.` è delegato a `ns1.example.ch.`, il resolver necessita dell’indirizzo di `ns1.example.ch.` prima di poter interrogare la zona figlia. Senza Glue nascerebbe un ciclo di risoluzione. Se invece la delega punta a `ns1.provider.net.`, il suo indirizzo può essere determinato tramite un’altra catena di delega.

In caso di cambio del nameserver, occorre pertanto verificare almeno quattro stati: nuova zona caricata su tutti i server, RRset NS della zona figlia aggiornato, delega del padre aggiornata e Glue necessario aggiornato. Solo allora i vecchi server dovrebbero essere rimossi dall’esercizio o dalla raggiungibilità.

## Cache, TTL e risposte negative

Dopo la risoluzione, una risposta continua a vivere nelle cache. Il suo TTL è il periodo massimo di utilizzo a partire dal momento in cui il rispettivo resolver l’ha appresa. Non esiste quindi un conto alla rovescia globale condiviso. Stub resolver, forwarder, resolver ricorsivi e applicazioni possono scartare lo stesso vecchio RRset in momenti diversi. Un TTL ridotto prima di una migrazione aiuta solo per le risposte ricaricate dopo; i dati già memorizzati nella cache non possono essere richiamati.

Anche l’inesistenza viene memorizzata nella cache. **NXDOMAIN** significa che il nome richiesto non esiste; **NODATA** è una risposta NOERROR in cui il nome esiste, ma non esiste alcun RRset del tipo interrogato. Il server autoritativo inserisce in entrambi i casi il proprio RR SOA nella Authority Section. Il tempo di cache negativo è il minimo tra SOA-TTL e SOA.MINIMUM ([RFC 2308, sezioni da 3 a 5](https://datatracker.ietf.org/doc/html/rfc2308#section-3)). Ciò spiega perché un selettore DKIM o un hostname appena creato possa continuare inizialmente a sembrare inesistente dopo un precedente tentativo fallito.

`SERVFAIL` va distinto da ciò: il resolver non è riuscito a stabilire una risposta utilizzabile. Le cause includono timeout verso server autoritativi, una delega difettosa, un errore di validazione DNSSEC o un limite interno di risorse. `REFUSED` significa invece che il server interrogato non esegue l’operazione in base alla propria policy. Uno strumento di diagnostica deve quindi mostrare RCODE, bit AA, server rispondente e sezioni; un output che indica solo «nessun indirizzo» nasconde differenze decisive.

## Trasporto: UDP, TCP e percorsi del resolver cifrati

Per questo scambio, il percorso di rete deve consentire UDP e TCP sulla porta 53. TCP non è limitato ai trasferimenti di zona: un resolver può usarlo direttamente e deve poterlo utilizzare come fallback dopo una risposta UDP troncata. Il blocco di TCP emerge pertanto spesso solo con RRset grandi, ricchi di DNSSEC o con molte risposte. Piccole interrogazioni A restano verdi e danno un’impressione errata ([RFC 7766](https://datatracker.ietf.org/doc/html/rfc7766)).

Senza EDNS, il payload DNS tramite UDP è limitato a 512 byte. EDNS consente al richiedente di annunciare un payload ricevibile più grande. Un valore troppo elevato può tuttavia imporre la frammentazione IP; se un percorso perde o blocca frammenti, si verifica il tipico scenario in cui le risposte piccole funzionano e quelle grandi vanno in timeout. La specifica EDNS raccomanda di considerare la reale capacità di ricezione e il percorso e, in caso di problemi, di ricorrere a valori minori o a TCP ([RFC 6891, sezione 6.2](https://datatracker.ietf.org/doc/html/rfc6891#section-6.2)).

[DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858), [DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) o [DNS over QUIC](https://datatracker.ietf.org/doc/html/rfc9250) cifrano un percorso di trasporto DNS. Queste procedure non modificano né il contenuto della zona né la delega e non sostituiscono DNSSEC: la crittografia del trasporto protegge la connessione verso un resolver, DNSSEC autentica i dati lungo la catena di delega. Il DNS classico sulla porta 53 non è cifrato; la terminologia comune per i tipi di trasporto è definita da [RFC 9499, sezione 6](https://datatracker.ietf.org/doc/html/rfc9499#section-6).

## DNSSEC e catena di fiducia

DNSSEC aggiunge alla risoluzione dei nomi origine e integrità verificabili. Una zona firma gli RRset con RRSIG e pubblica le chiavi pubbliche come DNSKEY. Il padre collega la zona figlia tramite DS alla catena di fiducia sovrastante; NSEC o NSEC3 può attestare anche l’inesistenza. Un resolver validante parte dal proprio Trust Anchor e verifica questa catena fino alla risposta. Il contenuto resta pubblico: DNSSEC non cifra alcuna interrogazione ([RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033), [RFC 4034](https://datatracker.ietf.org/doc/html/rfc4034)).

Per l’esercizio non sono rilevanti solo i file di chiave, ma diversi stati accoppiati temporalmente:

- Le RRSIG hanno inizio e scadenza; un’ora di sistema errata o una rifirma interrotta può rendere **bogus** un’intera zona.
- Un DS nel padre deve corrispondere a un DNSKEY utilizzabile della zona figlia. Un DS orfano è peggiore per i resolver validanti di una delega volutamente non firmata.
- Durante i rollover, pubblicazione, firma, modifica nel padre, TTL e tempi di cache devono essere pianificati come una macchina a stati.
- Un resolver validante fornisce spesso SERVFAIL per dati bogus. Un test senza validazione può contemporaneamente mostrare una risposta apparentemente normale.

Il solo flag AD è affidabile solo quanto il resolver e il percorso verso di esso. Per una verifica indipendente, un amministratore deve esaminare la catena con uno strumento validante e localizzare la transizione difettosa tra DS, DNSKEY e RRSIG.

## Gestione autoritativa e flusso dei dati

Dietro la risposta autoritativa si trova un proprio percorso di distribuzione. In un modello classico, una fonte primaria gestisce la zona, DNS NOTIFY informa i secondary di un nuovo SOA Serial e AXFR o IXFR trasferiscono dati completi o incrementali. Un listener sano non dimostra quindi ancora che il server abbia caricato la nuova zona. Serial, stato del trasferimento e risposta di ogni nodo autoritativo devono essere considerati insieme ([RFC 1996](https://datatracker.ietf.org/doc/html/rfc1996), [RFC 5936](https://datatracker.ietf.org/doc/html/rfc5936), [RFC 1995](https://datatracker.ietf.org/doc/html/rfc1995)).

La zona può essere generata da file di testo, un database, un’API, Active Directory o una pipeline Git/CI. Questa scelta di implementazione non modifica il protocollo DNS sul wire, ma determina confini transazionali, auditabilità e recovery. RFC 2136 definisce aggiornamenti dinamici atomici con Prerequisites; una chiamata API di un fornitore è invece un protocollo di controllo separato e deve documentare proprie regole di coerenza e gestione degli errori ([RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136)).

Un backup DNS è utilizzabile solo se da esso può essere ripristinata la zona effettivamente distribuita. A seconda della piattaforma, ciò include:

- zona o database sorgente inclusi SOA Serial e journal dinamico;
- configurazione del server, views, ACL, forwarder e assegnazione al catalogo;
- TSIG secrets, DNSSEC private keys e stati dei rollover automatici;
- dati del padre esterni alla propria zona, in particolare delega, Glue e DS;
- un percorso testato per rifornire nuovamente i secondary e verificare semanticamente i dati della zona.

I secondary sono copie per la disponibilità, ma non automaticamente un backup storico. Una modifica errata o malevola può essere replicata rapidamente su tutti i server autoritativi tramite NOTIFY e trasferimento di zona.

## Stack di implementazione e tecnologie

DNS non designa un singolo demone. Lo stack tecnologico comune comprende nomi e RRset, formato binario query/response, UDP e TCP, logica della cache e DNSSEC opzionale. Server autoritativi, resolver ricorsivi, forwarder e Managed DNS implementano questi elementi in modo diverso. Per l’esercizio, conta quindi prima il ruolo di un prodotto, poi il suo linguaggio o packaging:

| Implementazione | Ruolo primario | Focus tecnico |
|---|---|---|
| [BIND 9](https://bind9.readthedocs.io/en/latest/) | autoritativo e/o ricorsivo | nameserver universale, file di zona, aggiornamenti dinamici, DNSSEC e strumenti diagnostici |
| [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) | resolver ricorsivo con cache | pipeline modulare per resolver, validatore e cache; nessun ruolo autoritativo principale |
| [Knot DNS](https://www.knot-dns.cz/) | autoritativo | solo autoritativo, elaborazione parallela, trasferimento di zona, DDNS e DNSSEC |
| [Windows Server DNS](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) | autoritativo e ricorsivo | zone opzionalmente integrate in AD, aggiornamenti dinamici sicuri, policy, cache e forwarding |

La separazione dei ruoli è più importante del nome del prodotto. Per una piattaforma autoritativa pubblica contano provisioning delle zone, secondary, firma DNSSEC e resilienza DDoS; per un resolver aziendale contano cache, forwarding, spazi dei nomi interni, policy, privacy e validazione.

## DNS nella gestione di posta e identità

Nella messaggistica, una risposta DNS diventa direttamente un percorso di consegna. Un MTA mittente interroga gli RRset MX del dominio destinatario, preferisce il valore più basso e tratta preferenze uguali come equivalenti. Solo se non esiste alcun MX, il dominio stesso vale come destinazione implicita. Non appena esistono record MX, un valore A/AAAA all’apice del dominio non è un sostituto. Ogni destinazione MX richiede indirizzi propri e non può essere un alias; un dominio che non accetta posta pubblica Null MX `0 .` ([RFC 5321, sezione 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 2181, sezione 10.3](https://datatracker.ietf.org/doc/html/rfc2181#section-10.3), [RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)).

I record PTR vengono risolti attraverso l’albero degli indirizzi inverso. Le zone diretta e inversa hanno spesso proprietari diversi; le modifiche devono pertanto essere coordinate tra gestore del dominio e gestore dell’indirizzo IP. Un PTR è un nome, non una prova crittografica dell’identità. Le piattaforme di posta riceventi possono usare una mappatura diretta/inversa coerente come segnale, ma la loro specifica policy di reputazione o accettazione non è una caratteristica DNS.

SPF, DKIM e DMARC usano DNS come canale di pubblicazione, ma definiscono regole di valutazione proprie. SPF legge un singolo payload TXT logico e limita i termini che causano DNS; DKIM indirizza le chiavi tramite selettori; DMARC si trova sotto `_dmarc`. Queste procedure rientrano tecnicamente in [SPF, DKIM e DMARC](/kb/mail-auth). L’amministratore DNS deve soprattutto gestire correttamente owner name, suddivisione delle stringhe TXT, dimensione della risposta, TTL, delega e tempi di cache negativi.

Anche [LDAP](/kb/ldap) e [Kerberos](/kb/kerberos) usano spesso record SRV per la ricerca dei servizi. Un record SRV contiene, oltre a destinazione e porta, una priorità e un peso. Questi valori non sono una configurazione universale di load balancer; li interpretano solo i client che implementano la relativa procedura SRV.

## Diagnostica

La diagnostica non inizia quindi mai con «DNS non funziona», bensì con nome, tipo, classe, server interrogato, trasporto e momento. Resolver aziendali, resolver pubblici e server autoritativi possono fornire temporaneamente o a causa di Split DNS e policy risposte diverse. Questa divergenza non è rumore di misurazione, ma l’indizio più importante del punto in cui il percorso di risoluzione si divide.

### Interrogare RRset in modo mirato

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gezielte DNS-Abfragen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) può forzare un server specifico, tipo di record, solo DNS e TCP. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) mostra inoltre flag, sezioni, RCODE, server rispondente e tempo di query. Senza `-Server` o `@server` esplicito viene testato il resolver configurato, non necessariamente la fonte autoritativa.

### Verificare separatamente TCP e DNSSEC

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Transport- und DNSSEC-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`delv`](https://bind9.readthedocs.io/en/latest/manpages.html#delv-dns-lookup-and-validation-utility) usa la logica di resolver e validatore BIND per verificare una catena DNSSEC. Una query riuscita con `+dnssec` dimostra invece soltanto che sono stati richiesti e forniti dati DNSSEC; non valida automaticamente la catena. Il test TCP separato rileva firewall che consentono UDP/53 ma bloccano TCP/53.

### Svuotare miratamente le cache locali

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem zum Leeren des lokalen DNS-Caches">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache) svuota la cache del client DNS di Windows. [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html) gestisce la cache di `systemd-resolved`; sui sistemi con `nscd`, `dnsmasq`, un Unbound locale o una cache dell’applicazione, è responsabile un’altra cache. Uno svuotamento del client non modifica mai la cache di un resolver upstream.

### Assegnare lo scenario di errore al punto di transizione

| Osservazione | Livello probabile | Prossima verifica |
|---|---|---|
| un resolver fornisce un valore vecchio, i server autoritativi quello nuovo | cache positiva | TTL rimanente, catena di forwarder, cache dell’applicazione |
| NXDOMAIN persiste dopo la creazione del record | cache negativa o zona errata | SOA nella risposta negativa, delega del padre, owner name |
| NOERROR senza Answer | il nome esiste, manca il tipo richiesto | catena CNAME, QTYPE esatto, SOA NODATA |
| vanno in timeout solo risposte grandi o firmate | EDNS, frammentazione o fallback TCP | `TC`, dimensione EDNS minore, test TCP esplicito, firewall |
| il resolver validante fornisce SERVFAIL, quello non validante una risposta | DNSSEC bogus | corrispondenza DS/DNSKEY, tempi RRSIG, algoritmo, ora di sistema |
| i server autoritativi forniscono Serial diversi | replica | NOTIFY, IXFR/AXFR, ACL di trasferimento, journal, fonte primaria |
| client pubblici e interni vedono destinazioni diverse | Split DNS o policy del resolver | resolver interrogato, assegnazione View, subnet del client, forwarder |
| MX esiste, la consegna fallisce prima di SMTP | nome di destinazione derivato o trasporto | MX Preference, A/AAAA della destinazione MX, TCP/25, divieto CNAME |

## Monitoraggio e criteri operativi

Il monitoraggio deve rappresentare lo stesso percorso. Una singola ricerca A contro il resolver standard locale non rileva né una delega difettosa né TCP bloccato, firme scadute o un secondary con Serial vecchio. Per messaggistica e identità occorre pertanto rilevare separatamente almeno i seguenti segnali:

- risposte autoritative di ogni NS pubblicato tramite UDP e TCP, inclusi bit AA e SOA Serial;
- delega del padre, RRset NS della zona figlia, Glue e, per zone firmate, transizione DS/DNSKEY;
- latenza ricorsiva, rapporto di cache hit, tassi di timeout, SERVFAIL, REFUSED e NXDOMAIN;
- dimensione della risposta, troncamento, errori EDNS e fallback TCP;
- scadenze delle RRSIG, stato del key rollover e code di firma;
- successo di NOTIFY, AXFR e IXFR nonché età della zona su ogni secondary;
- RRset funzionali come MX, indirizzi A/AAAA associati, PTR e nomi TXT necessari per [autenticazione della posta](/kb/mail-auth).

Un test sintetico dovrebbe verificare sia il normale percorso del client sia la fonte autoritativa. Solo così è possibile distinguere se un problema risieda nei dati pubblicati, nella delega, nella cache di un resolver o nell’applicazione.

## Sicurezza e domini di guasto

I due ruoli del server richiedono misure di protezione diverse. La ricorsione deve essere disponibile solo per client affidabili; un resolver aperto può essere abusato per attacchi di riflessione e amplificazione. I server autoritativi devono invece rimanere raggiungibili a livello mondiale, ma non devono risolvere ricorsivamente nomi esterni arbitrari. Mescolare i ruoli amplia la superficie di attacco e rende più difficili da riconoscere le cause del carico ([RFC 5358](https://datatracker.ietf.org/doc/html/rfc5358)).

DNSSEC protegge gli RRset pubblicati da modifiche inosservate, non il processo del server né la disponibilità. **TSIG** autentica singoli messaggi DNS con una chiave condivisa ed è usato tra l’altro per aggiornamenti e trasferimenti di zona; non è una firma pubblica dei dati della zona ([RFC 8945](https://datatracker.ietf.org/doc/html/rfc8945)). ACL di trasferimento, TSIG secrets e DNSSEC private keys sono oggetti di protezione distinti.

Split DNS e risposte basate su policy possono essere necessari, ma producono più verità per lo stesso QNAME/QTYPE. Devono essere documentati almeno criterio di assegnazione, zona sorgente, percorso di forwarding, comportamento DNSSEC e monitoraggio per View. Altrimenti, alla successiva interruzione una deviazione voluta verrà interpretata come errore di cache o problema di replica.

## Storia tecnica

Prima di DNS, Internet distribuiva tabelle host gestite centralmente. Con l’aumento delle reti, degli host e dei gestori indipendenti, questa procedura divenne un collo di bottiglia. Paul Mockapetris descrisse nel 1983, in RFC 882 e RFC 883, un servizio dei nomi gerarchico e delegabile. RFC 1034 e RFC 1035 sostituirono questa versione nel 1987 e, come STD 13, costituiscono tuttora il nucleo del sistema.

La prima implementazione server funzionante, **Jeeves**, girava nel 1983/84 su sistemi DEC-TOPS-20. Poco dopo, presso la University of California, Berkeley, con finanziamento DARPA, nacque per Unix il Berkeley Internet Name Domain Package **BIND**. BIND 8 apparve nel 1997, BIND 9 nel settembre 2000 come ampia riscrittura. ISC documenta questa storia dello sviluppo in [A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html).

Il protocollo si è evoluto gradualmente senza sostituire il nucleo gerarchico: NOTIFY e IXFR accelerarono negli anni Novanta la sincronizzazione delle zone, EDNS estese nel 1999 il modello dei messaggi e fu poi consolidato in RFC 6891, DNSSEC ricevette nel 2005 gli attuali meccanismi fondamentali DNSKEY/DS/RRSIG e i trasporti cifrati verso resolver furono aggiunti con DoT, DoH e DoQ. DNS non è quindi un protocollo congelato del 1987, ma un sistema estensibile la cui retrocompatibilità e i lunghi stati di cache caratterizzano operativamente ogni modifica.

## Fonti

- [RFC Editor – STD 13: Domain Name System](https://www.rfc-editor.org/info/std13)
- [RFC 9499 – DNS Terminology](https://datatracker.ietf.org/doc/html/rfc9499) – terminologia attuale per ruoli, zone, cache, DNSSEC e trasporto.
- [RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034) – spazio dei nomi, zone, delega, resolver e ruoli dei server.
- [RFC 1035 – Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) – formato wire, Resource Record, sezioni dei messaggi e Master Files.
- [RFC 6891 – Extension Mechanisms for DNS (EDNS(0))](https://datatracker.ietf.org/doc/html/rfc6891) – OPT, flag estesi e payload UDP.
- [RFC 2181, sezione 5](https://datatracker.ietf.org/doc/html/rfc2181)
- [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596)
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – struttura e selezione dei record SRV.
- [RFC 8659 – DNS Certification Authority Authorization](https://datatracker.ietf.org/doc/html/rfc8659) – record CAA e valutazione da parte delle autorità di certificazione.
- [RFC 2308 – Negative Caching of DNS Queries](https://datatracker.ietf.org/doc/html/rfc2308) – NXDOMAIN, NODATA, SOA e tempo di cache negativo.
- [RFC 7766 – DNS Transport over TCP](https://datatracker.ietf.org/doc/html/rfc7766) – supporto TCP obbligatorio e comportamento della connessione.
- [RFC 7858 – DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858) – DNS tramite TLS.
- [RFC 8484 – DNS Queries over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) – query DNS tramite HTTPS.
- [RFC 9250 – DNS over Dedicated QUIC Connections](https://datatracker.ietf.org/doc/html/rfc9250) – DNS tramite QUIC.
- [RFC 4033 – DNS Security Introduction and Requirements](https://datatracker.ietf.org/doc/html/rfc4033) – obiettivi di protezione DNSSEC, validazione e limiti.
- [RFC 4034 – Resource Records for DNSSEC](https://datatracker.ietf.org/doc/html/rfc4034) – DNSKEY, DS, RRSIG e NSEC.
- [RFC 1996 – DNS NOTIFY](https://datatracker.ietf.org/doc/html/rfc1996) – notifica dei server secondary.
- [RFC 5936 – DNS Zone Transfer Protocol (AXFR)](https://datatracker.ietf.org/doc/html/rfc5936) – trasferimenti completi di zona tramite TCP.
- [RFC 1995 – Incremental Zone Transfer (IXFR)](https://datatracker.ietf.org/doc/html/rfc1995) – sincronizzazione incrementale delle zone.
- [RFC 2136 – Dynamic Updates in DNS](https://datatracker.ietf.org/doc/html/rfc2136) – modifiche atomiche con Prerequisites.
- [BIND 9](https://bind9.readthedocs.io/en/latest/)
- [Unbound Documentation – unbound(8)](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) – cache ricorsiva e validazione DNSSEC.
- [Knot DNS](https://www.knot-dns.cz/) – implementazione solo autoritativa e funzioni operative.
- [Microsoft Learn – DNS in Windows Server](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) – ruoli DNS di Windows, integrazione AD, cache e forwarding.
- [RFC 5321, sezione 5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7505 – Null MX](https://datatracker.ietf.org/doc/html/rfc7505) – indicazione esplicita dei domini senza ricezione di posta.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – query DNS in Windows.
- [ISC BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html) – opzioni di query, selezione del server e interpretazione dell’output.
- [`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache)
- [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html)
- [RFC 5358 – Preventing Use of Recursive Nameservers in Reflector Attacks](https://datatracker.ietf.org/doc/html/rfc5358) – limitazione della ricorsione e protezione dagli abusi.
- [RFC 8945 – Secret Key Transaction Authentication for DNS (TSIG)](https://datatracker.ietf.org/doc/html/rfc8945) – autenticazione dei messaggi per aggiornamenti e trasferimenti.
- [ISC – A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html) – Jeeves, Berkeley, BIND 8 e BIND 9.
