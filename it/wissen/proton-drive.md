---
title: "Proton Drive: modello crittografico dei dati e funzionamento della sincronizzazione"
blatt: "proton-drive"
description: "Proton Drive per amministratori di storage, sicurezza e piattaforme: archiviazione cloud centralizzata con crittografia end-to-end lato client, volumi, condivisioni, nodi, revisioni e blocchi, gerarchia di chiavi e firme OpenPGP, Drive SDK, sincronizzazione desktop e file on-demand, automazione CLI, condivisione, conflitti, versioni, logging, ripristino e storia tecnica."
fakten:
  - label: Ruolo del sistema
    wert: archiviazione cloud centralizzata di file con crittografia end-to-end lato client, sincronizzazione, condivisione e versionamento
    href: https://proton.me/support/drive
  - label: Modello di fiducia
    wert: il server memorizza contenuti e nomi cifrati; il testo in chiaro e il materiale delle chiavi sbloccato sono disponibili nel client autorizzato
    href: https://proton.me/blog/protondrive-security
  - label: Modello dei dati
    wert: un Volume contiene Shares; uno Share fa riferimento a una radice dell’albero; file e cartelle sono Nodes con Links, chiavi e revisioni
    href: https://proton.me/blog/protondrive-security
  - label: Trasferimento dei file
    wert: le revisioni sono costituite da blocchi cifrati; il client cifra, trasferisce, verifica e ricompone l’oggetto
    href: https://proton.me/blog/drive-cryptography-update
  - label: Crittografia
    wert: gerarchia di chiavi basata su OpenPGP; i nuovi file utilizzano v6 PKESK, v2 SEIPD e AES-GCM, mentre le revisioni precedenti restano vincolate allo schema
    href: https://proton.me/blog/drive-cryptography-update
  - label: Metadati
    wert: il contenuto e i nomi di file e cartelle sono E2EE; secondo Proton, alcuni campi quali la dimensione del file restano leggibili dal server
    href: https://proton.me/support/drive
  - label: Integrità
    wert: informazioni firmate sulle chiavi, sull’albero e sui blocchi, nonché firme OpenPGP, rilevano le manomissioni
    href: https://proton.me/support/drive-signature-management
  - label: Nucleo client
    wert: Drive SDK unifica le operazioni sui file e la crittografia nei client web, desktop, mobili e CLI
    href: https://github.com/ProtonDriveApps/sdk
  - label: Sincronizzazione desktop
    wert: Windows e macOS integrano segnaposto on-demand, stati offline locali e sincronizzazione in background rispettivamente in Explorer e Finder
    href: https://proton.me/support/proton-drive-windows-on-demand-sync
  - label: CLI
    wert: programma autonomo ufficiale per Windows, macOS e Linux; operazioni su file e condivisioni, ma nessun motore permanente di sincronizzazione desktop
    href: https://proton.me/support/drive-cli
  - label: Nucleo del ripristino
    wert: il solo accesso all’account non basta dopo una reimpostazione della password; la vecchia chiave viene nuovamente sbloccata tramite password precedente, Recovery Phrase o Recovery File
    href: https://proton.me/support/drive-data-recovery
  - label: Regola di backup
    wert: sincronizzazione e cronologia delle versioni non sono un backup indipendente; i dati critici richiedono esportazione separata, checksum, conservazione e test di ripristino
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
werbung:
  - newsletter
ctaThemen:
  - proton-drive
translationSourceHash: 6063c7d85190de63a8973bc7cdb304fff10f81fd617a2044cf40f8371849717c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:29:08.360Z
translationReview: automatic
---

# Proton Drive: modello crittografico dei dati e funzionamento della sincronizzazione

Proton Drive è un archivio cloud centralizzato per file, i cui client cifrano contenuti e metadati essenziali prima del caricamento. Il servizio combina accesso web, sincronizzazione desktop, app mobili, condivisioni, versioni e una riga di comando con un modello di dati crittografato end-to-end. La [panoramica del supporto](https://proton.me/support/drive) indica l’accesso da web, Windows, macOS, Android e iOS/iPadOS; la [Drive CLI ufficiale](https://proton.me/support/drive-cli) aggiunge operazioni scriptabili su Windows, macOS e Linux.

Per gli amministratori, Proton Drive non è né una classica condivisione file né un semplice portale di upload. Un client autorizzato gestisce la sessione dell’account e le chiavi, risolve un albero cifrato, osserva le modifiche locali, crea revisioni, cifra blocchi e sincronizza nuovamente nel file system gli eventi lato server. I server restano responsabili di disponibilità, autorizzazione, quota di archiviazione, versioni e condivisioni, senza poter elaborare in chiaro il contenuto dei file o i nomi cifrati.

La spiegazione segue un file dal client, attraverso chiavi, blocchi cifrati e API, fino all’archiviazione lato server. Sincronizzazione, condivisioni, versionamento, diagnostica e backup indipendente si basano su questo percorso dei dati.

Proton Drive archivia i dati centralmente, ma trasferisce la cifratura e la verifica delle firme ai client fidati. L’articolo segue un file dalla modifica locale attraverso la gerarchia delle chiavi, i blocchi e la revisione fino alla sincronizzazione, alla condivisione e al ripristino.

## Architettura: servizio centrale, client fidato

Proton ha scelto uno storage cloud centralizzato anziché una rete peer-to-peer. I server forniscono API, object storage, eventi, appartenenze, quote, versioni e disponibilità; i client si occupano dell’elaborazione in chiaro e della crittografia. Proton illustra la motivazione di questo modello nel [confronto tra storage centralizzato e decentralizzato](https://proton.me/blog/centalized-vs-p2p-protondrive).

Il percorso tecnico comprende sei livelli:

1. **Identità e sessione:** il client autentica un account Proton, carica le chiavi di indirizzo cifrate e le sblocca localmente.
2. **Volume e Share:** un Volume delimita lo spazio di archiviazione. Gli Shares definiscono radici dell’albero, membri e diritti.
3. **Node e Link:** un file o una cartella è un Node; il Link lo colloca nella gerarchia cifrata.
4. **Revisione e blocco:** un file può avere più revisioni. Il contenuto viene suddiviso in blocchi e cifrato lato client.
5. **API e object storage:** il client trasmette metadati cifrati, pacchetti di chiavi, manifest e blocchi tramite HTTPS.
6. **File system locale:** i client desktop proiettano l’albero cloud come segnaposto o file locale, osservano le modifiche e risolvono i conflitti.

Il [modello di sicurezza di Proton Drive](https://proton.me/blog/protondrive-security) documenta Volume, Share, Node, Link e chiavi gerarchiche. L’[aggiornamento crittografico del 2026](https://proton.me/blog/drive-cryptography-update) descrive inoltre Node, Revision, Block e Content-Key-Package del motore client condiviso.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-proton-drive.svg?v=20260813" title="Interaktive Infografik: Proton Drive von Konto und Share über Node-, Revisions- und Blockverschlüsselung bis zu API, SDK, Desktop-Sync, CLI, Sharing, Versionierung, Recovery und unabhängiger Sicherung" loading="lazy">
  <a href="/images/kb-interaktiv-proton-drive.svg?v=20260813">Aprire direttamente il grafico interattivo</a>.
</iframe>

## Modello dei dati: Volume, Share, Node, Link e Revision

Un **Volume** è uno spazio di archiviazione assegnato. La sua condivisione predefinita punta alla radice dell’albero di directory privato. Ulteriori **Shares** possono puntare a file o cartelle e associare membri con ruoli. Uno Share è quindi al tempo stesso punto di accesso e confine crittografico.

Un **Node** descrive un file o una cartella e contiene attributi tecnici nonché materiale crittografico. Un **Link** posiziona il Node nell’albero e lo collega al nodo padre. Per i file, le **Revisions** gestiscono stati di contenuto successivi; una revisione fa riferimento a **Blocks** cifrati. La distinzione concettuale spiega importanti fenomeni operativi:

- La ridenominazione modifica la designazione cifrata nell’albero, non necessariamente il contenuto del file.
- Lo spostamento modifica il riferimento al padre e quindi le informazioni Link protette crittograficamente.
- La sostituzione crea una revisione dello stesso oggetto file, mentre «mantieni entrambi» crea un secondo Node.
- La condivisione delega l’accesso a una radice Share, anziché decifrare lato server tutti i contenuti sottostanti.
- L’eliminazione sposta inizialmente una voce nel Cestino di Proton Drive; i cestini locali rappresentano uno stato separato.

La [gestione delle versioni](https://proton.me/support/version-history) ufficiale descrive Replace, Keep both, Skip e il ripristino di revisioni precedenti. La [documentazione su eliminazione e ripristino](https://proton.me/support/proton-drive-delete-restore-synced-files) spiega la differenza fra il Cestino cloud e quello locale.

Volume, Share, Node e Revision descrivono quale oggetto viene archiviato. La gerarchia delle chiavi spiega poi chi può decifrare e firmare nomi, metadati e contenuto.

## Gerarchia delle chiavi e OpenPGP

Le chiavi e le passphrase vengono generate sul client e trasmesse al server solo in forma cifrata. Il modello di sicurezza utilizza più livelli:

1. Un indirizzo Proton possiede una coppia di chiavi asimmetriche.
2. Per uno Share vengono generate una passphrase Share casuale e una chiave Share.
3. La passphrase Share viene cifrata e firmata per ogni membro autorizzato mediante la relativa chiave di indirizzo.
4. Ogni Node di file o cartella possiede chiavi Node e una passphrase Node.
5. La passphrase Node viene protetta con la chiave della cartella padre o della radice Share.
6. Il nome e ulteriori payload vengono cifrati in questo contesto gerarchico.
7. I contenuti dei file usano una Content Key simmetrica, il cui pacchetto di chiavi è protetto dalla chiave Node.

OpenPGP funge da formato per messaggi e pacchetti di chiavi. La crittografia asimmetrica protegge le chiavi di sessione simmetriche; la cifratura simmetrica elabora efficientemente i dati veri e propri. La base normativa di OpenPGP è [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580).

Per i file creati di recente dopo la migrazione crittografica, Proton descrive pacchetti v6-PKESK, v2-SEIPD e AES-GCM. Lo schema di un file rimane vincolato attraverso le sue revisioni: durante la sostituzione, un client deve usare lo schema di pacchetti appropriato all’oggetto file esistente, altrimenti la revisione viene rifiutata ([aggiornamento crittografico](https://proton.me/blog/drive-cryptography-update)). Questo è un motivo di interoperabilità per aggiornare i client in modo controllato.

## Cifrato, firmato e leggibile dal server

«Crittografato end-to-end» non significa che il server non veda alcun dato operativo. Nella [panoramica del supporto Drive](https://proton.me/support/drive), Proton indica contenuto dei file, nomi dei file e nomi delle cartelle come E2EE, ma segnala anche che determinati campi, come la dimensione del file, restano leggibili dal server per consentire funzioni quali l’ordinamento.

In un modello di minaccia, le classi di dati vengono separate:

| Classe di dati | Percorso di protezione | Rilevanza amministrativa |
|---|---|---|
| Contenuto del file | blocchi cifrati lato client | il server non può indicizzare o riparare il contenuto |
| Nome file/cartella | payload cifrato nell’albero | la ricerca dei nomi lato server è limitata |
| Pacchetti di chiavi | cifrati e firmati | il ripristino dipende dalle chiavi private |
| Firme/manifest | prova di integrità e autore | gli avvisi non devono essere ignorati indiscriminatamente |
| Dimensione del file | campo leggibile dal server | restano visibili i modelli di volume e traffico |
| Riferimenti ad account, sessione e Share | metadati del control plane | autorizzazione e audit restano lato server |
| IP, ora, versione del client | metadati operativi/di sicurezza | parte del modello di minaccia del provider e dell’endpoint |

Le firme digitali confermano l’uploader e i payload inalterati. La [gestione delle firme](https://proton.me/support/drive-signature-management) spiega la verifica delle firme OpenPGP e gli avvisi. Una firma non prova che il contenuto sia innocuo o tecnicamente corretto; attesta origine o inalterabilità nel contesto di chiavi verificato.

## Blocchi, revisioni e confine di commit

I file vengono suddivisi in blocchi lato client. Il client crea o acquisisce la Content Key, cifra i blocchi, li trasferisce e pubblica la revisione con le informazioni crittografiche associate. Il server può memorizzare e distribuire i blocchi senza conoscere il testo in chiaro.

L’avanzamento dell’upload non costituisce ancora un commit applicativo. Un amministratore distingue:

- file locale letto completamente,
- tutti i blocchi cifrati,
- tutti i blocchi trasferiti con successo,
- revisione accettata dal server,
- cursore degli eventi aggiornato,
- file risolvibile su un secondo client,
- firma e contenuto verificati durante il download.

Per download di grandi dimensioni, il client decifra e verifica localmente i dati. Per grandi quantità di dati, il client web utilizza un percorso di streaming con Service Worker, evitando di dover mantenere l’intero archivio nella memoria del browser; Proton descrive il processo in [download e streaming web](https://proton.me/blog/proton-drive-web-encryption-technique).

## Drive SDK e architettura del client

L’ecosistema client è nato originariamente da implementazioni specifiche per piattaforma. Il **Drive SDK** condiviso uniforma i percorsi di account, API, cache, Node, Share, upload, download e crittografia. Il codice sorgente è disponibile nel [repository ufficiale Proton Drive SDK](https://github.com/ProtonDriveApps/sdk); il contratto del pacchetto JavaScript/TypeScript è pubblicato come [`@protontech/drive-sdk`](https://www.npmjs.com/package/@protontech/drive-sdk).

L’organizzazione GitHub ufficiale mostra gli stack tecnologici principali ([ProtonDriveApps](https://github.com/ProtonDriveApps)):

| Client/componente | Stack principale | Ruolo operativo |
|---|---|---|
| Web | TypeScript/JavaScript e crittografia del browser | accesso interattivo, streaming, condivisione |
| Drive SDK / CLI | TypeScript; CLI come programma autonomo pacchettizzato con Bun | operazioni comuni sui file e automazione |
| [Windows](https://github.com/ProtonDriveApps/windows-drive) | C#/.NET più componenti SDK/Crypto condivisi | sincronizzazione in background, Explorer e Cloud Files |
| [macOS](https://github.com/ProtonDriveApps/mac-drive) | Swift più componenti SDK condivisi | sincronizzazione in background, Finder/File Provider |
| Android | Kotlin | file mobili, stato offline e foto |
| iOS/iPadOS | Swift | file mobili, stato offline e foto |

L’approccio SDK riduce le differenze fra implementazioni crittografiche e di gestione file, ma non elimina i confini fra piattaforme. Regole dei nomi di Windows, comportamento di macOS File Provider, limiti delle attività in background mobili e integrazione desktop Linux restano compiti del rispettivo client.

L’API crittografica non è ancora un servizio di sincronizzazione file. Il client desktop osserva modifiche locali e remote, le correla e decide se eseguire upload, download o gestire un conflitto.

## Sincronizzazione desktop come macchina a stati

La sincronizzazione desktop non mantiene una semplice copia di byte, ma riconcilia due spazi dei nomi mutevoli. Un ciclo astratto è il seguente:

1. Caricare lo stato del file system locale e il cursore degli eventi cloud.
2. Rilevare modifiche locali, eventi lato server e trasferimenti in sospeso.
3. Normalizzare i nomi e le regole della piattaforma oppure contrassegnare i conflitti.
4. Risolvere la gerarchia di chiavi e i Nodes necessari.
5. Pianificare upload, download, Move, Delete o hydration.
6. Trasferire il contenuto in blocchi ed eseguire il commit della revisione.
7. Aggiornare atomicamente segnaposto locali, attributi, cache e cursore.
8. Ritentare gli errori con un riferimento stabile all’oggetto oppure escalation visibile.

Lo stato di sincronizzazione «sincronizzato» è significativo solo per il client e il cursore considerati. Un secondo client, una condivisione o un backup separato forniscono una prova end-to-end più forte.

## File on-demand e stati locali

Windows e macOS mostrano l’albero cloud nel file manager senza memorizzare permanentemente tutti i contenuti in locale. La [documentazione Windows On-Demand](https://proton.me/support/proton-drive-windows-on-demand-sync) distingue **Available when online**, **Available on this device** e **Always available on this device**. La [guida macOS](https://proton.me/support/drive-macos-guide) distingue Online Only, temporaneamente locale e Make Available Offline.

Un nome file visibile non prova quindi l’esistenza di una copia locale. Per backup, antivirus, indicizzatore, sistema di build o flusso di lavoro per allegati, occorre chiarire se il file:

- esiste solo come segnaposto,
- viene idratato all’apertura,
- è temporaneamente memorizzato nella cache locale,
- è stato esplicitamente fissato per l’uso offline,
- è stato modificato localmente ma non ancora caricato,
- si trova solo su questo dispositivo a causa di un errore.

Annidare due client cloud on-demand nello stesso albero è rischioso. La [guida agli errori di Windows](https://proton.me/support/proton-drive-windows-common-issues) indica esplicitamente come problema i file di un altro storage cloud disponibili solo online. Un processo di backup fissa localmente l’insieme di input oppure usa la CLI ufficiale per trasferimenti espliciti.

## Conflitti, nomi e funzionamento su più dispositivi

I conflitti sorgono quando client e cloud modificano indipendentemente lo stesso Node logico oppure quando la piattaforma di destinazione non può rappresentare un nome. Le classi tipiche sono:

- modifica parallela del contenuto della stessa revisione,
- Delete contro Modify,
- Rename contro Move,
- maiuscole/minuscole su piattaforme con semantica del case diversa,
- caratteri non ammessi o nomi riservati in Windows,
- upload con lo stesso nome con Replace, Keep both o Skip,
- modifica offline dopo un lungo ritardo degli eventi,
- file locale bloccato da un’applicazione durante il trasferimento.

Proton documenta che un conflitto può creare un file rinominato o una nuova versione ([Shared with me in Windows](https://proton.me/support/shared-with-me-windows)). La guida agli errori di Windows descrive file aperti e nomi non validi ([problemi comuni di sincronizzazione](https://proton.me/support/proton-drive-windows-common-issues)). Gli amministratori non cercano soltanto «errori», ma anche copie in conflitto, upload in sospeso e file che non sono mai arrivati nel cloud a causa delle regole sui nomi.

## Inventario client, binario e log

Un endpoint gestito inventaria piattaforma, pacchetto client, firma, avvio automatico, account, radice di sincronizzazione, disponibilità locale, spazio libero e percorso dei log. I numeri di versione appartengono all’inventario di runtime e non a un fatto statico dell’articolo.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Clientinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process ProtonDrive -ErrorAction SilentlyContinue
Get-AuthenticodeSignature "$env:LOCALAPPDATA\Proton\Drive\ProtonDrive.exe"
Get-ChildItem "$env:LOCALAPPDATA\Proton\ProtonDrive\Logs"
proton-drive version</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive version
file ./proton-drive
sha256sum ./proton-drive
ldd ./proton-drive</code></pre>
  </div>
</div>

[`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) verifica il processo Windows; [`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) verifica la firma del codice; [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventaria i log. In Unix, [`file`](https://man7.org/linux/man-pages/man1/file.1.html) identifica il binario, [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) il relativo hash e [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html) le dipendenze dinamiche. L’accesso ai log supportato per Windows e macOS è riportato in [Proton Drive Logs](https://proton.me/support/proton-drive-logs).

## CLI: operazione esplicita invece della sincronizzazione in background

La Proton Drive CLI è un client basato su SDK per Windows, macOS e Linux. L’accesso avviene tramite flusso browser; secondo l’[introduzione alla CLI](https://proton.me/blog/proton-drive-cli), le sessioni vengono memorizzate in Windows Credential Manager, macOS Keychain o libsecret. La CLI elenca, carica e scarica, gestisce file, Cestino, condivisioni e inviti. Viene eseguita come operazione singola e poi termina.

Il confine operativo più importante è esplicitamente documentato: la CLI non sostituisce un motore di sincronizzazione desktop. Un upload pianificato non rileva automaticamente eliminazioni, ridenominazioni, conflitti bidirezionali o l’esito su un secondo dispositivo. Chi usa la CLI come trasporto di backup implementa autonomamente inventario sorgente, direzione, regola di eliminazione, ritentativi, exit code, verifica della destinazione e conservazione.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Anmeldung und Proton-Drive-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | ConvertFrom-Json
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | jq .
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
</div>

Tutti i comandi e le guide della CLI sono documentati in [Using Proton Drive CLI](https://proton.me/support/drive-cli). [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) e [`jq`](https://jqlang.org/manual/) elaborano output leggibile dalle macchine. Le automazioni si basano su campi JSON ed exit code, non su testi a schermo localizzati.

## Upload, strategia di conflitto e verifica

Un upload ripetibile indica sorgente locale, root di destinazione e strategia di conflitto. I caratteri jolly vengono espansi dalla shell chiamante e possono fornire insiemi diversi in Windows e Unix. I processi critici creano quindi preventivamente un manifest con percorso, dimensione e hash.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Upload und Manifest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\reports -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\reports-manifest.csv -NoTypeInformation
proton-drive filesystem upload .\reports\* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json | Out-File .\drive-result.json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./reports -type f -print0 | sort -z | xargs -0 sha256sum > reports-manifest.sha256
proton-drive filesystem upload ./reports/* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json > drive-result.json
jq . drive-result.json</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv), [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) e [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) generano l’insieme locale previsto. Tuttavia, l’elenco del provider non prova ancora l’uguaglianza del contenuto. Per dati critici segue un download in una destinazione separata e un confronto hash.

## Download e collaudo end-to-end

Un test di download utilizza un nuovo percorso di destinazione, affinché un file locale già esistente non alteri il risultato. Successivamente viene verificato rispetto al manifest della sorgente. In presenza di più revisioni, il test deve stabilire quale revisione sia prevista.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Downloadprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">New-Item -ItemType Directory .\restore-test
proton-drive filesystem download /my-files/Reports .\restore-test
Get-ChildItem .\restore-test -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\restore-manifest.csv -NoTypeInformation</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mkdir -p ./restore-test
proton-drive filesystem download /my-files/Reports ./restore-test
find ./restore-test -type f -print0 | sort -z | xargs -0 sha256sum > restore-manifest.sha256
diff -u reports-manifest.sha256 restore-manifest.sha256</code></pre>
  </div>
</div>

[`New-Item`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item) e [`mkdir`](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html) creano la destinazione isolata; [`diff`](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html) confronta i manifest. I percorsi assoluti o con prefissi differenti vengono normalizzati prima del confronto.

## Condivisione con account e tramite link pubblici

Proton Drive supporta almeno due modelli di condivisione:

- **Appartenenza/invito:** una persona diventa membro di uno Share con ruolo Viewer o Editor. La passphrase Share viene confezionata per la chiave di indirizzo del membro.
- **Link pubblico:** un link consente l’accesso senza account Proton. Una password protegge la passphrase Share; link e password formano insieme il segreto di accesso.

La [gestione degli accessi](https://proton.me/support/drive-manage-access-shared-files) documenta ruoli, delega Editor, revoca e impostazioni separate per invito e-mail e Public Link. [Password e data di scadenza](https://proton.me/support/password-protect-files-proton-drive) limitano l’accesso tramite link. Se una password casuale viene inviata come frammento URL, browser e destinatario vedono il link completo; il frammento non viene inviato al server come percorso HTTP, ma può essere esposto tramite appunti, cronologia del browser, screenshot o telemetria dell’endpoint.

Gli amministratori inventariano proprietario, radice Share, membri, ruolo, diritto di ricondivisione, stato del Public Link, data di scadenza e ultima verifica. A seconda dell’impostazione, un Editor può condividere autonomamente e modificare i diritti. L’offboarding non revoca quindi solo l’account, ma controlla anche tutti gli Shares e i link esterni.

## Versioni, Cestino e backup

La cronologia delle versioni memorizza revisioni precedenti dello stesso oggetto file. Una revisione ripristinata non elimina automaticamente le versioni più recenti. Il Cestino conserva i Nodes eliminati fino all’eliminazione definitiva. Entrambe le funzioni sono utili, ma rimangono nello stesso dominio di guasto amministrativo e crittografico.

Non proteggono in modo affidabile da:

- perdita o impossibilità di ripristinare la chiave dell’account,
- diritti errati o eliminazione definitiva dolosa,
- eventi relativi a quota, contratto o provider,
- errori verificatisi prima della prima revisione cloud riuscita,
- omissioni di sincronizzazione non rilevate,
- compromissione dell’identità o dell’endpoint a livello organizzativo.

Un backup indipendente legge un insieme cloud definito con un client autorizzato, lo scrive in una destinazione gestita separatamente, lo accompagna con manifest, conservazione e cronologia immutabile ed esegue test di ripristino. NIST SP 800-34 tratta backup, archiviazione alternativa e test di recovery come controlli interconnessi della pianificazione di continuità ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)). Le basi sono disponibili in [Backup e Disaster Recovery](/kb/backup-dr) e [Storage](/kb/storage).

## Identità, sessione e fiducia nell’endpoint

La crittografia end-to-end sposta la fiducia dal provider al client, all’account e all’endpoint. Chi controlla il dispositivo sbloccato, la sessione del browser o l’account può decifrare e firmare modifiche a nome dell’utente. Per questo MFA, chiavi di sicurezza hardware, protezione del dispositivo, revoca delle sessioni, provenienza del client e protezione della cache locale fanno parte dello stesso ambito.

Il [modello di minaccia](https://proton.me/blog/proton-drive-threat-model) cita, tra l’altro, dispositivi compromessi, phishing e autenticità dei client ufficiali come limiti. L’open source permette la revisione del codice, ma non sostituisce la verifica del binario realmente eseguito e del relativo percorso di aggiornamento.

Anche il controllo antimalware è limitato: il [rilevamento di file dannosi](https://proton.me/support/proton-drive-malware-protection) opzionale confronta, su richiesta, un hash con hash di malware noti. Non decifra lato server l’intero patrimonio, non rileva contenuti sconosciuti e non sostituisce un sensore endpoint.

## Reimpostazione della password e ripristino dei dati

Il ripristino dell’account e il ripristino dei dati sono processi diversi. Una reimpostazione della password può ripristinare l’accesso all’account, ma rendere inizialmente inaccessibili le precedenti chiavi private. Proton Drive mostra quindi i dati precedenti come bloccati.

Il [ripristino dei dati Drive](https://proton.me/support/drive-data-recovery) ufficiale indica password precedente, Recovery Phrase o Recovery File come modalità di sblocco. Nuovi upload dopo il reset possono essere creati con nuove chiavi, mentre il patrimonio precedente resta bloccato; foto e album possono richiedere un ulteriore passaggio di unione.

Un runbook organizzativo documenta quindi:

- chi è autorizzato a avviare una reimpostazione della password,
- quali metodi di recovery sono configurati e testati in precedenza,
- dove Recovery Phrase o File sono conservati offline e protetti dall’accesso,
- come riconoscere stati delle chiavi vecchi e nuovi,
- come verificare Shares e membri dopo il recovery,
- quando un Drive bloccato non deve in alcun caso essere eliminato.

«Unlock later» conserva la possibilità di un recovery successivo. «Delete locked drive» è un’eliminazione irreversibile dei dati e non deve rientrare in una procedura standard non confermata.

Se identità, chiavi e stato locale sono plausibili, segue il percorso di rete. Proxy, TLS, endpoint API e trasferimenti bloccati devono essere verificati separatamente dal modello crittografico dei dati.

## Percorso di rete e diagnostica del trasporto

HTTPS protegge il trasporto fra client e API Proton o endpoint di storage. Non sostituisce la crittografia end-to-end, ma protegge metadati di sessione e operativi, identità dell’endpoint e disponibilità. Proxy, filtri DNS, TLS Inspection, VPN, MTU o connessioni instabili possono compromettere in modo diverso le richieste del control plane e i trasferimenti dei blocchi.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Netzwerkdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName drive.proton.me
Test-NetConnection drive.proton.me -Port 443
curl.exe -sS -D - -o NUL https://drive.proton.me/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">getent ahosts drive.proton.me
curl -sS -D - -o /dev/null https://drive.proton.me/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect drive.proton.me:443 -servername drive.proton.me -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) e [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) verificano la risoluzione dei nomi; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) TCP 443; [`curl`](https://curl.se/docs/manpage.html) HTTP e redirect; [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) o [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) le connessioni attive; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) TLS e SNI. Il test contro il dominio web non prova che tutti gli endpoint API/blocchi siano raggiungibili; i log del client forniscono la destinazione e la classe di richiesta effettivamente fallite.

## Osservabilità e catena diagnostica

Una diagnostica efficace parte dall’oggetto applicativo:

1. Determinare account, Share, percorso logico e client interessato.
2. Distinguere lo stato locale: segnaposto, idratato, fissato, modificato, bloccato o difettoso.
3. Controllare vista attività e log per riferimenti a Node, Revision e Transfer.
4. Verificare spazio libero locale e quota cloud.
5. Verificare compatibilità di nomi e percorsi.
6. Verificare DNS, TCP, TLS e proxy/VPN tramite l’endpoint effettivo.
7. Trattare gli avvisi di firma, decifratura e chiavi separatamente dagli errori di trasporto.
8. Elencare o scaricare il file con la CLI ufficiale o un secondo client.
9. Confrontare hash del download e revisione attesa.
10. Solo dopo considerare pause/ripresa, riautenticazione, aggiornamento o intervento sulla cache.

Gli [errori di sincronizzazione Windows](https://proton.me/support/proton-drive-windows-common-issues) citano avanzamento, file aperti, altri client cloud, pausa/ripresa, regole sui nomi e rete. Nell’operatività amministrativa, queste guide utente vengono integrate con inventario degli oggetti e verifica end-to-end. La metodologia generale è disponibile in [Troubleshooting](/kb/troubleshooting).

## Modello di prestazioni e capacità

I costi non derivano solo dai byte trasferiti. Molti file piccoli generano lavoro per chiavi, metadati, API e firme per ciascun Node. Alberi profondi richiedono la ripetuta risoluzione della gerarchia. Un file di grandi dimensioni, invece, può essere elaborato in modo più efficiente in blocchi paralleli. Nella guida agli errori Windows, Proton segnala che molti file piccoli o profondamente annidati possono sincronizzarsi più lentamente di un singolo file della stessa dimensione.

La pianificazione della capacità considera:

- quota cloud incluse versioni e Cestino,
- stati offline locali e cache,
- spazio temporaneo per upload, download e copie in conflitto,
- CPU per crittografia e hashing,
- numero di file e profondità dell’albero,
- ritardo degli eventi dopo periodi offline,
- comportamento Fair Use/di limitazione per trasferimenti automatizzati.

L’introduzione alla CLI avverte di non caricare ripetutamente per intero dati invariati e fa riferimento al Fair Use ([Proton Drive CLI](https://proton.me/blog/proton-drive-cli)). Un processo pianificato mantiene quindi un proprio manifest della sorgente oppure verifica lo stato di destinazione, anziché trattare ogni esecuzione come un upload completo.

## Operatività con Windows, macOS e Linux

Windows e macOS offrono integrazione con il file manager e sincronizzazione in background. Le app mobili gestiscono upload, download, oggetti offline e foto, ma non costituiscono un sostituto identico di un servizio file system desktop. La CLI ufficiale rende scriptabili le operazioni su file e condivisioni su tutti e tre i sistemi operativi desktop.

Nel giugno 2026, Proton ha pubblicato la CLI e ha definito un’applicazione desktop Linux completa con sincronizzazione in background come progetto separato, ancora in corso ([annuncio della CLI](https://proton.me/blog/proton-drive-cli)). Questa distinzione storica resta importante: anche se il portafoglio di piattaforme si evolve, **operazione CLI**, **mount** e **motore di sincronizzazione bidirezionale** non devono essere equiparati. Per il rollout specifico, la [pagina delle note di rilascio](https://proton.me/support/drive-release-notes) viene consultata come fonte dipendente dal tempo.

Nei processi server, l’autenticazione browser viene eseguita in modo controllato e il token di sessione risultante viene conservato nel secret store del sistema operativo di un account di servizio dedicato. Un processo headless non riceve la home directory personale di un amministratore. Exit code, risultato JSON, manifest della sorgente e verifica della destinazione vengono conservati; segreti e link di condivisione vengono rimossi dai log.

Versioni e Cestino proteggono da singoli errori operativi, ma non sostituiscono un recovery indipendente. Il runbook deve trattare congiuntamente account, accesso alle chiavi, dati locali ed esportazione verificata.

## Runbook di backup e recovery

Proton Drive può essere la destinazione di una copia di backup, ma non sostituisce una progettazione di recovery. Un runbook affidabile comprende:

1. Definire l’insieme sorgente e la Source of Truth.
2. Idratare completamente i dati locali on-demand prima di un backup basato su file oppure esportarli direttamente tramite CLI.
3. Creare un manifest dei file con percorso, dimensione e hash.
4. Eseguire l’upload con regola esplicita per conflitti ed eliminazioni.
5. Archiviare exit code della CLI ed elenco della destinazione.
6. Eseguire un download a campione o completo in un percorso isolato.
7. Confrontare hash e revisioni attese.
8. Documentare condivisioni, versionamento, Cestino e conservazione.
9. Testare separatamente il recovery dell’account Proton e dei dati.
10. Conservare una seconda copia indipendente al di fuori dello stesso dominio di account e provider.

L’RPO si riferisce all’ultima revisione dimostrabilmente sottoposta a commit e verificata in Proton, non al momento della scrittura del file locale. L’RTO comprende accesso all’account, sblocco delle chiavi, listing, download, decifratura, larghezza di banda e ripristino della destinazione.

## Storia tecnica

Proton ha sviluppato Drive come complemento dell’ecosistema Proton cifrato. Il modello di sicurezza pubblicato nel 2020 ha stabilito storage centrale con gerarchia di chiavi OpenPGP lato client, nomi cifrati, alberi di Nodes, Shares, blocchi firmati e condivisioni tramite Link. I client web e mobili hanno preceduto i client di sincronizzazione desktop; Version History è stata introdotta nel 2023 come percorso di revisione separato ([introduzione della cronologia delle versioni](https://proton.me/blog/drive-version-history)).

Windows e macOS hanno aggiunto sincronizzazione in background e file on-demand con integrazione Explorer/Finder specifica per piattaforma. Le implementazioni separate hanno portato gradualmente al Drive SDK condiviso. All’inizio del 2026, Proton ha descritto il rollout dell’SDK su web, Windows, macOS, Android e iOS ([sviluppo SDK](https://proton.me/blog/drive-sdk-january-2026)).

Nel giugno 2026 sono seguiti due passaggi tecnici: una migrazione crittografica condivisa a v6 PKESK, v2 SEIPD e AES-GCM per nuovi oggetti file, nonché la CLI basata su SDK per Windows, macOS e Linux. Ciò ha reso possibile un accesso ufficialmente scriptabile senza dover ricostruire la logica crittografica del client. L’architettura resta tuttavia incentrata sul client: i server memorizzano e coordinano, i client autorizzati decifrano, firmano e riconciliano.

## Checklist per amministratori

Con lo storage cifrato lato client, modello dei dati, chiavi, sincronizzazione e backup indipendente devono essere compresi congiuntamente. La checklist sintetizza questi punti per l’approvazione operativa.

- **Modello dei dati:** distinguere Volume, Share, Node, Link, Revision e Block.
- **Crittografia:** documentare il contesto di chiavi di indirizzo, Share, Node e Content Key nonché lo schema dei pacchetti.
- **Metadati:** separare nel modello di minaccia i campi E2EE da quelli operativi leggibili dal server.
- **Client:** inventariare piattaforma, tecnologia, SDK, integrazione file system, firma binaria e percorso dei log.
- **Sincronizzazione:** monitorare stato locale, cursore degli eventi, copie in conflitto, nomi e trasferimenti in sospeso.
- **CLI:** non trattare un’operazione esplicita sui file come motore permanente di sincronizzazione o backup.
- **Condivisione:** verificare proprietario, membri, ruoli, delega Editor, Public Link, password e scadenza.
- **Recovery:** testare separatamente password dell’account, sblocco dei dati, Recovery Phrase/File e accesso Share.
- **Backup:** predisporre copia indipendente, manifest, versionamento e download di ripristino.
- **Collaudo:** decifrare un file su un secondo client o tramite download isolato e aprirlo con verifica di firma e hash.

## Fonti

- [NIST SP 800-34 Rev. 1 – Guida alla pianificazione di continuità](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [Proton Drive – Panoramica del supporto](https://proton.me/support/drive)
- [Proton Drive – CLI](https://proton.me/support/drive-cli)
- [Proton – Storage centralizzato e decentralizzato](https://proton.me/blog/centalized-vs-p2p-protondrive)
- [Proton Drive – Modello di sicurezza](https://proton.me/blog/protondrive-security)
- [Proton Drive – Aggiornamento crittografico](https://proton.me/blog/drive-cryptography-update)
- [Proton Drive – Gestione delle versioni](https://proton.me/support/version-history)
- [Proton Drive – Eliminare e ripristinare file sincronizzati](https://proton.me/support/proton-drive-delete-restore-synced-files)
- [RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [Proton Drive – Gestione delle firme](https://proton.me/support/drive-signature-management)
- [Proton Drive – Download e streaming web](https://proton.me/blog/proton-drive-web-encryption-technique)
- [Proton Drive – Repository SDK](https://github.com/ProtonDriveApps/sdk)
- [npm – @protontech/drive-sdk](https://www.npmjs.com/package/@protontech/drive-sdk)
- [GitHub – ProtonDriveApps](https://github.com/ProtonDriveApps)
- [Windows](https://github.com/ProtonDriveApps/windows-drive)
- [macOS](https://github.com/ProtonDriveApps/mac-drive)
- [Proton Drive – Sincronizzazione Windows On-Demand](https://proton.me/support/proton-drive-windows-on-demand-sync)
- [Proton Drive – Guida macOS](https://proton.me/support/drive-macos-guide)
- [Proton Drive – Problemi comuni di sincronizzazione Windows](https://proton.me/support/proton-drive-windows-common-issues)
- [Proton Drive – Shared with me in Windows](https://proton.me/support/shared-with-me-windows)
- [Microsoft – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – utility sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html)
- [Proton Drive – Log](https://proton.me/support/proton-drive-logs)
- [Proton – Introduzione alla Drive CLI](https://proton.me/blog/proton-drive-cli)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [jq – Manuale](https://jqlang.org/manual/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [Microsoft – New-Item](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [GNU Coreutils – mkdir](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html)
- [GNU Diffutils – diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [Proton Drive – Gestione degli accessi](https://proton.me/support/drive-manage-access-shared-files)
- [Proton Drive – Protezione con password e data di scadenza](https://proton.me/support/password-protect-files-proton-drive)
- [Proton Drive – Modello di minaccia](https://proton.me/blog/proton-drive-threat-model)
- [Proton Drive – Rilevamento di file dannosi](https://proton.me/support/proton-drive-malware-protection)
- [Proton Drive – Ripristino dei dati](https://proton.me/support/drive-data-recovery)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [curl – Manuale](https://curl.se/docs/manpage.html)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Proton Drive – Note di rilascio](https://proton.me/support/drive-release-notes)
- [Proton Drive – Introduzione alla cronologia delle versioni](https://proton.me/blog/drive-version-history)
- [Proton Drive – Sviluppo SDK 2026](https://proton.me/blog/drive-sdk-january-2026)
