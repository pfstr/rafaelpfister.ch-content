---
title: "Testare SMTP su Linux: dalla connessione TCP all'e-mail consegnata"
navTitle: "Testare SMTP"
description: "Quando un'appliance non consegna più e-mail, un test SMTP manuale è più utile di qualsiasi log. Come verificare livello per livello con gli strumenti di base, cosa significano i diversi errori e perché un load balancer può falsare la diagnosi."
date: "2026-07-31"
kategorie: "SMTP e flusso di posta"
timeToRead: "10 min di lettura"
themen:
  - smtp-mailflow
  - testing
  - e-mail-verschluesselung
slug: "testare-smtp-su-linux-dalla-connessione-tcp-all-email-consegnata"
translationId: "article-cb44a92c03a47bc0"
translationOf: smtp-verbindung-testen-linux
translationSourceHash: 17f985224e55b45d0395c6f5f6a100918ad6718f2bd22191bc29f30e204e5e11
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:21:03.597Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/testare-smtp-su-linux-dalla-connessione-tcp-all-email-consegnata
---

# Testare SMTP su Linux: dalla connessione TCP all'e-mail consegnata

Quando un gateway di posta smette improvvisamente di consegnare messaggi, i log dell'appliance spesso mostrano solo il risultato finale: una consegna fallisce, la coda cresce, un messaggio di errore indica un timeout. La causa effettiva emerge solo da un test manuale dalla riga di comando. SMTP è un protocollo in chiaro che può essere comunicato interamente a mano, ed è proprio questo a renderlo uno strumento diagnostico disponibile ovunque senza installazioni aggiuntive.

Il secondo motivo per il test manuale: sulle appliance di solito non si può installare nulla. Nessun gestore di pacchetti, nessun privilegio root, nessun `swaks`. Tutti i passaggi seguenti funzionano quindi con ciò che è già disponibile praticamente su ogni sistema Linux.

## Separare i livelli

L'invio di un'e-mail non riuscito può fallire a cinque livelli diversi, e ciascuno genera un quadro di errore differente:

1. **Risoluzione dei nomi:** L'host di destinazione non può essere tradotto in un indirizzo IP.
2. **Connessione TCP:** La connessione alla porta non viene stabilita oppure viene reimpostata.
3. **Dialogo SMTP:** La connessione è stabilita, ma il server rifiuta mittente, destinatario o contenuto.
4. **Crittografia del trasporto:** STARTTLS manca, il certificato non è valido oppure la versione TLS non è compatibile.
5. **Verifica del mittente:** L'e-mail viene accettata e scartata dal destinatario a causa di SPF, DKIM o DMARC.

La diagnosi migliora enormemente se si verificano questi livelli uno dopo l'altro e singolarmente, anziché inviare subito un'e-mail di test completa. Un tentativo complessivo fallito dice solo che qualcosa non funziona. La verifica dei livelli dice cosa.

## Passaggio 1: risoluzione dei nomi

```bash
getent hosts relay.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `hosts` | Database NSS da interrogare; utilizza le stesse Fonti e lo stesso ordine del sistema, secondo `nsswitch.conf` |
| `relay.example.com` | Nome host da risolvere |

</details>

Se l'output rimane vuoto, su questo host non è raggiungibile alcun nameserver oppure non risponde ai nomi esterni. Nella pratica accade regolarmente: le appliance in zone isolate ricevono spesso solo un resolver interno che conosce esclusivamente le proprie zone.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `/etc/resolv.conf` | File emesso da `cat` con i nameserver configurati |
| `hosts:` | Modello di ricerca per `grep`: la riga che definisce l'ordine delle Fonti di risoluzione (file, DNS) |
| `/etc/nsswitch.conf` | File con la configurazione NSS cercato da `grep` |

</details>

Se la risoluzione manca, nei passaggi successivi testate direttamente l'indirizzo IP. Per la diagnosi è del tutto sufficiente e separa nettamente il problema DNS dal problema di trasporto. Per l'esercizio in produzione, naturalmente, la risoluzione mancante resta un problema a sé stante da correggere.

## Passaggio 2: raggiungibilità della porta

Per il puro controllo TCP basta bash. Lo pseudodispositivo `/dev/tcp` apre una connessione senza che sia necessario installare `nc` o `telnet`:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `timeout 10` | Interrompe il comando seguente dopo 10 secondi e restituisce quindi il codice di uscita 124 |
| `bash -c '…'` | Esegue la catena di comandi in una bash; necessario perché `/dev/tcp` è una funzionalità di bash |
| `exec 3<>/dev/tcp/192.0.2.25/25` | Apre il descrittore di file 3 in lettura e scrittura come connessione TCP a 192.0.2.25, porta 25 |
| `echo "exit=$?"` | Emette il codice di uscita del comando precedente |

</details>

Il codice di uscita è qui l'informazione effettiva:

| exit | Significato |
|---|---|
| `0` | La connessione è stabilita, la porta è aperta |
| `124` | Timeout: i pacchetti vengono scartati, tipico di un firewall con regola DROP |
| `1` | Rifiuto immediato (RST) oppure route assente |

Nella pratica, la differenza tra 124 e 1 è l'indicazione più importante in assoluto. Un timeout significa che qualcuno scarta silenziosamente lungo il percorso, e quasi sempre si tratta di una regola firewall. Un RST immediato, invece, proviene da un sistema che risponde ma non offre il servizio.

Verificate subito entrambe le porte rilevanti e anche un'altra destinazione qualsiasi per vedere se l'host può stabilire connessioni in uscita:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do
  set -- $t
  timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  echo "$1:$2 -> exit=$?"
done
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `set -- $t` | Divide la coppia di valori allo spazio nei parametri posizionali `$1` (indirizzo IP) e `$2` (porta) |
| `timeout 8` | Interrompe il tentativo di connessione dopo 8 secondi (codice di uscita 124) |
| `bash -c "…"` | Esegue l'apertura della connessione `/dev/tcp` in una bash |
| `2>/dev/null` | Sopprime i messaggi di errore affinché compaia esattamente una riga di risultato per destinazione |

</details>

Se fallisce anche il test di controllo, il sistema non dispone in generale di un'uscita diretta e il traffico deve passare attraverso un relay interno o un proxy. Più avanti viene spiegato perché questo caso è particolarmente insidioso.

Se manca `/dev/tcp`, la shell non è bash. In `sh`, `ash` o `ksh` questa funzionalità non esiste, e ciò viene spesso interpretato erroneamente come un problema di rete:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-p $$` | Limita l'output al processo con il PID della shell attuale (`$$`) |
| `-o comm=` | Emette solo il nome del comando; l'etichetta vuota dopo `=` sopprime l'intestazione |
| `${BASH_VERSION:-keine bash}` | Emette la versione di bash oppure il testo sostitutivo se la variabile non è impostata |

</details>

## Passaggio 3: prima ascoltare, non inviare

Un server SMTP si presenta autonomamente con un banner `220`. Il singolo test più significativo consiste quindi nell'aprire una connessione e non fare nulla:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Apre il descrittore di file 3 come connessione TCP alla destinazione |
| `timeout 15 cat <&3` | Legge per 15 secondi tutto ciò che il server invia autonomamente e lo stampa |
| `echo "[ende exit=$?]"` | Mostra il codice di uscita allo scadere; 124 significa: per 15 secondi non è arrivato altro |

</details>

Questi pochi caratteri separano due situazioni completamente diverse. Se arriva un `220 mail.example.com ESMTP`, il server remoto comunica e tutti gli altri errori sono nel dialogo. Se non arriva nulla, non dipende da un comando formulato in modo errato da parte vostra, poiché non ne avete inviato nessuno.

Il descrittore di file resta quindi aperto nella shell. Chiudetelo prima di avviare il test successivo, altrimenti potreste continuare a lavorare con una vecchia connessione non più integra:

```bash
exec 3<&- 3>&-
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `3<&-` | Chiude il lato di lettura del descrittore di file 3 |
| `3>&-` | Chiude il lato di scrittura del descrittore di file 3 |

</details>

## Passaggio 4: il dialogo SMTP a mano

Se il banner è presente, eseguite il dialogo completo. È importante un processo di lettura in esecuzione, affinché vediate ogni risposta nel momento in cui arriva. Uno script che prima invia tutto e legge solo dopo non vi mostra nulla se la connessione si interrompe a metà del dialogo:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\n' >&3
printf 'Date: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `exec 3<>/dev/tcp/192.0.2.25/25` | Apre il descrittore di file 3 come connessione TCP alla destinazione |
| `cat <&3 & R=$!` | Avvia un lettore in background per il descrittore di file 3 e ne salva il PID in `R` |
| `printf '…\r\n' >&3` | Invia un comando SMTP con la terminazione di riga CRLF richiesta sulla connessione |
| `sleep n` | Attende il numero di secondi indicato per la risposta del server prima del comando successivo |
| `date -R` | Fornisce la data nel formato conforme a RFC per l'header `Date:` |
| `date +%s` | Fornisce l'ora Unix come semplice base univoca per il Message-ID |
| `kill $R 2>/dev/null` | Termina il lettore in background; il messaggio di errore viene omesso se è già terminato |

</details>

Due dettagli determinano il successo o il fallimento. SMTP richiede CRLF come terminazione di riga, quindi `printf` con `\r\n` e non `echo`. Il punto su una riga propria termina la parte del messaggio; deve essere inviato come `\r\n.\r\n`.

Il flusso previsto: `220` all'apertura della connessione, `250` su EHLO, `250 2.1.0` su MAIL FROM, `250 2.1.5` su RCPT TO, `354` su DATA e infine `250 2.0.0 Ok: queued as <id>`. Annotate l'ID della coda. Consente al provider che gestisce il servizio di rintracciare il messaggio se non arriva mai al destinatario.

Il nome EHLO merita attenzione: alcuni relay lo verificano rispetto al DNS forward e reverse e altrimenti rispondono con `501` o `504`. Utilizzate l'FQDN effettivo del sistema mittente, non il nome breve.

## Passaggio 5: STARTTLS e certificato

Per la connessione cifrata, `openssl s_client` esegue autonomamente la negoziazione STARTTLS e poi passa il canale allo standard input:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-connect 192.0.2.25:25` | Host e porta di destinazione della connessione |
| `-starttls smtp` | Esegue prima il dialogo SMTP in chiaro e poi passa a TLS tramite STARTTLS |
| `-tls1_2` | Negoziato esclusivamente TLS 1.2 |
| `-brief` | Riduce l'output a un breve riepilogo della connessione negoziata |
| `</dev/null` | Chiude immediatamente lo standard input affinché `s_client` non resti in attesa interattiva dopo l'handshake |

</details>

Se vi connettete tramite l'indirizzo IP perché il DNS manca, la verifica del nome host non può funzionare. Il nome del certificato non corrisponde quindi all'indirizzo numerico. SNI e nome di verifica possono essere impostati esplicitamente, senza alcuna interrogazione DNS:

```bash
openssl s_client -connect 192.0.2.25:25 \
  -servername mail.example.com -verify_hostname mail.example.com \
  -starttls smtp -tls1_2 -brief </dev/null
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-servername mail.example.com` | Imposta il nome SNI nel ClientHello, indipendentemente dall'indirizzo di connessione |
| `-verify_hostname mail.example.com` | Verifica il certificato del server rispetto a questo nome anziché all'indirizzo numerico |

</details>

Qui compaiono regolarmente due quadri di errore che vengono spesso interpretati erroneamente.

**«Didn't find STARTTLS in server response, trying anyway»** significa che il server non ha offerto STARTTLS nella risposta EHLO. `openssl` invia comunque un ClientHello TLS, il server vi vede dati di protocollo non validi e la connessione termina con `wrong version number` oppure `write:errno=32` (EPIPE). Entrambi i messaggi sono errori conseguenti. L'informazione effettiva è: nessun STARTTLS. Con il dialogo in chiaro del passaggio 4, verificate quali capability il server segnala realmente.

**Nessun STARTTLS su un hop interno** è spesso del tutto corretto. Se un load balancer inoltra la connessione a livello 4, non è lui a negoziare TLS, ma il sistema dietro di esso con la destinazione effettiva. Testare in chiaro sul segmento interno non è quindi una carenza di sicurezza, ma semplicemente l'architettura.

## Passaggio 6: Python come alternativa

Se Python è disponibile, potete risparmiarvi la temporizzazione manuale con `sleep`. Basta la libreria standard, non occorre installare nulla:

```python
#!/usr/bin/env python3
import smtplib, ssl
from email.message import EmailMessage
from email.utils import formatdate, make_msgid

msg = EmailMessage()
msg["From"] = "absender@example.com"
msg["To"] = "empfaenger@example.net"
msg["Subject"] = "Relay-Test"
msg["Date"] = formatdate(localtime=True)
msg["Message-ID"] = make_msgid(domain="example.com")
msg.set_content("Testnachricht\n")

ctx = ssl.create_default_context()
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

s = smtplib.SMTP("192.0.2.25", 25, timeout=30, local_hostname="host.example.com")
s.set_debuglevel(1)
s.ehlo()
if s.has_extn("starttls"):
    s.starttls(context=ctx, server_hostname="mail.example.com")
    s.ehlo()
    print("TLS:", s.sock.version(), s.sock.cipher()[0])
s.send_message(msg)
s.quit()
```

`set_debuglevel(1)` registra l'intero dialogo, inclusi tutti i codici di risposta, e `smtplib` legge ogni risposta in modo sincrono. Un'interruzione appare come `SMTPServerDisconnected` insieme all'ultima riga ricevuta, anziché come una Broken Pipe silenziosa.

Qui due aspetti vanno spesso storti: `server_hostname` è obbligatorio quando ci si connette tramite un indirizzo IP, altrimenti Python verifica il certificato rispetto all'indirizzo numerico. E se disattivate deliberatamente la verifica, `check_hostname = False` deve precedere `verify_mode = ssl.CERT_NONE`, altrimenti Python genera un `ValueError`.

## Indirizzo del mittente, SPF e alignment

Un test sorprendentemente spesso non fallisce nel trasporto, ma nell'indirizzo del mittente scelto. Tre aspetti devono essere verificati in anticipo.

Il dominio del mittente deve essere un FQDN. Un indirizzo come `test@meine-testdomain` senza dominio di primo livello viene già rifiutato da molti MTA durante MAIL FROM con `501` o `553`.

Il dominio deve autorizzare il percorso di invio utilizzato. Uno sguardo al record SPF mostra se l'indirizzo in uscita è coperto:

```bash
dig +short TXT example.com | grep spf1
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Emette solo i valori dei record, senza header e metadati |
| `TXT` | Tipo di record interrogato |
| `example.com` | Nome interrogato |
| `grep spf1` | Filtra la riga SPF da più record TXT |

</details>

E con DMARC attivo, decide l'alignment. Se nel record è presente `aspf=s`, il dominio nell'envelope (MAIL FROM) e il dominio nell'header `From:` devono corrispondere esattamente, non solo essere correlati:

```bash
dig +short TXT _dmarc.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `+short` | Emette solo i valori dei record, senza header e metadati |
| `TXT _dmarc.example.com` | Tipo di record e nome definito per DMARC sotto il dominio |

</details>

Con `p=reject` un'e-mail di test con alignment non corrispondente scompare silenziosamente presso il destinatario, anche se il vostro relay l'ha accettata con `250 queued`. Questa è la causa più frequente dei messaggi considerati inviati con successo lato mittente che tuttavia non arrivano mai.

## Quando c'è un load balancer nel mezzo

Negli ambienti più grandi, un'appliance raramente invia direttamente su Internet. È comune un server virtuale su un load balancer, che accetta la connessione, la riscrive tramite source NAT a un indirizzo definito e solo allora la inoltra all'esterno. Ciò ha una conseguenza spiacevole per la diagnosi.

Un server virtuale che opera a livello 4 conferma immediatamente l'handshake TCP, prima di aver stabilito esso stesso una connessione alla destinazione. Se questa seconda connessione fallisce, sul client vedete una connessione stabilita con successo e subito dopo reimpostata: `Connection reset by peer`, senza alcun banner SMTP. L'errore non è quindi presso di voi né presso la destinazione, ma nel pool dietro il server virtuale, ad esempio perché un membro è contrassegnato come down o l'FQDN configurato non viene risolto.

Questo spiega anche perché un test diretto contro la destinazione Internet deve fallire quando la regola di inoltro accetta solo traffico dall'indirizzo SNAT già riscritto. Le connessioni con l'indirizzo sorgente originale non corrispondono a nessuna regola e vengono scartate. In tali ambienti, testate sempre contro il server virtuale previsto, non contro la destinazione effettiva.

Quale indirizzo sorgente utilizza il vostro sistema per una determinata destinazione viene indicato in una sola riga. Il valore dopo `src` è esattamente l'informazione necessaria al team di rete per l'abilitazione:

```bash
ip route get 192.0.2.25
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `route get` | Interroga il kernel su quale route sceglierebbe per una destinazione concreta |
| `192.0.2.25` | Indirizzo di destinazione della connessione simulata |

</details>

Se il sistema è dietro NAT, il server remoto non vede questo indirizzo, bensì l'indirizzo pubblico del perimetro. Non è possibile determinarlo dall'interno finché non passa alcun traffico; è indicato nella regola NAT.

## Quadri di errore a colpo d'occhio

| Osservazione | Causa probabile |
|---|---|
| `Name or service not known` | Nessuna risoluzione dei nomi sull'host |
| Timeout, exit 124 | Il firewall scarta silenziosamente (DROP) |
| `Connection refused` | Nessun servizio sulla porta oppure regola REJECT |
| Connessione stabilita, nessun banner, poi RST | Il load balancer accetta, backend non raggiungibile |
| `Didn't find STARTTLS` | Il server non offre la crittografia del trasporto |
| `wrong version number`, `errno=32` | Errori conseguenti dopo TLS forzato senza STARTTLS |
| `501` / `553` su MAIL FROM | Dominio del mittente non FQDN o non consentito |
| `554 relay access denied` | IP sorgente non abilitato sul relay |
| `250 queued`, ma nessuna consegna | SPF, DKIM o alignment DMARC presso il destinatario |

## Test di carico e limiti di frequenza

Per i test di volume vale una regola spesso trascurata nella quotidianità: il problema non è il numero di messaggi, ma il numero di connessioni. I relay tipici consentono alcune centinaia di connessioni al minuto, ma decine di migliaia di messaggi. Mantenete quindi aperta una sessione e inviate molti envelope attraverso di essa, anziché riconnettervi per ogni messaggio.

In `smtplib` ciò significa semplicemente riutilizzare più volte lo stesso oggetto di connessione e ristabilire la sessione in modo controllato dopo un numero fisso di messaggi. Chi invece apre una nuova connessione per ogni e-mail supera il limite delle connessioni molto prima di quello dei messaggi e provoca rifiuti che sembrano un problema della controparte.

## Conclusione

Il test SMTP manuale non è un ripiego per ambienti senza strumenti, ma la diagnosi più precisa disponibile nel funzionamento della posta. Separa chiaramente risoluzione dei nomi, raggiungibilità, dialogo di protocollo e crittografia, e fornisce un risultato univoco per ogni livello. Chi prima si limita ad ascoltare, poi esegue il dialogo a mano e prende sul serio i codici di risposta, in pochi minuti ottiene un'affermazione con cui documentare un ticket al team di rete o al provider: con indirizzo sorgente, porta di destinazione, comportamento osservato e codice di uscita.

## Cronologia delle modifiche

- **25 ottobre 2026:** sostituita la fonte RFC 7489 con RFC 9989, che sostituisce la specifica DMARC dal maggio 2026.

## Fonti

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definisce il dialogo SMTP, la sequenza dei comandi e il significato dei codici di risposta.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Descrive STARTTLS come estensione, incluso il comportamento quando il server non la offre.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Struttura e valutazione del record SPF per l'autorizzazione dei sistemi mittenti.

4.  [RFC 9989: Domain-Based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc9989): Disciplina l'alignment tra mittente envelope e header, il tag `aspf` e la valutazione della policy; sostituisce RFC 7489 dal maggio 2026.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Riferimento alle opzioni utilizzate, tra cui `-starttls`, `-servername` e `-verify_hostname`.

6.  [Documentazione Python: smtplib](https://docs.python.org/3/library/smtplib.html): Libreria standard per sessioni SMTP, incluse STARTTLS e l'output di debug.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documenta `/dev/tcp` come pseudodispositivo specifico di bash per le connessioni di rete.
