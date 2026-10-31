---
title: "Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le fonti di errore più comuni"
navTitle: "Record DNS e-mail"
description: "Chi gestisce una zona riceve di solito i record e-mail già pronti e deve solo pubblicarli. Cosa va regolarmente storto: il limite di 255 byte per DKIM, record SPF duplicati, il limite di lookup, MX su un CNAME, il suffisso di zona aggiunto automaticamente e policy che nessuno applica più."
date: "2026-08-04"
kategorie: "SMTP e flusso di posta"
timeToRead: "15 min di lettura"
themen:
  - smtp-mailflow
  - e-mail-verschluesselung
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
translationOf: dns-records-e-mail-stolpersteine
translationSourceHash: e78b63f0179944cf4c36caf7aabfb9ceb0c5dd40d2d3cf1a25e92fee8cbacaca
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:42:18.385Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/guida-per-amministratori-dns-mx-spf-dkim-dmarc-e-le-insidie-piu-comuni
---

# Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le fonti di errore più comuni

Chi gestisce una zona DNS raramente riceve record e-mail scritti da sé. Il team e-mail, un provider o un fornitore di servizi di marketing invia una riga con l'indicazione che deve "solo essere pubblicata". È proprio da qui che nascono la maggior parte degli errori, perché i record e-mail sono il tipo di record in cui un refuso può avere due conseguenze completamente diverse. O la consegna si interrompe subito e qualcuno segnala il problema nel giro di pochi minuti, oppure continua invariata e fallisce silenziosamente solo la verifica del mittente. Il secondo caso passa regolarmente inosservato per mesi, finché un grande destinatario non mette il dominio in quarantena.

Da quando Google e Yahoo hanno inasprito i requisiti per i mittenti di grandi volumi nel febbraio 2024 e Microsoft li ha seguiti nel maggio 2025, la tolleranza per i domini configurati a metà è diventata ridotta. SPF, DKIM e un record DMARC non sono più un optional per i mittenti oltre un certo volume, ma un requisito per la consegna.

Tutti gli esempi di questo articolo utilizzano `example.com` e selettori generici. I valori mostrati sono abbreviati per mantenerli leggibili.

## Regole valide per ogni record e-mail

### Il limite di 255 byte per i record TXT

Secondo RFC 1035, un record TXT è costituito da una o più `character-strings`, e ciascuna singola stringa può contenere al massimo 255 byte. Il record nel suo complesso può essere più lungo, ma deve allora essere suddiviso in più stringhe. I sistemi di valutazione riuniscono nuovamente queste parti senza separatori.

Questo diventa rilevante nella pratica esattamente in un punto: per le chiavi DKIM da 2048 bit. Il loro valore Base64 è lungo circa 400 caratteri e non entra in una sola stringa.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La maggior parte dei sistemi di gestione DNS esegue autonomamente questa suddivisione quando il valore viene inserito nel normale campo di immissione. Chi invece aggiunge manualmente le virgolette deve rispettare esattamente il limite. Un valore spezzato con uno spazio nel punto di giunzione produce una chiave che esiste sintatticamente ma non è più valida dal punto di vista crittografico.

Il controllo successivo è importante, perché una chiave composta in modo errato appare del tutto normale nella GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Restituisce solo i valori dei record, senza intestazioni e metadati |
| `TXT selector1._domainkey.example.com` | Tipo di record e nome del record della chiave DKIM |
| `tr -d '" '` | Rimuove virgolette e spazi, ricomponendo le stringhe parziali come le legge un verificatore |
| `wc -c` | Conta i caratteri del valore ricomposto; la lunghezza deve corrispondere al modello |

</details>

### Un record per scopo

SPF e DMARC sono definiti in modo che a un nome possa corrispondere esattamente un record appropriato. Per SPF, due record `v=spf1` producono un `permerror`, quindi il controllo è considerato fallito e non superato. Con DMARC, i destinatari ignorano completamente il dominio quando più record iniziano con `v=DMARC1`: invece di una policy rigorosa, non viene applicata alcuna policy.

Questo è di gran lunga l'errore più comune nelle zone cresciute nel tempo. Viene collegato un nuovo fornitore di servizi, qualcuno aggiunge il "proprio" record SPF invece di ampliare quello esistente e, da quel momento, la verifica fallisce per tutti i mittenti. Prima di ogni nuovo record occorre quindi controllare obbligatoriamente ciò che esiste già:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Restituisce solo i valori dei record, senza intestazioni e metadati |
| `TXT` | Tipo di record interrogato |
| `example.com`, `_dmarc.example.com` | Nomi interrogati: il dominio stesso per SPF, il nome `_dmarc` per DMARC |
| `grep -i spf1` | Filtra la riga SPF; `-i` ignora maiuscole e minuscole |

</details>

Per DKIM vale il contrario: è previsto un record per selettore e avere più selettori affiancati è normale, perché ogni servizio di invio utilizza la propria chiave.

### Il suffisso di zona nelle interfacce web

In Infoblox, nel DNS di Windows e praticamente in tutte le interfacce di hosting, il nome della zona viene aggiunto automaticamente al nome inserito. Chi inserisce il nome pienamente qualificato nel campo "Nome" ottiene un record lungo il doppio del previsto:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Nel file di zona, l'equivalente è il punto finale mancante. `mail.example.com` senza punto finale è un nome relativo e viene completato con il nome della zona, mentre `mail.example.com.` con il punto è assoluto. Per le destinazioni MX e CNAME, questo singolo punto determina se il dominio è raggiungibile.

### Il copia-incolla è la fonte di errore più comune

I valori dei record e-mail non vengono quasi mai digitati, ma copiati da un PDF, un ticket, una cella Excel o una chat. Così si producono danni che restano invisibili nel campo di inserimento:

- Un doppio `p=` all'inizio della chiave DKIM, perché il prefisso è stato impostato due volte durante la composizione. Il valore `v=DKIM1;k=rsa;p=p=MIIBIjAN...` si presenta regolarmente nella pratica e produce una chiave inutilizzabile.
- Virgolette tipografiche di Word invece di quelle dritte.
- Spazi non separabili provenienti da layout PDF, indistinguibili da quelli normali.
- Interruzioni di riga nel blocco Base64 se il valore nel PDF occupava più righe.

Base64 conosce esattamente i caratteri da A a Z, da a a z, da 0 a 9, `+`, `/` e `=` come carattere di riempimento. Tutto il resto nella parte `p=` è un errore. Un breve filtro prima dell'inserimento evita successive ricerche del guasto:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `'%s' "$KEY"` | Emette il valore della chiave invariato e senza un'interruzione di riga finale |
| `tr -d 'A-Za-z0-9+/='` | Rimuove tutti i caratteri validi per Base64; restano solo i caratteri estranei |
| `wc -c` | Conta i caratteri rimanenti |

</details>

Se qui compare qualcosa di diverso da `0`, la chiave contiene caratteri estranei.

### Ridurre il TTL prima delle modifiche

Prima di ogni modifica pianificata di un record MX, SPF o DKIM, il TTL va impostato per alcune ore su un valore basso, tipicamente 300 secondi. Altrimenti, a seconda della zona, il vecchio valore resta nei resolver esterni ancora per un giorno o più, e anche un rollback richiede altrettanto tempo. Dopo la modifica e un periodo di osservazione, il TTL viene ripristinato al valore regolare.

## MX

Il record MX determina quale host riceve le e-mail per il dominio. Ci sono due regole che vengono regolarmente violate.

**La destinazione deve essere un hostname con record A o AAAA.** Non sono ammessi né un indirizzo IP né un CNAME. RFC 2181 stabilisce esplicitamente che la destinazione di un record MX non può essere un alias. Nella pratica funziona comunque con molti destinatari, ma non con altri, causando problemi che sembrano riguardare solo singoli mittenti.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Il numero è una preferenza, non una ponderazione.** Viene tentato prima il valore più basso. Un secondo MX con un numero elevato ha senso solo se quel sistema conosce lo stesso filtro destinatari. Le voci MX di backup su sistemi senza verifica dei destinatari sono un obiettivo popolare per lo spam, poiché gli aggressori puntano deliberatamente alla voce più debole.

I domini che inviano esclusivamente o non hanno nulla a che fare con la posta ricevono un Null MX secondo RFC 7505. Segnala che il dominio non accetta posta e garantisce un rifiuto immediato e inequivocabile anziché timeout:

```text
example.com.  IN  MX  0 .
```

Tuttavia, il Null MX non sostituisce un record SPF e DMARC. Non ricevere posta non significa che nessuno invii a vostro nome. Proprio i sottodomini parcheggiati vengono usati per lo spoofing, perché raramente qualcuno li controlla.

## A, AAAA, PTR e il nome HELO

Il record PTR per l'indirizzo IP in uscita non si trova nella vostra zona, bensì nella zona `in-addr.arpa` del provider a cui appartiene il blocco di indirizzi. Per questo va richiesto al provider e non impostato autonomamente. Molti grandi destinatari richiedono che PTR e il relativo record diretto coincidano, cioè che il nome del PTR risolva nuovamente allo stesso indirizzo IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Restituisce solo i valori dei record, senza intestazioni e metadati |
| `-x 192.0.2.10` | Query inversa: dig costruisce autonomamente il nome PTR nella zona `in-addr.arpa` |
| `A mail1.example.com` | Query diretta del nome dal PTR, per verificare che il percorso di ritorno conduca allo stesso indirizzo IP |

</details>

Anche il nome che il vostro server e-mail comunica in HELO o EHLO dovrebbe essere lo stesso e risolvibile. Un gateway che si presenta come `localhost.localdomain` o con un nome interno riceve una valutazione peggiore dai destinatari più grandi.

Occorre prestare attenzione quando si aggiunge un record AAAA. Non appena il server e-mail è raggiungibile e invia tramite IPv6, si applicano gli stessi requisiti di IPv4, in parte persino più severi. Google richiede un PTR valido per gli indirizzi IPv6 che inviano. Se manca, l'invio viene rifiutato, mentre tramite IPv4 funzionava perfettamente. Un record AAAA sul server e-mail non è quindi mai una pura modifica DNS.

## SPF

SPF stabilisce quali sistemi possono inviare a nome del dominio. Il record si trova come TXT sul dominio stesso.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Il limite di lookup

La valutazione di un record SPF può attivare al massimo dieci meccanismi che effettuano query DNS. Vengono conteggiati `include`, `a`, `mx`, `ptr`, `exists` e `redirect`, anche ricorsivamente: ogni `include` porta con sé i lookup del record incluso. Non vengono conteggiati `ip4`, `ip6` e `all`.

Se il limite viene superato, il risultato è un `permerror`. Per DMARC, questo significa SPF non superato, indipendentemente dal fatto che il server mittente sarebbe effettivamente autorizzato. L'aspetto insidioso è che l'errore spesso si verifica senza un vostro intervento, perché un provider incluso amplia il proprio record. Il record locale non è cambiato, ma la consegna subisce comunque un calo.

Inoltre, sono consentiti al massimo due "void lookup", cioè query senza risultato. Un `include` verso un dominio che non esiste più rientra in questo conteggio. I riferimenti a fornitori di servizi dismessi devono quindi essere rimossi, non mantenuti per precauzione.

### Cosa non deve figurare in un record SPF

- **`ptr`** è sì specificato, ma è considerato obsoleto da RFC 7208 e non va utilizzato. I sistemi di valutazione possono ignorarlo.
- **`+all`** autorizza qualsiasi mittente ed è quindi più dannoso dell'assenza totale di un record SPF.
- **`?all`** è neutro e quindi praticamente inutile per DMARC.
- **Un record separato di tipo SPF (tipo 99)** non è più necessario. È stato abolito con RFC 7208; SPF risiede esclusivamente in TXT.

Tra `~all` (softfail) e `-all` (hardfail) decide quanto completamente siano stati rilevati i percorsi di invio. Finché vi sono dubbi, `~all` è la scelta corretta. Chi applica già DMARC e valuta i report può passare a `-all`.

### I sottodomini non ereditano nulla

Un record SPF su `example.com` non vale per `newsletter.example.com`. Ogni sottodominio mittente necessita del proprio record. Per tutti gli altri è consigliabile una voce wildcard che chiarisca che da lì non proviene nulla:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attenzione: una wildcard TXT risponde anche alle query per nomi come `_dmarc.sub.example.com`, se lì non è presente un record esplicito. Di solito non è problematico, ma può confondere la ricerca del guasto, perché ogni query TXT riceve una risposta.

### SPF flattening

Gli strumenti che risolvono tutti i riferimenti `include` e li sostituiscono con gli indirizzi IP sottostanti eliminano il limite di lookup a scapito della manutenibilità. Se il provider modifica i propri indirizzi, l'invio si interrompe e nessuno se ne accorge, perché nel record locale tutto sembra corretto. Chi sceglie questa strada necessita quindi di un confronto automatizzato che verifichi regolarmente l'elenco rispetto alla fonte. Come lavoro manuale una tantum, prima o poi il metodo fallisce.

## DKIM

DKIM firma i messaggi in uscita. La chiave pubblica si trova sotto `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Il selettore è liberamente scegliibile ed è specificato dal sistema mittente. Un nome descrittivo con data facilita molto di più la rotazione successiva rispetto a `s1` e `s2`.

### Delega tramite CNAME

Dove il servizio di invio lo offre, la variante CNAME è preferibile all'inserimento diretto:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Il provider può quindi ruotare autonomamente la propria chiave senza che qualcuno debba intervenire nella vostra zona. Altrimenti, proprio questa rotazione viene regolarmente trascurata, perché richiede coordinamento tra due team. Un CNAME esclude però qualsiasi ulteriore record con lo stesso nome: è una regola fondamentale del DNS, non una particolarità di DKIM.

### Rotazione senza interruzioni

Nel cambio della chiave, il nuovo selettore viene prima pubblicato, poi il server mittente viene commutato su di esso e solo dopo viene rimosso il vecchio record. Chi elimina subito la vecchia chiave invalida le firme di tutti i messaggi ancora in transito o nelle code e rende impossibili le verifiche successive. Un intervallo di alcuni giorni tra la commutazione e l'eliminazione è adeguato.

Un record con `p=` vuoto non è una voce difettosa, ma il modo specificato per contrassegnare una chiave come ritirata.

### Lunghezza della chiave

1024 bit sono considerati obsoleti, 2048 bit sono lo standard. Chiavi RSA più grandi non apportano alcun vantaggio pratico e aumentano solo la probabilità che un sistema intermedio non elabori correttamente il record.

## DMARC

DMARC collega SPF e DKIM a un'istruzione su cosa debba accadere in caso di verifica fallita e restituisce report. Il record si trova sotto `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Dal maggio 2026, con RFC 9989 e le specifiche sui report RFC 9990 e RFC 9991, è in vigore la versione rivista che sostituisce RFC 7489. Nella pratica sono importanti tre modifiche:

- **`pct` è stato eliminato.** Non esiste più l'introduzione graduale tramite una percentuale. Al suo posto c'è `t=y`, che contrassegna il dominio come in modalità di test: i report continuano a funzionare, ma la policy non deve essere applicata.
- **`np` è nuovo.** Imposta la policy per i sottodomini inesistenti, colmando così una lacuna spesso sfruttata dagli aggressori, poiché finora i sottodomini inventati erano coperti solo da `sp`. Senza un'indicazione propria, `np` segue il valore di `sp`.
- **La Public Suffix List è stata sostituita da un `Tree Walk`.** Il dominio organizzativo non viene più determinato da un elenco esterno gestito, bensì tramite query DNS a livelli lungo l'albero dei nomi. Per grandi spazi dei nomi con molti livelli, questo modifica sensibilmente la valutazione.

### L'allineamento è il vero nucleo

DMARC non passa perché SPF o DKIM sono tecnicamente superati, ma solo se almeno uno dei due corrisponde inoltre al dominio del mittente visibile nell'header `From`. SPF viene verificato rispetto al dominio del mittente dell'envelope, che si discosta regolarmente in caso di inoltri, servizi newsletter e sistemi di ticket. Proprio per questo messaggi con SPF valido talvolta non superano il controllo DMARC.

Con `adkim=r` e `aspf=r` (relaxed, lo standard), è sufficiente la corrispondenza a livello di dominio organizzativo. `s` richiede l'uguaglianza esatta incluso il sottodominio e nella pratica fallisce quasi sempre per uno dei percorsi di invio.

### Gli indirizzi di report esterni richiedono autorizzazione

Se i report devono essere inviati a un indirizzo esterno al proprio dominio, ad esempio a un servizio di valutazione DMARC, il dominio ricevente deve autorizzarlo. Senza questo record, molti destinatari semplicemente non inviano nulla e la valutazione resta vuota, mentre nel proprio record tutto sembra corretto:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Questa voce viene creata dal gestore della zona di destinazione, non da voi. Nei servizi commerciali avviene automaticamente, ma non nel caso di una casella di raccolta gestita autonomamente in un altro dominio di vostra proprietà.

### Errori di sintassi tipici

I nomi dei tag e i valori delle policy devono essere in minuscolo; `p=Reject` non è valido. Tra i tag è presente un punto e virgola; un separatore mancante rende inefficace il resto della riga. Inoltre, `p` deve essere il primo tag dopo `v`. Un record composto solo da `v=DMARC1; rua=...` non contiene alcuna policy ed è incompleto.

### Il rollout

`p=none` è uno stato di misurazione, non un obiettivo. Non modifica il comportamento dei destinatari nei confronti delle vostre e-mail e serve esclusivamente a individuare tramite i report tutti i percorsi di invio legittimi. Chi dopo l'introduzione non passa entro pochi mesi tramite `quarantine` a `reject` ha sostenuto l'impegno senza ottenere la protezione. L'aspetto organizzativo di questo percorso, inclusa la documentazione per la decisione, è un argomento a sé ed è descritto nel blueprint DMARC.

## MTA-STS e TLS-RPT

SMTP cifra in modo opportunistico: se la controparte offre STARTTLS, viene usata la cifratura, altrimenti no. Un aggressore in grado di manipolare il traffico può rimuovere l'annuncio STARTTLS e mantenere così la connessione in chiaro. MTA-STS colma questa lacuna per i domini riceventi.

MTA-STS è composto da due parti, e solo una di esse si trova nel DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La policy vera e propria si trova come file sotto `https://mta-sts.example.com/.well-known/mta-sts.txt` e deve essere distribuita tramite un certificato valido:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Le fonti di errore sono quasi tutte al di fuori della zona:

- **L'`id` deve cambiare a ogni modifica della policy.** È l'unica indicazione per i sistemi mittenti che devono recuperare una nuova policy. Chi modifica il file e lascia invariato l'`id` lavora contro copie memorizzate nella cache fino alla scadenza di `max_age`.
- **L'elenco MX nella policy e i record MX devono corrispondere.** Un nuovo MX assente dalla policy viene rifiutato dai mittenti con `mode: enforce`. In caso di migrazioni, la policy va quindi adattata prima della modifica MX.
- **Prima `mode: testing`.** In questa modalità, le violazioni vengono solo segnalate, non applicate. Il passaggio a `enforce` avviene quando i report sono puliti.
- **Un record CAA può bloccare l'emissione del certificato per l'host della policy** se vi è indicata un'autorità di certificazione diversa da quella utilizzata.

TLS-RPT fornisce i report corrispondenti ed è un singolo record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT è utile anche senza MTA-STS, perché rende visibile la cifratura del trasporto non riuscita.

## DANE

DANE raggiunge lo stesso obiettivo di MTA-STS, ma ancora la fiducia nel DNS anziché nella Web PKI. Richiede una zona firmata con DNSSEC dall'inizio alla fine e, senza DNSSEC, un record TLSA è inefficace.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Aspetto decisivo durante l'esercizio: a ogni cambio di certificato, il record TLSA deve essere corretto in anticipo. La procedura consueta pubblica il nuovo hash in parallelo a quello vecchio, poi cambia il certificato e infine rimuove la vecchia voce. Chi inverte questo ordine rende il server e-mail irraggiungibile per tutti i mittenti che verificano DANE, tra cui i grandi provider di lingua tedesca. In Svizzera DANE è molto meno diffuso di MTA-STS, generalmente a causa della mancanza della firma DNSSEC della zona.

## BIMI

BIMI mostra il logo del marchio nella casella di posta ed è l'unico meccanismo qui trattato che non è ancora un RFC, ma continua a essere gestito come Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

I requisiti sono elevati: una policy DMARC applicata con `quarantine` o `reject`, un logo in formato SVG Tiny Portable/Secure e, per la maggior parte dei fornitori, un Verified Mark Certificate a pagamento. BIMI non è quindi un meccanismo di sicurezza, ma una questione di visibilità e deve essere posto alla fine della sequenza, non all'inizio.

## Altri record correlati

**Autodiscover e SRV:** Gli ambienti Exchange utilizzano `autodiscover.example.com` come CNAME o un record SRV `_autodiscover._tcp.example.com`. Entrambi riguardano la configurazione dei client e non il flusso della posta, ma vengono facilmente trascurati durante una migrazione e causano poi profili che non possono più essere configurati.

**CAA:** Non ha direttamente a che fare con la posta, ma determina quale autorità di certificazione può emettere un certificato per `mta-sts.example.com` o per il nome del server e-mail.

**Zone split-horizon:** Dove una zona DNS interna porta lo stesso nome di quella pubblica, spesso i record e-mail non esistono internamente. I sistemi interni che eseguono una verifica SPF o DKIM giungono così a risultati diversi dal mondo esterno. Per ogni modifica ai record e-mail occorre quindi chiedersi se anche la zona interna debba essere aggiornata.

## Alcuni test rapidi

Effettuare deliberatamente tutte le query verso un resolver pubblico, affinché non risponda la cache interna o una zona split-horizon:

```bash
dig @1.1.1.1 +short MX example.com
dig @1.1.1.1 +short TXT example.com
dig @1.1.1.1 +short TXT _dmarc.example.com
dig @1.1.1.1 +short TXT selector1._domainkey.example.com
dig @1.1.1.1 +short TXT _mta-sts.example.com
dig @1.1.1.1 +short TXT _smtp._tls.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `@1.1.1.1` | Invia la query a questo resolver anziché a quello configurato in `/etc/resolv.conf` |
| `+short` | Restituisce solo i valori dei record, senza intestazioni e metadati |
| `MX`, `TXT` | Tipi di record interrogati |
| `_dmarc.…`, `selector1._domainkey.…`, `_mta-sts.…`, `_smtp._tls.…` | I nomi definiti sotto il dominio per DMARC, DKIM, MTA-STS e TLS-RPT |

</details>

Contro il server autorevole, per aggirare completamente la cache:

```bash
dig +short NS example.com
dig @ns1.example.com +norecurse TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `NS example.com` | Determina i nameserver autorevoli della zona |
| `@ns1.example.com` | Invia la query successiva direttamente a uno di questi server autorevoli |
| `+norecurse` | Non imposta il bit Recursion Desired; il server risponde solo dai propri dati di zona, non da una cache |

</details>

In Windows senza `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-type=TXT` | Tipo di record da interrogare |
| `_dmarc.example.com` | Nome interrogato |
| `1.1.1.1` | Resolver da usare anziché quello configurato a livello di sistema |

</details>

Per una valutazione completa, incluso il conteggio dei lookup SPF, la ricerca del selettore DKIM e la verifica dell'allineamento, in questa pagina è disponibile il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), che verifica un dominio in un unico passaggio rispetto a tutti i record descritti qui.

Tuttavia, il test più significativo resta un messaggio reale. Inviate un'e-mail a una casella presso un grande fornitore e osservate la riga `Authentication-Results` nell'intestazione. Mostra in una riga ciò che SPF, DKIM e DMARC hanno effettivamente prodotto e sostituisce ogni teoria sul file di zona.

## Sequenza durante una migrazione

Quando si cambia fornitore e-mail, questa sequenza si è dimostrata efficace:

1. Ridurre il TTL di tutti i record interessati a 300 secondi, almeno un giorno prima.
2. Pubblicare i selettori DKIM del nuovo fornitore mentre quelli vecchi sono ancora presenti.
3. Estendere SPF con il nuovo fornitore senza rimuovere quello vecchio, ricalcolando il limite di lookup.
4. Per MTA-STS, adattare la policy ai nuovi nomi MX e aumentare l'`id` prima che cambino i record MX.
5. Commutare gli MX e osservare la consegna.
6. Solo dopo alcuni giorni senza contestazioni, rimuovere i vecchi include SPF e selettori DKIM.
7. Ripristinare il TTL.

Il problema più comune in questa sequenza è anticipare troppo il passaggio 6: le vecchie voci vengono eliminate insieme alla commutazione e tutto ciò che passa ancora dal percorso precedente fallisce la verifica del mittente.

## Conclusione

I record e-mail si differenziano da tutte le altre voci DNS perché un errore non si nota necessariamente. Un record A errato genera un ticket entro pochi minuti; un record SPF duplicato o una chiave DKIM con un carattere in più, invece, causano un tasso di consegna che diminuisce lentamente nel corso delle settimane.

Tre regole evitano la maggior parte di questi casi. Primo: prima di ogni nuovo record, verificare ciò che esiste già invece di affiancarne un secondo. Secondo: dopo ogni modifica, controllare contro un resolver pubblico e confrontare il valore carattere per carattere con il modello, non solo visivamente. Terzo: durante le modifiche pubblicare sempre prima il nuovo, poi commutare e infine rimuovere il vecchio. Chi rispetta questa sequenza ha sempre una via di ritorno per i record e-mail.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiornato il link alla nuova pagina di Microsoft.

## Fonti

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definisce tra l'altro il limite di 255 byte di una singola `character-string` nei record TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Stabilisce nella sezione 10.3 che la destinazione di un record MX non può essere un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Limite di lookup di dieci meccanismi, limite dei void lookup, abolizione del tipo RR SPF e sconsiglio del meccanismo `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Struttura del record della chiave sotto `_domainkey`, significato del selettore e del `p=` vuoto.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Specifica DMARC attuale del maggio 2026, sostituisce RFC 7489; eliminazione di `pct`, nuovo tag `np`, Tree Walk anziché Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Formato e consegna dei report aggregati, inclusa l'autorizzazione dei domini destinatari esterni.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Identificazione dei domini che non accettano posta.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Record DNS, file di policy, significato dell'`id` e delle modalità `testing` e `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Struttura del record `_smtp._tls` e dei report.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Record TLSA per SMTP e requisito di una zona firmata DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Stato attuale della specifica BIMI, ancora non un RFC.

12.  [Google: Linee guida per i mittenti e-mail](https://support.google.com/a/answer/81126): Requisiti per i mittenti, tra cui l'obbligo PTR per gli indirizzi IPv6 che inviano e le disposizioni per mittenti di grandi volumi in vigore dal febbraio 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoftdefenderforoffice365blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399730): Requisiti per mittenti a partire da 5000 messaggi al giorno, validi dal maggio 2025.
