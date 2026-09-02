---
title: "Rinnovare il certificato sulla Cisco SMA"
navTitle: "Certificato SMA"
description: "Sulla Cisco SMA i certificati possono essere installati solo tramite CLI e le versioni attuali di AsyncOS convalidano l'intera catena durante l'importazione: senza una Root CA archiviata, l'operazione fallisce. L'articolo illustra le modalità per ottenere una nuova coppia di chiavi, il metodo OpenSSL in dettaglio, come gestire l'errore RC2-40-CBC di OpenSSL 3 e l'importazione della Root CA interna nel truststore dell'appliance."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min di lettura"
themen:
  - cisco-esa-sma
  - smtp-mailflow
produkte:
  - "cisco"
protokolle:
  - "tls"
  - "smtp"
  - "ldap"
  - "troubleshooting"
hauptthema: "cisco-esa-sma"
slug: "rinnovare-il-certificato-sulla-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:08:21.121Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/rinnovare-il-certificato-sulla-cisco-sma
---

# Rinnovare il certificato sulla Cisco SMA

La Cisco SMA (Security Management Appliance, ora denominata Cisco Secure Email and Web Manager) gestisce in molti ambienti di posta elettronica la quarantena antispam centralizzata e il reporting per i Secure Email Gateway. Il suo certificato HTTPS copre la GUI di amministrazione e la pagina di quarantena, nella quale gli utenti finali consultano e rilasciano le e-mail trattenute. Quando scade, il flusso di posta non si interrompe. La scadenza diventa comunque subito visibile: ogni accesso alla pagina di quarantena genera un avviso relativo al certificato nel browser, e proprio gli utenti a cui i corsi di awareness insegnano a non proseguire in presenza di tali avvisi dovrebbero poi ignorarli.

Durante un rinnovo in un progetto cliente sono emersi subito due problemi: prima OpenSSL 3 ha risposto al file PFX della CA interna con un errore criptico relativo a `RC2-40-CBC`, poi l'appliance ha rifiutato l'importazione del certificato pronto perché non conosceva la Root CA emittente. Entrambi gli ostacoli e le relative soluzioni sono descritti più avanti.

## Cosa fa la SMA diversamente dalla ESA

Sulla ESA l'intero ciclo di vita del certificato può essere gestito tramite GUI (`Network > Certificates`). La SMA non può farlo: il certificato server viene installato esclusivamente tramite CLI, con il comando `certconfig` in una sessione SSH. La GUI della SMA mostra soltanto i certificati; qui è possibile gestire solo gli elenchi delle autorità di certificazione attendibili, come vedremo più avanti.

A questo si aggiungono altre due peculiarità:

- La finestra di incollaggio accetta solo il formato PEM. Un file PFX (PKCS#12) deve essere convertito prima dell'installazione; le versioni attuali di AsyncOS offrono inoltre un'importazione diretta di PKCS#12, ma il file deve prima essere trasferito sull'appliance.
- Le versioni AsyncOS meno recenti (quelle della Technote Cisco) non generano autonomamente né chiavi né CSR: la coppia di chiavi deve essere creata esternamente; i tre metodi praticabili sono illustrati più avanti. Le versioni attuali possono generare direttamente sull'appliance un certificato self-signed con CSR tramite `certconfig > CERTIFICATE > NEW`. Tuttavia, questo non aiuta per un certificato condiviso tra più appliance, poiché la chiave privata non lascia mai l'appliance.

Un singolo certificato può gestire tutti i servizi (TLS in entrata e in uscita, accesso HTTPS alla gestione, LDAPS) oppure essere archiviato separatamente per ciascun servizio. Questo viene gestito nella finestra di dialogo `certconfig`; l'intestazione del comando mostra in ogni momento l'assegnazione attiva (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Non esiste una schermata separata per l'assegnazione come sulla ESA e non è possibile modificarla dalla GUI. Nella maggior parte degli ambienti, un certificato per tutto è la scelta pragmatica: l'elenco dei nomi copre comunque gli FQDN delle appliance, mentre coppie di chiavi separate moltiplicano il lavoro a ogni rinnovo.

Il fatto che la finestra di dialogo su un'appliance di quarantena chieda informazioni sul TLS inbound e outbound può inizialmente confondere, poiché la SMA non si trova in un percorso MX. Tuttavia, comunica via SMTP in entrambe le direzioni. Inbound (Receiving) è il lato di ricezione: le ESA consegnano alla SMA i messaggi messi in quarantena via SMTP, alla quarantena antispam centralizzata sulla porta 6025 e alle quarantene centralizzate di policy, virus e outbreak sulla porta 7025; queste ultime connessioni sono crittografate TLS per impostazione predefinita e la SMA presenta proprio questo certificato. Outbound (Delivery) è il lato di invio: quando un utente rilascia un messaggio dalla quarantena, la SMA lo riconsegna autonomamente al flusso di posta tramite le proprie route SMTP; inoltre l'appliance invia come e-mail proprie le notifiche di quarantena, i report pianificati e gli avvisi. Per il rinnovo ciò significa che, nella pratica, HTTPS è critico; i due servizi SMTP sono semplicemente inclusi nel certificato per tutti i servizi.

## Definire i nomi: CN e SAN

Indipendentemente dal metodo scelto per la coppia di chiavi, si parte dalla lista dei nomi. Il Common Name deve corrispondere al nome host con cui gli utenti richiamano la pagina di quarantena. All'elenco SAN vanno aggiunti gli FQDN delle appliance affinché anche l'accesso diretto alla GUI di amministrazione funzioni senza avvisi. Per un ambiente con due appliance, la lista dei nomi è la seguente:

| Campo | Valore |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Due note in merito: da tempo i browser valutano solo le voci SAN; il solo CN non è sufficiente. Il nome host della quarantena deve quindi comparire anche come SAN. Inoltre, i nomi host brevi senza componente di dominio (ad esempio `SMA01`) vengono emessi solo da una CA interna; le CA pubbliche non firmano nomi interni.

## Tre modi per ottenere una nuova coppia di chiavi

Per un certificato che copra più appliance e il nome host della quarantena, la coppia di chiavi deve essere generata al di fuori dell'appliance. Si sono affermati tre approcci:

1. Generare chiave e CSR con OpenSSL all'interno del proprio ambiente. La chiave privata viene creata dove serve e non lascia mai l'ambiente. È il metodo consigliato, descritto in dettaglio nella sezione successiva.
2. La CA genera la coppia di chiavi e fornisce un file PFX. Funziona, ma presenta due svantaggi: la chiave passa attraverso mani esterne (la password deve quindi essere trasmessa tramite un canale separato e non nella stessa e-mail del file) e, a seconda dello strumento della CA, può essere restituito un PFX crittografato con RC2, che OpenSSL 3 apre solo con un lavoro aggiuntivo; maggiori dettagli più avanti.
3. Il passaggio tramite una ESA, documentato nella Technote Cisco: creare un certificato con il CN della SMA in `Network > Certificates`, scaricare la CSR e farla firmare dalla CA, ricaricare il certificato firmato sulla ESA ed esportare il tutto come PFX. Anche qui, alla fine, è necessaria la conversione in PEM.

## Le principali opzioni di openssl

Per orientarsi, ecco i sotto-comandi e le opzioni di `openssl` presenti in questo articolo, tradotti liberamente dalla documentazione OpenSSL:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Opzione | Significato |
|---|---|
| `req` | Sotto-comando per richieste di certificato (CSR): genera, visualizza, verifica |
| `-new` | Genera una nuova richiesta |
| `-newkey rsa:2048` | Genera una nuova coppia di chiavi RSA a 2048 bit |
| `-noenc` | Scrive la chiave privata non crittografata (fino a OpenSSL 3.0: `-nodes`) |
| `-keyout datei` | File di destinazione per la chiave privata |
| `-out datei` | File di destinazione per l'output, qui CSR o PEM |
| `-subj text` | Subject della richiesta nel formato `/C=…/O=…/CN=…` |
| `-addext text` | Aggiunge un'estensione alla richiesta, qui l'elenco SAN |
| `pkcs12` | Sotto-comando per contenitori PKCS#12 (PFX): crea ed estrae |
| `-in datei` | File di input |
| `-legacy` | Carica anche il provider Legacy per algoritmi meno recenti come RC2 |
| `list` | Sotto-comando per visualizzare le funzionalità dell'installazione |
| `-providers` | Elenca i provider caricati |
| `-provider name` | Carica inoltre il provider specificato per questa esecuzione |
| `s_client` | Sotto-comando: client di test TLS per connessioni a un server |
| `-connect host:port` | Host di destinazione e porta della connessione TLS |
| `-servername name` | Imposta la Server Name Indication (SNI) nell'handshake TLS |
| `x509` | Sotto-comando per visualizzare ed elaborare certificati |
| `-noout` | Sopprime l'output del certificato codificato |
| `-subject` | Mostra il Subject del certificato |
| `-enddate` | Mostra la data di scadenza (notAfter) |

</details>

La documentazione OpenSSL riporta i riferimenti completi come manpage separate per ciascun sotto-comando: `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` e `openssl-x509(1)`.

## Avviare OpenSSL su Windows

Tutti i passaggi successivi vengono eseguiti tramite OpenSSL, su un sistema interno all'ambiente, ad esempio un server di amministrazione. È sufficiente la Light Edition delle build Windows di Shining Light Productions; l'installer è grande circa 6 MB e può essere verificato rispetto all'elenco di checksum pubblicato da slproweb.

L'installer installa tutto in `C:\Program Files\OpenSSL-Win64`, mentre l'eseguibile si trova in `bin\openssl.exe`. Non viene aggiunto al percorso di ricerca: chi digita `openssl` in un prompt dei comandi appena aperto riceve un messaggio di errore. Esistono tre possibilità:

- Dal menu Start, aprire la voce `Win64 OpenSSL Command Prompt`. Avvia `start.bat` dalla directory di installazione, imposta l'ambiente e visualizza l'output di `openssl version -a`. In questa finestra `openssl` funziona direttamente.
- Specificare il percorso completo: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Aggiungere permanentemente `C:\Program Files\OpenSSL-Win64\bin` alla variabile d'ambiente `Path`; dopo di ciò `openssl` sarà disponibile in ogni shell.

Chi utilizza già Git per Windows non necessita di ulteriori installazioni: include il proprio OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), che è immediatamente presente nel percorso di ricerca in Git Bash. Le versioni attuali di Git includono OpenSSL 3.5 con provider Legacy attivo, quindi anche `-legacy` dalla sezione sulla conversione PFX funziona. È possibile verificarlo così:

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `list` | Mostra le funzionalità dell'installazione OpenSSL |
| `-providers` | Elenca i provider caricati con nome, versione e stato |
| `-provider legacy` | Carica inoltre il provider `legacy` per questa esecuzione; se compare nell'elenco, è disponibile |

</details>

Git Bash presenta tuttavia una particolarità: considera gli argomenti che iniziano con `/` come percorsi e li riscrive. Da `-subj "/C=CH/O=Example AG/CN=..."` diventa `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, e OpenSSL si interrompe:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Un prefisso `MSYS_NO_PATHCONV=1` disattiva la riscrittura per la singola esecuzione. Il problema non si presenta nel prompt dei comandi, in PowerShell e nell'OpenSSL Command Prompt.

## Generare chiave e CSR con OpenSSL

Una singola esecuzione genera chiave e CSR con l'intero elenco SAN:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-new` | Genera una nuova richiesta di certificato (CSR) |
| `-newkey rsa:2048` | Genera una nuova coppia di chiavi RSA a 2048 bit |
| `-noenc` | Scrive la chiave privata non crittografata nel file |
| `-keyout …` | File di destinazione per la chiave privata |
| `-out …` | File di destinazione per la CSR |
| `-subj …` | Subject con Paese, organizzazione e Common Name |
| `-addext …` | Aggiunge alla richiesta l'estensione SAN con tutti i nomi DNS |

</details>

Il file CSR viene inviato alla CA, mentre la chiave rimane sul server. Viene restituito il certificato firmato insieme all'intermediate, normalmente direttamente in formato PEM. In questo modo è tutto pronto per l'installazione e la conversione PFX non è necessaria.

Il file della chiave non è crittografato (`-noenc`), perché `certconfig` lo richiede esattamente in questa forma. Fino all'installazione, va mantenuto sul server con autorizzazioni restrittive; in seguito va eliminato o trasferito in un password manager.

## Convertire PFX in PEM

Questa e la sezione successiva riguardano i metodi 2 e 3, che terminano con un file PFX. `certconfig` richiede il certificato e la chiave privata in formato PEM, con la chiave non crittografata. Un'unica esecuzione di OpenSSL svolge entrambe le operazioni:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `pkcs12` | Sotto-comando per creare ed estrarre contenitori PKCS#12 |
| `-in …` | Il file PFX di input |
| `-out …` | Il file PEM di output con certificato, chiave e certificati della catena |
| `-noenc` | Scrive la chiave privata senza passphrase (fino a OpenSSL 3.0 l'opzione si chiamava `-nodes`) |

</details>

La password di importazione viene richiesta senza eco e non vengono visualizzati nemmeno asterischi. Il file PEM risultante contiene in un unico file certificato, chiave e certificati della catena forniti, pertanto deve essere protetto adeguatamente: dopo l'installazione, eliminarlo o trasferirlo nel password manager.

## Quando OpenSSL 3 rifiuta il file PFX

Con file PFX meno recenti, la conversione in OpenSSL 3.x si interrompe con questo messaggio:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La causa non è un file danneggiato, ma una scelta progettuale: OpenSSL 3 ha spostato algoritmi meno recenti come RC2, RC4 e DES in un provider Legacy separato, che non viene caricato per impostazione predefinita. Molte esportazioni PFX di sistemi Windows e strumenti CA meno recenti, però, cifrano la parte del certificato del contenitore proprio con RC2-40-CBC. OpenSSL 1.1 apriva questi file senza problemi, mentre OpenSSL 3 li rifiuta.

La soluzione consiste in una sola opzione aggiuntiva:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-legacy` | Carica il provider Legacy per questa esecuzione; così tornano disponibili algoritmi meno recenti come RC2-40-CBC e la conversione viene completata |

</details>

Il prerequisito è un'installazione OpenSSL che includa il provider Legacy; questo è il caso delle comuni build Windows.

Chi vuole eliminare definitivamente l'errore può intervenire alla fonte e far esportare il file PFX con crittografia moderna: le finestre di esportazione e gli strumenti CA attuali offrono AES-256, eliminando completamente il passaggio tramite Legacy.

Come alternativa grafica funziona XCA (X Certificate and Key Management): importare il file PFX tramite `Importieren > PKCS#12`, quindi esportare il certificato come PEM nella scheda `Zertifikate` ed esportare separatamente la chiave come PEM non crittografato nella scheda `Private Schlüssel`. Sono necessarie entrambe le esportazioni, poiché `certconfig` richiede certificato e chiave separatamente. XCA include la propria libreria crittografica e apre anche contenitori con algoritmi Legacy.

Un'ulteriore nota sulla fonte di download: il progetto OpenSSL non pubblica direttamente binari Windows, ma rimanda a build di terze parti come Win64 OpenSSL di Shining Light Productions. I portali di download con i propri installer non sono la fonte giusta per uno strumento crittografico.

## Importare prima la Root CA interna nel truststore dell'appliance

Le versioni AsyncOS attuali convalidano l'intera catena durante la creazione di un profilo di certificato. Se il certificato proviene da una CA interna la cui root non è nota all'appliance, l'importazione si interrompe con questo messaggio:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

L'appliance gestisce due elenchi di autorità di certificazione attendibili: l'elenco di sistema fornito e un elenco Custom per le proprie CA. La Root CA interna deve essere inserita nell'elenco Custom prima di installare il certificato server. È necessario solo il certificato CA pubblico come file PEM (`-----BEGIN CERTIFICATE-----` fino a `-----END CERTIFICATE-----`), non una chiave privata.

Ecco come trasferire la Root CA sull'appliance tramite l'interfaccia web:

1. Aprire `Network > Certificates`.
2. Nella sezione `Certificate Authorities`, fare clic su `Edit Settings`.
3. Per `Custom List`, selezionare l'opzione `Enable`.
4. Caricare il file PEM tramite `Choose File`.
5. Eseguire `Submit` e quindi `Commit Changes`.
6. In `Network > Certificates > Manage Trusted Root Certificates`, verificare che la CA compaia nell'elenco dei certificati personalizzati.

Se esiste già un elenco Custom, esportarlo prima e aggiungere la nuova CA al bundle PEM esistente: l'importazione sostituisce l'elenco, altrimenti le CA precedentemente archiviate scompaiono. Per una catena con un livello intermedio, importare prima la Root CA e poi la CA Intermediate. Durante l'importazione, AsyncOS controlla tra l'altro la data di scadenza, i duplicati e il flag `CA:TRUE` impostato, e rifiuta una Intermediate finché manca la Root corrispondente. La stessa importazione è possibile anche tramite CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, quindi `commit`.

Due precisazioni: per gli aggiornamenti tramite un proxy che ispeziona TLS, la SMA utilizza un truststore separato (`updateconfig > TRUSTED_CERTIFICATES > ADD`), al quale l'elenco Custom CA non si applica. Inoltre, la Root CA sulla SMA non elimina gli avvisi del browser: i client continuano a necessitare della Root tramite la propria distribuzione dei certificati, tipicamente mediante GPO, e l'appliance deve fornire il certificato server insieme all'intermediate.

## Installazione con certconfig

Accedere alla SMA tramite SSH e avviare `certconfig`. Nelle versioni AsyncOS attuali, la finestra di dialogo utilizza profili di certificato:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Dietro `CERTIFICATE` si trovano le operazioni `IMPORT` (file PKCS#12 precedentemente caricato sull'appliance), `PASTE` (incollare il certificato nella CLI), `NEW` (generare direttamente un certificato self-signed con CSR), `EDIT`, `EXPORT`, `DELETE` e `PRINT` (mostra l'assegnazione ai servizi). Il metodo usuale tramite SSH è `PASTE`: la finestra di dialogo richiede un nome per il profilo, quindi il certificato, la chiave privata e, facoltativamente, il certificato Intermediate della CA, ciascuno come blocco PEM, terminato da un singolo `.` su una riga separata. Alla domanda finale sulla verifica FQDN del Common Name si può rispondere con il valore predefinito. L'Intermediate deve essere incluso nel profilo, altrimenti ai client manca la catena e, a seconda del browser, l'avviso può persistere nonostante il certificato valido.

Le versioni AsyncOS meno recenti (quelle della Technote Cisco) mostrano invece una finestra di dialogo `SETUP`. Inizia con la domanda `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: un `y` assegna la stessa coppia a tutti e quattro i servizi, mentre un `n` passa attraverso la richiesta di certificato, chiave e Intermediate una volta per ogni servizio. Il principio di incollaggio è identico.

Due aspetti determinano il successo o il fallimento: non terminare la sessione con Ctrl+C, poiché ciò annulla immediatamente tutte le modifiche. Infine eseguire `commit`, solo allora il certificato diventa attivo. Con due appliance, la procedura va ripetuta su entrambe: la configurazione dei certificati non viene sincronizzata tra le SMA.

## Verifica

Il test più rapido viene eseguito dall'esterno contro la pagina di quarantena. Per impostazione predefinita, l'accesso degli utenti finali alla quarantena antispam utilizza la porta HTTPS 83, a meno che non sia stata configurata diversamente durante l'attivazione:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `s_client` | Client di test TLS: stabilisce la connessione e inoltra il certificato presentato |
| `-connect …:83` | Host e porta di destinazione, qui la porta HTTPS della quarantena antispam |
| `-servername …` | Imposta la Server Name Indication (SNI), affinché il server fornisca il certificato corretto |
| `x509` | Elabora il certificato inoltrato |
| `-noout` | Sopprime l'output del certificato codificato |
| `-subject` | Mostra il Subject del certificato |
| `-enddate` | Mostra la data di scadenza (notAfter) |

</details>

L'output deve mostrare il nuovo Subject e la nuova data di scadenza. Sull'appliance, `certconfig` con l'operazione `PRINT` elenca i certificati attivi; il controllo nel browser della GUI di amministrazione e della pagina di quarantena conferma che la catena è costruita correttamente.

## Fonti

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Technote Cisco con la procedura certconfig delle versioni AsyncOS meno recenti, il requisito PEM e i metodi per generare il certificato tramite ESA o OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): capitolo della guida di amministrazione sulla gestione degli elenchi di Certificate Authority (elenco System e Custom), incluse le verifiche durante l'importazione della CA.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): guida Cisco alla quarantena antispam, incluso l'accesso degli utenti finali tramite HTTPS sulla porta 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): riferimento per la generazione di chiave e CSR, incluso `-addext` per l'elenco SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): riferimento delle opzioni di conversione, tra cui `-noenc` (in precedenza `-nodes`) e `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): approfondimento sullo spostamento degli algoritmi meno recenti nel provider Legacy.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): strumento open source per l'importazione e l'esportazione di strutture PKCS#12 e PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): build Windows di Shining Light Productions a cui fa riferimento il progetto OpenSSL, incluso l'elenco di checksum pubblicato.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): descrizione della riscrittura automatica dei percorsi che in Git Bash modifica l'argomento `-subj`, incluso `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): riferimento del client di test TLS, tra cui `-connect` e `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): riferimento delle opzioni di visualizzazione, tra cui `-noout`, `-subject` e `-enddate`.
