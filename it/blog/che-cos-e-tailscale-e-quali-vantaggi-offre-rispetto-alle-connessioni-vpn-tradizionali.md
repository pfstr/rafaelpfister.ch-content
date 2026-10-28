---
title: "Che cos’è Tailscale e quali vantaggi offre rispetto alle connessioni VPN tradizionali?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale crea una VPN mesh basata su WireGuard, in cui i dispositivi si collegano direttamente invece di passare attraverso un concentratore VPN centrale. Come interagiscono Coordination Server, NAT traversal e relay DERP, quali sono i vantaggi rispetto alle VPN IPsec e SSL e quali dipendenze e limiti è opportuno conoscere prima dell’uso."
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
translationOf: tailscale-vorteile-vpn
translationSourceHash: f7642d64432f28cd0e53fd833c1accbd7a2a206550be04e07ace9cea8d23a04f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:21:50.718Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/it/blog/che-cos-e-tailscale-e-quali-vantaggi-offre-rispetto-alle-connessioni-vpn-tradizionali
---

# Che cos’è Tailscale e quali vantaggi offre rispetto alle connessioni VPN tradizionali?

Tailscale è un servizio VPN che collega i dispositivi a una rete privata, il cosiddetto Tailnet. Dal punto di vista tecnico si basa su WireGuard. La differenza rispetto a una VPN aziendale classica risiede nella topologia: i dispositivi stabiliscono tunnel cifrati direttamente tra loro (mesh). Non è necessario un concentratore VPN centrale attraverso il quale transita l’intero traffico. Rimane centrale solo la gestione: un Coordination Server distribuisce chiavi pubbliche, indirizzi e regole di accesso, ma non vede i dati utente.

Per iniziare sono sufficienti un account presso un Identity Provider (Microsoft, Google, GitHub, Apple o un provider OIDC) e il client su ciascun dispositivo. In genere non sono necessarie aperture di porte nel firewall. Tailscale è quindi diffuso sia per la rete domestica sia per l’accesso remoto a server e sistemi dei clienti.

## Come funziona una VPN tradizionale

Una classica VPN per accesso remoto funziona secondo il principio hub-and-spoke. Al perimetro della rete aziendale è presente un gateway VPN (firewall o appliance) raggiungibile da Internet. Il client sul notebook stabilisce un tunnel verso questo gateway; sono comuni IPsec/IKEv2 (UDP 500 e 4500), varianti SSL-VPN tramite TCP 443 o OpenVPN. Dopo l’autenticazione, il dispositivo riceve un indirizzo da un pool e rotte verso le reti interne.

Questo modello si è dimostrato valido per decenni, ma presenta caratteristiche strutturali che comportano oneri nell’operatività odierna:

- **Gateway raggiungibile pubblicamente:** il concentratore VPN deve essere raggiungibile da Internet ed è quindi un obiettivo privilegiato per gli attacchi. Negli ultimi anni le vulnerabilità nelle appliance VPN sono state ripetutamente sfruttate attivamente; nel gennaio 2024 l’autorità statunitense CISA ha persino ordinato, con l’Emergency Directive 24-01, la disconnessione dalla rete dei gateway Ivanti interessati.
- **Single point of failure e collo di bottiglia:** tutto il traffico passa attraverso il gateway. Se questo si guasta o la larghezza di banda è esaurita, tutti gli utenti ne risentono contemporaneamente.
- **Percorsi indiretti (hairpinning):** se due collaboratori in home office lavorano sullo stesso server nel cloud, il traffico passa prima dal data center e poi torna verso l’esterno.
- **Diritti di accesso grossolani:** dopo aver stabilito la connessione, spesso al dispositivo è aperto un intero segmento di rete. Regole granulari per utente e servizio sono possibili, ma raramente vengono gestite in modo coerente.
- **Onere site-to-site:** ogni sede aggiuntiva necessita di un proprio tunnel con parametri concordati (proposte Phase 1/Phase 2, chiavi precondivise o certificati, indirizzi IP pubblici fissi).

## Come è strutturato Tailscale

Tailscale separa il control plane dal data plane. Il control plane è il Coordination Server, che Tailscale gestisce come servizio cloud. Il data plane è costituito dai tunnel WireGuard tra i dispositivi.

| Componente | Funzione |
|---|---|
| Client (`tailscaled`) | Genera localmente la coppia di chiavi, crea tunnel WireGuard verso gli altri nodi, applica localmente le regole di accesso |
| Coordination Server | Autentica i dispositivi tramite l’Identity Provider, distribuisce chiavi pubbliche, indirizzi, impostazioni DNS e policy a tutti i nodi |
| Relay DERP | Inoltrano pacchetti cifrati quando non è possibile stabilire una connessione diretta |
| Peer Relay | Dispositivi propri nel Tailnet che fungono da relay con maggiore throughput; hanno la precedenza su DERP |

La chiave privata di un dispositivo non lascia mai il dispositivo. Il Coordination Server conosce solo le chiavi pubbliche e non può quindi decifrare il traffico. Ogni dispositivo riceve un indirizzo fisso dall’intervallo `100.64.0.0/10` (lo spazio di indirizzi per Carrier-Grade NAT) nonché un indirizzo IPv6 da `fd7a:115c:a1e0::/48`. Tramite MagicDNS, i dispositivi sono inoltre raggiungibili con il loro nome, ad esempio `nas` o `nas.tailnet-name.ts.net`.

### NAT traversal: perché non sono necessarie aperture di porte

La maggior parte dei dispositivi si trova dietro un router NAT o un firewall e non è direttamente raggiungibile dall’esterno. Tailscale risolve il problema con il NAT traversal: entrambe le parti rilevano tramite STUN il proprio indirizzo pubblico e la porta assegnata, scambiano queste informazioni attraverso il Coordination Server e inviano contemporaneamente pacchetti UDP l’una all’altra. I pacchetti in uscita aprono una voce di stato su entrambi i firewall, attraverso la quale possono poi entrare i pacchetti della controparte (UDP hole punching).

Se ciò non riesce, ad esempio con firewall restrittivi che bloccano UDP in uscita o con determinate forme di Carrier-Grade NAT, il traffico passa tramite un relay DERP via HTTPS. Anche in questo caso rimane cifrato end-to-end con WireGuard; il relay vede solo pacchetti cifrati. Il prezzo da pagare è una latenza maggiore e un throughput inferiore. I Peer Relay riducono questo svantaggio facendo assumere l’inoltro a un dispositivo proprio con una buona connettività.

## I vantaggi rispetto a una VPN tradizionale

| Criterio | VPN tradizionale | Tailscale |
|---|---|---|
| Topologia | Hub-and-spoke tramite un gateway centrale | Mesh, connessioni dirette tra i dispositivi |
| Porte in entrata | Il gateway deve essere raggiungibile da Internet | Non sono necessarie aperture di porte in entrata |
| Autenticazione | Account locali, RADIUS, certificati, spesso MFA separato | Accesso tramite l’Identity Provider esistente, incluso il relativo MFA |
| Diritti di accesso | Spesso per segmento di rete | Per utente, gruppo, dispositivo e porta in una policy centrale |
| Nuova sede | Tunnel site-to-site con parametri concordati | Installare client o Subnet Router |
| Guasto del centro | Nessun accesso per tutti | Le connessioni esistenti continuano a funzionare; nuovi dispositivi e modifiche delle policy attendono |
| Protocollo | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Minore superficie di attacco

Poiché i client stabiliscono le proprie connessioni in uscita, nessun dispositivo necessita di una porta aperta su Internet. Un server che deve essere raggiungibile solo tramite il Tailnet può associare i propri servizi esclusivamente all’interfaccia Tailscale. Risulta così invisibile agli scanner di porte su Internet. WireGuard stesso, con circa 4000 righe di codice kernel, è notevolmente più piccolo delle tipiche implementazioni IPsec o SSL-VPN e utilizza un insieme fisso di metodi moderni (Curve25519, ChaCha20-Poly1305, BLAKE2s). Non esiste una negoziazione di cipher suite, come quella che in IPsec porta regolarmente a configurazioni errate.

### Identità anziché indirizzo di rete

Ogni dispositivo nel Tailnet è associato a un utente o a un tag. L’accesso avviene tramite l’Identity Provider che già utilizzate; MFA e Conditional Access di Microsoft Entra ID si applicano quindi anche all’accesso di rete. Nuovi dispositivi possono entrare nel Tailnet solo dopo aver effettuato correttamente l’accesso presso l’Identity Provider e ogni chiave del dispositivo scade per impostazione predefinita dopo 180 giorni. Se un collaboratore lascia l’azienda, bloccate il suo account nell’Identity Provider; con il provisioning SCIM (dal piano Standard) l’utente viene disattivato automaticamente nel Tailnet e i suoi dispositivi perdono l’accesso.

### Regole di accesso secondo il principio del privilegio minimo

Per impostazione predefinita, in un nuovo Tailnet ogni dispositivo può raggiungere ogni altro dispositivo. Per l’uso in produzione, definite i diritti di accesso in un file di policy centrale (HuJSON). L’esempio seguente consente al gruppo degli amministratori SSH e HTTPS su tutti i server con il tag `tag:server`, mentre a tutti gli altri utenti consente solo HTTPS sul server intranet:

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
| `tagOwners` | Stabilisce chi può assegnare un tag ai dispositivi; i dispositivi con tag non appartengono a un utente, bensì al tag |
| `grants` | Elenco delle connessioni consentite; ciò che non è esplicitamente consentito viene bloccato |
| `src` | Origine della connessione: utente, gruppo, tag o `autogroup:member` (tutti gli utenti del Tailnet) |
| `dst` | Destinazione della connessione: tag, alias host, dispositivo o sottorete |
| `ip` | Protocolli e porte consentiti, ad esempio `tcp:22` o `*` per tutto |
| `hosts` | Nomi alias per indirizzi Tailnet o sottoreti utilizzati nelle regole |

</details>

Le regole vengono applicate localmente da ciascun client. Un pacchetto non consentito dalla policy viene scartato già dal dispositivo di destinazione. Il Coordination Server si limita a distribuire la policy.

### Connessioni dirette anziché percorsi indiretti

Poiché i tunnel esistono direttamente tra i dispositivi, il traffico segue il percorso più breve. Due dispositivi nello stesso ufficio comunicano localmente, un notebook in home office raggiunge direttamente un server cloud. Questo riduce la latenza e alleggerisce la connettività Internet della sede principale.

### Collegare reti esistenti

Non tutti i dispositivi possono eseguire un client Tailscale, ad esempio stampanti, sistemi NAS di vecchia generazione o controlli industriali. In questi casi un Subnet Router assume il ruolo di gateway: un server Linux nella rete di destinazione annuncia (advertise) la sottorete locale nel Tailnet e i dispositivi autorizzati la raggiungono tramite esso.

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
| `net.ipv4.ip_forward = 1` | Consente al kernel Linux di inoltrare pacchetti tra interfacce; senza questa impostazione nessun Subnet Router funziona |
| `sysctl -p <datei>` | Carica immediatamente l’impostazione, senza riavvio |
| `tailscale up` | Registra il dispositivo nel Tailnet; alla prima esecuzione appare un link per l’accesso |
| `--advertise-routes=<subnetz>` | Annuncia la sottorete indicata nel Tailnet; più sottoreti sono separate da virgole |
| `--advertise-tags=<tag>` | Assegna al dispositivo un tag a cui fanno riferimento le regole di accesso |

</details>

La rotta annunciata deve quindi essere approvata nella console di amministrazione, salvo che nella policy sia configurata un’approvazione automatica (`autoApprovers`). Analogamente, un dispositivo può essere utilizzato come Exit Node con `--advertise-exit-node`. I client che selezionano questo Exit Node inoltrano quindi attraverso di esso tutto il proprio traffico Internet, corrispondente alla modalità full tunnel di una VPN classica.

### Minore onere operativo

Il Coordination Server gestisce la rotazione delle chiavi, l’assegnazione degli indirizzi, DNS e routing. Una nuova sede necessita di un Subnet Router con accesso a Internet, ma non di un indirizzo IP pubblico fisso né del coordinamento dei parametri IPsec con la controparte. Per l’accesso remoto ai server sono inoltre disponibili Tailscale SSH (accesso tramite identità Tailnet senza chiavi SSH distribuite) e `tailscale serve` (condivisione di un servizio web locale nel Tailnet).

## Limiti e dipendenze

Tailscale sostituisce il gateway VPN classico, ma sposta parte della responsabilità su un servizio esterno. È opportuno verificare questi aspetti prima dell’introduzione.

| Tema | Aspetti da considerare |
|---|---|
| Dipendenza dal fornitore | Il control plane è un servizio cloud di Tailscale Inc. In caso di guasto, le connessioni esistenti continuano a funzionare, ma nuovi dispositivi, accessi e modifiche delle policy non sono possibili fino al ripristino |
| Metadati | Nomi dei dispositivi, indirizzi Tailnet, indirizzi IP pubblici, account utente e orari di connessione vengono trattati dal fornitore; i dati utente no |
| Fiducia nella distribuzione delle chiavi | Il Coordination Server determina quali chiavi pubbliche un dispositivo accetta. Tailnet Lock richiede inoltre una firma da dispositivi propri fidati |
| Codice sorgente | Il core del client è open source (BSD-3-Clause), il Coordination Server no. Headscale è un’alternativa open source self-hosted con funzionalità ridotte |
| Conflitti di indirizzi | `100.64.0.0/10` viene utilizzato anche da alcuni provider per Carrier-Grade NAT e da altri prodotti VPN; le sovrapposizioni causano problemi di routing |
| Coesistenza con altri client VPN | Un secondo client VPN con full tunnel può reindirizzare il traffico Tailscale. L’intervallo `100.64.0.0/10` e il servizio Tailscale devono esserne esclusi |
| Prestazioni tramite relay | Se non è possibile stabilire una connessione diretta, il throughput tramite DERP diminuisce sensibilmente; `tailscale netcheck` mostra se UDP in uscita funziona |
| Nessun sostituto del filtro web | Tailscale regola l’accesso alle risorse interne. Non si occupa del filtraggio dei contenuti e dell’ispezione del traffico Internet |

Per le aziende in Svizzera si applica la legge federale sulla protezione dei dati riveduta (revDSG). Poiché presso il fornitore vengono generati dati personali degli utenti (account, dispositivi, metadati di connessione), indicate Tailscale nel registro delle attività di trattamento, qualora la vostra azienda sia tenuta a tenerne uno (a partire da 250 collaboratori o in caso di trattamenti ad alto rischio). Verificate l’accordo sul trattamento dei dati (Data Processing Addendum) del fornitore e la base giuridica della comunicazione di dati all’estero ai sensi dell’art. 16 revDSG, ad esempio una certificazione ai sensi dello Swiss-U.S. Data Privacy Framework o clausole contrattuali standard. Se il trattamento dei dati presso il fornitore è escluso, Headscale rimane un control plane gestito autonomamente.

> **Nota UE:** per le sedi nell’UE si applicano il GDPR e le relative norme sul trasferimento verso paesi terzi (art. 44 e segg. GDPR). La verifica è simile nel merito; in questo caso è determinante l’EU-U.S. Data Privacy Framework.

## Costi

Il piano Personal è gratuito e comprende fino a sei utenti, un numero illimitato di dispositivi utente e 50 dispositivi con tag (situazione a ottobre 2026). Per l’uso aziendale, il piano Standard costa 8 USD e Premium 18 USD per utente al mese. Premium aggiunge, tra le altre cose, Network Flow Logs, log streaming e opzioni avanzate per Tailscale SSH. Enterprise è offerto su misura.

## Per chi è adatto Tailscale

Tailscale è particolarmente adatto quando utenti e risorse sono distribuiti: home office, server cloud presso più fornitori, piccole sedi distaccate senza indirizzo IP fisso o manutenzione remota presso i clienti. Per le piccole e medie imprese può sostituire completamente un gateway VPN. Negli ambienti più grandi, spesso funziona in parallelo alla VPN esistente, ad esempio per l’accesso amministrativo ai server, dove le regole di accesso granulari offrono il maggiore vantaggio.

È meno adatto quando i requisiti impongono un’infrastruttura interamente gestita in proprio e Headscale non copre le funzionalità necessarie, oppure quando l’intero traffico Internet deve essere filtrato centralmente. In questo caso rimane necessario un Secure Web Gateway che integri Tailscale.

Per effettuare una prova è sufficiente installare il client su due dispositivi e accedere con lo stesso account. Con `tailscale status` visualizzate quindi tutti i dispositivi nel Tailnet; con `tailscale ping <gerät>` verificate se viene utilizzata una connessione diretta o un relay.

## Cronologia delle modifiche

- **25 ottobre 2026:** aggiornati i link delle fonti ai nuovi indirizzi di Tailscale.

## Fonti

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): architettura con Coordination Server, mesh WireGuard e applicazione locale delle regole.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): spiegazione dettagliata di STUN, UDP hole punching e dei casi in cui è necessario un relay.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/docs/reference/derp-servers): funzione e cifratura dei server relay.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/docs/features/peer-relay): dispositivi propri come relay con priorità rispetto a DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/docs/features/access-control/grants): sintassi delle regole di accesso nel file di policy.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/docs/features/subnet-routers): configurazione dei Subnet Router, inclusi IP forwarding e approvazione delle rotte.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/docs/features/exit-nodes): inoltrare l’intero traffico Internet attraverso un dispositivo nel Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/docs/features/tailnet-lock): firma di nuovi dispositivi da parte di nodi propri fidati.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): piani e limiti, consultato il 1° ottobre 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): progettazione del protocollo e metodi crittografici utilizzati.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): codice sorgente del client sotto BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementazione open source del Coordination Server per l’uso autonomo.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): disposizione per la disconnessione dei gateway VPN Ivanti vulnerabili nel gennaio 2024.

14.  [Fedlex: Legge federale sulla protezione dei dati (LPD)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 sulla comunicazione di dati personali all’estero.
