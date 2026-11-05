---
title: "Proofpoint Protection Server 8.22.4: la patch può reimpostare la configurazione del certificato per il client SMTP"
navTitle: "Certificato PPS 8.22.4"
description: "Se in Proofpoint Protection Server, in System > Certificates > Services, l'associazione «SMTP Client» è impostata su «-none-», Sendmail effettua le consegne senza certificato client TLS. Le controparti con verifica del certificato rifiutano quindi i messaggi, ad esempio Exchange Online con 5.7.51 o uno Smarthost con «Relay access denied». Ecco come verificare e risolvere il problema."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min di lettura"
themen:
  - proofpoint
  - smtp-mailflow
produkte:
  - "proofpoint-pps"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-protection-server-8-22-4-la-patch-puo-reimpostare-la-configurazione-del-certificato"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
url: https://rafaelpfister.ch/it/blog/proofpoint-protection-server-8-22-4-la-patch-puo-reimpostare-la-configurazione-del-certificato
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:26:57.943Z
translationReview: automatic
---

Oggi ho analizzato un'interruzione del flusso di posta in uscita su un Proofpoint Protection Server (PPS) 8.22.4. In breve: dopo l'installazione automatica di una patch, in **System > Certificates > Services** l'associazione **SMTP Client** era impostata su `-none-`. Sendmail ha quindi stabilito connessioni in uscita senza certificato client TLS. Tutte le controparti che riconoscono Proofpoint tramite questo certificato hanno rifiutato le e-mail. I destinatari normali con TLS opportunistico hanno continuato ad accettarle, perciò l'errore è emerso solo quando singoli partner non erano più raggiungibili.

Se utilizzate PPS con patch automatiche e avete partner che richiedono un certificato client, verificate questa impostazione dopo ogni patch.

## Sintomo

Sono interessate solo le connessioni per le quali la controparte verifica il certificato client di Proofpoint. Due casi tipici:

- **Exchange Online con connettore partner**, limitato a un certificato tramite `RestrictDomainsToCertificate`. Il rifiuto è `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost che consente il relay solo con certificato client.** La risposta è `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Tutte le altre destinazioni continuano a ricevere le loro e-mail. La matrice di test mostra quindi un quadro incoerente: le e-mail esterne vengono recapitate, mentre quelle verso singoli partner o tramite uno Smarthost specifico falliscono.

Nel log di posta dell'agent compare uno dei due messaggi seguenti per ogni connessione in uscita. Se il file manca completamente:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Se il file è presente ma vuoto:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Subito dopo, Sendmail stabilisce comunque la connessione con TLS (`verify=OK`), ma senza certificato client. Per la controparte, appare come un mittente sconosciuto.

## Causa

PPS assegna un certificato a ciascun servizio per server. In **System > Certificates > Services**, per ogni server sono presenti tra gli altri `SMTP Server`, `SMTP Client`, `Administration Server` e `End User Web Application / Encryption`. Dall'associazione in `SMTP Client`, PPS genera il file `/etc/pki/tls/certs/sendmail-client.pem`, che Sendmail usa come certificato client. Se è impostato su `-none-`, PPS scrive il file vuoto oppure non lo crea affatto.

Nell'ambiente analizzato, dopo una patch automatica l'associazione era impostata su `-none-` su tutti i server. In **System > Config History**, la patch era visibile come voce `configuration changes for patch-…` con autore `system`; i file in `/etc/pki/tls/certs/` riportavano lo stesso timestamp. Le note di rilascio pubblicamente disponibili non descrivono se Proofpoint abbia introdotto l'associazione con questa patch oppure se la patch abbia reimpostato un'associazione esistente. Aggiornerò l'articolo non appena Proofpoint rilascerà una dichiarazione in merito.

Un rollback in **Config History** allo snapshot della patch non è stato d'aiuto. PPS ha poi riscritto il file, ma con 0 byte.

## Verifica

Nella GUI di amministrazione, sul master, andate a **System > Certificates > Services**, selezionate uno dopo l'altro in alto sotto **Server** ogni agent che effettua consegne verso l'esterno e controllate la riga `SMTP Client`.

Sull'agent stesso, due comandi mostrano lo stato:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-l` | Formato esteso con permessi, proprietario, dimensione e data |
| `--time-style=full-iso` | Timestamp con data, ora e fuso orario, per confrontarlo con il momento della patch |
| `grep -E` | Espressioni regolari estese, per poter combinare entrambi i termini di ricerca con `\|` |
| `tail -5` | Mostra solo gli ultimi cinque risultati |

</details>

Se `sendmail-client.pem` manca oppure il file è di 0 byte, mentre `sendmail.pem` è grande alcune migliaia di byte, al servizio `SMTP Client` non è assegnato alcun certificato.

## Correzione

In **System > Certificates > Services**, per ogni agent interessato selezionate un certificato valido per `SMTP Client` e salvate con **Save Changes**. Di norma è lo stesso certificato pubblico indicato per `SMTP Server`.

L'elenco di selezione mostra tutti i certificati importati, inclusi quelli scaduti con lo stesso nome. Possono essere distinti solo tramite il numero di serie tra parentesi. Prima di salvare, confrontate il numero di serie con l'elenco in **System > Certificates > Certificates**, dove sono indicati la data di emissione e di scadenza.

Dopo il salvataggio, PPS distribuisce la configurazione agli agent. Verifica sull'agent:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `ls -l` | Mostra dimensione e permessi del file; dopo la correzione non è più vuoto |
| `grep -E 'STARTTLS=client\|stat='` | Filtra i messaggi TLS delle connessioni in uscita e il risultato della consegna |
| `tail -20` | Mostra solo le ultime venti righe |

</details>

Dopo il salvataggio, i messaggi `SSL_CTX_… failed` e `unsafe` non devono più comparire e le consegne ai partner devono terminare con `stat=Sent`. Sendmail recapita autonomamente dalla coda le e-mail precedentemente rinviate con un codice 4xx. Le e-mail rifiutate con un codice 5xx devono essere inviate nuovamente dai mittenti.

Anche copiare manualmente `sendmail.pem` in `sendmail-client.pem` aggira temporaneamente l'interruzione. Tuttavia, PPS sovrascrive nuovamente il file al successivo push della configurazione, finché manca l'associazione nella GUI.

## Controllare dopo ogni patch

Finché non sarà chiarito perché la patch ha modificato l'associazione, consiglio di controllare la pagina **Services** dopo ogni patch. Ancora più semplice è un monitoraggio che generi un allarme alla comparsa del messaggio `sendmail-client.pem` nel log di posta degli agent. In **Licenses and Updates > Settings** potete inoltre definire quando installare le patch. Una finestra temporale in cui qualcuno possa verificare il flusso di posta riduce significativamente la durata di un'interruzione di questo tipo.

## Client Authentication scompare dai certificati pubblici

Indipendentemente dalla patch, una seconda modifica interessa proprio queste connessioni. Finora, i certificati TLS pubblici contenevano generalmente entrambi gli scopi d'uso nell'estensione `Extended Key Usage`: `TLS Web Server Authentication` e `TLS Web Client Authentication`. I gateway di posta usano quindi il proprio certificato server anche come certificato client.

Il Chrome Root Program richiede che i certificati pubblici emessi dal 15.03.2027 contengano solo `serverAuth`. DigiCert emette certificati TLS pubblici senza Client Authentication come impostazione predefinita dal 01.10.2025; su richiesta, lo scopo sarà ancora disponibile fino al 01.03.2027. Altre CA hanno calendari simili. OpenSSL mostra se un certificato include lo scopo client:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `x509` | Sottocomando per visualizzare e modificare certificati X.509 |
| `-in zertifikat.pem` | File del certificato in formato PEM |
| `-noout` | Non visualizza nuovamente il certificato stesso |
| `-subject` | Mostra il soggetto (CN e altri campi) |
| `-enddate` | Mostra la data di scadenza |
| `-ext extendedKeyUsage` | Mostra solo l'estensione con gli scopi d'uso |

</details>

In Windows, un doppio clic sul file mostra la stessa informazione: scheda **Dettagli**, campo **Utilizzo avanzato chiave**.

Le conseguenze per le connessioni in uscita dipendono dalla controparte:

| La controparte verifica | Conseguenza con un certificato senza Client Authentication |
|---|---|
| Nome nel certificato (ad es. Exchange Online tramite `TlsSenderCertificateName`) | Non è documentato se Exchange Online verifichi lo scopo d'uso; testate prima di sostituire il certificato |
| Scopo d'uso del certificato client | Rifiuto, anche se nome e CA sono corretti |
| Impronta digitale del certificato client | Rifiuto a ogni sostituzione del certificato, finché non viene registrata la nuova impronta digitale |

A questo si aggiunge la durata più breve dei certificati pubblici. In base alla decisione SC-081v3 del CA/Browser Forum, scende gradualmente da 398 giorni a 47 giorni entro marzo 2029; il primo livello, pari a 200 giorni, è in vigore da marzo 2026. Ogni sostituzione del certificato è un momento in cui una controparte potrebbe non conoscere il nuovo certificato.

Per le connessioni in cui i partner riconoscono Proofpoint tramite un certificato client, consiglio pertanto:

1.  **Verificare i certificati esistenti:** gli attuali certificati del gateway contengono ancora lo scopo client e quando scadono?

2.  **Pianificare un certificato client dedicato**, che includa stabilmente lo scopo client, ad esempio da una PKI privata o da un'offerta della CA per connessioni server-to-server. I partner devono conoscerlo.

3.  **Chiedere ai partner cosa verificano:** nome, scopo d'uso, emittente o impronta digitale. Da questo dipende se per loro una sostituzione del certificato costituisce una modifica.

4.  **Gestire la sostituzione del certificato come change**, con un elenco di tutte le controparti che devono conoscere il certificato e un test per ciascuna controparte.

## Fonti

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): parametri `RestrictDomainsToCertificate` e `TlsSenderCertificateName` per connettori partner in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): sezione 1.3.2 sulle gerarchie TLS server dedicate; certificati dal 15.03.2027 solo con `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): impostazione predefinita senza Client Authentication dal 01.10.2025, su richiesta fino al 01.03.2027, alternative per connessioni server-to-server.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): panoramica delle date posticipate per Client Authentication e dei livelli di durata presso DigiCert e Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): decisione sulla graduale riduzione della durata dei certificati TLS pubblici entro marzo 2029.
