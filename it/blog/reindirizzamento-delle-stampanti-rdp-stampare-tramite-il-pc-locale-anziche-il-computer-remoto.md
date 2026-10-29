---
title: "Reindirizzamento stampanti RDP: stampare tramite il PC locale anziché il computer remoto"
navTitle: "Reindirizzamento stampanti RDP"
description: "I lavori di stampa dalla sessione RDP devono arrivare alla stampante accanto all’utente, non a quella del computer remoto. L’impostazione si trova in tre punti: nel client RDP, nel file .rdp e nel sistema di destinazione. Include anche la gestione dell’avviso «Editore sconosciuto» e una checklist per la risoluzione dei problemi."
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
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:33:05.655Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/reindirizzamento-delle-stampanti-rdp-stampare-tramite-il-pc-locale-anziche-il-computer-remoto
---

# Reindirizzamento stampanti RDP: stampare tramite il PC locale anziché il computer remoto

Un utente lavora tramite Remote Desktop su un computer remoto e desidera stampare sulla stampante che si trova accanto a lui. È proprio a questo che serve il reindirizzamento delle stampanti: il client RDP registra le stampanti locali nella sessione, il lavoro di stampa ritorna al client tramite il canale RDP e viene stampato lì. Sul sistema di destinazione, la stampante compare con l’aggiunta **(reindirizzata, sessione n)**. Di norma non sono necessari driver sul computer remoto: Windows utilizza il driver universale **Remote Desktop Easy Print**; il driver della stampante appropriato deve essere installato solo sul client locale.

Il reindirizzamento viene applicato solo al momento della connessione. Dopo ogni modifica alle impostazioni, la sessione deve essere disconnessa completamente e riconnessa; non basta semplicemente ridurre a icona la finestra RDP.

## Lato client: attivare il reindirizzamento

Il modo più semplice per attivare il reindirizzamento delle stampanti è tramite l’interfaccia grafica: avviare `mstsc`, selezionare **Mostra opzioni**, la scheda **Risorse locali**, spuntare **Stampanti** e salvare la connessione nella scheda **Generale**. Chi lavora con file .rdp può modificare direttamente la riga nel file; i file .rdp sono semplici file di testo e possono essere modificati con qualsiasi editor:

```text
redirectprinters:i:1
```

Una particolarità delle scorciatoie senza file .rdp: se la connessione viene avviata con `mstsc /v:hostname`, valgono le impostazioni del file nascosto `Default.rdp` nella cartella Documenti dell’utente. Se manca la riga `redirectprinters:i:1`, la stampante non compare, anche se apparentemente tutto è configurato correttamente. Questo snippet aggiunge la riga in modo idempotente (`0` esistente diventa `1`, mentre una riga mancante viene aggiunta) e mostra il risultato per il controllo:

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

Altre due fonti di errore sul lato client: innanzitutto, Windows memorizza per ogni computer di destinazione in `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` quali reindirizzamenti l’utente ha autorizzato l’ultima volta nella finestra di dialogo di sicurezza; questa selezione salvata sovrascrive l’impostazione predefinita del file .rdp. L’eliminazione della chiave ripristina lo stato. In secondo luogo, il valore del Registro `DisablePrinterRedirection` (DWORD, valore 1) in `HKLM\Software\Microsoft\Terminal Server Client` disattiva completamente il reindirizzamento delle stampanti sul client; sui dispositivi gestiti vale la pena verificarlo prima di iniziare la ricerca del guasto nella sessione.

## Lato server: consentire il reindirizzamento

Sul sistema di destinazione decide il criterio **Non consentire il reindirizzamento delle stampanti client** (Configurazione computer → Modelli amministrativi → Componenti di Windows → Servizi Desktop remoto → Host sessione Desktop remoto → Reindirizzamento stampanti). Se è impostato su *Abilitato*, non vengono create stampanti client, indipendentemente da ciò che richiede il client. Vale il principio dell’impostazione più restrittiva: se una delle due parti blocca il reindirizzamento, questo non avviene.

Senza Criteri di gruppo, lo stesso meccanismo viene controllato tramite il Registro: `fDisableCpm` in `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = reindirizzamento consentito, 1 = bloccato). Inoltre, sul sistema di destinazione deve essere in esecuzione il servizio **Coda di stampa**; senza spooler non vengono create neppure le stampanti reindirizzate.

Nella stessa categoria GPO si trovano altre due impostazioni utili: **Usa prima il driver di stampa Easy Print di Desktop remoto** (impostazione predefinita e nella maggior parte dei casi la scelta corretta) e **Imposta la stampante predefinita del client come stampante predefinita della sessione**.

## L’avviso «Editore sconosciuto»

All’apertura di un file .rdp non firmato che richiede reindirizzamenti di dispositivi, il client mostra un avviso di sicurezza con caselle di controllo per le singole risorse. Le caselle selezionate o deselezionate valgono solo per l’avvio di quella connessione, ma vengono salvate nella chiave `LocalDevices` menzionata sopra e influenzano così silenziosamente le connessioni future. Chi si chiede perché la casella della stampante continui a mancare nonostante il file .rdp corretto, trova quasi sempre lì la causa.

Dall’aggiornamento di sicurezza dell’aprile 2026 (CVE-2026-26151), Connessione Desktop remoto mostra una nuova finestra di dialogo di sicurezza ogni volta che si apre un file .rdp, prima che venga stabilita la connessione. Tutti i reindirizzamenti richiesti dal file, comprese le stampanti, sono disattivati per impostazione predefinita e devono essere abilitati singolarmente. I file non firmati ricevono inoltre l’avviso «Caution: Unknown remote connection». Secondo Microsoft, le connessioni avviate direttamente con il nome di un computer in Connessione Desktop remoto non sono interessate. Come soluzione temporanea, Microsoft documenta il valore del Registro `RedirectionWarningDialogVersion` (DWORD, valore 1) in `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, che ripristina la finestra di dialogo precedente; il supporto per questa opzione potrebbe essere rimosso con un aggiornamento futuro.

Per gestire l’avviso esistono tre possibilità, in ordine di impegno crescente. Primo: avviare la connessione tramite `mstsc /v:hostname` invece che tramite il file .rdp; senza file non viene effettuata alcuna verifica dell’editore e le impostazioni provengono da `Default.rdp`. In questo caso non compare neppure la finestra di dialogo di sicurezza introdotta nell’aprile 2026. Secondo: approvare preventivamente i reindirizzamenti per il computer di destinazione tramite il Registro; in questo modo la parte relativa alle risorse della precedente finestra di dialogo non viene più visualizzata. Microsoft non documenta se questa approvazione preventiva si applichi anche alla nuova finestra di dialogo di sicurezza introdotta nell’aprile 2026:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Terzo, la soluzione corretta per file .rdp distribuiti in azienda: firmare il file con `rdpsign.exe` e un certificato, quindi memorizzare l’impronta digitale del certificato tramite GPO come editore attendibile. Per singole postazioni il lavoro raramente vale la pena, ma per file di connessione distribuiti centralmente è la soluzione giusta.

## Checklist per la risoluzione dei problemi

Se la stampante non compare nella sessione, verificare nell’ordine seguente:

1. **Riconnessi?** Il reindirizzamento viene applicato solo al momento della connessione, non in una sessione esistente.
2. **File corretto?** Per le scorciatoie, verificare quale file .rdp viene effettivamente aperto; con `mstsc /v:` conta `Default.rdp`.
3. **Selezione salvata?** Controllare o eliminare la chiave `LocalDevices` sul client. Quando l’avvio avviene tramite un file .rdp, verificare inoltre che la stampante sia stata abilitata nella finestra di dialogo di sicurezza; dall’aprile 2026 è disattivata per impostazione predefinita.
4. **Blocco client?** `DisablePrinterRedirection` in `HKLM\Software\Microsoft\Terminal Server Client` non deve essere impostato su 1.
5. **Blocco server?** Verificare il GPO «Non consentire il reindirizzamento delle stampanti client» oppure `fDisableCpm` sul sistema di destinazione.
6. **Spooler?** Il servizio Coda di stampa deve essere in esecuzione sul sistema di destinazione.
7. **Controllo nella sessione:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` elenca le stampanti reindirizzate con il relativo ID sessione.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: dall’aggiornamento di sicurezza dell’aprile 2026 (CVE-2026-26151), una nuova finestra di dialogo mostra tutti i reindirizzamenti, comprese le stampanti, disattivati per impostazione predefinita all’apertura dei file .rdp; aggiunti anche il valore del Registro RedirectionWarningDialogVersion e un ulteriore passaggio di verifica.

## Fonti

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): riferimento a tutte le proprietà .rdp, inclusa redirectprinters con valori e impostazione predefinita.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configurazione GPO e Intune, DisablePrinterRedirection e il test con Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): riferimento ai comandi per firmare file .rdp tramite l’impronta digitale di un certificato.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): nuova finestra di dialogo di sicurezza dall’aggiornamento dell’aprile 2026, reindirizzamenti disattivati per impostazione predefinita e valore del Registro RedirectionWarningDialogVersion.
