---
title: "Split tunneling VPN per Microsoft Teams: instradare il traffico multimediale al di fuori del tunnel"
navTitle: "Split tunneling Teams"
description: "Le chiamate Teams tramite VPN risentono di latenza, jitter e della deviazione attraverso il gateway VPN. L'articolo mostra quali reti e porte Microsoft gestiscono il traffico multimediale, perché lo split tunneling basato su IP è superiore all'esclusione per app e come implementarlo nelle VPN consumer, in WireGuard, OpenVPN e nei client enterprise."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 min di lettura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-per-microsoft-teams-instradare-il-traffico-multimediale-fuori-dal-tunnel"
translationId: "article-d15f1e7ff6af231c"
translationOf: vpn-split-tunneling-microsoft-teams
translationSourceHash: 9a96126164a12f1f411065507313733f451161a25c0cd051778cc306fd32ed4c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:25:58.017Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/vpn-split-tunneling-per-microsoft-teams-instradare-il-traffico-multimediale-fuori-dal-tunnel
---

Una chiamata Teams tramite connessione VPN spesso ha una qualità peggiore rispetto a una senza VPN: l'audio si interrompe, il video scatta e le condivisioni dello schermo si caricano con ritardo. La causa risiede di solito nella deviazione che il traffico in tempo reale compie attraverso il tunnel VPN, non in Teams stesso. Microsoft raccomanda quindi da anni di instradare direttamente su Internet il traffico multimediale di Teams, aggirando la VPN mediante split tunneling. Questo approccio funziona con praticamente qualsiasi prodotto VPN, dal client consumer al gateway enterprise; la configurazione differisce solo nei dettagli.

## Perché il traffico in tempo reale soffre nel tunnel

L'audio e il video di Teams utilizzano SRTP, un protocollo basato su UDP che dipende da bassa latenza e jitter ridotto. Microsoft indica come valori target un tempo di andata e ritorno inferiore a 100 ms verso il più vicino ingresso di rete Microsoft e un jitter inferiore a 30 ms. Un tunnel VPN peggiora entrambi i valori sotto diversi aspetti.

In primo luogo, il tunnel allunga il percorso: anziché raggiungere direttamente il punto di ingresso Microsoft geograficamente più vicino, il traffico passa prima dal gateway VPN, che può trovarsi nel data center del provider o dell'azienda, e solo da lì raggiunge Microsoft. In secondo luogo, il livello aggiuntivo di crittografia richiede tempo di elaborazione e aumenta l'overhead per pacchetto; il flusso multimediale è già crittografato con SRTP, mentre la crittografia VPN si aggiunge come secondo livello. In terzo luogo, il gateway VPN è un collo di bottiglia condiviso: nelle ore di punta tutti gli utenti condividono la sua larghezza di banda e i buffer dei pacchetti, generando proprio il jitter al quale il traffico in tempo reale è più sensibile. In quarto luogo, alcune configurazioni VPN bloccano completamente UDP o impongono TCP; Teams ripiega allora su TCP 443, peggiorando ulteriormente la qualità, poiché le ritrasmissioni TCP non sono adatte ai media in tempo reale.

Per il resto del traffico Teams (accesso, chat, accesso ai file), tutto ciò ha scarsissima importanza, perché non è sensibile al tempo reale. È quindi sufficiente escludere in modo mirato il traffico multimediale.

## Le reti e le porte rilevanti

Microsoft pubblica tutti gli endpoint Microsoft 365 in formato leggibile dalle macchine e li suddivide nelle categorie Optimize, Allow e Default. Per lo split tunneling è rilevante la categoria Optimize: comprende i pochi endpoint critici per la latenza con reti IP fisse, che insieme rappresentano la maggior parte del volume. Per i media Teams si tratta dell'ID endpoint 11 dell'elenco ufficiale. L'ID 12 comprende le stesse reti IPv4 e ulteriori reti IPv6 per TCP 443 e 80 nonché UDP 443, ma appartiene alla categoria Allow. La rete anch'essa indicata in precedenza `13.107.64.0/18` è stata rimossa da Microsoft da entrambe le voci nel 2024 (versione 2024053000 per l'ID 12, 2024073100 per l'ID 11):

| Rete | Protocollo e porte | Scopo |
|---|---|---|
| `52.112.0.0/14` | UDP da 3478 a 3481 (ID 11), TCP 443 (ID 12) | Media Teams e relay di trasporto |
| `52.122.0.0/15` | UDP da 3478 a 3481 (ID 11), TCP 443 (ID 12) | Media Teams e relay di trasporto |
| `2603:1063::/38` | UDP da 3478 a 3481 (ID 11), TCP 443 (ID 12) | Gli stessi servizi tramite IPv6 |

Le quattro porte UDP corrispondono alle classi multimediali audio (3478), video (3479 e 3480) e condivisione dello schermo (3481); TCP 443 è il percorso di fallback. Chi utilizza IPv6 dovrebbe escludere anche la rete IPv6, altrimenti una parte delle connessioni passerà nuovamente attraverso il tunnel.

Queste reti sono volutamente stabili: Microsoft annuncia le modifiche agli endpoint Optimize attraverso l'Endpoint Web Service e mantiene l'elenco ridotto, proprio affinché le aziende possano inserirlo nelle regole di routing e firewall. Ciononostante, un confronto occasionale con l'elenco ufficiale dovrebbe far parte della routine operativa.

## Basato su app o su IP: due approcci con punti di forza diversi

Molti client VPN offrono due tipi di split tunneling: eccezioni per applicazione oppure eccezioni per IP di destinazione.

L'esclusione dell'app sembra naturale, ma presenta due debolezze con Teams. Il nuovo Teams è un'applicazione WebView2: il processo principale si chiama `ms-teams.exe`, ma una parte del traffico passa attraverso `msedgewebview2.exe`. Chi esclude solo il processo principale non intercetta tutto il traffico; chi esclude anche WebView2 instrada al di fuori del tunnel anche il traffico di altre applicazioni WebView2, come il nuovo Outlook. E per Teams nel browser, l'esclusione dell'app non serve affatto, a meno di escludere l'intero browser, facendo così aggirare la VPN a tutto il traffico web.

L'esclusione basata su IP, invece, agisce a livello di rete ed è quindi indipendente dal fatto che il traffico provenga dall'app Teams, da WebView2 o da una scheda del browser. Esclude esattamente ciò che è critico per la latenza e mantiene nel tunnel accesso, chat e il resto del traffico web. Per Teams, l'approccio basato su IP è quindi la scelta migliore; l'esclusione dell'app è adatta come integrazione quando si desidera davvero che tutto il traffico Teams aggiri la VPN.

## Implementazione nei prodotti VPN più diffusi

Il principio è ovunque lo stesso: le due reti IPv4 (e, se necessario, la rete IPv6) vengono escluse dal tunnel, in modo che le route del sistema operativo per queste destinazioni puntino all'interfaccia fisica.

**VPN consumer (Proton VPN, NordVPN, Surfshark e simili):** I client Windows e Android offrono in genere una voce di menu come “Split Tunneling” con un elenco di esclusione per indirizzi IP o subnet. Inserire qui le reti in notazione CIDR e ristabilire la connessione VPN affinché le route abbiano effetto. Su macOS e iOS questa funzione manca presso la maggior parte dei provider, poiché le API di sistema non consentono lo split tunneling controllato dall'applicazione in questa forma.

**WireGuard:** WireGuard non conosce un elenco di esclusione, ma solo l'impostazione `AllowedIPs` che definisce ciò che entra nel tunnel. Le eccezioni si ottengono sostituendo `0.0.0.0/0` con l'elenco di tutte le reti che non contengono l'area da escludere. Nessuno calcola a mano questo elenco complementare; calcolatori online come WireGuard AllowedIPs Calculator usano `0.0.0.0/0` come base, le reti Microsoft come “Disallowed IPs” e forniscono la riga pronta per il file di configurazione.

**OpenVPN:** Con `redirect-gateway` attivo, prevalgono le route più specifiche. Due righe aggiuntive nella configurazione client instradano le reti Microsoft al di fuori del tunnel:

```text
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` indica il gateway predefinito della rete locale, non il gateway VPN.

**Client enterprise (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Qui l'azienda configura centralmente le eccezioni, in Cisco come elenco “Split Exclude” nella policy di gruppo, in GlobalProtect come “Exclude Access Route”. Microsoft documenta espressamente questa procedura come modello consigliato per il traffico Microsoft 365 e fornisce l'elenco Optimize attraverso l'Endpoint Web Service, così da mantenere aggiornate automaticamente le eccezioni. Chi, come dipendente, si trova dietro una VPN aziendale non può quindi impostare autonomamente l'eccezione, ma deve richiederla al team di rete; il documento Microsoft relativo costituisce la base argomentativa adatta.

**Strumenti integrati di Windows:** In una connessione VPN configurata con gli strumenti integrati di Windows in modalità split (`Set-VpnConnection -SplitTunneling $true`), solo le reti aggiunte tramite `Add-VpnConnectionRoute` entrano nel tunnel. Finché le reti Microsoft non vi compaiono, vengono instradate automaticamente in modo diretto; non è quindi necessaria un'esclusione esplicita.

## Valutazione della sicurezza: cosa aggira il tunnel

Lo split tunneling è un allentamento consapevole del principio di instradare tutto il traffico attraverso il tunnel. Prima dell'implementazione, occorre chiarire tre punti.

Il proprio indirizzo IP pubblico diventa visibile a Microsoft, poiché è proprio questo lo scopo: il flusso multimediale deve percorrere il tragitto più breve. Chi utilizza una VPN principalmente per nascondere la propria posizione rinuncia a questa protezione per le chiamate Teams. I contenuti restano inalterati, poiché SRTP crittografa il flusso multimediale end-to-end tra il client e l'infrastruttura Microsoft.

In ambito aziendale, il gateway di sicurezza centrale perde visibilità sul traffico escluso: ispezione TLS, firme IDS e analisi dei volumi non si applicano più a queste reti. Poiché l'eccezione è limitata a poche reti assegnate stabilmente a Microsoft con porte definite, Microsoft valuta questo rischio residuo come basso; gli endpoint Optimize sono curati proprio a questo scopo. Un'esclusione indiscriminata di intere applicazioni o addirittura del browser, invece, presenta una superficie di attacco significativamente maggiore e dovrebbe essere evitata in ambito aziendale.

Infine, il Kill Switch: alcuni client VPN applicano le eccezioni di split tunneling solo dopo aver ristabilito la connessione oppure si comportano diversamente con Kill Switch attivo. Dopo ogni modifica all'elenco di esclusione è quindi necessario ristabilire la connessione ed eseguire un test di controllo.

## Verifica: il traffico multimediale passa davvero direttamente?

È possibile verificare se l'eccezione viene applicata su due livelli. A livello di routing, PowerShell mostra quale interfaccia Windows seleziona per una destinazione nelle reti Microsoft:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Se qui compare l'interfaccia fisica (Ethernet o WLAN) anziché l'adattatore VPN, la route è corretta. A livello applicativo, Teams stesso fornisce la conferma: durante una chiamata, l'integrità della chiamata (in “Altre azioni” nella finestra della chiamata) mostra il tipo di connessione negoziato, il tempo di andata e ritorno e il tasso di perdita dei pacchetti. Un tempo di andata e ritorno che diminuisce sensibilmente dopo la modifica e il tipo di connessione UDP anziché TCP sono i due segnali di un'eccezione funzionante.

Se il traffico resta nel tunnel nonostante la route corretta, vale la pena controllare l'ordine degli adattatori di rete e le peculiarità del client: alcuni client VPN impongono nuovamente le proprie route con una metrica inferiore dopo ogni connessione, e un elenco di esclusione obsoleto si nota solo quando Microsoft aggiunge una rete. Il confronto con l'elenco ufficiale degli endpoint dovrebbe quindi seguire lo stesso ritmo delle altre verifiche di rete ricorrenti.

## Cronologia delle modifiche

- **25 ottobre 2026:** Corretto: Microsoft ha rimosso la rete 13.107.64.0/18 dall'elenco degli endpoint nel 2024; è stata eliminata dalla tabella e dall'esempio OpenVPN. Corretto anche: solo l'ID 11 appartiene alla categoria Optimize, mentre l'ID 12 appartiene alla categoria Allow.

## Fonti

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): elenco ufficiale degli endpoint; le reti per i media Teams si trovano nell'ID 11, nella categoria Optimize, e nell'ID 12 con le stesse reti nella categoria Allow.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): guida all'implementazione di Microsoft per VPN enterprise, inclusa la motivazione della valutazione del rischio e un esempio con le reti per i media Teams `52.112.0.0/14`, `52.122.0.0/15` e `2603:1063::/38`.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): i principi alla base dell'uscita Internet locale, inclusi i valori di latenza target per i media in tempo reale.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): esempio di un client consumer con split tunneling basato su IP e app in Windows e Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): calcolatore per l'elenco complementare quando le eccezioni devono essere gestite tramite AllowedIPs.

6.  [Microsoft 365 IP and URL web service: modifiche Worldwide](https://endpoints.office.com/changes/worldwide/2024010100?clientrequestid=b10c5ed1-bad1-445f-b386-b919946339a7): registro delle modifiche dell'elenco degli endpoint, inclusa la rimozione di `13.107.64.0/18` dagli ID 12 (versione 2024053000) e 11 (versione 2024073100).
