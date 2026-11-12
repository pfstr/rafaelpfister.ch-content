---
title: "RustDesk 1.5.3: connessioni con approvazione, destinazioni Relay verificate e IPC Windows rafforzato"
navTitle: "RustDesk 1.5.3"
description: "RustDesk 1.5.3 introduce connessioni senza password con approvazione dalla controparte, accetta richieste Relay solo per indirizzi offerti e autentica le pipe IPC su Windows. Le strategie del server non possono più modificare impostazioni sensibili. Si raccomanda di aggiornare tutti i client."
date: "2026-11-11"
kategorie: "Assistenza remota e supporto"
timeToRead: "5 min di lettura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "releases"
  - "haertung"
slug: "rustdesk-1-5-3-connessioni-con-approvazione-destinazioni-relay-verificate-e-ipc-windows"
translationId: article-557eece28ec889f1
translationOf: rustdesk-1-5-3
url: https://rafaelpfister.ch/it/blog/rustdesk-1-5-3-connessioni-con-approvazione-destinazioni-relay-verificate-e-ipc-windows
translationSourceHash: 3d06a4983979b612a9c43bd0a3eacbb0e13d9692a772e95478c351673d707815
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:34:00.577Z
translationReview: automatic
---

# RustDesk 1.5.3: connessioni con approvazione, destinazioni Relay verificate e IPC Windows rafforzato

RustDesk ha pubblicato il 9 novembre 2026 la versione 1.5.3 del client, otto giorni dopo la 1.5.1. Nella pagina delle release non è elencata una versione 1.5.2. La release introduce una nuova funzione (stabilire connessioni con l’approvazione della controparte), una modifica alla logica Relay e 22 correzioni di errori. Diverse riguardano processi rilevanti per la sicurezza: verifica degli indirizzi Relay e degli UUID Relay, autenticazione delle Named Pipe su Windows, chiavi di configurazione bloccate nelle strategie del server e nuovamente il passaggio di consegne dell’installer su Windows. Il produttore non indica né una CVE né un Security Advisory al riguardo (stato: 11 novembre 2026). La release si rivolge a tutti coloro che gestiscono client RustDesk, in particolare su Windows e con un proprio server. Lo stato precedente con aggiornamenti firmati e Trusted Devices V2 è descritto nell’articolo [RustDesk 1.5.1: aggiornamenti firmati, Trusted Devices V2 e rafforzamento del trasferimento file](/blog/rustdesk-1-5-1).

## Le versioni in sintesi

| Versione | Data | Tipo | Punti principali |
|---|---|---|---|
| 1.5.3 | 09.11.2026 | Release client | Request approval, Relay solo verso indirizzi offerti, autenticazione IPC su Windows, chiavi bloccate nelle strategie, correzioni dell’installer (F-Secure, Junction), correzioni per macOS, DRM, Wayland, Android e iOS |

Il server open source (`rustdesk-server`) non ha ricevuto una nuova release; l’ultima versione rimane la 1.1.16 del 20 luglio 2026.

## Nuova funzione: Request approval

Con la PR #16784 è possibile stabilire una connessione senza password RustDesk. La controparte deve accettarla in una finestra di dialogo. Secondo la PR, in questa modalità il client non invia password da alcuna fonte, nemmeno dalla memoria locale o dalla rubrica. Un’eventuale password passata esplicitamente viene ignorata. La selezione compare come sottomenu «Request approval» nel menu a discesa dell’ID e nei menu delle schede dei peer, ciascuno con i tipi di connessione supportati dalla piattaforma e dalla controparte (connessione, trasferimento file e altri).

Per script e collegamenti, la PR indica il parametro della riga di comando `--request-approval` e il parametro URI `request-approval=true`. La modalità viene mantenuta alla riconnessione, spostando una sessione in un’altra finestra e per ulteriori connessioni dalla barra degli strumenti remota. Le connessioni normali mantengono il precedente comportamento relativo alla password.

Secondo la PR, si applicano i seguenti limiti:

- I metodi di autenticazione disponibili non vengono negoziati con la controparte. Se sono consentiti solo accessi tramite password o si applicano limitazioni nella schermata di accesso, queste restano in vigore come prima.
- Se esiste già una finestra per lo stesso peer e lo stesso tipo di connessione, RustDesk la attiva senza modificare il metodo di accesso. Per una nuova connessione con approvazione, chiudere prima la sessione esistente.
- Con il multiplexing dei tunnel, un tunnel già approvato può essere riutilizzato senza una nuova richiesta.

La PR non contiene informazioni sulla compatibilità con versioni precedenti della controparte.

## Relay solo verso indirizzi offerti

La PR #16858 chiude una lacuna nella negoziazione Relay. Finora, il lato controllato adottava senza verifica l’indirizzo Relay e l’UUID da un messaggio `RequestRelay` inoltrato e si connetteva all’indirizzo indicato. Ora il lato controllato memorizza gli indirizzi Relay che gli sono stati offerti, nonché il Relay che sceglierebbe autonomamente (al massimo 32 voci, validità di dieci minuti dall’ultimo messaggio). Le richieste con un Relay diverso vengono scartate prima della risoluzione dei nomi. Entrambi i lati rifiutano inoltre UUID vuoti o più lunghi di 128 byte, oppure contenenti caratteri diversi da lettere ASCII, cifre, `-` e `_`. Secondo la PR, ciò corrisponde al controllo già eseguito da hbbs.

Una richiesta rifiutata non riceve risposta; il tentativo Relay del controller va pertanto in timeout anziché generare un messaggio di errore. Un Relay rimosso dalla configurazione non viene più accettato al più tardi dieci minuti dopo il suo ultimo messaggio.

Per gli operatori di un proprio server è rilevante una limitazione: se hbbs viene eseguito con l’opzione contrassegnata come obsoleta `--mask`, il fallback a un Relay dopo un tentativo WebRTC fallito viene rifiutato se hbbs ha riscritto l’indirizzo Relay per la LAN. Questo punto era ancora aperto al momento del merge. Verificare se l’istanza hbbs utilizza `--mask` e testare le connessioni Relay dopo l’aggiornamento.

## Ulteriori correzioni rilevanti per la sicurezza

- **IPC Windows (PR #16755):** Secondo la PR, altri utenti potevano finora creare ulteriori istanze server delle Named Pipe per l’IPC principale e di servizio e quindi intercettare connessioni a RustDesk, leggere dati IPC o falsificare risposte di configurazione. I client ora aprono le pipe senza questo diritto e autenticano il server prima dello scambio dei dati. Le pipe di servizio devono appartenere a SYSTEM. Secondo la PR, un Denial of Service resta possibile se un attaccante occupa per primo il nome della pipe, e le versioni precedenti con regole di accesso più aperte non possono stabilire connessioni fino all’aggiornamento.
- **Strategie (PR #16696):** Le strategie distribuite dal server tramite heartbeat non possono più modificare impostazioni sensibili. Sono bloccate, tra le altre, `api-server`, `custom-rendezvous-server`, `relay-server`, `key`, `2fa`, `bot`, `enable-trusted-devices`, `trusted-device-expiration-days` e `websocket-port`. Rimangono impostabili, ad esempio, `approve-mode`, `verification-method`, `access-mode`, `whitelist`, `id-whitelist` e `allow-auto-update`. Le chiavi rifiutate vengono registrate senza il rispettivo valore. L’autore definisce espressamente la modifica un rafforzamento limitato: le risposte delle strategie continuano a non essere autenticate.
- **Installer Windows (PR #16889, #16908, #16871):** Il passaggio tramite Named Pipe introdotto con la 1.5.1 è stato nuovamente rimosso poiché, secondo la PR, sui sistemi con F-Secure DeepGuard impediva l’installazione e «Stop service» (errore `0x800700E8`). La protezione dei file predisposti si basa ora sulle DACL restrittive della 1.5.1 e su directory di staging dedicate, i cui handle restano aperti durante il passaggio. Se `C:\Windows\Temp` viene reindirizzato tramite una Junction, l’installer utilizza invece la cartella ProgramData del sistema, anziché interrompersi con un errore.
- **macOS (PR #16729, #16879):** La Privacy Mode viene disattivata al termine della sessione prima del blocco, e il daemon di servizio viene avviato direttamente anziché tramite `sh`.
- **Whitelist IP (PR #16692):** Gli spazi attorno alle voci della whitelist IP vengono ignorati.

Se l’installazione su Windows si interrompe con F-Secure, testare la 1.5.3 su un dispositivo interessato prima della distribuzione su larga scala. Secondo la PR #16889, al momento del merge mancava ancora la conferma della correzione F-Secure da parte del segnalante. Se finora distribuite valori di configurazione come server API, ID o Relay tramite strategie, in futuro dovrete impostarli in un altro modo.

## Correzioni di errori per piattaforma

- **Windows:** La disinstallazione prosegue dopo errori nella sincronizzazione della configurazione, l’IPC di servizio viene ritentato prima della chiusura della GUI e il marker del proprietario degli appunti viene scritto per primo.
- **Linux:** La schermata di accesso con greetd e Wayland viene fornita tramite DRM, i contenuti dello schermo invariati non vengono ricodificati con DRM e gli errori di cattura Wayland non mappati vengono visualizzati con il loro messaggio effettivo anziché «x11 expected».
- **Android:** nessun ulteriore arresto anomalo con contenuti degli appunti superiori a 1 MB, i frame vengono copiati prima del rilascio dell’immagine, l’ebraico e l’indonesiano vengono assegnati correttamente dai vecchi codici locale.
- **iOS:** la schermata iniziale segue la Dark Mode del sistema.
- **Altri:** il port forwarding mantiene aperto l’handle KCP e invia un motivo di chiusura, le barre laterali della finestra di accettazione restano sul monitor corrente in Windows, i dispositivi di input audio vengono cercati solo tra i dispositivi di input e la dimensione minima della finestra principale Sciter non viene più scalata due volte.

## Problemi noti

- **Protezione dal rollback negli aggiornamenti firmati (dalla 1.5.1):** aperto. Le note di rilascio della 1.5.3 non menzionano modifiche al riguardo.
- **Variante portabile su Windows (dalla 1.5.1):** Secondo la PR #15914, il rafforzamento era stato spostato in un branch separato. Le note di rilascio della 1.5.3 non menzionano modifiche al riguardo.
- **Trusted Devices V2 (dalla 1.5.1):** I limiti noti (revoca asincrona, scadenza basata sull’orologio di sistema dell’host) non sono trattati nelle note di rilascio della 1.5.3.
- **Fallback Relay con `--mask` (nuovo):** aperto secondo la PR #16858.
- **Nome della pipe occupato in anticipo (nuovo):** secondo la PR #16755, un Denial of Service resta possibile.
- **Richiesta UAC interrotta (nuovo):** secondo la PR #16889, i file predisposti restano nella directory protetta e possono essere rimossi da un amministratore.
- **Risposte delle strategie (nuovo):** continuano a non essere autenticate. In deployment con HTTP anziché HTTPS, le opzioni ancora consentite sono manipolabili secondo la PR #16696.

## Procedura consigliata

La versione di destinazione per tutti i client è la 1.5.3. Poiché, secondo la PR #16755, le versioni precedenti non possono stabilire una connessione alle pipe IPC rafforzate fino all’aggiornamento, su Windows aggiornare insieme tutti i componenti RustDesk di un dispositivo. Prima del rollout, verificare se distribuite indirizzi server o chiavi tramite strategie, se hbbs viene eseguito con `--mask` e se sui sistemi di destinazione è in uso F-Secure. Il server API dovrebbe essere raggiungibile tramite HTTPS, come già raccomandato per la 1.5.1. Il server open source non deve essere aggiornato per questa release. Le note di rilascio non contengono informazioni sui termini di supporto (stato: 11 novembre 2026).

## Fonti

1.  [GitHub: release RustDesk 1.5.3](https://github.com/rustdesk/rustdesk/releases/tag/1.5.3): note di rilascio con data, nuova funzione, modifica e correzioni di errori.

2.  [GitHub: release RustDesk](https://github.com/rustdesk/rustdesk/releases): elenco delle versioni senza la release 1.5.2.

3.  [GitHub: PR #16784, request approval](https://github.com/rustdesk/rustdesk/pull/16784): procedura, parametri e limiti della connessione con approvazione.

4.  [GitHub: PR #16858, Relay solo verso indirizzi offerti](https://github.com/rustdesk/rustdesk/pull/16858): allowlist per gli indirizzi Relay, verifica UUID e limitazione con `--mask`.

5.  [GitHub: PR #16755, Windows IPC server auth](https://github.com/rustdesk/rustdesk/pull/16755): protezione delle Named Pipe, compatibilità e rischio residuo.

6.  [GitHub: PR #16696, chiavi bloccate nelle strategie](https://github.com/rustdesk/rustdesk/pull/16696): opzioni bloccate e ancora consentite, autenticazione mancante delle risposte.

7.  [GitHub: PR #16889, rimozione del passaggio tramite pipe nell’installer](https://github.com/rustdesk/rustdesk/pull/16889): riferimento a F-Secure DeepGuard e comportamento in caso di richiesta UAC interrotta.

8.  [GitHub: PR #16908, protezione della directory dell’installer](https://github.com/rustdesk/rustdesk/pull/16908): directory di staging dedicate con handle aperti.

9.  [GitHub: PR #16871, staging dell’installer con Junction](https://github.com/rustdesk/rustdesk/pull/16871): utilizzo di ProgramData in caso di Windows Temp reindirizzato.

10.  [GitHub: RustDesk Security Advisories](https://github.com/rustdesk/rustdesk/security/advisories): nessun advisory pubblicato (stato: 11 novembre 2026).

11.  [GitHub: release rustdesk-server](https://github.com/rustdesk/rustdesk-server/releases): ultima versione del server 1.1.16 del 20 luglio 2026.

12.  [RustDesk 1.5.1: aggiornamenti firmati, Trusted Devices V2 e rafforzamento del trasferimento file](/blog/rustdesk-1-5-1): stato della release precedente.
