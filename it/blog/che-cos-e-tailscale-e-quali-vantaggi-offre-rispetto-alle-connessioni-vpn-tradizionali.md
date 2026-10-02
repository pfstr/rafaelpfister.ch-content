---
title: "Che cos’è Tailscale e quali vantaggi offre rispetto alle connessioni VPN tradizionali?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale utilizza WireGuard per creare una VPN mesh in cui i dispositivi si connettono direttamente anziché tramite un concentratore VPN centrale. Come interagiscono Coordination Server, attraversamento NAT e relay DERP, quali sono i vantaggi rispetto alle VPN IPsec e SSL e quali dipendenze e limiti occorre conoscere prima dell’adozione."
date: "2026-10-01"
kategorie: "VPN e accesso remoto"
timeToRead: "11 min di lettura"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "che-cos-e-tailscale-e-quali-vantaggi-offre-rispetto-alle-connessioni-vpn-tradizionali"
translationId: "article-91fddf1e0239f5c4"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/it/blog/che-cos-e-tailscale-e-quali-vantaggi-offre-rispetto-alle-connessioni-vpn-tradizionali
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:05:20.329Z
translationReview: automatic
---

# Che cos’è Tailscale e quali vantaggi offre rispetto alle connessioni VPN tradizionali?

Tailscale è un servizio VPN che collega i dispositivi a una rete privata, la cosiddetta Tailnet. Dal punto di vista tecnico si basa su WireGuard. La differenza rispetto a una VPN aziendale classica risiede nella topologia: i dispositivi stabiliscono tunnel cifrati direttamente tra loro (mesh). Non è necessario un concentratore VPN centrale attraverso il quale transita tutto il traffico. Rimane centrale solo la gestione: un Coordination Server distribuisce chiavi pubbliche, indirizzi e regole di accesso, ma non vede i dati degli utenti.

Per iniziare sono sufficienti un account presso un Identity Provider (Microsoft, Google, GitHub, Apple o un provider OIDC) e il client su ogni dispositivo. In genere non sono necessarie aperture di porte nel firewall. Tailscale è quindi diffuso sia per la rete domestica sia per l’accesso remoto a server e sistemi dei clienti.

## Come funziona una VPN tradizionale

Una classica VPN di accesso remoto funziona secondo il principio hub-and-spoke. Al confine della rete aziendale si trova un gateway VPN (firewall o appliance) raggiungibile da Internet. Il client sul notebook stabilisce un tunnel verso questo gateway; sono comuni IPsec/IKEv2 (UDP 500 e 4500), varianti SSL-VPN tramite TCP 443 oppure OpenVPN. Dopo l’autenticazione, il dispositivo riceve un indirizzo da un pool e le route verso le reti interne.

Questo modello si è dimostrato valido per decenni, ma presenta caratteristiche strutturali che comportano oneri nell’operatività attuale:

- **Gateway raggiungibile pubblicamente:** il concentratore VPN deve essere raggiungibile da Internet ed è quindi un obiettivo privilegiato degli attacchi. Negli ultimi anni le vulnerabilità nelle VPN appliance sono state sfruttate attivamente più volte; nel gennaio 2024 l’autorità statunitense CISA ha persino ordinato, con l’Emergency Directive 24-01, la disconnessione dalla rete dei gateway Ivanti interessati.
- **Single point of failure e collo di bottiglia:** tutto il traffico passa attraverso il gateway. Se questo si guasta o la larghezza di banda è esaurita, ne risentono contemporaneamente tutti gli utenti.
- **Percorsi indiretti (hairpinning):** se due collaboratori in home office lavorano sullo stesso server nel cloud, il traffico passa prima dal data center e poi torna verso l’esterno.
- **Diritti di accesso troppo ampi:** dopo la connessione, il dispositivo ha spesso accesso a un intero segmento di rete. Regole granulari per utente e servizio sono possibili, ma raramente vengono mantenute con coerenza.
- **Onere site-to-site:** ogni sede aggiuntiva necessita di un proprio tunnel con parametri coordinati (proposte Phase 1/Phase 2, chiavi precondivise o certificati, indirizzi IP pubblici statici).

## Com’è strutturato Tailscale

Tailscale separa la control plane dalla data plane. La control plane è il Coordination Server, che Tailscale gestisce come servizio cloud. La data plane è costituita dai tunnel WireGuard tra i dispositivi.

| Componente | Compito |
|---|---|
| Client (`tailscaled`) | Genera localmente la coppia di chiavi, stabilisce tunnel WireGuard verso gli altri nodi e applica localmente le regole di accesso |
| Coordination Server | Autentica i dispositivi tramite l’Identity Provider, distribuisce chiavi pubbliche, indirizzi, impostazioni DNS e policy a tutti i nodi |
| Relay DERP | Inoltrano pacchetti cifrati quando non è possibile stabilire una connessione diretta |
| Peer Relays | Dispositivi propri nella Tailnet che fungono da relay con maggiore throughput; hanno priorità rispetto a DERP |

La chiave privata di un dispositivo non lascia mai il dispositivo. Il Coordination Server conosce solo le chiavi pubbliche e non può quindi decifrare il traffico. Ogni dispositivo riceve un indirizzo fisso dall’intervallo `100.64.0.0/10` (lo spazio di indirizzamento per il Carrier-Grade NAT) e un indirizzo IPv6 da `fd7a:115c:a1e0::/48`. Tramite MagicDNS, i dispositivi sono inoltre raggiungibili con il proprio nome, ad esempio `nas` o `nas.tailnet-name.ts.net`.

### Attraversamento NAT: perché non sono necessarie aperture di porte

La maggior parte dei dispositivi si trova dietro un router NAT o un firewall e non è direttamente raggiungibile dall’esterno. Tailscale risolve questo problema con l’attraversamento NAT: entrambi i lati rilevano tramite STUN il proprio indirizzo pubblico e la porta assegnata, scambiano queste informazioni attraverso il Coordination Server e inviano contemporaneamente pacchetti UDP l’uno all’altro. I pacchetti in uscita aprono una voce di stato su entrambi i firewall, attraverso la quale possono poi entrare i pacchetti della controparte (UDP hole punching).

Se ciò non riesce, ad esempio con firewall restrittivi che bloccano l’UDP in uscita o con determinate forme di Carrier-Grade NAT, il traffico passa attraverso un relay DERP via HTTPS. Anche in questo caso resta cifrato end-to-end con WireGuard; il relay vede solo pacchetti cifrati. Il prezzo da pagare è una maggiore latenza e un throughput inferiore. I Peer Relays riducono questo svantaggio facendo assumere l’inoltro a un dispositivo proprio con una buona connettività.

## I vantaggi rispetto a una VPN tradizionale

| Criterio | VPN tradizionale | Tailscale |
|---|---|---|
| Topologia | Hub-and-spoke tramite un gateway centrale | Mesh, connessioni dirette tra i dispositivi |
| Porte in ingresso | Il gateway deve essere raggiungibile da Internet | Non sono necessarie aperture di porte in ingresso |
| Autenticazione | Account locali, RADIUS, certificati, spesso MFA separato | Accesso tramite l’Identity Provider esistente, incluso il relativo MFA |
| Diritti di accesso | Spesso per segmento di rete | Per utente, gruppo, dispositivo e porta in una policy centrale |
| Nuova sede | Tunnel site-to-site con parametri coordinati | Installare un client o un Subnet Router |
| Guasto del centro | Nessun accesso per tutti | Le connessioni esistenti continuano; nuovi dispositivi e modifiche alla policy restano in attesa |
| Protocollo | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Minore superficie di attacco

Poiché i client stabiliscono le connessioni in uscita, nessun dispositivo necessita di una porta aperta su Internet. Un server che deve essere raggiungibile solo tramite la Tailnet può associare i propri servizi esclusivamente all’interfaccia Tailscale. Per gli scanner di porte su Internet risulta quindi invisibile. WireGuard stesso, con circa 4000 righe di codice kernel, è notevolmente più piccolo delle tipiche implementazioni IPsec o SSL-VPN e utilizza un insieme fisso di meccanismi moderni (Curve25519, ChaCha20-Poly1305, BLAKE2s). Non esiste una negoziazione delle cipher suite, che con IPsec porta regolarmente a configurazioni errate.

### Identità anziché indirizzo di rete

Ogni dispositivo nella Tailnet è associato a un utente o a un tag. L’accesso avviene tramite l’Identity Provider già utilizzato; MFA e Conditional Access di Microsoft Entra ID si applicano pertanto anche all’accesso alla rete. I nuovi dispositivi possono unirsi alla Tailnet solo dopo un’autenticazione riuscita presso l’Identity Provider e ogni chiave di dispositivo scade per impostazione predefinita dopo 180 giorni. Se un collaboratore lascia l’azienda, bloccate il suo account nell’Identity Provider; con il provisioning SCIM (dal piano Standard) l’utente viene disattivato automaticamente nella Tailnet e i suoi dispositivi perdono l’accesso.

### Regole di accesso secondo il principio del privilegio minimo

Per impostazione predefinita, in una nuova Tailnet ogni dispositivo può raggiungere ogni altro dispositivo. Per l’uso in produzione, definite i diritti di accesso in un file di policy centrale (HuJSON). L’esempio seguente consente al gruppo degli amministratori SSH e HTTPS su tutti i server con il tag `tag:server`, mentre a tutti gli altri utenti consente solo HTTPS verso il server Intranet:

```json
{
  "groups": {
    "group:admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:server": ["group:admins"]
  },
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "ip":  ["tcp:22", "tcp:443"]
    },
    {
      "src": ["autogroup:member"],
      "dst": ["intranet"],
      "ip":  ["tcp:443"]
    }
  ],
  "hosts": {
    "intranet": "100.101.102.103"
  }
}
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `groups` | Definisce gruppi di utenti; i membri sono indicati tramite il loro indirizzo di accesso presso l’Identity Provider |
| `tagOwners` | Stabilisce chi può assegnare un tag ai dispositivi; i dispositivi con tag non appartengono a un utente, ma al tag |
| `grants` | Elenco delle connessioni consentite; ciò che non è espressamente consentito viene bloccato |
| `src` | Origine della connessione: utente, gruppo, tag o `autogroup:member` (tutti gli utenti della Tailnet) |
| `dst` | Destinazione della connessione: tag, alias host, dispositivo o subnet |
| `ip` | Protocolli e porte consentiti, ad esempio `tcp:22` oppure `*` per tutto |
| `hosts` | Nomi alias per indirizzi o subnet della Tailnet utilizzati nelle regole |

</details>

Le regole vengono applicate localmente da ogni client. Un pacchetto non consentito dalla policy viene scartato già dal dispositivo di destinazione. Il Coordination Server distribuisce soltanto la policy.

### Connessioni dirette anziché percorsi indiretti

Poiché i tunnel esistono direttamente tra i dispositivi, il traffico segue il percorso più breve. Due dispositivi nello stesso ufficio comunicano localmente, un notebook in home office raggiunge direttamente un server cloud. Questo riduce la latenza e alleggerisce la connettività Internet della sede principale.

### Collegare reti esistenti

Non tutti i dispositivi possono eseguire un client Tailscale, ad esempio stampanti, sistemi NAS meno recenti o controlli industriali. In questi casi un Subnet Router assume il ruolo di gateway: un server Linux nella rete di destinazione annuncia la subnet locale nella Tailnet (advertise) e i dispositivi autorizzati la raggiungono tramite esso.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
echo 'net.ipv4.ip_forward = 1' | \
  sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale up \
  --advertise-routes=192.168.10.0/24 \
  --advertise-tags=tag:server
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Scarica lo script di installazione ufficiale e configura il repository dei pacchetti della distribuzione |
| `net.ipv4.ip_forward = 1` | Consente al kernel Linux di inoltrare pacchetti tra interfacce; senza questa impostazione nessun Subnet Router può funzionare |
| `sysctl -p <datei>` | Carica immediatamente l’impostazione, senza riavvio |
| `tailscale up` | Registra il dispositivo nella Tailnet; alla prima esecuzione appare un link di accesso |
| `--advertise-routes=<subnetz>` | Annuncia la subnet indicata nella Tailnet; più subnet sono separate da virgole |
| `--advertise-tags=<tag>` | Assegna al dispositivo un tag a cui fanno riferimento le regole di accesso |

</details>

La route annunciata deve essere successivamente approvata nella console di amministrazione, a meno che nella policy non sia configurata un’approvazione automatica (`autoApprovers`). Analogamente, un dispositivo può funzionare come Exit Node con `--advertise-exit-node`. I client che selezionano questo Exit Node inoltrano quindi attraverso di esso tutto il proprio traffico Internet, equivalente alla modalità full tunnel di una VPN classica.

### Minore onere operativo

Il Coordination Server gestisce la rotazione delle chiavi, l’assegnazione degli indirizzi, DNS e routing. Una nuova sede necessita di un Subnet Router con accesso a Internet, ma non di un indirizzo IP pubblico statico né del coordinamento dei parametri IPsec con la controparte. Per l’accesso remoto ai server sono inoltre disponibili Tailscale SSH (accesso tramite identità Tailnet senza chiavi SSH distribuite) e `tailscale serve` (condivisione di un servizio web locale nella Tailnet).

## Limiti e dipendenze

Tailscale sostituisce il classico gateway VPN, ma trasferisce parte della responsabilità a un servizio esterno. Prima dell’introduzione dovreste verificare questi punti.

| Tema | Aspetti da considerare |
|---|---|
| Dipendenza dal fornitore | La control plane è un servizio cloud di Tailscale Inc. In caso di guasto, le connessioni esistenti continuano; nuovi dispositivi, accessi e modifiche alla policy non sono possibili fino al ripristino |
| Metadati | Nomi dei dispositivi, indirizzi Tailnet, indirizzi IP pubblici, account utente e orari di connessione vengono trattati dal fornitore; i dati degli utenti no |
| Fiducia nella distribuzione delle chiavi | Il Coordination Server determina quali chiavi pubbliche un dispositivo accetta. Tailnet Lock richiede inoltre una firma da parte di dispositivi propri fidati |
| Codice sorgente | Il nucleo del client è open source (BSD-3-Clause), il Coordination Server no. Headscale è un’alternativa open source self-hosted con funzionalità ridotte |
| Conflitti di indirizzi | `100.64.0.0/10` viene utilizzato anche da alcuni provider per il Carrier-Grade NAT e da altri prodotti VPN; le sovrapposizioni causano problemi di routing |
| Coesistenza con altri client VPN | Un secondo client VPN con full tunnel può deviare il traffico Tailscale. L’intervallo `100.64.0.0/10` e il servizio Tailscale devono esserne esclusi |
| Prestazioni tramite relay | Se non è possibile stabilire una connessione diretta, il throughput tramite DERP diminuisce sensibilmente; `tailscale netcheck` mostra se UDP verso l’esterno funziona |
| Non sostituisce il filtraggio web | Tailscale regola l’accesso alle risorse interne. Non effettua il filtraggio dei contenuti né l’ispezione del traffico Internet |

Per le aziende in Svizzera si applica la legge federale sulla protezione dei dati riveduta (nLPD). Poiché presso il fornitore vengono trattati dati personali degli utenti (account, dispositivi, metadati di connessione), includete Tailscale nel registro delle attività di trattamento, se la vostra azienda è tenuta a mantenerne uno (a partire da 250 collaboratori o per trattamenti ad alto rischio). Verificate l’accordo sul trattamento dei dati (Data Processing Addendum) del fornitore e la base giuridica per la comunicazione all’estero ai sensi dell’art. 16 nLPD, ad esempio una certificazione nell’ambito dello Swiss-U.S. Data Privacy Framework oppure clausole contrattuali standard. Se è escluso un trattamento dei dati presso il fornitore, Headscale rimane una control plane gestita autonomamente.

> **Nota UE:** per le filiali nell’UE si applicano il GDPR e le relative norme sul trasferimento verso paesi terzi (art. 44 e segg. GDPR). La verifica è simile nel merito; in questo caso è determinante l’EU-U.S. Data Privacy Framework.

## Costi

Il piano Personal è gratuito e comprende fino a sei utenti, un numero illimitato di dispositivi utente e 50 dispositivi con tag (situazione a ottobre 2026). Per l’uso aziendale, il piano Standard costa 8 USD e Premium 18 USD per utente al mese. Premium aggiunge, tra le altre cose, Network Flow Logs, log streaming e opzioni avanzate per Tailscale SSH. Enterprise è disponibile su misura.

## Per chi è adatto Tailscale

Tailscale è particolarmente adatto quando utenti e risorse sono distribuiti: home office, server cloud presso più fornitori, piccole sedi esterne senza indirizzo IP statico o assistenza remota presso i clienti. Per le piccole e medie imprese può sostituire completamente un gateway VPN. In ambienti più grandi viene spesso utilizzato in parallelo alla VPN esistente, ad esempio per l’accesso amministrativo ai server, dove le regole di accesso granulari offrono il massimo beneficio.

È meno adatto se le prescrizioni richiedono un’infrastruttura interamente gestita in proprio e Headscale non copre le funzionalità necessarie, oppure se tutto il traffico Internet deve essere filtrato centralmente. In questo caso resta necessario un Secure Web Gateway che integri Tailscale.

Per provarlo basta installare il client su due dispositivi e accedere con lo stesso account. Con `tailscale status` vedrete quindi tutti i dispositivi nella Tailnet, mentre con `tailscale ping <gerät>` potrete verificare se viene utilizzata una connessione diretta o un relay.

## Fonti

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): architettura con Coordination Server, mesh WireGuard e applicazione locale delle regole.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): spiegazione dettagliata di STUN, UDP hole punching e dei casi in cui è necessario un relay.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): funzione e cifratura dei server relay.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): dispositivi propri come relay con priorità rispetto a DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): sintassi delle regole di accesso nel file di policy.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): configurazione dei Subnet Router, incluso l’IP forwarding e l’approvazione delle route.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): inoltrare tutto il traffico Internet tramite un dispositivo nella Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): firma dei nuovi dispositivi da parte di nodi propri e fidati.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): piani e limiti, consultato il 1° ottobre 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): progettazione del protocollo e meccanismi crittografici utilizzati.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): codice sorgente del client sotto BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementazione open source del Coordination Server per la gestione autonoma.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): disposizione di disconnettere i gateway VPN Ivanti vulnerabili nel gennaio 2024.

14.  [Fedlex: Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 sulla comunicazione di dati personali all’estero.
