---
title: "Rescuezilla: migrare Windows su un nuovo SSD, con presentazione dello strumento"
navTitle: "Migrazione con Rescuezilla"
description: "Rescuezilla è una soluzione di imaging gratuita e compatibile con Clonezilla, dotata di interfaccia grafica. L’articolo presenta lo strumento e mostra la migrazione completa di un’installazione Windows su un nuovo SSD: preparazione in Windows, chiavetta di avvio, backup, verifica, ripristino e operazioni finali."
date: "2026-09-29"
kategorie: "PC e hardware"
timeToRead: "10 min di lettura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "rescuezilla-migrare-windows-su-un-nuovo-ssd-con-presentazione-dello-strumento"
translationId: "article-d9438c5774d0167c"
translationOf: rescuezilla-windows-migration
url: https://rafaelpfister.ch/it/blog/rescuezilla-migrare-windows-su-un-nuovo-ssd-con-presentazione-dello-strumento
translationSourceHash: 33320c51c0a77d69bc5804e2c7e69d14c63731a23df6ef55969b37b043a49cd5
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:50:49.043Z
translationReview: automatic
---

Chi desidera trasferire un’installazione Windows su un nuovo SSD o su un nuovo computer senza reinstallare Windows ha bisogno di uno strumento che esegua il backup e il ripristino dell’intero disco con tutte le partizioni. Rescuezilla è uno di questi strumenti: gratuito, open source e dotato di un’interfaccia grafica utilizzabile anche senza conoscenze di Linux. Questo articolo presenta lo strumento e descrive la migrazione su un SSD di pari dimensioni o più grande. Per un SSD di destinazione più piccolo è disponibile una guida separata: [Migrare Windows su un SSD più piccolo con Rescuezilla](/blog/windows-kleinere-ssd-rescuezilla).

## Che cos’è Rescuezilla

Rescuezilla è un sistema live basato su Ubuntu che si avvia da una chiavetta USB. Funziona indipendentemente dal sistema operativo installato e può quindi eseguire il backup anche di volumi che Windows blocca durante il funzionamento. Il progetto è nato nel 2019 come fork di Redo Backup and Recovery, che a quel momento non veniva più mantenuto da sette anni. Dalla versione 2.0 (2020), Rescuezilla scrive immagini nel formato di Clonezilla: un backup creato con Rescuezilla può essere ripristinato con Clonezilla e viceversa. La licenza è GPL-3.0. La versione attuale è la 2.6.2 di maggio 2026, basata su Ubuntu 26.04 LTS con Partclone 0.3.47.

Il lavoro effettivo viene svolto da Partclone. Supporta i comuni file system (NTFS, FAT, ext4 e altri) e legge soltanto i blocchi occupati. Una partizione da 1 TB con 350 GB di dati genera quindi un’immagine di circa 215 GB (compressa con gzip), non di 1 TB. Le partizioni senza un file system riconosciuto, come la partizione riservata Microsoft, vengono salvate da Rescuezilla blocco per blocco con `dd`; nell’immagine questi file hanno l’estensione `.dd-ptcl-img`.

<details class="options-details">
<summary>Panoramica delle funzioni</summary>

| Funzione | Scopo |
|---|---|
| Backup | salva le partizioni selezionate di un disco, inclusa la tabella delle partizioni, come immagine su un’unità locale o una condivisione di rete (SMB, SSH) |
| Restore | ripristina un’immagine su un disco, con la possibilità di sovrascrivere la tabella delle partizioni |
| Verify Image | verifica se un’immagine esistente è completa e leggibile |
| Clone | copia direttamente un disco su un secondo, senza archiviazione intermedia |
| Image Explorer (beta) | monta un’immagine in sola lettura per recuperare singoli file |
| Immagini di VM | legge, oltre alle immagini Clonezilla, anche immagini VDI, VMDK, VHDX, QCOW2 e Raw |
| Strumenti aggiuntivi | GParted, gestore file, browser web e strumenti per recuperare file eliminati sul desktop live |
| CLI | riga di comando sperimentale (dalla versione 2.5) per Backup, Verify, Restore e Clone |

</details>

Una limitazione è importante per le migrazioni: Rescuezilla non ridimensiona le partizioni. Il disco di destinazione deve arrivare almeno fino alla fine dell’ultima partizione salvata. Se è più piccolo, è necessario prepararlo con GParted, come descritto nella guida collegata sopra.

## Immagine o clonazione

Rescuezilla offre due metodi per una migrazione. Con la **clonazione**, il disco di origine e quello di destinazione sono collegati contemporaneamente e Rescuezilla copia direttamente. Questo consente di risparmiare tempo e un terzo supporto dati, ma richiede che entrambi i dischi siano collegati contemporaneamente al computer o a un adattatore. Con l’**imaging**, viene prima creata un’immagine su un’unità esterna, che viene poi ripristinata sul nuovo disco. Richiede più tempo, ma ha un vantaggio: l’immagine rimane come backup completo dello stato precedente, anche se qualcosa dovesse andare storto durante il ripristino. Per una migrazione, l’immagine è quindi la scelta più sicura; è questa la variante descritta nell’articolo.

## Requisiti

1.  **Chiavetta USB** per Rescuezilla. Il suo contenuto verrà eliminato durante la scrittura dell’immagine di avvio.

2.  **Unità esterna** per l’immagine, con spazio libero pari approssimativamente alle dimensioni dei dati occupati. Sia exFAT sia NTFS funzionano.

3.  **Disco di destinazione**, almeno grande quanto il disco di origine. Se è più piccolo, seguire prima la guida per SSD più piccoli.

4.  **Chiave di ripristino BitLocker**, se C: è crittografato. Disponibile su aka.ms/myrecoverykey o nel portale Entra per i dispositivi gestiti.

## Passaggio 1: preparare Windows

Prima del backup, controllare tre punti in una PowerShell con diritti di amministratore.

**BitLocker.** Partclone non può leggere un volume BitLocker come NTFS. Rescuezilla lo salva quindi blocco per blocco, come qualsiasi partizione senza file system riconosciuto, e l’immagine risulterà grande quanto l’intera partizione. Per una migrazione, si consiglia di disattivare prima BitLocker e di riattivarlo dopo il trasferimento:

```powershell
manage-bde -status C:
manage-bde -off C:
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-status` | mostra il livello di crittografia e lo stato di protezione del volume |
| `-off` | decrittografa completamente il volume; continua a essere eseguito in background |
| `C:` | argomento posizionale: il volume interessato |

</details>

Attendere finché `manage-bde -status C:` non mostra il valore `Fully Decrypted`. Sui dispositivi gestiti tramite Intune, un criterio può riattivare BitLocker; controllare quindi nuovamente lo stato subito prima del riavvio.

**Ibernazione e avvio rapido.** Con l’avvio rapido attivo, Windows non si spegne completamente e NTFS risulta ancora in uso. In questo caso Partclone si interrompe. Un comando disattiva entrambi:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/h off` | forma breve di `/hibernate off`: disattiva l’ibernazione e l’avvio rapido, `hiberfil.sys` viene rimosso |

</details>

**Stato del file system.** Se il dirty bit è impostato, Partclone si interrompe con il messaggio che il volume è “scheduled for a check or it was shutdown uncleanly”. Verificare prima:

```powershell
fsutil dirty query C:
```

Se il comando segnala `is Dirty`, pianificare un controllo per il successivo avvio con `chkdsk C: /f`, riavviare e verificare nuovamente.

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `dirty query C:` | fsutil: verifica se il dirty bit del volume è impostato |
| `/f` | chkdsk: corregge gli errori; sul volume di sistema, il controllo viene pianificato per il successivo avvio |

</details>

## Passaggio 2: creare e avviare la chiavetta di avvio

Scaricare l’immagine ISO dalla pagina delle release del progetto su GitHub. La variante standard riporta il nome in codice di Ubuntu nel nome del file; per la versione 2.6.2 è `rescuezilla-2.6.2-64bit.resolute.iso`. Scriverla sulla chiavetta USB con un programma di scrittura immagini come balenaEtcher; la pagina del progetto raccomanda questo programma per Windows, macOS e Linux.

Windows stesso consente di avviare dalla chiavetta:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/r` | riavvia invece di spegnere |
| `/o` | avvia nelle opzioni di avvio avanzate; solo insieme a `/r` |
| `/t 0` | tempo di attesa in secondi prima dell’esecuzione |

</details>

Nelle opzioni di avvio avanzate, selezionare “Usa un dispositivo” e la chiavetta USB. In alternativa, all’accensione si apre il menu di avvio del firmware (a seconda del produttore F8, F11 o F12). Rescuezilla si avvia con Secure Boot attivato; non è necessario disattivarlo. Se lo schermo resta nero dopo la selezione della lingua, usare la voce “Graphical Fallback Mode” nel menu di avvio della chiavetta.

## Passaggio 3: creare il backup

Rescuezilla si avvia automaticamente sul desktop. La procedura guidata conduce attraverso il backup in passaggi numerati:

1.  Selezionare **Backup**.

2.  *Step 1: Select Drive To Backup*: selezionare il disco di sistema. Modello e dimensioni aiutano a distinguerli; i nomi dei dispositivi (`nvme0n1`, `sda`) possono cambiare tra due avvii.

3.  *Step 2: Select Partitions to Save*: lasciare selezionate tutte le partizioni. Per un ripristino avviabile servono la partizione di sistema EFI, la partizione riservata Microsoft, C: e la partizione di ripristino.

4.  *Step 3: Select Destination Drive*: l’unità esterna (Local) o una condivisione di rete (Network).

5.  *Step 4: Select Destination Folder*: la cartella di destinazione sull’unità. Rescuezilla vi crea una sottocartella con data e ora.

6.  *Step 5: Name Your Backup*: facoltativamente, inserire una descrizione; verrà visualizzata successivamente nell’elenco delle immagini.

7.  *Step 6: Customize Compression Settings*: l’impostazione predefinita gzip è adatta nella maggior parte dei casi.

8.  *Step 7: Confirm Backup Configuration*: controllare le informazioni e avviare.

Per 350 GB di dati su un SSD NVMe, il backup su un SSD USB esterno ha richiesto circa 20 minuti. Al termine, Rescuezilla indica per ogni partizione se il backup è stato eseguito correttamente. Se per una partizione viene visualizzato un errore, l’immagine è incompleta, anche se la cartella esiste.

## Passaggio 4: verificare l’immagine

Prima del ripristino, verificare l’immagine. **Verify Image** nel menu principale legge tutte le parti e controlla che siano complete e leggibili. Nell’elenco delle immagini, la colonna **Partitions** mostra le partizioni salvate con le relative dimensioni; un triangolo di avviso giallo contrassegna le immagini con partizioni mancanti.

La cartella dell’immagine può essere esaminata anche in Windows. I file più importanti:

| File | Contenuto |
|---|---|
| `nvme0n1-pt.parted` | tabella delle partizioni in formato testo, con inizio e fine di ogni partizione in settori |
| `nvme0n1-gpt-1st`, `nvme0n1-gpt-2nd` | copie binarie della GPT primaria e di backup |
| `nvme0n1p3.ntfs-ptcl-img.gz.aa`, `.ab`, … | immagine Partclone di C:, compressa con gzip e suddivisa in parti da 4 GB |
| `nvme0n1p2.dd-ptcl-img.gz.aa` | copia blocco per blocco di una partizione senza file system riconosciuto |
| `clonezilla-img` | registro del backup con il risultato per ogni partizione |
| `Info-*.txt` | dati hardware e SMART del sistema di origine al momento del backup |

Da `nvme0n1-pt.parted` si può verificare se l’immagine entra nel disco di destinazione: il settore finale dell’ultima partizione più uno, moltiplicato per 512 byte, deve essere inferiore alle dimensioni del disco di destinazione in byte.

## Passaggio 5: ripristino sul nuovo SSD

Installare il nuovo SSD e avviare nuovamente Rescuezilla.

1.  Selezionare **Restore**.

2.  *Step 1: Select Image Location*: l’unità esterna con l’immagine.

3.  *Step 2: Select Backup Image*: selezionare l’immagine in base alla data e alle dimensioni delle partizioni.

4.  *Step 3: Select Drive To Restore*: il nuovo SSD. Controllare due volte modello e dimensioni: tutti i dati su questa unità verranno sovrascritti.

5.  *Step 4: Select Partitions to Restore*: lasciare selezionate tutte le partizioni e anche **Overwrite partition table**.

6.  *Step 5: Confirm Restore Configuration*: controllare e avviare.

Il ripristino richiede all’incirca lo stesso tempo del backup. Al termine, spegnere il computer e rimuovere la chiavetta.

## Passaggio 6: avviare dal nuovo disco

La soluzione più semplice è scollegare il vecchio disco prima del primo avvio. Se entrambi i dischi sono collegati, hanno gli stessi ID di partizione e volume e il firmware potrebbe scegliere il vecchio per l’avvio. Se il vecchio disco deve continuare a essere utilizzato, formattarlo solo dopo aver verificato il nuovo sistema.

Se il nuovo SSD è più grande di quello vecchio, lo spazio aggiuntivo si trova come area non allocata alla fine. Tuttavia, non è possibile espandere direttamente C: in Gestione disco, perché la partizione di ripristino si trova tra C: e lo spazio libero. Con GParted sulla chiavetta Rescuezilla, spostare prima la partizione di ripristino alla fine (Resize/Move, **Free space following** su 0), quindi espandere C: sull’intera area libera. Windows RE ritroverà poi la sua partizione poiché mantiene il suo numero di partizione; `reagentc /info` mostra lo stato.

Se con la migrazione cambia anche il computer, Windows in genere si avvia senza modifiche e installa successivamente i driver mancanti. L’attivazione avviene tramite la licenza digitale; in caso di sostituzione della scheda madre, eventualmente solo dopo aver effettuato l’accesso con l’account Microsoft tramite lo strumento di risoluzione dei problemi di attivazione.

## Operazioni finali

Sul nuovo disco, riattivare ciò che è stato disattivato per la migrazione. L’ibernazione viene attivata con `powercfg /h on`. BitLocker si attiva tramite Impostazioni → Privacy e sicurezza → Crittografia dispositivo oppure con `manage-bde -on C:`; quindi controllare con `manage-bde -protectors -get C:` che sia presente e salvata una chiave di ripristino.

Conservare l’immagine sull’unità esterna finché Windows non ha funzionato per alcuni giorni senza anomalie sul nuovo disco e tutti i dati non sono stati controllati. Successivamente può essere eliminata o archiviata come backup dello stato di consegna.

## Fonti

1.  [Rescuezilla](https://rescuezilla.com/): pagina del progetto con panoramica delle funzioni, download e FAQ.

2.  [Rescuezilla su GitHub](https://github.com/rescuezilla/rescuezilla): codice sorgente, storia del progetto ed elenco delle funzioni nel README.

3.  [Release di Rescuezilla](https://github.com/rescuezilla/rescuezilla/releases/latest): versione attuale con note di rilascio, elenco di compatibilità e varianti ISO.

4.  [Changelog di Rescuezilla](https://raw.githubusercontent.com/rescuezilla/rescuezilla/master/CHANGELOG.md): introduzione di CLI, Verify Image e Clone, nonché aggiornamento dello shim Secure Boot.

5.  [Wiki di Rescuezilla: Restoring to a smaller disk](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procedura ufficiale per dischi di destinazione più piccoli dell’originale.

6.  [Partclone](https://partclone.org/): strumento utilizzato da Rescuezilla e Clonezilla per backup consapevoli del file system.

7.  [Manuale di GParted](https://gparted.org/display-doc.php?name=help-manual): spostamento e ridimensionamento delle partizioni.

8.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): verifica dello stato di BitLocker.

9.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): decrittografia di un volume.

10.  [Microsoft Learn: Powercfg command-line options](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): opzione `/hibernate` e relativo effetto sull’avvio rapido.

11.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): verifica del dirty bit.

12.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): controllo e riparazione dei volumi NTFS.

13.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parametro `/o` per l’avvio nelle opzioni di avvio avanzate.

14.  [Microsoft Support: Reactivating Windows after a hardware change](https://support.microsoft.com/en-us/windows/reactivating-windows-after-a-hardware-change-2c0e962a-f04c-145b-6ead-fb3fc72b6665): attivazione tramite licenza digitale dopo la sostituzione della scheda madre.
