---
title: "Prerequisiti per il funzionamento di Remote PowerShell"
navTitle: "Remote PowerShell"
description: "Il remoting di PowerShell raramente fallisce a causa del comando, ma piuttosto dei prerequisiti: servizio WinRM, listener, firewall, autenticazione e particolarità degli account locali. Cosa deve essere configurato sul computer di destinazione e sul client, come verificarlo con Test-WSMan e perché Access denied nella maggior parte dei casi non dipende dalla password."
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
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/it/blog/prerequisiti-per-il-funzionamento-di-remote-powershell
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:46:27.691Z
translationReview: automatic
---

# Prerequisiti per il funzionamento di Remote PowerShell

`Invoke-Command` e `Enter-PSSession` si digitano rapidamente, ma la connessione viene stabilita solo quando i prerequisiti sono soddisfatti da entrambe le parti. Il remoting di PowerShell si basa su WS-Management (WinRM), un servizio di gestione basato su SOAP tramite HTTP. Quando una sessione fallisce, quasi mai è colpa del cmdlet stesso, ma di un servizio mancante, una porta chiusa, una regola firewall o dell'autenticazione. Questo articolo esamina i prerequisiti nell'ordine corretto e mostra come verificare ciascuno di essi.

Prima i termini: il computer di destinazione è quello su cui devono essere eseguiti i comandi; il client è il computer dal quale ci si connette. Per impostazione predefinita, WinRM è in ascolto sulla porta 5985 (HTTP) e, se configurato, sulla porta 5986 (HTTPS). Il traffico HTTP sulla porta 5985 viene crittografato a livello di messaggio non appena l'autenticazione avviene tramite Kerberos o NTLM.

## Panoramica dei cmdlet

Per orientarsi, ecco i cmdlet utilizzati in questo articolo:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Cmdlet | Scopo |
|---|---|
| `Enable-PSRemoting` | Configura WinRM sul lato di destinazione: servizio, listener, regola firewall |
| `Test-WSMan` | Verifica se il servizio WinRM della controparte risponde |
| `Enter-PSSession` | Apre una sessione remota interattiva verso un computer |
| `Invoke-Command` | Esegue un blocco di comandi su uno o più computer |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Aggiunge controparti attendibili per l'autenticazione al di fuori di un dominio |
| `Get-Service WinRM` | Mostra lo stato e il tipo di avvio del servizio WinRM |

</details>

## Lato di destinazione: configurare WinRM

Sul computer di destinazione, un unico comando configura il servizio, il listener e la regola firewall. Eseguirlo in PowerShell con privilegi di amministratore:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Force` | Esegue senza richieste di conferma |
| `-SkipNetworkProfileCheck` | Configura il remoting anche se una connessione di rete è classificata come pubblica |

</details>

`Enable-PSRemoting` avvia il servizio WinRM, imposta il tipo di avvio su automatico, crea un listener HTTP e aggiunge la regola firewall appropriata. Una riserva riguarda il profilo di rete: se una scheda di rete è classificata come pubblica, per impostazione predefinita il comando rifiuta la configurazione. Su server o in reti controllate, `-SkipNetworkProfileCheck` consente comunque di completare la configurazione.

È importante l'ambito della regola firewall. Per i profili di rete pubblici, la regola predefinita limita l'accesso alla sottorete locale. Se ci si connette tramite un'altra rete, ad esempio una VPN, questa limitazione si applica e la connessione fallisce nonostante il servizio sia in esecuzione. Aprire quindi la regola in modo mirato per l'intervallo di indirizzi necessario, non indiscriminatamente per tutti gli indirizzi:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Seleziona le regole WinRM HTTP create da Enable-PSRemoting tramite il modello di nome |
| `-RemoteAddress <Bereich>` | Limita gli indirizzi di origine consentiti all'intervallo indicato (qui un blocco CIDR); `Any` consente qualsiasi indirizzo |

</details>

## Lato client: TrustedHosts e servizio

Sul client deve essere in esecuzione il servizio WinRM, altrimenti già l'impostazione delle configurazioni fallisce. Verificarlo innanzitutto:

```powershell
Get-Service WinRM
```

Se il servizio risulta Stopped, avviarlo con `Start-Service WinRM` (sono necessari privilegi di amministratore). Sui client il tipo di avvio è spesso manuale, quindi dopo un riavvio il servizio è nuovamente arrestato. Se si accede regolarmente da questo computer, impostare il tipo di avvio su automatico.

Il secondo punto riguarda l'autenticazione al di fuori di un dominio. Se ci si connette tramite indirizzo IP o in un gruppo di lavoro, il client non può verificare la controparte tramite Kerberos e ricorre a NTLM. Per motivi di sicurezza, WinRM lo rifiuta finché la controparte non è registrata come attendibile. Aggiungere l'indirizzo di destinazione a TrustedHosts (sono necessari privilegi di amministratore):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Value <Liste>` | Controparti attendibili (IP o nome), più elementi separati da virgole, `*` come carattere jolly |
| `-Force` | Imposta il valore senza richiesta di conferma |
| `-Concatenate` | Aggiunge all'elenco esistente invece di sostituirlo |

</details>

TrustedHosts è un'impostazione del client, non del computer di destinazione, e riguarda la sicurezza del client: le controparti registrate vengono considerate attendibili senza che la loro identità sia verificata crittograficamente. Inserire quindi indirizzi specifici e non il carattere jolly `*`. In un dominio con Kerberos non è necessaria alcuna voce; la soluzione corretta al di fuori di un dominio, senza TrustedHosts, è un listener HTTPS con un certificato considerato attendibile dal client.

## Autenticazione: perché Access denied raramente dipende dalla password

Un errore frequente con gli account locali è il messaggio Access denied, anche se la password è corretta. Il motivo è il filtro UAC remoto: con gli account locali (non l'Administrator integrato), Windows rimuove per impostazione predefinita i privilegi amministrativi per l'accesso tramite rete. L'accesso riesce, ma ogni azione con privilegi elevati viene rifiutata. Se la controparte segnala Access denied anziché credenziali errate, questa è la causa probabile.

È possibile risolvere il problema sul computer di destinazione con un valore di registro che concede agli amministratori locali pieni privilegi tramite rete:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Si tratta di un allentamento consapevole: gli account amministrativi locali ottengono così pieni privilegi tramite rete. Impostare il valore solo in reti controllate e con password robuste. In un dominio è preferibile usare un account di dominio, e il problema non si pone.

Quando si stabilisce la connessione, per gli account locali indicare il nome utente preceduto dal nome del computer, affinché il sistema di destinazione risolva l'account localmente:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

Nella finestra di accesso, inserire l'utente come `RECHNERNAME\Benutzer`; per gli account di dominio come `DOMAENE\Benutzer`. Un PIN dell'accesso a Windows non funziona tramite rete; è necessaria la password dell'account. Per un account Microsoft, si tratta della relativa password e il nome dell'account può differire dal nome visualizzato.

## Verificare nell'ordine corretto

Isolare gli errori dal basso verso l'alto: in questo modo si individua rapidamente quale prerequisito manca.

Per prima cosa, la raggiungibilità della porta:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Se la porta non risponde, manca il listener oppure il firewall blocca la comunicazione. Se risponde, verificare il servizio WinRM della controparte:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Una risposta con versione del protocollo e produttore significa che servizio e listener sono attivi. Solo dopo verificare con le credenziali:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Se questa chiamata restituisce il nome del computer della controparte, tutti i prerequisiti sono soddisfatti.

## Errori tipici e relative cause

| Messaggio o sintomo | Causa probabile | Approccio |
|---|---|---|
| Porta 5985 non raggiungibile | Nessun listener o firewall bloccato | `Enable-PSRemoting`, verificare la regola firewall e il relativo ambito |
| WinRM cannot complete the operation | Servizio sul lato di destinazione disattivato oppure accesso consentito solo dalla sottorete locale | Avviare il servizio, aprire la regola firewall per l'intervallo di indirizzi necessario |
| The WinRM client cannot process the request … TrustedHosts | Connessione non di dominio senza voce TrustedHosts | Inserire l'indirizzo di destinazione in TrustedHosts sul client oppure usare HTTPS |
| Access is denied (nonostante la password corretta) | Filtro UAC remoto con account locale | Impostare `LocalAccountTokenFilterPolicy` su 1 oppure utilizzare un account di dominio |
| L'accesso a una seconda risorsa fallisce nella sessione | Double hop: le credenziali non vengono inoltrate | Eseguire l'attività direttamente sulla destinazione oppure utilizzare CredSSP o un'autenticazione separata |

## Limiti: il problema del double hop

Anche con una configurazione completa rimane una limitazione, aggirabile solo con apposite soluzioni: per impostazione predefinita, una sessione remota non può inoltrare le credenziali a un terzo sistema. Se in una sessione sul computer di destinazione si accede a una condivisione di rete o a un altro server, l'operazione fallisce per mancanza di credenziali. Questo problema di double hop è una caratteristica di sicurezza, non una configurazione errata. Per la maggior parte delle attività di supporto è sufficiente eseguire il comando direttamente sul computer di destinazione. Quando l'inoltro è effettivamente necessario, entrano in gioco CredSSP o una delega vincolata, entrambi con specifiche considerazioni di sicurezza.

## Fonti

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): prerequisiti per il remoting di PowerShell, autorizzazioni e profili di rete.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): cosa configura il comando, inclusa la riserva relativa al profilo di rete e la regola firewall.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autenticazione al di fuori del dominio e messaggi di errore frequenti.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): causa del problema del double hop e approcci risolutivi con i relativi compromessi.
