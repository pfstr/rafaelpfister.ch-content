---
title: "Accesso RDP non riuscito: la password deve essere modificata prima del primo accesso"
navTitle: "RDP: modificare la password"
description: "Un nuovo account con password iniziale e la connessione Desktop remoto si interrompe con «You must change your password before logging on the first time». La causa è Network Level Authentication, che non prevede la modifica della password. La password può comunque essere modificata: tramite un'altra sessione Windows, dal proprio PC, con PowerShell o tramite RD Web Access."
date: "2026-11-11"
kategorie: "Client Windows"
timeToRead: "6 min di lettura"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "accesso-rdp-non-riuscito-la-password-deve-essere-modificata-prima-del-primo-accesso"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/it/blog/accesso-rdp-non-riuscito-la-password-deve-essere-modificata-prima-del-primo-accesso
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:32:14.014Z
translationReview: automatic
---

# Accesso RDP non riuscito: la password deve essere modificata prima del primo accesso

In un ambiente cliente con un account appena configurato, mi sono imbattuto di nuovo in questa situazione: la password iniziale è corretta, ma la connessione Desktop remoto viene comunque interrotta subito. Il client Windows `mstsc` visualizza il seguente messaggio:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Altri client RDP e gestori di connessioni indicano la stessa condizione con formulazioni proprie, ad esempio «The user password must be changed before logging on for the first time». In caso di password scaduta, il messaggio è «This user account's password has expired. The password must change in order to logon». La causa e la soluzione sono le stesse in entrambi i casi.

In breve: in questa situazione non è possibile modificare la password tramite la connessione RDP. Occorre modificarla prima in un altro modo e, nella maggior parte dei casi, è sufficiente la connessione di rete a un Domain Controller.

## Causa: NLA autentica prima dell'avvio della sessione

Nell'account è impostata in Active Directory l'opzione **L'utente deve modificare la password al successivo accesso** (tecnicamente: `pwdLastSet` = 0). Su una postazione di lavoro nel dominio non è un problema: la schermata di accesso riconosce la condizione e richiede subito la nuova password.

Con RDP questo passaggio manca non appena è attiva la Network Level Authentication (NLA). NLA autentica l'utente tramite CredSSP prima che sul sistema di destinazione venga creata una sessione con schermata di accesso. L'autenticazione con una password da modificare non riesce e in quel punto non è previsto alcun dialogo per la modifica della password. Il client interrompe la connessione e visualizza soltanto il messaggio. NLA è l'impostazione predefinita nelle versioni attuali di Windows e viene imposta tramite Criteri di gruppo negli ambienti protetti.

## Opzioni per l'utente

Esistono quattro modi per modificare la password senza utilizzare la connessione RDP interessata. La scelta dipende da ciò che è raggiungibile dalla propria postazione di lavoro.

| Opzione | Prerequisito |
|---|---|
| 1. Portale password o service desk | fornito dal gestore |
| 2. Sessione Windows esistente nel dominio | accesso con un altro account o a un altro sistema |
| 3. Finestra di dialogo della password sul proprio PC | Domain Controller raggiungibile, ad esempio tramite VPN |
| 4. PowerShell con `NetUserChangePassword` | TCP 445 raggiungibile verso il Domain Controller |

### Opzione 1: portale password o service desk

Negli ambienti più grandi, per i collaboratori esterni esiste solitamente una procedura prevista: un portale self-service, una funzione password nel portale VPN o Citrix oppure un sistema PAM. Se non è disponibile nulla di tutto ciò, il service desk può rimuovere il flag o impostare una password senza obbligo di modifica (vedere la sezione per gli amministratori).

### Opzione 2: sessione Windows esistente nel dominio

Se esiste già una sessione su un sistema del dominio, ad esempio su un jumphost o con un secondo account, è possibile modificare la password da lì. In una sessione RDP, `Ctrl+Alt+End` apre la schermata di sicurezza del sistema remoto (`Ctrl+Alt+Del` verrebbe elaborato sul PC locale). In **Cambia password**, il campo contenente il nome utente è modificabile: inserire qui l'account interessato nel formato `DOMAIN\benutzer`, quindi la password iniziale e due volte la nuova password.

### Opzione 3: finestra di dialogo della password sul proprio PC

La stessa finestra di dialogo funziona anche su un PC che non è membro del dominio, purché possa raggiungere un Domain Controller, ad esempio tramite una connessione VPN. Premere `Ctrl+Alt+Del`, scegliere **Cambia password** e sostituire il nome utente locale precompilato con `DOMAIN\benutzer`. Il PC deve poter risolvere il nome del dominio tramite DNS; senza i server DNS dell'ambiente di destinazione, Windows non trova alcun Domain Controller.

### Opzione 4: PowerShell con NetUserChangePassword

L'API Windows `NetUserChangePassword` modifica una password direttamente su un Domain Controller. Durante la modifica verifica la password precedente e non richiede un accesso preliminare. Per questo funziona anche quando la password è scaduta o deve essere modificata, nonché da un PC esterno al dominio. Lo script integra la funzione tramite P/Invoke e richiede entrambe le password in modo nascosto:

```powershell
$api = Add-Type -Namespace Win32 -Name NetApi32 -PassThru -MemberDefinition @'
[DllImport("netapi32.dll", CharSet = CharSet.Unicode)]
public static extern uint NetUserChangePassword(
    string domainname, string username,
    string oldpassword, string newpassword);
'@

$dc   = 'dc01.example.com'
$user = 'm.muster'
$alt  = Read-Host -AsSecureString 'Bisheriges Kennwort'
$neu  = Read-Host -AsSecureString 'Neues Kennwort'

$klartext = { param($s) [System.Net.NetworkCredential]::new('', $s).Password }
$rc = $api::NetUserChangePassword(
    $dc, $user, (& $klartext $alt), (& $klartext $neu))

if ($rc -eq 0) { 'Kennwort geändert' } else { "Fehler $rc" }
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Argomento | Effetto |
|---|---|
| `domainname` | Nome DNS o NetBIOS di un Domain Controller o del dominio. Da un PC esterno al dominio, indicare direttamente il Domain Controller (nome o indirizzo IP). |
| `username` | Nome di accesso dell'account (`sAMAccountName`), senza la parte relativa al dominio. |
| `oldpassword` | Password precedente, qui la password iniziale. Deve essere esatta. |
| `newpassword` | Nuova password. Deve soddisfare i criteri password del dominio. |

</details>

Il valore di ritorno 0 indica che l'operazione è riuscita. I codici di errore più comuni:

| Codice | Significato |
|---|---|
| 5 | Accesso negato, ad esempio perché il Domain Controller non consente la chiamata senza autenticazione |
| 86 | La password precedente non è corretta |
| 2221 | Il nome utente non è stato trovato |
| 2245 | La nuova password non soddisfa i criteri password (lunghezza, complessità, cronologia, età minima) |
| 2351 | Il nome del server specificato non è valido |

La chiamata avviene tramite SMB e richiede la porta TCP 445 verso il Domain Controller. Nelle reti fortemente segmentate, questa porta è spesso bloccata dalle reti client o dalla VPN; in tal caso rimane l'opzione 1.

`Set-ADAccountPassword` del modulo Active Directory non aiuta l'utente interessato in questa situazione: il cmdlet richiede una connessione autenticata ai Active Directory Web Services e proprio questo accesso non è possibile con l'account interessato. È lo strumento per l'amministratore.

## Disattivare NLA nel client: solo senza NLA imposta

Spesso viene consigliato di disattivare CredSSP nel file .rdp:

```text
enablecredsspsupport:i:0
```

Il client si connette quindi senza NLA e riceve la classica schermata di accesso del sistema di destinazione, dalla quale è possibile modificare la password. Ciò riesce solo se il sistema di destinazione accetta connessioni senza NLA. Se è attiva l'opzione **Consenti connessioni solo da computer che eseguono Desktop remoto con Autenticazione a livello di rete**, la connessione viene rifiutata. Nelle versioni attuali di Windows Server questa è l'impostazione predefinita.

## Opzioni per gli amministratori

Dal lato amministrativo è possibile evitare o risolvere il problema:

- **Rimuovere il flag:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` elimina l'obbligo di modifica. L'utente può accedere con la password iniziale e modificarla nella sessione tramite `Ctrl+Alt+End`. La password iniziale rimane valida fino a quel momento e deve quindi essere trasmessa tramite un canale sicuro.
- **Reimpostare la password:** `Set-ADAccountPassword -Identity m.muster -Reset` imposta una nuova password. Verificare poi se il flag è ancora impostato.
- **RD Web Access:** se il ruolo è presente, può offrire la modifica della password per le password scadute. In Gestione IIS, in **Sites → Default Web Site → RDWeb → Pages**, impostare nelle **Application Settings** il valore `PasswordChangeEnabled` su `true`. La pagina per la modifica della password si trova in `/RDWeb/Pages/en-US/password.aspx`.
- **Account per accesso esclusivamente RDP:** per gli utenti che lavorano esclusivamente tramite RDP (fornitori esterni, amministratori su jumphost), l'obbligo di modifica al primo accesso non è soddisfacibile senza una delle procedure indicate sopra. La procedura per la prima modifica della password dovrebbe pertanto far parte dell'onboarding.

Disattivare NLA sul sistema di destinazione risolve anch'esso il sintomo, ma rimuove un livello di protezione: senza NLA, ogni client nella rete raggiunge la schermata di accesso e quindi il servizio RDP prima di essersi autenticato. Microsoft lo sconsiglia.

## Fonti

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): elenco dei codici di disconnessione del client RDP, inclusi 4615 (la password deve essere modificata prima del primo accesso) e 3847 (password scaduta), con il testo dei messaggi.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): riferimento API con parametri, autorizzazioni e valori di ritorno.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): resoconto pratico di Evotec sulla modifica della password tramite NetUserChangePassword da un PC esterno al dominio, incluso il riferimento a TCP 445 e alla voce enablecredsspsupport inefficace quando NLA è imposta.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): riferimento alle proprietà .rdp, inclusa enablecredsspsupport con valori e impostazione predefinita.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): sezione sulla modifica delle password scadute tramite RD Web Access e l'impostazione PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): riferimento al cmdlet con i parametri Reset, OldPassword, NewPassword, Credential e Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): indicazioni Microsoft sugli errori di accesso con NLA; la disattivazione di NLA è indicata come non consigliata.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): articolo di blog sul campo utente modificabile nella finestra di dialogo «Cambia password», che consente di modificare la password di un account in un altro dominio.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): guida di Windows OS Hub sull'impostazione PasswordChangeEnabled e sul percorso della pagina password.aspx.
