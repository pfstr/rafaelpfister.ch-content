---
title: "Controllare localmente Midea PortaSplit con Home Assistant e utilizzarla in sicurezza"
navTitle: "Configurare PortaSplit"
description: "Dall'integrazione della community più adatta alla VLAN IoT: come configurare PortaSplit, proteggere token e chiave e limitare gli accessi al cloud e alla rete."
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
translationSourceHash: 859c24ec38af3b4b931702c7be50cf2224580d30045559ba089224d0de25339c
translatedAt: 2026-08-08T14:19:44.037Z
url: https://rafaelpfister.ch/it/blog/controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant
translationModel: gpt-5.6-terra
---

Dopo la configurazione, Midea PortaSplit può essere controllata direttamente nella rete locale tramite Home Assistant. A tale scopo, l'integrazione della community richiede due credenziali specifiche del dispositivo dal cloud Midea: token e chiave.

Questo articolo illustra la selezione, la configurazione e la protezione dell'integrazione. Le soluzioni descritte provengono dalla community e non sono supportate ufficialmente né da Midea né da Home Assistant. Modifiche al firmware o al cloud possono quindi influenzarne il comportamento in qualsiasi momento. Il contesto relativo all'interfaccia dei token e all'avviso ambiguo di dismissione è disponibile nell'[analisi delle API cloud Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Come funziona il controllo locale

Dopo la configurazione, i comandi di controllo effettivi vengono inviati direttamente da Home Assistant a PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Un comando di commutazione non deve passare attraverso un server Midea esterno, il tempo di risposta è breve, un guasto del cloud Midea non paralizza necessariamente il controllo locale già configurato e il dispositivo resta in linea di principio controllabile anche senza accesso a Internet.

Sui dispositivi più recenti con il cosiddetto protocollo V3, tuttavia, PortaSplit non accetta comandi locali senza protezione. Home Assistant richiede due valori specifici del dispositivo, un token e una chiave, utilizzati per l'autenticazione e la crittografia della connessione locale. Durante la configurazione iniziale, l'integrazione li recupera una sola volta tramite un'interfaccia cloud Midea e poi li salva localmente; per il controllo successivo non è necessaria alcuna connessione cloud.

In forma semplificata, il processo è il seguente:

1. PortaSplit viene collegata a MSmartHome.
2. Home Assistant accede a un cloud Midea.
3. Home Assistant riceve ID dispositivo, token e chiave.
4. Token e chiave vengono salvati localmente.
5. Home Assistant controlla PortaSplit direttamente nella LAN.

## Quale integrazione scegliere

### Midea Smart AC

Il repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> si concentra sui climatizzatori Midea e sui modelli OEM correlati e supporta i tipi di dispositivo `0xAC` e `0xCC`. Offre controllo locale, configurazione grafica, rilevamento automatico, configurazione manuale con token e chiave nonché interrogazione automatica delle capacità del dispositivo. La modalità “Out Silent Mode” di PortaSplit è supportata esplicitamente.

Come indizio di compatibilità, il progetto cita tra le altre le app Artic King, Midea Air, NetHome Plus, SmartHome o MSmartHome, Toshiba AC NA e 美的美居. In Europa PortaSplit utilizza tipicamente MSmartHome e rientra quindi in questo ecosistema.

### Midea AC LAN

Il repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> supporta non solo climatizzatori, ma anche numerose altre classi di dispositivi Midea: deumidificatori, ventilatori, purificatori d'aria, lavatrici, asciugatrici, lavastoviglie, apparecchi per l'acqua calda, pompe di calore, frigoriferi e altro ancora, in parte anche con marchi terzi come Carrier o Electrolux. Offre anch'esso comunicazione locale, rilevamento automatico dei dispositivi e sensori aggiuntivi e, secondo la descrizione del progetto, mantiene una connessione TCP più lunga con il dispositivo per sincronizzare tempestivamente le variazioni di stato. Richiede almeno Home Assistant 2024.4.1.

Il maggiore svantaggio al momento è l'avviso dello sviluppatore: le API dei token cloud utilizzate per aggiungere nuovi dispositivi vengono progressivamente dismesse. Di conseguenza, in futuro potrebbe diventare impossibile aggiungere nuovi dispositivi.

### Raccomandazione

Per una configurazione esclusivamente PortaSplit inizierei con `Midea Smart AC` e terrei `Midea AC LAN` presente come alternativa. `Midea Smart AC` è più specificamente orientata ai climatizzatori e documenta esplicitamente le attuali funzioni di PortaSplit.

Non è sensato utilizzare entrambe le integrazioni contemporaneamente e in modo permanente con lo stesso dispositivo. Più connessioni parallele causano problemi di stato, traffico di rete superfluo e comportamenti difficili da comprendere.

## Cosa offre l'integrazione

Dopo la configurazione, PortaSplit appare come entità `climate` in Home Assistant. A seconda del firmware e dell'integrazione, sono disponibili tra l'altro le seguenti funzioni:

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

Le entità che appaiono effettivamente dipendono dal modello, dal firmware, dal protocollo utilizzato e dalla rispettiva integrazione. `Midea Smart AC` interroga le capacità segnalate dal dispositivo e nasconde le funzioni non supportate dal modello. `Midea AC LAN` documenta anch'essa numerose entità per il clima, tra cui temperatura, umidità dell'aria, potenza attuale, energia totale, frequenza del compressore, stato della pompa e varie modalità operative, e indica metodi specifici per alcuni sottotipi di PortaSplit per decodificare i dati energetici.

Non tutte le misurazioni visualizzate devono necessariamente essere corrette. In particolare, consumo energetico e potenza vengono trasmessi in formati diversi nei vari modelli Midea. Se Home Assistant mostra valori palesemente errati, di solito occorre adattare il metodo di decodifica utilizzato e il dispositivo non è guasto.

## Requisiti

Sono necessari una Midea PortaSplit con funzione Wi-Fi, una rete Wi-Fi a 2,4 GHz, l'app MSmartHome, un account utente Midea, Home Assistant, HACS e accesso di rete tra Home Assistant e PortaSplit. PortaSplit dovrebbe essere collegata dapprima normalmente tramite l'app MSmartHome e aggiunta a Home Assistant solo in seguito.

## Passaggio 1: collegare PortaSplit a MSmartHome

1. Installare l'app MSmartHome.
2. Creare un account Midea o accedere.
3. Mettere PortaSplit in modalità di associazione Wi-Fi.
4. Collegare il dispositivo alla rete Wi-Fi a 2,4 GHz.
5. Verificare che PortaSplit possa essere controllata tramite l'app.

Molti dispositivi IoT supportano ancora solo 2,4 GHz. Se il router utilizza lo stesso SSID per 2,4 e 5 GHz, la configurazione di solito funziona comunque. In caso di problemi, può essere utile predisporre temporaneamente una rete Wi-Fi separata a 2,4 GHz.

## Passaggio 2: installare HACS

HACS è il Community Store di Home Assistant. Consente di installare integrazioni della community che non fanno parte di Home Assistant Core. Dopo l'installazione di HACS, aprire HACS, passare alle integrazioni, cercare `Midea Smart AC`, scaricare l'integrazione e riavviare Home Assistant. In alternativa, è possibile cercare `Midea AC LAN`.

HACS semplifica installazione e aggiornamenti. Tuttavia, non rende una Custom Integration un componente Home Assistant verificato ufficialmente. Questa differenza è fondamentale dal punto di vista della sicurezza e sarà trattata più avanti.

## Passaggio 3: aggiungere Midea Smart AC

Dopo il riavvio, andare su Impostazioni, Dispositivi e servizi e Aggiungi integrazione, quindi cercare `Midea Smart AC` e selezionare `Discover devices`. L'integrazione può cercare nell'intera rete locale oppure interrogare specificamente l'indirizzo IP di PortaSplit.

Se il dispositivo viene trovato, l'integrazione richiede per i dispositivi V3 più recenti regione, account Midea, password e ID dispositivo, nonché il token e la chiave derivati da essi. La regione cloud deve corrispondere all'account utilizzato. In caso di problemi, il progetto raccomanda di provare anche le altre regioni disponibili.

### Configurazione manuale

Se la configurazione automatica non riesce, il dispositivo può essere configurato manualmente. Per `Midea Smart AC` sono necessarie le seguenti informazioni:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

La porta standard documentata è:

```text
6444/TCP
```

Per i dispositivi V3, la documentazione indica il token come stringa esadecimale di 128 caratteri e la chiave come stringa esadecimale di 64 caratteri. Entrambi i valori sono segreti e devono essere trattati di conseguenza. Chi non desidera ottenere le credenziali tramite Discovery può recuperarle con il proprio account tramite la CLI `msmart-ng`.

## Utilizzare PortaSplit in sicurezza

Chi controlla PortaSplit localmente recupera una parte del controllo dal cloud del produttore, ma trasferisce anche la responsabilità nella propria rete. I punti seguenti fanno sì che token e chiave causino danni limitati anche in caso di incidente e che il dispositivo rimanga correttamente isolato.

### Token e chiave sono segreti

Token e chiave autenticano la comunicazione locale con il dispositivo e devono essere trattati come una password. Per l'uso quotidiano vale soprattutto: non devono finire nei log, nei backup non cifrati o in un repository.

### Nessun port forwarding a PortaSplit

L'errore evitabile più comune sarebbe rendere la porta locale del dispositivo direttamente raggiungibile da Internet. Una regola come questa sarebbe pericolosa:

```text
Internet → TCP 6444 → PortaSplit
```

Non esiste una buona ragione per rendere PortaSplit direttamente raggiungibile da Internet. Home Assistant si trova già nella rete locale e funge da istanza di controllo. Il router non dovrebbe avere alcun inoltro di porta verso PortaSplit, UPnP dovrebbe essere limitato o disattivato se possibile, le connessioni in ingresso dovrebbero essere bloccate per impostazione predefinita e non dovrebbe essere utilizzata alcuna condivisione DMZ per il dispositivo.

### VLAN IoT dedicata

La migliore architettura di rete è una rete IoT separata:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

PortaSplit si trova nella VLAN IoT. Home Assistant può accedere selettivamente al dispositivo, ma PortaSplit non può accedere arbitrariamente a PC, NAS e altri sistemi interni. Una possibile logica firewall:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la configurazione iniziale, il dispositivo richiede l'accesso a Internet per il cloud Midea. Dopo aver completato con successo la configurazione locale, è possibile testare se l'accesso a Internet in uscita possa essere bloccato. Non si dovrebbe tuttavia impostare subito un blocco definitivo. Occorre prima verificare che il controllo locale continui a funzionare, che il dispositivo resti raggiungibile dopo un riavvio, che superi un riavvio del router, che risponda ancora dopo diversi giorni, che l'app MSmartHome sia ancora necessaria e che siano ancora disponibili aggiornamenti del firmware. Chi desidera continuare a usare il cloud e gli aggiornamenti firmware può consentire temporaneamente l'accesso a Internet in uscita e bloccarlo nuovamente in seguito.

### La segmentazione di rete può impedire Discovery

La ricerca automatica dei dispositivi si basa spesso sul traffico broadcast o multicast, che normalmente non viene instradato oltre i confini delle VLAN. Home Assistant potrebbe quindi non trovare automaticamente PortaSplit, anche se fosse consentita una normale connessione IP.

In tal caso, può essere utile configurare temporaneamente PortaSplit nella stessa VLAN di Home Assistant, indicare manualmente l'IP del dispositivo, utilizzare un'adeguata funzione di relay per broadcast oppure definire regole firewall mirate dopo la configurazione. Dal punto di vista della sicurezza, la configurazione manuale è spesso persino la variante migliore, poiché non richiede di consentire traffico broadcast aggiuntivo tra le reti.

### Assegnazione DHCP statica

Nel router, a PortaSplit dovrebbe essere assegnata una prenotazione DHCP fissa:

```text
PortaSplit → 192.168.30.25
```

Una prenotazione DHCP è generalmente preferibile a un IP statico impostato nel dispositivo. Home Assistant trova il dispositivo in modo affidabile, le regole firewall possono essere limitate a un indirizzo fisso, l'analisi degli errori è più semplice e l'assegnazione rimane stabile dopo il riavvio del router o del dispositivo. Una regola firewall può quindi essere formulata in modo molto restrittivo:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

La porta effettivamente necessaria deve essere verificata in base all'integrazione e al proprio dispositivo.

### Home Assistant come principale ancoraggio di fiducia

Chi controlla PortaSplit localmente sposta in parte la fiducia dal cloud Midea a Home Assistant. Se Home Assistant viene compromesso, un aggressore potrebbe controllare non solo il climatizzatore, ma l'intera smart home.

Home Assistant dovrebbe quindi essere aggiornato regolarmente, non essere pubblicato tramite un inoltro di porta non protetto, essere protetto con una password forte e univoca, utilizzare l'autenticazione a più fattori, creare backup cifrati, contenere solo add-on necessari e non consentire accesso SSH non necessario da Internet. Per l'accesso remoto, una VPN, Home Assistant Cloud o un reverse proxy configurato correttamente sono opzioni migliori di un semplice inoltro della porta 8123.

### HACS e il rischio della supply chain

`Midea Smart AC` e `Midea AC LAN` sono Custom Integrations. Vengono eseguite all'interno di Home Assistant e hanno quindi un accesso esteso al suo ambiente di runtime. Un'integrazione malevola o compromessa potrebbe teoricamente leggere dati di configurazione, estrarre segreti, stabilire connessioni di rete, scansionare dispositivi nella rete locale, leggere gli stati di altre entità, trasferire dati a sistemi esterni e compromettere la disponibilità di Home Assistant.

Ciò non significa che le integrazioni menzionate siano malevole. Entrambi i progetti sono pubblicamente consultabili, sviluppati attivamente e dispongono di una community visibile. L'open source non è tuttavia una garanzia automatica di sicurezza. Prima dell'installazione vale almeno la pena verificare se il repository è mantenuto attivamente, se vengono pubblicate release regolari, quante persone contribuiscono al codice, se esistono problemi di sicurezza aperti, se i maintainer o i proprietari del repository sono cambiati di recente, se HACS rimanda al repository previsto e se un aggiornamento contiene modifiche insolitamente grandi o inspiegabili.

Gli aggiornamenti non dovrebbero essere installati alla cieca immediatamente dopo la pubblicazione. Soprattutto nei sistemi smart home rilevanti per la sicurezza, è sensato attendere alcuni giorni e verificare le note di rilascio e i problemi segnalati.

### Proteggere l'account cloud

Finché il cloud Midea viene utilizzato per la configurazione o il controllo tramite app, anche l'account Midea rimane parte del modello di sicurezza. Deve avere una password univoca non condivisa con altri servizi, un gestore di password, l'autenticazione a più fattori se disponibile, la rimozione di vecchi smartphone e sessioni, l'assenza di account condivisi e un controllo regolare dei dispositivi registrati nell'account.

Se l'integrazione di Home Assistant richiede nome utente e password durante la configurazione, occorre verificare se le credenziali vengono utilizzate solo per il recupero una tantum del token o memorizzate permanentemente. Gli sviluppatori di `Midea Smart AC` scrivono che dopo la configurazione i dispositivi non vengono associati ad account integrati nell'integrazione e che token e chiave possono essere ottenuti manualmente anche tramite CLI usando il proprio account. Ove possibile, il proprio account è preferibile ad account di terzi o account collettivi integrati.

### Bloccare il cloud oppure no?

Dopo una configurazione riuscita, si pone la questione se l'accesso a Internet di PortaSplit debba essere bloccato completamente. A favore del blocco vi sono meno telemetria, minore dipendenza da servizi esterni, una minore superficie di attacco attraverso il cloud del produttore, il fatto che il dispositivo non possa contattare arbitrariamente destinazioni esterne e un minore impatto delle modifiche lato cloud.

In senso contrario, l'app MSmartHome potrebbe non funzionare più fuori dalla rete domestica, gli aggiornamenti firmware non verrebbero scaricati, potrebbero venir meno le funzioni di orario o cloud, un nuovo accesso o ripristino potrebbe diventare più difficile e alcuni dispositivi potrebbero reagire in modo inatteso dopo un lungo periodo offline.

Una sequenza pragmatica: configurare normalmente il dispositivo, testare Home Assistant e l'app, proteggere token e configurazione, bloccare l'accesso a Internet, riavviare il dispositivo e Home Assistant, osservare per diversi giorni e, se necessario, riabilitare l'accesso a Internet solo temporaneamente.

### Aggiornamenti firmware: vantaggio per la sicurezza o rischio per l'integrazione?

Gli aggiornamenti firmware sono un dilemma per i dispositivi IoT. Possono chiudere vulnerabilità note, migliorare la stabilità, modernizzare i meccanismi di sicurezza e introdurre nuove funzioni. Possono però anche modificare le interfacce locali, interrompere integrazioni basate su reverse engineering, rendere non validi i token, disattivare l'API locale e introdurre nuove dipendenze dal cloud.

Il firmware PortaSplit distribuito nel gennaio 2026 ha introdotto, ad esempio, una nuova modalità silenziosa per l'unità esterna che riduce il rumore di circa 6 decibel. Le integrazioni della community hanno dovuto prima comprenderla e implementarla, come documentato in una issue GitHub dedicata a PortaSplit.

Ne consegue che gli aggiornamenti firmware non dovrebbero essere impediti in assoluto: prima di un aggiornamento occorre verificare se altri utenti di Home Assistant segnalano problemi, salvare prima configurazione e token, creare un backup di Home Assistant e testare completamente il controllo locale dopo l'aggiornamento. Sicurezza non significa “non aggiornare mai”. Un firmware obsoleto può essere più pericoloso di un'integrazione temporaneamente incompatibile.

### I log di debug contengono dati sensibili

In caso di problemi, i progetti open source richiedono spesso log di debug. La documentazione di `Midea AC LAN` mostra come attivare il logging per i due componenti rilevanti:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Successivamente, i log possono essere scaricati tramite Impostazioni, Sistema e Registri. A seconda dell'integrazione e dell'errore, tali log possono contenere indirizzi IP locali, ID dispositivo, numero di serie, identificativo del modello, risposte cloud, informazioni sull'account, token o parti di essi, pacchetti di rete nonché timestamp e comportamento di utilizzo. Prima di caricarli in una issue GitHub pubblica, devono quindi essere controllati e i valori sensibili devono essere oscurati.

Al termine della ricerca dell'errore, il logging di debug va nuovamente rimosso. Il logging di debug attivo in modo permanente non aumenta solo il consumo di spazio, ma anche la quantità di informazioni sensibili nei backup.

### Cosa dice Midea stessa sulla sicurezza

Midea pubblicizza il proprio ecosistema SmartHome con l'orientamento a diversi standard di sicurezza e protezione dei dati, tra cui EN 303 645, UK PSTI, NIST, trattamento dei dati conforme al GDPR e requisiti della direttiva europea sulle apparecchiature radio. Sono segnali positivi, ma non costituiscono un'affermazione su come siano effettivamente implementati ogni singolo firmware PortaSplit, ogni endpoint cloud e ogni API locale. Le dichiarazioni di certificazione e marketing non sostituiscono una verifica tecnica del dispositivo concreto.

Sarebbe altrettanto errato dedurre dall'avviso di un'integrazione della community che PortaSplit sia generalmente insicura. Il problema descritto riguarda l'architettura dei token a lunga durata e il loro utilizzo da parte di client non ufficiali.

### Rischio per scenario

| Scenario | Rischio | Motivazione |
| --- | --- | --- |
| Rete domestica normale senza inoltro di porta | gestibile | Un aggressore deve prima accedere al Wi-Fi, a Home Assistant o a un backup. |
| Rete domestica piatta con molti dispositivi IoT insicuri | medio | Un altro dispositivo IoT compromesso può raggiungere PortaSplit o Home Assistant nella stessa rete. |
| PortaSplit raggiungibile direttamente da Internet | alto | Il dispositivo non dovrebbe mai essere pubblicato tramite inoltro di porta. |
| Token e chiave pubblici su GitHub | alto | I segreti sono da considerare compromessi; non è garantito che possano essere revocati. |
| VLAN IoT separata, firewall restrittivo, controllo locale | relativamente basso | Anche in presenza di una vulnerabilità nel dispositivo, la libertà di movimento nella rete è fortemente limitata. |

## Backup della configurazione

Il salvataggio di token, chiave e configurazione è il passaggio una tantum più importante: una volta chiuse le interfacce cloud per i token, un backup è l'unica via per una nuova configurazione. `Midea AC LAN` salva, dopo una configurazione riuscita per i dispositivi V3, un file di configurazione JSON. Il percorso documentato è:

```text
/config/.storage/midea_ac_lan/
```

Il file utilizza l'ID dispositivo come nome file:

```text
<device-id>.json
```

Questo file non è una normale nota di testo. Può contenere ID dispositivo, numero di serie, indirizzo IP, token, chiave, informazioni sul protocollo e parametri cloud e del dispositivo. Di conseguenza:

- Non caricarlo in un repository GitHub pubblico.
- Non pubblicarlo nei forum.
- Non condividerlo come screenshot non oscurato.
- Non inviarlo tramite e-mail non cifrata.

Neppure un repository Git privato è automaticamente il luogo di archiviazione corretto, poiché i segreti rimangono nella cronologia Git anche se in seguito vengono eliminati dal file corrente. Sono più adatti un backup cifrato, un gestore di password con allegato di file, un backup NAS cifrato, un supporto offline cifrato o un archivio cifrato con password salvata separatamente.

Per il backup tramite il terminale di Home Assistant:

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

I file in `.storage` non dovrebbero essere modificati manualmente. Lo sviluppatore raccomanda espressamente, in caso di problemi, di non eliminare né modificare direttamente il file JSON, ma di rinominarlo e salvarlo prima di apportare modifiche.

Un backup completo di Home Assistant contiene anch'esso questi file. Una copia separata resta comunque sensata, poiché i backup di Home Assistant possono danneggiarsi, un ripristino può sovrascrivere l'integrazione, il file potrebbe essere necessario specificamente per una futura nuova configurazione e un backup non dovrebbe mai risiedere soltanto sullo stesso sistema.

## Rimuovere i segreti da un repository Git pubblicato

Se un file JSON è stato pubblicato per errore su GitHub, non basta cancellarlo normalmente e creare un nuovo commit. Il file rimane recuperabile nella cronologia Git. Sono necessari almeno questi passaggi:

1. Rendere subito privato il repository, se possibile.
2. Rimuovere il file dall'intera cronologia Git.
3. Considerare cache e fork di GitHub.
4. Trattare il token come compromesso.
5. Rimuovere il dispositivo dall'account Midea e ricollegarlo, se ciò genera nuove chiavi.
6. Configurare nuovamente l'integrazione di Home Assistant.
7. Modificare la password dell'account Midea, se sono state coinvolte anche le credenziali.

Il fatto che una nuova associazione generi effettivamente un nuovo token varia in base al dispositivo e all'architettura cloud. Non si dovrebbe fare affidamento sul fatto che la modifica della password dell'account renda automaticamente non valido il token locale del dispositivo.

## Automazioni utili

Dopo una corretta integrazione, PortaSplit può essere utilizzata in modo decisamente più intelligente. Gli ID delle entità devono essere adattati alla propria installazione.

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

Midea PortaSplit può essere integrata sorprendentemente bene in Home Assistant. Dopo una configurazione riuscita, può essere controllata localmente e inclusa nelle automazioni, eliminando così gran parte della dipendenza dal cloud per l'uso quotidiano.

Dal punto di vista della sicurezza, l'integrazione è accettabile se vengono rispettate alcune regole di base: nessun inoltro di porta, mantenere segreti token e chiave, cifrare i backup, controllare i log di debug prima della pubblicazione, proteggere Home Assistant, segmentare i dispositivi IoT, limitare l'accesso a Internet in uscita allo stretto necessario e non installare alla cieca gli aggiornamenti firmware e HACS. Utilizzata in questo modo, PortaSplit resta un climatizzatore potente e diventa al contempo una componente sensatamente integrabile di una smart home controllata localmente.

## Fonti

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: tipi di dispositivo supportati `0xAC` e `0xCC`, PortaSplit con “Out Silent Mode”, utilizzo del cloud per ottenere token e chiave sui dispositivi V3, configurazione manuale e porta standard 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN`: classi di dispositivi supportate, connessione TCP più lunga per la sincronizzazione dello stato e versione minima Home Assistant 2024.4.1.

3.  [midea_ac_lan: documentazione delle entità climatiche](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entità e attributi per climatizzatori, tra cui potenza, energia totale, frequenza del compressore e metodi di decodifica dei valori energetici di singoli sottotipi.

4.  [midea_ac_lan: indicazioni su debug e configurazione](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): archiviazione della configurazione del dispositivo in `/config/.storage/midea_ac_lan/`, raccomandazione di salvare anziché eliminare il file JSON e configurazione del logger per i log di debug.

5.  [Issue 779: Out Silent Mode di PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): richiesta di supporto per la modalità silenziosa dell'unità esterna introdotta con l'aggiornamento firmware di gennaio 2026, che riduce il rumore di circa 6 decibel.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): informazioni del produttore sugli standard di sicurezza e protezione dei dati EN 303 645, PSTI, NIST, GDPR e RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione e gestione di Custom Integrations che non fanno parte di Home Assistant Core.
