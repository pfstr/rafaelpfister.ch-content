---
title: "Reindirizzamento della stampante RDP: stampare tramite il PC locale anziché il computer remoto"
navTitle: "Reindirizzamento della stampante RDP"
description: "I lavori di stampa della sessione RDP devono finire sulla stampante accanto all'utente, non sul computer remoto. L'impostazione si trova in tre punti: nel client RDP, nel file .rdp e nel sistema di destinazione. Include anche la gestione dell'avviso «Editore sconosciuto» e una checklist per la risoluzione dei problemi."
date: "2026-08-24"
kategorie: "Client Windows"
timeToRead: "5 min di lettura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "reindirizzamento-delle-stampanti-rdp-stampare-tramite-il-pc-locale-anziche-il-computer-remoto"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
translationSourceHash: 2cb3845d308ebda202c6c33b20cbe791ddfbeeb584341876bdc340e0febf65b5
translationModel: gpt-5.6-terra
translatedAt: 2026-08-30T09:29:31.920Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/reindirizzamento-delle-stampanti-rdp-stampare-tramite-il-pc-locale-anziche-il-computer-remoto
---

# Reindirizzamento della stampante RDP: stampare tramite il PC locale anziché il computer remoto

Un utente lavora su un computer remoto tramite Remote Desktop e desidera stampare sulla stampante che si trova accanto a lui. È proprio a questo che serve il reindirizzamento della stampante: il client RDP registra le stampanti locali nella sessione, il lavoro di stampa torna al client attraverso il canale RDP e viene emesso lì. Nel sistema di destinazione, la stampante compare con l'aggiunta **(reindirizzata, sessione n)**. In genere non sono necessari driver sul computer remoto: Windows utilizza il driver universale **Remote Desktop Easy Print**; il driver della stampante appropriato deve essere installato solo sul client locale.

Il reindirizzamento ha effetto solo al momento della connessione. Dopo ogni modifica alle impostazioni, la sessione deve essere disconnessa completamente e riconnessa; non è sufficiente ridurre a icona la finestra RDP.

## Lato client: attivare il reindirizzamento

Il modo più semplice per attivare il reindirizzamento della stampante è tramite l'interfaccia grafica: avviare `mstsc`, selezionare **Mostra opzioni**, aprire la scheda **Risorse locali**, spuntare **Stampanti** e salvare la connessione nella scheda **Generale**. Chi lavora con file .rdp può modificare direttamente la riga nel file; i file .rdp sono semplici file di testo e possono essere modificati con qualsiasi editor:

```text
redirectprinters:i:1
```

Una particolarità delle scorciatoie senza file .rdp: se la connessione viene avviata con `mstsc /v:hostname`, vengono applicate le impostazioni del file nascosto `Default.rdp` nella cartella Documenti dell'utente. Se manca la riga `redirectprinters:i:1`, la stampante non compare, anche se apparentemente tutto è configurato correttamente. Questo snippet aggiunge la riga in modo idempotente (il valore `0` esistente diventa `1`, mentre una riga mancante viene aggiunta) e visualizza il risultato per verifica:

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

Altre due possibili cause di errore sul lato client: in primo luogo, Windows memorizza per ogni computer di destinazione in `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` quali reindirizzamenti l'utente ha autorizzato per ultimi nella finestra di dialogo di sicurezza; questa selezione memorizzata sovrascrive l'impostazione predefinita del file .rdp. L'eliminazione della chiave ripristina lo stato. In secondo luogo, il valore del Registro `DisablePrinterRedirection` (DWORD, valore 1) in `HKLM\Software\Microsoft\Terminal Server Client` disattiva completamente il reindirizzamento della stampante sul client; sui dispositivi gestiti vale la pena controllarlo prima di iniziare la ricerca dell'errore nella sessione.

## Lato server: consentire il reindirizzamento

Nel sistema di destinazione decide il criterio **Non consentire il reindirizzamento della stampante client** (Configurazione computer → Modelli amministrativi → Componenti di Windows → Servizi Desktop remoto → Host sessione Desktop remoto → Reindirizzamento della stampante). Se è impostato su *Abilitato*, non vengono create stampanti client, indipendentemente da ciò che richiede il client. Vale il principio dell'impostazione più restrittiva: se una delle due parti blocca il reindirizzamento, questo non avviene.

Senza Criteri di gruppo, lo stesso meccanismo viene controllato tramite il Registro: `fDisableCpm` in `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = reindirizzamento consentito, 1 = bloccato). Inoltre, nel sistema di destinazione deve essere in esecuzione il servizio **Coda di stampa**; senza spooler non vengono create neppure le stampanti reindirizzate.

Nella stessa categoria GPO sono presenti altre due impostazioni utili: **Usa prima il driver della stampante Remote Desktop Easy Print** (impostazione predefinita e nella maggior parte dei casi la scelta corretta) e **Imposta la stampante predefinita del client come stampante predefinita della sessione**.

## L'avviso «Editore sconosciuto»

Quando si apre un file .rdp non firmato che richiede reindirizzamenti di dispositivi, il client mostra un avviso di sicurezza con caselle di controllo per le singole risorse. Le caselle selezionate o deselezionate in tale punto valgono solo per questo avvio della connessione, ma vengono memorizzate nella chiave `LocalDevices` menzionata sopra e influenzano così silenziosamente le connessioni future. Chi si chiede perché la casella della stampante continui a mancare nonostante il file .rdp corretto, trova quasi sempre la causa lì.

Per gestire l'avviso esistono tre possibilità, in ordine di complessità crescente. Primo: avviare la connessione tramite `mstsc /v:hostname` anziché dal file .rdp; senza file non avviene alcun controllo dell'editore e le impostazioni provengono da `Default.rdp`. Secondo: autorizzare preventivamente tramite Registro i reindirizzamenti per il computer di destinazione; in questo modo la parte relativa alle risorse della finestra di dialogo non viene visualizzata:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Terzo, la soluzione corretta per i file .rdp distribuiti in azienda: firmare il file con `rdpsign.exe` e un certificato, quindi memorizzare l'impronta digitale del certificato tramite GPO come editore attendibile. Per singole postazioni di lavoro lo sforzo raramente vale la pena, mentre per file di connessione distribuiti centralmente è la soluzione giusta.

## Checklist per la risoluzione dei problemi

Se la stampante non compare nella sessione, verificare in questo ordine:

1. **Riconnessi?** Il reindirizzamento ha effetto solo al momento della connessione, non in una sessione esistente.
2. **File corretto?** Per le scorciatoie, verificare quale file .rdp viene effettivamente aperto; con `mstsc /v:` conta `Default.rdp`.
3. **Selezione memorizzata?** Controllare o eliminare la chiave `LocalDevices` sul client.
4. **Blocco del client?** `DisablePrinterRedirection` in `HKLM\Software\Microsoft\Terminal Server Client` non deve essere impostato su 1.
5. **Blocco del server?** Controllare il criterio GPO «Non consentire il reindirizzamento della stampante client» oppure `fDisableCpm` nel sistema di destinazione.
6. **Spooler?** Il servizio Coda di stampa deve essere in esecuzione nel sistema di destinazione.
7. **Verifica nella sessione:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` elenca le stampanti reindirizzate insieme all'ID della sessione.

## Fonti

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): riferimento a tutte le proprietà .rdp, incluso redirectprinters con valori e impostazione predefinita.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configurazione GPO e Intune, DisablePrinterRedirection e il test con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): riferimento ai comandi per firmare file .rdp tramite l'impronta digitale del certificato.
