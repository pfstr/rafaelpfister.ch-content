---
title: "S/MIME con voce organizzativa: organizationIdentifier e UID nel certificato"
navTitle: "S/MIME con organizzazione"
description: "Cosa contiene un certificato S/MIME con voce organizzativa, come l'UID viene inserito nel certificato come organizationIdentifier e cosa verifica l'autorità di certificazione per l'azienda e la persona. Include un confronto tra SwissSign e DigiCert, comandi per verificare il tipo di certificato e la distinzione rispetto a ZertES."
date: "2026-12-15"
kategorie: "Crittografia e-mail"
timeToRead: "13 min di lettura"
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
translationSourceHash: 849361658c86c3e7bae12ee2d7e018cf43807a611de4911b9ad1bdbe870ea0fe
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T14:56:15.486Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/s-mime-con-indicazione-dell-organizzazione-organizationidentifier-e-uid-nel-certificato
---

# S/MIME con voce organizzativa: organizationIdentifier e UID nel certificato

Le autorità di certificazione (CA) distinguono i certificati S/MIME con e senza voce organizzativa. Oltre all'indirizzo e-mail, un certificato con voce organizzativa indica anche l'azienda a cui appartiene la casella di posta: con il suo nome verificato e un numero di registro. Per le aziende svizzere, questo numero di registro è l'UID, ossia il numero CHE del registro di commercio.

In breve: una voce organizzativa esiste solo per i tipi di certificato `Organization-validated` e `Sponsor-validated` dei S/MIME Baseline Requirements. È composta dal nome dell'azienda nel campo `O` e dal campo `organizationIdentifier` (OID 2.5.4.97), per un'azienda svizzera ad esempio `NTRCH-CHE-123.456.789`. Prima di inserire queste informazioni nel certificato, la CA verifica l'azienda in un registro ufficiale. Un certificato convalidato per la casella di posta, invece, attesta soltanto l'accesso a una casella di posta. Con SwissSign, ciò corrisponde alla scelta tra Silver e Gold; con DigiCert, alla scelta tra «Secure Email for Individual Mailbox» e i prodotti «for Employee» e «for Organization». Una firma qualificata ai sensi di ZertES non deriva dalla voce organizzativa.

**Focus Svizzera:** la classificazione giuridica segue il diritto svizzero (ZertES, CO). Laddove le regole dell'UE differiscono, è riportata un'apposita nota.

## I quattro tipi di certificato

Dal 1° settembre 2023, per i certificati S/MIME pubblicamente attendibili si applicano i `S/MIME Baseline Requirements` (BR) del CA/Browser Forum. Questi distinguono quattro tipi di certificato in base a chi figura come titolare (`Subject`) nel certificato:

| Tipo | Titolare nel certificato | Voce organizzativa | Cosa verifica la CA |
|---|---|---|---|
| `Mailbox-validated` (MV) | solo l'indirizzo e-mail | non consentita | controllo sulla casella di posta o sul dominio e-mail |
| `Organization-validated` (OV) | un'organizzazione, nessuna persona | obbligatoria | organizzazione nonché casella di posta o dominio e-mail |
| `Sponsor-validated` (SV) | una persona appartenente a un'organizzazione | obbligatoria | come OV, oltre alla persona; la verifica è svolta dalla CA o dall'organizzazione stessa |
| `Individual-validated` (IV) | una persona senza organizzazione | non consentita | identità della persona e casella di posta |

Con «S/MIME con voce organizzativa» si intendono i tipi OV e SV. Per entrambi, i BR prescrivono obbligatoriamente i campi `organizationName` e `organizationIdentifier`, in tutte e tre le generazioni `Legacy`, `Multipurpose` e `Strict`. Le generazioni differiscono per i campi Subject, le estensioni e gli usi consentiti. I BR hanno ritirato la generazione `Legacy` il 15 luglio 2025; da allora SwissSign e DigiCert non la rilasciano più.

I certificati OV sono adatti a caselle funzionali come `info@` o `buchhaltung@`, che non sono assegnate a una persona. I certificati SV sono destinati alle caselle personali dei collaboratori e contengono inoltre il nome della persona o uno pseudonimo.

## Struttura dell'organizationIdentifier

Il campo `organizationIdentifier` (OID 2.5.4.97) contiene un numero di registro dell'organizzazione. Lo stesso campo è utilizzato anche dai certificati ai sensi di eIDAS e ZertES. Per i certificati S/MIME, i BR prescrivono la seguente struttura:

| Parte | Esempio | Significato |
|---|---|---|
| Schema di registrazione, 3 caratteri | `NTR` | `National Trade Register`, il registro di commercio |
| Codice Paese secondo ISO 3166-1 | `CH` | Paese in cui è tenuto il registro |
| Trattino | `-` | Separatore |
| Numero di registro | `CHE-123.456.789` | Numero dell'organizzazione nel registro |

Combinati, danno `NTRCH-CHE-123.456.789`. Poiché l'UID stesso contiene un trattino, i BR stabiliscono che lo schema e il codice Paese non contengano mai un trattino. Il primo trattino da sinistra separa dunque lo schema dal numero di registro; tutti gli altri fanno parte del numero. Se un registro assegna numeri solo a livello di Stato federato o provincia, dopo il codice Paese segue un `+` con il codice della regione, ad esempio `NTRUS+CA-12345678` per la California.

I BR prevedono questi schemi di registrazione:

| Schema | Origine del numero | Esempio dai BR |
|---|---|---|
| `NTR` | registro commerciale o delle imprese | `NTRGB-12345678` |
| `VAT` | autorità fiscale (numero IVA) | `VATEL-123456789` |
| `PSD` | numero di autorizzazione di un prestatore di servizi di pagamento ai sensi della direttiva UE 2015/2366 | `PSDBE-NBB-1234.567.890` |
| `LEI` | `Legal Entity Identifier` secondo ISO 17442, codice Paese sempre `XG` | `LEIXG-` seguito dal LEI |
| `GOV` | autorità senza numero in uno degli altri schemi | `GOVUS` |
| `INT` | organizzazioni internazionali | `INTXG` |

Per le aziende svizzere, `NTR` con l'UID costituisce il caso standard. L'UID è assegnato dall'Ufficio federale di statistica. I registri di commercio lo usano come numero dell'azienda, serve anche come numero IVA e resta invariato se un'azienda trasferisce la propria sede in un altro Cantone. Un numero IVA può invece decadere se termina l'assoggettamento all'imposta, e non tutte le organizzazioni ne possiedono uno. SwissSign riprende l'UID con i punti, come appare nel registro UID, e cita come esempio `NTRCH-CHE-109.357.012` per SwissSign AG.

**Nota UE:** in Germania, i tribunali locali assegnano i numeri del registro di commercio e lo stesso numero può comparire presso più tribunali. Dalla versione 1.0.9 del maggio 2025, i BR consentono quindi nello schema `NTR` la `European Unique Identifier` (EUID) e la raccomandano per le aziende tedesche, ad esempio `NTRDE-DER3306.HRB12345` per un'azienda iscritta nel registro di commercio del tribunale locale di Colonia. Secondo i BR, il numero di registro deve essere univoco solo laddove il registro e l'ordinamento giuridico assegnano numeri univoci.

## Cosa verifica la CA

Prima di rilasciare un certificato OV o SV, una CA deve verificare le seguenti informazioni secondo i BR:

- **Nome ed esistenza dell'organizzazione:** tramite un'autorità del Paese dell'organizzazione, dati LEI, una visita in loco o una conferma con estratto del registro. Per un'azienda svizzera, si tratta del registro di commercio e del registro UID. Se l'azienda vi risulta cancellata o inattiva, la CA non può rilasciare il certificato.
- **Numero di registro:** la CA conferma che il numero nell'`organizationIdentifier` appartiene all'organizzazione nel campo `O`.
- **Potere di rappresentanza:** attraverso un canale di comunicazione affidabile, la CA verifica che la persona richiedente sia autorizzata a richiedere certificati per l'organizzazione o ad agire come `Enterprise RA`.
- **Casella di posta o dominio e-mail:** la CA verifica il controllo sugli indirizzi e-mail nel certificato. Se l'organizzazione agisce come `Enterprise RA`, deve dimostrare il controllo sull'intero dominio e-mail.
- **Persona (solo SV):** nome o pseudonimo della persona. La sezione successiva descrive quali prove sono ammesse a tale scopo.

Una CA può riutilizzare i risultati di tali verifiche: la verifica dell'organizzazione e del potere di rappresentanza per 825 giorni, la verifica del controllo del dominio per 398 giorni. Un certificato delle generazioni `Multipurpose` e `Strict` è valido al massimo 825 giorni.

## Come viene verificata la persona nei certificati SV

I BR ammettono le seguenti prove per l'identità della persona:

| Prova | Procedura |
|---|---|
| documento d'identità ufficiale | originale in loco o in tempo reale davanti a una videocamera; la persona incaricata della verifica confronta la foto del documento con la persona e registra emittente, validità e numero del documento |
| chip nel passaporto o nella carta d'identità | verifica della firma dell'emittente sui dati del chip, oltre al confronto della foto |
| eID | solo sistemi eID notificati ai sensi di eIDAS con livello di garanzia «sostanziale» o «elevato» |
| patente di guida mobile (mDL) | interrogazione in tempo reale secondo ISO/IEC 18013-5 e 18013-7 con verifica della catena di firme dell'emittente |
| firma digitale | firma della richiesta con un certificato personale esistente di un quadro riconosciuto; l'elenco di tali quadri è ancora vuoto nei BR |
| conferma di un notaio o di uno studio legale | conferma scritta del Paese della persona richiedente |
| documenti del `Enterprise RA` | l'azienda verifica autonomamente i propri collaboratori e può basarsi su dati interni esistenti, ad esempio delle risorse umane |

Quali di queste prove una CA accetti deve essere pubblicato nel suo CPS. Al momento del rilascio, la verifica della persona non può risalire a più di 825 giorni. Se l'azienda si verifica autonomamente come `Enterprise RA`, la CA deve vincolarla contrattualmente e controllarla almeno una volta all'anno, salvo che l'azienda sia essa stessa sottoposta a revisione annuale. In questo caso, l'associazione alla persona è affidabile quanto il processo interno dell'azienda.

## Confronto tra SwissSign e DigiCert

Entrambe le CA offrono i tipi MV, SV e OV con nomi di prodotto propri. Nessuna delle due offre un certificato IV. La struttura dell'`organizationIdentifier` è stabilita dai BR ed è identica per entrambi i fornitori.

| | SwissSign | DigiCert |
|---|---|---|
| MV | Personal S/MIME E-Mail ID Silver | Secure Email for Individual Mailbox |
| SV | Pro S/MIME E-Mail ID Gold | Secure Email for Employee (in precedenza Secure Email for Business) |
| OV | Organization Validated E-Mail Certificate | Secure Email for Organization |
| IV | non disponibile | non disponibile |
| Acquisto | Silver nel webshop o tramite Managed PKI, Gold e OV solo tramite la Managed PKI dell'azienda | tutti e tre tramite l'account CertCentral dell'azienda |
| Verifica della persona per SV | documento d'identità ufficiale in originale, in loco o con procedura equivalente; altre prove solo con l'approvazione di SwissSign | conferma da parte dell'azienda; DigiCert non verifica la persona |
| Validità delle verifiche | dati e prove al massimo 398 giorni | organizzazione 825 giorni, dominio 398 giorni |
| Generazione per SV e OV | `Multipurpose`; una CA intermedia per SV nella generazione `Strict` è già descritta nel CP/CPS | `Strict` come standard, `Multipurpose` selezionabile |
| Persona per SV | nome e cognome o pseudonimo con prefisso `pseudo:` nel `CN` | nome e cognome o pseudonimo; nel `CN` il nome o l'indirizzo e-mail a scelta |
| Usi aggiuntivi per SV | nessuno nel profilo standard; nel profilo con autenticazione, autenticazione client, accesso con smart card ed EFS | con `Strict` non ripudio, con `Multipurpose` anche cifratura dei dati e autenticazione client |

DigiCert descrive esplicitamente il prodotto SV secondo il modello del `Enterprise RA`: l'organizzazione conferma che la persona nel certificato appartiene ai suoi collaboratori o rappresentanti e conserva le relative prove. DigiCert stessa verifica solo l'organizzazione. Chi è autorizzato a ordinare nell'account CertCentral dell'azienda può quindi inserire qualsiasi nome su un indirizzo dei propri domini. Per OV e SV, DigiCert richiede sempre la prova del controllo del dominio. La verifica dell'organizzazione vale 825 giorni, quella del dominio 398 giorni. Si può scegliere una durata di uno o due anni oppure una data di scadenza personalizzata fino a un massimo di 824 giorni. DigiCert ha ritirato i precedenti prodotti Premium, Email Security Plus, Digital Signature Plus e Class 1 S/MIME il 10 luglio 2025, sostituendoli con questi prodotti.

SwissSign rilascia certificati Gold e OV solo tramite la Managed PKI dell'azienda. La persona deve comunque identificarsi con un documento d'identità ufficiale in originale, di persona o con una procedura equivalente secondo ETSI EN 319 411-1. L'identificazione è svolta dall'ufficio di registrazione che tratta la richiesta oppure da un partner di identificazione formato e sotto contratto: copia, scansione o fotografia del documento, confronto della foto con la persona e conferma della procedura a SwissSign. L'uso della ragione sociale deve essere autorizzato dalla direzione iscritta nel registro di commercio o dal consiglio di amministrazione; anche le persone che richiedono certificati nella Managed PKI vengono identificate con un documento. Un'altra procedura, ad esempio basata su dati del personale, può essere utilizzata da un'azienda solo se SwissSign l'ha verificata e approvata come equivalente. Presso SwissSign, dati e prove sono validi al massimo 398 giorni, dopodiché SwissSign effettua nuovamente la verifica.

Con SwissSign, il certificato OV contiene nel `CN` il nome dell'organizzazione come nel campo `O`. L'articolo [S/MIME nelle banche: quadro giuridico in Svizzera e scelta tra Silver e Gold](/blog/smime-silver-gold-schweiz) descrive quale livello dovrebbe scegliere una banca per quale casella di posta e cosa ciò significhi sul piano giuridico.

## Verificare il tipo di certificato

In Outlook e in altri programmi di posta è possibile aprire i dettagli del certificato di un'e-mail firmata tramite l'icona della firma. In Windows, i campi si trovano nella scheda «Dettagli», alla voce «Soggetto». Per l'OID 2.5.4.97, Windows non conosce un nome visualizzato e lo mostra come `OID.2.5.4.97`.

### Visualizzare il titolare con PowerShell

Per un certificato esportato, ad esempio dai dettagli del certificato di un'e-mail firmata:

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

Per un certificato MV, l'output contiene solo `CN=` con l'indirizzo e-mail.

### Determinare tipo e generazione tramite l'OID della policy

Ogni certificato secondo i BR contiene nell'estensione `Certificate Policies` un OID secondo il modello `2.23.140.1.5.<Typ>.<Generation>`:

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
| `'2.5.29.32'` | OID dell'estensione `Certificate Policies` |
| `Format($true)` | restituisce il contenuto dell'estensione su più righe |

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
| `Cert:\CurrentUser\My` | archivio certificati personale dell'utente connesso |
| `EnhancedKeyUsageList.ObjectId` | usi avanzati della chiave del certificato |
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
| `x509` | sottocomando per i certificati |
| `-in zertifikat.pem` | file di input in formato PEM; per un file DER aggiungere `-inform der` |
| `-noout` | sopprime l'output del certificato codificato |
| `-subject` | restituisce il titolare |
| `-nameopt multiline` | un campo per riga con i nomi dei campi per esteso |

</details>

OpenSSL 3 conosce il nome del campo e lo visualizza per esteso:

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

Come destinatari di un'e-mail firmata, potete verificare la voce organizzativa in quattro passaggi:

1. Il programma di posta segnala che la firma è valida e il certificato proviene da una CA di cui il programma di posta si fida.
2. L'OID della policy inizia con `2.23.140.1.5.2` oppure `2.23.140.1.5.3`, e il certificato contiene i campi `O` e `organizationIdentifier`.
3. L'UID nell'`organizationIdentifier` appartiene, nel registro UID, all'azienda attesa e l'azienda vi risulta attiva.
4. Il dominio dell'indirizzo del mittente appartiene a questa azienda.

## Distinzione rispetto a ZertES

La voce organizzativa non trasforma un certificato S/MIME in un certificato regolamentato ai sensi di ZertES. I certificati regolamentati sono rilasciati soltanto da prestatrici riconosciute ai sensi di ZertES e il certificato deve contenere l'indicazione che si tratta di un certificato regolamentato (art. 2 lett. g e art. 7 cpv. 2 lett. b ZertES).

Esistono parallelismi sul piano del contenuto. Un certificato regolamentato per un'unità UID deve contenere il relativo UID (art. 7 cpv. 2 lett. e ZertES), come lo contiene l'`organizationIdentifier` per OV e SV. Un certificato per una persona fisica può inoltre contenere l'indicazione che essa è autorizzata a rappresentare una determinata unità UID (art. 7 cpv. 3 lett. b ZertES); ciò corrisponde nella struttura a un certificato SV.

| | S/MIME con voce organizzativa | certificato regolamentato ai sensi di ZertES |
|---|---|---|
| Base | S/MIME Baseline Requirements del CA/Browser Forum | ZertES e disposizioni di esecuzione |
| Emittente | CA pubblicamente attendibile | prestatore riconosciuto ai sensi di ZertES |
| Identificativo dell'organizzazione | `organizationIdentifier`, per le aziende svizzere con l'UID | UID, obbligatorio per le unità UID |
| Verifica della persona | per SV, secondo la CA, con documento d'identità o mediante conferma dell'organizzazione come `Enterprise RA` | in linea di principio, comparizione personale con prova dell'identità; per le unità UID, un rappresentante con prova del potere di rappresentanza (art. 9 ZertES) |
| Chiave | secondo le prescrizioni della CA, anche su un gateway | requisiti ai sensi dell'art. 6 ZertES |

Ai sensi dell'art. 14 cpv. 2bis CO, soltanto la firma elettronica qualificata con marca temporale qualificata è equiparata alla firma autografa, e un certificato qualificato può essere rilasciato solo a una persona fisica (art. 8 cpv. 1 ZertES). Per la firma di un'organizzazione, ZertES prevede il sigillo elettronico regolamentato: una firma elettronica avanzata creata con un dispositivo sicuro di creazione del sigillo e basata su un certificato regolamentato di un'unità UID (art. 2 lett. d ZertES). Un'e-mail firmata S/MIME è un mezzo di prova il cui valore è liberamente apprezzato dal giudice (art. 157 CPC); una voce organizzativa verificata rafforza l'attribuzione all'azienda mittente.

**Nota UE:** nell'UE, il regolamento eIDAS disciplina firme e sigilli elettronici. Anche ai sensi dell'art. 25 cpv. 2 eIDAS, solo la firma elettronica qualificata è equiparata alla firma autografa.

## Raccomandazione

- **Caselle funzionali:** certificato OV (SwissSign Organization Validated, DigiCert Secure Email for Organization). Indica l'azienda senza nominare una persona.
- **Caselle personali con firma verso clienti e partner:** certificato SV (SwissSign Gold, DigiCert Secure Email for Employee). Indica persona e azienda.
- **Solo cifratura:** dal punto di vista crittografico, tutti i tipi sono equivalenti. Per la sola cifratura è sufficiente un certificato MV.
- **Pianificare le verifiche:** la verifica dell'organizzazione vale 825 giorni, quella del dominio 398 giorni. Dopo una modifica del nome o della sede dell'azienda, le informazioni nel certificato non corrispondono più al registro; pianificate quindi nuovi certificati.
- **Verificare in ricezione:** l'OID della policy e l'UID mostrano se una firma è attribuita a un'azienda verificata. La qualità della verifica della persona dipende, per SV, dalla procedura della CA: per SwissSign vi è una verifica del documento d'identità, per DigiCert la conferma dell'azienda.

## Fonti

1.  [CA/Browser Forum: S/MIME Baseline Requirements](https://cabforum.org/working-groups/smime/requirements/): versione 1.0.17 del 20 novembre 2026; tipi di certificato e OID delle policy (sezione 1.1), `Enterprise RA` (1.3.2), verifica dell'organizzazione (3.2.3), verifica della persona (3.2.4), verifica del potere di rappresentanza (3.2.6), riutilizzo dei risultati delle verifiche (4.2.1), durate (6.3.2), campi Subject (7.1.4.2) e schemi di registrazione (appendice A).

2.  [cabforum/smime su GitHub: SBR.md](https://github.com/cabforum/smime/blob/main/SBR.md): versione Markdown dei S/MIME Baseline Requirements con cronologia delle versioni, inclusa tra l'altro l'EUID dalla versione 1.0.9.

3.  [SwissSign: EUID, How Your Digital Certificates Become Even More Trustworthy](https://www.swisssign.com/en/blog/european-unique-identifier-euid): denominazioni dei prodotti Silver e Gold, esempio `NTRCH-CHE-109.357.012`, UID come numero di registro ed EUID per le aziende tedesche (28.04.2025).

4.  [SwissSign CP/CPS S/MIME](https://repository.swisssign.com/SwissSign_CPS_SMIME.pdf): quadro normativo combinato di SwissSign, versione 1.0 del 31.08.2026, sostituisce tra l'altro la CPR S/MIME; identificazione della persona (3.2.3), potere di rappresentanza (3.2.5), validità delle prove (3.3.1), modalità di acquisto (4.1.1) e profili di certificato per MV, SV e OV (7.1).

5.  [DigiCert: Secure Email products](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates.html): panoramica dei prodotti con assegnazione a MV, OV e SV, fine della generazione `Legacy` e dei precedenti prodotti S/MIME il 10 luglio 2025.

6.  [DigiCert: Order your Secure Email for Employee certificate](https://docs.digicert.com/en/certcentral/order-and-manage-certificates/secure-email-certificates/order-your-secure-email-for-employee-certificate.html): verifica dell'organizzazione e del dominio, conferma dei collaboratori da parte dell'organizzazione senza verifica autonoma della persona da parte di DigiCert, profili `Strict` e `Multipurpose`, durate.

7.  [Legge federale sulla firma elettronica (ZertES)](https://www.fedlex.admin.ch/eli/cc/2016/752/de): art. 2 (termini, sigillo regolamentato), art. 6 (chiavi), art. 7 (contenuto dei certificati regolamentati, UID), art. 8 (certificati qualificati) e art. 9 (verifica dell'identità).

8.  [Codice delle obbligazioni (CO)](https://www.fedlex.admin.ch/eli/cc/27/317_321_377/de): art. 14 cpv. 2bis sull'equiparazione della firma elettronica qualificata alla firma autografa.

9.  [Registro UID](https://www.uid.admin.ch): consultazione di un UID con nome, sede e stato, gestito dall'Ufficio federale di statistica.

10.  [Zefix](https://www.zefix.ch): indice centrale delle aziende dei registri cantonali di commercio.

11.  [Codice di procedura civile (CPC)](https://www.fedlex.admin.ch/eli/cc/2010/262/de): art. 157 sulla libera valutazione delle prove.

12.  [Regolamento (UE) n. 910/2014 (eIDAS)](https://eur-lex.europa.eu/legal-content/DE/TXT/?uri=CELEX:32014R0910): art. 25 cpv. 2 sull'equiparazione della firma elettronica qualificata alla firma autografa.
