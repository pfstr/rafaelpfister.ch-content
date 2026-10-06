---
title: "Rinnovo del certificato Totemomail: chiave a 4096 bit, importazione PKCS#12 e riavvio per nodo"
navTitle: "Rinnovare il certificato"
description: "La finestra di richiesta di Totemomail genera solo chiavi a 2048 bit senza nomi alternativi. Tuttavia, molte CA interne firmano ormai solo chiavi a 4096 bit. La chiave e la richiesta vengono quindi create con openssl, poi seguono l'ordine presso la PKI, l'importazione PKCS#12, l'associazione alle porte e il riavvio per ciascun nodo."
date: "2026-10-06"
kategorie: "Totemomail"
timeToRead: "12 min di lettura"
themen:
  - totemomail
  - e-mail-verschluesselung
produkte:
  - "totemomail"
protokolle:
  - "tls"
  - "smtp"
slug: "rinnovo-del-certificato-totemomail-chiave-a-4096-bit-importazione-pkcs-12-e-riavvio-per-nodo"
translationId: "article-1e59c4ee01e408a3"
translationOf: totemomail-zertifikat-erneuern
url: https://rafaelpfister.ch/it/blog/rinnovo-del-certificato-totemomail-chiave-a-4096-bit-importazione-pkcs-12-e-riavvio-per-nodo
translationSourceHash: 3ba1992d93abcd33fda47f86cb3b1ea4c8884c36fcfa41fa5c098f4aff9dff32
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T10:32:53.590Z
translationReview: automatic
---

# Rinnovo del certificato Totemomail: chiave a 4096 bit, importazione PKCS#12 e riavvio per nodo

Rinnovare il certificato server di un cluster Totemomail (oggi Kiteworks Email Protection Gateway) sembra un'attività di routine: generare la richiesta nell'interfaccia di amministrazione, farla firmare, importare la risposta. Nella pratica, questa procedura fallisce spesso già al primo passaggio. La finestra «New PKCS#10» genera invariabilmente una chiave a 2048 bit e non offre un campo per i nomi alternativi (Subject Alternative Names). Molte autorità di certificazione interne firmano ormai solo chiavi a 4096 bit e le controparti TLS attuali non riconoscono nemmeno un certificato senza nomi alternativi come valido per un nome.

La procedura seguente si è dimostrata valida durante una sostituzione nell'autunno 2026 in preproduzione e produzione: chiave e richiesta con openssl sul gateway, ordine presso il servizio PKI, importazione come PKCS#12, associazione alle porte e riavvio nodo per nodo. Durante la sostituzione sono emerse diverse particolarità, tra cui un errore nell'interfaccia che si verifica quando si scollega il vecchio certificato.

## Due certificati con finalità diverse

Un cluster Totemomail dietro Exchange Online necessita di norma di due tipi di certificati.

| Tipo | Finalità | Emittente | Durata |
|---|---|---|---|
| Interno | Interfaccia web, amministrazione, connessioni SMTP interne. Contiene i nomi interni dei nodi. | PKI interna, ad esempio Active Directory Certificate Services | selezionabile liberamente, tipicamente da 12 a 13 mesi |
| Pubblico | Collegamento tra Exchange Online e il gateway, quando Exchange Online deve verificare il certificato | autorità di certificazione pubblica | dal 15.03.2026 al massimo 200 giorni, dal 15.03.2027 al massimo 100 giorni |

Un unico certificato per entrambe le finalità non funziona. Le autorità di certificazione pubbliche non emettono nomi interni né nomi brevi senza dominio, e ogni certificato pubblico compare nei registri di Certificate Transparency. Viceversa, Exchange Online non considera attendibile alcuna catena interna. L'articolo [Loop di posta con gateway di crittografia dietro EXO](https://rafaelpfister.ch/blog/verschluesselungsgateway-hinter-exchange-online) descrive come Exchange Online gestisce il collegamento verso un gateway di crittografia.

I passaggi seguenti si applicano al certificato interno. Per quello pubblico, la procedura fino all'importazione è identica; le differenze sono riportate nell'ultima sezione.

## Panoramica della procedura

1. Generare chiave e richiesta con openssl su un nodo.
2. Ordinare la richiesta presso il servizio PKI.
3. Verificare il certificato consegnato.
4. Riunire certificato, chiave e certificato intermedio in un file PKCS#12 e trasferirlo sul computer con il browser.
5. Importarlo nell'interfaccia Totemomail e associarlo alle porte.
6. Riavviare e verificare ogni nodo singolarmente.
7. Testare il flusso di posta, quindi eseguire la pulizia.

Pianificate almeno tre settimane prima della scadenza. Finché il vecchio certificato è valido, è possibile tornare indietro. Se nel vostro ambiente è disponibile una preproduzione, eseguite prima la sostituzione lì e usate l'esecuzione come riferimento.

## Passaggio 1: chiave e richiesta con openssl

Lavorate su un nodo del cluster con il vostro utente personale, non con l'account di servizio `totemo`. In questo modo potrete successivamente trasferire direttamente il file PKCS#12 tramite `scp`. Per tutti i passaggi non sono necessari privilegi di root.

```bash
umask 077
mkdir -m 700 ~/csr-2026
cd ~/csr-2026
```

La configurazione contiene titolare, scopi di utilizzo e tutti i nomi alternativi. I nomi nell'esempio sono segnaposto: tre nodi e i nomi di servizio con cui il cluster viene raggiunto internamente.

```bash
cat > intern.cnf <<'EOF'
[ req ]
default_md         = sha256
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
C  = CH
O  = Beispiel AG
CN = SecureMail

[ ext ]
basicConstraints = critical, CA:FALSE
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName   = @alt

[ alt ]
DNS.1 = gw01.intern.example.ch
DNS.2 = gw02.intern.example.ch
DNS.3 = gw03.intern.example.ch
DNS.4 = securemail.intern.example.ch
EOF
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Voce | Effetto |
|---|---|
| `default_md = sha256` | Algoritmo hash per la firma della richiesta |
| `prompt = no` | Acquisire i valori dal file anziché richiederli in modo interattivo |
| `req_extensions = ext` | Scrivere nella richiesta le estensioni dalla sezione `[ ext ]` |
| `basicConstraints = critical, CA:FALSE` | Certificato finale, non un'autorità di certificazione |
| `keyUsage` | Chiave per firma e scambio di chiavi, come consueto per i server TLS |
| `extendedKeyUsage = serverAuth, clientAuth` | Autenticazione server e client. Il gateway è server su alcuni collegamenti e client su altri. |
| `subjectAltName = @alt` | Nomi alternativi dalla sezione `[ alt ]` |

</details>

Includete solo nomi completi. Molte autorità di registrazione rifiutano i nomi brevi senza dominio e non dovreste fare affidamento sul fatto che vengano inclusi nel certificato.

Generate la chiave senza passphrase e proteggetela tramite i permessi del file. Rimane sul nodo solo fino all'importazione e viene poi eliminata. Una passphrase su una chiave che esiste per pochi giorni offre poca protezione e crea un nuovo rischio: se viene persa, la chiave è inutilizzabile e il certificato deve essere emesso nuovamente.

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out intern.key
openssl req -new -key intern.key -config intern.cnf -out intern.csr
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `genpkey -algorithm RSA` | Generare una nuova chiave privata RSA |
| `-pkeyopt rsa_keygen_bits:4096` | Lunghezza della chiave: 4096 bit |
| `-out intern.key` | File della chiave, senza passphrase |
| `req -new` | Generare una nuova richiesta di certificato (CSR) |
| `-key intern.key` | Utilizzare la chiave esistente |
| `-config intern.cnf` | Titolare, estensioni e nomi dalla configurazione |
| `-out intern.csr` | File della richiesta |

</details>

Verificate la richiesta prima che lasci la macchina:

```bash
openssl req -in intern.csr -noout -verify -subject
openssl req -in intern.csr -noout -text | grep -E "Public-Key|Signature Algorithm" | head -2
openssl req -in intern.csr -noout -text | grep -o "DNS:[^,]*" | wc -l
```

Sono previsti `verify OK`, il titolare corretto, `4096 bit` e il numero dei vostri nomi. Annotate inoltre l'impronta digitale della chiave pubblica. In seguito vi consentirà di associare in modo inequivocabile il certificato consegnato a questa chiave, anche se due richieste hanno lo stesso titolare:

```bash
openssl req -in intern.csr -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
```

## Passaggio 2: ordine presso il servizio PKI

L'ente che emette il certificato necessita di queste informazioni per ogni richiesta:

- il CSR come testo
- l'impronta SHA-256 della chiave pubblica
- il tipo: interno o pubblico e, in caso di più ambienti, quale
- l'elenco dei nomi da copiare, un nome per riga
- lunghezza della chiave 4096, uso esteso Server Authentication e Client Authentication
- la data entro cui il certificato deve essere disponibile, almeno due settimane prima della scadenza

In una sostituzione con più ambienti e tipi di certificato, conviene fornire una panoramica all'inizio dell'e-mail: quali certificati esistono, quali vengono emessi internamente e quali pubblicamente, e in quale ordine sono necessari. Per l'emissione non è necessaria alcuna modifica al sistema. I certificati di tutti gli ambienti possono quindi essere emessi contemporaneamente, anche se vengono installati in sequenza.

Durante la sostituzione nell'autunno 2026, presso l'autorità di registrazione (RA) sono emersi diversi aspetti che probabilmente si verificano in modo simile in molti ambienti:

- **La RA imposta autonomamente il titolare.** Nel certificato erano presenti solo paese, organizzazione e Common Name, anche se la richiesta conteneva unità organizzativa, località e cantone.
- **Nessun nome breve.** I nomi senza dominio mancavano nel certificato emesso.
- **Nomi riportati manualmente.** La RA non ha acquisito i nomi alternativi dal CSR; sono stati digitati nella maschera. Un nome è arrivato troncato. L'elenco dei nomi consegnato va quindi sempre verificato.
- **Profilo errato.** Una RA che gestisce sia la CA interna sia una CA pubblica ha inizialmente emesso le richieste di certificati pubblici tramite il profilo interno. Come emittente figurava la CA interna. Tali certificati sono inutili per il collegamento a Exchange Online.
- **Un solo nome nei certificati pubblici a dominio singolo.** Un prodotto Single-Domain si interrompe con una richiesta contenente due nomi, ad esempio con «Only one Subject Alternative Name is allowed». Tuttavia, un solo nome è sufficiente; vedere la sezione sul certificato pubblico.

La PKI dovrebbe in seguito revocare i certificati derivanti da emissioni errate. Contengono una chiave valida e altrimenti restano validi per un anno.

## Passaggio 3: verificare la consegna

Salvate il certificato consegnato nella stessa directory della chiave, ad esempio con `cat > intern.crt`, incollate il contenuto, quindi `Strg+D`. Poi verificate:

```bash
openssl x509 -in intern.crt -noout -subject -issuer -serial -dates
openssl x509 -in intern.crt -noout -ext subjectAltName,extendedKeyUsage,keyUsage
```

Controllate l'emittente, l'elenco completo dei nomi senza errori di battitura o duplicati e i due scopi di utilizzo. Il confronto delle impronte digitali indica se certificato e chiave appartengono insieme. I due valori devono essere uguali:

```bash
openssl x509 -in intern.crt -noout -pubkey |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256
openssl pkey -in intern.key -pubout -outform DER |
  openssl dgst -sha256
```

Per un certificato pubblico si aggiungono due verifiche. L'OID dei criteri `2.23.140.1.2.2` identifica un certificato convalidato dall'organizzazione secondo le regole del CA/Browser Forum e il certificato deve contenere prove Certificate Transparency incorporate. Pochi minuti dopo l'emissione compare su crt.sh con il suo nome. Se manca uno dei due elementi, non si tratta di un certificato pubblico, indipendentemente dall'etichetta della consegna.

## Passaggio 4: creare il file PKCS#12

Totemomail importa certificato e chiave insieme come PKCS#12. A questo scopo, recuperate il certificato dell'autorità intermedia emittente. L'indirizzo è riportato nel certificato sotto `Authority Information Access`:

```bash
openssl x509 -in intern.crt -noout -ext authorityInfoAccess
curl -sS -o issuing.crt http://pki.example.ch/crt/Issuing-CA.crt
file issuing.crt
```

Se `file` non segnala `PEM certificate`, il file è codificato DER e deve essere convertito:

```bash
openssl x509 -inform DER -in issuing.crt -out issuing.pem
mv issuing.pem issuing.crt
```

Il Subject Key Identifier dell'autorità intermedia deve corrispondere all'Authority Key Identifier del certificato. Poi create il file. Il comando richiede una password di esportazione, necessaria per l'importazione:

```bash
openssl pkcs12 -export \
  -inkey intern.key \
  -in intern.crt \
  -certfile issuing.crt \
  -name "SecureMail" \
  -out intern.p12
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-export` | Creare il file PKCS#12 |
| `-inkey intern.key` | Chiave privata |
| `-in intern.crt` | Certificato emesso |
| `-certfile issuing.crt` | Altri certificati della catena, qui l'intermedio |
| `-name "SecureMail"` | Nome visualizzato della voce nel file |
| `-out intern.p12` | File di destinazione, protetto dalla password di esportazione |

</details>

Verifica: devono comparire due voci, il certificato e l'intermedio:

```bash
openssl pkcs12 -in intern.p12 -nokeys 2>/dev/null | grep -E "subject=|issuer="
```

L'interfaccia Totemomail viene eseguita nel browser, tipicamente su un jump host. Il file deve essere trasferito lì. In Windows il client OpenSSH con `scp` è disponibile:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\Downloads\zert" -Force | Out-Null
scp benutzer@gw01.intern.example.ch:csr-2026/intern.p12 "$env:USERPROFILE\Downloads\zert\"
scp benutzer@gw01.intern.example.ch:csr-2026/issuing.crt "$env:USERPROFILE\Downloads\zert\"
```

Se avete invece generato la chiave come `totemo`, si trova in `/opt/totemomail` e il vostro utente non può leggerla. Copiate allora brevemente il file PKCS#12 in `/tmp`, recuperatelo da lì e cancellatelo immediatamente. Il passaggio tramite gli appunti con Base64 funziona, ma con una riga di circa 10'000 caratteri è soggetto a errori.

## Passaggio 5: importare e associare in Totemomail

In `Key Management`:

1. **`Issuer Certificates`**: importare l'autorità intermedia `issuing.crt`.
2. **`Own Server Certificates`**, pulsante **`Import`**: la finestra offre due modalità. A sinistra, «Import certificate» serve per un certificato con chiave, quindi per il file PKCS#12. A destra, «Import a PKCS#10 certificate reply» serve solo per le risposte a richieste generate da Totemomail stesso. Scegliete la modalità a sinistra e inserite la password di esportazione al secondo passaggio.
3. Aprite il nuovo certificato (icona a forma di matita). In `Connector` tutte le porte dovrebbero essere selezionate; nell'installazione descritta `8443=Admin`, `443=SecMail`, `7444=MailAPI`, `10443=SENDIT` e `8444=AdminAPI`. `Host` è impostato su `*`.
4. Aprite il vecchio certificato e deselezionate tutte le porte tranne una.

Al punto 4 si riferisce l'errore verificatosi durante la sostituzione: se si deselezionano **tutte** le porte del vecchio certificato, l'interfaccia segnala «Could not edit selected server certificate». Nel log compare:

```
ERROR [EditServerCertBean] Could not edit certificate
ch.totemo.core.actions.ActionException: Failed to edit key.
Caused by: java.lang.NullPointerException
```

Totemomail non gestisce un certificato server senza porta. Lasciate quindi una porta selezionata, ad esempio `8444=AdminAPI`, e cancellate il vecchio certificato dopo il periodo di osservazione. La cancellazione anziché la deselezione funziona; il certificato finisce in `Deleted Certificates`.

Per comprendere l'associazione sono utili tre osservazioni:

- **L'elenco delle porte vale solo per i servizi web.** La porta 25 non vi compare. Se non è presente alcun certificato di tipo SMTPS, SMTP utilizza il certificato HTTPS. Per questo la porta 25 mostra il nuovo certificato dopo la sostituzione, anche se nell'elenco SMTPS non è selezionata.
- **Un'associazione esplicita ha la precedenza sull'asterisco.** La porta lasciata sul vecchio certificato continua a mostrare quello vecchio, anche se il nuovo è registrato per tutti con `*`.
- **L'elenco è condiviso per l'intero cluster.** Appare uguale su ogni nodo, indipendentemente dal fatto che il nodo abbia già applicato la modifica. Solo una verifica sulle porte mostra ciò che un nodo presenta effettivamente.

## Passaggio 6: riavvio nodo per nodo

Totemomail carica i certificati all'avvio. Dopo l'importazione, tutti i nodi continuano a mostrare il vecchio certificato finché non vengono riavviati. Riavviate ogni nodo singolarmente, mai tutti insieme, affinché dietro il load balancer restino sempre nodi attivi. Iniziate dai nodi sui quali non avete effettuato l'accesso e lasciate per ultimo il nodo con l'interfaccia aperta.

Sul nodo come `totemo`:

```bash
totemomail stop
totemomail start
```

Ulteriori indicazioni sull'arresto controllato sono disponibili nell'articolo [I controlli più importanti per gli amministratori Totemomail](https://rafaelpfister.ch/blog/totemomail-server-stoppen-queues-bereinigen).

I servizi web sono di nuovo disponibili dopo circa un minuto, mentre il servizio SMTP sulla porta 25 solo dopo alcuni minuti. Una risposta vuota sulla porta 25 subito dopo l'avvio non indica quindi ancora un errore. Solo quando un nodo è passato al nuovo certificato su tutte le porte si procede con il successivo.

La verifica su tutti i nodi e le porte:

```bash
for h in gw01 gw02 gw03; do
  for p in 25 443 8443; do
    if [ "$p" = 25 ]; then s="-starttls smtp"; else s=""; fi
    c=$(echo | openssl s_client -connect "$h:$p" $s 2>/dev/null |
        openssl x509 -noout -serial -enddate 2>/dev/null | tr '\n' ' ')
    printf "%-6s %-5s %s\n" "$h" "$p" "$c"
  done
done
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `s_client -connect host:port` | Stabilire una connessione TLS al servizio |
| `-starttls smtp` | Sulla porta 25 eseguire prima il dialogo SMTP, quindi passare a TLS con STARTTLS |
| `x509 -noout` | Leggere il certificato senza visualizzarlo |
| `-serial -enddate` | Visualizzare numero di serie e data di scadenza |

</details>

Se la porta 25 rimane vuota dopo cinque minuti, `ss -lnt | grep ':25 '` indica se il servizio è in ascolto e il log in `/opt/totemomail` ne riporta il motivo.

L'opzione `-showcerts` indica se Totemomail invia anche l'autorità intermedia:

```bash
echo | openssl s_client -connect gw01:25 -starttls smtp -showcerts 2>/dev/null | grep -E " s:| i:"
```

Durante la sostituzione descritta appariva solo il certificato finale, anche se l'intermedio era nel file PKCS#12 e in `Issuer Certificates`. Per i collegamenti interni sui quali nessuno verifica il certificato, ciò non ha conseguenze. Per la verifica da parte di Exchange Online, invece, la catena deve essere completa.

## Passaggio 7: test, ritorno indietro, pulizia

Dopo l'ultimo riavvio, testate il flusso di posta in entrambe le direzioni: un messaggio dall'esterno attraverso il loop e uno verso l'esterno tramite il gateway. Nella traccia messaggi di Exchange Online entrambi i percorsi devono risultare consegnati e verso il gateway non deve rimanere nulla nella coda.

Il **ritorno indietro** consiste nel selezionare nuovamente le porte del vecchio certificato, deselezionare tutte tranne una sul nuovo e riavviare ancora una volta i nodi singolarmente. È possibile solo finché il vecchio certificato è valido.

Dopo test riusciti:

- cancellare il file PKCS#12 sul jump host
- cancellare la directory di lavoro sul nodo; la chiave ora si trova nel keystore di Totemomail
- dopo alcuni giorni cancellare il vecchio certificato e riavviare ancora una volta i nodi singolarmente, affinché anche l'ultima porta passi al nuovo certificato
- far revocare le emissioni errate presso il servizio PKI
- annotare la nuova data di scadenza nel promemoria

Annunciate la sostituzione agli amministratori se il nuovo certificato non contiene più nomi brevi. Chi ha finora aperto l'interfaccia con `https://gw01:8443` visualizzerà successivamente un avviso relativo al certificato. Lo stesso vale per monitoraggi e script che usano nomi brevi.

## Particolarità in sintesi

| Osservazione | Conseguenza | Gestione |
|---|---|---|
| «New PKCS#10» genera 2048 bit senza nomi alternativi | Richiesta inutilizzabile per CA a 4096 bit | Generare chiave e richiesta con openssl |
| Le modifiche hanno effetto solo dopo il riavvio | I nodi continuano a mostrare il vecchio certificato | Riavviare ogni nodo singolarmente |
| La porta 25 si attiva alcuni minuti dopo i servizi web | Risposta vuota subito dopo l'avvio | Attendere, poi procedere con il nodo successivo |
| Un certificato senza porta genera una NullPointerException | Non è possibile scollegare completamente il vecchio certificato | Lasciare una porta selezionata, poi cancellare |
| Un'associazione esplicita ha la precedenza su `*` | Una porta continua a mostrare il vecchio certificato | Cancellare il vecchio certificato dopo l'osservazione |
| Viene inviato solo il certificato finale | Le controparti che verificano non possono costruire la catena | Chiarire prima di una verifica da parte di Exchange Online |
| La RA acquisisce i nomi manualmente | Possibili errori di battitura e nomi mancanti | Verificare l'elenco dei nomi della consegna |

## Il certificato pubblico per il collegamento a Exchange Online

Affinché Exchange Online verifichi il collegamento al gateway oltre a cifrarlo, il gateway necessita di un certificato pubblico sulla porta 25. Solo allora il connettore in uscita può essere configurato con `TlsSettings DomainValidation` e un `TlsDomain`, mentre il connettore in entrata può essere associato al certificato tramite `TlsSenderCertificateName`.

A questo scopo è sufficiente un solo nome. Exchange Online confronta in entrambe le direzioni solo il nome nel certificato con il valore configurato, senza verificare l'indirizzo sottostante. Un certificato per il nome con cui il gateway è noto esternamente copre il percorso di andata e ritorno. Al momento dell'ordine dovreste richiedere esplicitamente Client Authentication: diverse autorità di certificazione pubbliche hanno rimosso questo utilizzo dai certificati TLS nel 2026. È necessario per il percorso di ritorno, nel quale il gateway si identifica come client.

Per l'utilizzo in Totemomail, in base alle osservazioni sopra riportate, si delinea questa procedura: importare il certificato pubblico come tipo SMTPS, affinché la porta 25 lo presenti e i servizi web mantengano quello interno. Prima devono essere chiariti due aspetti. La catena deve essere inviata completa e la porta 25 presenterà poi il certificato pubblico a tutti i mittenti, inclusi quelli interni come un gateway a monte. Se un mittente si aspetta un certificato specifico, questo collegamento si interrompe.

## Fonti

1.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): tabella di marcia per la durata dei certificati TLS pubblici, 200 giorni da marzo 2026, 100 giorni da marzo 2027, 47 giorni da marzo 2029.

2.  [Microsoft Learn: Set-OutboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-outboundconnector): parametri `TlsSettings` e `TlsDomain` per la verifica del certificato della controparte.

3.  [Microsoft Learn: Set-InboundConnector](https://learn.microsoft.com/powershell/module/exchange/set-inboundconnector): parametro `TlsSenderCertificateName` per l'associazione tramite il certificato del mittente.

4.  [Documentazione OpenSSL: openssl-req](https://docs.openssl.org/master/man1/openssl-req/): struttura del file di configurazione e opzioni per le richieste di certificato.

5.  [Documentazione OpenSSL: openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): creazione e verifica dei file PKCS#12.

6.  [crt.sh](https://crt.sh): ricerca nei registri di Certificate Transparency per verificare l'emissione di un certificato pubblico.
