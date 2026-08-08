---
title: "Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le insidie più comuni"
navTitle: "Record DNS per e-mail"
description: "Chi gestisce una zona riceve di solito i record e-mail già pronti e deve solo pubblicarli. Cosa va regolarmente storto: il limite di 255 byte per DKIM, record SPF duplicati, il limite di lookup, MX su un CNAME, il suffisso della zona aggiunto automaticamente e policy che nessuno applica più."
date: "2026-08-04"
kategorie: "SMTP e flusso di posta"
timeToRead: "15 min di lettura"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
produkte:
  - "uebergreifend"
protokolle:
  - "dns"
  - "smtp"
  - "tls"
  - "verschluesselung"
  - "mail-auth"
hauptthema: "smtp-mailflow"
related:
  - smtp-verbindung-testen-linux
  - ghost-sender-exchange-online-nebeneingang
slug: "guida-per-amministratori-dns-mx-spf-dkim-dmarc-e-le-insidie-piu-comuni"
translationId: "article-e4699ad7fcea2e20"
aiPrompt: |
  Du bist mein Assistent für DNS-Records rund um E-Mail. Ich gebe dir einen Record-Wert oder eine Zonendatei, du prüfst sie gegen die Regeln aus diesem Artikel: Syntax, doppelte Records, SPF-Lookup-Limit und Void-Lookups, DKIM-Base64 auf Copy-Paste-Schäden, DMARC-Tags nach RFC 9989 inklusive sp und np, externe Report-Adressen mit Autorisierungsrecord, MX ohne CNAME-Ziel, MTA-STS-ID. Frage mich zuerst: 1. um welche Domain und welchen Record es geht, 2. ob die Domain sendet, empfängt oder beides, 3. welche Versanddienste beteiligt sind (Marketing, ERP, Ticketsystem, Scan-to-Mail), 4. welches DNS-System die Zone hält. Gib mir am Ende den korrigierten Record als kopierfertige Zeile plus die dig-Befehle zur Kontrolle.
translationOf: dns-records-e-mail-stolpersteine
url: https://rafaelpfister.ch/it/blog/guida-per-amministratori-dns-mx-spf-dkim-dmarc-e-le-insidie-piu-comuni
translationSourceHash: dc806bed491a47ecc1118249566d9303b0201f4bdb5153a966385a7c9373b31f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-08T14:04:35.733Z
translationReview: automatic
---

# Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le insidie più comuni

Chi gestisce una zona DNS raramente riceve record e-mail scritti da sé. Il team e-mail, un provider o un fornitore di servizi di marketing invia una riga con l'indicazione che debba "solo essere pubblicata". È proprio qui che nascono la maggior parte degli errori, perché i record e-mail sono il tipo di record in cui un refuso può avere due conseguenze completamente diverse. O la consegna si interrompe subito e qualcuno si fa sentire nel giro di pochi minuti, oppure continua invariata e fallisce silenziosamente solo la verifica del mittente. Il secondo caso passa regolarmente inosservato per mesi, finché un grande destinatario non mette il dominio in quarantena.

Da quando Google e Yahoo hanno inasprito i requisiti per i mittenti di grandi volumi nel febbraio 2024 e Microsoft ha seguito l'esempio nel maggio 2025, la tolleranza per i domini configurati a metà è diventata ridotta. SPF, DKIM e un record DMARC non sono più un optional per i mittenti oltre un certo volume, ma un requisito per la consegna.

Tutti gli esempi di questo articolo utilizzano `example.com` e selettori generici. I valori mostrati sono abbreviati per mantenerli leggibili.

## Regole valide per ogni record e-mail

### Il limite di 255 byte nei record TXT

Secondo RFC 1035, un record TXT è composto da una o più `character-strings`, e una singola stringa di questo tipo può contenere al massimo 255 byte. Il record nel suo complesso può essere più lungo, ma deve allora essere suddiviso in più stringhe. I sistemi di valutazione ricompongono queste parti senza separatori.

Questo diventa rilevante nella pratica esattamente in un punto: con chiavi DKIM a 2048 bit. Il loro valore Base64 è lungo circa 400 caratteri e non entra in una sola stringa.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La maggior parte dei sistemi di gestione DNS esegue autonomamente questa suddivisione quando il valore viene inserito nel normale campo di input. Chi invece aggiunge manualmente le virgolette deve rispettare esattamente il limite. Un valore spezzato con uno spazio nel punto di giunzione produce una chiave che esiste sintatticamente ma non è più valida crittograficamente.

Il controllo successivo è importante, perché una chiave composta in modo errato appare del tutto normale nella GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

### Un record per scopo

SPF e DMARC sono definiti in modo tale che a un nome possa corrispondere esattamente un record appropriato. Con SPF, due record `v=spf1` portano a un `permerror`, e la verifica viene quindi considerata fallita, non superata. Con DMARC, i destinatari ignorano completamente il dominio se più record iniziano con `v=DMARC1`: invece di applicare una policy rigorosa, non ne viene applicata alcuna.

Questo è di gran lunga l'errore più frequente nelle zone evolute nel tempo. Viene collegato un nuovo fornitore di servizi, qualcuno aggiunge il "proprio" record SPF invece di estendere quello esistente e, da quel momento, la verifica fallisce per tutti i mittenti. Prima di ogni nuovo record, è quindi indispensabile controllare cosa esiste già:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

Per DKIM vale il contrario: è previsto un record per selettore e più selettori affiancati sono la norma, poiché ogni servizio di invio dispone della propria chiave.

### Il suffisso della zona nelle interfacce web

In Infoblox, nel DNS di Windows e in quasi tutte le interfacce di hosting, il nome della zona viene aggiunto automaticamente al nome inserito. Chi inserisce il nome pienamente qualificato nel campo "Nome" ottiene un record lungo il doppio del previsto:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Nel file di zona, il corrispettivo è il punto finale mancante. `mail.example.com` senza punto finale è un nome relativo e viene completato con il nome della zona; `mail.example.com.` con il punto è assoluto. Per le destinazioni MX e CNAME, questo singolo punto determina se il dominio è raggiungibile.

### Il copia-incolla è la fonte di errori più comune

I valori dei record e-mail non vengono quasi mai digitati, ma copiati da un PDF, un ticket, una cella Excel o una chat. Così si producono problemi invisibili nel campo di input:

- Un doppio `p=` all'inizio della chiave DKIM, perché il prefisso è stato impostato due volte durante la composizione. Il valore `v=DKIM1;k=rsa;p=p=MIIBIjAN...` è un classico reale e produce una chiave inutilizzabile.
- Virgolette tipografiche da Word invece di quelle dritte.
- Spazi non interrompibili dai layout PDF, che sembrano normali.
- Interruzioni di riga nel blocco Base64 quando il valore nel PDF era distribuito su più righe.

Base64 conosce esattamente i caratteri da A a Z, da a a z, da 0 a 9, `+`, `/` e `=` come carattere di riempimento. Qualsiasi altro carattere nella parte `p=` è un errore. Un breve filtro prima dell'inserimento evita successive ricerche del problema:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

Se qui compare qualcosa di diverso da `0`, la chiave contiene caratteri estranei.

### Ridurre il TTL prima delle modifiche

Prima di ogni modifica pianificata a un record MX, SPF o DKIM, il TTL deve essere impostato per alcune ore su un valore basso, tipicamente 300 secondi. Altrimenti il vecchio valore rimane nei resolver esterni per un giorno o più, a seconda della zona, e anche un rollback richiede lo stesso tempo. Dopo la modifica e una fase di osservazione, il TTL viene riportato al valore regolare.

## MX

Il record MX stabilisce quale host riceve e-mail per il dominio. Esistono due regole che vengono regolarmente violate.

**La destinazione deve essere un hostname con record A o AAAA.** Non sono consentiti né un indirizzo IP né un CNAME. RFC 2181 afferma espressamente che la destinazione di un record MX non può essere un alias. Nella pratica funziona comunque con molti destinatari, ma non con altri, producendo problemi che apparentemente riguardano solo singoli mittenti.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Il numero è una preferenza, non una ponderazione.** Si prova prima il valore più basso. Un secondo MX con un numero alto ha senso solo se quel sistema conosce lo stesso filtro per i destinatari. I record MX di backup su sistemi senza verifica dei destinatari sono un bersaglio popolare per lo spam, perché gli aggressori puntano deliberatamente all'entry più debole.

I domini che inviano soltanto o che non hanno nulla a che fare con la posta ricevono un Null MX secondo RFC 7505. Segnala che il dominio non accetta posta e assicura un rifiuto immediato e inequivocabile anziché timeout:

```text
example.com.  IN  MX  0 .
```

Tuttavia, il Null MX non sostituisce un record SPF e DMARC. Non ricevere posta non significa che nessuno invii messaggi a vostro nome. In particolare, i sottodomini parcheggiati vengono usati per lo spoofing, perché raramente qualcuno li controlla.

## A, AAAA, PTR e il nome HELO

Il record PTR per l'indirizzo IP in uscita non si trova nella vostra zona, bensì nella zona `in-addr.arpa` del provider proprietario del blocco di indirizzi. Va quindi richiesto al provider e non impostato autonomamente. Molti grandi destinatari richiedono che PTR e il corrispondente record diretto coincidano, ossia che il nome nel PTR risolva nuovamente allo stesso indirizzo IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

Il nome che il vostro server e-mail indica in HELO o EHLO dovrebbe essere lo stesso e anch'esso risolvibile. Un gateway che si presenta come `localhost.localdomain` o con un nome interno riceve una valutazione peggiore dai destinatari più grandi.

Occorre prestare attenzione quando si aggiunge un record AAAA. Non appena il server e-mail diventa raggiungibile e invia tramite IPv6, si applicano gli stessi requisiti di IPv4, in parte persino più rigidi. Google richiede un PTR valido per gli indirizzi IPv6 che inviano messaggi. Se manca, l'invio viene rifiutato, mentre tramite IPv4 funzionava perfettamente. Un record AAAA sul server e-mail non è quindi mai una semplice modifica DNS.

## SPF

SPF stabilisce quali sistemi possono inviare a nome del dominio. Il record si trova come TXT sul dominio stesso.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Il limite di lookup

La valutazione di un record SPF può generare al massimo dieci meccanismi che interrogano il DNS. Vengono conteggiati `include`, `a`, `mx`, `ptr`, `exists` e `redirect`, in modo ricorsivo: ogni `include` porta con sé i lookup del record incluso. Non vengono conteggiati `ip4`, `ip6` e `all`.

Se il limite viene superato, il risultato è un `permerror`. Per DMARC ciò significa SPF non superato, indipendentemente dal fatto che il server mittente fosse effettivamente autorizzato. L'aspetto insidioso è che l'errore spesso si verifica senza alcun intervento: un provider incluso amplia il proprio record. Il proprio record non è cambiato, ma la consegna subisce comunque un calo.

Inoltre, sono consentiti al massimo due "void lookup", ossia interrogazioni senza risultato. Un `include` verso un dominio che non esiste più rientra in questo conteggio. I riferimenti a fornitori dismessi devono quindi essere rimossi e non lasciati per prudenza.

### Cosa non deve comparire in un record SPF

- **`ptr`** è specificato, ma è considerato obsoleto da RFC 7208 e non deve essere usato. I sistemi di valutazione possono ignorarlo.
- **`+all`** autorizza qualsiasi mittente ed è quindi più dannoso dell'assenza totale di un record SPF.
- **`?all`** è neutro e quindi praticamente inutile per DMARC.
- **Un record separato di tipo SPF (tipo 99)** non è più necessario. È stato eliminato con RFC 7208; SPF è esclusivamente in TXT.

La scelta fra `~all` (softfail) e `-all` (hardfail) dipende da quanto siano completi i percorsi di invio rilevati. Finché vi sono dubbi, `~all` è la scelta corretta. Chi applica già DMARC e analizza i report può passare a `-all`.

### I sottodomini non ereditano nulla

Un record SPF su `example.com` non vale per `newsletter.example.com`. Ogni sottodominio mittente necessita di un record proprio. Per tutti gli altri è consigliabile un record wildcard che chiarisca che da lì non proviene nulla:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attenzione: una wildcard TXT risponde anche alle richieste per nomi quali `_dmarc.sub.example.com`, se non esiste un record esplicito. Di solito non è un problema, ma può confondere la ricerca degli errori, perché ogni query TXT riceve una risposta.

### SPF flattening

Gli strumenti che risolvono tutti i riferimenti `include` e li sostituiscono con gli indirizzi IP sottostanti eliminano il limite di lookup a scapito della manutenibilità. Se il provider modifica i propri indirizzi, l'invio si interrompe e nessuno se ne accorge, perché nel record locale apparentemente è tutto corretto. Chi percorre questa strada necessita quindi di una verifica automatizzata che confronti regolarmente l'elenco con la fonte. Come lavoro manuale una tantum, prima o poi il metodo fallisce.

## DKIM

DKIM firma i messaggi in uscita. La chiave pubblica si trova in `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Il selettore è liberamente scelto e viene specificato dal sistema mittente. Un nome descrittivo con data facilita la successiva rotazione molto più di `s1` e `s2`.

### Delega tramite CNAME

Dove il servizio di invio lo offre, la variante CNAME è preferibile all'inserimento diretto:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Il provider può quindi ruotare autonomamente la propria chiave senza che qualcuno debba intervenire nella vostra zona. Altrimenti proprio questa rotazione resta regolarmente in sospeso, perché richiede coordinamento tra due team. Un CNAME esclude tuttavia qualsiasi altro record con lo stesso nome: è una regola di base del DNS, non una peculiarità di DKIM.

### Rotazione senza interruzioni

Durante il cambio di chiave, si pubblica prima il nuovo selettore, poi si configura il server mittente affinché lo utilizzi e solo dopo si rimuove il vecchio record. Chi elimina immediatamente la vecchia chiave invalida le firme di tutti i messaggi ancora in transito o in coda e rende impossibili le verifiche successive. Un intervallo di alcuni giorni tra il cambio e l'eliminazione è appropriato.

Un record con `p=` vuoto non è un record difettoso, bensì il modo specificato per contrassegnare una chiave come ritirata.

### Lunghezza della chiave

1024 bit sono considerati obsoleti, 2048 bit sono lo standard. Chiavi RSA più grandi non offrono praticamente alcun vantaggio aggiuntivo e aumentano soltanto la probabilità che un sistema intermedio non elabori correttamente il record.

## DMARC

DMARC combina SPF e DKIM con un'istruzione su cosa debba accadere quando una verifica fallisce e restituisce report. Il record si trova in `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Da maggio 2026, con RFC 9989 e le specifiche sui report RFC 9990 e RFC 9991, è in vigore la versione rivista che sostituisce RFC 7489. Per la pratica sono importanti tre modifiche:

- **`pct` è stato eliminato.** L'introduzione graduale tramite una percentuale non esiste più. Al suo posto c'è `t=y`, che contrassegna il dominio come in fase di test: i report continuano, ma la policy non deve essere applicata.
- **`np` è nuovo.** Imposta la policy per sottodomini inesistenti, colmando così una lacuna che gli aggressori sfruttano volentieri, poiché finora i sottodomini inventati erano coperti solo da `sp`. Senza indicazione esplicita, `np` segue il valore di `sp`.
- **La Public Suffix List è sostituita da un `Tree Walk`.** Il dominio organizzativo non viene più determinato da un elenco mantenuto esternamente, bensì tramite interrogazioni DNS progressive lungo l'albero dei nomi. Per grandi spazi dei nomi con molti livelli, questo modifica sensibilmente la valutazione.

### L'allineamento è il vero nucleo

DMARC non passa perché SPF o DKIM hanno superato tecnicamente la verifica, ma solo se almeno uno dei due corrisponde inoltre al dominio del mittente visibile nell'header `From`. SPF viene verificato rispetto al dominio del mittente envelope, che differisce regolarmente nei inoltri, nei servizi newsletter e nei sistemi di ticketing. Proprio per questo, messaggi con SPF valido talvolta non superano la verifica DMARC.

Con `adkim=r` e `aspf=r` (relaxed, lo standard), è sufficiente la corrispondenza a livello di dominio organizzativo. `s` richiede un'uguaglianza esatta incluso il sottodominio e, nella pratica, fallisce quasi sempre in uno dei percorsi di invio.

### Gli indirizzi di report esterni necessitano di autorizzazione

Se i report devono essere inviati a un indirizzo esterno al proprio dominio, ad esempio a un servizio di analisi DMARC, il dominio ricevente deve autorizzarlo. Senza questo record, molti destinatari semplicemente non inviano nulla e l'analisi resta vuota, mentre nel proprio record tutto sembra corretto:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Questo record viene creato dal gestore della zona di destinazione, non da voi. Per i servizi commerciali avviene automaticamente, ma non per una casella di raccolta gestita autonomamente in un altro dominio di vostra proprietà.

### Errori di sintassi tipici

I nomi dei tag e i valori di policy devono essere in minuscolo; `p=Reject` non è valido. I tag sono separati da un punto e virgola; un separatore mancante rende inefficace il resto della riga. Inoltre, `p` deve essere il primo tag dopo `v`. Un record composto solo da `v=DMARC1; rua=...` non contiene alcuna policy ed è incompleto.

### Il rollout

`p=none` è uno stato di misurazione, non un obiettivo. Non modifica il trattamento delle vostre e-mail da parte dei destinatari e serve soltanto a individuare, tramite i report, tutti i percorsi di invio legittimi. Chi dopo l'introduzione non passa entro pochi mesi da `quarantine` a `reject`, ha sostenuto lo sforzo senza ottenere la protezione. L'aspetto organizzativo di questo percorso, incluso il documento decisionale, è un tema a sé ed è descritto nel blueprint DMARC.

## MTA-STS e TLS-RPT

SMTP cifra in modo opportunistico: se la controparte offre STARTTLS, la connessione viene cifrata; altrimenti no. Un aggressore in grado di manipolare il traffico può rimuovere l'annuncio STARTTLS e mantenere così la connessione in chiaro. MTA-STS chiude questa lacuna per i domini riceventi.

MTA-STS è composto da due parti e solo una di esse si trova nel DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La policy effettiva si trova come file in `https://mta-sts.example.com/.well-known/mta-sts.txt` e deve essere servita tramite un certificato valido:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Le insidie sono quasi tutte al di fuori della zona:

- **L'`id` deve cambiare a ogni modifica della policy.** È l'unico segnale per i sistemi mittenti che una nuova policy deve essere recuperata. Chi modifica il file e lascia invariato l'`id` continua a lavorare contro copie in cache fino alla scadenza di `max_age`.
- **L'elenco MX nella policy e i record MX devono coincidere.** Un nuovo MX assente dalla policy viene rifiutato dai mittenti con `mode: enforce`. Durante le migrazioni, la policy deve quindi essere adattata prima del cambio degli MX.
- **Prima `mode: testing`.** In questa modalità le violazioni vengono soltanto segnalate, non applicate. Il passaggio a `enforce` avviene quando i report sono puliti.
- **Un record CAA può bloccare l'emissione del certificato per l'host della policy**, se vi è registrata un'autorità di certificazione diversa da quella utilizzata.

TLS-RPT fornisce i report corrispondenti ed è un singolo record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT è utile anche senza MTA-STS, perché rende visibili i fallimenti della cifratura del trasporto.

## DANE

DANE raggiunge lo stesso obiettivo di MTA-STS, ma ancora la fiducia nel DNS anziché nella PKI web. Richiede una zona firmata integralmente con DNSSEC e senza DNSSEC un record TLSA è inefficace.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Fondamentale durante l'esercizio: a ogni cambio di certificato, il record TLSA deve essere corretto in anticipo. La procedura abituale pubblica il nuovo hash accanto a quello vecchio, poi cambia il certificato e infine rimuove il vecchio record. Chi inverte quest'ordine rende il server e-mail irraggiungibile per tutti i mittenti che verificano DANE, tra cui i grandi provider di lingua tedesca. In Svizzera DANE è molto meno diffuso di MTA-STS, soprattutto per la mancanza della firma DNSSEC della zona.

## BIMI

BIMI mostra il logo del marchio nella casella di posta ed è l'unico meccanismo qui trattato che non è ancora un RFC, ma continua a essere pubblicato come Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

I requisiti sono elevati: una policy DMARC applicata con `quarantine` o `reject`, un logo in formato SVG Tiny Portable/Secure e, per la maggior parte dei fornitori, un Verified Mark Certificate a pagamento. BIMI non è quindi un meccanismo di sicurezza, ma una questione di visibilità e deve venire alla fine della sequenza, non all'inizio.

## Altri record correlati

**Autodiscover e SRV:** gli ambienti Exchange utilizzano `autodiscover.example.com` come CNAME oppure un record SRV `_autodiscover._tcp.example.com`. Entrambi riguardano la configurazione del client e non il flusso di posta, ma vengono spesso trascurati durante le migrazioni, causando profili che non possono più essere configurati.

**CAA:** non riguarda direttamente la posta, ma determina quale autorità di certificazione può emettere un certificato per `mta-sts.example.com` o per il nome del server e-mail.

**Zone split-horizon:** dove una zona DNS interna porta lo stesso nome di quella pubblica, spesso i record e-mail non esistono internamente. I sistemi interni che eseguono una verifica SPF o DKIM giungono allora a risultati diversi rispetto all'esterno. A ogni modifica dei record e-mail occorre quindi chiedersi se la zona interna debba essere aggiornata.

## Alcuni test rapidi

Eseguite volutamente tutte le query contro un resolver pubblico, affinché non risponda la cache interna o una zona split-horizon:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

Contro il server autorevole, per bypassare completamente la cache:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

In Windows senza `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

Per la valutazione completa, incluso il conteggio dei lookup SPF, la ricerca del selettore DKIM e il controllo dell'allineamento, questa pagina offre il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), che verifica un dominio in un unico passaggio rispetto a tutti i record descritti qui.

Il test più significativo resta tuttavia un messaggio reale. Inviate un'e-mail a una casella presso un grande fornitore e osservate la riga `Authentication-Results` nell'intestazione. Mostra in una riga i risultati effettivi di SPF, DKIM e DMARC e sostituisce qualsiasi teoria sul file di zona.

## Ordine durante una migrazione

Quando si cambia provider e-mail, questa sequenza si è dimostrata efficace:

1. Ridurre a 300 secondi il TTL di tutti i record interessati, almeno un giorno prima.
2. Pubblicare i selettori DKIM del nuovo provider mentre quelli vecchi sono ancora presenti.
3. Estendere SPF con il nuovo provider senza rimuovere quello vecchio e ricalcolare il limite di lookup.
4. Per MTA-STS, adattare la policy ai nuovi nomi MX e aumentare l'`id` prima di cambiare i record MX.
5. Cambiare gli MX e osservare la consegna.
6. Solo dopo alcuni giorni senza anomalie rimuovere i vecchi include SPF e selettori DKIM.
7. Ripristinare il TTL.

Il problema più frequente in questa sequenza è anticipare troppo il passo 6: i vecchi record vengono eliminati insieme al cambio e tutto ciò che passa ancora dal percorso precedente fallisce la verifica del mittente.

## Conclusione

I record e-mail si distinguono da tutte le altre voci DNS perché un errore non è necessariamente evidente. Un record A errato genera un ticket nel giro di pochi minuti; un record SPF duplicato o una chiave DKIM con un carattere di troppo, invece, comportano un tasso di consegna che cala lentamente per settimane.

Tre regole evitano la maggior parte di questi casi. Primo: prima di ogni nuovo record, verificare ciò che esiste già invece di affiancarne un secondo. Secondo: dopo ogni modifica, controllare tramite un resolver pubblico e confrontare il valore carattere per carattere con il modello, non solo visivamente. Terzo: nelle modifiche, pubblicare sempre prima il nuovo, poi effettuare il cambio e infine rimuovere il vecchio. Chi rispetta questa sequenza ha sempre una via di ritorno per i record e-mail.

## Fonti

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definisce, tra l'altro, il limite di 255 byte di una singola `character-string` nei record TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Stabilisce nella sezione 10.3 che la destinazione di un record MX non può essere un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Limite di dieci meccanismi di lookup, limite dei void lookup, eliminazione del tipo RR SPF e sconsigliato il meccanismo `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Struttura del record della chiave in `_domainkey`, significato del selettore e del `p=` vuoto.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Specifica DMARC attuale del maggio 2026, sostituisce RFC 7489; eliminazione di `pct`, nuovo tag `np`, Tree Walk al posto della Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Formato e consegna dei report aggregati, inclusa l'autorizzazione dei domini destinatari esterni.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identificazione dei domini che non accettano posta.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Record DNS, file di policy, significato dell'`id` e delle modalità `testing` e `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Struttura del record `_smtp._tls` e dei report.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Record TLSA per SMTP e requisito di una zona firmata con DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Stato attuale della specifica BIMI, ancora non un RFC.

12.  [Google: Linee guida per mittenti e-mail](https://support.google.com/a/answer/81126): Requisiti per i mittenti, inclusi l'obbligo di PTR per gli indirizzi IPv6 in invio e le disposizioni per mittenti di grandi volumi in vigore da febbraio 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requisiti per mittenti con almeno 5000 messaggi al giorno, in vigore da maggio 2025.
