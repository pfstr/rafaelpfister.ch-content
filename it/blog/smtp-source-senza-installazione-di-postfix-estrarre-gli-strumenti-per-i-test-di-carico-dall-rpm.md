---
title: "smtp-source senza installazione di Postfix: estrarre gli strumenti per i test di carico dall'RPM"
navTitle: "Estrarre smtp-source"
description: "smtp-source e smtp-sink fanno parte di Postfix, ma funzionano anche senza un mail server installato. Come estrarre i due strumenti dal pacchetto su RHEL, perché l'esecuzione da /tmp può fallire a causa dell'opzione di mount noexec e quali librerie devono essere incluse."
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
url: https://rafaelpfister.ch/it/blog/smtp-source-senza-installazione-di-postfix-estrarre-gli-strumenti-per-i-test-di-carico-dall-rpm
translationSourceHash: 2b4bda3ea22f49c9d5269ec15b0c1dbfd779ccc6d03ad5b234aba738e5bb119f
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:24:04.730Z
translationReview: automatic
---

# smtp-source senza installazione di Postfix: estrarre gli strumenti per i test di carico dall'RPM

Per i test di carico SMTP, `smtp-source` è una buona scelta: lo strumento apre sessioni parallele, le mantiene attive per più messaggi e riproduce quindi il comportamento di connessione di un mittente massivo in modo molto più realistico rispetto agli strumenti di test che stabiliscono una nuova connessione per ogni mail. La controparte `smtp-sink` accetta le mail e le scarta senza consegnarle. Entrambi fanno parte della distribuzione di Postfix.

Ed è proprio qui il problema: spesso sul sistema dal quale si desidera eseguire i test Postfix non è installato. Su un'appliance mail gateway, inoltre, l'installazione non è auspicabile, poiché un Postfix aggiuntivo porta con sé una propria configurazione in `/etc/postfix` e un servizio di sistema che, nel peggiore dei casi, occupa la porta 25 bloccando il sistema di posta vero e proprio. A questo si aggiunge la questione di cosa pensi il supporto del produttore dei pacchetti installati successivamente sulla propria appliance.

Tuttavia, entrambi gli strumenti possono essere usati anche senza installazione: scaricare l'RPM, estrarre binari e librerie, fatto. Il percorso presenta due particolarità, illustrate in questo articolo su un sistema RHEL 8. Non sono necessari privilegi root, ma solo l'accesso alle fonti dei pacchetti.

## smtp-source è già disponibile?

Per prima cosa, verificate se lo strumento non sia già presente sul sistema. `smtp-source` si trova, a seconda della distribuzione, al di fuori del normale PATH:

```bash
command -v smtp-source || \
  ls /usr/sbin/smtp-source /usr/lib/postfix/sbin/smtp-source 2>/dev/null
```

Se l'output resta vuoto, manca anche il pacchetto associato. Sui sistemi RPM potete confermarlo e verificare al contempo se i repository offrono Postfix:

```bash
rpm -qa | grep -i postfix
```

```bash
yum list available postfix
```

Sul sistema di test non era installato Postfix, ma il repository BaseOS offriva `postfix-3.5.8-8.el8_10` . La strada è quindi libera: il pacchetto può essere scaricato senza installarlo.

## Scaricare soltanto l'RPM

`yum download` (dal pacchetto plugin `dnf-plugins-core`, normalmente presente su RHEL 8) scarica un pacchetto nella directory corrente senza installarlo. Funziona senza privilegi root, purché la directory di destinazione sia scrivibile:

```bash
cd /tmp && yum download postfix
```

Se yum segnala `No such command: download`, manca il plugin. Con privilegi root potete ottenere lo stesso risultato tramite il comando di installazione con `--downloadonly`:

```bash
sudo yum install --downloadonly --downloaddir=/tmp postfix
```

In assenza di entrambe le possibilità, resta il passaggio tramite un secondo sistema con la stessa versione RHEL: scaricare lì l'RPM e copiarlo sul sistema di destinazione tramite `scp`.

## Estrarre binari e librerie

`rpm2cpio` converte l'RPM in un flusso di archivio cpio, dal quale `cpio` estrae selettivamente singoli percorsi. Oltre ai due binari, sono necessarie anche le librerie Postfix, poiché su RHEL gli strumenti sono collegati dinamicamente a `libpostfix-*.so`:

```bash
cd /tmp && rpm2cpio postfix-*.rpm | \
  cpio -idmv ./usr/sbin/smtp-source ./usr/sbin/smtp-sink \
  './usr/lib64/postfix/*'
```

I file si trovano quindi sotto `/tmp/usr/`. Le indicazioni di percorso iniziano con `./`, perché cpio si aspetta i percorsi esattamente come sono presenti nell'archivio.

## Problema 1: /tmp è montato con noexec

L'avvio apparentemente ovvio direttamente da /tmp fallisce sui sistemi rafforzati:

```text
bash: /tmp/usr/sbin/smtp-sink: Permission denied
[1]+  Exit 126
```

Il codice di uscita 126 nonostante il bit di esecuzione impostato correttamente è il comportamento tipico di un filesystem con l'opzione di mount `noexec`. Il kernel rifiuta quindi qualsiasi esecuzione di programmi da quel filesystem, indipendentemente dai permessi del file. Potete verificarlo direttamente:

```bash
mount | grep ' /tmp '
```

La soluzione: copiare binari e librerie in una directory il cui filesystem consenta l'esecuzione, ad esempio nella propria home:

```bash
mkdir -p ~/bin && \
  cp /tmp/usr/sbin/smtp-source /tmp/usr/sbin/smtp-sink \
     /tmp/usr/lib64/postfix/libpostfix-*.so ~/bin/ && \
  chmod +x ~/bin/smtp-source ~/bin/smtp-sink
```

Tenete presente che `noexec` interessa anche il caricamento delle shared library. Non basta quindi spostare soltanto i binari e lasciare le librerie in /tmp.

## Problema 2: il percorso delle librerie

Senza ulteriori indicazioni, il linker dinamico cerca le librerie Postfix in `/usr/lib64/postfix`, dove non si trovano poiché non è stata eseguita l'installazione:

```text
smtp-sink: error while loading shared libraries: libpostfix-global.so:
cannot open shared object file: No such file or directory
```

`LD_LIBRARY_PATH` aggiunge la directory personale al percorso di ricerca del linker. La variabile viene anteposta a ogni chiamata:

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source ...
```

Con `ldd ~/bin/smtp-source` potete verificare in anticipo se tutte le dipendenze sono risolvibili. Oltre alle librerie Postfix, gli strumenti dipendono soltanto dalle librerie standard del sistema.

## Test funzionale in loopback

Potete verificare che tutto funzioni senza inviare una sola mail reale: `smtp-sink` resta in ascolto come destinatario usa e getta su una porta alta, mentre `smtp-source` consegna verso di esso. Tutto il traffico rimane su localhost.

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-sink -v 127.0.0.1:2525 100 &
```

```bash
LD_LIBRARY_PATH=~/bin ~/bin/smtp-source -s 2 -m 10 -l 5120 \
  -f test@example.com -t test@example.com 127.0.0.1:2525
```

| Opzione | Effetto |
|---|---|
| `-v` (smtp-sink) | Registra ogni fase del dialogo delle connessioni accettate |
| `127.0.0.1:2525` | smtp-sink resta in ascolto solo su localhost, porta 2525 |
| `100` | Backlog: lunghezza massima della coda delle connessioni in attesa secondo listen(2) |
| `-s 2` | Due sessioni SMTP parallele |
| `-m 10` | Dieci messaggi complessivi, distribuiti tra le sessioni |
| `-l 5120` | Dimensione del messaggio in byte (senza header), qui 5 KB |
| `-f` / `-t` | Indirizzo del mittente e del destinatario |

In caso di successo, `smtp-source` non produce alcun output, mentre smtp-sink emette per ogni messaggio il dialogo SMTP completo da `HELO` fino a `QUIT`. Quindi terminate il processo in background e rimuovete i residui da /tmp:

```bash
kill %1
```

```bash
rm -rf /tmp/usr /tmp/postfix-*.rpm
```

## Indicazioni per il vero test di carico

Per misurazioni di throughput affidabili, il generatore di carico deve trovarsi su una macchina separata nello stesso segmento di rete, non sull'oggetto di test stesso. Se `smtp-source` viene eseguito sul gateway verificato, generatore e sistema di posta competono per CPU e I/O, e la misurazione mostra questa concorrenza anziché la capacità effettiva. Localmente sul sistema di destinazione, lo strumento estratto è utile soprattutto per test funzionali del set di regole e per prime verifiche di plausibilità.

Non appena il test riguarda la vera porta 25, si tratta di mail reali che attraversano il set di regole del gateway e che, a seconda della configurazione, vengono consegnate. Utilizzate pertanto indirizzi di destinatari che terminino in modo controllato: una casella di test dedicata, una regola che scarti i mittenti di test oppure un dominio di scarto predisposto dal provider. Gli indirizzi di produzione non devono essere usati in un test di carico.

La procedura descritta è adatta, oltre che ai due strumenti SMTP, a qualsiasi programma da riga di comando fornito in un pacchetto la cui installazione sul sistema di destinazione non sia un'opzione. La combinazione di `yum download`, `rpm2cpio` e una directory eseguibile nella home è identica su ogni sistema RPM.

## Fonti

1.  [Postfix: smtp-source(1)](https://www.postfix.org/smtp-source.1.html): pagina di manuale con tutti i parametri del generatore di carico, incluso il controllo delle sessioni e dei messaggi.

2.  [Postfix: smtp-sink(1)](https://www.postfix.org/smtp-sink.1.html): pagina di manuale del destinatario di test, incluse opzioni per ritardi artificiali e risposte di errore.

3.  [Red Hat: How to download a package without installing it](https://access.redhat.com/solutions/10154): documenta `yum download` e l'alternativa `--downloadonly`.

4.  [man7.org: mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html): descrizione dell'opzione di mount `noexec` e del suo effetto sull'esecuzione dei programmi.
