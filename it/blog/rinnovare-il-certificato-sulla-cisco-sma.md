---
title: "Rinnovare il certificato sulla Cisco SMA"
navTitle: "Certificato SMA"
description: "Sulla Cisco SMA i certificati possono essere installati solo tramite CLI e le versioni AsyncOS attuali convalidano l’intera catena durante l’importazione: senza una Root CA memorizzata, l’operazione non riesce. L’articolo illustra le modalità per ottenere una nuova coppia di chiavi, il metodo OpenSSL in dettaglio, la gestione dell’errore RC2-40-CBC di OpenSSL 3 e l’importazione della Root CA interna nel truststore dell’appliance."
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
url: https://rafaelpfister.ch/it/blog/rinnovare-il-certificato-sulla-cisco-sma
translationSourceHash: 0c12510db6a327680d08d3f4eb6924738cef4987860e42c41043ce66467d4249
translationModel: gpt-5.6-terra
translatedAt: 2026-08-10T04:59:46.912Z
translationReview: automatic
---

# Rinnovare il certificato sulla Cisco SMA

La Cisco SMA (Security Management Appliance, ora commercializzata con il nome Cisco Secure Email and Web Manager) gestisce in molti ambienti di posta elettronica la quarantena spam centralizzata e il reporting per i Secure Email Gateway. Il suo certificato HTTPS copre la GUI di amministrazione e la pagina di quarantena, nella quale gli utenti finali visualizzano e rilasciano le email trattenute. Alla sua scadenza il flusso di posta non si interrompe. Tuttavia, la scadenza diventa subito visibile: ogni accesso alla pagina di quarantena termina con un avviso relativo al certificato nel browser e proprio gli utenti ai quali i corsi di awareness insegnano a non proseguire in presenza di tali avvisi dovrebbero ignorarli.

Durante un rinnovo in un progetto cliente sono emersi subito due ostacoli: dapprima OpenSSL 3 ha risposto al file PFX della CA interna con un errore criptico relativo a `RC2-40-CBC`, poi l’appliance ha rifiutato l’importazione del certificato finale perché non conosceva la Root CA emittente. Entrambi gli ostacoli e le relative soluzioni sono illustrati più avanti.

## Cosa rende la SMA diversa dalla ESA

Sulla ESA l’intero ciclo di vita del certificato può essere gestito tramite GUI (`Network > Certificates`). La SMA non può farlo: il certificato server viene installato esclusivamente tramite CLI, con il comando `certconfig` in una sessione SSH. La GUI della SMA mostra soltanto i certificati; è possibile gestire unicamente gli elenchi delle autorità di certificazione attendibili, come vedremo più avanti.

Vi sono inoltre altre due peculiarità:

- La finestra di inserimento accetta solo il formato PEM. Un file PFX (PKCS#12) deve essere convertito prima dell’installazione; le versioni AsyncOS attuali offrono anche un’importazione diretta PKCS#12, ma il file deve prima essere trasferito sull’appliance.
- Le versioni AsyncOS meno recenti (quelle documentate nella technote Cisco) non generano autonomamente né chiavi né CSR: la coppia di chiavi deve essere creata esternamente; i tre metodi possibili sono descritti più avanti. Le versioni attuali possono generare direttamente sull’appliance un certificato self-signed con CSR tramite `certconfig > CERTIFICATE > NEW`. Ciò non è però utile per un certificato condiviso tra più appliance, perché in questo caso la chiave privata non lascia mai l’appliance.

Un singolo certificato può servire tutti i servizi (TLS in entrata e in uscita, accesso HTTPS di amministrazione, LDAPS) oppure essere assegnato separatamente a ciascun servizio. Questo viene gestito nella finestra di dialogo `certconfig`; l’intestazione del comando mostra in ogni momento l’assegnazione attiva (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). Non esiste una schermata di assegnazione separata come sulla ESA e non è possibile modificare questa configurazione tramite GUI. Nella maggior parte degli ambienti, un certificato per tutto è la scelta pragmatica: l’elenco dei nomi copre comunque gli FQDN delle appliance e coppie di chiavi separate moltiplicano il lavoro a ogni rinnovo.

Il fatto che la finestra di dialogo di un’appliance di quarantena chieda informazioni sul TLS inbound e outbound può inizialmente sorprendere, perché la SMA non è collocata in un percorso MX. Tuttavia, comunica via SMTP in entrambe le direzioni. Inbound (Receiving) è il lato di ricezione: le ESA consegnano alla SMA i messaggi messi in quarantena tramite SMTP, nella quarantena spam centrale sulla porta 6025 e nelle quarantene centrali per policy, virus e outbreak sulla porta 7025; queste ultime connessioni sono cifrate con TLS per impostazione predefinita e la SMA presenta proprio questo certificato. Outbound (Delivery) è il lato di invio: quando un utente rilascia un messaggio dalla quarantena, la SMA lo riconsegna autonomamente al flusso di posta tramite le proprie route SMTP; inoltre, l’appliance invia come messaggi propri anche le notifiche di quarantena, i report pianificati e gli avvisi. Per il rinnovo questo significa che in pratica HTTPS è l’aspetto critico; i due servizi SMTP vengono coperti automaticamente dal certificato per tutti i servizi.

## Definire i nomi: CN e SAN

Indipendentemente dal metodo scelto per la coppia di chiavi, occorre prima definire l’elenco dei nomi. Il Common Name deve corrispondere al nome host con cui gli utenti accedono alla pagina di quarantena. Nell’elenco SAN vanno aggiunti gli FQDN delle appliance, affinché anche l’accesso diretto alla GUI di amministrazione funzioni senza avvisi. Per un ambiente con due appliance, l’elenco dei nomi è il seguente:

| Campo | Valore |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Due osservazioni: da tempo i browser valutano soltanto le voci SAN; il solo CN non è sufficiente. Il nome host della quarantena deve quindi comparire anche come SAN. Inoltre, nomi host brevi senza componente di dominio (ad esempio `SMA01`) vengono emessi solo da una CA interna; le CA pubbliche non firmano nomi interni.

## Tre metodi per la nuova coppia di chiavi

Per un certificato che copra più appliance e il nome host della quarantena, la coppia di chiavi deve essere generata al di fuori dell’appliance. Si sono affermati tre metodi:

1. Generare chiave e CSR con OpenSSL all’interno del proprio ambiente. La chiave privata viene creata dove serve e non lascia mai l’ambiente. È il metodo consigliato; i dettagli sono nella sezione successiva.
2. La CA genera la coppia di chiavi e fornisce un file PFX. Funziona, ma presenta due svantaggi: la chiave passa attraverso mani esterne (per questo la password va trasmessa tramite un canale separato e non nella stessa email del file) e, a seconda dello strumento della CA, può essere restituito un PFX cifrato con RC2, che OpenSSL 3 apre solo con un intervento aggiuntivo; maggiori dettagli più avanti.
3. Il passaggio tramite una ESA, documentato nella technote Cisco: creare un certificato con il CN della SMA in `Network > Certificates`, scaricare la CSR e farla firmare dalla CA, caricare nuovamente il certificato firmato sulla ESA ed esportare il tutto come PFX. Anche in questo caso, alla fine è necessaria la conversione in PEM.

## Avviare OpenSSL in Windows

Tutti i passaggi seguenti utilizzano OpenSSL, su un sistema interno all’ambiente, ad esempio un server di amministrazione. È sufficiente l’edizione Light delle build Windows di Shining Light Productions; l’installer ha una dimensione di circa 6 MB e può essere verificato rispetto all’elenco di checksum pubblicato da slproweb.

L’installer colloca tutto in `C:\Program Files\OpenSSL-Win64`, mentre l’eseguibile si trova in `bin\openssl.exe`. Non viene aggiunto al percorso di ricerca: chi digita `openssl` in un prompt dei comandi appena aperto riceve un messaggio di errore. Ci sono tre modi per risolvere:

- Dal menu Start, aprire la voce `Win64 OpenSSL Command Prompt`. Questa avvia `start.bat` dalla directory di installazione, imposta l’ambiente e visualizza l’output di `openssl version -a`. In questa finestra `openssl` funziona direttamente.
- Specificare il percorso completo: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Aggiungere in modo permanente `C:\Program Files\OpenSSL-Win64\bin` alla variabile d’ambiente `Path`; successivamente `openssl` sarà disponibile in ogni shell.

Chi utilizza già Git per Windows non necessita di installazioni aggiuntive: include la propria versione di OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`) e nella Git Bash è subito presente nel percorso di ricerca. Le versioni attuali di Git includono OpenSSL 3.5 con Legacy Provider attivo, quindi anche `-legacy` della sezione sulla conversione PFX funziona. È possibile verificarlo così:

```bash
openssl list -providers -provider legacy
```

La Git Bash presenta tuttavia un’insidia: considera percorsi gli argomenti che iniziano con `/` e li riscrive. Da `-subj "/C=CH/O=Example AG/CN=..."` diventa `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, e OpenSSL si interrompe:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Un `MSYS_NO_PATHCONV=1` anteposto disattiva la riscrittura per quella singola chiamata. Il problema non si presenta nel prompt dei comandi, in PowerShell e nell’OpenSSL Command Prompt.

## Generare chiave e CSR con OpenSSL

Un unico comando genera chiave e CSR con l’intero elenco SAN:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

Il file CSR viene inviato alla CA, mentre la chiave rimane sul server. Si riceve indietro il certificato firmato insieme all’intermediate, in genere direttamente in formato PEM. In questo modo è già disponibile tutto il necessario per l’installazione e la conversione PFX non serve affatto.

Il file della chiave non è cifrato (`-noenc`), perché `certconfig` lo richiede esattamente in questo formato. Fino all’installazione deve restare sul server con autorizzazioni restrittive; successivamente va eliminato o spostato nel gestore di password.

## Convertire PFX in PEM

Questa e la sezione successiva riguardano i metodi 2 e 3, che terminano con un file PFX. `certconfig` richiede certificato e chiave privata in formato PEM, con la chiave non cifrata. Un singolo comando OpenSSL esegue entrambe le operazioni:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (fino a OpenSSL 3.0 l’opzione era denominata `-nodes`) scrive la chiave privata senza passphrase nel file di output. La password di importazione viene richiesta senza eco e non vengono visualizzati asterischi. Il file PEM risultante contiene in un unico file certificato, chiave e certificati di catena forniti, quindi va protetto adeguatamente: dopo l’installazione, eliminarlo o spostarlo nel gestore di password.

## Quando OpenSSL 3 rifiuta il file PFX

Con i file PFX meno recenti, la conversione in OpenSSL 3.x si interrompe con il seguente messaggio:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La causa non è un file danneggiato, bensì una scelta progettuale: OpenSSL 3 ha spostato gli algoritmi legacy come RC2, RC4 e DES in un Legacy Provider separato, che per impostazione predefinita non viene caricato. Tuttavia, molte esportazioni PFX di sistemi Windows e strumenti CA meno recenti cifrano proprio la parte del certificato del contenitore con RC2-40-CBC. OpenSSL 1.1 apriva questi file senza problemi, mentre OpenSSL 3 li rifiuta.

La soluzione consiste in un’unica opzione aggiuntiva:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` carica il Legacy Provider per questa chiamata e la conversione viene completata. È necessario disporre di un’installazione OpenSSL che includa il Legacy Provider; le comuni build Windows lo includono.

Per eliminare definitivamente l’errore, occorre intervenire alla fonte ed esportare il file PFX con una cifratura moderna: le finestre di esportazione e gli strumenti CA attuali offrono AES-256, eliminando completamente il ricorso al Legacy Provider.

Come alternativa grafica, funziona XCA (X Certificate and Key Management): importare il file PFX tramite `Importieren > PKCS#12`, quindi esportare il certificato in PEM dalla scheda `Zertifikate` e la chiave separatamente come PEM non cifrato dalla scheda `Private Schlüssel`. Sono necessarie entrambe le esportazioni; `certconfig` richiede certificato e chiave separatamente. XCA include una propria libreria crittografica e apre anche contenitori con algoritmi legacy.

Un’ultima nota sulla fonte di download: il progetto OpenSSL non pubblica direttamente binari per Windows, ma rimanda a build di terzi come Win64 OpenSSL di Shining Light Productions. I portali di download con installer propri non sono la fonte adatta per uno strumento crittografico.

## Importare prima la Root CA interna nel truststore dell’appliance

Le versioni AsyncOS attuali convalidano l’intera catena durante la creazione di un profilo certificato. Se il certificato proviene da una CA interna la cui root non è nota all’appliance, l’importazione si interrompe con il seguente messaggio:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

L’appliance mantiene due elenchi di autorità di certificazione attendibili: l’elenco di sistema incluso e un elenco Custom per CA proprie. La Root CA interna deve essere aggiunta all’elenco Custom prima di installare il certificato server. È necessario solo il certificato CA pubblico in formato PEM (`-----BEGIN CERTIFICATE-----` fino a `-----END CERTIFICATE-----`), non una chiave privata.

Ecco come caricare la Root CA sull’appliance tramite l’interfaccia Web:

1. Aprire `Network > Certificates`.
2. Nella sezione `Certificate Authorities`, fare clic su `Edit Settings`.
3. In `Custom List`, selezionare l’opzione `Enable`.
4. Caricare il file PEM tramite `Choose File`.
5. Eseguire `Submit` e quindi `Commit Changes`.
6. In `Network > Certificates > Manage Trusted Root Certificates`, verificare che la CA compaia nell’elenco dei certificati personalizzati.

Se esiste già un elenco Custom, esportarlo prima e aggiungere la nuova CA al bundle PEM esistente: l’importazione sostituisce l’elenco, altrimenti le CA memorizzate in precedenza scompariranno. In caso di catena con livello intermedio, importare prima la Root CA e poi la CA Intermediate. Durante l’importazione, AsyncOS verifica tra l’altro data di scadenza, duplicati e il flag `CA:TRUE` impostato, e rifiuta una Intermediate finché manca la Root corrispondente. La stessa importazione è possibile anche tramite CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, quindi `commit`.

Due distinzioni importanti: per gli aggiornamenti tramite proxy con ispezione TLS, la SMA utilizza un truststore separato (`updateconfig > TRUSTED_CERTIFICATES > ADD`), al quale l’elenco Custom CA non si applica. Inoltre, la Root CA sulla SMA non elimina gli avvisi del browser: i client devono continuare a ricevere la root tramite la propria distribuzione di certificati, in genere tramite GPO, e l’appliance deve fornire il certificato server insieme all’intermediate.

## Installazione con certconfig

Accedere alla SMA tramite SSH e avviare `certconfig`. Nelle versioni AsyncOS attuali, la finestra di dialogo utilizza profili certificato:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Dietro `CERTIFICATE` si trovano le operazioni `IMPORT` (file PKCS#12 precedentemente caricato sull’appliance), `PASTE` (incollare il certificato nella CLI), `NEW` (generare un certificato self-signed con CSR), `EDIT`, `EXPORT`, `DELETE` e `PRINT` (mostra l’assegnazione ai servizi). Il metodo consueto tramite SSH è `PASTE`: la finestra richiede un nome per il profilo, quindi il certificato, la chiave privata e, facoltativamente, il certificato Intermediate della CA, ciascuno come blocco PEM e terminato da un singolo `.` su una riga separata. Alla domanda conclusiva relativa alla verifica FQDN del Common Name si può rispondere con il valore predefinito. L’Intermediate deve essere incluso nel profilo, altrimenti ai client manca la catena e, a seconda del browser, l’avviso può rimanere nonostante il certificato valido.

Le versioni AsyncOS meno recenti (quelle documentate nella technote Cisco) mostrano invece una finestra `SETUP`. Inizia con la domanda `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: un `y` assegna la stessa coppia a tutti e quattro i servizi, mentre un `n` ripete la richiesta di certificato, chiave e Intermediate una volta per ciascun servizio. Il principio di inserimento è identico.

Due aspetti determinano il successo o il fallimento: non terminare la sessione con Ctrl+C, poiché questo annulla immediatamente tutte le modifiche. Infine, eseguire `commit`; solo allora il certificato è attivo. Con due appliance, la procedura va ripetuta su entrambe: la configurazione dei certificati non viene sincronizzata tra le SMA.

## Verifica

Il test più rapido viene eseguito dall’esterno contro la pagina di quarantena. L’accesso degli utenti finali alla quarantena spam utilizza per impostazione predefinita la porta HTTPS 83, se non è stata configurata diversamente durante l’attivazione:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

L’output deve mostrare il nuovo Subject e la nuova data di scadenza. Sull’appliance, `certconfig` con l’operazione `PRINT` elenca i certificati attivi; il controllo nel browser della GUI di amministrazione e della pagina di quarantena conferma invece che la catena è configurata correttamente.

## Fonti

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): technote Cisco con la procedura certconfig delle versioni AsyncOS meno recenti, il requisito PEM e i metodi per la generazione del certificato tramite ESA o OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 per Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): capitolo dell’Admin Guide sulla gestione degli elenchi Certificate Authority (elenco System e Custom), incluse le verifiche durante l’importazione della CA.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): guida Cisco alla quarantena spam, incluso l’accesso degli utenti finali tramite HTTPS sulla porta 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): riferimento per la generazione di chiave e CSR, incluso `-addext` per l’elenco SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): riferimento sulle opzioni di conversione, tra cui `-noenc` (in precedenza `-nodes`) e `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): approfondimento sullo spostamento degli algoritmi legacy nel Legacy Provider.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): strumento open source per l’importazione e l’esportazione di strutture PKCS#12 e PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): build Windows di Shining Light Productions alle quali rimanda il progetto OpenSSL, incluso l’elenco di checksum pubblicato.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): descrizione della riscrittura automatica dei percorsi che nella Git Bash modifica l’argomento `-subj`, incluso `MSYS_NO_PATHCONV`.
