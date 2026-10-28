---
title: "Prerequisiti per il funzionamento di Remote PowerShell"
navTitle: "Remote PowerShell"
description: "Il remoting di PowerShell raramente fallisce per colpa del comando, ma piuttosto dei prerequisiti: servizio WinRM, listener, firewall, autenticazione e peculiarità degli account locali. Cosa deve essere configurato sul computer di destinazione e sul client, come verificarlo con Test-WSMan e perché Access denied di solito non dipende dalla password."
date: "2026-09-01"
kategorie: "Windows e PowerShell"
timeToRead: "10 min di lettura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "prerequisiti-per-il-funzionamento-di-remote-powershell"
translationId: "article-7315c1ae9e67a24d"
translationOf: remote-powershell-voraussetzungen
translationSourceHash: edebb6e6be909c38c0d70f1bf09aaac37328f86bf2def4e23a26a34456b3492f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:57:19.948Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/prerequisiti-per-il-funzionamento-di-remote-powershell
---

# Prerequisiti per il funzionamento di Remote PowerShell

`Invoke-Command` e `Enter-PSSession` si digitano rapidamente, ma la connessione viene stabilita solo quando i prerequisiti sono soddisfatti su entrambi i lati. Il remoting di PowerShell si basa su WS-Management (WinRM), un servizio di gestione basato su SOAP tramite HTTP. Quando una sessione fallisce, quasi mai è colpa del cmdlet stesso, bensì di un servizio mancante, una porta chiusa, una regola del firewall o dell'autenticazione. Questo articolo illustra i prerequisiti nell'ordine corretto e mostra come verificarli singolarmente.

Prima i termini: il computer di destinazione è quello sul quale devono essere eseguiti i comandi; il client è il computer dal quale ci si connette. Per impostazione predefinita, WinRM è in ascolto sulla porta 5985 (HTTP) e, se configurato, sulla porta 5986 (HTTPS). Il traffico HTTP sulla porta 5985 è crittografato a livello di messaggio non appena l'autenticazione avviene tramite Kerberos o NTLM.

## Panoramica dei cmdlet

Per orientarsi, ecco i cmdlet presenti in questo articolo:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Cmdlet | Scopo |
|---|---|
| `Enable-PSRemoting` | Configura WinRM sul computer di destinazione: servizio, listener, regola del firewall |
| `Test-WSMan` | Verifica se il servizio WinRM della controparte risponde |
| `Enter-PSSession` | Apre una sessione remota interattiva con un computer |
| `Invoke-Command` | Esegue un blocco di comandi su uno o più computer |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Registra controparti attendibili per l'autenticazione al di fuori di un dominio |
| `Get-Service WinRM` | Mostra stato e tipo di avvio del servizio WinRM |

</details>

## Computer di destinazione: configurare WinRM

Sul computer di destinazione, un singolo comando configura il servizio, il listener e la regola del firewall. Eseguirlo in PowerShell con privilegi di amministratore:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Force` | Esegue senza richiedere conferma |
| `-SkipNetworkProfileCheck` | Configura il remoting anche se una connessione di rete è classificata come pubblica |

</details>

`Enable-PSRemoting` avvia il servizio WinRM, imposta il tipo di avvio su automatico, crea un listener HTTP e aggiunge la relativa regola del firewall. Una riserva riguarda il profilo di rete: se una scheda di rete è classificata come pubblica, per impostazione predefinita il comando rifiuta la configurazione. Su server o in reti controllate, `-SkipNetworkProfileCheck` consente comunque di completarla.

È importante l'ambito della regola del firewall. Per i profili di rete pubblici, la regola predefinita limita l'accesso alla sottorete locale. Se ci si connette tramite un'altra rete, ad esempio una VPN, tale limitazione si applica e la connessione fallisce nonostante il servizio sia in esecuzione. Aprire quindi la regola in modo mirato per l'intervallo di indirizzi necessario, non indiscriminatamente per tutti gli indirizzi:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Seleziona, tramite il modello del nome, le regole WinRM HTTP create da Enable-PSRemoting |
| `-RemoteAddress <Bereich>` | Limita gli indirizzi sorgente consentiti all'intervallo specificato (qui un blocco CIDR); `Any` consente qualsiasi indirizzo |

</details>

## Lato client: TrustedHosts e servizio

Sul client il servizio WinRM deve essere in esecuzione, altrimenti fallisce già l'impostazione delle configurazioni. Verificarlo prima:

```powershell
Get-Service WinRM
```

Se il servizio risulta Stopped, avviarlo con `Start-Service WinRM` (sono necessari privilegi di amministratore). Sui client il tipo di avvio è spesso manuale, quindi il servizio torna a essere arrestato dopo un riavvio. Se si accede regolarmente da questo computer, impostare il tipo di avvio su automatico.

Il secondo punto riguarda l'autenticazione al di fuori di un dominio. Se ci si connette tramite indirizzo IP o in un gruppo di lavoro, il client non può verificare la controparte tramite Kerberos e ricorre a NTLM. Per motivi di sicurezza, WinRM lo rifiuta finché la controparte non è registrata come attendibile. Aggiungere l'indirizzo di destinazione ai TrustedHosts (sono necessari privilegi di amministratore):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Value <Liste>` | Controparti attendibili (IP o nome), più elementi separati da virgole, `*` come carattere jolly |
| `-Force` | Imposta il valore senza richiedere conferma |
| `-Concatenate` | Aggiunge all'elenco esistente invece di sostituirlo |

</details>

TrustedHosts è un'impostazione del client, non del computer di destinazione, e riguarda la sicurezza del client: le controparti registrate sono considerate attendibili senza che la loro identità venga verificata crittograficamente. Inserire quindi indirizzi specifici e non il carattere jolly `*`. In un dominio con Kerberos, la voce non è necessaria; al di fuori di un dominio, la soluzione corretta senza TrustedHosts è un listener HTTPS con un certificato considerato attendibile dal client.

## Autenticazione: perché Access denied raramente dipende dalla password

Un errore frequente con gli account locali è il messaggio Access denied, anche se la password è corretta. Il motivo è il filtro UAC remoto: per impostazione predefinita, Windows rimuove i privilegi amministrativi agli account locali (non all'Administrator integrato) quando l'accesso avviene dalla rete. L'accesso riesce, ma ogni azione che richiede privilegi elevati viene rifiutata. Se la controparte segnala Access denied invece di credenziali errate, questa è la causa probabile.

Sul computer di destinazione è possibile risolvere il problema con un valore del Registro che assegna agli amministratori locali pieni privilegi tramite la rete:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Si tratta di un allentamento deliberato: gli account amministrativi locali ottengono così pieni privilegi tramite la rete. Impostare il valore solo in reti controllate e con password robuste. In un dominio è preferibile usare un account di dominio, e il problema non si pone.

Quando si stabilisce la connessione, per gli account locali indicare il nome utente preceduto dal nome del computer, affinché il sistema di destinazione risolva l'account localmente:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

Nella finestra di accesso, inserire l'utente come `RECHNERNAME\Benutzer` e, per gli account di dominio, come `DOMAENE\Benutzer`. Un PIN dell'accesso a Windows non funziona tramite la rete; è necessaria la password dell'account. Con un account Microsoft è richiesta la relativa password e il nome dell'account può differire dal nome visualizzato.

## Verificare nell'ordine corretto

Isolare gli errori dal basso verso l'alto per individuare rapidamente quale prerequisito manca.

Per prima cosa, verificare la raggiungibilità della porta:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Se la porta non risponde, manca il listener oppure il firewall blocca la connessione. Se risponde, verificare il servizio WinRM della controparte:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Una risposta con versione del protocollo e produttore significa che servizio e listener sono disponibili. Solo dopo testare con le credenziali:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Se questa chiamata restituisce il nome del computer della controparte, tutti i prerequisiti sono soddisfatti.

## Errori tipici e relative cause

| Messaggio o sintomo | Causa probabile | Approccio |
|---|---|---|
| Porta 5985 non raggiungibile | Nessun listener o firewall bloccante | Verificare `Enable-PSRemoting`, la regola del firewall e il relativo ambito |
| WinRM cannot complete the operation | Servizio sul computer di destinazione arrestato, oppure accesso consentito solo dalla sottorete locale | Avviare il servizio, aprire la regola del firewall per l'intervallo di indirizzi necessario |
| The WinRM client cannot process the request … TrustedHosts | Connessione fuori dominio senza voce TrustedHosts | Aggiungere l'indirizzo di destinazione ai TrustedHosts sul client oppure usare HTTPS |
| Access is denied (nonostante la password corretta) | Filtro UAC remoto per account locale | Impostare `LocalAccountTokenFilterPolicy` su 1 oppure usare un account di dominio |
| L'accesso a una seconda risorsa fallisce nella sessione | Double hop: le credenziali non vengono inoltrate | Eseguire l'attività direttamente sulla destinazione oppure usare CredSSP o credenziali delegate |

## Limiti: il problema del double hop

Anche con una configurazione completa resta una limitazione, aggirabile solo con misure specifiche: per impostazione predefinita, una sessione remota non può inoltrare le credenziali a un terzo sistema. Se durante una sessione sul computer di destinazione si accede a una condivisione di rete o a un ulteriore server, l'operazione fallisce per mancanza di credenziali. Questo problema del double hop è una caratteristica di sicurezza, non una configurazione errata. Per la maggior parte delle attività di supporto è sufficiente eseguire il comando direttamente sul computer di destinazione. Quando l'inoltro è realmente necessario, entrano in gioco CredSSP o una delega vincolata, entrambi con specifiche considerazioni di sicurezza.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornato il link alle fonti al nuovo indirizzo di Microsoft Learn.

## Fonti

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): prerequisiti per il remoting di PowerShell, autorizzazioni e profili di rete.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): cosa configura il comando, inclusi l'avvertimento relativo al profilo di rete e la regola del firewall.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autenticazione fuori dal dominio e messaggi di errore frequenti.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop): causa del problema del double hop e soluzioni con le relative considerazioni.
