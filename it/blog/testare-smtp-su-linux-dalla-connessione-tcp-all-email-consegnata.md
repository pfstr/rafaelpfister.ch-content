---
title: "Testare SMTP su Linux: dalla connessione TCP all'email consegnata"
navTitle: "Testare SMTP"
description: "Quando un'appliance non consegna più email, un test SMTP manuale è più utile di qualsiasi log. Come verificare livello per livello con gli strumenti integrati, cosa significano i diversi errori e perché un load balancer può falsare la diagnosi."
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
url: https://rafaelpfister.ch/it/blog/testare-smtp-su-linux-dalla-connessione-tcp-all-email-consegnata
translationSourceHash: 650b4717ca00ffd3d02cebae8f1484027cf0f9b47de1b607caa951cd7264454a
translationModel: gpt-5.6-terra
translatedAt: 2026-08-01T06:13:04.776Z
translationReview: automatic
---

# Testare SMTP su Linux: dalla connessione TCP all'email consegnata

Quando un mail gateway smette improvvisamente di consegnare messaggi, i log dell'appliance spesso mostrano solo l'ultima fase della storia: una consegna fallisce, la coda cresce, un messaggio di errore indica un timeout. La causa reale emerge solo con un test manuale dalla riga di comando. SMTP è un protocollo in chiaro che può essere gestito interamente a mano, ed è proprio questo a renderlo uno degli strumenti diagnostici più efficaci nella gestione della posta.

Il secondo motivo per eseguire il test manualmente: sulle appliance di solito non è possibile installare nulla. Nessun gestore di pacchetti, nessun diritto root, nessun `swaks`. Tutti i passaggi seguenti funzionano quindi con ciò che è già disponibile praticamente su ogni sistema Linux.

## Separare i livelli

L'invio di un'email può fallire su cinque livelli diversi, e ciascuno produce un errore differente:

1. **Risoluzione dei nomi:** l'host di destinazione non può essere tradotto in un indirizzo IP.
2. **Connessione TCP:** la connessione alla porta non viene stabilita oppure viene reimpostata.
3. **Dialogo SMTP:** la connessione è attiva, ma il server rifiuta mittente, destinatario o contenuto.
4. **Cifratura del trasporto:** STARTTLS non è disponibile, il certificato non è valido oppure la versione TLS non è compatibile.
5. **Verifica del mittente:** l'email viene accettata e scartata dal destinatario a causa di SPF, DKIM o DMARC.

La diagnosi migliora notevolmente quando si verificano questi livelli uno dopo l'altro e separatamente, invece di inviare subito un'email di prova completa. Un tentativo complessivo fallito dice solo che qualcosa non funziona. La verifica per livelli indica cosa.

## Passaggio 1: risoluzione dei nomi

```bash
getent hosts relay.example.com
```

Se l'output rimane vuoto, su questo host non è raggiungibile alcun nameserver oppure non risponde ai nomi esterni. È più frequente di quanto si pensi: le appliance in zone isolate ricevono spesso solo un resolver interno che conosce esclusivamente le proprie zone.

```bash
cat /etc/resolv.conf; grep hosts: /etc/nsswitch.conf
```

Se la risoluzione manca, nei passaggi seguenti testate direttamente l'indirizzo IP. Per la diagnosi è del tutto sufficiente e separa chiaramente il problema DNS dal problema di trasporto. In produzione, naturalmente, la mancata risoluzione rimane un problema distinto da correggere.

## Passaggio 2: raggiungibilità della porta

Per la sola verifica TCP è sufficiente bash. Il dispositivo pseudo `/dev/tcp` apre una connessione senza richiedere l'installazione di `nc` o `telnet`:

```bash
timeout 10 bash -c 'exec 3<>/dev/tcp/192.0.2.25/25' ; echo "exit=$?"
```

Qui l'informazione principale è il codice di uscita:

| exit | Significato |
|---|---|
| `0` | Connessione stabilita, la porta è aperta |
| `124` | Timeout: i pacchetti vengono scartati, tipico di un firewall con regola DROP |
| `1` | Rifiuto immediato (RST) oppure route assente |

Nella pratica, la differenza tra 124 e 1 è l'indizio più importante in assoluto. Un timeout significa che qualcuno lungo il percorso sta scartando silenziosamente il traffico, e quasi sempre si tratta di una regola firewall. Un RST immediato, invece, proviene da un sistema che risponde ma non offre il servizio.

Verificate subito entrambe le porte rilevanti e anche un'altra destinazione qualsiasi, per capire se l'host può stabilire connessioni in uscita:

```bash
for t in "192.0.2.25 25" "192.0.2.25 587" "1.1.1.1 443"; do set -- $t; timeout 8 bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null; echo "$1:$2 -> exit=$?"; done
```

Se anche la controprova non produce risultati, il sistema non dispone in generale di accesso diretto in uscita e il traffico deve passare attraverso un relay interno o un proxy. Più avanti vedremo perché questo caso è particolarmente insidioso.

Se manca `/dev/tcp`, la shell non è bash. In `sh`, `ash` o `ksh` questa funzionalità non esiste, cosa che viene spesso interpretata erroneamente come un problema di rete:

```bash
ps -p $$ -o comm= ; echo "BASH_VERSION=${BASH_VERSION:-keine bash}"
```

## Passaggio 3: prima ascoltare, poi inviare

Un server SMTP si presenta autonomamente con un banner `220`. Il singolo test più significativo consiste quindi nell'aprire una connessione e non fare nulla:

```bash
{ exec 3<>/dev/tcp/192.0.2.25/25 && timeout 15 cat <&3; echo "[ende exit=$?]"; }
```

Questi pochi caratteri distinguono due situazioni completamente diverse. Se arriva un `220 mail.example.com ESMTP`, l'altra parte comunica e tutti gli ulteriori errori sono nel dialogo. Se non arriva nulla, non dipende da un comando formulato male, perché non ne avete inviato alcuno.

Il descrittore di file rimane aperto nella shell. Chiudetelo prima di avviare il test successivo, altrimenti potreste continuare a lavorare con una connessione vecchia e parzialmente inattiva:

```bash
exec 3<&- 3>&-
```

## Passaggio 4: il dialogo SMTP manuale

Se il banner è presente, eseguite il dialogo completo. È importante avere un processo di lettura attivo, così da vedere ogni risposta nell'istante in cui arriva. Uno script che invia prima tutto e legge solo dopo non mostra nulla se l'interruzione avviene a metà dialogo:

```bash
{
exec 3<>/dev/tcp/192.0.2.25/25
cat <&3 & R=$!
sleep 1; printf 'EHLO host.example.com\r\n' >&3
sleep 2; printf 'MAIL FROM:<absender@example.com>\r\n' >&3
sleep 2; printf 'RCPT TO:<empfaenger@example.net>\r\n' >&3
sleep 2; printf 'DATA\r\n' >&3
sleep 2; printf 'From: absender@example.com\r\nTo: empfaenger@example.net\r\nSubject: Relay-Test\r\nDate: %s\r\nMessage-ID: <%s@example.com>\r\n\r\nTestnachricht\r\n.\r\n' "$(date -R)" "$(date +%s).$$" >&3
sleep 3; printf 'QUIT\r\n' >&3
sleep 2; kill $R 2>/dev/null
}
```

Due dettagli determinano il successo o la frustrazione. SMTP richiede CRLF come fine riga, quindi `printf` con `\r\n` e non `echo`. Inoltre, il punto su una riga separata termina la parte del messaggio; deve essere inviato come `\r\n.\r\n`.

Il flusso previsto: `220` all'apertura della connessione, `250` in risposta a EHLO, `250 2.1.0` per MAIL FROM, `250 2.1.5` per RCPT TO, `354` per DATA e infine `250 2.0.0 Ok: queued as <id>`. Annotate l'ID della coda. Consente al provider che gestisce il servizio di tracciare il messaggio se non arriva mai al destinatario.

Il nome EHLO merita attenzione: alcuni relay lo verificano rispetto al DNS diretto e inverso e altrimenti rispondono con `501` o `504`. Utilizzate l'FQDN effettivo del sistema mittente, non il nome breve.

## Passaggio 5: STARTTLS e certificato

Per la connessione cifrata, `openssl s_client` gestisce autonomamente la negoziazione STARTTLS e poi passa il canale allo standard input:

```bash
openssl s_client -connect 192.0.2.25:25 -starttls smtp -tls1_2 -brief </dev/null
```

Se vi connettete tramite indirizzo IP perché manca il DNS, la verifica del nome host non può funzionare. Il nome nel certificato non corrisponde all'indirizzo numerico. SNI e nome da verificare possono essere impostati esplicitamente, senza alcuna interrogazione DNS:

```bash
openssl s_client -connect 192.0.2.25:25 -servername mail.example.com -verify_hostname mail.example.com -starttls smtp -tls1_2 -brief </dev/null
```

Qui si presentano regolarmente due scenari di errore che vengono spesso interpretati in modo errato.

**«Didn't find STARTTLS in server response, trying anyway»** significa che il server non ha offerto STARTTLS nella risposta EHLO. `openssl` invia comunque un TLS ClientHello, il server lo interpreta come dati di protocollo non validi e la connessione termina con `wrong version number` oppure `write:errno=32` (EPIPE). Entrambi i messaggi sono errori conseguenti. L'informazione reale è: nessun STARTTLS. Con il dialogo in chiaro del passaggio 4, verificate quali capability il server comunica effettivamente.

**Nessun STARTTLS su un hop interno** è spesso del tutto corretto. Quando un load balancer inoltra la connessione a livello 4, non negozia lui TLS: lo fa il sistema posto dietro di esso verso la destinazione effettiva. Testare in chiaro sul segmento interno non è quindi una carenza di sicurezza, ma semplicemente parte dell'architettura.

## Passaggio 6: Python come alternativa

Se Python è disponibile, potete evitare la gestione manuale dei tempi con `sleep`. La libreria standard è sufficiente e non occorre installare nulla:

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

Due insidie: `server_hostname` è obbligatorio quando ci si connette tramite indirizzo IP, altrimenti Python verifica il certificato rispetto all'indirizzo numerico. Inoltre, se disattivate consapevolmente la verifica, `check_hostname = False` deve precedere `verify_mode = ssl.CERT_NONE`, altrimenti Python genera un `ValueError`.

## Indirizzo del mittente, SPF e allineamento

Un test sorprendentemente spesso non fallisce nel trasporto, ma nell'indirizzo del mittente scelto. Tre aspetti vanno verificati in anticipo.

Il dominio del mittente deve essere un FQDN. Un indirizzo come `test@meine-testdomain` senza dominio di primo livello viene già rifiutato da molti MTA durante MAIL FROM con `501` o `553`.

Il dominio deve autorizzare il percorso di invio utilizzato. Uno sguardo al record SPF mostra se l'indirizzo in uscita è coperto:

```bash
dig +short TXT example.com | grep spf1
```

Con DMARC attivo, è l'allineamento a decidere. Se il record contiene `aspf=s`, il dominio nell'envelope (MAIL FROM) e il dominio nell'header `From:` devono corrispondere esattamente, non essere soltanto correlati:

```bash
dig +short TXT _dmarc.example.com
```

Con `p=reject`, un'email di prova con allineamento non corretto scompare silenziosamente presso il destinatario, anche se il relay l'ha accettata con `250 queued`. È la causa più comune dei messaggi considerati inviati con successo dal lato mittente ma che non arrivano mai.

## Quando è presente un load balancer

Negli ambienti più grandi, un'appliance raramente invia direttamente su Internet. È comune usare un server virtuale su un load balancer, che accetta la connessione, riscrive l'indirizzo in un indirizzo definito tramite Source NAT e solo allora la inoltra all'esterno. Per la diagnosi ciò comporta una conseguenza spiacevole.

Un server virtuale che lavora a livello 4 conferma immediatamente l'handshake TCP, prima ancora di aver stabilito una connessione verso la destinazione. Se questa seconda connessione fallisce, sul client vedrete una connessione stabilita con successo e immediatamente dopo reimpostata: `Connection reset by peer`, senza alcun banner SMTP. L'errore non è quindi vostro né della destinazione, ma nel pool dietro il server virtuale, ad esempio perché un membro è contrassegnato come down oppure l'FQDN configurato non viene risolto.

Questo spiega anche perché un test diretto verso la destinazione Internet deve fallire quando la regola di inoltro accetta solo traffico proveniente dall'indirizzo SNAT già riscritto. Le connessioni con l'indirizzo sorgente originale non corrispondono a nessuna regola e vengono scartate. In questi ambienti, testate sempre il server virtuale previsto, non la destinazione effettiva.

Quale indirizzo sorgente il sistema utilizza per una determinata destinazione è indicato da una sola riga. Il valore dopo `src` è esattamente l'informazione necessaria al team di rete per l'abilitazione:

```bash
ip route get 192.0.2.25
```

Se il sistema si trova dietro NAT, la controparte non vede questo indirizzo bensì quello pubblico del perimetro. Non è possibile determinarlo dall'interno finché nessun traffico riesce a passare; è indicato nella regola NAT.

## Errori a colpo d'occhio

| Osservazione | Causa probabile |
|---|---|
| `Name or service not known` | Nessuna risoluzione dei nomi sull'host |
| Timeout, exit 124 | Il firewall scarta silenziosamente (DROP) |
| `Connection refused` | Nessun servizio sulla porta oppure regola REJECT |
| Connessione stabilita, nessun banner, poi RST | Il load balancer accetta, backend non raggiungibile |
| `Didn't find STARTTLS` | Il server non offre la cifratura del trasporto |
| `wrong version number`, `errno=32` | Errori conseguenti dopo TLS forzato senza STARTTLS |
| `501` / `553` su MAIL FROM | Il dominio del mittente non è un FQDN oppure non è autorizzato |
| `554 relay access denied` | IP sorgente non abilitato sul relay |
| `250 queued`, ma nessuna consegna | Allineamento SPF, DKIM o DMARC presso il destinatario |

## Test di carico e limiti di frequenza

Per i test di volume vale una regola spesso trascurata nella pratica quotidiana: il problema non è il numero di messaggi, ma il numero di connessioni. I relay tipici consentono alcune centinaia di connessioni al minuto, ma decine di migliaia di messaggi. Mantenete quindi aperta una sessione e inviate molti envelope attraverso di essa, invece di riconnettervi per ogni messaggio.

In `smtplib` ciò significa semplicemente riutilizzare più volte lo stesso oggetto di connessione e ristabilire la sessione in modo controllato dopo un numero fisso di messaggi. Chi invece apre una nuova connessione per ogni email raggiunge il limite di connessioni molto prima del limite di messaggi e provoca rifiuti che sembrano un problema della controparte.

## Conclusione

Il test SMTP manuale non è un espediente per ambienti privi di strumenti, ma la diagnosi più precisa disponibile nella gestione della posta. Separa chiaramente risoluzione dei nomi, raggiungibilità, dialogo di protocollo e cifratura, fornendo un risultato univoco per ogni livello. Chi prima ascolta soltanto, poi esegue manualmente il dialogo e prende sul serio i codici di risposta, arriva in pochi minuti a una conclusione con cui documentare un ticket al team di rete o al provider: con indirizzo sorgente, porta di destinazione, comportamento osservato e codice di uscita.

## Fonti

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Definisce il dialogo SMTP, l'ordine dei comandi e il significato dei codici di risposta.

2.  [RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/rfc/rfc3207): Descrive STARTTLS come estensione, incluso il comportamento quando il server non la offre.

3.  [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208): Struttura e valutazione del record SPF per l'autorizzazione dei sistemi mittenti.

4.  [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489): Regola l'allineamento tra mittente envelope e header, nonché la valutazione delle policy.

5.  [OpenSSL: s_client](https://docs.openssl.org/master/man1/openssl-s_client/): Riferimento per le opzioni utilizzate, tra cui `-starttls`, `-servername` e `-verify_hostname`.

6.  [Documentazione Python: smtplib](https://docs.python.org/3/library/smtplib.html): Libreria standard per sessioni SMTP, inclusi STARTTLS e output di debug.

7.  [Bash Reference Manual: Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections): Documenta `/dev/tcp` come dispositivo pseudo specifico di bash per le connessioni di rete.
