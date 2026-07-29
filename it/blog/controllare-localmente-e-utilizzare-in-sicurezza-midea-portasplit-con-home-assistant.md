---
title: "Controllare localmente e utilizzare in sicurezza Midea PortaSplit con Home Assistant"
navTitle: "Configurare PortaSplit"
description: "Dall'integrazione community adatta alla VLAN IoT: come configurare PortaSplit, proteggere token e chiave e limitare gli accessi al cloud e alla rete."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "14 min di lettura"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant"
translationOf: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/it/blog/controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant"
---

Midea PortaSplit può essere controllata direttamente nella rete locale tramite Home Assistant dopo la configurazione. A tale scopo, l'integrazione community richiede due credenziali specifiche del dispositivo provenienti dal cloud Midea: token e chiave.

Questo articolo guida nella scelta, configurazione e protezione dell'integrazione. Le soluzioni descritte provengono dalla community e non sono supportate ufficialmente né da Midea né da Home Assistant. Modifiche del firmware o del cloud possono quindi influenzarne il comportamento in qualsiasi momento. Il contesto sull'interfaccia dei token e sull'avviso ambiguo relativo alla disattivazione è disponibile nell'[analisi delle API cloud Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Come funziona il controllo locale

Dopo la configurazione, i comandi di controllo effettivi vengono inviati direttamente da Home Assistant alla PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Un comando di commutazione non deve passare da un server Midea esterno, il tempo di risposta è breve, un guasto del cloud Midea non paralizza necessariamente il controllo locale già configurato e il dispositivo resta sostanzialmente controllabile anche senza accesso a Internet.

Sui dispositivi più recenti con il cosiddetto protocollo V3, PortaSplit non accetta tuttavia comandi locali senza protezione. Home Assistant necessita di due valori specifici del dispositivo, un token e una chiave, che servono all'autenticazione e alla cifratura della connessione locale. Durante la prima configurazione, l'integrazione li recupera una sola volta tramite un'interfaccia cloud Midea e li memorizza poi localmente; per i successivi controlli non è necessaria alcuna connessione cloud.

In forma semplificata, il processo è il seguente:

1. PortaSplit viene collegata a MSmartHome.
2. Home Assistant accede a un cloud Midea.
3. Home Assistant riceve ID dispositivo, token e chiave.
4. Token e chiave vengono salvati localmente.
5. Home Assistant controlla PortaSplit direttamente nella LAN.

## Quale integrazione scegliere

### Midea Smart AC

Il repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> si concentra sui climatizzatori Midea e sui modelli OEM correlati e supporta i tipi di dispositivo `0xAC` e `0xCC`. Offre controllo locale, configurazione grafica, rilevamento automatico, configurazione manuale con token e chiave e una query automatica delle capacità del dispositivo. La modalità “Out Silent Mode” di PortaSplit è supportata esplicitamente.

Come indicazione di compatibilità, il progetto cita tra le altre le app Artic King, Midea Air, NetHome Plus, SmartHome o MSmartHome, Toshiba AC NA e 美的美居. In Europa PortaSplit utilizza tipicamente MSmartHome e rientra quindi in questo ecosistema.

### Midea AC LAN

Il repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> supporta non solo climatizzatori, ma anche numerose altre classi di dispositivi Midea: deumidificatori, ventilatori, purificatori d'aria, lavatrici, asciugatrici, lavastoviglie, apparecchi per l'acqua calda, pompe di calore, frigoriferi e altro ancora, talvolta anche con marchi esterni come Carrier o Electrolux. Offre anch'esso comunicazione locale, rilevamento automatico dei dispositivi e sensori aggiuntivi e, secondo la descrizione del progetto, mantiene una connessione TCP più lunga con il dispositivo per sincronizzare tempestivamente le modifiche di stato. È richiesto almeno Home Assistant 2024.4.1.

Lo svantaggio maggiore al momento è l'avviso dello sviluppatore: le API cloud dei token usate per aggiungere nuovi dispositivi vengono disattivate gradualmente. Di conseguenza, in futuro potrebbe diventare impossibile aggiungere nuovi dispositivi.

### Raccomandazione

Per un'installazione esclusivamente PortaSplit, inizierei con `Midea Smart AC` e terrei `Midea AC LAN` come alternativa. `Midea Smart AC` è più specifica per i climatizzatori e documenta esplicitamente le attuali funzioni di PortaSplit.

Non è sensato utilizzare entrambe le integrazioni contemporaneamente e in modo permanente con lo stesso dispositivo. Più connessioni parallele comportano problemi di stato, traffico di rete superfluo e comportamenti difficili da comprendere.

## Cosa offre l'integrazione

Dopo la configurazione, PortaSplit appare come entità `climate` in Home Assistant. A seconda del firmware e dell'integrazione, sono disponibili tra le altre le seguenti funzioni:

- Accensione e spegnimento
- Impostazione della temperatura desiderata
- Lettura della temperatura ambiente attuale
- Raffreddamento, deumidificazione e sola ventilazione
- Impostazione della velocità della ventola
- Controllo della funzione swing
- Modalità Eco e Boost
- Lettura dell'umidità dell'aria
- Visualizzazione dei codici di errore
- Lettura dei valori di energia e potenza
- Visualizzazione dei valori del compressore
- Attivazione della modalità silenziosa dell'unità esterna

Le entità effettivamente visualizzate dipendono dal modello, dal firmware, dal protocollo utilizzato e dalla rispettiva integrazione. `Midea Smart AC` interroga le capacità segnalate dal dispositivo e nasconde le funzioni non supportate dal modello. `Midea AC LAN` documenta anch'essa numerose entità climatiche, tra cui temperatura, umidità dell'aria, potenza attuale, energia totale, frequenza del compressore, stato della pompa e diverse modalità operative, e cita metodi propri per decodificare i dati energetici per specifici sottotipi di PortaSplit.

Non tutte le misurazioni visualizzate devono essere corrette. In particolare, consumo energetico e potenza vengono trasmessi in formati diversi nei vari modelli Midea. Se Home Assistant mostra valori palesemente errati, di solito va adattato il metodo di decodifica utilizzato e non è il dispositivo a essere difettoso.

## Requisiti

Sono necessari una Midea PortaSplit con funzione Wi-Fi, una rete Wi-Fi a 2,4 GHz, l'app MSmartHome, un account utente Midea, Home Assistant, HACS e l'accesso di rete tra Home Assistant e PortaSplit. PortaSplit deve essere prima collegata normalmente tramite l'app MSmartHome e solo successivamente aggiunta a Home Assistant.

## Passaggio 1: collegare PortaSplit a MSmartHome

1. Installare l'app MSmartHome.
2. Creare un account Midea o effettuare l'accesso.
3. Mettere PortaSplit in modalità di associazione Wi-Fi.
4. Collegare il dispositivo alla rete Wi-Fi a 2,4 GHz.
5. Verificare che PortaSplit possa essere controllata tramite l'app.

Molti dispositivi IoT supportano ancora solo 2,4 GHz. Se il router usa lo stesso SSID per 2,4 e 5 GHz, la configurazione funziona comunque nella maggior parte dei casi. In caso di problemi, può essere utile predisporre temporaneamente una rete Wi-Fi separata a 2,4 GHz.

## Passaggio 2: installare HACS

HACS è il Community Store di Home Assistant. Consente di installare integrazioni community che non fanno parte di Home Assistant Core. Dopo l'installazione di HACS, aprire HACS, passare alle integrazioni, cercare `Midea Smart AC`, scaricare l'integrazione e riavviare Home Assistant. In alternativa, si può cercare `Midea AC LAN`.

HACS semplifica l'installazione e gli aggiornamenti. Tuttavia, non trasforma una Custom Integration in un componente Home Assistant verificato ufficialmente. Questa differenza è rilevante dal punto di vista della sicurezza e verrà trattata più avanti.

## Passaggio 3: aggiungere Midea Smart AC

Dopo il riavvio, andare su Impostazioni, Dispositivi e servizi e Aggiungi integrazione, quindi cercare `Midea Smart AC` e selezionare `Discover devices`. L'integrazione può cercare l'intera rete locale oppure interrogare in modo mirato l'indirizzo IP di PortaSplit.

Se il dispositivo viene trovato, sui dispositivi V3 più recenti l'integrazione richiede regione, account Midea, password e ID dispositivo, oltre al token e alla chiave derivati da tali dati. La regione cloud deve corrispondere all'account utilizzato. In caso di problemi, il progetto raccomanda di provare anche le altre regioni disponibili.

### Configurazione manuale

Se la configurazione automatica non riesce, il dispositivo può essere configurato manualmente. Per `Midea Smart AC` sono necessari i seguenti dati:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

La porta predefinita documentata è:

```text
6444/TCP
```

Per i dispositivi V3, la documentazione indica il token come una stringa esadecimale di 128 caratteri e la chiave come una stringa esadecimale di 64 caratteri. Entrambi i valori sono segreti e devono essere trattati di conseguenza. Chi non desidera ottenere le credenziali tramite Discovery può recuperarle con il proprio account tramite la CLI `msmart-ng`.

## Utilizzare PortaSplit in sicurezza

Chi controlla PortaSplit localmente recupera parte del controllo dal cloud del produttore, ma trasferisce così la responsabilità nella propria rete. I seguenti punti fanno sì che token e chiave possano causare pochi danni anche in caso di incidente e che il dispositivo resti correttamente isolato.

### Token e chiave sono segreti

Token e chiave autenticano la comunicazione locale con il dispositivo e devono essere trattati come una password. Per il funzionamento, soprattutto: non devono finire nei log, in backup non cifrati o in un repository.

### Nessun port forwarding verso PortaSplit

L'errore evitabile più comune sarebbe rendere la porta del dispositivo locale direttamente raggiungibile da Internet. Una regola come questa sarebbe pericolosa:

```text
Internet → TCP 6444 → PortaSplit
```

Non vi è alcun buon motivo per rendere PortaSplit raggiungibile direttamente da Internet. Home Assistant si trova già nella rete locale e funge da istanza di controllo. Il router non dovrebbe avere alcun inoltro di porte verso PortaSplit, dovrebbe limitare o disattivare UPnP quando possibile, bloccare per impostazione predefinita le connessioni in entrata e non usare un'esposizione DMZ per il dispositivo.

### VLAN IoT dedicata

La migliore architettura di rete è una rete IoT separata:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit si trova nella VLAN IoT. Home Assistant può accedere in modo mirato al dispositivo, ma PortaSplit non può accedere arbitrariamente a PC, NAS e altri sistemi interni. Una possibile logica firewall:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la prima configurazione, il dispositivo necessita di accesso a Internet per il cloud Midea. Dopo il completamento della configurazione locale, è possibile verificare se l'accesso a Internet in uscita può essere bloccato. Non andrebbe però impostato subito un blocco definitivo. Occorre prima verificare se il controllo locale continua a funzionare, se il dispositivo resta raggiungibile dopo un riavvio, se supera un riavvio del router, se risponde ancora anche dopo diversi giorni, se l'app MSmartHome continua a essere necessaria e se vengono ancora offerti aggiornamenti firmware. Chi desidera continuare a usare il cloud e gli aggiornamenti firmware può consentire temporaneamente l'accesso a Internet in uscita e bloccarlo nuovamente in seguito.

### La segmentazione della rete può impedire Discovery

La ricerca automatica dei dispositivi si basa spesso sul traffico broadcast o multicast, che normalmente non viene instradato oltre i confini delle VLAN. Home Assistant potrebbe quindi non trovare automaticamente PortaSplit, anche se una normale connessione IP sarebbe consentita.

In tal caso, può essere utile configurare temporaneamente PortaSplit nella stessa VLAN di Home Assistant, specificare manualmente l'IP del dispositivo, utilizzare un'adeguata funzione di relay broadcast oppure definire regole firewall mirate dopo la configurazione. La configurazione manuale è spesso persino la variante migliore dal punto di vista della sicurezza, poiché non richiede di consentire traffico broadcast aggiuntivo tra le reti.

### Assegnazione DHCP statica

Nel router, PortaSplit dovrebbe ricevere un'assegnazione DHCP fissa:

```text
PortaSplit → 192.168.30.25
```

Una prenotazione DHCP è in genere preferibile a un IP statico impostato nel dispositivo. Home Assistant trova il dispositivo in modo affidabile, le regole firewall possono essere limitate a un indirizzo fisso, l'analisi degli errori diventa più semplice e l'assegnazione resta stabile dopo il riavvio del router o del dispositivo. In questo modo una regola firewall può essere formulata in modo molto restrittivo:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

La porta effettivamente necessaria deve essere verificata in base all'integrazione e al proprio dispositivo.

### Home Assistant come pilastro centrale di fiducia

Chi controlla PortaSplit localmente trasferisce in parte la fiducia dal cloud Midea a Home Assistant. Se Home Assistant viene compromesso, un aggressore potrebbe controllare non solo il climatizzatore, ma l'intera smart home.

Home Assistant dovrebbe quindi essere aggiornato regolarmente, non essere esposto tramite un port forwarding non protetto, essere protetto con una password robusta e unica, utilizzare l'autenticazione a più fattori, creare backup cifrati, contenere solo gli add-on necessari e non consentire accesso SSH non necessario da Internet. Per l'accesso remoto, VPN, Home Assistant Cloud o un reverse proxy configurato correttamente sono opzioni migliori di un semplice port forwarding sulla porta 8123.

### HACS e il rischio della supply chain

`Midea Smart AC` e `Midea AC LAN` sono Custom Integrations. Vengono eseguite all'interno di Home Assistant e hanno quindi accesso esteso al suo ambiente di esecuzione. Un'integrazione dannosa o compromessa potrebbe teoricamente leggere dati di configurazione, estrarre segreti, stabilire connessioni di rete, scansionare dispositivi nella rete locale, leggere gli stati di altre entità, trasferire dati a sistemi esterni e compromettere la disponibilità di Home Assistant.

Questo non significa che le integrazioni citate siano dannose. Entrambi i progetti sono pubblicamente consultabili, sviluppati attivamente e dotati di una community visibile. Tuttavia, l'open source non è una garanzia automatica di sicurezza. Prima dell'installazione vale almeno la pena verificare se il repository è gestito attivamente, se vi sono release regolari, quante persone contribuiscono al codice, se esistono problemi di sicurezza aperti, se recentemente sono cambiati maintainer o proprietari del repository, se HACS rimanda al repository previsto e se un aggiornamento contiene modifiche insolitamente grandi o inspiegabili.

Gli aggiornamenti non dovrebbero essere installati alla cieca subito dopo la pubblicazione. Soprattutto per sistemi smart home critici per la sicurezza, è opportuno attendere alcuni giorni e controllare le note di rilascio e i problemi segnalati.

### Proteggere l'account cloud

Finché il cloud Midea viene utilizzato per la configurazione o il controllo tramite app, anche l'account Midea resta parte del modello di sicurezza. Sono necessari una password unica non condivisa con altri servizi, un gestore di password, l'autenticazione a più fattori se disponibile, la rimozione di vecchi smartphone e sessioni, la rinuncia agli account condivisi e un controllo regolare dei dispositivi registrati nell'account.

Se l'integrazione Home Assistant richiede nome utente e password durante la configurazione, è necessario verificare se le credenziali vengono utilizzate solo per il recupero una tantum del token o se restano memorizzate in modo permanente. Gli sviluppatori di `Midea Smart AC` scrivono che, dopo la configurazione, i dispositivi non vengono collegati ad account integrati dell'integrazione e che token e chiave possono essere ottenuti manualmente anche con il proprio account tramite CLI. Dove possibile, è preferibile il proprio account rispetto ad account collettivi esterni o integrati.

### Bloccare il cloud oppure no?

Dopo una configurazione riuscita, si pone la domanda se l'accesso a Internet di PortaSplit debba essere bloccato completamente. A favore del blocco vi sono meno telemetria, minore dipendenza da servizi esterni, una superficie d'attacco più piccola attraverso il cloud del produttore, il fatto che il dispositivo non possa contattare obiettivi esterni arbitrari e un minore impatto delle modifiche lato cloud.

Contro il blocco vi sono il fatto che l'app MSmartHome potrebbe non funzionare più fuori dalla rete domestica, che gli aggiornamenti firmware non vengano scaricati, che possano venire meno funzioni di orario o cloud, che una nuova autenticazione o il ripristino diventino più difficili e che alcuni dispositivi possano reagire in modo imprevisto dopo un lungo periodo offline.

Una sequenza pragmatica: configurare normalmente il dispositivo, testare Home Assistant e l'app, mettere al sicuro token e configurazione, bloccare l'accesso a Internet, riavviare il dispositivo e Home Assistant, osservare per diversi giorni e, se necessario, riabilitare l'accesso a Internet solo temporaneamente.

### Aggiornamenti firmware: vantaggio per la sicurezza o rischio di integrazione?

Gli aggiornamenti firmware sono un dilemma per i dispositivi IoT. Possono chiudere vulnerabilità note, migliorare la stabilità, modernizzare i meccanismi di sicurezza e introdurre nuove funzioni. Possono però anche modificare interfacce locali, interrompere integrazioni basate sul reverse engineering, invalidare token, disattivare l'API locale e introdurre nuove dipendenze dal cloud.

Il firmware PortaSplit distribuito nel gennaio 2026 ha introdotto, ad esempio, una nuova modalità silenziosa per l'unità esterna, che riduce la rumorosità di circa 6 decibel. Le integrazioni community hanno dovuto prima analizzarla e implementarla, come documentato in un apposito issue GitHub per PortaSplit.

Ne consegue: non bloccare gli aggiornamenti firmware in modo assoluto; prima di un aggiornamento verificare se altri utenti Home Assistant segnalano problemi, salvare prima configurazione e token, creare un backup di Home Assistant e testare completamente il controllo locale dopo l'aggiornamento. Sicurezza non significa “non aggiornare mai”. Un firmware obsoleto può essere più pericoloso di un'integrazione temporaneamente incompatibile.

### I log di debug contengono dati sensibili

In caso di problemi, i progetti open source richiedono spesso log di debug. La documentazione di `Midea AC LAN` mostra come attivare il logging per i due componenti rilevanti:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Successivamente, i log possono essere scaricati da Impostazioni, Sistema e Registri. A seconda dell'integrazione e del tipo di errore, tali log possono contenere indirizzi IP locali, ID dispositivo, numero di serie, identificativo del modello, risposte cloud, informazioni sull'account, token o parti di essi, pacchetti di rete nonché timestamp e comportamenti di utilizzo. Prima di caricarli in un issue GitHub pubblico, vanno quindi controllati e i valori sensibili devono essere oscurati.

Al termine della ricerca degli errori, il logging di debug va rimosso nuovamente. Il logging di debug attivo in modo permanente non solo aumenta il consumo di memoria, ma amplia anche la quantità di informazioni sensibili nei backup.

### Cosa dice Midea stessa sulla sicurezza

Midea pubblicizza il proprio ecosistema SmartHome dichiarando di orientarsi a diversi standard di sicurezza e protezione dei dati, tra cui EN 303 645, UK PSTI, NIST, trattamento dei dati conforme al GDPR e requisiti della direttiva UE sulle apparecchiature radio. Sono segnali positivi, ma non affermano come ogni singolo firmware PortaSplit, ogni endpoint cloud e ogni API locale siano effettivamente implementati. Le dichiarazioni di certificazione e marketing non sostituiscono una verifica tecnica del dispositivo concreto.

Sarebbe altrettanto sbagliato dedurre dall'avviso di un'integrazione community che PortaSplit sia generalmente insicura. Il problema descritto riguarda l'architettura dei token a lunga durata e il loro utilizzo da parte di client non ufficiali.

### Rischio per scenario

| Scenario | Rischio | Motivazione |
| --- | --- | --- |
| Rete domestica normale senza port forwarding | gestibile | Un aggressore deve prima ottenere accesso al Wi-Fi, a Home Assistant o a un backup. |
| Rete domestica piatta con molti dispositivi IoT non sicuri | medio | Un altro dispositivo IoT compromesso può raggiungere PortaSplit o Home Assistant nella stessa rete. |
| PortaSplit raggiungibile direttamente da Internet | elevato | Il dispositivo non dovrebbe mai essere esposto tramite port forwarding. |
| Token e chiave pubblici su GitHub | elevato | I segreti vanno considerati compromessi; non è garantito che possano essere revocati. |
| VLAN IoT separata, firewall restrittivo, controllo locale | relativamente basso | Anche in presenza di una vulnerabilità nel dispositivo, la libertà di movimento nella rete è fortemente limitata. |

## Backup della configurazione

Il salvataggio di token, chiave e configurazione è il passaggio una tantum più importante: una volta chiuse le interfacce cloud dei token, un backup è l'unico modo per effettuare una nuova configurazione. `Midea AC LAN` salva, dopo una configurazione riuscita per dispositivi V3, un file di configurazione JSON. Il percorso documentato è:

```text
/config/.storage/midea_ac_lan/
```

Il file usa l'ID dispositivo come nome file:

```text
<device-id>.json
```

Questo file non è una normale nota di testo. Può contenere ID dispositivo, numero di serie, indirizzo IP, token, chiave, informazioni sul protocollo nonché parametri cloud e del dispositivo. Di conseguenza:

- Non caricarlo in un repository GitHub pubblico.
- Non pubblicarlo nei forum.
- Non condividerlo come screenshot non oscurato.
- Non inviarlo tramite e-mail non cifrata.

Nemmeno un repository Git privato è automaticamente il luogo di archiviazione adatto, poiché i segreti restano nella cronologia Git anche se in seguito vengono rimossi dal file corrente. Sono più adatti un backup cifrato, un gestore di password con allegato file, un backup NAS cifrato, un supporto offline cifrato o un archivio cifrato con password memorizzata separatamente.

Per il backup tramite il terminale di Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visualizzare il file:

```bash
cat <id-dispositivo>.json
```

Per la copia, il file non dovrebbe essere trasferito tramite un servizio web pubblico. È preferibile un archivio cifrato, da trasferire poi in un backup cifrato:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

I file in `.storage` non devono essere modificati manualmente. Lo sviluppatore raccomanda espressamente, in caso di problemi, di non eliminare né modificare direttamente il file JSON, ma di rinominarlo e salvarlo prima di apportare modifiche.

Anche un backup completo di Home Assistant contiene questi file. Una copia separata resta comunque utile, perché i backup di Home Assistant possono danneggiarsi, un ripristino può sovrascrivere l'integrazione, il file potrebbe essere necessario specificamente per una futura nuova configurazione e un backup non dovrebbe mai trovarsi solo sullo stesso sistema.

## Rimuovere segreti da un repository Git pubblicato

Se un file JSON è stato pubblicato accidentalmente su GitHub, non basta una normale eliminazione e un nuovo commit. Il file resta recuperabile nella cronologia Git. Sono necessari almeno questi passaggi:

1. Rendere subito privato il repository, se possibile.
2. Rimuovere il file dall'intera cronologia Git.
3. Considerare cache e fork di GitHub.
4. Trattare il token come compromesso.
5. Rimuovere il dispositivo dall'account Midea e collegarlo nuovamente, se ciò genera nuove chiavi.
6. Configurare nuovamente l'integrazione Home Assistant.
7. Cambiare la password dell'account Midea, se sono state coinvolte anche le credenziali di accesso.

Il fatto che la nuova associazione generi effettivamente un nuovo token varia in base al dispositivo e all'architettura cloud. Non bisogna fare affidamento sul fatto che la modifica della password dell'account invalidi automaticamente il token locale del dispositivo.

## Automazioni utili

Dopo un'integrazione riuscita, PortaSplit può essere utilizzata in modo decisamente più intelligente. Gli ID delle entità vanno adattati alla propria installazione.

Raffreddare solo con finestre chiuse:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Accendere in caso di temperatura ambiente elevata:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Preraffreddare prima di andare a dormire:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Spegnere quando non c'è nessuno in casa:

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Configurazione consigliata in sintesi

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

La direzione di comunicazione desiderata è quindi la seguente:

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

## Stato operativo consigliato

Midea PortaSplit può essere integrata sorprendentemente bene in Home Assistant. Dopo una configurazione riuscita, è controllabile localmente e può essere inclusa nelle automazioni, eliminando così per l'uso quotidiano gran parte della dipendenza dal cloud.

Dal punto di vista della sicurezza, l'integrazione è accettabile se vengono rispettate alcune regole di base: nessun port forwarding, mantenere segreti token e chiave, cifrare i backup, controllare i log di debug prima della pubblicazione, proteggere Home Assistant, segmentare i dispositivi IoT, limitare l'accesso a Internet in uscita a quanto necessario e non installare alla cieca aggiornamenti firmware e HACS. Utilizzata in questo modo, PortaSplit resta un climatizzatore potente e diventa al contempo una componente sensatamente integrabile di una smart home controllata localmente.

## Fonti

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: tipi di dispositivo supportati `0xAC` e `0xCC`, PortaSplit con “Out Silent Mode”, utilizzo del cloud per ottenere token e chiave nei dispositivi V3, configurazione manuale e porta predefinita 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN`: classi di dispositivi supportate, connessione TCP più lunga per la sincronizzazione dello stato e versione minima Home Assistant 2024.4.1.

3.  [midea_ac_lan: documentazione delle entità climatiche](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entità e attributi per climatizzatori, tra cui potenza, energia totale, frequenza del compressore e metodi di decodifica per i valori energetici di singoli sottotipi.

4.  [midea_ac_lan: indicazioni su debug e configurazione](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): salvataggio della configurazione del dispositivo in `/config/.storage/midea_ac_lan/`, raccomandazione di salvare anziché eliminare il file JSON e configurazione del logger per i log di debug.

5.  [Issue 779: Out Silent Mode di PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): richiesta di supporto per la modalità silenziosa dell'unità esterna introdotta con l'aggiornamento firmware di gennaio 2026, che riduce la rumorosità di circa 6 decibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): informazioni del produttore sugli standard di sicurezza e protezione dei dati EN 303 645, PSTI, NIST, GDPR e RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione e gestione di Custom Integrations che non fanno parte di Home Assistant Core.
