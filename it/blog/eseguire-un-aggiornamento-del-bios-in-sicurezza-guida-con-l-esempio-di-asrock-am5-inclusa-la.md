---
title: "Eseguire un aggiornamento del BIOS in sicurezza: guida con l'esempio di ASRock AM5, inclusa la preparazione di BitLocker"
navTitle: "Aggiornamento del BIOS"
description: "La procedura completa per un aggiornamento del BIOS usando come esempio una scheda ASRock AM5: determinare la versione, verificare il download tramite hash, sospendere correttamente BitLocker, avviare l'UEFI (anche se F2 non funziona), aggiornare con Instant Flash e configurare sensatamente le impostazioni dopo l'aggiornamento."
date: "2026-08-26"
kategorie: "PC e hardware"
timeToRead: "8 min di lettura"
themen:
  - pc-hardware
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
  - "releases"
slug: "eseguire-un-aggiornamento-del-bios-in-sicurezza-guida-con-l-esempio-di-asrock-am5-inclusa-la"
translationId: "article-82840b2d159b9367"
translationOf: bios-update-asrock-am5-sicher-durchfuehren
url: https://rafaelpfister.ch/it/blog/eseguire-un-aggiornamento-del-bios-in-sicurezza-guida-con-l-esempio-di-asrock-am5-inclusa-la
translationSourceHash: 60fff28a10b0f91f3d59996b00afe614f2230b9831514dcf01a1f496b99f4fbd
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:26:08.131Z
translationReview: automatic
---

Un aggiornamento del BIOS rientra tra gli interventi di manutenzione che si effettuano raramente e che quindi sollevano ogni volta le stesse domande: qual è la versione corretta, come la si trasferisce in sicurezza sulla scheda e cosa occorre considerare prima e dopo? Questa guida documenta l'intera procedura usando come esempio una ASRock A620I Lightning WiFi (socket AM5) con il metodo proprietario Instant Flash. I passaggi sono applicabili a qualsiasi scheda madre moderna; i punti critici (BitLocker, Fast Boot, ripristino delle impostazioni) sono indipendenti dal produttore.

## Quando è opportuno un aggiornamento del BIOS

Tre situazioni giustificano l'intervento. Primo, le correzioni di sicurezza: le vulnerabilità del firmware possono essere risolte solo tramite un aggiornamento del BIOS e i changelog dei produttori di solito le menzionano solo brevemente. Secondo, la compatibilità: il supporto per nuove generazioni di CPU e una migliore compatibilità della memoria arrivano esclusivamente con nuove versioni del firmware; su AM5, tramite il firmware di riferimento AGESA di AMD, che i produttori di schede integrano nelle loro versioni del BIOS. Terzo, la stabilità: se un sistema si riavvia spontaneamente e il registro eventi riporta soltanto Kernel-Power 41 con `BugcheckCode=0`, l'arresto si è verificato a livello hardware o firmware, senza il coinvolgimento di Windows; cause tipiche sono tensioni instabili e il training della memoria, ed è proprio questo il livello gestito dalle versioni AGESA. Voci come "Improve memory compatibility and system stability" o una gestione EXPO rivista nei changelog indicano che un aggiornamento affronta tali problemi. Se invece un sistema funziona stabilmente e non è interessato dalle vulnerabilità corrette, attendere è legittimo; un aggiornamento del BIOS senza motivo è un rischio senza beneficio.

## Passaggio 1: determinare lo stato attuale

Prima di scaricare qualsiasi cosa, servono due informazioni: il modello esatto della scheda e la versione del BIOS installata. PowerShell fornisce entrambe senza riavviare:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

Annotate la versione. Servirà in seguito per verificare il risultato e, leggendo i changelog, per sapere quali versioni state saltando.

## Passaggio 2: scaricare il BIOS e verificare il checksum

Scaricate il BIOS esclusivamente dalla pagina del prodotto del produttore, mai da portali di terze parti. ASRock pubblica il checksum SHA256 per ogni versione; dopo il download, confrontatelo prima ancora che il file si avvicini a una chiavetta USB:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

Se il valore non corrisponde a quello indicato dal produttore, il download è danneggiato o manomesso: non effettuate il flash. Dopo l'estrazione rimane un singolo file ROM, nell'esempio `A62IRW_4.43.ROM` da 32 MB.

## Passaggio 3: preparare la chiavetta USB

Il meccanismo di flash nell'UEFI ("Instant Flash" per ASRock, Q-Flash, EZ Flash o M-Flash per altri produttori) legge la chiavetta direttamente dal firmware. Questo significa che viene riconosciuto in modo affidabile solo FAT32, non NTFS ed exFAT. Quasi tutte le chiavette acquistate già pronte sono formattate in FAT32; potete verificarlo così:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

Copiate il file ROM nella directory principale della chiavetta. Una nuova formattazione è necessaria solo se il file system non è adatto. La capacità della chiavetta non è rilevante: il file è più piccolo di qualsiasi capacità oggi comune.

Una nota sulla scelta del metodo: molte schede offrono anche un pulsante BIOS Flashback, che esegue il flash senza CPU e senza un sistema funzionante. È la procedura di recupero per una scheda che non si avvia più. Per un sistema funzionante, Instant Flash nell'UEFI è il metodo corretto e più semplice. Gli strumenti di flash basati su Windows non sono né necessari né consigliabili sulle piattaforme attuali.

## Passaggio 4: sospendere BitLocker, altrimenti si rischia la richiesta della chiave

Questo è il punto assente da molte guide. Se l'unità di sistema è crittografata con BitLocker (spesso attivato automaticamente in Windows 11 con un account Microsoft), BitLocker associa la chiave ai valori misurati dal TPM. Un aggiornamento del BIOS modifica tali valori e, al successivo avvio, Windows richiede la chiave di ripristino di 48 cifre. Chi non la ha a portata di mano si ritrova con un sistema inaccessibile.

BitLocker include un meccanismo specifico per questo scenario. In una PowerShell con diritti di amministratore:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

Il parametro `RebootCount 2` copre entrambi i riavvii imminenti (uno nell'UEFI, uno dopo il flash); in seguito la protezione si riattiva automaticamente e sigilla la chiave rispetto ai nuovi valori misurati. Verificate comunque in anticipo che la chiave di ripristino sia reperibile, ad esempio nell'account Microsoft su aka.ms/myrecoverykey oppure tramite `manage-bde -protectors -get C:`.

## Passaggio 5: accedere all'UEFI, anche se F2 non risponde

Il metodo classico tramite F2 o Canc all'accensione spesso fallisce sui sistemi moderni: con Fast Boot attivato, il firmware inizializza la tastiera USB solo dopo il POST e la pressione del tasto non viene rilevata. Tuttavia non dipendete da quel tasto: Windows può indirizzare il riavvio successivo direttamente alla configurazione UEFI. In una PowerShell con diritti di amministratore:

```powershell
shutdown /r /fw /t 5
```

Se il comando segnala l'errore 203 ("The system could not find the environment option that was entered"), quasi sempre mancano i diritti di amministratore: senza elevazione il processo non può impostare la necessaria variabile firmware e il messaggio di errore non indica questa causa. Un secondo percorso possibile, senza variabile firmware, passa attraverso l'ambiente di ripristino: `shutdown /r /o`, quindi Risoluzione dei problemi, Opzioni avanzate, Impostazioni firmware UEFI.

## Passaggio 6: eseguire il flash con Instant Flash

Nell'UEFI trovate Instant Flash nel menu Tool. Lo strumento elenca tutti i file ROM presenti sulla chiavetta; dopo la selezione controlla il file, esegue il flash e riavvia automaticamente. Durante questi pochi minuti vale l'unica regola imprescindibile dell'intera procedura: non interrompete l'alimentazione e non spegnete il computer. Un flash interrotto è l'unico passaggio di questa guida che può effettivamente rendere la scheda impossibile da avviare (rendendo necessario il percorso di recupero Flashback menzionato).

## Passaggio 7: completare la configurazione, poiché l'aggiornamento ripristina tutto

Dopo il flash, tutte le impostazioni del BIOS vengono ripristinate ai valori di fabbrica. È previsto e offre un'opportunità diagnostica: la RAM ora funziona senza profilo EXPO alla velocità base JEDEC. Se avete eseguito il flash per problemi di stabilità, lasciatela intenzionalmente così per una o due settimane. Se gli arresti anomali non si ripresentano, il profilo di memoria era coinvolto e potete testare nuovamente EXPO in modo mirato con il nuovo firmware. La differenza nell'uso quotidiano tra 4800 e 6000 MT/s è appena percepibile al di fuori dei benchmark; un computer stabile vale ogni punto nei benchmark.

In ogni caso, vale la pena visitare due impostazioni nell'UEFI: chi ha avuto riavvii in inattività può impostare, in Advanced, AMD CBS, l'opzione "Power Supply Idle Control" su "Typical Current Idle"; ciò attenua una nota incompatibilità di alcuni alimentatori con gli stati di inattività profondi delle CPU Ryzen. Chi invece vuole tornare ad accedere al setup con F2 può disattivare Fast Boot.

La verifica del risultato una volta tornati in Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

La prima riga deve mostrare la nuova versione, la seconda la frequenza di memoria prevista e BitLocker deve nuovamente indicare "Protezione attivata". Con questo, l'aggiornamento è completato e documentato. Se il flash è stato eseguito per problemi di stabilità, solo l'osservazione nelle settimane successive mostrerà se sono stati risolti, nel modo più semplice controllando la presenza di nuovi eventi Kernel-Power 41 nel registro eventi di sistema.

## Fonti

1.  [ASRock A620I Lightning WiFi, download del BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): elenco delle versioni con changelog, checksum SHA256 e metodi di aggiornamento supportati dalla scheda dell'esempio.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): riferimento per sospendere la protezione BitLocker, incluso il parametro RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): inquadramento di Kernel-Power 41 e del significato di BugcheckCode 0.

4.  [Microsoft Learn: comando shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentazione dei parametri /fw e /o per il riavvio rispettivamente nell'UEFI e nell'ambiente di ripristino.
