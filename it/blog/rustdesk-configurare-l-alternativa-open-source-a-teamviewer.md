---
title: "RustDesk: configurare l'alternativa open source a TeamViewer"
navTitle: "Configurare RustDesk"
description: "RustDesk è un software di assistenza remota open source con licenza AGPL, gratuito e self-hosted. Come installare il client su Windows (anche senza supervisione tramite MSI), come stabilire la connessione tramite il server di mediazione pubblico, un server proprio o una connessione diretta, quali funzioni servono nel supporto quotidiano e dove si trovano i limiti dell'utilizzo gratuito."
date: "2026-09-01"
kategorie: "Assistenza remota e supporto"
timeToRead: "9 min di lettura"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-configurare-l-alternativa-open-source-a-teamviewer"
translationId: "article-425ae4b8d562ae41"
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/it/blog/rustdesk-configurare-l-alternativa-open-source-a-teamviewer
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:49:08.139Z
translationReview: automatic
---

# RustDesk: configurare l'alternativa open source a TeamViewer

TeamViewer e AnyDesk offrono un'assistenza remota affidabile, ma per l'uso commerciale richiedono una licenza e i prezzi aumentano con il numero di dispositivi gestiti. RustDesk è un'alternativa con licenza AGPL-3.0: open source, gratuita e senza obbligo di licenza. Il client funziona su Windows, macOS, Linux, Android e iOS, oltre che nel browser. È scritto in Rust e l'interfaccia è realizzata in Flutter.

La differenza decisiva rispetto ai prodotti commerciali sta nella mediazione: RustDesk separa il client dall'infrastruttura server. È possibile utilizzare il server di mediazione pubblico gratuito, gestire un proprio server oppure stabilire una connessione diretta senza alcun server di mediazione. In questo modo RustDesk può essere utilizzato dalla singola postazione fino a una piattaforma di supporto self-hosted, senza che i dati di connessione debbano transitare attraverso un fornitore.

## I tre tipi di connessione

Prima di installare, è opportuno definire il tipo di connessione, poiché da questo dipendono la configurazione e le porte aperte.

| Tipo di connessione | Come funziona | Quando è utile |
|---|---|---|
| Server di mediazione pubblico | Due client si trovano tramite l'ID (numero di nove cifre) sul server RustDesk; la connessione avviene direttamente o tramite un relay | Avvio rapido, test, supporto occasionale privato |
| Server proprio (self-hosted) | Si gestiscono autonomamente i componenti server `hbbs` (mediazione) e `hbbr` (relay), e tutti i client ne inseriscono l'indirizzo | Uso commerciale, molti dispositivi, pieno controllo dei dati |
| Connessione diretta (Direct IP Access) | Il client si connette direttamente all'indirizzo IP della controparte senza server di mediazione | Entrambi i dispositivi sono raggiungibili nella stessa rete o tramite VPN |

Il server pubblico è espressamente destinato ai test e all'uso privato. Per l'uso produttivo e commerciale, il progetto raccomanda un server proprio, anche perché il servizio pubblico è soggetto a limitazioni e non offre alcuna garanzia di disponibilità.

## Installazione su Windows

L'installer si scarica dalla fonte ufficiale, le release del progetto su GitHub (`github.com/rustdesk/rustdesk`). Per Windows sono disponibili un file eseguibile e un pacchetto MSI. Per l'installazione interattiva è sufficiente un doppio clic. Per distribuire RustDesk su più computer o in background, utilizzare l'MSI con un'installazione silenziosa:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `/i <paket>` | Installa il pacchetto MSI specificato |
| `/qn` | Nessuna interfaccia, nessuna finestra di dialogo (silenziosa) |
| `/norestart` | Impedisce il riavvio automatico dopo l'installazione |

</details>

L'installazione silenziosa configura il servizio `RustDesk`, che viene eseguito all'avvio del sistema e consente l'accesso non supervisionato. Dopo l'installazione è possibile leggere l'ID del dispositivo dalla riga di comando, senza aprire l'interfaccia:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Anche la password fissa per l'accesso non supervisionato può essere impostata dalla riga di comando. Assegnare una password indipendente e sufficientemente lunga, non la password di accesso dell'utente:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--get-id` | Mostra l'ID RustDesk di nove cifre del dispositivo |
| `--password <wert>` | Imposta la password fissa per l'accesso non supervisionato |
| `--silent-install` | Installa la versione eseguibile (`.exe`) come servizio senza interfaccia |

</details>

## Inserire un server proprio

Se si gestisce un proprio server di mediazione, i client devono contenere il relativo indirizzo e la chiave pubblica. Nell'interfaccia queste impostazioni si trovano nelle impostazioni di rete come server ID, server relay e chiave. Per la distribuzione di massa, la configurazione può essere definita anche tramite file o variabili d'ambiente, in modo che ogni client si avvii già preconfigurato.

Un server proprio richiede i due componenti `hbbs` e `hbbr`, che di norma vengono eseguiti come container Docker. Entrambi richiedono porte aperte affinché i client possano registrarsi e utilizzare un relay.

| Porta | Protocollo | Componente e scopo |
|---|---|---|
| 21114 | TCP | Interfaccia web della versione Pro (solo in essa) |
| 21115 | TCP | `hbbs`, test del tipo NAT |
| 21116 | TCP e UDP | `hbbs`, registrazione (UDP) e instaurazione della connessione (TCP) |
| 21117 | TCP | `hbbr`, traffico relay |
| 21118, 21119 | TCP | Supporto per client web |

Aprite solo le porte effettivamente necessarie per il vostro tipo di connessione e limitate tramite firewall l'accesso alle reti da cui viene fornito il supporto.

## Connessione diretta senza server di mediazione

Se entrambi i dispositivi sono raggiungibili nella stessa rete o tramite VPN, RustDesk funziona completamente senza server di mediazione. A tal fine, attivate sul dispositivo di destinazione l'accesso diretto (nell'interfaccia, alla voce sicurezza, come "Abilita accesso diretto IP", internamente l'interruttore `direct-server`). Il client resta quindi in ascolto sulla porta standard 21118 (TCP). Nella finestra di connessione, inserite l'indirizzo IP della controparte al posto dell'ID.

Limitate l'accesso diretto tramite firewall alla rete dalla quale accedete. Se l'accesso avviene tramite VPN, abilitate la porta solo per l'intervallo di indirizzi VPN, non per l'intera Internet.

## Funzioni nel supporto quotidiano

RustDesk offre le funzioni necessarie per l'assistenza remota quotidiana:

- Trasmissione dello schermo e controllo remoto di tastiera e mouse, con selezione del monitor in presenza di più schermi.
- Trasferimento file in entrambe le direzioni tramite una finestra divisa in due.
- Chat di testo durante la sessione.
- Accesso non supervisionato tramite password fissa, per dispositivi senza un utente presente.
- Registrazione della sessione come file video, automaticamente se desiderato.
- Tunnel TCP e inoltro per raggiungere localmente singoli servizi della controparte.
- Rubrica e più dispositivi salvati: locali nella versione gratuita, condivisi lato server nella versione Pro.

Per il supporto assistito è importante sapere che, per impostazione predefinita, RustDesk chiede alla controparte se accettare la connessione e mostra durante la sessione che è in corso un accesso. La persona al dispositivo ne è quindi informata. Solo una password fissa per l'accesso non supervisionato elimina la richiesta di conferma. Utilizzate l'accesso non supervisionato solo su dispositivi i cui utenti sanno che il software è installato e a cosa serve.

## Limitazioni e confini

RustDesk sostituisce TeamViewer in molti casi, ma presenta limiti che è bene conoscere prima dell'uso:

- Il server di mediazione pubblico è soggetto a limitazioni, non offre garanzie di disponibilità e non è pensato per il funzionamento commerciale continuativo. Chi vuole lavorare in modo affidabile deve ospitare autonomamente il servizio.
- Un server proprio comporta oneri operativi: container, porte aperte, certificati e aggiornamenti sono di vostra responsabilità.
- Una rubrica condivisa lato server, la gestione centralizzata degli utenti e l'interfaccia web di amministrazione fanno parte della versione Pro, che diventa a pagamento a partire da un certo numero di dispositivi. Il client stesso e il funzionamento di base restano gratuiti.
- Senza password fissa non è possibile l'accesso non supervisionato, il che è corretto per il supporto assistito ma impedisce l'accesso spontaneo a un dispositivo non presidiato.
- L'ampiezza delle funzioni e la stabilità delle singole piattaforme, in particolare sui dispositivi mobili, non raggiungono i prodotti commerciali in ogni dettaglio. Verificate le funzioni importanti per voi prima di effettuare la migrazione.
- Alcuni programmi di sicurezza segnalano i software di assistenza remota come potenzialmente indesiderati. Se necessario, configurate un'eccezione e documentate perché il software è installato.

Per l'uso privato e il supporto di singoli dispositivi è sufficiente la versione gratuita con il server pubblico o una connessione diretta. Non appena gestite molti dispositivi, lavorate in ambito commerciale o avete bisogno del pieno controllo dei dati, vi serve un server proprio, con il relativo onere operativo in cambio dell'indipendenza.

## Fonti

1.  [RustDesk su GitHub](https://github.com/rustdesk/rustdesk): codice sorgente, release con gli installer e licenza AGPL-3.0.

2.  [Documentazione RustDesk](https://rustdesk.com/docs/): installazione, server proprio, porte e configurazione dei client.

3.  [rustdesk-server su GitHub](https://github.com/rustdesk/rustdesk-server): componenti server `hbbs` e `hbbr`, inclusa la panoramica delle porte per la gestione autonoma.
