---
title: "Inoltro delle porte con netsh portproxy: raggiungere servizi interni tramite un jumphost"
navTitle: "netsh portproxy"
description: "Windows include un inoltro delle porte TCP integrato con netsh interface portproxy. Insieme a una VPN come Tailscale, consente di raggiungere dall'esterno un servizio interno, ad esempio l'interfaccia di un NAS, senza esporlo pubblicamente. Come configurare, proteggere e rimuovere l'inoltro e quali sono i suoi limiti: niente UDP, nessuna crittografia aggiuntiva, insidie legate a certificati e reindirizzamenti."
date: "2026-09-02"
kategorie: "Windows e rete"
timeToRead: "9 min di lettura"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "inoltro-porte-con-netsh-portproxy-raggiungere-servizi-interni-tramite-un-jumphost"
translationId: "article-236adcb4ae982572"
translationOf: windows-portproxy-portweiterleitung
translationSourceHash: 152d2f6d20774179128718ab4e218fcc0cd76cbcf0706060df7d60f30e012900
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:55:16.228Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/inoltro-porte-con-netsh-portproxy-raggiungere-servizi-interni-tramite-un-jumphost
---

# Inoltro delle porte con netsh portproxy: raggiungere servizi interni tramite un jumphost

Un servizio interno spesso resta in ascolto solo sulla rete locale: l'interfaccia web di un NAS, il pannello di una stampante, una pagina di amministrazione. Se volete accedervi dall'esterno senza esporre il servizio a Internet, vi serve un percorso attraverso un computer che veda entrambi i lati. Windows offre uno strumento integrato a questo scopo: `netsh interface portproxy` inoltra le connessioni TCP in ingresso verso un'altra destinazione. In combinazione con una VPN come Tailscale o WireGuard, un computer nella rete di destinazione diventa un jumphost attraverso il quale potete raggiungere il servizio interno.

Un esempio concreto: un NAS con l'interfaccia web su `10.0.0.245:5000` è raggiungibile solo nella rete locale. Nella stessa rete si trova un PC Windows raggiungibile anche tramite VPN. Se configurate su questo PC un inoltro di porta dal suo indirizzo VPN al NAS, potrete poi aprire l'interfaccia del NAS nel browser tramite l'indirizzo VPN del PC. Il servizio resta nella rete interna; solo il jumphost è raggiungibile tramite VPN.

## Come funziona portproxy

`portproxy` è un componente del servizio IP Helper (`iphlpsvc`). Il servizio accetta connessioni su una porta locale e le inoltra a una destinazione. Si tratta di un puro relay TCP a livello applicativo: non una regola NAT del firewall, bensì un processo che copia byte tra due connessioni. Se `iphlpsvc` non è in esecuzione, nessun inoltro funziona. Il servizio è presente per impostazione predefinita; il suo tipo di avvio dovrebbe essere impostato su automatico affinché l'inoltro sopravviva a un riavvio.

## Configurazione

Un inoltro richiede due passaggi: la regola portproxy e una regola firewall che consenta l'accesso al listener. Eseguite entrambi in un prompt dei comandi o in PowerShell con privilegi di amministratore.

Per prima cosa, l'inoltro. Si associa a un indirizzo e a una porta locali e punta a un IP e a una porta di destinazione:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `v4tov4` | IPv4 ascolta, IPv4 si connette; sono possibili anche: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Indirizzo locale su cui ascoltare; qui l'indirizzo VPN del jumphost, affinché le connessioni arrivino solo tramite VPN |
| `listenport` | Porta locale su cui ascoltare |
| `connectaddress` | IP di destinazione verso cui inoltrare (il servizio interno) |
| `connectport` | Porta di destinazione del servizio interno |

</details>

L'associazione all'indirizzo VPN anziché a `0.0.0.0` è la prima misura di sicurezza: il listener compare solo sull'interfaccia VPN, non su tutte le schede di rete del jumphost. La seconda misura di sicurezza è il firewall. Aprite la porta del listener esclusivamente per l'intervallo di indirizzi della vostra VPN, non per tutti gli indirizzi:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `-Direction Inbound` | Regola per il traffico in ingresso |
| `-Protocol TCP` | portproxy inoltra solo TCP, quindi TCP |
| `-LocalPort 5000` | La porta del listener della regola portproxy |
| `-RemoteAddress 100.64.0.0/10` | Sono ammesse solo origini di questo intervallo; qui l'intervallo Tailscale, altrimenti il blocco CIDR della vostra VPN |

</details>

## Verificare e utilizzare

Per prima cosa, verificate sul jumphost stesso se il servizio interno sia effettivamente raggiungibile e visualizzate l'inoltro attivo:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Se la destinazione risponde e la regola compare nell'elenco, testate dal vostro dispositivo remoto. Il servizio è ora raggiungibile tramite indirizzo e porta del jumphost:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Nel browser aprite quindi `http://100.100.10.10:5000`. Se vi servono più porte dello stesso servizio, ad esempio 5000 e 5001 per http e https, create una regola portproxy separata e la relativa autorizzazione firewall per ogni porta.

## Panoramica in stile manpage

I principali sottocomandi di `netsh interface portproxy`:

<details class="options-details">
<summary>Panoramica delle opzioni</summary>

| Comando | Scopo |
|---|---|
| `add v4tov4 …` | Creare un inoltro (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Visualizzare gli inoltri IPv4 attivi |
| `show all` | Visualizzare tutti gli inoltri di tutte le varianti di protocollo |
| `delete v4tov4 listenaddress=… listenport=…` | Rimuovere un inoltro |
| `reset` | Eliminare tutte le regole portproxy |

</details>

Le regole sono archiviate nel registro in `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` e sopravvivono a un riavvio. Sono visibili solo tramite `netsh` o direttamente nel registro, non nell'interfaccia grafica del firewall.

## Alternative

`portproxy` è pratico se il jumphost è già Windows e non volete installare nulla. Due alternative risolvono lo stesso problema con caratteristiche diverse.

Un tunnel SSH con inoltro locale (`ssh -L 5000:10.0.0.245:5000 benutzer@jumphost`) cifra il tratto fino al jumphost stesso e funziona su più piattaforme. Richiede un server SSH sul jumphost e resta attivo solo finché dura la sessione SSH.

Un subnet router Tailscale (`tailscale set --advertise-routes=10.0.0.0/24`) rende l'intera sottorete interna raggiungibile dai vostri dispositivi VPN. Potete quindi indirizzare il servizio interno direttamente al suo IP reale, senza un inoltro per porta. È la soluzione più diretta se volete raggiungere più dispositivi interni, ma richiede l'autorizzazione della route nella gestione di Tailscale.

## Limiti

Un inoltro di porta con portproxy risolve l'accesso, ma presenta limiti chiari che dovreste conoscere prima dell'uso:

- **Solo TCP.** `portproxy` inoltra esclusivamente TCP. Non può quindi gestire servizi che richiedono UDP (DNS, molti protocolli VPN e di gioco, alcune trasmissioni video).
- **Nessuna crittografia aggiuntiva.** L'inoltro copia i byte senza modificarli. La riservatezza del percorso è garantita esclusivamente dalla VPN attraverso cui raggiungete il jumphost. Su una rete di trasporto non cifrata, il traffico sarebbe privo di protezione.
- **Avviso del certificato con HTTPS tramite IP.** Se inoltrate un servizio HTTPS e lo richiamate tramite l'IP del jumphost, il certificato della destinazione non corrisponde all'indirizzo richiesto. Il browser avvisa. Può essere accettabile per un breve test, ma non per l'uso continuativo.
- **Reindirizzamenti e indirizzi assoluti.** Alcune interfacce web reindirizzano autonomamente al proprio nome host o a un'altra porta, oppure creano link assoluti con il proprio indirizzo interno. In tal caso, l'accesso tramite jumphost non funziona nonostante l'inoltro sia configurato. Questi servizi richiedono un vero reverse proxy anziché un semplice relay di porta.
- **Associazione a un indirizzo che deve esistere all'avvio.** Se la regola si associa a un determinato `listenaddress`, quell'indirizzo deve essere presente all'avvio del servizio. Se l'interfaccia VPN si attiva solo in seguito, l'associazione può fallire finché il servizio o la regola non vengono reimpostati.
- **Un percorso aggiuntivo verso la rete interna.** Ogni inoltro costituisce un percorso dall'esterno a un servizio interno. Limitate rigorosamente il firewall all'intervallo VPN, associatevi all'indirizzo VPN e rimuovete l'inoltro non appena non vi serve più.

## Rimuovere nuovamente

Al termine del lavoro, eliminate l'inoltro e la regola firewall:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Un inoltro di porta è uno strumento per un accesso mirato e limitato nel tempo, non un canale permanentemente aperto. Per l'uso continuativo di un servizio interno via Internet, un reverse proxy con certificato valido o una VPN con routing della sottorete rappresentano la soluzione più pulita.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: le route di sottorete vengono annunciate con tailscale set --advertise-routes anziché con tailscale up. Aggiornati i link delle Fonti ai nuovi indirizzi di Microsoft Learn e Tailscale.

## Fonti

1.  [netsh interface (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-interface): Riferimento ai comandi di `netsh interface` con la sintassi dei sottocomandi portproxy e delle varianti di protocollo.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps): Parametri della regola firewall, inclusa la limitazione agli intervalli di indirizzi tramite RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Rendere un'intera sottorete raggiungibile tramite VPN, come alternativa all'inoltro per porta; le route vengono annunciate con `tailscale set --advertise-routes`.
