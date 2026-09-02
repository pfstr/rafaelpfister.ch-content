---
title: "Tailscale: confronto tra Exit Node e route di sottorete e il loro funzionamento tecnico"
navTitle: "Exit Node vs. sottorete"
description: "In Tailscale, l'Exit Node e il router di sottorete sono due modalità operative correlate ma diverse. Un router di sottorete apre in modo mirato specifici intervalli IP, mentre un Exit Node instrada tutto il traffico Internet attraverso di sé. Cosa significa questa differenza nella pratica, come Tailscale la implementa tramite WireGuard, l'approvazione delle route e SNAT, e quali sono i limiti di ciascuna variante."
date: "2026-09-02"
kategorie: "Rete e VPN"
timeToRead: "11 min di lettura"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-confronto-tra-exit-node-e-route-di-sottorete-e-il-loro-funzionamento-tecnico"
translationId: "article-c26cca4d635b9a04"
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
translationOf: tailscale-exit-node-subnet-routes
url: https://rafaelpfister.ch/it/blog/tailscale-confronto-tra-exit-node-e-route-di-sottorete-e-il-loro-funzionamento-tecnico
translationSourceHash: f05a193f13dd2b8aba3c9d049ea1c0a1fcc25b12c420a1d520f99854b7883a79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:00:37.884Z
translationReview: automatic
---

# Tailscale: confronto tra Exit Node e route di sottorete e il loro funzionamento tecnico

Un nodo Tailscale inizialmente è raggiungibile solo in quanto tale: tramite il suo indirizzo Tailscale, e basta. Affinché un nodo possa offrire ad altri dispositivi l'accesso a qualcosa oltre a sé stesso, esistono due modalità operative che vengono spesso confuse: il **router di sottorete** e l'**Exit Node**. Entrambi estendono la portata di un nodo, ma in direzioni diverse. Chi conosce la differenza sceglie la variante adatta ed evita di instradare accidentalmente tutto il traffico attraverso un computer esterno.

In breve: un router di sottorete apre **in modo mirato specifici intervalli IP** dietro il nodo, ad esempio la rete locale con un NAS e una stampante. Un Exit Node instrada **tutto il traffico Internet** di un dispositivo attraverso di sé, come una classica VPN full tunnel. Entrambi si basano tecnicamente sullo stesso meccanismo: l'annuncio delle route. L'Exit Node è sostanzialmente un caso particolare del router di sottorete, in cui viene annunciata la route predefinita.

## Router di sottorete: accesso mirato a una rete

Un router di sottorete annuncia uno o più intervalli IP che può raggiungere nella rete locale. Gli altri dispositivi nel tailnet che accettano queste route possono così raggiungere i dispositivi nell'intervallo annunciato, anche se su di essi non è installato Tailscale. È il modo per rendere raggiungibili un NAS, una stampante o un'interfaccia di amministrazione senza configurare un client VPN su ogni singolo dispositivo.

L'intervallo viene annunciato sul nodo router:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--advertise-routes=<CIDR>` | Annuncia uno o più intervalli IP (separati da virgole) che questo nodo inoltra |
| `--snat-subnet-routes=false` | Inoltra senza NAT sorgente, affinché i dispositivi di destinazione vedano il vero indirizzo sorgente Tailscale; richiede una route di ritorno nella rete locale |
| `--advertise-exit-node` | Forma abbreviata che annuncia `0.0.0.0/0` e `::/0`, offrendo quindi il nodo come Exit Node |

</details>

Il traffico fluisce solo dopo che la route è stata **approvata** nell'amministrazione di Tailscale. Il semplice annuncio non è sufficiente: è l'errore più comune. La route compare nella tabella di routing dei dispositivi che la accettano solo dopo l'approvazione.

## Exit Node: tutto il traffico attraverso un nodo

Un Exit Node annuncia la route predefinita (`0.0.0.0/0` e `::/0`). Quando un dispositivo seleziona questo Exit Node, **tutto** il suo traffico Internet in uscita passa attraverso il nodo, non solo il traffico verso una rete specifica. È utile per accedere a Internet tramite una sede con IP fisso o per instradare il traffico, in una rete non sicura, attraverso un'uscita affidabile.

La differenza rispetto a una route di sottorete è la selezione sul lato client: una route di sottorete viene usata automaticamente non appena il dispositivo la accetta e contatta una destinazione in quell'intervallo. Un Exit Node, invece, deve essere selezionato attivamente e si applica quindi a tutto il traffico:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `--exit-node=<IP oder Name>` | Seleziona un Exit Node; lasciandolo vuoto (`--exit-node=`) lo disattiva nuovamente |
| `--exit-node-allow-lan-access` | Consente l'accesso alla propria rete locale anche con un Exit Node attivo |

</details>

Proprio per questo, nell'assistenza quotidiana era sbagliato selezionare l'Exit Node per accedere a un singolo NAS: avrebbe reindirizzato tutto il traffico personale attraverso il computer esterno, invece di aprire soltanto quell'intervallo.

## Confronto

| Caratteristica | Router di sottorete | Exit Node |
|---|---|---|
| Route annunciata | Intervalli mirati, ad es. `192.168.1.0/24` | Route predefinita `0.0.0.0/0`, `::/0` |
| Utilizzo client | Automatico per le destinazioni nell'intervallo | Deve essere selezionato attivamente come Exit Node |
| Ambito | Solo le reti annunciate | Tutto il traffico Internet |
| Approvazione nell'amministrazione | Per ogni sottorete | Separatamente come Exit Node |
| Scopo tipico | Rendere raggiungibili servizi interni | Instradare il traffico in uscita attraverso una sede |

## Come Tailscale lo implementa tecnicamente

Entrambe le modalità operative si basano sullo stesso fondamento. Vale la pena separare i livelli.

**Data plane tramite WireGuard.** Ogni nodo dispone di una coppia di chiavi WireGuard. Il traffico effettivo tra due nodi viaggia direttamente come pacchetti WireGuard cifrati su UDP, ove possibile peer-to-peer dopo il NAT traversal, altrimenti tramite un server relay DERP come soluzione di ripiego. Tailscale non reinventa la cifratura, ma utilizza WireGuard come trasporto.

**Control plane tramite il Coordination Server.** Un Coordination Server centrale distribuisce le chiavi pubbliche e una network map che indica quale nodo possiede quali indirizzi e route. Il Coordination Server vede i metadati (chi può comunicare con chi, quali route sono approvate), ma non il contenuto dei pacchetti WireGuard. Quando si annuncia una route, il nodo lo comunica al control plane; solo con l'approvazione la route entra a far parte della network map ricevuta da tutti i nodi.

**Sul nodo router.** Affinché un nodo inoltri il traffico per altri dispositivi, deve avere abilitato l'inoltro IP e deve instradare i pacchetti tra l'interfaccia Tailscale e la rete locale. Per impostazione predefinita, Tailscale maschera il traffico inoltrato mediante NAT sorgente (SNAT): i dispositivi di destinazione nella rete locale vedono come mittente l'indirizzo locale del nodo router, non l'indirizzo Tailscale del dispositivo che accede. È il caso più semplice, perché i pacchetti di risposta ritrovano così automaticamente la strada verso il router. Se si disattiva SNAT, i dispositivi di destinazione vedono il vero indirizzo sorgente Tailscale, ma allora la rete locale deve sapere come instradare di nuovo l'intervallo Tailscale verso il router.

**Sul lato client.** Un dispositivo usa route esterne solo se le accetta. Sui client grafici per Windows e macOS, l'accettazione delle route di sottorete è preimpostata; su Linux viene attivata con `--accept-routes`. Quando il client accetta una route, la inserisce nella propria tabella di routing e la indirizza all'interfaccia Tailscale. I pacchetti destinati a un indirizzo in quell'intervallo vengono quindi incapsulati in WireGuard e inviati al nodo router. Con l'Exit Node il meccanismo è lo stesso, ma qui la route predefinita punta all'Exit Node, motivo per cui tutto il traffico passa attraverso di esso.

**L'approvazione.** Il fatto che le route abbiano effetto solo dopo l'approvazione è una funzione di sicurezza, non un passaggio superfluo: un nodo qualsiasi non deve poter attirare senza autorizzazione il traffico di intere reti. L'approvazione può essere effettuata manualmente nell'amministrazione o automaticamente tramite `autoApprovers` nelle regole di accesso (ACL). Exit Node e route di sottorete vengono approvati separatamente.

## Limiti

Entrambe le varianti hanno limiti che influenzano la scelta:

- **Il nodo router è un collo di bottiglia e un single point of failure.** Tutto il traffico per la rete annunciata passa attraverso questo unico nodo, la sua cifratura WireGuard e la sua connettività. Per garantire la disponibilità, più nodi possono annunciare la stessa route; Tailscale ne usa quindi uno e passa a un altro in caso di guasto.
- **SNAT nasconde la sorgente.** Con il NAT sorgente predefinito, tutti gli accessi appaiono sotto l'indirizzo del nodo router. Per la registrazione o le regole di accesso sui dispositivi di destinazione che richiedono la vera sorgente, è necessario disattivare SNAT e configurare la route di ritorno nella rete locale.
- **Un Exit Node instrada davvero tutto.** Tutto il traffico passa attraverso il nodo, con le relative conseguenze per throughput, latenza e riservatezza. Il gestore dell'Exit Node vede il traffico nel punto in cui lascia il tailnet. Usare come Exit Node solo nodi di cui ci si fida.
- **Le sottoreti sovrapposte sono un problema.** Se due sedi annunciano lo stesso intervallo privato, ad esempio `192.168.1.0/24`, un client non può distinguerli. Per questo Tailscale offre una riscrittura tramite IPv6 (`4via6`), che rende univoci gli intervalli.
- **Le chiavi in scadenza interrompono l'inoltro.** Se la chiave del nodo router scade, l'intera rete che si trova dietro non è più raggiungibile. Per un nodo router permanente, disattivare la scadenza delle chiavi nell'amministrazione.

Per l'accesso mirato a servizi interni, il router di sottorete è quasi sempre la scelta giusta: apre solo ciò che è necessario. Usare l'Exit Node quando si desidera consapevolmente instradare tutto il traffico in uscita attraverso una sede specifica.

## Fonti

1.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): annuncio delle route, approvazione, comportamento SNAT e alta disponibilità con più router.

2.  [Tailscale: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): annuncio della route predefinita, selezione sul client e accesso alla propria rete locale.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): interazione tra data plane WireGuard, Coordination Server e relay DERP.

4.  [WireGuard: panoramica del protocollo](https://www.wireguard.com/protocol/): la base crittografica del data plane che Tailscale utilizza come trasporto.
