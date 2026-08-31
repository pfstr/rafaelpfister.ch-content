---
title: "Eseguire un aggiornamento del BIOS in sicurezza: guida con l’esempio di ASRock AM5, inclusa la preparazione di BitLocker"
navTitle: "Aggiornamento del BIOS"
description: "La procedura completa per un aggiornamento del BIOS con l’esempio di una scheda ASRock AM5: determinare la versione, verificare il download tramite hash, sospendere correttamente BitLocker, avviare l’UEFI (anche se F2 non funziona), aggiornare con Instant Flash e impostare opportunamente le configurazioni dopo l’aggiornamento."
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
translationSourceHash: 555b16e753b2ac5dec357741b071ed6aa33de367a2197a8dbb10fef7c9f6a946
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:14:27.244Z
translationReview: automatic
url: https://rafaelpfister.ch/it/blog/eseguire-un-aggiornamento-del-bios-in-sicurezza-guida-con-l-esempio-di-asrock-am5-inclusa-la
---

Un aggiornamento del BIOS rientra tra le operazioni di manutenzione che si effettuano di rado e che quindi sollevano ogni volta nuove domande: qual è la versione giusta, come la si trasferisce in sicurezza sulla scheda e cosa occorre considerare prima e dopo? Questa guida documenta l’intera procedura usando come esempio una ASRock A620I Lightning WiFi (socket AM5) con il metodo proprietario Instant Flash. I passaggi sono applicabili a qualsiasi scheda madre moderna; i punti critici (BitLocker, Fast Boot, ripristino delle impostazioni) sono indipendenti dal produttore.

## Quando è opportuno un aggiornamento del BIOS

Tre motivi giustificano l’intervento. Primo, le correzioni di sicurezza: le vulnerabilità del firmware possono essere chiuse solo mediante un aggiornamento del BIOS, e i changelog dei produttori di solito le descrivono solo brevemente. Secondo, la compatibilità: il supporto per le nuove generazioni di CPU e una migliore compatibilità della memoria arrivano esclusivamente tramite nuove versioni del firmware; su AM5 attraverso il firmware di riferimento AGESA di AMD, che i produttori di schede integrano nelle proprie versioni del BIOS. Terzo, la stabilità: se un sistema si riavvia spontaneamente e il registro eventi documenta soltanto Kernel-Power 41 con `BugcheckCode=0`, l’arresto anomalo è avvenuto a livello hardware o firmware, senza il coinvolgimento di Windows; cause tipiche sono tensioni instabili e il memory training, ed è proprio questo livello a essere curato dalle release AGESA. Voci nei changelog come "Improve memory compatibility and system stability" oppure una gestione EXPO rivista indicano che un aggiornamento affronta tali problemi. Se invece un sistema funziona stabilmente e non è interessato dalle vulnerabilità corrette, attendere è legittimo; un aggiornamento del BIOS senza motivo è un rischio senza beneficio.

## Passaggio 1: determinare lo stato attuale

Prima di scaricare qualsiasi cosa, sono necessarie due informazioni: il modello esatto della scheda e la versione del BIOS installata. PowerShell fornisce entrambe senza riavviare:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Win32_ComputerSystem` | Argomento posizionale ClassName: classe CIM con produttore e modello del sistema |
| `Win32_BIOS` | Classe CIM con le informazioni sul firmware, tra cui versione e data |
| `Select-Object <eigenschaften>` | riduce l’output alle proprietà indicate |

</details>

Annotate la versione. Vi servirà più avanti per il controllo dell’esito e, leggendo i changelog, per sapere quali versioni state saltando.

## Passaggio 2: scaricare il BIOS e verificare il checksum

Scaricate il BIOS esclusivamente dalla pagina del prodotto del produttore, mai da portali di terze parti. ASRock pubblica il checksum SHA256 per ogni versione; dopo il download confrontatelo prima ancora che il file si avvicini a una chiavetta USB:

```powershell
Get-FileHash .\A620I_Lightning_WiFi_4.43.zip -Algorithm SHA256
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `.\A620I_…_4.43.zip` | Argomento posizionale Path: il file da verificare |
| `-Algorithm SHA256` | Algoritmo hash; deve corrispondere al tipo di checksum pubblicato dal produttore |

</details>

Se il valore non coincide con quello indicato dal produttore, il download è danneggiato o manomesso: non eseguite il flash. Dopo l’estrazione rimane un singolo file ROM, nell’esempio `A62IRW_4.43.ROM` da 32 MB.

## Passaggio 3: preparare la chiavetta USB

Il meccanismo di flash nell’UEFI (per ASRock "Instant Flash", per altri produttori Q-Flash, EZ Flash o M-Flash) legge la chiavetta direttamente dal firmware. Ciò significa che viene riconosciuto in modo affidabile solo FAT32, non NTFS ed exFAT. Quasi tutte le chiavette acquistate già pronte sono formattate in FAT32; potete verificarlo così:

```powershell
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=2" |
  Select-Object DeviceID, FileSystem, VolumeName
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Win32_LogicalDisk` | Classe CIM delle unità logiche |
| `-Filter "DriveType=2"` | Filtro WQL sui supporti rimovibili; esclude dischi rigidi e unità CD |
| `Select-Object DeviceID, FileSystem, VolumeName` | mostra lettera dell’unità, file system e nome del volume |

</details>

Copiate il file ROM nella directory principale della chiavetta. Una nuova formattazione è necessaria solo se il file system non è adatto. La capacità della chiavetta non è rilevante: il file è più piccolo di qualsiasi capacità comune odierna.

Una nota sulla scelta del metodo: molte schede offrono anche un pulsante BIOS Flashback, che effettua il flash senza CPU e senza un sistema funzionante. È la via di recupero per una scheda che non si avvia più. Per un sistema funzionante, Instant Flash nell’UEFI è la soluzione corretta e più semplice. Gli strumenti di flash basati su Windows non sono né necessari né consigliabili sulle piattaforme attuali.

## Passaggio 4: sospendere BitLocker, altrimenti si rischia la richiesta della chiave

Questo è il punto che manca in molte guide. Se il disco di sistema è crittografato con BitLocker (spesso attivato automaticamente in Windows 11 con un account Microsoft), BitLocker vincola la chiave ai valori misurati dal TPM. Un aggiornamento del BIOS modifica questi valori e, al successivo avvio, Windows richiede la chiave di ripristino di 48 cifre. Chi non l’ha a portata di mano si ritrova con un sistema inaccessibile.

BitLocker include un meccanismo dedicato per questo scenario. In PowerShell con diritti di amministratore:

```powershell
Suspend-BitLocker -MountPoint C: -RebootCount 2
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-MountPoint C:` | Il volume interessato, qui il disco di sistema |
| `-RebootCount 2` | Numero di riavvii durante i quali la protezione resta sospesa (da 0 a 15; 0 = fino alla riattivazione manuale) |

</details>

Il valore 2 copre entrambi i riavvii previsti (uno nell’UEFI, uno dopo il flash); in seguito la protezione si riattiva automaticamente e sigilla la chiave rispetto ai nuovi valori misurati. Verificate comunque in anticipo di poter reperire la chiave di ripristino, ad esempio nell’account Microsoft su aka.ms/myrecoverykey oppure tramite `manage-bde -protectors -get C:`.

## Passaggio 5: accedere all’UEFI, anche se F2 non risponde

Il metodo classico con F2 o Canc all’accensione spesso fallisce sui sistemi moderni: con Fast Boot attivato, il firmware inizializza la tastiera USB solo dopo il POST e la pressione del tasto non viene rilevata. Tuttavia non dipendete da quel tasto: Windows può indirizzare direttamente il prossimo riavvio alla configurazione UEFI. In PowerShell con diritti di amministratore:

```powershell
shutdown /r /fw /t 5
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `/r` | Riavvio invece dello spegnimento |
| `/fw` | Imposta la variabile firmware che indirizza il successivo avvio direttamente alla configurazione UEFI; solo insieme a un’opzione di spegnimento come `/r`, richiede diritti di amministratore |
| `/t 5` | Tempo di attesa in secondi prima dell’esecuzione |

</details>

Se il comando segnala l’errore 203 ("The system could not find the environment option that was entered"), quasi sempre mancano i diritti di amministratore: senza elevazione, il processo non può impostare la necessaria variabile firmware e il messaggio di errore non indica questa causa. Un secondo percorso possibile senza variabile firmware passa attraverso l’ambiente di ripristino: `shutdown /r /o`, quindi Risoluzione dei problemi, Opzioni avanzate, Impostazioni firmware UEFI.

## Passaggio 6: eseguire il flash con Instant Flash

Nell’UEFI trovate Instant Flash nel menu Tool. Lo strumento elenca tutti i file ROM sulla chiavetta; dopo la selezione verifica il file, esegue il flash e riavvia autonomamente. Durante questi pochi minuti vale l’unica regola tassativa dell’intera procedura: non interrompete l’alimentazione e non spegnete il computer. Un flash interrotto è l’unico passaggio di questa guida che può effettivamente rendere la scheda non avviabile (richiedendo quindi la già citata procedura di recupero Flashback).

## Passaggio 7: operazioni successive, perché l’aggiornamento ripristina tutto

Dopo il flash, tutte le impostazioni del BIOS sono ai valori di fabbrica. È previsto e offre un’opportunità diagnostica: la RAM ora funziona senza profilo EXPO alla velocità JEDEC di base. Se avete eseguito il flash a causa di problemi di stabilità, lasciatela volutamente così per una o due settimane. Se gli arresti anomali non si ripetono, il profilo della memoria era coinvolto e potrete testare nuovamente EXPO in modo mirato con il nuovo firmware. La differenza nell’uso quotidiano tra 4800 e 6000 MT/s è appena percettibile al di fuori dei benchmark; un computer stabile vale ogni punto di benchmark.

Vale comunque la pena visitare due impostazioni nell’UEFI: chi ha avuto riavvii in stato di inattività può impostare l’opzione "Power Supply Idle Control" su "Typical Current Idle" in Advanced, AMD CBS; ciò attenua una nota incompatibilità di alcuni alimentatori con gli stati di inattività profondi delle CPU Ryzen. E chi in futuro vuole accedere di nuovo alla configurazione tramite F2 può disattivare Fast Boot.

Il controllo dell’esito, una volta tornati in Windows:

```powershell
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion
Get-CimInstance Win32_PhysicalMemory |
  Select-Object PartNumber, ConfiguredClockSpeed
manage-bde -status C:
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `Win32_BIOS` | Classe CIM con la versione del firmware; `SMBIOSBIOSVersion` deve ora mostrare la nuova versione |
| `Win32_PhysicalMemory` | Classe CIM dei moduli di memoria; `ConfiguredClockSpeed` mostra la frequenza effettivamente applicata in MT/s |
| `-status` | manage-bde: mostra lo stato di crittografia e protezione del volume |
| `C:` | Argomento posizionale: il volume da verificare |

</details>

La prima riga deve mostrare la nuova versione, la seconda la frequenza di memoria prevista e BitLocker deve segnalare di nuovo "Protezione attivata". Con questo l’aggiornamento è completato e documentato. Se il flash è stato eseguito per problemi di stabilità, solo l’osservazione nelle settimane successive mostrerà se sono stati risolti, nel modo più semplice controllando le nuove voci Kernel-Power 41 nel registro eventi di sistema.

## Fonti

1.  [ASRock A620I Lightning WiFi, download del BIOS](https://pg.asrock.com/mb/AMD/A620I%20Lightning%20WiFi/index.asp#BIOS): elenco delle versioni con changelog, checksum SHA256 e metodi di aggiornamento supportati dalla scheda di esempio.

2.  [Microsoft Learn: Suspend-BitLocker](https://learn.microsoft.com/en-us/powershell/module/bitlocker/suspend-bitlocker): riferimento per la sospensione della protezione BitLocker, incluso il parametro RebootCount.

3.  [Microsoft Learn: Advanced troubleshooting for Event ID 41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): spiegazione di Kernel-Power 41 e del significato di BugcheckCode 0.

4.  [Microsoft Learn: comando shutdown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/shutdown): documentazione dei parametri /fw e /o per il riavvio nell’UEFI o nell’ambiente di ripristino.
