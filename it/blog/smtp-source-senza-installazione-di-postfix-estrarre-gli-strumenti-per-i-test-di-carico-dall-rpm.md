---
title: "smtp-source senza installare Postfix: estrarre gli strumenti di test di carico dall'RPM"
navTitle: "Estrarre smtp-source"
description: "smtp-source e smtp-sink fanno parte di Postfix, ma funzionano anche senza un server di posta installato. Come estrarre i due strumenti dal pacchetto su RHEL, perché l'esecuzione da /tmp può fallire a causa dell'opzione di mount noexec e quali librerie devono essere incluse."
date: "2026-08-27"
kategorie: "SMTP e flusso di posta"
timeToRead: "7 min di lettura"
themen:
  - smtp-mailflow
  - smtp-lasttests
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "testing"
  - "troubleshooting"
slug: "smtp-source-senza-installazione-di-postfix-estrarre-gli-strumenti-per-i-test-di-carico-dall-rpm"
translationId: "article-d0a27da11509d24b"
translationOf: smtp-source-ohne-postfix-installation
translationSourceHash: fd4ad6beb5036817db9b7758653a2b7d015a6adb15d7b4a0b47f94161e34b4e6
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:52:58.598Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/smtp-source-senza-installazione-di-postfix-estrarre-gli-strumenti-per-i-test-di-carico-dall-rpm
---

# smtp-source senza installare Postfix: estrarre gli strumenti di test di carico dall'RPM

Per i test di carico SMTP, `smtp-source` è una buona scelta: lo strumento apre sessioni parallele, le mantiene aperte per più messaggi e riproduce quindi il comportamento di connessione di un mittente massivo in modo molto più realistico rispetto agli strumenti di test che aprono una nuova connessione per ogni email. La controparte `smtp-sink` accetta le email e le scarta, senza consegnare nulla. Entrambi fanno parte della distribuzione di Postfix.

Ed è proprio qui che sta il problema: sul sistema dal quale si desidera effettuare i test spesso Postfix non è installato. Su un'appliance gateway di posta, inoltre, l'installazione non è desiderata, poiché un Postfix aggiuntivo porta con sé una propria configurazione in `/etc/postfix` e un servizio di sistema che, nel peggiore dei casi, occupa la porta 25, bloccando così il sistema di posta vero e proprio. Si aggiunge poi la questione di cosa pensi il supporto del produttore dei pacchetti installati successivamente sulla propria appliance.

Entrambi gli strumenti possono però essere utilizzati anche senza installazione: scaricare l'RPM, estrarre binari e librerie, fatto. Il percorso presenta due particolarità, illustrate in questo articolo su un sistema RHEL 8. Non sono necessari privilegi di root, ma solo l'accesso alle fonti dei pacchetti.

## smtp-source è già presente?

Per prima cosa, verificare che lo strumento non sia già presente sul sistema. `smtp-source` si trova, a seconda della distribuzione, al di fuori del normale PATH:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `command -v smtp-source` | Restituisce il percorso se il programma è nel PATH; altrimenti non restituisce nulla |
| `/usr/sbin/... /usr/lib/postfix/sbin/...` | Le posizioni consuete al di fuori del PATH (RHEL o Debian/Ubuntu) |
| `2>/dev/null` | Sopprime i messaggi di errore di `ls` per percorsi inesistenti |

</details>

Se l'output rimane vuoto, manca anche il pacchetto associato. Sui sistemi RPM, confermarlo e verificare al contempo se i repository offrono Postfix:

```bash
rpm -qa | grep -i postfix
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-q` | Modalità di interrogazione di rpm |
| `-a` | Elenca tutti i pacchetti installati |
| `grep -i postfix` | Filtra l'elenco senza distinguere maiuscole e minuscole |

</details>

```bash
yum list available postfix
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `list available` | Mostra solo i pacchetti disponibili nei repository ma non installati |
| `postfix` | Limita l'output al pacchetto cercato |

</details>

Sul sistema di test non era installato alcun Postfix, ma il repository BaseOS offriva `postfix-3.5.8-8.el8_10` . La strada è quindi libera: il pacchetto può essere scaricato senza installarlo.

## Scaricare solo l'RPM

`yum download` (dal pacchetto plugin `dnf-plugins-core`, solitamente presente su RHEL 8) scarica un pacchetto nella directory corrente senza installarlo. Funziona senza privilegi di root, purché la directory di destinazione sia scrivibile:

```bash
cd /tmp && yum download postfix
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `cd /tmp` | Passa a una directory scrivibile; `yum download` salva l'RPM nella directory corrente |
| `download` | Sottocomando di `dnf-plugins-core`: scarica il pacchetto senza installarlo |
| `postfix` | Nome del pacchetto da scaricare |

</details>

Se yum segnala `No such command: download`, manca il plugin. Con privilegi di root si ottiene lo stesso risultato tramite il comando di installazione con `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--downloadonly` | Si interrompe dopo il download, senza installare nulla |
| `--downloaddir=/tmp` | Directory di destinazione per l'RPM scaricato |
| `postfix` | Nome del pacchetto |

</details>

In assenza di entrambe le opzioni, resta il passaggio tramite un secondo sistema con la stessa versione di RHEL: scaricare lì l'RPM e copiarlo sul sistema di destinazione con `scp`.

## Estrarre binari e librerie

`rpm2cpio` converte l'RPM in un flusso di archivio cpio, dal quale `cpio` estrae selettivamente singoli percorsi. Oltre ai due binari sono necessarie anche le librerie Postfix, poiché su RHEL gli strumenti sono collegati dinamicamente a `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `rpm2cpio postfix-*.rpm` | Converte l'RPM in un flusso di archivio cpio su stdout |
| `-i` | Modalità di estrazione cpio (copy-in) |
| `-d` | Crea le directory mancanti durante l'estrazione |
| `-m` | Mantiene le date di modifica dei file |
| `-v` | Elenca ogni file estratto |
| `./usr/sbin/smtp-source ./usr/sbin/smtp-sink` | I due binari, percorsi esattamente come nell'archivio (con `./` iniziale) |
| `'./usr/lib64/postfix/*'` | Le librerie Postfix; il modello è tra virgolette affinché venga interpretato da cpio e non dalla shell |

</details>

I file si trovano quindi sotto `/tmp/usr/`.

## Problema 1: /tmp è montato con noexec

L'avvio diretto da /tmp, apparentemente ovvio, fallisce sui sistemi rafforzati:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Il codice di uscita 126 nonostante il bit di esecuzione impostato correttamente è il quadro tipico di un filesystem con l'opzione di mount `noexec`. Il kernel rifiuta allora qualsiasi esecuzione di programma da quel filesystem, indipendentemente dai permessi del file. È possibile verificarlo direttamente:

```bash
mount | grep ' /tmp '
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `mount` | Senza argomenti, elenca tutti i filesystem montati con le rispettive opzioni di mount |
| `' /tmp '` | Modello di ricerca con uno spazio prima e dopo, affinché corrisponda solo al mountpoint `/tmp` e non, ad esempio, a `/var/tmp` |

</details>

La soluzione: copiare binari e librerie in una directory il cui filesystem consenta l'esecuzione, ad esempio la propria home:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `mkdir -p ~/bin` | Crea la directory di destinazione; senza errori se esiste già |
| `cp ... ~/bin/` | Copia i due binari e le librerie `libpostfix-*.so` nella directory eseguibile |
| `chmod +x` | Imposta il bit di esecuzione su entrambi i binari |

</details>

Si noti che `noexec` riguarda anche il caricamento delle librerie condivise. Non basta quindi spostare solo i binari e lasciare le librerie in /tmp.

## Problema 2: il percorso delle librerie

Senza ulteriori indicazioni, il linker dinamico cerca le librerie Postfix in `/usr/lib64/postfix`, dove non si trovano in assenza di installazione:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` aggiunge la propria directory al percorso di ricerca del linker. La variabile viene anteposta a ogni chiamata:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `LD_LIBRARY_PATH=~/bin` | Aggiunge `~/bin` al percorso di ricerca del linker dinamico per questa singola chiamata |
| `~/bin/smtp-source` | Chiamata tramite percorso completo, poiché `~/bin` potrebbe non essere nel PATH |

</details>

Con `ldd ~/bin/smtp-source` è possibile verificare in anticipo se tutte le dipendenze sono risolvibili. Oltre alle librerie Postfix, gli strumenti dipendono solo dalle librerie standard del sistema.

## Test funzionale in loopback

È possibile verificare che tutto funzioni senza una sola email reale: `smtp-sink` ascolta come destinatario che scarta i messaggi su una porta alta, mentre `smtp-source` invia. Tutto il traffico rimane su localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-v` (smtp-sink) | Registra ogni fase del dialogo delle connessioni accettate |
| `127.0.0.1:2525` | smtp-sink ascolta solo su localhost, porta 2525 |
| `100` | Backlog: lunghezza massima della coda delle connessioni in attesa secondo listen(2) |
| `-s 2` | Due sessioni SMTP parallele |
| `-m 10` | Dieci messaggi in totale, distribuiti tra le sessioni |
| `-l 5120` | Dimensione del messaggio in byte (senza intestazione), qui 5 KB |
| `-f` / `-t` | Indirizzo del mittente e del destinatario |

</details>

In caso di successo, `smtp-source` non produce output, mentre smtp-sink visualizza il dialogo SMTP completo per ogni messaggio, da `HELO` a `QUIT`. Quindi terminare il processo in background e rimuovere i residui da /tmp:

```bash
kill %1
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `%1` | Specifica del job della shell: termina il primo job in background, qui smtp-sink |

</details>

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-r` | Rimuove ricorsivamente l'albero della directory |
| `-f` | Nessuna richiesta di conferma, nessun errore per percorsi inesistenti |
| `/tmp/usr /tmp/postfix-*.rpm` | L'albero estratto e l'RPM scaricato |

</details>

## Indicazioni per il test di carico reale

Per misurazioni affidabili del throughput, il generatore di carico deve trovarsi su una macchina separata nello stesso segmento di rete, non sull'oggetto del test stesso. Se `smtp-source` viene eseguito sul gateway sottoposto a verifica, generatore e sistema di posta competono per CPU e I/O, e la misurazione mostra questa concorrenza anziché la capacità effettiva. In locale sul sistema di destinazione, lo strumento estratto è adatto soprattutto per test funzionali del set di regole e per prime verifiche di plausibilità.

Non appena il test coinvolge la vera porta 25, si tratta di email reali che attraversano il set di regole del gateway e che, a seconda della configurazione, vengono consegnate. Utilizzare quindi indirizzi destinatari che terminano in modo controllato: una casella di test dedicata, una regola che scarta i mittenti di test oppure un dominio di scarto previsto dal provider a tale scopo. Gli indirizzi di produzione non devono essere usati in un test di carico.

La procedura descritta è adatta, oltre ai due strumenti SMTP, a qualsiasi programma da riga di comando fornito da un pacchetto la cui installazione sul sistema di destinazione non è un'opzione. La combinazione di `yum download`, `rpm2cpio` e una directory eseguibile nella home è identica su ogni sistema RPM.

## Fonti

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): pagina man con tutti i parametri del generatore di carico, inclusi il controllo delle sessioni e dei messaggi.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): pagina man del destinatario di test, con opzioni tra l'altro per ritardi artificiali e risposte di errore.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documenta `yum download` e l'alternativa `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): descrizione dell'opzione di mount `noexec` e del suo effetto sull'esecuzione dei programmi.
