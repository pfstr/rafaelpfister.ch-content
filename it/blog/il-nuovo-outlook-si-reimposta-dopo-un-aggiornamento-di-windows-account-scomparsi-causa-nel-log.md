---
title: "Il nuovo Outlook si reimposta dopo un aggiornamento di Windows: account scomparsi, causa nel log"
navTitle: "Reimpostazione di Outlook"
description: "Dopo il riavvio per un aggiornamento, il nuovo Outlook per Windows si avvia come appena installato e tutti gli account e le impostazioni risultano assenti. La causa è ricostruibile nel log Olk: un controllo attivato lato server (Flight nh-userDataSentinel) non trova un file marcatore e cancella i dati locali senza chiedere conferma. L’articolo mostra le voci di log, come verificare il caso sui propri dispositivi e cosa fare dopo la reimpostazione."
date: "2026-11-11"
kategorie: "Outlook"
timeToRead: "6 min di lettura"
themen:
  - outlook
  - windows-client
produkte:
  - "outlook"
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "il-nuovo-outlook-si-reimposta-dopo-un-aggiornamento-di-windows-account-scomparsi-causa-nel-log"
translationId: "article-c1340c0bc1862647"
translationOf: outlook-new-reset-nach-update
url: https://rafaelpfister.ch/it/blog/il-nuovo-outlook-si-reimposta-dopo-un-aggiornamento-di-windows-account-scomparsi-causa-nel-log
translationSourceHash: 405d93eb98233f046e7ecb2823969a601db0041bb6ddaef8eeaa4c7d858e393c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:27:50.626Z
translationReview: automatic
---

# Il nuovo Outlook si reimposta dopo un aggiornamento di Windows: account scomparsi, causa nel log

Dopo il riavvio per gli aggiornamenti di Windows del novembre 2026, il nuovo Outlook per Windows si è avviato su uno dei miei dispositivi come dopo una nuova installazione. L’account aziendale configurato come account principale mancava, così come gli altri account, le firme e le impostazioni personali. Non è apparso alcun messaggio di errore: Outlook ha mostrato direttamente la procedura guidata per aggiungere un account.

La causa è riportata nel registro locale di Outlook: l’app ha eliminato autonomamente i propri dati locali perché un controllo appena attivato ha valutato lo stato come "reimpostazione necessaria". Non vi sono indicazioni di un intervento da parte di Windows Update, di un utente o di un aggressore.

## Sequenza

| Ora (ora locale) | Evento | Fonte |
|---|---|---|
| 18:26 | Ultima voce della sessione Outlook in esecuzione, nessuna chiusura ordinata | `Olk\logs\olk_*.log` |
| 18:27 e 18:28 | Due riavvii tramite Windows Update (`MoNotificationUx.exe`, quindi `TrustedInstaller.exe`) | Log di sistema, evento 1074 |
| 18:41:55 | Outlook si avvia, il controllo `nh-userDataSentinel` è attivo e segnala `ResetRequired` | nuovo `olk_*.log` |
| 18:41:56 | Outlook elimina i dati locali e si avvia come nuova installazione | nuovo `olk_*.log` |

I timestamp nel log Olk e nei nomi dei file di log sono in UTC. Nella tabella sono convertiti in ora locale (CET).

## Le voci nel log

Le prime righe della nuova sessione mostrano il controllo e il risultato:

```text
Checking flight enabled status  flightName nh-userDataSentinel  isEnabled true
Reset sentinel check completed.  installState ResetRequired
  settingsFound true  resetSentinelFlagFound true
Reset sentinel check triggered user data reset.
Prompted for clear local state  clearLocalStateName Silent
```

Seguono quindi le operazioni di eliminazione, ciascuna con `succeeded true`:

| Percorso in `%LOCALAPPDATA%\Microsoft\Olk` | Contenuto |
|---|---|
| `EBWebView` | Profilo WebView2 con stato di accesso, elenco degli account e dati offline |
| `UserSettings.json` | Impostazioni dell’app |
| `Attachments` | Allegati memorizzati localmente nella cache |
| `pst_index_v2` | Indice dei file PST collegati |
| `cache\Notification`, `cache\*.db` | Cache delle notifiche e dei processi in background |
| `updated.txt`, `xpdApi.log`, `Feedback` | File di stato e di supporto |

Alcune righe più avanti compare `Last update date file missing, creating it. Barring clearing local state, this is likely a fresh install`. Da quel momento Outlook si comporta come dopo una nuova installazione.

## Perché il controllo è scattato

`nh-userDataSentinel` è un Flight, ovvero una funzione che Microsoft abilita lato server per singole installazioni. In tutti i log precedenti del dispositivo, per questo Flight compare `Flight status not set; using default (false)`, l’ultima volta alle 13:23 UTC dello stesso giorno. All’avvio delle 17:41 UTC era attivo per la prima volta.

Il controllo cerca un file marcatore nella cartella del pacchetto dell’app:

```text
%LOCALAPPDATA%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalState\OlkSentinel
```

Se Outlook trova impostazioni utente nella cartella Olk (`settingsFound true`), ma nessun marcatore corrispondente nella cartella del pacchetto, lo interpreta come un’indicazione che l’app è stata reimpostata o reinstallata e cancella i dati utente. Sul dispositivo interessato, `OlkSentinel` è stato creato di nuovo alle 18:41:56, mentre gli altri file in `LocalState` risalgono a settembre 2026. La cartella del pacchetto non è stata quindi reimpostata; il file marcatore semplicemente non esisteva prima, perché il controllo era nuovo. Al primo avvio con il Flight attivo, il risultato è stato quindi una reimpostazione, sebbene non fosse necessaria.

Secondo i log, il riavvio tramite Windows Update è solo il momento dell’evento, non la causa: il Flight viene valutato all’avvio e il primo avvio dopo l’aggiornamento è stato il primo con il Flight attivo. La versione dell’app (1.2026.1026.100) era la stessa prima e dopo il riavvio.

## Verificare il caso sui propri dispositivi

La seguente query cerca in tutti i log Olk le voci del controllo:

```powershell
$logs = "$env:LOCALAPPDATA\Microsoft\Olk\logs"
Get-ChildItem $logs -Filter 'olk_*.log' |
  Select-String -Pattern 'Reset sentinel|Delete directory tree' |
  Select-Object Filename, LineNumber, Line
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `-Filter 'olk_*.log'` | solo i log delle sessioni dell’app, senza archivi ZIP e log push |
| `-Pattern` | espressione regolare; intercetta il controllo e ogni operazione di eliminazione |
| `Select-Object Filename, LineNumber, Line` | mostra per ogni risultato file di log, numero di riga e testo |

</details>

Outlook archivia le sessioni meno recenti in file ZIP nella stessa cartella. Se la reimpostazione risale a qualche giorno prima, la voce potrebbe trovarsi solo nell’archivio e deve essere estratta prima.

Come secondo indizio, sono utili gli orari di creazione: se `EBWebView` e `UserSettings.json` nella cartella Olk, nonché `OlkSentinel` nella cartella del pacchetto, hanno lo stesso timestamp, la reimpostazione è avvenuta in quel momento.

```powershell
$olk = "$env:LOCALAPPDATA\Microsoft\Olk"
$pkg = "$env:LOCALAPPDATA\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe"
Get-Item "$olk\EBWebView", "$olk\UserSettings.json", "$pkg\LocalState\OlkSentinel" |
  Select-Object FullName, CreationTime
```

## Cosa è stato perso e cosa no

Sono stati eliminati solo i dati locali dell’app. Cassette postali, calendari e contatti sono archiviati sul server in Microsoft 365 e Outlook.com e riappariranno dopo un nuovo accesso. L’accesso a Windows (Web Account Manager) resta invariato; gli account Microsoft configurati in Windows vengono proposti direttamente da Outlook durante l’aggiunta.

È necessario configurare nuovamente:

- tutti gli account, nell’ordine desiderato (il primo account aggiunto è quello principale, vedi sotto)
- le firme salvate localmente, se non vengono ripristinate tramite la sincronizzazione cloud dell’account
- i file PST collegati
- le impostazioni dell’app, quali visualizzazione, notifiche e comportamento all’avvio

Per l’account principale, l’ordine è decisivo, poiché il nuovo Outlook offre alcune funzioni solo nell’account primario, ad esempio S/MIME (vedi [Firma S/MIME nell’account secondario](/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen)). Aggiungete quindi per primo l’account che deve essere principale. Se Outlook ha già acquisito automaticamente un altro account dopo la reimpostazione, rimuovetelo, aggiungete l’account principale desiderato e quindi configurate nuovamente gli altri account.

## Per gli amministratori

Dopo che si è verificata, la reimpostazione non può essere annullata; un backup della cartella `%LOCALAPPDATA%\Microsoft\Olk` prima dell’avvio è l’unico modo per recuperare lo stato precedente. Negli ambienti con contenitori di profilo (FSLogix) e sessioni VDI non persistenti, è opportuno verificare che nel profilo vengano mantenute sia la cartella Olk sia la cartella del pacchetto `Microsoft.OutlookForWindows_8wekyb3d8bbwe`. Se viene mantenuta solo una delle due cartelle, ciò corrisponde esattamente allo stato che il controllo interpreta come nuova installazione.

Le segnalazioni degli utenti secondo cui il nuovo Outlook avrebbe "dimenticato tutti gli account" possono essere classificate in pochi minuti con la query del log riportata sopra. Se compare `Reset sentinel check triggered user data reset`, non si tratta di un problema di account né di un incidente di sicurezza.

## Fonti

1.  [Microsoft Learn: Deploy the new Outlook for Windows on VDI](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/virtualized-desktop-infrastructure): percorsi di archiviazione dei dati locali del nuovo Outlook e indicazioni sui contenitori di profilo.

2.  [Microsoft Learn: Control installing and using the new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/control-install): distribuzione e gestione dell’app in ambienti gestiti.

3.  [Microsoft Q&A: Outlook (new) won't open, makes me attempt to sign in](https://learn.microsoft.com/en-us/answers/questions/5863503/outlook-new-wont-open-makes-me-attempt-to-sign-in): segnalazioni degli utenti su account persi dopo la reimpostazione dei dati locali.
