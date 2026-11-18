---
title: "KB5131085 su Windows Server 2019: Send-MailMessage non riesce dopo la firma NTLM"
navTitle: "KB5131085 invio email"
description: "Dopo l’aggiornamento di novembre KB5131085, su Windows Server 2019 l’invio di email tramite Send-MailMessage a server Postfix non riesce. La causa è considerata la nuova firma NTLM introdotta da KB5135557. L’articolo spiega perché un client SMTP utilizza NTLM, come verificare il caso e quali workaround sono disponibili: forzare AUTH LOGIN, disattivare NTLM sul server di posta o ripristinare temporaneamente il comportamento precedente tramite il Registro di sistema."
date: "2026-11-17"
kategorie: "Windows e SMTP"
timeToRead: "7 min di lettura"
themen:
  - windows-client
  - smtp-mailflow
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-su-windows-server-2019-send-mailmessage-non-riesce-dopo-la-firma-ntlm"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/it/blog/kb5131085-su-windows-server-2019-send-mailmessage-non-riesce-dopo-la-firma-ntlm
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:19:29.761Z
translationReview: automatic
---

# KB5131085 su Windows Server 2019: Send-MailMessage non riesce dopo la firma NTLM

Dopo il Patch Tuesday del 10 novembre 2026, gli amministratori segnalano che gli script su Windows Server 2019 non inviano più email. Sono interessati report di backup, notifiche di monitoraggio e attività simili, che da anni inviano a un server Postfix con il cmdlet PowerShell `Send-MailMessage`. Dopo la disinstallazione dell’aggiornamento cumulativo KB5131085 (OS Build 17763.9487), l’invio torna a funzionare. Günter Born ha descritto il caso il 17 novembre 2026 sul blog Borns IT- und Windows-Blog, in seguito alla segnalazione di un lettore.

Microsoft non elenca finora il problema come Known Issue. L’unica voce Known Issue relativa a KB5131085 riguarda i comandi di attivazione con l’errore `0x80041008`. La causa non è quindi confermata ufficialmente, ma può essere dedotta dalle modifiche dell’aggiornamento e dal comportamento della classe .NET `SmtpClient`.

## Cosa modifica KB5131085 in NTLM

L’aggiornamento include misure di protezione contro gli attacchi NTLM relay, che Microsoft descrive in KB5135557 (“NTLM signing security improvements in Windows”). Si applicano a Windows 11, Windows 10 Enterprise LTSC 2021 e Windows Server dal 2016 al 2025. Nel changelog di KB5131085 è indicato che i client ora offrono per impostazione predefinita NTLM Message Signing per tutte le connessioni non anonime.

Tecnicamente, ciò significa che un client Windows ora imposta nella negoziazione NTLM il flag `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) e il necessario `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`), anche quando l’applicazione non ha richiesto l’integrità. In precedenza, tali applicazioni ricevevano soltanto `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, un segnaposto senza protezione crittografica. Microsoft distribuisce la modifica in due fasi:

| Fase | Data | Comportamento |
|---|---|---|
| 1 | 10 novembre 2026 | I client offrono la firma su tutte le connessioni. I server accettano client firmati e non firmati e registrano quelli non firmati. |
| 2 | Maggio 2027 | I server Windows richiedono per impostazione predefinita la firma e rifiutano i client senza firma con `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

Secondo Microsoft, un client Windows torna senza errori al comportamento precedente se il server non conferma la firma. Tutti i server Windows lo fanno. Le controparti con un’implementazione NTLM propria, ad esempio un server di posta Linux, non sono considerate in questa affermazione.

## Perché un client SMTP utilizza NTLM

`Send-MailMessage` in Windows PowerShell 5.1 utilizza internamente la classe .NET `System.Net.Mail.SmtpClient`. Questa verifica la riga `AUTH` nella risposta EHLO del server e prova i metodi in un ordine fisso: prima `GSSAPI` (Negotiate), poi `NTLM`, quindi `WDigest`, infine `LOGIN`. Se il server offre NTLM, `SmtpClient` usa NTLM, anche quando nome utente e password vengono passati tramite `-Credential`.

Postfix stesso non implementa l’autenticazione SMTP, ma la delega a Cyrus SASL o Dovecot. Entrambi possono offrire NTLM: Cyrus SASL tramite un proprio plugin, Dovecot fino alla versione 2.3 tramite `ntlm` in `auth_mechanisms` (Dovecot 2.4 non include più NTLM). Se NTLM è attivo, Postfix annuncia ad esempio `AUTH PLAIN LOGIN NTLM`, e ogni client Windows si autentica tramite NTLM senza che nessuno se ne accorga. Con KB5131085, il client offre inoltre firma e scambio di chiavi durante questa negoziazione. Se l’implementazione NTLM del server di posta non è in grado di gestirli, l’accesso non riesce e `Send-MailMessage` si interrompe. Un commentatore dell’articolo di Born indica la firma NTLM come causa, e il riscontro è coerente: il workaround del lettore funziona perché esclude NTLM.

## Verificare il caso

Per prima cosa, verificate se il server di posta offre NTLM. Da un computer Linux, la risposta EHLO dopo STARTTLS mostra i metodi offerti:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-starttls smtp` | Stabilisce la connessione in chiaro e passa a TLS tramite STARTTLS |
| `-crlf` | Invia le terminazioni di riga come CRLF, come richiesto da SMTP |
| `-quiet` | Sopprime i dettagli del certificato e della sessione |
| `-connect mail.example.com:587` | Host di destinazione e porta di submission |

</details>

Dopo aver stabilito la connessione, inserite `EHLO test`. Se la riga `250-AUTH` contiene la voce `NTLM`, il server è interessato. Sul server Postfix stesso, due comandi mostrano quale backend SASL è attivo e quali metodi offre Dovecot:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

Sul server Windows, il log NTLM registra le connessioni con i flag negoziati a partire da Windows Server 2019. L’evento 4022 è la voce del client:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Legge il log NTLM operativo |
| `-MaxEvents 50` | Limita l’output alle 50 voci più recenti |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtra gli eventi di audit per client (4022) e server (4023) |
| `Format-List TimeCreated, Id, Message` | Mostra data e ora, ID evento e testo con i Negotiated Flags |

</details>

Una voce al momento del tentativo di invio con il server di posta come destinazione conferma che `SmtpClient` ha utilizzato NTLM.

## Workaround 1: forzare AUTH LOGIN nello script

`Send-MailMessage` non offre un parametro per definire il metodo di accesso. La classe `SmtpClient`, richiamata direttamente, può farlo: un `CredentialCache` fornisce le credenziali solo per uno specifico metodo. Se è configurato soltanto `Login`, `SmtpClient` salta NTLM e usa `AUTH LOGIN`. Lo script seguente si basa sull’esempio del lettore nell’articolo di Born:

```powershell
$server = 'mail.example.com'
$port   = 587
$cred   = Import-Clixml -Path 'C:\Skripte\smtp.cred'

$cache = New-Object System.Net.CredentialCache
$cache.Add($server, $port, 'Login', $cred.GetNetworkCredential())

$smtp = New-Object System.Net.Mail.SmtpClient($server, $port)
$smtp.EnableSsl   = $true
$smtp.Credentials = $cache

$msg = New-Object System.Net.Mail.MailMessage
$msg.From = 'backup@example.com'
$msg.To.Add('admin@example.com')
$msg.Subject      = 'Backup-Report'
$msg.Body         = 'Sicherung abgeschlossen.'
$msg.BodyEncoding = [System.Text.Encoding]::UTF8

try {
    $smtp.Send($msg)
}
finally {
    $msg.Dispose()
    $smtp.Dispose()
}
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Import-Clixml -Path …` | Carica un PSCredential precedentemente salvato con `Get-Credential \| Export-Clixml`; vincolato tramite DPAPI all’utente e al computer |
| `$cache.Add($server, $port, 'Login', …)` | Memorizza le credenziali solo per host, porta e metodo `Login` |
| `SmtpClient($server, $port)` | Crea il client SMTP per il server e la porta di submission |
| `EnableSsl = $true` | Passa a TLS tramite STARTTLS prima di trasmettere le credenziali |
| `Credentials = $cache` | Passa il CredentialCache anziché una semplice NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Codifica il testo in UTF-8 affinché gli umlaut arrivino correttamente |

</details>

`EnableSsl` è obbligatorio qui: `AUTH LOGIN` trasmette nome utente e password solo codificati in Base64, quindi senza TLS in chiaro. Il workaround richiede un intervento per ogni script, ma non modifica la configurazione del sistema e lascia intatta la protezione NTLM dell’aggiornamento.

## Workaround 2: disattivare NTLM sul server di posta

Se gestite direttamente il server Postfix, potete risolvere il problema centralmente: rimuovete `ntlm` da `auth_mechanisms` nella configurazione Dovecot (di norma `/etc/dovecot/conf.d/10-auth.conf`), in modo che rimanga solo `plain login`, quindi ricaricate Dovecot:

```bash
sudo systemctl reload dovecot
```

Con Cyrus SASL, `mech_list: plain login` nel file `smtpd.conf` della directory di configurazione SASL limita di conseguenza i metodi. In seguito, Postfix non annuncia più NTLM e `Send-MailMessage` torna a `LOGIN` senza modifiche allo script. Verificate prima se altri client dipendono da NTLM, ad esempio vecchi profili Outlook con l’opzione “Autenticazione password protetta (SPA)”. Per la submission tramite TLS, NTLM non offre alcun vantaggio di sicurezza rispetto a `PLAIN` e `LOGIN`.

## Workaround 3: Registro di sistema, solo temporaneamente

KB5135557 documenta un valore del Registro di sistema con cui il client non offre più la firma:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Chiave del pacchetto di autenticazione NTLM |
| `/v ClientAllowUnsignedNtlm` | Nome del valore; `0` (predefinito) offre la firma, `1` ripristina il comportamento precedente |
| `/t REG_DWORD` | Tipo di dati del valore |
| `/d 1` | Imposta il valore su 1 |
| `/f` | Sovrascrive senza richiesta di conferma |

</details>

Secondo Microsoft, non è necessario riavviare. Tuttavia, il valore si applica a tutte le connessioni NTLM del server, non solo all’invio di email, e rimuove quindi la protezione relay dell’aggiornamento per l’intero sistema. Microsoft lo descrive come una misura esclusivamente di compatibilità e si riserva di rimuovere in futuro i valori del Registro di sistema. Colma al massimo il periodo fino all’implementazione del workaround 1 o 2. Disinstallare KB5131085 non è un’alternativa, poiché mancherebbero tutte le correzioni di sicurezza del Patch Tuesday di novembre.

## A medio termine: sostituire Send-MailMessage

Microsoft indica `Send-MailMessage` come obsoleto da anni e ne sconsiglia l’uso, poiché il cmdlet non garantisce connessioni sicure. Anche per `SmtpClient`, la documentazione .NET raccomanda altre librerie come MailKit per i nuovi sviluppi. Per gli script che inviano email tramite Exchange Online, l’API Microsoft Graph con `Send-MgUserMail` e la registrazione dell’app è la soluzione supportata; la configurazione è descritta nell’articolo [Accedere a Exchange Online con PowerShell: sostituire EWS con Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). Al più tardi con la fase 2 nel maggio 2027, conviene fare un inventario degli script e dei servizi che usano ancora NTLM: gli eventi di audit 4022 e 4023 forniscono l’elenco.

## Fonti

1.  [Windows Server 2019: KB5131085 interferisce con l’invio di email tramite PowerShell (Borns IT- und Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): segnalazione di un lettore del 17 novembre 2026 con sintomo, test di disinstallazione, script CredentialCache e riferimento alla firma NTLM nei commenti.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): changelog dell’aggiornamento di novembre per Windows Server 2019 e Windows 10 Enterprise LTSC 2019 con la voce NTLM e il Known Issue relativo all’errore 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): flag, pianificazione delle due fasi, eventi di audit 4022/4023 e valori del Registro di sistema ClientAllowUnsignedNtlm e ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): ordine di registrazione dei moduli di autenticazione SMTP Negotiate, NTLM, Digest e Login in SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): indicazione che il cmdlet è obsoleto e non garantisce connessioni sicure.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): raccomandazione di utilizzare MailKit o altre librerie per i nuovi sviluppi.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): autenticazione SMTP in Postfix tramite Cyrus SASL o Dovecot, inclusa la limitazione dei metodi offerti.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): elenco attuale dei metodi SASL in Dovecot, che non include più NTLM.
