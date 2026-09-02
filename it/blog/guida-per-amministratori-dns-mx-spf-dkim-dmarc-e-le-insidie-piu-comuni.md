---
title: "Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le consuete fonti di errore"
navTitle: "Record DNS per l’e-mail"
description: "Chi gestisce una zona riceve di solito i record e-mail già pronti e deve solo pubblicarli. Cosa va regolarmente storto: il limite di 255 byte per DKIM, record SPF duplicati, il limite di lookup, un MX su un CNAME, il suffisso di zona aggiunto automaticamente e policy che nessuno applica più."
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
translationSourceHash: 63c8a888f2ebd4548bd4222c4273896228649bf02f0406082ec337194af65280
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:06:21.607Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/guida-per-amministratori-dns-mx-spf-dkim-dmarc-e-le-insidie-piu-comuni
---

# Guida per amministratori DNS: MX, SPF, DKIM, DMARC e le consuete fonti di errore

Chi gestisce una zona DNS raramente riceve record e-mail scritti da sé. Il team e-mail, un provider o un fornitore di servizi di marketing invia una riga con l’indicazione che deve «solo essere pubblicata». È proprio qui che nascono la maggior parte degli errori, perché i record e-mail sono il tipo di record in cui un refuso può avere due conseguenze completamente diverse. O la consegna si interrompe subito e qualcuno segnala il problema nel giro di pochi minuti, oppure continua invariata e fallisce silenziosamente solo la verifica del mittente. Il secondo caso passa regolarmente inosservato per mesi, finché un grande destinatario non mette il dominio in quarantena.

Da quando Google e Yahoo hanno inasprito i requisiti per i mittenti di grandi volumi nel febbraio 2024 e Microsoft ha seguito l’esempio nel maggio 2025, la tolleranza per domini configurati a metà è diventata ridotta. SPF, DKIM e un record DMARC non sono più un optional per i mittenti oltre un certo volume, ma un requisito per la consegna.

Tutti gli esempi di questo articolo usano `example.com` e selettori generici. I valori mostrati sono abbreviati per mantenerli leggibili.

## Regole valide per ogni record e-mail

### Il limite di 255 byte per i record TXT

Secondo RFC 1035, un record TXT è composto da una o più `character-strings`, e una singola stringa di questo tipo può contenere al massimo 255 byte. Il record nel suo insieme può essere più lungo, ma deve allora essere suddiviso in più stringhe. I sistemi di valutazione ricompongono queste parti senza separatori.

Questo diventa rilevante nella pratica esattamente in un caso: le chiavi DKIM a 2048 bit. Il loro valore Base64 è lungo circa 400 caratteri e non entra in una sola stringa.

```text
selector1._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0ZenWBnGUqydzg5w"
    "yWxRRNBZjbagzDh1BlW3b145Wer/GWfbz6XkCyqsN918N+/Va6mVe37rXNaZAS"
    "/js/L3m7d2OlUp5I3jHC5EsU6XwU5trKFxPWErLxtanYbXLXabyVIRGkop+1s3"
    "SXNg2Oy5eNyZf5MGlEAo+JM6oXtgkABQQn5kE1ShzalXJUVL/wIDAQAB" )
```

La maggior parte dei sistemi di gestione DNS effettua questa suddivisione autonomamente se il valore viene inserito tramite il normale campo di immissione. Chi invece aggiunge manualmente le virgolette deve rispettare esattamente il limite. Un valore spezzato con uno spazio nel punto di giunzione produce una chiave che esiste sintatticamente ma non è più valida dal punto di vista crittografico.

Il controllo successivo è importante, perché una chiave composta in modo errato appare del tutto normale nella GUI:

```bash
dig +short TXT selector1._domainkey.example.com | tr -d '" ' | wc -c
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Mostra solo i valori dei record, senza intestazioni e metadati |
| `TXT selector1._domainkey.example.com` | Tipo di record e nome del record della chiave DKIM |
| `tr -d '" '` | Rimuove virgolette e spazi, ricomponendo le stringhe parziali come le legge un verificatore |
| `wc -c` | Conta i caratteri del valore ricomposto; la lunghezza deve corrispondere al modello |

</details>

### Un record per scopo

SPF e DMARC sono definiti in modo che per un nome possa esistere esattamente un record appropriato. Per SPF, due record `v=spf1` causano un `permerror`, e la verifica è quindi considerata fallita, non superata. Per DMARC, i destinatari ignorano completamente il dominio se più record iniziano con `v=DMARC1`: invece di una policy rigorosa, non si applica alcuna policy.

Questo è di gran lunga l’errore più frequente nelle zone cresciute nel tempo. Si collega un nuovo fornitore di servizi, qualcuno aggiunge «il proprio» record SPF invece di estendere quello esistente e, da quel momento, la verifica fallisce per tutti i mittenti. Prima di ogni nuovo record è quindi indispensabile controllare ciò che esiste già:

```bash
dig +short TXT example.com | grep -i spf1
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Mostra solo i valori dei record, senza intestazioni e metadati |
| `TXT` | Tipo di record interrogato |
| `example.com`, `_dmarc.example.com` | Nomi interrogati: il dominio stesso per SPF, il nome `_dmarc` per DMARC |
| `grep -i spf1` | Filtra la riga SPF; `-i` ignora maiuscole e minuscole |

</details>

Per DKIM vale il contrario: è previsto un record per selettore, e più selettori affiancati sono la norma, perché ogni servizio di invio porta la propria chiave.

### Il suffisso di zona nelle interfacce web

In Infoblox, in Windows DNS e in quasi tutte le interfacce di hosting, il nome della zona viene aggiunto automaticamente al nome inserito. Chi inserisce il nome pienamente qualificato nel campo «Nome» ottiene un record lungo il doppio del previsto:

```text
Eingabe:   selector1._domainkey.example.com
Ergebnis:  selector1._domainkey.example.com.example.com
```

Nel file di zona, il corrispettivo è il punto finale mancante. `mail.example.com` senza punto finale è un nome relativo e viene integrato con il nome della zona; `mail.example.com.` con il punto è assoluto. Per le destinazioni MX e CNAME, questo singolo punto decide se il dominio è raggiungibile.

### Il copia-incolla è la fonte di errore più frequente

I valori dei record e-mail non vengono quasi mai digitati, ma copiati da un PDF, un ticket, una cella Excel o una chat. Questo può causare danni che restano invisibili nel campo di immissione:

- Un `p=` duplicato all’inizio della chiave DKIM, perché il prefisso è stato impostato due volte durante la composizione. Il valore `v=DKIM1;k=rsa;p=p=MIIBIjAN...` si verifica regolarmente nella pratica e produce una chiave inutilizzabile.
- Virgolette tipografiche di Word al posto di quelle dritte.
- Spazi non separabili provenienti dai layout PDF, che sembrano normali.
- Interruzioni di riga nel mezzo del blocco Base64, se il valore nel PDF era distribuito su più righe.

Base64 conosce esattamente i caratteri da A a Z, da a a z, da 0 a 9, `+`, `/` e `=` come carattere di riempimento. Tutto il resto nella parte `p=` è un errore. Un breve filtro prima dell’inserimento evita successive ricerche del problema:

```bash
printf '%s' "$KEY" | tr -d 'A-Za-z0-9+/=' | wc -c
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `'%s' "$KEY"` | Mostra il valore della chiave invariato e senza un’interruzione di riga aggiunta |
| `tr -d 'A-Za-z0-9+/='` | Rimuove tutti i caratteri validi per Base64; rimangono solo caratteri estranei |
| `wc -c` | Conta i caratteri rimanenti |

</details>

Se qui compare qualcosa di diverso da `0`, la chiave contiene caratteri estranei.

### Ridurre il TTL prima delle modifiche

Prima di ogni modifica pianificata di un record MX, SPF o DKIM, il TTL va impostato per alcune ore su un valore basso, tipicamente 300 secondi. Altrimenti, a seconda della zona, il vecchio valore rimane nei resolver esterni per un giorno o più, e un rollback richiede lo stesso tempo. Dopo la modifica e una fase di osservazione, il TTL viene ripristinato al valore normale.

## MX

Il record MX stabilisce quale host accetta e-mail per il dominio. Vi sono due regole che vengono regolarmente violate.

**La destinazione deve essere un hostname con record A o AAAA.** Non sono ammessi né un indirizzo IP né un CNAME. RFC 2181 stabilisce espressamente che la destinazione di un record MX non può essere un alias. Nella pratica funziona comunque con molti destinatari, ma non con altri, causando schemi di errore che sembrano riguardare solo singoli mittenti.

```text
example.com.        IN  MX  10 mail1.example.com.
example.com.        IN  MX  20 mail2.example.com.
mail1.example.com.  IN  A   192.0.2.10
mail2.example.com.  IN  A   192.0.2.11
```

**Il numero è una preferenza, non una ponderazione.** Si prova prima il valore più basso. Un secondo MX con un valore elevato ha senso solo se quel sistema conosce lo stesso filtro per i destinatari. Gli MX di backup su sistemi senza verifica dei destinatari sono un bersaglio popolare per lo spam, perché gli aggressori puntano deliberatamente alla voce più debole.

I domini che inviano esclusivamente o non hanno nulla a che fare con l’e-mail ricevono un Null MX secondo RFC 7505. Segnala che il dominio non accetta e-mail e garantisce un rifiuto immediato e univoco invece di timeout:

```text
example.com.  IN  MX  0 .
```

Tuttavia, il Null MX non sostituisce un record SPF e DMARC. Non ricevere non significa che nessuno invii a vostro nome. Le sottodomini parcheggiati, in particolare, vengono usati per lo spoofing perché raramente qualcuno li controlla.

## A, AAAA, PTR e il nome HELO

Il record PTR per l’indirizzo IP in uscita non si trova nella vostra zona, bensì nella zona `in-addr.arpa` del provider a cui appartiene il blocco di indirizzi. Va quindi richiesto al provider e non impostato autonomamente. Molti grandi destinatari richiedono che il PTR e il record diretto associato corrispondano, ossia che il nome del PTR si risolva di nuovo nello stesso indirizzo IP.

```bash
dig +short -x 192.0.2.10
dig +short A mail1.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Mostra solo i valori dei record, senza intestazioni e metadati |
| `-x 192.0.2.10` | Interrogazione inversa: dig genera autonomamente il nome PTR nella zona `in-addr.arpa` |
| `A mail1.example.com` | Interrogazione diretta del nome dal PTR, per verificare che il percorso circolare conduca allo stesso indirizzo IP |

</details>

Il nome che il vostro server e-mail comunica in HELO o EHLO dovrebbe essere lo stesso e anch’esso risolvibile. Un gateway che si presenta come `localhost.localdomain` o con un nome interno viene valutato peggio dai destinatari più grandi.

Occorre prestare attenzione quando si aggiunge un record AAAA. Non appena il server e-mail diventa raggiungibile e inviante tramite IPv6, valgono gli stessi requisiti di IPv4, in parte persino più severi. Google richiede un PTR valido per gli indirizzi IPv6 mittenti. Se manca, l’invio viene rifiutato, mentre via IPv4 funzionava perfettamente. Un record AAAA sul server e-mail non è quindi mai una semplice modifica DNS.

## SPF

SPF stabilisce quali sistemi sono autorizzati a inviare a nome del dominio. Il record è un TXT sul dominio stesso.

```text
example.com.  IN  TXT  "v=spf1 mx include:spf.provider.example -all"
```

### Il limite di lookup

La valutazione di un record SPF può attivare al massimo dieci meccanismi che interrogano il DNS. Si contano `include`, `a`, `mx`, `ptr`, `exists` e `redirect`, in modo ricorsivo: ogni `include` include i lookup del record incorporato. Non vengono contati `ip4`, `ip6` e `all`.

Se il limite viene superato, il risultato è un `permerror`. Per DMARC ciò significa SPF non superato, indipendentemente dal fatto che il server mittente sarebbe effettivamente autorizzato. L’aspetto insidioso è che l’errore spesso si presenta senza alcun intervento proprio, perché un provider incluso estende il suo record. Il proprio record non è cambiato, ma la consegna peggiora comunque.

Inoltre, sono consentiti al massimo due «void lookup», ossia interrogazioni senza risultato. Un `include` verso un dominio che non esiste più rientra in questo conteggio. I riferimenti a fornitori dismessi vanno quindi rimossi e non lasciati per prudenza.

### Cosa non deve essere incluso in un record SPF

- **`ptr`** è specificato, ma è considerato obsoleto da RFC 7208 e non dovrebbe essere utilizzato. I sistemi di valutazione possono ignorarlo.
- **`+all`** autorizza qualunque mittente ed è quindi più dannoso che non avere alcun record SPF.
- **`?all`** è neutro e quindi praticamente inutile per DMARC.
- **Un record separato di tipo SPF (tipo 99)** non è più necessario. È stato eliminato da RFC 7208; SPF risiede esclusivamente in TXT.

Tra `~all` (softfail) e `-all` (hardfail), la scelta dipende da quanto sono completi i percorsi di invio rilevati. Finché sussistono dubbi, `~all` è la scelta corretta. Chi applica già DMARC e valuta i report può passare a `-all`.

### I sottodomini non ereditano nulla

Un record SPF su `example.com` non vale per `newsletter.example.com`. Ogni sottodominio mittente necessita di un proprio record. Per tutti gli altri è consigliabile una voce wildcard che chiarisca che da lì non proviene nulla:

```text
*.example.com.  IN  TXT  "v=spf1 -all"
```

Attenzione: un wildcard TXT risponde anche a richieste di nomi come `_dmarc.sub.example.com`, se non esiste un record esplicito. Di solito non è un problema, ma può rendere più difficile la ricerca degli errori perché ogni interrogazione TXT riceve una risposta.

### SPF flattening

Gli strumenti che risolvono tutti i riferimenti `include` e li sostituiscono con gli indirizzi IP sottostanti risolvono il limite di lookup a scapito della manutenibilità. Se il provider modifica i propri indirizzi, l’invio si interrompe e nessuno se ne accorge, perché nel proprio record apparentemente è tutto corretto. Chi sceglie questa strada necessita quindi di un confronto automatizzato che verifichi regolarmente l’elenco rispetto alla fonte. Come lavoro manuale una tantum, il metodo prima o poi fallisce.

## DKIM

DKIM firma i messaggi in uscita. La chiave pubblica si trova sotto `<selector>._domainkey.<domain>`.

```text
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjANBg..."
```

Il selettore è liberamente scegliibile e viene specificato dal sistema mittente. Un nome descrittivo con data facilita la rotazione successiva molto più di `s1` e `s2`.

### Delega tramite CNAME

Laddove il servizio di invio lo offra, è preferibile la variante CNAME rispetto all’inserimento diretto:

```text
selector1._domainkey.example.com.  IN  CNAME  selector1.dkim.provider.example.
```

Il provider può così ruotare autonomamente la propria chiave, senza che qualcuno debba intervenire nella vostra zona. Altrimenti questa rotazione viene regolarmente trascurata, poiché richiede coordinamento tra due team. Tuttavia, un CNAME esclude qualsiasi ulteriore record con lo stesso nome: è una regola fondamentale del DNS, non una peculiarità di DKIM.

### Rotazione senza interruzioni

Nel cambio di chiave, si pubblica prima il nuovo selettore, poi si passa il server mittente a usarlo e solo successivamente si rimuove il vecchio record. Chi elimina subito la vecchia chiave invalida le firme di tutti i messaggi ancora in transito o in code e rende impossibili le verifiche successive. È opportuno attendere alcuni giorni tra il passaggio e l’eliminazione.

Un record con `p=` vuoto non è una voce difettosa, bensì il metodo specificato per contrassegnare una chiave come ritirata.

### Lunghezza della chiave

1024 bit sono considerati obsoleti, 2048 bit sono lo standard. Chiavi RSA più grandi non apportano alcun vantaggio pratico e aumentano soltanto la probabilità che un sistema intermedio non elabori correttamente il record.

## DMARC

DMARC collega SPF e DKIM a un’istruzione su cosa debba accadere quando una verifica non viene superata e restituisce report. Il record si trova sotto `_dmarc.<domain>`.

```text
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@example.com; sp=none; np=reject; adkim=r; aspf=r"
```

Da maggio 2026, con RFC 9989 e le specifiche sui report RFC 9990 e RFC 9991, si applica la versione rivista, che sostituisce RFC 7489. Per la pratica sono importanti tre modifiche:

- **`pct` è stato eliminato.** Non esiste più l’introduzione graduale tramite percentuale. Al suo posto vi è `t=y`, che contrassegna il dominio come in fase di test: i report continuano, la policy non dovrebbe essere applicata.
- **`np` è nuovo.** Imposta la policy per sottodomini non esistenti, colmando così una lacuna che gli aggressori sfruttano volentieri, poiché finora i sottodomini inventati erano coperti solo da `sp`. Senza indicazione esplicita, `np` segue il valore di `sp`.
- **La Public Suffix List è stata sostituita da un `Tree Walk`.** Il dominio organizzativo non viene più determinato tramite un elenco mantenuto esternamente, ma attraverso interrogazioni DNS graduali lungo l’albero dei nomi. Per grandi spazi di nomi con molti livelli, ciò modifica sensibilmente la valutazione.

### L’allineamento è il vero punto centrale

DMARC non viene superato perché SPF o DKIM hanno avuto tecnicamente esito positivo, ma solo se almeno uno dei due corrisponde inoltre al dominio del mittente visibile nell’header `From`. SPF viene verificato rispetto al dominio del mittente envelope, che differisce regolarmente in caso di inoltri, servizi newsletter e sistemi di ticketing. Per questo motivo, messaggi con SPF valido occasionalmente non superano la verifica DMARC.

Con `adkim=r` e `aspf=r` (relaxed, lo standard) è sufficiente la corrispondenza a livello di dominio organizzativo. `s` richiede l’uguaglianza esatta, incluso il sottodominio, e nella pratica fallisce quasi sempre su uno dei percorsi di invio.

### Gli indirizzi di report esterni richiedono un’autorizzazione

Se i report devono essere inviati a un indirizzo esterno al proprio dominio, ad esempio a un servizio di valutazione DMARC, il dominio ricevente deve autorizzarlo. Senza questo record, molti destinatari semplicemente non inviano nulla e la valutazione resta vuota, mentre nel proprio record tutto sembra corretto:

```text
example.com._report._dmarc.reports.provider.example.  IN  TXT  "v=DMARC1"
```

Questa voce viene creata dal gestore della zona di destinazione, non da voi. Con i servizi commerciali avviene automaticamente, ma non nel caso di una casella di raccolta gestita in proprio in un altro dominio di vostra proprietà.

### Errori di sintassi tipici

I nomi dei tag e i valori delle policy devono essere scritti in minuscolo; `p=Reject` non è valido. Tra i tag deve esserci un punto e virgola; un separatore mancante rende inefficace il resto della riga. Inoltre, `p` deve essere il primo tag dopo `v`. Un record composto solo da `v=DMARC1; rua=...` non contiene alcuna policy ed è incompleto.

### Il rollout

`p=none` è uno stato di misurazione, non un obiettivo. Non modifica il modo in cui i destinatari trattano le vostre e-mail e serve esclusivamente a individuare tramite i report tutti i percorsi di invio legittimi. Chi, dopo l’introduzione, non passa entro pochi mesi da `quarantine` a `reject`, ha sostenuto lo sforzo senza ottenere la protezione. L’aspetto organizzativo di questo percorso, inclusa una proposta decisionale, è un tema a sé ed è descritto nel blueprint DMARC.

## MTA-STS e TLS-RPT

SMTP cifra in modo opportunistico: se la controparte offre STARTTLS, la connessione viene cifrata, altrimenti no. Un aggressore in grado di manipolare il traffico può rimuovere l’annuncio STARTTLS e mantenere così la connessione in chiaro. MTA-STS colma questa lacuna per i domini riceventi.

MTA-STS è composto da due parti, e solo una di esse si trova nel DNS:

```text
_mta-sts.example.com.  IN  TXT    "v=STSv1; id=20260804120000"
mta-sts.example.com.   IN  CNAME  policyhost.example.net.
```

La policy vera e propria è un file all’indirizzo `https://mta-sts.example.com/.well-known/mta-sts.txt` e deve essere distribuita tramite un certificato valido:

```text
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
max_age: 604800
```

Le fonti di errore si trovano quasi tutte al di fuori della zona:

- **L’`id` deve cambiare a ogni modifica della policy.** È l’unico segnale per i sistemi mittenti che una nuova policy deve essere recuperata. Chi modifica il file e lascia invariato l’`id` lavora contro copie memorizzate nella cache fino alla scadenza di `max_age`.
- **L’elenco MX nella policy e i record MX devono corrispondere.** Un nuovo MX assente dalla policy viene rifiutato dai mittenti con `mode: enforce`. Durante le migrazioni, la policy va quindi adattata prima del cambio degli MX.
- **Prima `mode: testing`.** In questa modalità le violazioni vengono solo segnalate, non applicate. Il passaggio a `enforce` avviene quando i report sono puliti.
- **Un record CAA può bloccare l’emissione del certificato per l’host della policy**, se vi è registrata un’autorità di certificazione diversa da quella utilizzata.

TLS-RPT fornisce i report corrispondenti ed è un singolo record:

```text
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

TLS-RPT è utile anche senza MTA-STS, perché rende visibile per la prima volta la cifratura del trasporto non riuscita.

## DANE

DANE raggiunge lo stesso obiettivo di MTA-STS, ma ancora la fiducia nel DNS invece che nella PKI web. Richiede una zona firmata integralmente con DNSSEC; senza DNSSEC, un record TLSA è inefficace.

```text
_25._tcp.mail1.example.com.  IN  TLSA  3 1 1 <hash>
```

Fondamentale nell’esercizio: a ogni cambio di certificato, il record TLSA deve essere aggiornato prima. La procedura usuale pubblica il nuovo hash in parallelo a quello vecchio, poi cambia il certificato e infine rimuove la vecchia voce. Chi inverte questo ordine rende il server e-mail irraggiungibile per tutti i mittenti che verificano DANE, tra cui figurano i grandi provider di lingua tedesca. In Svizzera DANE è nettamente meno diffuso di MTA-STS, di solito per la mancanza della firma DNSSEC della zona.

## BIMI

BIMI mostra il logo del marchio nella posta in arrivo ed è l’unico meccanismo trattato qui che non è ancora un RFC, ma continua a essere gestito come Internet-Draft.

```text
default._bimi.example.com.  IN  TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

I requisiti sono elevati: una policy DMARC applicata con `quarantine` o `reject`, un logo in formato SVG Tiny Portable/Secure e, per la maggior parte dei fornitori, un Verified Mark Certificate a pagamento. BIMI non è quindi un meccanismo di sicurezza, ma una questione di visibilità, e va posto alla fine della sequenza, non all’inizio.

## Altri record correlati

**Autodiscover e SRV:** gli ambienti Exchange utilizzano `autodiscover.example.com` come CNAME o un record SRV `_autodiscover._tcp.example.com`. Entrambi riguardano la configurazione del client e non il flusso di posta, ma vengono spesso trascurati durante la migrazione e causano quindi profili che non possono più essere configurati.

**CAA:** non ha nulla a che fare direttamente con l’e-mail, ma determina quale autorità di certificazione può emettere un certificato per `mta-sts.example.com` o per il nome del server e-mail.

**Zone split-horizon:** dove una zona DNS interna porta lo stesso nome di quella pubblica, spesso i record e-mail non esistono internamente. I sistemi interni che eseguono una verifica SPF o DKIM arrivano quindi a risultati diversi dal mondo esterno. A ogni modifica dei record e-mail occorre quindi chiedersi se la zona interna debba essere aggiornata.

## Alcuni test rapidi

Eseguire deliberatamente tutte le interrogazioni verso un resolver pubblico, affinché non risponda la cache interna o una zona split-horizon:

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
| `@1.1.1.1` | Invia l’interrogazione a questo resolver anziché a quello configurato in `/etc/resolv.conf` |
| `+short` | Mostra solo i valori dei record, senza intestazioni e metadati |
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
| `@ns1.example.com` | Invia l’interrogazione successiva direttamente a uno di questi server autorevoli |
| `+norecurse` | Non imposta il bit Recursion Desired; il server risponde solo dai propri dati di zona, non dalla cache |

</details>

Su Windows senza `dig`:

```text
nslookup -type=TXT _dmarc.example.com 1.1.1.1
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-type=TXT` | Tipo di record da interrogare |
| `_dmarc.example.com` | Nome interrogato |
| `1.1.1.1` | Resolver da utilizzare anziché quello configurato a livello di sistema |

</details>

Per la valutazione completa, incluso il conteggio dei lookup SPF, la ricerca del selettore DKIM e la verifica dell’allineamento, questa pagina offre il [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check), che verifica un dominio in un’unica operazione rispetto a tutti i record qui descritti.

Tuttavia, il test più significativo resta un messaggio reale. Inviate un’e-mail a una casella presso un grande fornitore e osservate la riga `Authentication-Results` nell’intestazione. Mostra in una riga l’effettivo risultato di SPF, DKIM e DMARC e sostituisce qualsiasi teoria sul file di zona.

## Sequenza per una migrazione

Quando si cambia fornitore e-mail, questa sequenza si è dimostrata efficace:

1. Ridurre il TTL di tutti i record coinvolti a 300 secondi, almeno un giorno prima.
2. Pubblicare i selettori DKIM del nuovo fornitore mentre quelli vecchi sono ancora presenti.
3. Estendere SPF con il nuovo fornitore senza rimuovere quello vecchio e ricalcolare il limite di lookup.
4. Per MTA-STS, adattare la policy ai nuovi nomi MX e aumentare l’`id` prima del cambio dei record MX.
5. Cambiare gli MX e monitorare la consegna.
6. Solo dopo alcuni giorni senza problemi, rimuovere i vecchi include SPF e selettori DKIM.
7. Ripristinare il TTL.

Il problema più frequente in questa sequenza è anticipare troppo il passaggio 6: le vecchie voci vengono eliminate insieme al cambio e tutto ciò che continua a passare per il percorso precedente fallisce la verifica del mittente.

## Conclusione

I record e-mail differiscono da tutte le altre voci DNS perché un errore non è necessariamente evidente. Un record A errato genera un ticket entro pochi minuti; un record SPF duplicato o una chiave DKIM con un carattere in più, invece, porta a un tasso di consegna che diminuisce lentamente nel corso delle settimane.

Tre regole evitano la maggior parte di questi casi. Primo: prima di ogni nuovo record, controllare ciò che esiste già invece di aggiungerne un secondo accanto. Secondo: dopo ogni modifica, verificare contro un resolver pubblico e confrontare il valore carattere per carattere con il modello, non solo visivamente. Terzo: nelle modifiche, pubblicare sempre prima il nuovo, poi effettuare il passaggio e infine rimuovere il vecchio. Chi rispetta questa sequenza dispone sempre di una via di ritorno per i record e-mail.

## Fonti

1.  [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035): Definisce, tra le altre cose, il limite di 255 byte di una singola `character-string` nei record TXT.

2.  [RFC 2181: Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181): Stabilisce nella sezione 10.3 che la destinazione di un record MX non può essere un alias.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Limite di lookup di dieci meccanismi, limite dei void lookup, eliminazione del tipo RR SPF e sconsiglio del meccanismo `ptr`.

4.  [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376): Struttura del record della chiave sotto `_domainkey`, significato del selettore e dell’`p=` vuoto.

5.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Specifica DMARC attuale del maggio 2026, sostituisce RFC 7489; eliminazione di `pct`, nuovo tag `np`, Tree Walk al posto della Public Suffix List.

6.  [RFC 9990: DMARC Aggregate Reporting](https://www.rfc-editor.org/rfc/rfc9990): Formato e consegna dei report aggregati, inclusa l’autorizzazione dei domini destinatari esterni.

7.  [RFC 7505: A "Null MX" No Service Resource Record for Domains That Accept No Mail](https://www.rfc-editor.org/rfc/rfc7505): Contrassegno dei domini che non accettano e-mail.

8.  [RFC 8461: SMTP MTA Strict Transport Security (MTA-STS)](https://www.rfc-editor.org/rfc/rfc8461): Record DNS, file di policy, significato dell’`id` e delle modalità `testing` e `enforce`.

9.  [RFC 8460: SMTP TLS Reporting](https://www.rfc-editor.org/rfc/rfc8460): Struttura del record `_smtp._tls` e dei report.

10.  [RFC 7672: SMTP Security via Opportunistic DNS-Based Authentication of Named Entities (DANE)](https://www.rfc-editor.org/rfc/rfc7672): Record TLSA per SMTP e requisito di una zona firmata DNSSEC.

11.  [Brand Indicators for Message Identification (BIMI), Internet-Draft](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/): Stato attuale della specifica BIMI, ancora non un RFC.

12.  [Google: Linee guida per i mittenti e-mail](https://support.google.com/a/answer/81126): Requisiti per i mittenti, tra cui l’obbligo di PTR per gli indirizzi IPv6 mittenti e le disposizioni per mittenti di grandi volumi in vigore da febbraio 2024.

13.  [Microsoft: Strengthening Email Ecosystem, Outlook's New Requirements for High-Volume Senders](https://techcommunity.microsoft.com/blog/microsoft-defender-for-office365-blog/strengthening-email-ecosystem-outlook%E2%80%99s-new-requirements-for-high%E2%80%90volume-senders/4399008): Requisiti per mittenti con almeno 5000 messaggi al giorno, validi dal maggio 2025.
