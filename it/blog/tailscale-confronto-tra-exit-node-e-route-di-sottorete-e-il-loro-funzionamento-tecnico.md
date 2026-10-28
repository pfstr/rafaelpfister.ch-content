---
title: "Tailscale: confronto tra Exit Node e route di subnet e il loro funzionamento tecnico"
navTitle: "Exit Node vs. subnet"
description: "In Tailscale, gli Exit Node e i router di subnet sono due modalità operative correlate ma diverse. Un router di subnet apre in modo mirato determinati intervalli IP, mentre un Exit Node instrada attraverso di sé tutto il traffico Internet. Cosa significa in pratica questa differenza, come Tailscale la implementa tramite WireGuard, l’approvazione delle route e SNAT, e quali sono i limiti di ciascuna variante."
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
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:53:04.322Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/tailscale-confronto-tra-exit-node-e-route-di-sottorete-e-il-loro-funzionamento-tecnico
---

# Tailscale: confronto tra Exit Node e route di subnet e il loro funzionamento tecnico

Un nodo Tailscale inizialmente è raggiungibile solo in quanto tale: tramite il suo indirizzo Tailscale e nient’altro. Affinché un nodo consenta ad altri dispositivi di accedere a qualcosa di più di sé stesso, esistono due modalità operative che vengono spesso confuse: il **router di subnet** e l’**Exit Node**. Entrambi estendono la portata di un nodo, ma in direzioni diverse. Chi conosce la differenza sceglie la variante adatta ed evita di instradare accidentalmente tutto il traffico attraverso un computer di terzi.

In breve: un router di subnet apre **in modo mirato determinati intervalli IP** dietro il nodo, ad esempio la rete locale con un NAS e una stampante. Un Exit Node instrada attraverso di sé **l’intero traffico Internet** di un dispositivo, come una classica VPN full tunnel. Entrambi si basano tecnicamente sullo stesso meccanismo: l’advertising delle route. L’Exit Node è sostanzialmente un caso speciale del router di subnet, in cui viene advertised la route predefinita.

## Router di subnet: accesso mirato a una rete

Un router di subnet advertised uno o più intervalli IP che può raggiungere nella rete locale. Gli altri dispositivi nel tailnet che accettano queste route possono così raggiungere i dispositivi nell’intervallo advertised, anche se Tailscale non è installato su di essi. È il modo per rendere raggiungibili un NAS, una stampante o un’interfaccia di amministrazione senza configurare un client VPN su ogni singolo dispositivo.

L’intervallo viene advertised sul nodo router:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--advertise-routes=<CIDR>` | Advertised uno o più intervalli IP (separati da virgole) che questo nodo inoltra |
| `--snat-subnet-routes=false` | Inoltra senza NAT di origine, affinché i dispositivi di destinazione vedano il vero indirizzo sorgente Tailscale; richiede una route di ritorno nella rete locale |
| `--advertise-exit-node` | Abbreviazione che advertised `0.0.0.0/0` e `::/0`, offrendo quindi il nodo come Exit Node |

</details>

Il traffico passa solo dopo che la route è stata **approvata** nell’amministrazione di Tailscale. Il solo advertising non basta: è l’errore più comune. La route compare nella tabella di routing dei dispositivi che la accettano solo dopo l’approvazione.

## Exit Node: tutto il traffico attraverso un nodo

Un Exit Node advertised la route predefinita (`0.0.0.0/0` e `::/0`). Quando un dispositivo seleziona questo Exit Node, **tutto** il suo traffico Internet in uscita passa attraverso il nodo, non solo il traffico verso una rete specifica. È utile per accedere a Internet tramite una sede con IP fisso oppure per instradare il traffico attraverso un’uscita affidabile quando si è in una rete non sicura.

La differenza rispetto a una route di subnet è la selezione lato client: una route di subnet viene utilizzata automaticamente non appena il dispositivo accetta la route e si rivolge a una destinazione in tale intervallo. Un Exit Node, invece, deve essere selezionato attivamente e allora si applica a tutto il traffico:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Spiegazione delle opzioni</summary>

| Opzione | Effetto |
|---|---|
| `--exit-node=<IP oder Name>` | Seleziona un Exit Node; vuoto (`--exit-node=`) lo disattiva nuovamente |
| `--exit-node-allow-lan-access` | Consente l’accesso alla propria rete locale anche con un Exit Node attivo |

</details>

Proprio per questo, nel supporto quotidiano era sbagliato selezionare l’Exit Node per accedere a un singolo NAS: avrebbe reindirizzato tutto il traffico personale attraverso il computer di terzi, invece di aprire solo quell’unico intervallo.

## Confronto

| Caratteristica | Router di subnet | Exit Node |
|---|---|---|
| Route advertised | Intervalli mirati, ad es. `192.168.1.0/24` | Route predefinita `0.0.0.0/0`, `::/0` |
| Utilizzo lato client | Automatico per le destinazioni nell’intervallo | Deve essere selezionato attivamente come Exit Node |
| Ambito | Solo le reti advertised | L’intero traffico Internet |
| Approvazione nell’amministrazione | Per ogni subnet | Separatamente come Exit Node |
| Scopo tipico | Rendere raggiungibili i servizi interni | Instradare il traffico in uscita attraverso una sede |

## Come Tailscale lo implementa tecnicamente

Entrambe le modalità operative si basano sullo stesso fondamento. Vale la pena distinguere i livelli.

**Data plane tramite WireGuard.** Ogni nodo possiede una coppia di chiavi WireGuard. Il traffico effettivo fra due nodi passa direttamente come pacchetti WireGuard crittografati tramite UDP, ove possibile peer-to-peer dopo NAT traversal, altrimenti tramite un server relay DERP come soluzione di riserva. Tailscale non reinventa la crittografia, ma utilizza WireGuard come trasporto.

**Control plane tramite il Coordination Server.** Un Coordination Server centrale distribuisce le chiavi pubbliche e una network map che stabilisce quale nodo possiede quali indirizzi e route. Il Coordination Server vede i metadati (chi può comunicare con chi, quali route sono approvate), ma non il contenuto dei pacchetti WireGuard. Quando advertised una route, il nodo lo comunica al control plane; solo con l’approvazione la route diventa parte della network map ricevuta da tutti i nodi.

**Sul nodo router.** Affinché un nodo inoltri traffico per altri dispositivi, deve avere attivato l’inoltro IP e instradare i pacchetti tra l’interfaccia Tailscale e la rete locale. Per impostazione predefinita, Tailscale maschera il traffico inoltrato tramite NAT di origine (SNAT): i dispositivi di destinazione nella rete locale vedono come mittente l’indirizzo locale del nodo router, non l’indirizzo Tailscale del dispositivo che accede. È il caso più semplice, perché i pacchetti di risposta ritrovano così automaticamente la strada verso il router. Se disattiva SNAT, i dispositivi di destinazione vedono il vero indirizzo sorgente Tailscale, ma la rete locale deve allora sapere come instradare nuovamente l’intervallo Tailscale verso il router.

**Lato client.** Un dispositivo utilizza route di terzi solo se le accetta. Nei client grafici per Windows e macOS, l’accettazione delle route di subnet è preimpostata; su Linux viene attivata con `--accept-routes`. Quando il client accetta una route, la inserisce nella propria tabella di routing e la indirizza verso l’interfaccia Tailscale. I pacchetti destinati a un obiettivo in tale intervallo vengono quindi incapsulati in WireGuard e inviati al nodo router. Con l’Exit Node il meccanismo è lo stesso, ma qui la route predefinita punta all’Exit Node, motivo per cui tutto il traffico passa attraverso di esso.

**L’approvazione.** Il fatto che le route diventino effettive solo dopo l’approvazione è una caratteristica di sicurezza, non un passaggio superfluo: un nodo qualsiasi non dovrebbe poter attirare verso di sé, senza richiesta, il traffico di intere reti. L’approvazione può avvenire manualmente nell’amministrazione o automaticamente tramite `autoApprovers` nelle regole di accesso (ACL). Exit Node e route di subnet vengono approvati separatamente.

## Limiti

Entrambe le varianti hanno limiti che influenzano la scelta:

- **Il nodo router è un collo di bottiglia e un single point of failure.** Tutto il traffico per la rete advertised passa attraverso questo unico nodo, la sua crittografia WireGuard e la sua connettività. Per garantire la tolleranza ai guasti, più nodi possono advertised la stessa route; Tailscale ne usa quindi uno e cambia in caso di guasto.
- **SNAT nasconde la sorgente.** Con il NAT di origine predefinito, tutti gli accessi appaiono sotto l’indirizzo del nodo router. Per la registrazione o le regole di accesso sui dispositivi di destinazione che richiedono la vera sorgente, è necessario disattivare SNAT e configurare la route di ritorno nella rete locale.
- **Un Exit Node instrada davvero tutto.** Tutto il traffico passa attraverso il nodo, con le conseguenze corrispondenti su throughput, latenza e riservatezza. Il gestore dell’Exit Node vede il traffico nel punto in cui lascia il tailnet. Utilizzi come Exit Node solo nodi di cui si fida.
- **Le subnet sovrapposte sono un problema.** Se due sedi advertised lo stesso intervallo privato, ad esempio `192.168.1.0/24`, un client non riesce a distinguerli. Per questo Tailscale offre una riscrittura tramite IPv6 (`4via6`), che rende univoci gli intervalli.
- **Le chiavi in scadenza bloccano l’inoltro.** Se la chiave del nodo router scade, l’intera rete situata dietro di esso non è più raggiungibile. Per un nodo router permanente, disattivi la scadenza della chiave nell’amministrazione.

Per l’accesso mirato a servizi interni, il router di subnet è quasi sempre la scelta giusta: apre solo ciò che è necessario. Utilizzi l’Exit Node quando desidera instradare consapevolmente tutto il traffico in uscita attraverso una determinata sede.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornati i link alle Fonti con i nuovi indirizzi di Tailscale.

## Fonti

1.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): advertising delle route, approvazione, comportamento SNAT e alta disponibilità con più router.

2.  [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes): advertising della route predefinita, selezione sul client e accesso alla propria rete locale.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): interazione tra data plane WireGuard, Coordination Server e relay DERP.

4.  [WireGuard: panoramica del protocollo](https://www.wireguard.com/protocol/): la base crittografica del data plane, che Tailscale utilizza come trasporto.
