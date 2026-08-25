---
title: "Reindirizzamento delle stampanti RDP: stampare tramite il PC locale anziché il computer remoto"
navTitle: "Reindirizzamento delle stampanti RDP"
description: "I lavori di stampa della sessione RDP devono finire sulla stampante accanto all'utente, non sul computer remoto. L'impostazione si trova in tre punti: nel client RDP, nel file .rdp e nel sistema di destinazione. Include la gestione dell'avviso «Editore sconosciuto» e una checklist di troubleshooting."
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
url: https://rafaelpfister.ch/it/blog/reindirizzamento-delle-stampanti-rdp-stampare-tramite-il-pc-locale-anziche-il-computer-remoto
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:14:20.841Z
translationReview: automatic
---

# Reindirizzamento delle stampanti RDP: stampare tramite il PC locale anziché il computer remoto

Un utente lavora tramite Remote Desktop su un computer remoto e desidera stampare sulla stampante accanto a lui. È proprio a questo che serve il reindirizzamento delle stampanti: il client RDP annuncia le stampanti locali nella sessione, il lavoro di stampa torna al client attraverso il canale RDP e viene stampato lì. Nel sistema di destinazione, la stampante appare con l'aggiunta **(reindirizzata, sessione n)**. In genere non sono necessari driver sul computer remoto: Windows utilizza il driver universale **Remote Desktop Easy Print**; il driver di stampa appropriato deve essere installato solo sul client locale.

Il reindirizzamento viene applicato solo al momento della connessione. Dopo ogni modifica alle impostazioni, la sessione deve essere disconnessa completamente e riconnessa; non basta minimizzare la finestra RDP.

## Lato client: attivare il reindirizzamento

Il modo più rapido passa dall'interfaccia grafica: avviare `mstsc`, selezionare **Mostra opzioni**, aprire la scheda **Risorse locali**, selezionare **Stampanti** e salvare la connessione nella scheda **Generale**. Chi invece lavora con file .rdp può inserire direttamente la riga corrispondente nel file; i file .rdp sono semplici file di testo e possono essere modificati con qualsiasi editor:

```text
redirectprinters:i:1
```

Un'insidia con i collegamenti senza file .rdp: se la connessione viene avviata con `mstsc /v:hostname`, si applicano le impostazioni del file nascosto `Default.rdp` nella cartella Documenti dell'utente. Se manca la riga `redirectprinters:i:1`, la stampante non appare anche se apparentemente tutto è configurato correttamente. Questo snippet aggiunge la riga in modo idempotente (`0` esistente diventa `1`, la riga mancante viene aggiunta) e visualizza il risultato per il controllo:

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

Altre due insidie sul lato client: primo, Windows memorizza per ogni computer di destinazione in `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` quali reindirizzamenti l'utente ha consentito per ultimi nella finestra di dialogo di sicurezza; questa selezione salvata sovrascrive l'impostazione del file .rdp. L'eliminazione della chiave ripristina lo stato. Secondo, il valore del Registro `DisablePrinterRedirection` (DWORD, valore 1) in `HKLM\Software\Microsoft\Terminal Server Client` disattiva completamente il reindirizzamento delle stampanti sul client; sui dispositivi gestiti conviene verificarlo prima di iniziare la ricerca dei guasti nella sessione.

## Lato server: consentire il reindirizzamento

Nel sistema di destinazione decide il criterio **Non consentire il reindirizzamento delle stampanti client** (Configurazione computer → Modelli amministrativi → Componenti di Windows → Servizi Desktop remoto → Host sessione Desktop remoto → Reindirizzamento stampanti). Se è impostato su *Abilitato*, non vengono create stampanti client, indipendentemente da ciò che richiede il client. Vale il principio dell'impostazione più restrittiva: se una delle due parti blocca il reindirizzamento, questo non avviene.

Senza Criteri di gruppo, lo stesso meccanismo viene controllato tramite il Registro: `fDisableCpm` in `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = reindirizzamento consentito, 1 = bloccato). Inoltre, nel sistema di destinazione deve essere in esecuzione il servizio **Coda di stampa**; senza spooler non vengono create nemmeno le stampanti reindirizzate.

Nella stessa categoria GPO sono presenti due opzioni utili: **Usa prima il driver di stampa Remote Desktop Easy Print** (predefinito e nella maggior parte dei casi la scelta corretta) e **Imposta la stampante predefinita del client come stampante predefinita della sessione**.

## L'avviso «Editore sconosciuto»

Quando si apre un file .rdp non firmato che richiede reindirizzamenti di dispositivi, il client mostra un avviso di sicurezza con caselle di controllo per le singole risorse. Le selezioni aggiunte o rimosse vengono applicate solo a questo avvio della connessione, ma vengono salvate nella chiave `LocalDevices` sopra menzionata e influenzano quindi silenziosamente le connessioni future. Chi si chiede perché la casella della stampante continui a mancare nonostante un file .rdp corretto trova quasi sempre la causa lì.

Per gestire l'avviso esistono tre strade, in ordine di complessità crescente. Primo: avviare la connessione tramite `mstsc /v:hostname` anziché attraverso il file .rdp; senza file non c'è verifica dell'editore e le impostazioni provengono da `Default.rdp`. Secondo: autorizzare in anticipo i reindirizzamenti per il computer di destinazione tramite il Registro; in questo modo viene omessa la parte relativa alle risorse della finestra di dialogo:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Terzo, il metodo corretto per i file .rdp distribuiti in azienda: firmare il file con `rdpsign.exe` e un certificato e memorizzare l'impronta digitale del certificato tramite GPO come editore attendibile. Per singole postazioni di lavoro lo sforzo raramente vale la pena; per file di connessione distribuiti centralmente è la soluzione giusta.

## Checklist di troubleshooting

Se la stampante non appare nella sessione, controllare nell'ordine seguente:

1. **Riconnesso?** Il reindirizzamento viene applicato solo al momento della connessione, non in una sessione esistente.
2. **File corretto?** Con i collegamenti, verificare quale file .rdp viene effettivamente aperto; con `mstsc /v:` conta `Default.rdp`.
3. **Selezione salvata?** Controllare o eliminare la chiave `LocalDevices` sul client.
4. **Blocco del client?** `DisablePrinterRedirection` in `HKLM\Software\Microsoft\Terminal Server Client` non deve essere impostato su 1.
5. **Blocco del server?** Verificare la GPO «Non consentire il reindirizzamento delle stampanti client» o `fDisableCpm` nel sistema di destinazione.
6. **Spooler?** Il servizio Coda di stampa deve essere in esecuzione nel sistema di destinazione.
7. **Controllo nella sessione:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` elenca le stampanti reindirizzate con il relativo ID di sessione.

## Fonti

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): riferimento di tutte le proprietà .rdp, incluso redirectprinters con valori e impostazione predefinita.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configurazione GPO e Intune, DisablePrinterRedirection e il test con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): riferimento del comando per firmare file .rdp tramite impronta digitale del certificato.
