---
title: "Controllare localmente Midea PortaSplit con Home Assistant e utilizzarla in sicurezza"
navTitle: "Configurare PortaSplit"
description: "Dall’integrazione community più adatta alla VLAN IoT: come configurare PortaSplit, proteggere token e key e limitare gli accessi cloud e di rete."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "14 min di lettura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - serverloser-newsletter-cloudflare-workers-d1
slug: "controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant"
translationOf: "midea-portasplit-home-assistant-einrichten"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: bbe70b67dd255184cf0db69f7308c756937dc961c3c83e152268ee668f93dd07
translatedAt: 2026-09-04T08:33:40.636Z
translationModel: gpt-5.6-terra
url: https://rafaelpfister.ch/it/blog/controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant
---

La Midea PortaSplit può essere controllata direttamente sulla rete locale tramite Home Assistant dopo la configurazione. A questo scopo, l’integrazione community richiede due credenziali specifiche del dispositivo dalla cloud Midea: token e key.

Questo articolo illustra la scelta, la configurazione e la protezione dell’integrazione. Le soluzioni descritte provengono dalla community e non sono supportate ufficialmente né da Midea né da Home Assistant. Modifiche al firmware o alla cloud possono quindi influenzarne il comportamento in qualsiasi momento. Il contesto relativo all’interfaccia token e all’avviso ambiguo di dismissione è disponibile nell’[analisi delle API cloud Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Come funziona il controllo locale

Dopo la configurazione, i comandi di controllo effettivi vengono inviati direttamente da Home Assistant alla PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Un comando non deve passare da un server Midea esterno, il tempo di risposta è breve, un guasto della cloud Midea non interrompe necessariamente il controllo locale già configurato e il dispositivo resta in linea di principio controllabile anche senza accesso a Internet.

Tuttavia, nei dispositivi più recenti con il cosiddetto protocollo V3, PortaSplit non accetta comandi locali senza protezione. Home Assistant necessita di due valori specifici del dispositivo, un token e una key, utilizzati per l’autenticazione e la crittografia della connessione locale. Durante la prima configurazione, l’integrazione li recupera una sola volta tramite un’interfaccia cloud Midea e poi li memorizza localmente; per il controllo successivo non è necessaria alcuna connessione cloud.

In forma semplificata, la procedura è la seguente:

1. PortaSplit viene collegata a MSmartHome.
2. Home Assistant accede a una cloud Midea.
3. Home Assistant riceve ID dispositivo, token e key.
4. Token e key vengono salvati localmente.
5. Home Assistant controlla PortaSplit direttamente nella LAN.

## Quale integrazione scegliere

### Midea Smart AC

Il repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> si concentra sui climatizzatori Midea e sui relativi modelli OEM e supporta i tipi di dispositivo `0xAC` e `0xCC`. Offre controllo locale, configurazione grafica, rilevamento automatico, configurazione manuale con token e key nonché una query automatica delle capacità del dispositivo. La modalità “Out Silent Mode” di PortaSplit è esplicitamente supportata.

Come indicazione di compatibilità, il progetto cita tra le altre le app Artic King, Midea Air, NetHome Plus, SmartHome o MSmartHome, Toshiba AC NA e 美的美居. In Europa PortaSplit utilizza tipicamente MSmartHome e rientra quindi in questo ecosistema.

### Midea AC LAN

Il repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> supporta non solo i climatizzatori, ma numerose altre categorie di dispositivi Midea: deumidificatori, ventilatori, purificatori d’aria, lavatrici, asciugatrici, lavastoviglie, apparecchi per acqua calda, pompe di calore, frigoriferi e altro ancora, in parte anche con marchi terzi come Carrier o Electrolux. Offre anch’esso comunicazione locale, rilevamento automatico dei dispositivi e sensori aggiuntivi e, secondo la descrizione del progetto, mantiene aperta una connessione TCP più lunga con il dispositivo per sincronizzare tempestivamente le variazioni di stato. Richiede almeno Home Assistant 2024.4.1.

Il maggiore svantaggio è attualmente l’avviso dello sviluppatore: le API cloud token utilizzate per aggiungere nuovi dispositivi vengono gradualmente dismesse. L’aggiunta successiva di nuovi dispositivi potrebbe quindi diventare impossibile.

### Raccomandazione

Per un’installazione esclusivamente PortaSplit inizierei con `Midea Smart AC` e terrei presente `Midea AC LAN` come alternativa. `Midea Smart AC` è più mirata ai climatizzatori e documenta esplicitamente le attuali funzioni di PortaSplit.

Non è opportuno utilizzare entrambe le integrazioni contemporaneamente e in modo permanente con lo stesso dispositivo. Più connessioni parallele causano problemi di stato, traffico di rete superfluo e comportamenti difficili da comprendere.

## Cosa offre l’integrazione

Dopo la configurazione, PortaSplit appare come entità `climate` in Home Assistant. A seconda del firmware e dell’integrazione, sono disponibili tra le altre le seguenti funzioni:

- Accensione e spegnimento
- Impostazione della temperatura desiderata
- Lettura della temperatura ambiente attuale
- Raffreddamento, deumidificazione e sola ventilazione
- Impostazione della velocità del ventilatore
- Controllo della funzione swing
- Modalità Eco e Boost
- Lettura dell’umidità dell’aria
- Visualizzazione dei codici di errore
- Lettura dei valori di energia e potenza
- Visualizzazione dei valori del compressore
- Attivazione della modalità silenziosa dell’unità esterna

Le entità effettivamente visualizzate dipendono dal modello, dal firmware, dal protocollo utilizzato e dalla rispettiva integrazione. `Midea Smart AC` interroga le capacità segnalate dal dispositivo e nasconde le funzioni non supportate dal modello. `Midea AC LAN` documenta anch’essa numerose entità climatiche, tra cui temperatura, umidità dell’aria, potenza attuale, energia totale, frequenza del compressore, stato della pompa e varie modalità operative, e menziona metodi specifici per alcuni sottotipi di PortaSplit per decodificare i dati energetici.

Non tutte le misurazioni visualizzate devono essere corrette. In particolare, il consumo energetico e la potenza vengono trasmessi in formati diversi nei vari modelli Midea. Se Home Assistant mostra valori palesemente errati, di solito occorre adattare il metodo di decodifica utilizzato e il dispositivo non è difettoso.

## Requisiti

Sono necessari una Midea PortaSplit con funzione WLAN, una WLAN a 2,4 GHz, l’app MSmartHome, un account utente Midea, Home Assistant, HACS e accesso di rete tra Home Assistant e PortaSplit. PortaSplit deve prima essere collegata normalmente tramite l’app MSmartHome e solo successivamente aggiunta a Home Assistant.

## Passaggio 1: collegare PortaSplit a MSmartHome

1. Installare l’app MSmartHome.
2. Creare un account Midea o effettuare l’accesso.
3. Mettere PortaSplit in modalità di associazione WLAN.
4. Collegare il dispositivo alla WLAN a 2,4 GHz.
5. Verificare che PortaSplit possa essere controllata tramite l’app.

Molti dispositivi IoT supportano ancora solo i 2,4 GHz. Se il router utilizza lo stesso SSID per 2,4 e 5 GHz, la configurazione di solito funziona comunque. In caso di problemi, può essere utile predisporre temporaneamente una WLAN a 2,4 GHz separata.

## Passaggio 2: installare HACS

HACS è il Community Store di Home Assistant. Consente di installare integrazioni della community che non fanno parte di Home Assistant Core. Dopo l’installazione di HACS, aprire HACS, passare alle integrazioni, cercare `Midea Smart AC`, scaricare l’integrazione e riavviare Home Assistant. In alternativa, è possibile cercare `Midea AC LAN`.

HACS semplifica l’installazione e gli aggiornamenti. Tuttavia, non rende una Custom Integration un componente Home Assistant ufficialmente verificato. Questa differenza è fondamentale dal punto di vista della sicurezza e verrà trattata più avanti.

## Passaggio 3: aggiungere Midea Smart AC

Dopo il riavvio, andare su Impostazioni, Dispositivi e servizi, quindi Aggiungi integrazione e cercare `Midea Smart AC`, quindi `Discover devices`. L’integrazione può analizzare l’intera rete locale oppure interrogare in modo mirato l’indirizzo IP di PortaSplit.

Se il dispositivo viene trovato, per i dispositivi V3 più recenti l’integrazione richiede regione, account Midea, password e ID dispositivo, nonché il token e la key derivati da questi dati. La regione cloud deve corrispondere all’account utilizzato. In caso di problemi, il progetto consiglia di provare anche le altre regioni disponibili.

### Configurazione manuale

Se la configurazione automatica fallisce, il dispositivo può essere configurato manualmente. Per `Midea Smart AC` sono necessari i seguenti dati:

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

Per i dispositivi V3, la documentazione indica il token come stringa esadecimale di 128 caratteri e la key come stringa esadecimale di 64 caratteri. Entrambi i valori sono segreti e devono essere trattati di conseguenza. Chi non desidera recuperare le credenziali tramite Discovery può ottenerle con il proprio account tramite la CLI `msmart-ng`.

## Utilizzare PortaSplit in sicurezza

Chi controlla PortaSplit localmente recupera parte del controllo dalla cloud del produttore, ma trasferisce così la responsabilità nella propria rete. I punti seguenti fanno sì che token e key causino danni limitati anche in caso di incidente e che il dispositivo resti correttamente isolato.

### Token e key sono segreti

Token e key autenticano la comunicazione locale con il dispositivo e devono essere trattati come una password. Per l’utilizzo è soprattutto importante che non finiscano nei log, nei backup non cifrati o in un repository.

### Nessun port forwarding verso PortaSplit

L’errore evitabile più comune sarebbe rendere la porta locale del dispositivo direttamente raggiungibile da Internet. Una regola come questa sarebbe pericolosa:

```text
Internet → TCP 6444 → PortaSplit
```

Non c’è alcun buon motivo per rendere PortaSplit direttamente raggiungibile da Internet. Home Assistant si trova già nella rete locale e funge da istanza di controllo. Il router non dovrebbe avere alcun inoltro di porta verso PortaSplit, UPnP dovrebbe essere limitato o disattivato se possibile, le connessioni in entrata dovrebbero essere bloccate per impostazione predefinita e non dovrebbe essere usata alcuna condivisione DMZ per il dispositivo.

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

Durante la prima configurazione il dispositivo richiede accesso a Internet verso la cloud Midea. Dopo aver completato la configurazione locale, è possibile verificare se l’accesso a Internet in uscita possa essere bloccato. Non dovrebbe però essere impostato immediatamente un blocco definitivo. Occorre prima verificare se il controllo locale continua a funzionare, se il dispositivo rimane raggiungibile dopo un riavvio, se resiste a un riavvio del router, se risponde ancora dopo diversi giorni, se l’app MSmartHome è ancora necessaria e se vengono ancora offerti aggiornamenti firmware. Chi desidera continuare a utilizzare cloud e aggiornamenti firmware può consentire temporaneamente l’accesso a Internet in uscita e bloccarlo nuovamente in seguito.

### La segmentazione di rete può impedire Discovery

La ricerca automatica dei dispositivi si basa spesso sul traffico broadcast o multicast, che normalmente non viene instradato oltre i confini delle VLAN. Home Assistant potrebbe quindi non trovare automaticamente PortaSplit, anche se una normale connessione IP fosse consentita.

In questo caso può essere utile configurare temporaneamente PortaSplit nella stessa VLAN di Home Assistant, indicare manualmente l’IP del dispositivo, utilizzare un’idonea funzione di broadcast relay oppure definire regole firewall mirate dopo la configurazione. La configurazione manuale è spesso persino l’opzione migliore dal punto di vista della sicurezza, perché non richiede di consentire traffico broadcast aggiuntivo tra le reti.

### Assegnazione DHCP statica

PortaSplit dovrebbe ricevere un’assegnazione DHCP fissa nel router:

```text
PortaSplit → 192.168.30.25
```

Una DHCP reservation è generalmente preferibile a un IP statico impostato nel dispositivo. Home Assistant trova il dispositivo in modo affidabile, le regole firewall possono essere limitate a un indirizzo fisso, l’analisi degli errori risulta più semplice e l’assegnazione resta stabile dopo riavvii del router o del dispositivo. Una regola firewall può quindi essere formulata in modo molto restrittivo:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

La porta effettivamente necessaria va verificata in base all’integrazione e al proprio dispositivo.

### Home Assistant come pilastro centrale della fiducia

Chi controlla PortaSplit localmente trasferisce parte della fiducia dalla cloud Midea a Home Assistant. Se Home Assistant viene compromesso, un attaccante potrebbe controllare non solo il climatizzatore, ma l’intera smart home.

Home Assistant dovrebbe quindi essere aggiornato regolarmente, non essere pubblicato tramite inoltro di porta non protetto, essere protetto con una password forte e unica, utilizzare l’autenticazione a più fattori, creare backup cifrati, contenere solo add-on necessari e non consentire accessi SSH non necessari da Internet. Per l’accesso remoto, VPN, Home Assistant Cloud o un reverse proxy correttamente configurato sono opzioni migliori di un semplice inoltro di porta sulla porta 8123.

### HACS e il rischio della supply chain

`Midea Smart AC` e `Midea AC LAN` sono Custom Integrations. Vengono eseguite all’interno di Home Assistant e hanno quindi accesso esteso al suo ambiente di esecuzione. Un’integrazione dannosa o compromessa potrebbe teoricamente leggere dati di configurazione, estrarre segreti, stabilire connessioni di rete, scansionare dispositivi nella rete locale, leggere gli stati di altre entità, trasferire dati a sistemi esterni e compromettere la disponibilità di Home Assistant.

Ciò non significa che le integrazioni citate siano dannose. Entrambi i progetti sono pubblicamente consultabili, sviluppati attivamente e hanno una community visibile. L’open source, tuttavia, non è una garanzia automatica di sicurezza. Prima dell’installazione vale almeno la pena controllare se il repository è mantenuto attivamente, se esistono release regolari, quante persone contribuiscono al codice, se sono presenti Security Issue aperte, se di recente sono cambiati maintainer o proprietari del repository, se HACS rimanda al repository previsto e se un aggiornamento contiene modifiche insolitamente grandi o non spiegabili.

Gli aggiornamenti non dovrebbero essere installati ciecamente subito dopo la pubblicazione. Specialmente nei sistemi smart home rilevanti per la sicurezza, è opportuno attendere alcuni giorni e verificare le note di rilascio e i problemi segnalati.

### Proteggere l’account cloud

Finché la cloud Midea viene utilizzata per la configurazione o il controllo tramite app, anche l’account Midea resta parte del modello di sicurezza. Richiede una password unica, non condivisa con altri servizi, un password manager, l’autenticazione a più fattori se disponibile, la rimozione di vecchi smartphone e sessioni, l’assenza di account condivisi e un controllo regolare dei dispositivi registrati nell’account.

Se l’integrazione Home Assistant richiede nome utente e password durante la configurazione, occorre verificare se le credenziali vengono usate solo per il recupero una tantum del token oppure memorizzate in modo permanente. Gli sviluppatori di `Midea Smart AC` scrivono che i dispositivi non vengono collegati ad account integrati dell’integrazione dopo la configurazione e che token e key possono essere recuperati anche manualmente tramite il proprio account usando la CLI. Ove possibile, è preferibile il proprio account rispetto ad account condivisi di terzi o integrati.

### Bloccare la cloud oppure no?

Dopo una configurazione completata con successo, sorge la domanda se l’accesso a Internet di PortaSplit debba essere completamente bloccato. A favore del blocco vi sono meno telemetria, minore dipendenza da servizi esterni, una superficie di attacco più ridotta attraverso la cloud del produttore, il fatto che il dispositivo non possa contattare destinazioni esterne arbitrarie e minori effetti delle modifiche lato cloud.

Contro il blocco vi sono il possibile mancato funzionamento dell’app MSmartHome al di fuori della rete domestica, l’impossibilità di scaricare aggiornamenti firmware, eventuali guasti delle funzioni di orario o cloud, una nuova registrazione o ripristino più difficili e reazioni impreviste di alcuni dispositivi dopo lunghi periodi offline.

Una sequenza pragmatica: configurare normalmente il dispositivo, testare Home Assistant e l’app, salvare token e configurazione, bloccare l’accesso a Internet, riavviare il dispositivo e Home Assistant, osservare per diversi giorni e, se necessario, consentire nuovamente l’accesso a Internet solo temporaneamente.

### Aggiornamenti firmware: vantaggio di sicurezza o rischio per l’integrazione?

Gli aggiornamenti firmware sono un dilemma per i dispositivi IoT. Possono chiudere vulnerabilità note, migliorare la stabilità, modernizzare i meccanismi di sicurezza e introdurre nuove funzioni. Possono però anche modificare interfacce locali, interrompere integrazioni basate sul reverse engineering, invalidare token, disattivare l’API locale e introdurre nuove dipendenze cloud.

Il firmware PortaSplit distribuito nel gennaio 2026 ha introdotto, ad esempio, una nuova modalità silenziosa per l’unità esterna che riduce il rumore di circa 6 decibel. Le integrazioni della community hanno dovuto prima comprenderla e implementarla, come documentato in una GitHub Issue dedicata a PortaSplit.

Ne consegue che gli aggiornamenti firmware non devono essere impediti in modo assoluto: prima di un aggiornamento, verificare se altri utenti di Home Assistant segnalano problemi, salvare preventivamente configurazione e token, creare un backup di Home Assistant e testare completamente il controllo locale dopo l’aggiornamento. Sicurezza non significa “non aggiornare mai”. Un firmware obsoleto può essere più pericoloso di un’integrazione temporaneamente incompatibile.

### I log di debug contengono dati sensibili

In caso di problemi, i progetti open source richiedono spesso log di debug. La documentazione di `Midea AC LAN` mostra come attivare il logging per i due componenti rilevanti:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Successivamente è possibile scaricare i log da Impostazioni, Sistema e Registri. A seconda dell’integrazione e dell’errore, tali log possono contenere indirizzi IP locali, ID dispositivo, numero di serie, identificatore del modello, risposte cloud, informazioni sull’account, token o parti di essi, pacchetti di rete nonché timestamp e comportamento di utilizzo. Prima di caricarli in una GitHub Issue pubblica, occorre quindi verificarli e oscurare i valori sensibili.

Al termine della ricerca degli errori, il logging di debug deve essere nuovamente rimosso. Il logging di debug permanentemente attivo non aumenta solo il consumo di memoria, ma amplia anche la quantità di informazioni sensibili nei backup.

### Cosa dice Midea sulla sicurezza

Midea promuove il proprio ecosistema SmartHome dichiarando l’orientamento a vari standard di sicurezza e privacy, tra cui EN 303 645, UK PSTI, NIST, trattamento dei dati conforme al GDPR e requisiti della EU Radio Equipment Directive. Sono segnali positivi, ma non indicano come siano effettivamente implementati ogni singolo firmware PortaSplit, ogni endpoint cloud e ogni API locale. Le dichiarazioni di certificazione e marketing non sostituiscono una verifica tecnica del dispositivo concreto.

Sarebbe altrettanto errato dedurre dall’avviso di un’integrazione community che PortaSplit sia generalmente insicura. Il problema descritto riguarda l’architettura dei token di lunga durata e il loro utilizzo da parte di client non ufficiali.

### Rischio per scenario

| Scenario | Rischio | Motivazione |
| --- | --- | --- |
| Rete domestica normale senza port forwarding | gestibile | Un attaccante deve prima ottenere accesso alla WLAN, a Home Assistant o a un backup. |
| Rete domestica piatta con molti dispositivi IoT insicuri | medio | Un altro dispositivo IoT compromesso può raggiungere PortaSplit o Home Assistant nella stessa rete. |
| PortaSplit direttamente raggiungibile da Internet | alto | Il dispositivo non deve mai essere pubblicato tramite port forwarding. |
| Token e key pubblici su GitHub | alto | I segreti devono essere considerati compromessi; non è garantito che possano essere revocati. |
| VLAN IoT separata, firewall restrittivo, controllo locale | relativamente basso | Anche in presenza di una vulnerabilità nel dispositivo, la libertà di movimento nella rete è fortemente limitata. |

## Backup della configurazione

Il backup di token, key e configurazione è il più importante passaggio una tantum: una volta chiuse le interfacce cloud token, un backup è l’unica via per una nuova configurazione. `Midea AC LAN` salva un file di configurazione JSON per i dispositivi V3 dopo una configurazione riuscita. Il percorso documentato è:

```text
/config/.storage/midea_ac_lan/
```

Il file utilizza l’ID dispositivo come nome file:

```text
<device-id>.json
```

Questo file non è una normale nota di testo. Può contenere ID dispositivo, numero di serie, indirizzo IP, token, key, informazioni sul protocollo nonché parametri cloud e del dispositivo. Si applica quindi quanto segue:

- Non caricarlo in un repository GitHub pubblico.
- Non pubblicarlo nei forum.
- Non condividerlo come screenshot non oscurato.
- Non inviarlo tramite e-mail non cifrata.

Anche un repository Git privato non è automaticamente il luogo di archiviazione corretto, poiché i segreti restano nella cronologia Git anche se vengono successivamente eliminati dal file corrente. Sono più adatti un backup cifrato, un password manager con allegato file, un backup NAS cifrato, un supporto offline cifrato o un archivio cifrato con password memorizzata separatamente.

Per il backup tramite il terminale Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Visualizzare il file:

```bash
cat <device-id>.json
```

Per la copia, il file non dovrebbe essere trasferito tramite un servizio web pubblico. È preferibile un archivio cifrato, da trasferire poi in un backup cifrato:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

I file in `.storage` non devono essere modificati manualmente. Lo sviluppatore raccomanda espressamente di non eliminare né modificare direttamente il file JSON in caso di problemi, ma di rinominarlo e salvarlo prima di apportare modifiche.

Un backup completo di Home Assistant contiene anch’esso questi file. Una copia separata è comunque utile, poiché i backup di Home Assistant possono danneggiarsi, un ripristino può sovrascrivere l’integrazione, il file potrebbe servire specificamente per una nuova configurazione futura e un backup non dovrebbe mai risiedere solo sullo stesso sistema.

## Rimuovere i segreti da un repository Git pubblicato

Se un file JSON è stato pubblicato accidentalmente su GitHub, non è sufficiente eliminarlo normalmente e creare un nuovo commit. Il file resta recuperabile nella cronologia Git. Sono necessari almeno questi passaggi:

1. Rendere subito privato il repository, se possibile.
2. Rimuovere il file dall’intera cronologia Git.
3. Considerare cache e fork di GitHub.
4. Trattare il token come compromesso.
5. Rimuovere il dispositivo dall’account Midea e ricollegarlo, se ciò genera nuove chiavi.
6. Configurare nuovamente l’integrazione Home Assistant.
7. Modificare la password dell’account Midea se erano coinvolte anche le credenziali.

Il fatto che un nuovo pairing generi effettivamente un nuovo token varia in base al dispositivo e all’architettura cloud. Non bisogna fare affidamento sul fatto che la modifica della password dell’account invalidi automaticamente il token locale del dispositivo.

## Automazioni utili

Dopo una corretta integrazione, PortaSplit può essere utilizzata in modo molto più intelligente. Gli ID delle entità devono essere adattati alla propria installazione.

Raffreddare solo quando le finestre sono chiuse:

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

Accendere a temperatura ambiente elevata:

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

Spegnere quando non c’è nessuno a casa:

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

La Midea PortaSplit si integra bene in Home Assistant. Dopo una configurazione riuscita, può essere controllata localmente e inclusa nelle automazioni, eliminando gran parte della dipendenza dalla cloud per l’uso quotidiano.

Dal punto di vista della sicurezza, l’integrazione è accettabile se vengono rispettate alcune regole di base: nessun port forwarding, mantenere segreti token e key, cifrare i backup, controllare i log di debug prima della pubblicazione, proteggere Home Assistant, segmentare i dispositivi IoT, limitare l’accesso a Internet in uscita a quanto necessario e non installare ciecamente aggiornamenti firmware e HACS. Utilizzata in questo modo, PortaSplit resta un climatizzatore efficiente e diventa al tempo stesso una componente sensatamente integrabile di una smart home controllata localmente.

## Fonti

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: tipi di dispositivo supportati `0xAC` e `0xCC`, PortaSplit con “Out Silent Mode”, utilizzo della cloud per ottenere token e key nei dispositivi V3, configurazione manuale e porta predefinita 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN`: categorie di dispositivi supportate, connessione TCP più lunga per la sincronizzazione dello stato e versione minima Home Assistant 2024.4.1.

3.  [midea_ac_lan: documentazione delle entità climatiche](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entità e attributi per climatizzatori, tra cui potenza, energia totale, frequenza del compressore e metodi di decodifica per i valori energetici di singoli sottotipi.

4.  [midea_ac_lan: indicazioni su debug e configurazione](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): memorizzazione della configurazione del dispositivo in `/config/.storage/midea_ac_lan/`, raccomandazione di salvare anziché eliminare il file JSON e configurazione del logger per i log di debug.

5.  [Issue 779: Out Silent Mode di PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): richiesta di supporto per la modalità silenziosa dell’unità esterna introdotta con l’aggiornamento firmware di gennaio 2026, che riduce il rumore di circa 6 decibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): dichiarazioni del produttore sugli standard di sicurezza e privacy EN 303 645, PSTI, NIST, GDPR e RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione e gestione di Custom Integrations che non fanno parte di Home Assistant Core.
