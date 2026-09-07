---
title: "Limite di licenze Totemomail raggiunto: ripulire gli utenti orfani tramite LDAP"
navTitle: "Limite di licenze raggiunto"
description: "Gli account AD disattivati rimangono in totemomail e continuano a occupare licenze. Con un accesso LDAPS verificato e il Cleanup-Agent, Active Directory diventa la fonte autorevole."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min di lettura"
themen:
  - totemomail
slug: "limite-di-licenze-totemomail-raggiunto-ripulire-gli-utenti-orfani-tramite-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
translationId: article-cdc60310665049b8
translationReview: automatic
translationSourceHash: e3ae37a51d159128640441aba3bc6993b47b8c5d10968228c05281452940756d
translatedAt: 2026-09-05T07:59:55.619Z
url: https://rafaelpfister.ch/it/blog/limite-di-licenze-totemomail-raggiunto-ripulire-gli-utenti-orfani-tramite-ldap
translationModel: gpt-5.6-terra
---

# Limite di licenze Totemomail raggiunto: ripulire gli utenti orfani tramite LDAP

Il messaggio *«The licensed user limit has been reached»* non significa che il flusso di posta si interrompa immediatamente. Indica una sotto-licenza. Negli ambienti operativi da tempo, la causa di solito non è una crescita improvvisa, bensì gli ex collaboratori: l'account AD è stato disattivato, l'utente interno in totemomail è rimasto e continua a occupare una licenza.

La soluzione sostenibile è una sincronizzazione LDAP regolare con Active Directory. I passaggi seguenti configurano la connessione e il Cleanup-Agent e verificano l'intero percorso prima della prima esecuzione in produzione. I nomi host, i DN e gli account di servizio con `example.com` sono segnaposto e devono essere adattati al proprio ambiente.

## Quali utenti occupano una licenza

Totemomail distingue due classi di utenti. Solo gli utenti interni vengono conteggiati ai fini del limite di licenze.

| Tipo di utente | Descrizione | Rilevante per la licenza |
| --- | --- | --- |
| Internal Users | Utenti della propria organizzazione che inviano e ricevono messaggi crittografati | Sì |
| External Users | Partner di comunicazione esterni (WebMail, PDF, S/MIME, PGP) | No |


Un utente interno viene creato non appena comunica per la prima volta tramite il gateway. Ciò avviene automaticamente. La rimozione, invece, no: quando un collaboratore lascia l'organizzazione, di norma si disattiva l'account AD. Tuttavia, la voce in totemomail rimane. Negli anni si accumulano così account orfani che continuano a occupare licenze.

### Indicazione dello stato

Lo stato attuale è disponibile in **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users è impostato su* `*-17*`*. Ai 4017 utenti interni corrisponde un numero inferiore di posti licenziati.*

Le righe importanti:

-   **Internal users** (`4017`): utenti interni creati
    
-   **Internal blocked users** (`14`): bloccati, ma ancora rilevanti per la licenza
    
-   **Available Users** (`-17`): licenze disponibili; un valore negativo indica una sotto-licenza
    

Non appena *Available Users* scende sotto zero, viene visualizzato l'avviso sulla campanella:

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*«The licensed user limit has been reached.» Il flusso di posta continua, ma il messaggio rimane visibile in modo permanente.*

Importante: la sotto-licenza non blocca il flusso di posta. È una condizione contrattuale relativa alle licenze, non tecnica. C'è quindi tempo per una soluzione corretta, ma non si dovrebbe ignorare questa situazione in modo permanente.

## Dalla misura immediata alla soluzione permanente

### Eliminazione manuale

È possibile cercare ed eliminare singolarmente gli utenti interni in **Internal Users**. Questo risolve la situazione acuta, ma il problema ritorna dopo alcuni mesi. Con diverse migliaia di account non è praticabile.

### Connessione LDAP con Cleanup-Agent

La strada sostenibile è la connessione ad Active Directory tramite LDAP. Un agente confronta regolarmente gli utenti interni con la directory e rimuove o disattiva gli account che non esistono più nell'AD. In questo modo l'AD diventa la fonte autorevole e il processo di offboarding nell'AD si occupa anche dell'igiene delle licenze.

## Fondamenti LDAP

| Termine | Significato |
| --- | --- |
| DN (Distinguished Name) | Percorso univoco verso un oggetto, ad es. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Radice della ricerca, ad es. `DC=corp,DC=example,DC=com` |
| Bind DN | Account con cui totemomail si autentica nell'AD |
| Filtro | Espressione di ricerca LDAP, ad es. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Porte

| Porta | Protocollo | Utilizzo |
| --- | --- | --- |
| 389 | LDAP | non crittografato / STARTTLS |
| 636 | LDAPS | LDAP tramite TLS |
| 3268 | Global Catalog | ricerca nell'intera foresta, non crittografata |
| 3269 | Global Catalog SSL | ricerca nell'intera foresta tramite TLS |


In un ambiente a dominio singolo è sufficiente la porta 636 verso un Domain Controller. Se si gestisce una foresta con più domini, solo il Global Catalog (porta 3269) fornisce risultati per l'intera foresta. Un DC sulla porta 636 conosce esclusivamente gli oggetti del proprio dominio e risponde alle ricerche esterne alla propria partizione con un referral, un dettaglio che viene spesso trascurato negli ambienti multi-dominio.

### userAccountControl

L'informazione se un account AD è disattivato si trova nel campo di bit `userAccountControl`. Il flag `ACCOUNTDISABLE` ha valore `2`. Tramite la LDAP matching rule `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) si valutano singoli bit:

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Passaggio 1: account di servizio nell'AD

Per la connessione, creare un account dedicato con soli diritti di lettura. Non utilizzare un account amministratore. L'utente di bind deve soltanto poter leggere l'AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Passwort") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Name` | Nome visualizzato e CN del nuovo account |
| `-SamAccountName` | Nome di accesso (nome di accesso pre-Windows 2000) |
| `-UserPrincipalName` | UPN nel formato `benutzer@domäne` |
| `-Path` | OU di destinazione come Distinguished Name |
| `-AccountPassword` | Password come SecureString; `Read-Host -AsSecureString` la richiede in modo nascosto nella console |
| `-PasswordNeverExpires $true` | La password non scade |
| `-Enabled $true` | Crea l'account direttamente attivato (per impostazione predefinita sarebbe disattivato) |

</details>

Un normale utente di dominio può già leggere l'AD, quindi l'account non necessita di diritti aggiuntivi. Per la password si consiglia un valore lungo e casuale, da conservare nel proprio password manager.

Se previsto dalla propria politica di sicurezza, è possibile utilizzare anche un gMSA (Group Managed Service Account). Tuttavia, totemomail richiede Bind DN e password, per cui in pratica si usa solitamente un classico account di servizio con `PasswordNeverExpires`.

## Passaggio 2: verificare la connessione LDAP dalla riga di comando

Prima di configurare qualsiasi cosa in totemomail, è consigliabile verificare la connessione LDAP dalla riga di comando. È il passaggio che la maggior parte delle persone salta. Se `ldapsearch` funziona, funzionerà anche la connessione in totemomail. Se il test fallisce, si saprà almeno in quale punto si verifica il problema, invece di procedere per tentativi nella GUI di totemomail.

### 2.1 Verifica della porta

In Linux, ad esempio dall'appliance totemomail:

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `nc -v` | Output dettagliato: segnala se l'apertura della connessione è riuscita o meno |
| `nc -z` | Verifica solo l'apertura della connessione, senza inviare dati |
| `dc01.corp.example.com 636` | Host e porta di destinazione della verifica |
| `nmap -p 389,636,3268,3269` | Elenco delle porte TCP da verificare |
| `dc01.corp.example.com` | Host di destinazione della scansione delle porte |

</details>

In Windows con PowerShell:

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-ComputerName` | Host di destinazione della verifica di connessione |
| `-Port` | Porta TCP da verificare, qui 636 per LDAPS |

</details>

Se qui non viene stabilita alcuna connessione, si tratta di un problema di firewall o routing, non di LDAP.

### 2.2 Verificare il certificato TLS

Nella pratica, LDAPS fallisce più spesso a causa del certificato. Verificare quindi cosa restituisce il DC:

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `s_client` | Client di test TLS di OpenSSL: stabilisce la connessione e mostra i dettagli dell'handshake |
| `-connect dc01.corp.example.com:636` | Host e porta di destinazione dell'handshake TLS |
| `-showcerts` | Mostra l'intera catena di certificati fornita dal server, non solo il certificato del server |
| `</dev/null` | Chiude lo standard input affinché `s_client` termini dopo l'handshake invece di attendere input |

</details>

Prestare attenzione a due aspetti:

-   `**subject=**` **/** `**issuer=**`: il nome host nel certificato (CN o SAN) deve corrispondere al nome host usato per la connessione. Se ci si connette tramite indirizzo IP, la verifica fallisce se il certificato contiene solo l'FQDN.
    
-   `**Verify return code: 0 (ok)**`: la CA emittente deve essere nota a totemomail. Con una Enterprise CA interna, è necessario importare il relativo certificato root o issuing nel truststore di totemomail.
    

### 2.3 Bind e ricerca con ldapsearch

`ldapsearch` appartiene a `ldap-utils` (Debian/Ubuntu) o `openldap-clients` (RHEL):

```bash
ldapsearch -x \
  -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" \
  -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(sAMAccountName=jdoe))" \
  dn sAMAccountName mail userAccountControl
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Flag | Significato |
| --- | --- |
| `-x` | Simple Authentication (Bind DN e password) |
| `-H` | URI LDAP incluso schema (`ldaps://`) e porta |
| `-D` | Bind DN |
| `-W` | Richiede la password in modo interattivo |
| `-b` | Search Base |
| in seguito | Filtro, quindi gli attributi da restituire |

</details>

Se la query restituisce l'oggetto con i relativi attributi, la connessione è attiva. Il numero di account disattivati nell'AD si determina tramite il filtro di bit:

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-x`, `-H`, `-D`, `-W`, `-b` | come nella query precedente: Simple Bind tramite LDAPS con Search Base |
| `sAMAccountName` | unico attributo richiesto; mantiene l'output a una riga di attributo per risultato |
| `grep -c sAMAccountName` | conta le righe con questo attributo e quindi gli account trovati |

</details>

### 2.4 Strumenti in Windows

`**ldp.exe**` è lo strumento LDAP grafico di Microsoft, presente su ogni DC e incluso in RSAT. Ci si connette tramite `Connection → Connect` (host, porta 636, attivare SSL), ci si autentica con `Connection → Bind` e si naviga nell'albero della directory tramite `View → Tree` con il Base DN.

Senza RSAT, è possibile ottenere il risultato in PowerShell tramite ADSI Searcher:

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `[adsisearcher]"(…)"` | crea un `DirectorySearcher` con il filtro LDAP indicato |
| `SearchRoot` | punto di partenza della ricerca come percorso ADSI: server più Base DN |
| `FindOne()` | restituisce il primo risultato; `.Properties` ne mostra gli attributi |

</details>

Con RSAT e il modulo AD è più breve:

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Server` | Domain Controller su cui viene eseguita la query |
| `-SearchBase` | radice della ricerca come Distinguished Name |
| `-Filter` | filtro nella sintassi PowerShell; qui solo account attivati |
| `Measure-Object` | conta gli oggetti restituiti invece di elencarli |

</details>

In modo classico tramite `dsquery`, disponibile su ogni DC:

```bash
dsquery user -disabled -limit 0
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `user` | Tipo di oggetto della ricerca: account utente |
| `-disabled` | solo account disattivati |
| `-limit 0` | nessun limite al numero di risultati (predefinito: 100) |

</details>

Solo quando uno di questi test viene eseguito correttamente, si procede in totemomail.

## Passaggio 3: configurare la connessione LDAP in totemomail

Creare la directory LDAP nella GUI di amministrazione in **Directories / LDAP**. Utilizzare esattamente i valori testati in precedenza:

| Campo | Valore di esempio |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Password dell'account di servizio |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (in alternativa `mail` o `userPrincipalName`) |


Se si utilizza LDAPS con una CA interna, è necessario importare il relativo certificato root o issuing nel truststore di totemomail. Altrimenti l'handshake TLS fallisce con «certificate verify failed», anche se `ldapsearch` con `-x` ha funzionato in precedenza: `ldapsearch` in questa forma non verifica infatti il certificato in modo rigoroso.

Dopo il salvataggio, avviare il test di connessione integrato. Esso conferma il bind.

## Passaggio 4: creare il Cleanup-Agent

In **Maintenance → Agents → Add**, creare un agente di tipo **«Check presence of internal users in directories»**.

### 4.1 Scheda «Schedule»

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Qui l'agente viene eseguito mensilmente il giorno 1 alle 00:30. Tramite «Agent runs on server» si definisce il nodo esecutore nel cluster.*

| Campo | Raccomandazione | Motivazione |
| --- | --- | --- |
| The agent should run | `monthly`, giorno `1`, `00:30` | al di fuori dell'orario di lavoro; mensilmente è sufficiente per l'igiene delle licenze |
| Agent enabled | attivare solo dopo il test | vedere il passaggio 5 |
| Produced emails are not sent but cached in a queue | attivare per la prima esecuzione | esecuzione di test senza invio di e-mail |
| Agent runs on server | un nodo del cluster | il job deve essere eseguito su un solo nodo |


### 4.2 Scheda «Parameters»

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*I parametri controllano quali utenti interni vengono eliminati, disattivati o creati.*

| Parametro | Raccomandazione | Effetto |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | attivare | Gli utenti interni inattivi senza voce AD vengono eliminati. È il fulcro della pulizia delle licenze. |
| Delete blocked users that are not found in a directory? | attivare | Vengono eliminati anche gli utenti interni bloccati senza voce AD |
| Delete administrators? | lasciare vuoto | Gli account amministratore non devono essere eliminati automaticamente |
| Only set users found in the defined groups to inactive | facoltativo | Gli utenti vengono impostati su inattivi anziché eliminati. Un `!` iniziale esclude i membri del gruppo indicato. Separare i DN con `;`. |
| Additional filter attribute | facoltativo | attributo aggiuntivo per la ricerca nella directory, ad es. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | lasciare vuoto | si applica solo se è impostato il parametro del gruppo |
| Create users based on group membership | facoltativo | crea nuovi utenti interni in base all'appartenenza a gruppi AD. Separare più gruppi con `;`. |


La negazione nel campo *«Only set users found in the defined groups to inactive»* funziona tramite un `!` prima di un Group DN. I membri di questo gruppo vengono esclusi dall'azione:

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

In questo esempio, gli utenti del gruppo *Mitarbeiter* vengono impostati su inattivi in caso di assenza dall'AD, mentre i membri del gruppo *Dienstkonten* rimangono invariati.

## Passaggio 5: esecuzione di test e convalida

Non eseguire l'agente sul parco di produzione senza un test. Procedere invece nel seguente ordine:

1.  **Attivare la modalità coda**: tramite l'opzione *«Produced emails are not sent but cached in a queue»*. L'agente determina le azioni pianificate senza inviare e-mail.
    
2.  **Eseguire manualmente** e valutare il log dell'agente: quanti utenti sarebbero coinvolti e ci sono account inattesi, come caselle funzionali, nell'elenco?
    
3.  **Verificare la plausibilità con** `**ldapsearch**`: il numero di utenti non trovati nell'AD deve corrispondere alla query LDAP manuale.
    
4.  Se il risultato è corretto, disattivare la modalità coda, impostare *Agent enabled* e attivare lo schedule.
    
5.  Dopo la prima esecuzione in produzione, controllare nuovamente **Settings → Overview → User Information**. *Available Users* dovrebbe tornare a un valore positivo.
    

## Risoluzione dei problemi

| Sintomo | Causa | Misura |
| --- | --- | --- |
| `Can't contact LDAP server` | Porta 636 non raggiungibile / host errato | verificare con `Test-NetConnection` o `nc -vz`, controllare il firewall |
| `Invalid credentials (49)` | Bind DN o password errati | specificare il Bind DN come DN completo, non come `user@domain` |
| `certificate verify failed` | CA sconosciuta al truststore | importare la Root CA o la Issuing CA |
| Mancata corrispondenza del nome host in TLS | connessione tramite IP invece di FQDN | usare CN/SAN del certificato come host |
| `Referral (10)` | La ricerca supera il confine del dominio | usare Global Catalog sulla porta 3269 anziché DC sulla 636 |
| Gli utenti disattivati non vengono riconosciuti | filtro `userAccountControl` mancante | usare la bit matching rule `:1.2.840.113556.1.4.803:=2` |
| L'agente elimina troppi account | filtro troppo ampio / Base DN errato | testare in modalità coda, restringere il Base DN |


Con il flag `-d 1`, `ldapsearch` fornisce l'output di debug dell'apertura della connessione:

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-d 1` | Livello di debug 1: registra il processo di apertura della connessione, incluso l'handshake TLS, su stderr |
| `-x`, `-H` | Simple Bind e URI LDAP come nelle query precedenti |

</details>

In questo modo è possibile vedere se fallisce l'handshake TLS o solo il bind. La GUI di totemomail non mostra questa distinzione dietro il suo messaggio di errore generico.

## Sicurezza

-   **Account di servizio in sola lettura.** L'utente di bind necessita esclusivamente di diritti di lettura.
    
-   **LDAPS anziché LDAP.** Utilizzare la porta 636 o 3269. LDAP sulla porta 389 trasmette la password di bind in chiaro. Active Directory impone inoltre sempre più connessioni protette tramite LDAP Channel Binding e Signing.
    
-   **Rotazione della password.** `PasswordNeverExpires` è praticabile operativamente. Documentare l'account e ruotare la password secondo pianificazione.
    
-   **Monitoraggio.** Monitorare *Available Users* (idealmente tramite alerting), invece di aspettare l'avviso sulla campanella.
    
-   **Prima esecuzione in modalità coda.** Un filtro errato può coinvolgere un gran numero di account.
    

## La procedura sicura in quattro passaggi

Il raggiungimento del limite di licenze non è un difetto tecnico, ma la conseguenza di un processo di offboarding mancante. La soluzione sostenibile è il confronto regolare con Active Directory come fonte autorevole. L'ordine è decisivo:

1.  Verificare la connessione LDAP dalla riga di comando (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configurare la connessione in totemomail
    
3.  Convalidare l'agente in modalità coda
    
4.  Mettere l'agente in produzione
    

Chi rispetta questo ordine risolve il problema di licenza acuto e ne impedisce il ripetersi.

## Fonti

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Documentazione del prodotto su totemomail (modello di licenza, connessione LDAP, Cleanup-Agent); la tecnologia è ora proseguita da Kiteworks come Email Protection Gateway.
    
2.  [Microsoft Learn – «UserAccountControl property flags»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Significato dei flag, tra cui `ACCOUNTDISABLE` (0x0002) e `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – «Search Filter Syntax»](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Filtro LDAP bit per bit tramite la matching rule OID `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – «ldapsearch» (Manpage)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Opzioni di chiamata (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) per bind e ricerca.
    
5.  [Microsoft Learn – «Service overview and network port requirements»](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): Porte LDAP 389/636 e porte Global Catalog 3268/3269.
