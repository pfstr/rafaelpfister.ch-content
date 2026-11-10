---
title: "Proofpoint Protection Server 8.22.4: Patch 5807 lascia SMTP Client senza certificato"
navTitle: "PPS 8.22.4 certificato"
description: "Le patch 5806 e 5807 introducono in Proofpoint Protection Server 8.22.4, in System > Certificates > Services, un'assegnazione separata «SMTP Client». Se è impostata su «-none-», Sendmail effettua le consegne senza certificato client TLS. Le controparti che verificano il certificato lo rifiutano, come Exchange Online con 5.7.51 o uno Smarthost con «Relay access denied». Ecco come verificarlo e risolverlo."
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
translationSourceHash: fcb36b2ae8af7420156aca54a99d61f4702aeb47781e1081e27edf7419e77db8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-07T03:20:55.328Z
translationReview: automatic
aktualisiert: 2026-11-06
url: https://rafaelpfister.ch/it/blog/proofpoint-protection-server-8-22-4-la-patch-puo-reimpostare-la-configurazione-del-certificato
---

Oggi ho analizzato un'interruzione del flusso di posta in uscita di un Proofpoint Protection Server (PPS) 8.22.4. In breve: dopo l'installazione automatica di una patch, in **System > Certificates > Services** l'assegnazione **SMTP Client** era impostata su `-none-`. Sendmail ha quindi stabilito connessioni in uscita senza certificato client TLS. Tutte le controparti che riconoscono Proofpoint tramite questo certificato hanno rifiutato i messaggi. I normali destinatari con TLS opportunistico hanno continuato ad accettarli, perciò il problema è emerso solo quando alcuni partner non erano più raggiungibili.

Se utilizzate PPS con patch automatiche e avete partner che richiedono un certificato client, controllate questa impostazione dopo ogni patch.

## Sintomo

Sono interessate solo le connessioni in cui la controparte verifica il certificato client di Proofpoint. Due casi tipici:

- **Exchange Online con connettore partner**, limitato a un certificato tramite `RestrictDomainsToCertificate`. Il rifiuto è `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost che consente il relay solo con certificato client.** La risposta è `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Tutte le altre destinazioni continuano a ricevere i messaggi. La matrice di test mostra quindi un quadro non uniforme: i messaggi esterni passano, mentre quelli verso singoli partner o tramite uno specifico Smarthost falliscono.

Nel log di posta dell'agent compare uno dei due messaggi seguenti per ogni connessione in uscita. Se il file manca del tutto:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Se il file esiste ma è vuoto:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Subito dopo, Sendmail stabilisce comunque la connessione con TLS (`verify=OK`), ma senza certificato client. Per la controparte, ciò appare come un mittente sconosciuto.

## Causa

PPS assegna un certificato a ogni servizio per server. In **System > Certificates > Services**, per ogni server sono elencati tra gli altri `SMTP Server`, `SMTP Client`, `Administration Server` e `End User Web Application / Encryption`. Dall'assegnazione in `SMTP Client`, PPS genera il file `/etc/pki/tls/certs/sendmail-client.pem`, che Sendmail utilizza come certificato client. Se è impostato su `-none-`, PPS scrive il file vuoto oppure non lo crea affatto.

L'assegnazione `SMTP Client` esiste in 8.22.4 solo a partire dalle patch 5806 e 5807, pubblicate da Proofpoint il 29.10.2026. Entrambe hanno il titolo «TLS client and server certificate authentication improvement» e, secondo la Knowledge Base, risolvono il problema per cui PPS 8.22.4 non può assegnare certificati separati per SMTP server e SMTP client. Nella release 8.23.0 la funzionalità è inclusa regolarmente (Release Notes, sezione «SMTP Client Certificate»).

| Patch | Contenuto secondo la Knowledge Base |
|---|---|
| 5806 | Aggiorna tra l'altro `SysCertificates.pm`, `Certificate.pm`, `TLS.pm`, `pp-sendmail.pl` e il modello `sendmail.mc`, riavvia la GUI amministrativa (`admind`) e aggiorna l'RPM `pps-pfpt` |
| 5807 | Richiede 5806, non modifica file né riavvia processi; applica solo le modifiche di configurazione necessarie per certificati server e client separati |

Proofpoint ha quindi introdotto una nuova assegnazione e non ne ha ripristinata una esistente. La patch non rileva alcun certificato: dopo l'installazione, la nuova riga è impostata su `-none-`, e da quel momento Sendmail non utilizza più un certificato client. Le descrizioni delle patch non menzionano questo aspetto.

Nell'ambiente analizzato, l'assegnazione era poi impostata su `-none-` su tutti i server. In **System > Config History**, la patch 5807 era visibile come voce `configuration changes for patch-…` con creatore `system`, e i file in `/etc/pki/tls/certs/` riportavano lo stesso timestamp. In **Software Patch History**, invece, 5807 non compariva. Ciò è coerente con il fatto che 5807 non installa software ma modifica soltanto la configurazione. **Config History** mostra quindi il momento della modifica, mentre la modifica software vera e propria appartiene alla patch 5806.

Un rollback in **Config History** allo snapshot della patch non ha aiutato. PPS ha poi riscritto il file, ma con 0 byte.

## Verifica

Nella GUI amministrativa sul master, in **System > Certificates > Services**, selezionate uno alla volta, in alto in **Server**, ogni agent che effettua consegne verso l'esterno e controllate la riga `SMTP Client`.

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
| `grep -E` | Espressioni regolari estese, per combinare entrambi i termini di ricerca con `\|` |
| `tail -5` | Visualizza solo gli ultimi cinque risultati |

</details>

Se `sendmail-client.pem` manca o il file ha 0 byte, mentre `sendmail.pem` è grande alcune migliaia di byte, al servizio `SMTP Client` non è assegnato alcun certificato.

## Risoluzione

In **System > Certificates > Services**, per ogni agent interessato selezionate un certificato valido per `SMTP Client` e salvate con **Save Changes**. Di norma è lo stesso certificato pubblico assegnato a `SMTP Server`.

L'elenco di selezione può contenere più certificati con lo stesso nome, inclusi certificati scaduti. È possibile distinguerli solo tramite il numero di serie tra parentesi. Prima di salvare, confrontate il numero di serie con l'elenco in **System > Certificates > Certificates**, dove sono indicati data di emissione e data di scadenza.

L'elenco per `SMTP Client` non mostra necessariamente tutti i certificati importati. Nell'ambiente analizzato mancavano alcuni certificati, sebbene fossero validi e gli esemplari scaduti fossero già stati eliminati. Né gli articoli sulle patch né le Release Notes 8.23 descrivono i criteri con cui PPS filtra l'elenco. Se manca un certificato, verificate due aspetti:

- **Chiave privata:** Se il certificato è stato importato come PEM senza chiave, PPS non può assegnarlo a un servizio. I file PKCS12 contengono sempre la chiave.
- **Scopo d'uso:** Il certificato contiene in `Extended Key Usage` lo scopo `TLS Web Client Authentication`? Molti certificati pubblici emessi dall'autunno 2025 non lo contengono più (vedi sotto). Per il servizio `SMTP Client`, un certificato senza questo scopo è comunque inadeguato.

Quando Proofpoint confermerà i criteri, li aggiungerò qui.

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
| `grep -E 'STARTTLS=client\|stat='` | Filtra i messaggi TLS delle connessioni in uscita e l'esito della consegna |
| `tail -20` | Visualizza solo le ultime venti righe |

</details>

Dopo il salvataggio non devono più comparire i messaggi `SSL_CTX_… failed` e `unsafe`, mentre le consegne ai partner devono terminare con `stat=Sent`. I messaggi precedentemente rinviati con un codice 4xx vengono consegnati da Sendmail dalla coda. I messaggi rifiutati con un codice 5xx devono essere inviati di nuovo dai mittenti.

Anche una copia manuale di `sendmail.pem` in `sendmail-client.pem` aggira l'interruzione. Tuttavia, PPS sovrascrive nuovamente il file al successivo push di configurazione finché manca l'assegnazione nella GUI.

## Controllare dopo ogni patch

La patch 5807 ha modificato la configurazione senza che la descrizione della patch indicasse le conseguenze. Consiglio quindi di controllare dopo ogni patch la pagina **Services** e **Config History**, per verificare se la patch ha introdotto modifiche di configurazione. Se la patch 5807 non è ancora installata su un sistema 8.22.4, assegnate un certificato a `SMTP Client` subito dopo l'installazione. Ancora più semplice è un monitoraggio che generi un allarme per il messaggio `sendmail-client.pem` nel log di posta degli agent. In **Licenses and Updates > Settings** è inoltre possibile definire quando installare le patch. Una finestra temporale in cui qualcuno può verificare il flusso di posta riduce sensibilmente la durata di un'interruzione simile.

## Client Authentication scompare dai certificati pubblici

Indipendentemente dalla patch, una seconda modifica interessa proprio queste connessioni. Finora, i certificati TLS pubblici contenevano generalmente entrambi gli scopi d'uso nell'estensione `Extended Key Usage`: `TLS Web Server Authentication` e `TLS Web Client Authentication`. I gateway di posta utilizzano quindi il proprio certificato server anche come certificato client.

Il Chrome Root Program richiede che i certificati pubblici emessi a partire dal 15.03.2027 contengano solo `serverAuth`. DigiCert emette certificati TLS pubblici senza Client Authentication come impostazione predefinita dal 01.10.2025; su richiesta, lo scopo è ancora possibile fino al 01.03.2027. Altre CA hanno calendari analoghi. OpenSSL mostra se un certificato contiene lo scopo client:

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
| `-subject` | Visualizza il soggetto (CN e altri campi) |
| `-enddate` | Visualizza la data di scadenza |
| `-ext extendedKeyUsage` | Visualizza solo l'estensione con gli scopi d'uso |

</details>

In Windows, un doppio clic sul file mostra la stessa informazione: scheda **Details**, campo **Enhanced Key Usage**.

Le conseguenze per le connessioni in uscita dipendono dalla controparte:

| La controparte verifica | Conseguenza con un certificato senza Client Authentication |
|---|---|
| Nome nel certificato (ad es. Exchange Online tramite `TlsSenderCertificateName`) | Non è documentato se Exchange Online verifica lo scopo d'uso; eseguire un test prima della sostituzione del certificato |
| Scopo d'uso del certificato client | Rifiuto, anche se nome e CA corrispondono |
| Impronta digitale del certificato client | Rifiuto a ogni sostituzione del certificato, finché non viene memorizzata la nuova impronta digitale |

A ciò si aggiunge la durata più breve dei certificati pubblici. Dopo la decisione SC-081v3 del CA/Browser Forum, essa diminuisce gradualmente da 398 giorni a 47 giorni entro marzo 2029; il primo livello di 200 giorni è in vigore da marzo 2026. Ogni sostituzione del certificato è un momento in cui una controparte potrebbe non conoscere il nuovo certificato.

Per le connessioni in cui i partner riconoscono Proofpoint tramite un certificato client, consiglio quindi:

1.  **Verificare i certificati esistenti:** I certificati gateway attuali contengono ancora lo scopo client e quando scadono?

2.  **Pianificare un certificato client dedicato**, che contenga permanentemente lo scopo client, ad esempio da una PKI privata o da un'offerta della CA per connessioni server-to-server. I partner devono conoscerlo.

3.  **Chiedere ai partner cosa verificano:** nome, scopo d'uso, emittente o impronta digitale. Da questo dipende se una sostituzione del certificato costituisce una modifica per loro.

4.  **Gestire la sostituzione dei certificati come change**, con un elenco di tutte le controparti che devono conoscere il certificato e un test per ciascuna controparte.

## Fonti

1.  [Patch 5806 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, accesso richiesto)](https://proofpoint.my.site.com/community/s/article/Patch-5806---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): articolo 000027579 del 29.10.2026, file modificati e riavvio della GUI amministrativa.

2.  [Patch 5807 - TLS client and server certificate authentication improvement - Fix for 8.22.4 (Proofpoint Knowledge Base, accesso richiesto)](https://proofpoint.my.site.com/community/s/article/Patch-5807---TLS-client-and-server-certificate-authentication-improvement---Fix-for-8-22-4): articolo 000027580 del 29.10.2026, sola modifica di configurazione con dipendenza da 5806.

3.  [Proofpoint Enterprise Protection 8.23.X Release Notes (Proofpoint Knowledge Base, accesso richiesto)](https://proofpoint.my.site.com/community/s/article/Email-Protection-PPS-PoD-Proofpoint-Enterprise-Protection-8-23-X-Release-Notes): articolo 000027117, sezione «SMTP Client Certificate» a pagina 5 della revisione A.

4.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): parametri `RestrictDomainsToCertificate` e `TlsSenderCertificateName` per connettori partner in Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): sezione 1.3.2 sulle gerarchie TLS server dedicate; certificati dal 15.03.2027 solo con `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): impostazione predefinita senza Client Authentication dal 01.10.2025, su richiesta fino al 01.03.2027, alternative per connessioni server-to-server.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): panoramica delle date rinviate per Client Authentication e dei livelli di durata presso DigiCert e Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): decisione sulla riduzione graduale della durata dei certificati TLS pubblici fino a marzo 2029.
