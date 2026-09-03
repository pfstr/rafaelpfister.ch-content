---
title: "Outlook New: firma S/MIME non verificabile nell'account secondario, allegati mancanti"
navTitle: "S/MIME nell'account secondario"
description: "Il nuovo Outlook segnala che la firma S/MIME non può essere verificata nell'account secondario per una cassetta postale condivisa e non mostra alcun allegato. L'articolo spiega la differenza tra Clear Signing e Opaque Signing, perché gli allegati scompaiono nelle e-mail firmate in modo opaco, perché il nuovo Outlook elabora S/MIME solo nell'account principale e quali soluzioni esistono, incluso l'estrazione di smime.p7m con PowerShell o OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min di lettura"
themen:
  - outlook
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "outlook"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
slug: "outlook-new-firma-s-mime-non-verificabile-nell-account-secondario-allegati-mancanti"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/it/blog/outlook-new-firma-s-mime-non-verificabile-nell-account-secondario-allegati-mancanti
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:11:51.071Z
translationReview: automatic
---

# Outlook New: firma S/MIME non verificabile nell'account secondario, allegati mancanti

Nel nuovo Outlook per Windows, quando si apre un'e-mail firmata digitalmente in una cassetta postale condivisa, compare una barra rossa: "La firma S/MIME non può essere verificata durante la visualizzazione nell'account secondario." L'e-mail viene visualizzata, ma gli allegati no, anche se il mittente li ha inviati. Le colleghe e i colleghi che usano la stessa cassetta postale come account principale vedono gli allegati senza problemi.

Alla base vi sono due fattori che si rafforzano a vicenda: il nuovo Outlook elabora S/MIME solo nell'account principale e il mittente ha firmato l'e-mail in modo opaco. In questa forma di firma, l'intero contenuto, inclusi gli allegati, si trova in un unico contenitore crittografico. Se il client non riesce ad aprire il contenitore, gli allegati rimangono invisibili. Entrambi i problemi possono essere risolti singolarmente.

## Cosa significa il messaggio

Nel nuovo Outlook, "account secondario" indica qualsiasi cassetta postale diversa dall'account con cui è stato effettuato l'accesso. Ciò vale per le cassette postali condivise (Shared Mailboxes) visualizzate tramite accesso completo e automapping, per le cassette postali aggiunte tramite "Aggiungi cassetta postale condivisa" e per le deleghe. L'elaborazione S/MIME nel nuovo Outlook è vincolata all'account principale. Se un messaggio firmato arriva in un altro account, il client non verifica la firma e mostra invece il messaggio.

Questo non dice nulla sulla validità della firma e non indica un problema di certificato presso il mittente. La stessa e-mail può essere verificata e aperta nell'account principale o in Outlook classico.

## Clear Signing e Opaque Signing

Lo standard S/MIME (RFC 8551) prevede due formati per i messaggi firmati. Entrambi forniscono la stessa firma, ma impacchettano il messaggio in modo diverso.

| | Clear Signing | Opaque Signing |
|---|---|---|
| Tipo MIME | `multipart/signed` con `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` con `smime-type=signed-data` |
| Struttura | Due parti: il testo leggibile del messaggio con gli allegati e, accanto, la firma separata | Una parte: testo del messaggio, allegati e firma insieme in un contenitore CMS-SignedData, codificato in Base64 |
| Allegato visualizzato da un client senza S/MIME | `smime.p7s` (solo la firma, pochi KB) | `smime.p7m` (l'intero messaggio) |
| Leggibile senza supporto S/MIME | Sì, testo e allegati vengono visualizzati normalmente | No, il client vede solo il contenitore |
| Sensibilità durante il trasporto | La firma diventa non valida se un server di posta o un gateway modifica interruzioni di riga, codifica o spazi | Il contenitore protegge il contenuto da tali modifiche |
| Sezione RFC 8551 | 3.5.3 | 3.5.2 |

Nel sorgente del messaggio, è possibile riconoscere i due formati dall'intestazione `Content-Type`. Un'e-mail con firma clear inizia così:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

Un'e-mail con firma opaque invece così:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

La differenza spiega completamente il comportamento nel nuovo Outlook. Con un'e-mail firmata clear, il client mostra testo e allegati anche se non verifica la firma; manca solo lo stato della firma. Con un'e-mail firmata opaque, il client deve prima estrarre il contenitore tramite l'elaborazione S/MIME per accedere a testo e allegati. Se lo rifiuta perché il messaggio si trova in un account secondario, il contenitore resta chiuso. Il fatto che il testo sia comunque leggibile è dovuto a Exchange Online: il servizio esegue il rendering lato server della parte testuale, ma non degli allegati nel contenitore.

Nessuno dei due formati cifra alcunché. Anche la variante opaca è solo codificata in Base64 e leggibile da chiunque entri in possesso del messaggio. Microsoft lo sottolinea espressamente nella documentazione di Exchange Online.

## Il formato scelto dal mittente

In Outlook classico, l'opzione "Invia messaggi firmati come testo normale" (File > Opzioni > Centro protezione > Sicurezza posta elettronica) determina il formato. È attivata per impostazione predefinita, quindi Outlook firma in formato clear. Chi disattiva l'opzione invia in formato opaco. Il nuovo Outlook e Outlook sul web non offrono questa impostazione.

I gateway di posta che firmano centralmente dispongono di una propria impostazione per il formato di firma. Alcuni prodotti firmano in formato opaco per impostazione predefinita per ragioni di robustezza, poiché la firma rimane valida anche dopo una modifica da parte di sistemi a valle. Se si ricevono regolarmente e-mail con allegati mancanti da un determinato mittente, vale la pena controllare la configurazione del suo gateway.

## Perché il nuovo Outlook elabora S/MIME solo nell'account principale

Microsoft documenta la limitazione, ma non ne indica un motivo tecnico. Il motivo emerge dall'architettura del client.

Il nuovo Outlook è essenzialmente il client web di Outlook sul web in un involucro nativo. Nel browser, JavaScript non può accedere all'archivio certificati di Windows. Per questo Outlook sul web ha richiesto per anni un'estensione separata del browser per S/MIME. Il nuovo Outlook sostituisce questa estensione con un ponte integrato tra l'interfaccia web e la crittografia di Windows. Questo ponte viene inizializzato all'accesso dell'account principale e riceve la sessione della relativa cassetta postale, i relativi certificati e le relative impostazioni S/MIME da Impostazioni > Posta > S/MIME.

Le cassette postali condivise e gli account secondari utilizzano altri percorsi dati. Gli account secondari hanno sessioni proprie, mentre le cassette postali condivise vengono caricate tramite la delega dell'account principale. Finora il ponte non era collegato a questi percorsi. Ciò vale anche per la semplice verifica di una firma, sebbene a tal fine non sarebbe necessaria alcuna chiave privata: l'analisi e l'estrazione della struttura PKCS#7 avvengono tramite lo stesso componente.

In Outlook classico il problema non si presenta, perché l'elaborazione S/MIME avviene nello stack MAPI per ogni messaggio, indipendentemente dallo store da cui proviene il messaggio.

Microsoft sta aggiungendo il collegamento mancante: la voce della roadmap 565861 estende S/MIME nel nuovo Outlook alle cassette postali condivise e delegate collegate all'account principale. La disponibilità generale è annunciata per luglio 2026, con un rollout graduale. Se il messaggio continua a comparire, la modifica non è ancora arrivata nel tenant o nella versione client in uso. La voce non copre gli account secondari aggiunti separatamente con un proprio accesso.

## Soluzioni

La soluzione adatta dipende da come è integrata la cassetta postale e dalla necessità di verificare la firma o solo di accedere agli allegati.

| Soluzione | Requisito | Risultato |
|---|---|---|
| Aprire l'e-mail nell'account principale | Si possiede personalmente la cassetta postale come account principale oppure l'e-mail è stata inoltrata | Verifica della firma e allegati |
| Aggiungere la cassetta postale come account autonomo nel nuovo Outlook (Impostazioni > Account > Aggiungi account) | La cassetta postale dispone di credenziali proprie; non possibile per cassette postali condivise senza password | Verifica della firma e allegati, non appena si passa a questo account |
| Outlook classico | Ancora installato o raggiungibile tornando indietro tramite l'interruttore "Nuovo Outlook"; integrare lì la cassetta postale come account proprio (File > Impostazioni account > Nuovo) | Verifica della firma e allegati in ogni store |
| Outlook sul web | Aprire direttamente la cassetta postale (`outlook.office.com/mail/<adresse>`), con l'estensione S/MIME per Edge o Chrome installata | Verifica della firma e allegati |
| Chiedere al mittente il Clear Signing | Il mittente utilizza Outlook classico o un gateway con formato selezionabile | Allegati visibili, ma lo stato della firma nell'account secondario continua a non essere disponibile |
| Estrarre manualmente il contenitore | Salvare l'e-mail come `.eml` o conservare `smime.p7m` | Allegati senza verifica della firma |

## Estrarre manualmente il contenitore

Per un caso singolo, è sufficiente aprire il contenitore al di fuori di Outlook. La firma viene verificata matematicamente, ma non la catena di attendibilità del certificato. Salvare il messaggio come `.eml` oppure conservare l'allegato `smime.p7m` in una cartella.

In Windows è sufficiente PowerShell. Il .NET Framework include la classe `SignedCms`, che legge il contenitore PKCS#7:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Istruzione | Effetto |
|---|---|
| `Add-Type -AssemblyName System.Security` | Carica l'assembly con le classi PKCS (necessario in Windows PowerShell 5.1, già caricato in PowerShell 7) |
| `[IO.File]::ReadAllBytes(...)` | Legge il contenitore DER binario; il file `smime.p7m` salvato da Outlook è già decodificato |
| `$cms.Decode($bytes)` | Analizza la struttura CMS-SignedData |
| `$cms.CheckSignature($true)` | Verifica solo la firma sul contenuto (`$true` = verifySignatureOnly); con `$false` verrebbe verificata anche la catena di certificati. In caso di firma non valida, il comando si interrompe con un'eccezione |
| `$cms.ContentInfo.Content` | Il contenuto firmato: un messaggio MIME completo con testo e allegati |
| `[IO.File]::WriteAllBytes(...)` | Scrive questo messaggio MIME come `.eml`, che è possibile aprire in Outlook o Thunderbird |

</details>

Su Linux, macOS o con Git per Windows è disponibile OpenSSL. Se l'intera e-mail è disponibile come `.eml`, OpenSSL esegue anche la decodifica Base64:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `cms` | Strumento CMS di OpenSSL; `smime` funziona in modo equivalente, `cms` è l'interfaccia più recente |
| `-verify` | Verifica la firma e restituisce il contenuto firmato |
| `-noverify` | Salta la verifica della catena di certificati; la firma stessa viene comunque verificata |
| `-in nachricht.eml` | L'intera e-mail in formato S/MIME (Base64 con intestazioni MIME); per un file `smime.p7m` conservato, aggiungere `-inform DER` |
| `-out inhalt.eml` | Il contenuto estratto come messaggio MIME |

</details>

Il file `inhalt.eml` contiene il testo originale del messaggio e tutti gli allegati come normali parti MIME. Un doppio clic lo apre in Outlook, dove è possibile salvare gli allegati come di consueto.

## Fonti

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): Il caso pratico con lo stesso messaggio nella cassetta postale condivisa; la risposta conferma che il comportamento è noto e non indica alcuna soluzione nel nuovo Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Sezioni 3.5.2 (application/pkcs7-mime con SignedData) e 3.5.3 (multipart/signed), con le indicazioni sulla leggibilità senza S/MIME e sulla robustezza durante il trasporto.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): L'opzione "Invia messaggi firmati come testo normale" in Outlook classico, attivata per impostazione predefinita; non presente nel nuovo Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): Impostazioni S/MIME nel nuovo Outlook in Impostazioni > Posta > S/MIME; i certificati devono essere installati manualmente o tramite criterio.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Indicazione che i messaggi firmati in modo opaco sono solo codificati in Base64 e non riservati.

6.  [Microsoft 365 Roadmap, voce 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME per cassette postali condivise e delegate nel nuovo Outlook per Windows, annunciato per luglio 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Quali tipi di account supporta il nuovo Outlook e come vengono integrate le cassette postali condivise.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature e ContentInfo per estrarre il contenitore con PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Opzioni `-verify`, `-noverify`, `-inform` e `-out`.
