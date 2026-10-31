---
title: "Proteggere Midea PortaSplit in Home Assistant: token, key e rete domestica"
navTitle: "Proteggere PortaSplit"
description: "Il token e il key della PortaSplit provengono dal cloud Midea e non scadono mai. Ecco come proteggere questi valori, isolare il dispositivo nella rete domestica e mantenere Home Assistant, l'integrazione e il firmware aggiornati in modo controllato."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "14 min di lettura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-v2-cloud-api-portasplit-home-assistant
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard.png"
slug: "midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali"
translationOf: "midea-portasplit-home-assistant-absichern"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: d9e59ab5647df261814123d662df93c68c9f6cd35c20cec518de0a9cf54d6e22
translatedAt: 2026-10-31T03:38:26.681Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali
---

<aside class="article-update">
  <p class="article-update__label">Cosa dovrebbero fare ora i proprietari di PortaSplit</p>
  <p>Durante la configurazione, Home Assistant ottiene il token e il key della PortaSplit tramite interfacce cloud private. Il progetto Midea AC LAN avverte dal 19 maggio 2025 di possibili modifiche; non è documentata una data di disattivazione da parte del produttore. Per i proprietari ciò significa:</p>
  <ol>
    <li><strong>Salvare in modo cifrato token, key e configurazione.</strong> Se in seguito il recupero non dovesse più funzionare, il backup sarà l'unico modo per ripristinare tutto.</li>
    <li><strong>Non annullare l'associazione senza necessità.</strong> Il ripristino delle impostazioni di fabbrica, la rimozione dall'account Midea o la sostituzione del modulo Wi-Fi impongono di ottenere nuovamente il token.</li>
    <li><strong>Isolare la PortaSplit nella rete domestica.</strong> Nessun port forwarding, VLAN IoT dedicata, accesso consentito solo da Home Assistant.</li>
  </ol>
</aside>

Il controllo locale della Midea PortaSplit si basa su due valori specifici del dispositivo: token e key. Autenticano la connessione tra Home Assistant e il dispositivo e al momento possono essere ottenuti solo tramite il cloud Midea. Ne derivano due compiti: salvare i valori in modo che una nuova configurazione resti possibile senza cloud, e gestire il dispositivo e Home Assistant in modo che, anche in caso di incidente, i valori causino danni limitati.

La serie è composta da tre parti: [la parte 1](/blog/midea-portasplit-home-assistant) descrive la configurazione fino alla dashboard, questa parte tratta la protezione, [la parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) illustra il contesto degli avvisi sull'API cloud.

![Dashboard di Home Assistant della Midea PortaSplit in modalità raffreddamento: indicatori in alto, termostato impostato a 22 °C, grafici di temperatura ambiente, assorbimento di potenza, energia giornaliera, frequenza del compressore, funzionamento del compressore e velocità della ventola, seguiti da valori tecnici e stato.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Da dove provengono token e key

Nei dispositivi con protocollo V3, la PortaSplit accetta comandi locali solo con token e key. I valori non vengono generati dal dispositivo, bensì dal cloud Midea; anche l'app ufficiale li ottiene da lì. Le integrazioni della community hanno reimplementato questa chiamata cloud: effettuano l'accesso agli stessi endpoint dell'app, ricevono token e key e li memorizzano entrambi localmente. Per il funzionamento quotidiano non è più necessaria una connessione cloud.

Non esiste un meccanismo di pairing locale documentato che fornisca i valori senza cloud. In teoria potrebbero essere estratti dall'app, ad esempio tramite reverse engineering o strumentazione a runtime; per il singolo utente è complesso e non sostituisce il recupero dal cloud. Se l'endpoint viene meno, viene meno anche la possibilità di ottenerli.

Il progetto `Midea AC LAN` avverte nel suo README che Midea potrebbe chiudere gradualmente in futuro le interfacce token. Dopo la revisione del README del 3 settembre 2026, l'integrazione ottiene token e key tramite i cloud Meiju e SmartHome. Secondo il README, ciò funziona solo con un account personale a cui il dispositivo è associato; al momento non è possibile recuperare token tramite NetHome Plus. Dal 9 settembre 2026, il README segnala inoltre che un account SmartHome migrato da un'altra app non fornisce token; è necessario un account appena registrato con dispositivi aggiunti nuovamente. I dispositivi già configurati continuano a funzionare localmente; sarebbero interessati i nuovi dispositivi e le nuove configurazioni. Non si tratta di una roadmap vincolante di Midea. Nel giugno 2026 è inoltre emerso che l'API token SmartHome, apparentemente chiusa, continuava a funzionare; la richiesta della libreria della community era semplicemente incompleta. L'inquadramento dell'avviso e delle varie denominazioni «V2» è riportato nella [parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Cosa consentono token e key

Token e key non hanno una data di scadenza. Secondo `Midea AC LAN`, in origine la comunicazione client era considerata sufficientemente protetta, per cui il cloud emetteva token senza scadenza. Di per sé non è una vulnerabilità; diventa problematico se i valori finiscono in log o backup non protetti, vengono ceduti a terzi o non possono essere né revocati né ruotati.

Chi possiede token e key e può raggiungere il dispositivo in rete può autenticarsi presso la PortaSplit, leggere informazioni di stato, accenderla e spegnerla, cambiare modalità operative e modificare la temperatura impostata. I soli valori non consentono un attacco da Internet; l'aggressore necessita inoltre di una connessione di rete al dispositivo. Token e key vanno quindi trattati come una password e la rete dovrebbe consentire questa connessione possibilmente solo a Home Assistant.

L'integrazione della community non attacca il condizionatore. Implementa un protocollo proprietario ricostruito tramite reverse engineering. Il rischio deriva dal fatto che segreti di lunga durata vengono memorizzati al di fuori dell'app prevista.

## Proteggere token, key e configurazione

Il salvataggio di token, key e configurazione è il più importante passaggio una tantum: una volta chiuse le interfacce cloud per i token, un backup sarà l'unico modo per effettuare una nuova configurazione. `Midea AC LAN` salva, dopo una configurazione riuscita dei dispositivi V3, un file di configurazione JSON. Il percorso documentato è:

```text
/config/.storage/midea_ac_lan/
```

Il file ha come nome la ID del dispositivo:

```text
<device-id>.json
```

Questo file non è una normale nota di testo. Può contenere ID dispositivo, numero di serie, indirizzo IP, token, key, informazioni sul protocollo nonché parametri cloud e del dispositivo. Di conseguenza:

- Non caricarlo in un repository GitHub pubblico.
- Non pubblicarlo nei forum.
- Non condividerlo come screenshot non oscurato.
- Non inviarlo tramite e-mail non cifrata.

Nemmeno un repository Git privato è automaticamente il luogo giusto, poiché i segreti restano nella cronologia Git anche se in seguito vengono rimossi dal file corrente. Sono più adatti un backup cifrato, un gestore di password con allegato di file, un backup NAS cifrato, un supporto offline cifrato o un archivio cifrato con password salvata separatamente.

Per il backup tramite il terminale di Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visualizzare il file:

```bash
cat <device-id>.json
```

Per la copia, il file non dovrebbe essere trasferito tramite un servizio web pubblico. È preferibile un archivio cifrato, successivamente trasferito in un backup cifrato:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

I file in `.storage` non devono essere modificati manualmente. Lo sviluppatore raccomanda espressamente di non eliminare né modificare direttamente il file JSON in caso di problemi, ma di rinominarlo e salvarlo prima di apportare modifiche.

Un backup completo di Home Assistant contiene anch'esso questi file. Una copia separata resta comunque utile, poiché i backup di Home Assistant possono danneggiarsi, un ripristino può sovrascrivere l'integrazione, il file potrebbe essere necessario specificamente per una futura nuova configurazione e un backup non dovrebbe mai trovarsi solo sullo stesso sistema.

### Rimuovere i segreti da un repository Git pubblicato

Se un file JSON è stato pubblicato accidentalmente su GitHub, non basta cancellarlo normalmente e creare un nuovo commit. Il file resta accessibile nella cronologia Git. Sono necessari almeno questi passaggi:

1. Rendere subito privato il repository, se possibile.
2. Rimuovere il file dall'intera cronologia Git.
3. Considerare cache e fork di GitHub.
4. Trattare il token come compromesso.
5. Rimuovere il dispositivo dall'account Midea e associarlo di nuovo, se ciò genera nuove chiavi.
6. Configurare nuovamente l'integrazione Home Assistant.
7. Cambiare la password dell'account Midea, se sono state coinvolte anche le credenziali.

Il fatto che il nuovo pairing generi effettivamente un nuovo token varia in base al dispositivo e all'architettura cloud. Non si dovrebbe fare affidamento sul fatto che cambiare la password dell'account invalidi automaticamente il token locale del dispositivo.

## Isolare la PortaSplit nella rete

### Nessun port forwarding alla PortaSplit

L'errore evitabile più comune sarebbe rendere la porta locale del dispositivo direttamente raggiungibile da Internet. Una regola di questo tipo sarebbe pericolosa:

```text
Internet → TCP 6444 → PortaSplit
```

Non c'è alcun buon motivo per rendere la PortaSplit direttamente raggiungibile da Internet. Home Assistant si trova già nella rete locale e funge da istanza di controllo. Il router non dovrebbe avere alcun port forwarding verso la PortaSplit, dovrebbe limitare o disattivare UPnP quando possibile, bloccare per impostazione predefinita le connessioni in ingresso e non utilizzare un'esposizione DMZ per il dispositivo.

### VLAN IoT dedicata

La migliore architettura di rete è una rete IoT separata:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit si trova nella VLAN IoT. Home Assistant può accedere specificamente al dispositivo, ma la PortaSplit non può accedere liberamente a PC, NAS e altri sistemi interni. Una possibile logica firewall:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la prima configurazione, il dispositivo necessita di accesso a Internet verso il cloud Midea. Dopo una configurazione locale riuscita, è possibile verificare se l'accesso a Internet in uscita può essere bloccato. Non si dovrebbe però impostare subito un blocco definitivo. Occorre prima verificare che il controllo locale continui a funzionare, che il dispositivo resti raggiungibile dopo un riavvio, che superi un riavvio del router, che risponda ancora dopo diversi giorni, che l'app MSmartHome sia ancora necessaria e che gli aggiornamenti firmware siano ancora disponibili. Chi desidera continuare a usare il cloud e gli aggiornamenti firmware può consentire temporaneamente l'accesso a Internet in uscita e bloccarlo nuovamente in seguito.

### La segmentazione di rete può impedire il discovery

La ricerca automatica dei dispositivi si basa spesso su traffico broadcast o multicast, che normalmente non viene instradato oltre i confini delle VLAN. Home Assistant potrebbe quindi non trovare automaticamente la PortaSplit, anche se sarebbe consentita una normale connessione IP.

In tal caso può essere utile configurare temporaneamente la PortaSplit nella stessa VLAN di Home Assistant, indicare manualmente l'IP del dispositivo, utilizzare un'idonea funzione di relay broadcast o definire regole firewall specifiche dopo la configurazione. Dal punto di vista della sicurezza, la configurazione manuale è spesso persino la variante migliore, perché non richiede di consentire traffico broadcast aggiuntivo tra le reti.

### Assegnazione DHCP statica

Alla PortaSplit dovrebbe essere assegnata una prenotazione DHCP fissa nel router:

```text
PortaSplit → 192.168.30.25
```

Una prenotazione DHCP è in genere preferibile a un IP statico impostato sul dispositivo. Home Assistant trova il dispositivo in modo affidabile, le regole firewall possono essere limitate a un indirizzo fisso, l'analisi degli errori diventa più semplice e l'assegnazione resta stabile dopo il riavvio del router o del dispositivo. Una regola firewall può quindi essere formulata in modo molto restrittivo:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

La porta effettivamente necessaria va verificata in base all'integrazione e al proprio dispositivo.

## Proteggere Home Assistant e le integrazioni

### Home Assistant come ancoraggio centrale di fiducia

Chi controlla localmente la PortaSplit trasferisce in parte la fiducia dal cloud Midea a Home Assistant. Se Home Assistant viene compromesso, un aggressore potrebbe controllare non solo il condizionatore, ma l'intera smart home.

Home Assistant dovrebbe quindi essere aggiornato regolarmente, non essere pubblicato tramite port forwarding non protetto, essere protetto con una password forte e univoca, utilizzare l'autenticazione a più fattori, creare backup cifrati, contenere solo gli add-on necessari e non consentire accesso SSH non necessario da Internet. Per l'accesso remoto, VPN, Home Assistant Cloud o un reverse proxy configurato correttamente sono opzioni migliori di un semplice port forwarding sulla porta 8123.

### HACS e il rischio della supply chain

`Midea Smart AC` e `Midea AC LAN` sono Custom Integrations. Vengono eseguite all'interno di Home Assistant e hanno quindi accesso esteso al suo ambiente di runtime. Un'integrazione malevola o compromessa potrebbe teoricamente leggere dati di configurazione, estrarre segreti, stabilire connessioni di rete, analizzare dispositivi nella rete locale, leggere gli stati di altre entità, trasferire dati a sistemi esterni e compromettere la disponibilità di Home Assistant.

Ciò non significa che le integrazioni citate siano malevole. Entrambi i progetti sono pubblicamente consultabili, sviluppati attivamente e hanno una community visibile. Tuttavia, l'open source non è una garanzia automatica di sicurezza. Prima dell'installazione vale almeno la pena verificare se il repository viene mantenuto attivamente, se ci sono release regolari, quante persone contribuiscono al codice, se esistono issue di sicurezza aperte, se di recente sono cambiati maintainer o proprietario del repository, se HACS punta al repository previsto e se un aggiornamento contiene modifiche insolitamente grandi o inspiegabili.

Gli aggiornamenti non dovrebbero essere installati alla cieca subito dopo la pubblicazione. Soprattutto nei sistemi smart home critici per la sicurezza, è opportuno attendere alcuni giorni e verificare note di rilascio e problemi segnalati.

### I log di debug contengono dati sensibili

In caso di problemi, i progetti open source richiedono spesso log di debug. La documentazione di `Midea AC LAN` mostra come attivare il logging per i due componenti rilevanti:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Successivamente, i log possono essere scaricati tramite Impostazioni, Sistema e Registri. A seconda dell'integrazione e del tipo di errore, tali log possono contenere indirizzi IP locali, ID dispositivo, numero di serie, identificativo del modello, risposte cloud, informazioni sull'account, token o parti di essi, pacchetti di rete nonché timestamp e comportamenti d'uso. Prima di caricarli in una issue pubblica su GitHub, vanno quindi controllati e i valori sensibili devono essere oscurati.

Al termine della ricerca del problema, il logging di debug va nuovamente rimosso. Il logging di debug attivo in modo permanente non aumenta solo l'uso di spazio, ma anche la quantità di informazioni sensibili nei backup.

## Cloud e firmware

### Proteggere l'account cloud

Finché il cloud Midea viene utilizzato per la configurazione o il controllo tramite app, anche l'account Midea resta parte del modello di sicurezza. Sono necessari una password univoca non condivisa con altri servizi, un gestore di password, l'autenticazione a più fattori se disponibile, la rimozione di vecchi smartphone e sessioni, la rinuncia ad account condivisi e un controllo regolare dei dispositivi registrati nell'account.

Se l'integrazione Home Assistant richiede nome utente e password durante la configurazione, occorre verificare se le credenziali sono utilizzate solo per il recupero una tantum del token o vengono memorizzate permanentemente. Gli sviluppatori di `Midea Smart AC` scrivono che i dispositivi non vengono collegati ad account di integrazione incorporati dopo la configurazione e che token e key possono essere ottenuti manualmente anche tramite CLI usando il proprio account. Ove possibile, è preferibile il proprio account rispetto ad account collettivi esterni o integrati. Per `Midea AC LAN`, dal settembre 2026 un account Meiju o SmartHome personale, a cui il dispositivo è associato, è un requisito per il recupero del token.

### Bloccare il cloud o no?

Dopo una configurazione riuscita sorge la domanda se l'accesso a Internet della PortaSplit debba essere bloccato completamente. A favore del blocco vi sono minore telemetria, minore dipendenza da servizi esterni, una superficie di attacco più ridotta tramite il cloud del produttore, il fatto che il dispositivo non possa contattare destinazioni esterne arbitrarie e un minore impatto delle modifiche lato cloud.

Contro il blocco vi sono il possibile mancato funzionamento dell'app MSmartHome fuori dalla rete domestica, il mancato download degli aggiornamenti firmware, il possibile malfunzionamento di funzioni orarie o cloud, una nuova autenticazione o un ripristino più difficili e il fatto che alcuni dispositivi possano reagire in modo inatteso dopo un lungo periodo offline.

Una sequenza pragmatica: configurare normalmente il dispositivo, testare Home Assistant e l'app, salvare token e configurazione, bloccare l'accesso a Internet, riavviare il dispositivo e Home Assistant, osservare per diversi giorni e, se necessario, consentire nuovamente l'accesso a Internet solo temporaneamente.

### Aggiornamenti firmware: vantaggio per la sicurezza o rischio per l'integrazione?

Gli aggiornamenti firmware rappresentano un dilemma per i dispositivi IoT. Possono correggere vulnerabilità note, migliorare la stabilità, modernizzare i meccanismi di sicurezza e aggiungere nuove funzioni. Possono però anche modificare interfacce locali, interrompere integrazioni basate sul reverse engineering, invalidare token, disattivare l'API locale e introdurre nuove dipendenze dal cloud.

Il firmware PortaSplit distribuito nel gennaio 2026 ha introdotto, ad esempio, una nuova modalità silenziosa per l'unità esterna, che riduce il rumore di circa 6 decibel. Le integrazioni della community hanno dovuto prima ricostruirla e implementarla, come documentato in una issue GitHub dedicata alla PortaSplit.

Ne consegue che gli aggiornamenti firmware non vanno impediti in modo categorico: prima di un aggiornamento occorre verificare se altri utenti di Home Assistant segnalano problemi, salvare prima configurazione e token, creare un backup di Home Assistant e testare completamente il controllo locale dopo l'aggiornamento. Sicurezza non significa «non aggiornare mai». Un firmware obsoleto può essere più pericoloso di un'integrazione temporaneamente incompatibile.

### Cosa dice Midea stessa sulla sicurezza

Midea promuove il proprio ecosistema SmartHome dichiarando l'orientamento a diversi standard di sicurezza e protezione dei dati, tra cui EN 303 645, UK PSTI, NIST, trattamento dei dati conforme al GDPR e requisiti della EU Radio Equipment Directive. Sono segnali positivi, ma non indicano come siano effettivamente implementati ogni singolo firmware PortaSplit, ogni endpoint cloud e ogni API locale. Le dichiarazioni di certificazione e marketing non sostituiscono una verifica tecnica del dispositivo concreto.

Allo stesso modo, sarebbe errato dedurre dall'avviso di un'integrazione della community che la PortaSplit sia generalmente insicura. Il problema descritto riguarda l'architettura di token di lunga durata e il loro utilizzo da parte di client non ufficiali.

## Rischio per scenario

| Scenario | Rischio | Motivazione |
| --- | --- | --- |
| Rete domestica normale senza port forwarding | gestibile | Un aggressore deve prima ottenere accesso al Wi-Fi, a Home Assistant o a un backup. |
| Rete domestica piatta con molti dispositivi IoT non sicuri | medio | Un altro dispositivo IoT compromesso può raggiungere PortaSplit o Home Assistant nella stessa rete. |
| PortaSplit direttamente raggiungibile da Internet | alto | Il dispositivo non dovrebbe mai essere pubblicato tramite port forwarding. |
| Token e key pubblici su GitHub | alto | I segreti sono da considerarsi compromessi; non è garantito che possano essere revocati. |
| VLAN IoT separata, firewall restrittivo, controllo locale | relativamente basso | Anche in presenza di una vulnerabilità nel dispositivo, la libertà di movimento nella rete è fortemente limitata. |

## Checklist

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

La direzione di comunicazione desiderata:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

Gestito in questo modo, il controllo locale è accettabile dal punto di vista della sicurezza: token e key restano segreti e protetti, il dispositivo è raggiungibile solo da Home Assistant e gli aggiornamenti del firmware e dell'integrazione vengono installati in modo controllato.

## Cronologia delle modifiche

- **25 ottobre 2026:** Aggiunto: dalla revisione del README di settembre 2026, Midea AC LAN richiede per il recupero del token un account Meiju o SmartHome personale a cui il dispositivo è associato; NetHome Plus e gli account SmartHome migrati non forniscono token.

## Fonti

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN` con l'«Important Notice» (dal 19 maggio 2025, rivista il 3 e il 9 settembre 2026), la motivazione relativa ai token senza scadenza e la descrizione del recupero dei token basato sul cloud tramite un account Meiju o SmartHome personale.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: recupero cloud di token e key per dispositivi V3, memorizzazione locale dei valori, porta standard 6444.

3.  [midea_ac_lan: indicazioni per debug e configurazione](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): archiviazione della configurazione del dispositivo in `/config/.storage/midea_ac_lan/`, raccomandazione di salvare anziché eliminare il file JSON e configurazione dei logger per i log di debug.

4.  [Issue 779: modalità silenziosa esterna della PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): richiesta di supporto per la modalità silenziosa dell'unità esterna introdotta con l'aggiornamento firmware di gennaio 2026, che riduce il rumore di circa 6 decibel.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): indicazioni del produttore sugli standard di sicurezza e protezione dei dati EN 303 645, PSTI, NIST, GDPR e RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione e gestione di Custom Integrations che non fanno parte di Home Assistant Core.
