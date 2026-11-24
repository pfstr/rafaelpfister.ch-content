---
title: "S/MIME con indicazione dell’organizzazione: organizationIdentifier e UID nel certificato"
navTitle: "S/MIME con organizzazione"
description: "Cosa contiene un certificato S/MIME con indicazione dell’organizzazione, come l’UID viene inserito nel certificato come organizationIdentifier e cosa verifica a tale scopo l’autorità di certificazione. Include un confronto tra i prodotti di SwissSign e DigiCert, comandi per verificare il tipo di certificato e la distinzione rispetto a ZertES."
date: "2026-12-15"
kategorie: "Crittografia e-mail"
timeToRead: "11 min di lettura"
themen:
  - e-mail-verschluesselung
produkte: []
protokolle:
  - "verschluesselung"
  - "powershell"
hauptthema: "e-mail-verschluesselung"
startseiteAb: "2026-12-15"
slug: "s-mime-con-indicazione-dell-organizzazione-organizationidentifier-e-uid-nel-certificato"
translationId: "article-e0f9bcc57f360835"
translationOf: smime-organisation-uid
url: https://rafaelpfister.ch/it/blog/s-mime-con-indicazione-dell-organizzazione-organizationidentifier-e-uid-nel-certificato
translationSourceHash: f628a8f86635449d2a9e43f78f1a960ce5c8bc20af519eac110b77b2dad385a9
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T13:53:09.215Z
translationReview: automatic
---

# S/MIME con indicazione dell’organizzazione: organizationIdentifier e UID nel certificato

Le autorità di certificazione (CA) distinguono tra certificati S/MIME con e senza indicazione dell’organizzazione. Oltre all’indirizzo e-mail, un certificato con indicazione dell’organizzazione attesta anche l’azienda cui appartiene la casella di posta: con il nome verificato e un numero di registro. Per le aziende svizzere, questo numero di registro è l’UID, ossia il numero CHE del registro di commercio.

In breve: un’indicazione dell’organizzazione esiste solo per i tipi di certificato `Organization-validated` e `Sponsor-validated` dei S/MIME Baseline Requirements. È costituita dal nome dell’azienda nel campo `O` e dal campo `organizationIdentifier` (OID 2.5.4.97), per un’azienda svizzera ad esempio `NTRCH-CHE-123.456.789`. Prima di inserire tali dati nel certificato, la CA verifica l’azienda in un registro ufficiale. Un certificato convalidato sulla casella di posta, invece, dimostra soltanto l’accesso a una casella di posta. Per SwissSign ciò corrisponde alla scelta tra Silver e Gold, mentre per DigiCert alla scelta tra «Secure Email for Individual Mailbox» e i prodotti «for Employee» e «for Organization». L’indicazione dell’organizzazione non crea una firma qualificata ai sensi della ZertES.

**Focus Svizzera:** la classificazione giuridica segue il diritto svizzero (ZertES, CO). Laddove le norme UE differiscono, viene fornita un’indicazione contrassegnata.

## I quattro tipi di certificato

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` (BR) del CA/Browser Forum. Essi distinguono quattro tipi di certificato in base a chi è indicato come titolare (`Subject`) nel certificato:

| Tipo | Titolare nel certificato | Indicazione dell’organizzazione | Cosa verifica la CA |
|---|---|---|---|
| `Mailbox-validated` (MV) | solo l’indirizzo e-mail | non ammessa | controllo della casella di posta o del dominio e-mail |
| `Organization-validated` (OV) | un’organizzazione, nessuna persona | obbligatoria | organizzazione nonché casella di posta o dominio e-mail |
| `Sponsor-validated` (SV) | una persona appartenente a un’organizzazione | obbligatoria | come OV, più la persona; la CA o l’organizzazione stessa verifica quest’ultima |
| `Individual-validated` (IV) | una persona senza organizzazione | non ammessa | identità della persona e casella di posta |

Con «S/MIME con indicazione dell’organizzazione» si intendono i tipi OV e SV. Per entrambi, i BR prescrivono obbligatoriamente i campi `organizationName` e `organizationIdentifier`, in tutte e tre le generazioni `Legacy`, `Multipurpose` e `Strict`. Le generazioni si differenziano per i campi Subject consentiti, le estensioni e gli scopi d’uso. I BR hanno dismesso la generazione `Legacy` al 15 luglio 2025; da allora SwissSign e DigiCert non la rilasciano più.

I certificati OV sono adatti alle caselle funzionali, quali `info@` o `buchhaltung@`, che non sono assegnate a una persona. I certificati SV sono destinati alle caselle personali dei collaboratori e contengono inoltre il nome della persona o uno pseudonimo.

## Struttura dell’organizationIdentifier

Il campo `organizationIdentifier` (OID 2.5.4.97) contiene un numero di registro dell’organizzazione. Lo stesso campo viene utilizzato anche dai certificati secondo eIDAS e ZertES. Per i certificati S/MIME, i BR prescrivono la seguente struttura:

| Parte | Esempio | Significato |
|---|---|---|
| Schema di registrazione, 3 caratteri | `NTR` | `National Trade Register`, il registro di commercio |
| Codice paese secondo ISO 3166-1 | `CH` | Paese in cui è tenuto il registro |
| Trattino | `-` | Separatore |
| Numero di registro | `CHE-123.456.789` | Numero dell’organizzazione nel registro |

Combinati, danno `NTRCH-CHE-123.456.789`. Poiché l’UID stesso contiene un trattino, i BR stabiliscono che lo schema e il codice paese non contengano mai trattini. Il primo trattino da sinistra separa quindi lo schema dal numero di registro, mentre tutti gli altri appartengono al numero. Se un registro assegna numeri soltanto a livello di Stato federale o provincia, dopo il codice paese segue un `+` con il codice della regione, ad esempio `NTRUS+CA-12345678` per la California.

I BR prevedono i seguenti schemi di registrazione:

| Schema | Origine del numero | Esempio dai BR |
|---|---|---|
| `NTR` | registro di commercio o delle imprese | `NTRGB-12345678` |
| `VAT` | autorità fiscale (numero IVA) | `VATEL-123456789` |
| `PSD` | numero di autorizzazione di un prestatore di servizi di pagamento ai sensi della direttiva UE 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` secondo ISO 17442, codice paese sempre `XG` | `LEIXG-` seguito dal LEI |
| `GOV` | autorità senza numero in uno degli altri schemi | `GOVUS` |
| `INT` | organizzazioni internazionali | `INTXG` |

Per le aziende svizzere, `NTR` con l’UID è la norma. L’UID è assegnato dall’Ufficio federale di statistica. I registri di commercio lo utilizzano come numero dell’impresa, serve anche da numero IVA e rimane invariato quando un’azienda trasferisce la propria sede in un altro Cantone. Un numero IVA, invece, può decadere quando termina l’assoggettamento fiscale e non tutte le organizzazioni ne possiedono uno. SwissSign riporta l’UID con i punti, come figura nel registro UID, e indica come esempio `NTRCH-CHE-109.357.012` per SwissSign AG.

**Nota UE:** in Germania, i tribunali locali assegnano i numeri del registro di commercio e lo stesso numero ricorre presso più tribunali. Dalla versione 1.0.9 del maggio 2025, i BR consentono pertanto nello schema `NTR` l’`European Unique Identifier` (EUID) e lo raccomandano per le aziende tedesche, ad esempio `NTRDE-DER3306.HRB12345` per un’azienda iscritta nel registro di commercio del tribunale locale di Colonia. Secondo i BR, il numero di registro deve essere univoco solo laddove il registro e l’ordinamento giuridico assegnino numeri univoci.

## Cosa verifica la CA

Prima di rilasciare un certificato OV o SV, una CA deve verificare secondo i BR le seguenti informazioni:

- **Nome ed esistenza dell’organizzazione:** tramite un’autorità del Paese dell’organizzazione, dati LEI, una visita in loco o una conferma con estratto del registro. Per un’azienda svizzera si tratta del registro di commercio e del registro UID. Se l’azienda vi risulta cancellata o inattiva, la CA non può rilasciare il certificato.
- **Numero di registro:** la CA conferma che il numero nell’`organizationIdentifier` appartiene all’organizzazione nel campo `O`.
- **Procura:** attraverso un canale di comunicazione affidabile, la CA verifica che la persona richiedente sia autorizzata a richiedere certificati per l’organizzazione o ad agire come `Enterprise RA`.
- **Casella di posta o dominio e-mail:** la CA verifica il controllo degli indirizzi e-mail presenti nel certificato. Se l’organizzazione agisce come `Enterprise RA`, deve dimostrare il controllo dell’intero dominio e-mail.
- **Persona (solo SV):** i BR consentono all’organizzazione, in qualità di `Enterprise RA`, di confermare direttamente i propri collaboratori. La CA verifica quindi l’organizzazione e non ogni singola persona.

Una CA può riutilizzare i risultati di tali verifiche: la verifica dell’organizzazione e della procura per 825 giorni, la verifica del controllo del dominio per 398 giorni. Un certificato delle generazioni `Multipurpose` e `Strict` è valido al massimo 825 giorni.

## Confronto tra SwissSign e DigiCert

Entrambe le CA offrono i tipi MV, SV e OV con propri nomi di prodotto. Nessuna delle due offre un certificato IV. La struttura dell’`organizationIdentifier` è definita dai BR e vale allo stesso modo per entrambi i fornitori.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (precedentemente Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | non disponibile | non disponibile |
| Generazione per SV e OV | `Multipurpose` secondo i profili dei certificati | `Strict` come standard, `Multipurpose` selezionabile |
| Persona per SV | nome e cognome o pseudonimo con prefisso `pseudo:` nel `CN` | nome e cognome o pseudonimo; nel `CN` è possibile scegliere il nome o l’indirizzo e-mail |
| Usi aggiuntivi per SV | nel profilo con autenticazione: autenticazione client, accesso con smart card, EFS | con `Strict` non ripudio, con `Multipurpose` anche crittografia dei dati e autenticazione client |

DigiCert descrive esplicitamente il prodotto SV secondo il modello dell’`Enterprise RA`: l’organizzazione conferma che la persona nel certificato appartiene ai propri collaboratori o rappresentanti e conserva le relative prove. DigiCert verifica soltanto l’organizzazione. Per OV e SV, DigiCert richiede sempre la prova del controllo del dominio. La verifica dell’organizzazione è valida 825 giorni, quella del dominio 398 giorni. Si può scegliere una durata di uno o due anni oppure una data di scadenza personalizzata fino a un massimo di 824 giorni. DigiCert ha ritirato il 10 luglio 2025 i precedenti prodotti Premium, Email Security Plus, Digital Signature Plus e Class 1 S/MIME, sostituendoli con questi prodotti.

Con SwissSign, il certificato OV contiene nel `CN` il nome dell’organizzazione come nel campo `O`. L’articolo [S/MIME nelle banche: situazione giuridica in Svizzera e scelta tra Silver e Gold](/blog/smime-silver-gold-schweiz) descrive quale livello una banca dovrebbe scegliere per quale casella di posta e cosa ciò comporta sul piano giuridico.

## Verificare il tipo di certificato

In Outlook e in altri programmi di posta è possibile aprire i dettagli del certificato di un’e-mail firmata tramite l’icona della firma. In Windows, i campi sono disponibili nella scheda «Dettagli» alla voce «Richiedente». Per l’OID 2.5.4.97, Windows non conosce un nome visualizzato e lo mostra come `OID.2.5.4.97`.

### Visualizzare il titolare con PowerShell

Per un certificato esportato (ad esempio dai dettagli del certificato di un’e-mail firmata):

```powershell
$zert = Get-PfxCertificate -FilePath .\zertifikat.cer
$zert.SubjectName.Format($true)
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Get-PfxCertificate` | legge un certificato da un file, anche da un file `.cer` senza chiave privata |
| `-FilePath .\zertifikat.cer` | percorso del file del certificato |
| `SubjectName.Format($true)` | restituisce i campi del titolare riga per riga |

</details>

Output per un certificato SV (esempio):

```text
CN=Max Muster
G=Max
SN=Muster
O=Muster AG
OID.2.5.4.97=NTRCH-CHE-123.456.789
S=ZH
C=CH
```

In un certificato MV, l’output contiene solo `CN=` con l’indirizzo e-mail.

### Determinare tipo e generazione tramite la Policy OID

Ogni certificato secondo i BR contiene nell’estensione `Certificate Policies` un OID secondo il modello `2.23.140.1.5.<Typ>.<Generation>`:

| Posizione | Valore | Significato |
|---|---|---|
| Tipo | `1` | `Mailbox-validated` |
| Tipo | `2` | `Organization-validated` |
| Tipo | `3` | `Sponsor-validated` |
| Tipo | `4` | `Individual-validated` |
| Generazione | `1` | `Legacy` |
| Generazione | `2` | `Multipurpose` |
| Generazione | `3` | `Strict` |

```powershell
$zert.Extensions |
  Where-Object { $_.Oid.Value -eq '2.5.29.32' } |
  ForEach-Object { $_.Format($true) }
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `$zert.Extensions` | estensioni del certificato letto in precedenza |
| `'2.5.29.32'` | OID dell’estensione `Certificate Policies` |
| `Format($true)` | restituisce il contenuto dell’estensione su più righe |

</details>

Output (abbreviato):

```text
[1]Certificate Policy:
     Policy Identifier=2.23.140.1.5.3.2
```

`2.23.140.1.5.3.2` indica un certificato SV della generazione `Multipurpose`. Ulteriori voci con OID della CA stessa sono normali.

### Elencare i certificati S/MIME nel proprio archivio certificati

```powershell
Get-ChildItem Cert:\CurrentUser\My |
  Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains '1.3.6.1.5.5.7.3.4' } |
  Select-Object Subject, NotAfter
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Cert:\CurrentUser\My` | archivio certificati personale dell’utente connesso |
| `EnhancedKeyUsageList.ObjectId` | usi estesi della chiave del certificato |
| `'1.3.6.1.5.5.7.3.4'` | OID per la protezione e-mail (`id-kp-emailProtection`) |
| `Select-Object Subject, NotAfter` | mostra titolare e data di scadenza |

</details>

### Con OpenSSL

```bash
openssl x509 -in zertifikat.pem -noout -subject -nameopt multiline
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `x509` | sottocomando per certificati |
| `-in zertifikat.pem` | file di input in formato PEM; per un file DER, aggiungere `-inform der` |
| `-noout` | sopprime l’output del certificato codificato |
| `-subject` | restituisce il titolare |
| `-nameopt multiline` | un campo per riga con nomi dei campi per esteso |

</details>

OpenSSL 3 conosce il nome del campo e lo mostra per esteso:

```text
subject=
    commonName                = Max Muster
    givenName                 = Max
    surname                   = Muster
    organizationName          = Muster AG
    organizationIdentifier    = NTRCH-CHE-123.456.789
    stateOrProvinceName       = ZH
    countryName               = CH
```

### Verificare il numero di registro

Come destinatari di un’e-mail firmata, potete verificare l’indicazione dell’organizzazione in quattro passaggi:

1. Il programma di posta segnala la firma come valida e il certificato proviene da una CA di cui il programma di posta si fida.
2. La Policy OID inizia con `2.23.140.1.5.2` o `2.23.140.1.5.3`, e il certificato contiene i campi `O` e `organizationIdentifier`.
3. L’UID dell’`organizationIdentifier` appartiene nel registro UID all’azienda prevista e l’azienda è attiva.
4. Il dominio dell’indirizzo del mittente appartiene a tale azienda.

## Distinzione rispetto alla ZertES

L’indicazione dell’organizzazione non trasforma un certificato S/MIME in un certificato regolamentato ai sensi della ZertES. I certificati regolamentati sono rilasciati solo da prestatrici riconosciute ai sensi della ZertES e il certificato deve contenere l’indicazione che si tratta di un certificato regolamentato (art. 2 lett. g e art. 7 cpv. 2 lett. b ZertES).

Sul piano dei contenuti vi sono analogie. Un certificato regolamentato per un’unità UID deve contenere il relativo UID (art. 7 cpv. 2 lett. e ZertES), così come l’`organizationIdentifier` lo contiene per OV e SV. Un certificato per una persona fisica può inoltre contenere l’indicazione che essa è autorizzata a rappresentare una determinata unità UID (art. 7 cpv. 3 lett. b ZertES); nella struttura ciò corrisponde a un certificato SV.

| | S/MIME con indicazione dell’organizzazione | certificato regolamentato secondo ZertES |
|---|---|---|
| Base | S/MIME Baseline Requirements del CA/Browser Forum | ZertES e disposizioni di esecuzione |
| Emittente | CA pubblicamente attendibile | prestatrice riconosciuta ai sensi della ZertES |
| Identificativo dell’organizzazione | `organizationIdentifier`, per le aziende svizzere con l’UID | UID, obbligatorio per le unità UID |
| Verifica della persona | per SV dalla CA o dall’organizzazione in qualità di `Enterprise RA` | in linea di principio presenza personale con prova dell’identità; per le unità UID, un rappresentante con prova del potere di rappresentanza (art. 9 ZertES) |
| Chiave | secondo le prescrizioni della CA, anche su un gateway | requisiti ai sensi dell’art. 6 ZertES |

Solo la firma elettronica qualificata con marca temporale qualificata è equiparata alla firma autografa ai sensi dell’art. 14 cpv. 2bis CO, e un certificato qualificato può essere rilasciato soltanto a una persona fisica (art. 8 cpv. 1 ZertES). Per la firma di un’organizzazione, la ZertES prevede il sigillo elettronico regolamentato: una firma elettronica avanzata creata con un’unità sicura di creazione del sigillo e basata su un certificato regolamentato di un’unità UID (art. 2 lett. d ZertES). Un’e-mail firmata S/MIME è un mezzo di prova il cui valore è liberamente valutato dal giudice (art. 157 CPC); un’indicazione dell’organizzazione verificata supporta l’attribuzione all’azienda mittente.

**Nota UE:** nell’UE, il regolamento eIDAS disciplina le firme e i sigilli elettronici. Ai sensi dell’art. 25 cpv. 2 eIDAS, anche qui soltanto la firma elettronica qualificata è equiparata alla firma autografa.

## Raccomandazione

- **Caselle funzionali:** certificato OV (SwissSign Organization Validated, DigiCert Secure Email for Organization). Attesta l’azienda senza menzionare una persona.
- **Caselle personali con firma nei confronti di clienti e partner:** certificato SV (SwissSign Gold, DigiCert Secure Email for Employee). Attesta la persona e l’azienda.
- **Solo crittografia:** dal punto di vista crittografico, tutti i tipi sono equivalenti. Per la sola crittografia è sufficiente un certificato MV.
- **Pianificare le verifiche:** la verifica dell’organizzazione è valida 825 giorni, quella del dominio 398 giorni. Dopo una modifica del nome dell’azienda o della sede, i dati nel certificato non corrispondono più al registro; pianificate quindi nuovi certificati.
- **Verificare in ricezione:** Policy OID e UID mostrano se una firma è effettivamente attribuita a un’azienda verificata.

## Fonti

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): versione 1.0.17 del 20 novembre 2026; tipi di certificato e Policy OID (sezione 1.1), verifica dell’organizzazione (3.2.3), verifica della procura (3.2.6), riutilizzo dei risultati delle verifiche (4.2.1), durate di validità (6.3.2), campi Subject (7.1.4.2) e schemi di registrazione (appendice A).

2.  [cabforum/smime su GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): versione Markdown dei S/MIME Baseline Requirements con cronologia delle versioni, tra cui EUID dalla versione 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): denominazioni dei prodotti Silver e Gold, esempio `NTRCH-CHE-109.357.012`, UID come numero di registro ed EUID per le aziende tedesche (28.04.2025).

4.  [SwissSign CPR S/MIME](https://repository.swisssign.com/SwissSign_CPR_SMIME_R14.pdf): profili dei certificati SwissSign per MV, SV e OV con campi Subject, scopi d’uso e Policy OID, versione 14 del 18.09.2025.

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): panoramica dei prodotti con assegnazione a MV, OV e SV, fine della generazione `Legacy` e dei precedenti prodotti S/MIME il 10 luglio 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): verifica dell’organizzazione e del dominio, conferma dei collaboratori da parte dell’organizzazione, profili `Strict` e `Multipurpose`, durate di validità.

7.  [Legge federale concernente la firma elettronica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (termini, sigillo regolamentato), art. 6 (chiavi), art. 7 (contenuto dei certificati regolamentati, UID), art. 8 (certificati qualificati) e art. 9 (verifica dell’identità).

8.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull’equiparazione della firma elettronica qualificata alla firma autografa.

9.  [Registro UID](https://www.uid.admin.ch): consultazione di un UID con nome, sede e stato, gestito dall’Ufficio federale di statistica.

10.  [Zefix](https://www.zefix.ch): indice centrale delle imprese dei registri di commercio cantonali.

11.  [Codice di procedura civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sulla libera valutazione delle prove.

12.  [Regolamento (UE) n. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25 cpv. 2 sull’equiparazione della firma elettronica qualificata alla firma autografa.
