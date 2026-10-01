---
title: "Migrare Windows su un SSD più piccolo con Rescuezilla: da 1 TB a 512 GB"
navTitle: "Migrazione su SSD più piccolo"
description: "Rescuezilla ripristina un'immagine solo su un disco che arriva almeno fino alla fine dell'ultima partizione. Questa guida mostra, con l'esempio di un NVMe da 1 TB, come ridurre C: con GParted, spostare la partizione di ripristino, rimuovere il dirty flag e ripristinare la nuova immagine su un SSD da 512 GB."
date: "2026-09-29"
kategorie: "PC e hardware"
timeToRead: "9 min di lettura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "backup-dr"
  - "migration"
slug: "migrare-windows-su-un-ssd-piu-piccolo-con-rescuezilla-da-1-tb-a-512-gb"
translationId: "article-6947213279383f27"
translationOf: windows-kleinere-ssd-rescuezilla
url: https://rafaelpfister.ch/it/blog/migrare-windows-su-un-ssd-piu-piccolo-con-rescuezilla-da-1-tb-a-512-gb
translationSourceHash: ad4485736511552671c4e469ddaa672e9439cffb3e74d43bd8dd7333eaf662d3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-30T09:54:53.612Z
translationReview: automatic
---

Rescuezilla è un sistema live gratuito per il backup e il ripristino di interi supporti dati, compatibile con il formato di immagine di Clonezilla (presentazione dello strumento e procedura di base: [Rescuezilla: migrare Windows su un nuovo SSD](/blog/rescuezilla-windows-migration)). Finché il disco di destinazione è uguale o più grande, sono sufficienti backup e ripristino. Se è più piccolo, il ripristino si interrompe anche se i dati occupati vi entrerebbero. Questo articolo documenta la migrazione di un sistema Windows 11 da un NVMe da 1 TB (C: con 930 GB, di cui circa 350 GB occupati) a un NVMe da 512 GB, incluse le schermate di errore che si presentano durante il percorso.

## Perché il ripristino su un disco più piccolo fallisce

Rescuezilla esegue il backup di ogni partizione separatamente con Partclone e salva inoltre la tabella delle partizioni. Durante il ripristino scrive questa tabella invariata sul disco di destinazione. Rescuezilla non può ridurre una partizione. Prima di avviare l'operazione verifica quindi che l'ultima partizione sia interamente sul disco di destinazione e, in caso contrario, si interrompe con questo messaggio:

```text
The source partition table's final partition (/dev/nvme0n1p4:
1000203091968 bytes) must refer to a region completely within
the destination disk (512110190592 bytes).
```

Una tipica installazione di Windows ha quattro partizioni: la partizione di sistema EFI (200 MB), la partizione riservata Microsoft (16 MB), C: e la partizione di ripristino con Windows RE (qui 904 MiB). Windows colloca la partizione di ripristino alla fine del disco. Non basta quindi ridurre C:: anche la partizione di ripristino deve essere spostata in avanti, direttamente dietro C:.

La procedura descritta dal wiki di Rescuezilla per questo caso è: ridurre il disco di origine con GParted, creare una nuova immagine, ripristinare questa immagine. Un'immagine creata in precedenza del disco non modificato viene conservata come backup finché il nuovo sistema non è operativo.

## Passaggio 1: calcolare la dimensione di destinazione

Un SSD venduto come 512 GB ha 512'110'190'592 byte, ovvero 476,9 GiB (Rescuezilla mostra anch'esso questo valore). Da questo valore vanno sottratte EFI, MSR e la partizione di ripristino, per un totale di poco più di 1,1 GiB. Per C: rimangono quindi poco meno di 475 GiB. Con un po' di margine, **470 GiB = 481'280 MiB** è un valore di destinazione sensato. I circa 6 GB che resteranno liberi alla fine sul nuovo SSD non sono rilevanti.

I dati occupati devono rimanere sotto questo valore. Il comando seguente, eseguito in PowerShell con diritti di amministratore, indica di quanto Windows potrebbe ridurre un volume:

```powershell
$s = Get-PartitionSupportedSize -DriveLetter C
"{0:N1} GB" -f ($s.SizeMin / 1GB)
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-DriveLetter C` | Volume di cui vengono richieste le possibili dimensioni |
| `SizeMin` | Dimensione minima alla quale Windows potrebbe ridurre autonomamente il volume |
| `"{0:N1} GB" -f` | Formatta il valore in byte come gigabyte con una cifra decimale |

</details>

Se `SizeMin` è nettamente superiore alla quantità di dati occupati, file non spostabili (file di paging, punti di ripristino, MFT) impediscono la riduzione con gli strumenti integrati di Windows. GParted sposta anche questi dati durante la riduzione, perciò tale limite non si applica.

## Passaggio 2: disattivare BitLocker e l'ibernazione

GParted non può né leggere né ridurre un volume crittografato con BitLocker, e Partclone non può eseguirne il backup come NTFS. Verificate lo stato in PowerShell con diritti di amministratore:

```powershell
manage-bde -status C:
```

Se non compare `Fully Decrypted`, disattivate BitLocker e attendete il completamento della decrittazione:

```powershell
manage-bde -off C:
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-status` | Mostra il livello e il metodo di crittografia, nonché lo stato di protezione del volume |
| `-off` | Decritta completamente il volume e rimuove i dispositivi di protezione delle chiavi |
| `C:` | Argomento posizionale: il volume interessato |

</details>

La decrittazione avviene in background e, a seconda della quantità di dati, può richiedere un'ora o più. Nei dispositivi gestiti tramite Intune, un criterio potrebbe riattivare BitLocker dopo breve tempo. Verificate quindi nuovamente lo stato immediatamente prima di avviare GParted.

Inoltre, l'ibernazione deve essere disattivata. Con l'avvio rapido attivo, Windows non viene arrestato completamente, ma salva lo stato del kernel in `hiberfil.sys`. NTFS risulta quindi ancora in uso e GParted rifiuta qualsiasi modifica. Un comando disattiva insieme l'ibernazione e l'avvio rapido e cancella `hiberfil.sys`:

```powershell
powercfg /h off
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/h off` | Forma abbreviata di `/hibernate off`: disattiva l'ibernazione e l'avvio rapido, `hiberfil.sys` viene rimosso |

</details>

## Passaggio 3: avviare Rescuezilla

Windows può indirizzare il successivo avvio direttamente al menu di avvio con le opzioni avanzate. Qui selezionate «Usa un dispositivo» e la chiavetta USB con Rescuezilla:

```powershell
shutdown /r /o /t 0
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `/r` | Riavvia anziché arrestare il sistema |
| `/o` | Avvia nelle opzioni di avvio avanzate (Windows RE); solo insieme a `/r` |
| `/t 0` | Tempo di attesa in secondi prima dell'esecuzione |

</details>

In alternativa, all'accensione si apre il menu di avvio del firmware, sui sistemi ASRock con F11. Se manca «Usa un dispositivo», `shutdown /r /fw /t 0` porta direttamente alla configurazione UEFI, dove è possibile selezionare l'unità di avvio per il prossimo avvio.

## Passaggio 4: ridurre C: e spostare la partizione di ripristino

Sul desktop di Rescuezilla avviate **Partition Editor** (GParted), non Rescuezilla stesso.

1.  In alto a destra selezionate il disco di origine. Fate attenzione alle dimensioni (qui 931.51 GiB), per non modificare accidentalmente il disco di backup esterno o un secondo disco interno.

2.  Fate clic con il tasto destro sulla partizione NTFS con C: → **Resize/Move**. Nel campo **New size (MiB)** inserite il valore di destinazione, qui `481280`, e confermate con il tasto Tab. **Free space preceding** resta invariato. Confermate con **Resize/Move**.

3.  Sotto C: compare ora una riga `unallocated`, seguita dalla partizione di ripristino (NTFS, circa 900 MiB, flag `hidden, diag`). Fate clic con il tasto destro sulla partizione di ripristino → **Resize/Move**.

4.  Nel campo **Free space preceding (MiB)** eliminate il valore, inserite `0` e confermate con Tab. **New size** rimane invariato, **Free space following** passa all'intera area libera. Se dopo Tab compare `1` anziché `0`, si tratta dell'allineamento a MiB interi ed è normale. Confermate con **Resize/Move**.

5.  Confermate con OK l'avviso che lo spostamento potrebbe impedire l'avvio. Riguarda le partizioni con bootloader; Windows si avvia dalla partizione di sistema EFI, che rimane invariata.

6.  L'ordine è ora: EFI, MSR, C:, partizione di ripristino, `unallocated`. Fino a questo punto le modifiche sono soltanto pianificate. Solo un clic sul segno di spunta verde (**Apply All Operations**) le esegue.

Windows RE ritrova la propria partizione dopo lo spostamento perché mantiene il suo numero di partizione. `reagentc /info` continua quindi a mostrare `Enabled` con il percorso `harddisk0\partition4\Recovery\WindowsRE`.

La guida di Rescuezilla riduce soltanto l'ultima partizione. Questo è sufficiente se C: è l'ultima partizione. In un'installazione standard di Windows 10 o 11, la partizione di ripristino si trova dopo di essa e il suo spostamento è obbligatorio.

## Passaggio 5: rimuovere il dirty flag

Dopo la riduzione, il backup successivo si interrompe su C: con questo messaggio:

```text
ntfsclone-ng.c: NTFS Volume '/dev/nvme0n1p3' is scheduled for a check
or it was shutdown uncleanly. Please boot Windows or fix it by fsck.
```

La causa è intenzionale: `ntfsresize`, usato da GParted per NTFS, contrassegna il file system per il controllo prima di ogni modifica delle dimensioni e lascia tale contrassegno. Secondo la pagina man, Windows dovrebbe eseguire `chkdsk` al successivo avvio. Nel caso descritto, tuttavia, il flag rimaneva impostato anche dopo un normale avvio di Windows e `Get-Volume C` segnalava `Full Repair Needed`. Potete verificare lo stato in PowerShell con diritti di amministratore:

```powershell
fsutil dirty query C:
```

Se il comando segnala `Volume - C: is Dirty`, programmate il controllo al successivo avvio:

```powershell
chkdsk C: /f
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `dirty query C:` | fsutil: verifica se il dirty bit del volume è impostato |
| `C:` | chkdsk: il volume da controllare |
| `/f` | Corregge gli errori trovati; sul volume di sistema il controllo viene programmato per il successivo avvio |

</details>

Rispondete J alla domanda se il controllo debba essere eseguito al prossimo riavvio e riavviate Windows normalmente. Dopodiché `fsutil dirty query C:` deve mostrare il messaggio `is NOT Dirty` e `Get-Volume C` deve segnalare nuovamente `Healthy`. Solo allora avviate di nuovo Rescuezilla.

## Passaggio 6: creare e verificare una nuova immagine

Ora create con Rescuezilla una nuova immagine del disco ridotto. La dimensione dell'immagine rimane praticamente uguale a quella del primo backup (qui circa 214 GB), poiché Partclone salva soltanto i blocchi occupati. Rescuezilla salva ogni backup in una propria cartella con marca temporale, ad esempio `2026-09-29-1343-img-rescuezilla`.

Nell'elenco per la selezione delle immagini, tutte le immagini vengono quindi visualizzate una accanto all'altra. La colonna **Partitions** mostra le dimensioni: l'immagine precedente alla riduzione contiene `ntfs 930.4GB`, quella nuova `ntfs 470GB`. Un tentativo di backup interrotto appare con un triangolo di avviso giallo; contiene la vecchia tabella delle partizioni e durante il ripristino causa esattamente il messaggio di errore della prima sezione. Eliminate tali cartelle per non selezionarle accidentalmente.

Potete anche verificare in anticipo se un'immagine entra nel disco di destinazione consultando il file `<disk>-pt.parted` nella cartella dell'immagine. Contiene la tabella delle partizioni salvata in settori da 512 byte:

```text
Number  Start       End         Size        File system  Name
 1      2048s       411647s     409600s     fat32        EFI system partition
 2      411648s     444415s     32768s                   Microsoft reserved partition
 3      444416s     986105855s  985661440s  ntfs         Basic data partition
 4      986105856s  987957247s  1851392s    ntfs
```

La fine dell'ultima partizione (987'957'247 + 1 settori × 512 byte = 505,8 GB) è inferiore ai 512,1 GB del disco di destinazione. L'immagine è adatta.

## Passaggio 7: ripristinare sul nuovo SSD

1.  In Rescuezilla selezionate **Restore**, l'unità con le immagini e poi l'immagine **nuova** (senza triangolo di avviso, con la partizione C: ridotta).

2.  Selezionate il nuovo SSD come destinazione. Verificate due volte modello e dimensioni: il ripristino sovrascrive completamente il disco di destinazione.

3.  Lasciate selezionate tutte le partizioni, così come **Overwrite partition table**, e avviate il ripristino.

4.  Quindi avviate dal nuovo SSD. Il modo più semplice è scollegare prima il vecchio disco; altrimenti selezionate il nuovo SSD nel menu di avvio del firmware.

## Operazioni finali

Sul nuovo disco riattivate ciò che è stato disattivato per la migrazione. L'ibernazione si riattiva con `powercfg /h on`. Potete attivare BitLocker tramite Impostazioni → Privacy e sicurezza → Crittografia dispositivo oppure con `manage-bde -on C:`; verificate poi con `manage-bde -protectors -get C:` che la chiave di ripristino sia stata salvata. Se Secure Boot è stato disattivato nell'UEFI per avviare Rescuezilla, riattivatelo; finché è disattivato, BitLocker registra l'evento 810 a ogni avvio.

Potete eliminare l'immagine del disco da 1 TB non modificato non appena Windows si avvia correttamente dal nuovo SSD e i dati sono completi.

## Fonti

1.  [Rescuezilla Wiki: ripristino su un disco più piccolo](https://github.com/rescuezilla/rescuezilla/wiki/HOWTO:-Restoring-to-a-smaller-disk.-Eg,-1000GB-HDD-to-500GB-SSD): procedura ufficiale con GParted, motivo della limitazione durante il ripristino.

2.  [Rescuezilla](https://rescuezilla.com/): pagina del progetto con il download del sistema live.

3.  [Manuale di GParted](https://gparted.org/display-doc.php?name=help-manual): utilizzo di Resize/Move, allineamento a MiB ed esecuzione delle operazioni pianificate.

4.  [ntfsresize(8), pagina man di Ubuntu](https://manpages.ubuntu.com/manpages/noble/man8/ntfsresize.8.html): strumento alla base della riduzione NTFS in GParted; descrive il contrassegno di controllo impostato intenzionalmente.

5.  [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk): parametro `/f` e pianificazione del controllo sul volume di sistema.

6.  [Microsoft Learn: fsutil dirty](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty): verifica e significato del dirty bit.

7.  [Microsoft Learn: manage-bde status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status): verifica dello stato di BitLocker.

8.  [Microsoft Learn: manage-bde off](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-off): decrittazione di un volume.

9.  [Microsoft Learn: opzioni della riga di comando Powercfg](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options): opzione `/hibernate` e relativo impatto sull'avvio rapido.

10.  [Microsoft Learn: Get-PartitionSupportedSize](https://learn.microsoft.com/en-us/powershell/module/storage/get-partitionsupportedsize): dimensioni minime e massime della partizione dal punto di vista di Windows.

11.  [Microsoft Learn: opzioni della riga di comando REAgentC](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options): verifica dello stato e del percorso di Windows RE.

12.  [Microsoft Learn: shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): parametri `/o` e `/fw` per l'avvio nelle opzioni avanzate o nell'UEFI.
