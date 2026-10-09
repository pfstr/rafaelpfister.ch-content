---
title: "Proteggere Midea PortaSplit in Home Assistant: token, key e rete domestica"
navTitle: "Proteggere PortaSplit"
description: "Il token e la key di PortaSplit provengono dal cloud Midea e non scadono mai. Ecco come proteggere questi valori, isolare il dispositivo nella rete domestica e mantenere aggiornati in modo controllato Home Assistant, l'integrazione e il firmware."
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
translationSourceHash: c72a9e3147727e1ec8bb37ab078eb3a73c3cc5a4c92a38fc4b3f845966e2f405
translatedAt: 2026-10-09T10:57:34.398Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/it/blog/midea-portasplit-in-home-assistant-perche-token-e-chiave-sono-fondamentali
---

<aside class="article-update">
  <p class="article-update__label">Cosa dovrebbero fare ora i proprietari di PortaSplit</p>
  <p>Durante la configurazione, Home Assistant ottiene il token e la key di PortaSplit tramite interfacce cloud private. Il progetto Midea AC LAN avverte dal 19 maggio 2025 di possibili modifiche; non è documentata una data di dismissione da parte del produttore. Per i proprietari ciò significa:</p>
  <ol>
    <li><strong>Eseguire un backup cifrato di token, key e configurazione.</strong> Se in seguito il recupero non dovesse più funzionare, il backup sarà l'unico modo per ripristinarli.</li>
    <li><strong>Non annullare l'associazione senza necessità.</strong> Il ripristino delle impostazioni di fabbrica, la rimozione dall'account Midea o la sostituzione di un modulo Wi-Fi impongono di ottenere nuovamente il token.</li>
    <li><strong>Isolare PortaSplit nella rete domestica.</strong> Nessun port forwarding, VLAN IoT dedicata, accesso consentito solo a Home Assistant.</li>
  </ol>
</aside>

Il controllo locale di Midea PortaSplit si basa su due valori specifici del dispositivo: token e key. Autenticano la connessione tra Home Assistant e il dispositivo e al momento possono essere ottenuti solo tramite il cloud Midea. Ne derivano due compiti: proteggere i valori in modo che una nuova configurazione resti possibile senza cloud e gestire sia il dispositivo sia Home Assistant in modo che i valori causino pochi danni anche in caso di incidente.

La serie è composta da tre parti: [la parte 1](/blog/midea-portasplit-home-assistant) descrive la configurazione fino alla dashboard, questa parte tratta la protezione, mentre [la parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) spiega il contesto degli avvisi relativi alle API cloud.

![Dashboard di Home Assistant per Midea PortaSplit in modalità raffreddamento: indicatori in alto, termostato a 22 °C, grafici di temperatura ambiente, assorbimento di potenza, energia giornaliera, frequenza del compressore, funzionamento del compressore e velocità della ventola, sotto valori tecnici e stato.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Da dove provengono token e key

Sui dispositivi con protocollo V3, PortaSplit accetta comandi locali solo con token e key. I valori non vengono generati dal dispositivo, ma dal cloud Midea; anche l'app ufficiale li ottiene da lì. Le integrazioni della community hanno reimplementato questa chiamata al cloud: effettuano l'accesso agli stessi endpoint dell'app, ricevono token e key e li salvano localmente. In seguito, per il funzionamento corrente non è più necessaria una connessione cloud.

Non esiste un meccanismo di pairing locale documentato che restituisca i valori senza cloud. In teoria potrebbero essere estratti dall'app, ad esempio tramite reverse engineering o strumentazione a runtime; per il singolo utente è però complesso e non sostituisce il recupero dal cloud. Se l'endpoint viene meno, viene meno anche la possibilità di ottenerli.

Il progetto `Midea AC LAN` avverte nel proprio README che Midea sta chiudendo gradualmente le interfacce dei token; l'integrazione passa quindi da un cloud all'altro. I dispositivi già configurati continuano a funzionare localmente, mentre sarebbero interessati i nuovi dispositivi e le nuove configurazioni. Non si tratta di una roadmap vincolante di Midea. Nel giugno 2026 è inoltre emerso che la presunta API SmartHome Token chiusa continuava a funzionare; la richiesta della libreria della community era semplicemente incompleta. La classificazione dell'avviso e delle diverse denominazioni «V2» è riportata nella [parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Cosa consentono token e key

Token e key non hanno una data di scadenza. Secondo `Midea AC LAN`, la comunicazione client era originariamente considerata sufficientemente protetta, motivo per cui il cloud emetteva token senza scadenza. Questo, di per sé, non è una vulnerabilità; diventa problematico quando i valori finiscono in log o backup non protetti, arrivano a terzi o non possono essere né revocati né ruotati.

Chi possiede token e key e raggiunge il dispositivo in rete può autenticarsi presso PortaSplit, leggere informazioni di stato, accenderla e spegnerla, cambiare modalità operative e modificare la temperatura impostata. I valori da soli non consentono un attacco da Internet; l'aggressore necessita inoltre di una connessione di rete al dispositivo. Token e key devono quindi essere trattati come una password e la rete dovrebbe consentire questa connessione, per quanto possibile, solo a Home Assistant.

L'integrazione della community non attacca il condizionatore. Implementa un protocollo proprietario ricostruito tramite reverse engineering. Il rischio deriva dal fatto che segreti a lunga durata vengono salvati al di fuori dell'app prevista.

## Mettere al sicuro token, key e configurazione

Il backup di token, key e configurazione è il più importante intervento una tantum: una volta chiuse le interfacce cloud dei token, un backup sarà l'unico modo per effettuare una nuova configurazione. `Midea AC LAN` salva, dopo una configurazione riuscita dei dispositivi V3, un file di configurazione JSON. Il percorso documentato è:

```text
/config/.storage/midea_ac_lan/
```

Il file porta come nome la ID del dispositivo:

```text
<device-id>.json
```

Questo file non è una normale nota di testo. Può contenere ID del dispositivo, numero di serie, indirizzo IP, token, key, informazioni sul protocollo nonché parametri cloud e del dispositivo. Di conseguenza:

- Non caricarlo in un repository GitHub pubblico.
- Non pubblicarlo nei forum.
- Non condividerlo come screenshot non oscurato.
- Non inviarlo tramite e-mail non cifrata.

Nemmeno un repository Git privato è automaticamente il luogo di archiviazione corretto, perché i segreti restano nella cronologia Git anche se in seguito vengono eliminati dal file corrente. Sono più adatti un backup cifrato, un password manager con allegato, un backup NAS cifrato, un supporto offline cifrato o un archivio cifrato con password conservata separatamente.

Per il backup tramite il terminale di Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visualizzare il file:

```bash
cat <device-id>.json
```

Per la copia, il file non dovrebbe essere trasferito tramite un servizio web pubblico. Meglio creare un archivio cifrato, da trasferire poi in un backup cifrato:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

I file in `.storage` non dovrebbero essere modificati manualmente. Lo sviluppatore raccomanda esplicitamente, in caso di problemi, di non eliminare né modificare direttamente il file JSON, ma di rinominarlo e salvarne una copia prima delle modifiche.

Un backup completo di Home Assistant contiene anch'esso questi file. Una copia separata è comunque utile, perché i backup di Home Assistant possono danneggiarsi, un ripristino può sovrascrivere l'integrazione, il file potrebbe essere necessario specificamente per una futura nuova configurazione e un backup non dovrebbe mai trovarsi solo sullo stesso sistema.

### Rimuovere segreti da un repository Git pubblicato

Se un file JSON è stato pubblicato accidentalmente su GitHub, non basta una normale eliminazione e un nuovo commit. Il file resta recuperabile nella cronologia Git. Sono necessari almeno questi passaggi:

1. Impostare immediatamente il repository come privato, se possibile.
2. Rimuovere il file dall'intera cronologia Git.
3. Considerare cache e fork di GitHub.
4. Trattare il token come compromesso.
5. Rimuovere il dispositivo dall'account Midea e ricollegarlo, se questo genera nuove chiavi.
6. Configurare nuovamente l'integrazione di Home Assistant.
7. Modificare la password dell'account Midea, se sono state coinvolte anche le credenziali.

Il fatto che un nuovo pairing generi effettivamente un nuovo token varia a seconda del dispositivo e dell'architettura cloud. Non si dovrebbe fare affidamento sul fatto che la modifica della password dell'account renda automaticamente non valido il token locale del dispositivo.

## Isolare PortaSplit nella rete

### Nessun port forwarding verso PortaSplit

L'errore evitabile più comune sarebbe rendere la porta locale del dispositivo raggiungibile direttamente da Internet. Una regola come questa sarebbe pericolosa:

```text
Internet → TCP 6444 → PortaSplit
```

Non c'è alcun buon motivo per rendere PortaSplit raggiungibile direttamente da Internet. Home Assistant si trova già nella rete locale e funge da istanza di controllo. Il router non dovrebbe avere alcun port forwarding verso PortaSplit, UPnP dovrebbe essere limitato o disattivato ove possibile, le connessioni in ingresso dovrebbero essere bloccate per impostazione predefinita e non dovrebbe essere usata alcuna autorizzazione DMZ per il dispositivo.

### VLAN IoT dedicata

La migliore architettura di rete è una rete IoT separata:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit si trova nella VLAN IoT. Home Assistant può accedere in modo mirato al dispositivo, ma PortaSplit non deve poter accedere liberamente a PC, NAS e altri sistemi interni. Una possibile logica firewall:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la prima configurazione, il dispositivo necessita dell'accesso a Internet per il cloud Midea. Dopo una configurazione locale riuscita, si può verificare se l'accesso Internet in uscita può essere bloccato. Non si dovrebbe però impostare subito un blocco definitivo. Occorre prima verificare che il controllo locale continui a funzionare, che il dispositivo resti raggiungibile dopo un riavvio, che superi un riavvio del router, che risponda ancora anche dopo diversi giorni, che l'app MSmartHome sia ancora necessaria e che vengano ancora offerti aggiornamenti del firmware. Chi desidera continuare a utilizzare cloud e aggiornamenti firmware può consentire temporaneamente l'accesso a Internet in uscita e bloccarlo nuovamente in seguito.

### La segmentazione di rete può impedire il discovery

La ricerca automatica dei dispositivi si basa spesso sul traffico broadcast o multicast, che normalmente non viene instradato oltre i confini delle VLAN. Home Assistant potrebbe quindi non trovare automaticamente PortaSplit, anche se fosse consentita una normale connessione IP.

In tal caso può essere utile configurare temporaneamente PortaSplit nella stessa VLAN di Home Assistant, inserire manualmente l'IP del dispositivo, utilizzare un'adeguata funzione di broadcast relay o definire regole firewall mirate dopo la configurazione. Dal punto di vista della sicurezza, la configurazione manuale è spesso persino la scelta migliore, perché non richiede di consentire traffico broadcast aggiuntivo tra le reti.

### Assegnazione DHCP statica

Al router dovrebbe essere assegnata a PortaSplit un'associazione DHCP fissa:

```text
PortaSplit → 192.168.30.25
```

Una prenotazione DHCP è di solito preferibile a un IP statico impostato sul dispositivo. Home Assistant trova il dispositivo in modo affidabile, le regole firewall possono essere limitate a un indirizzo fisso, l'analisi degli errori diventa più semplice e l'associazione resta stabile dopo il riavvio del router o del dispositivo. Una regola firewall può quindi essere formulata in modo molto restrittivo:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

La porta effettivamente necessaria deve essere verificata in base all'integrazione e al proprio dispositivo.

## Proteggere Home Assistant e le integrazioni

### Home Assistant come ancoraggio centrale di fiducia

Chi controlla PortaSplit localmente trasferisce in parte la fiducia dal cloud Midea a Home Assistant. Se Home Assistant viene compromesso, un aggressore potrebbe controllare non solo il condizionatore, ma l'intera smart home.

Home Assistant dovrebbe quindi essere aggiornato regolarmente, non essere pubblicato tramite port forwarding non protetto, essere protetto con una password forte e unica, utilizzare l'autenticazione a più fattori, creare backup cifrati, contenere solo gli add-on necessari e non consentire accesso SSH non necessario da Internet. Per l'accesso remoto, una VPN, Home Assistant Cloud o un reverse proxy configurato correttamente sono opzioni migliori di un semplice port forwarding sulla porta 8123.

### HACS e il rischio della supply chain

`Midea Smart AC` e `Midea AC LAN` sono Custom Integrations. Vengono eseguite all'interno di Home Assistant e ricevono quindi ampio accesso al suo ambiente di runtime. Un'integrazione malevola o compromessa potrebbe teoricamente leggere dati di configurazione, estrarre segreti, stabilire connessioni di rete, scansionare dispositivi nella rete locale, leggere gli stati di altre entità, trasferire dati a sistemi esterni e compromettere la disponibilità di Home Assistant.

Questo non significa che le integrazioni citate siano malevole. Entrambi i progetti sono pubblicamente consultabili, sviluppati attivamente e hanno una community visibile. L'open source non è tuttavia una garanzia automatica di sicurezza. Prima dell'installazione vale almeno la pena verificare se il repository è mantenuto attivamente, se ci sono release regolari, quante persone contribuiscono al codice, se esistono problemi di sicurezza aperti, se di recente sono cambiati maintainer o proprietari del repository, se HACS rimanda al repository previsto e se un aggiornamento contiene modifiche insolitamente ampie o inspiegabili.

Gli aggiornamenti non dovrebbero essere installati ciecamente subito dopo la pubblicazione. Soprattutto per sistemi smart home rilevanti per la sicurezza, è opportuno attendere qualche giorno e controllare le note di rilascio e i problemi segnalati.

### I log di debug contengono dati sensibili

In caso di problemi, i progetti open source richiedono spesso log di debug. La documentazione di `Midea AC LAN` mostra come attivare il logging per i due componenti rilevanti:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Successivamente, i log possono essere scaricati tramite Impostazioni, Sistema e Registri. A seconda dell'integrazione e del caso di errore, tali log possono contenere indirizzi IP locali, ID del dispositivo, numero di serie, identificativo del modello, risposte cloud, informazioni dell'account, token o parti di essi, pacchetti di rete nonché timestamp e comportamento di utilizzo. Prima di caricarli in un issue GitHub pubblico, occorre quindi controllarli e oscurare i valori sensibili.

Al termine della ricerca guasti, il logging di debug va nuovamente rimosso. Un logging di debug attivo in modo permanente non aumenta solo il consumo di spazio, ma amplia anche la quantità di informazioni sensibili nei backup.

## Cloud e firmware

### Proteggere l'account cloud

Finché il cloud Midea viene utilizzato per la configurazione o per il controllo tramite app, anche l'account Midea rimane parte del modello di sicurezza. Sono necessari una password unica, non condivisa con altri servizi, un password manager, l'autenticazione a più fattori se disponibile, la rimozione di vecchi smartphone e sessioni, la rinuncia agli account condivisi e un controllo regolare dei dispositivi registrati nell'account.

Se l'integrazione di Home Assistant richiede nome utente e password durante la configurazione, occorre verificare se le credenziali sono utilizzate solo per il recupero una tantum del token oppure vengono salvate in modo permanente. Gli sviluppatori di `Midea Smart AC` scrivono che dopo la configurazione i dispositivi non sono collegati a account integrati dell'integrazione e che token e key possono essere ottenuti manualmente anche tramite CLI con il proprio account. Ove possibile, il proprio account è preferibile ad account collettivi di terzi o integrati.

### Bloccare il cloud oppure no?

Dopo una configurazione riuscita, si pone la domanda se l'accesso a Internet di PortaSplit debba essere bloccato completamente. A favore di un blocco vi sono meno telemetria, minore dipendenza da servizi esterni, una superficie di attacco più piccola tramite il cloud del produttore, il fatto che il dispositivo non possa contattare obiettivi esterni arbitrari e un minore impatto delle modifiche lato cloud.

Contro vi sono il possibile mancato funzionamento dell'app MSmartHome al di fuori della rete domestica, il mancato download degli aggiornamenti firmware, la possibile indisponibilità di funzioni di orario o cloud, una nuova autenticazione o un ripristino più difficili e reazioni impreviste di alcuni dispositivi dopo un lungo periodo offline.

Una sequenza pragmatica: configurare normalmente il dispositivo, testare Home Assistant e l'app, salvare token e configurazione, bloccare l'accesso a Internet, riavviare il dispositivo e Home Assistant, osservare per diversi giorni e, se necessario, consentire nuovamente l'accesso a Internet solo temporaneamente.

### Aggiornamenti firmware: vantaggio di sicurezza o rischio di integrazione?

Gli aggiornamenti firmware sono un dilemma nei dispositivi IoT. Possono chiudere vulnerabilità note, migliorare la stabilità, modernizzare i meccanismi di sicurezza e introdurre nuove funzioni. Possono però anche modificare le interfacce locali, rompere le integrazioni basate sul reverse engineering, invalidare i token, disattivare l'API locale e introdurre nuove dipendenze dal cloud.

Il firmware PortaSplit distribuito nel gennaio 2026 ha introdotto, ad esempio, una nuova modalità silenziosa per l'unità esterna, che riduce il rumore di circa 6 decibel. Le integrazioni della community hanno dovuto prima analizzarla e implementarla, come documentato in uno specifico issue GitHub per PortaSplit.

Ne consegue che gli aggiornamenti firmware non vanno impediti a priori: prima di un aggiornamento occorre verificare se altri utenti di Home Assistant segnalano problemi, salvare prima configurazione e token, creare un backup di Home Assistant e testare completamente il controllo locale dopo l'aggiornamento. Sicurezza non significa «non aggiornare mai». Un firmware obsoleto può essere più pericoloso di un'integrazione temporaneamente incompatibile.

### Cosa dice Midea stessa sulla sicurezza

Midea promuove il proprio ecosistema SmartHome dichiarando l'orientamento a vari standard di sicurezza e protezione dei dati, tra cui EN 303 645, UK PSTI, NIST, trattamento dei dati conforme al GDPR e requisiti della EU Radio Equipment Directive. Sono segnali positivi, ma non dicono nulla su come siano effettivamente implementati ogni singolo firmware PortaSplit, ogni endpoint cloud e ogni API locale. Le dichiarazioni di certificazione e marketing non sostituiscono un esame tecnico del dispositivo concreto.

Allo stesso modo, sarebbe errato dedurre dall'avviso di un'integrazione della community che PortaSplit sia generalmente insicura. Il problema descritto riguarda l'architettura dei token a lunga durata e il loro impiego da parte di client non ufficiali.

## Rischio per scenario

| Scenario | Rischio | Motivazione |
| --- | --- | --- |
| Rete domestica normale senza port forwarding | gestibile | Un aggressore deve prima ottenere accesso al Wi-Fi, a Home Assistant o a un backup. |
| Rete domestica piatta con molti dispositivi IoT insicuri | medio | Un altro dispositivo IoT compromesso può raggiungere PortaSplit o Home Assistant nella stessa rete. |
| PortaSplit raggiungibile direttamente da Internet | alto | Il dispositivo non dovrebbe mai essere pubblicato tramite port forwarding. |
| Token e key pubblici su GitHub | alto | I segreti sono da considerare compromessi; non è garantito che possano essere revocati. |
| VLAN IoT separata, firewall restrittivo, controllo locale | relativamente basso | Anche in caso di vulnerabilità nel dispositivo, la libertà di movimento nella rete è fortemente limitata. |

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

Gestito in questo modo, il controllo locale è giustificabile dal punto di vista della sicurezza: token e key restano segreti e protetti, il dispositivo è raggiungibile solo da Home Assistant e gli aggiornamenti di firmware e integrazione vengono applicati in modo controllato.

## Fonti

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN` con l'«Important Notice» (dal 19 maggio 2025, aggiornata il 14 luglio 2025), la motivazione relativa ai token senza scadenza e la descrizione dell'ottenimento dei token basato sul cloud.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: ottenimento di token e key basato sul cloud per dispositivi V3, archiviazione locale dei valori, porta standard 6444.

3.  [midea_ac_lan: note su debug e configurazione](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): archiviazione della configurazione del dispositivo in `/config/.storage/midea_ac_lan/`, raccomandazione di salvare anziché eliminare il file JSON e configurazione del logger per i log di debug.

4.  [Issue 779: modalità silenziosa esterna di PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): richiesta di supporto per la modalità silenziosa dell'unità esterna introdotta con l'aggiornamento firmware di gennaio 2026, che riduce il rumore di circa 6 decibel.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): informazioni del produttore sugli standard di sicurezza e protezione dei dati EN 303 645, PSTI, NIST, GDPR e RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione e gestione di Custom Integrations che non fanno parte di Home Assistant Core.
