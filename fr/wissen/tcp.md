---
title: "TCP : états, flux de données et diagnostic"
blatt: "tcp"
description: "TCP comme base de transport pour les services de messagerie et d’annuaire : modèle de socket, états de connexion, numéros de séquence, accusés de réception, contrôle de flux et de congestion, temporisateurs, erreurs typiques et outils de diagnostic pour les administrateurs."
fakten:
  - label: Nom complet
    wert: Transmission Control Protocol
    href: https://datatracker.ietf.org/doc/html/rfc9293
  - label: Standard
    wert: RFC 9293 · STD 7
    href: https://www.rfc-editor.org/info/rfc9293
  - label: Modèle de service
    wert: flux d’octets fiable, ordonné et bidirectionnel
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-2
  - label: Point de terminaison
    wert: adresse IP et numéro de port sur 16 bits
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Connexion
    wert: paire unique composée d’un socket local et d’un socket distant
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: En-tête
    wert: au moins 20 octets plus options
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Fiabilité
    wert: numéros de séquence, ACK cumulatifs et retransmission
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.3
  - label: Contrôle de flux
    wert: fenêtre de réception rwnd
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.4
  - label: Contrôle de congestion
    wert: cwnd · Slow Start · Congestion Avoidance
    href: https://datatracker.ietf.org/doc/html/rfc5681
  - label: Plages de ports
    wert: 0–1023 · 1024–49151 · 49152–65535
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Chiffrement
    wert: aucun ; protection par TLS, IPsec ou TCP-AO
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-7
  - label: Diagnostic
    wert: test de port · état du socket · capture de paquets
    href: https://www.tcpdump.org/manpages/tcpdump.1.html
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 6c749be6d80debccf17ceb4386ba0e298c11deedf855db326e7d20da9593fc05
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:33:26.976Z
translationReview: automatic
---

# TCP : états, flux de données et diagnostic

TCP transforme le transport de paquets non fiable d’IP en un flux de données bidirectionnel et ordonné pour les applications. L’application écrit des octets dans un socket ; TCP les segmente, numérote leur position, confirme les plages reçues et retransmet les lacunes. SMTP, IMAP, POP3, LDAP et HTTPS reposent sur cette garantie. Lorsqu’une telle application « bloque », l’administrateur doit donc d’abord déterminer si TCP attend encore un accusé de réception ou si le protocole applicatif est déjà bloqué.

L’article suit le cycle de vie d’une connexion : socket et listener, handshake à trois voies, transfert de données, contrôle de flux et de congestion, fermeture et TIME-WAIT. Le diagnostic, la planification de capacité, la surveillance et le durcissement sont ensuite déduits de ce déroulement normal.

TCP ne connaît toutefois ni les messages, ni les utilisateurs, ni les certificats ou les adresses e-mail. Il ne conserve pas non plus les limites applicatives : deux opérations d’écriture consécutives peuvent être regroupées dans un segment ou réparties sur plusieurs segments. Un handshake TCP réussi prouve donc uniquement que deux points de terminaison TCP ont pu établir une connexion. L’apparition ultérieure d’une bannière SMTP, la négociation de TLS ou la réussite d’une liaison LDAP dépendent de l’application de niveau supérieur.

Pour les administrateurs, TCP est moins pertinent comme modèle en couches que comme **système d’états et de signaux**. SYN, ACK, FIN et RST indiquent à quel point de passage une connexion échoue. Les états de socket, les retransmissions, les fenêtres de réception et les temporisateurs expliquent pourquoi un service peut être accessible, mais lent ou apparemment figé.

L’explication suit une connexion TCP depuis le premier SYN jusqu’au flux d’octets ordonné, puis à la fermeture ou à la réinitialisation. Les fenêtres, retransmissions, surcharges et diagnostics sont expliqués dans les phases correspondantes.

## Modèle de socket et de connexion

Avant que les paquets circulent, une application lie un socket à une adresse et à un port. Un serveur écoute par exemple sur TCP 25 ; chaque connexion acceptée est ensuite identifiée de manière univoque par l’adresse source, le port source, l’adresse de destination et le port de destination. Des milliers de clients peuvent donc utiliser le même listener, à condition que leur adresse source ou leur port éphémère diffère. Pour le diagnostic, « port 25 » ne suffit donc pas : la paire de connexion complète est toujours nécessaire ([RFC 9293, section 3.1](https://datatracker.ietf.org/doc/html/rfc9293#section-3.1)).

Sur un serveur, un processus lie un socket à une adresse locale et le place dans l’état **LISTEN**. Le choix de l’adresse de liaison est important sur le plan opérationnel :

| Liaison | Visibilité | Conséquence typique |
|---|---|---|
| `127.0.0.1:25` ou `::1:25` | hôte local uniquement | la surveillance sur le serveur ne révèle rien ; les systèmes distants n’obtiennent aucune connexion |
| `0.0.0.0:25` | toutes les adresses IPv4 locales | le listener suit les adresses IPv4 supplémentaires, dans la mesure où le pare-feu et le routage le permettent |
| `[::]:25` | joker IPv6 | le comportement double pile dépend du système d’exploitation et de l’option `IPV6_V6ONLY` |
| IP de service concrète | cette adresse uniquement | séparation contrôlée, mais sensible aux changements d’IP ou de cluster |

Le port serveur est généralement défini par un [enregistrement IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) ; le client choisit habituellement un port éphémère dans la plage gérée par le système d’exploitation. Les pare-feux, passerelles NAT et équilibreurs de charge doivent donc autoriser non seulement le port de destination connu, mais aussi le chemin de retour complet vers le port source dynamique.

### Bloc de contrôle TCP

Pour chaque connexion, la pile TCP conserve un **Transmission Control Block** (TCB). Il contient notamment l’état, les sockets locaux et distants, les séquences d’envoi et de réception, les fenêtres, les temporisateurs et les paramètres de sécurité. Le TCB explique pourquoi une connexion conserve un état et pourquoi les équilibreurs de charge ou pare-feux doivent fonctionner avec un chemin de retour symétrique : si un segment de réponse atteint un autre nœud sans état correspondant, celui-ci peut le rejeter ou répondre par RST.

## Établissement de la connexion, transfert de données et fermeture

L’ouverture active marque le début de l’état visible de la connexion. Le client envoie SYN avec son numéro de séquence initial, le listener répond avec SYN+ACK et son propre numéro de séquence, puis le client confirme avec ACK. Ces trois segments négocient déjà des options telles que Maximum Segment Size, Window Scale, Selective Acknowledgment et Timestamps. Si un paquet manque, si la réponse emprunte un autre chemin ou si un équipement intermédiaire modifie les options, le handshake indique déjà où l’accessibilité et les performances ultérieures se séparent.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813" title="Interaktive Infografik: TCP-Verbindungslebenszyklus und Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813">Ouvrir l’infographie sur le cycle de vie d’une connexion TCP</a>
</iframe>

Après le handshake, la connexion se trouve dans l’état **ESTABLISHED**. Les deux côtés peuvent envoyer des données indépendamment l’un de l’autre et fermer leur sens d’envoi avec FIN. Une fermeture ordonnée nécessite donc typiquement un FIN et un ACK dans chaque direction. Le côté qui envoie le dernier ACK reste dans l’état **TIME-WAIT**, afin que les segments retardés d’une ancienne connexion ne se retrouvent pas dans une nouvelle connexion utilisant la même paire de sockets ([RFC 9293, sections 3.3.2 et 3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2)).

Un **RST** n’est pas une fermeture ordonnée. Il interrompt la connexion et abandonne les états en attente. Lors de l’établissement de la connexion, un RST valide signifie généralement que l’hôte cible est joignable, mais que le socket n’est pas accepté. Après le transfert de données, un RST peut provenir de l’application, du système d’exploitation ou d’un nœud intermédiaire avec état. La position du RST dans la capture de paquets est donc plus importante que le message d’erreur général « connection reset ».

### Les états comme indicateurs opérationnels

| État | Signification technique | Ce qu’une accumulation suggère |
|---|---|---|
| `LISTEN` | un socket passif attend SYN | état serveur attendu ; l’absence de listener indique un problème de service ou de liaison |
| `SYN-SENT` | SYN local envoyé, réponse correspondante absente | filtre, route, adresse incorrecte ou homologue surchargé |
| `SYN-RECEIVED` | SYN accepté, dernier ACK absent | SYN flood, chemin de retour asymétrique, problème client ou pare-feu |
| `ESTABLISHED` | espaces de séquence synchronisés | normal ; de nombreuses connexions longues peuvent être souhaitées, par exemple avec IMAP ou HTTP/2 |
| `FIN-WAIT-1/2` | le côté local a fermé | l’homologue ou l’application termine son sens avec retard |
| `CLOSE-WAIT` | FIN distant reçu, l’application locale n’a pas encore fermé | souvent une fuite de ressources ou un thread applicatif bloqué |
| `LAST-ACK` | FIN local envoyé après la fermeture distante | ACK de l’homologue absent ou chemin de retour perturbé |
| `TIME-WAIT` | l’ancien quadruplet est protégé avant réutilisation | normal du côté qui ferme activement ; problématique uniquement avec une pression sur les ports ou les TCB |

Les noms des états sont définis dans [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2). Un nombre élevé n’est pas une erreur en soi. L’âge, le taux de croissance, les homologues concernés et le rapport avec le taux de connexion, les descripteurs de fichiers, la plage de ports éphémères et les limites applicatives sont déterminants.

## Numéros de séquence, ACK et retransmission

Après le handshake, TCP compte le flux de données octet par octet. Le numéro de séquence désigne le premier octet d’un segment ; l’ACK indique le prochain octet attendu et confirme ainsi tout ce qui le précède. Les paquets peuvent être perdus, arriver en double ou changer d’ordre. Le récepteur peut les mettre en mémoire tampon, mais ne transmet à l’application qu’un flux sans lacune. C’est précisément pourquoi une seule plage manquante peut bloquer une application apparemment inactive, même si des paquets ultérieurs sont déjà arrivés à l’hôte.

Lorsqu’un segment ou son ACK est perdu, l’expéditeur peut détecter la plage manquante de deux façons :

1. **Retransmission Timeout (RTO) :** un temporisateur est dérivé du Round-Trip Time lissé et de sa variation. À son expiration, le plus ancien segment non confirmé est renvoyé. [RFC 6298](https://datatracker.ietf.org/doc/html/rfc6298) définit le calcul, les valeurs minimales et le backoff exponentiel.
2. **Fast Retransmit :** plusieurs ACK dupliqués indiquent une lacune dans le flux de données. L’expéditeur peut renvoyer la plage probablement perdue sans attendre le RTO ([RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681#section-3.2)).

Une retransmission ne prouve pas automatiquement une perte de paquets sur le réseau. Une capture peut avoir été réalisée sur un seul côté, des paquets peuvent manquer en raison de pertes de capture ou révéler une retransmission parasite. Wireshark indique explicitement que son analyse TCP est une interprétation dérivée du flux de paquets observé ([Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)). Pour une conclusion fiable, les captures du client, du serveur et d’un point intermédiaire pertinent doivent donc être corrélées autant que possible.

## Contrôle de flux et contrôle de congestion

La fiabilité seule n’empêche pas un expéditeur rapide de surcharger le récepteur ou le réseau. TCP limite donc les octets non confirmés simultanément à l’aide de deux fenêtres, dont la plus petite est finalement déterminante :

- La **fenêtre de réception** `rwnd` protège le récepteur. Elle décrit le tampon actuellement disponible. Lorsque le récepteur annonce une Zero Window, l’expéditeur doit attendre, hormis les Window Probes périodiques. Une Zero Window durable n’est souvent pas un problème réseau, mais signifie que l’application réceptrice ne lit pas assez rapidement les données de son socket.
- La **Congestion Window** `cwnd` protège le chemin. Elle est gérée localement par l’expéditeur et ajustée selon les accusés de réception et les signaux de perte. [RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681) décrit Slow Start, Congestion Avoidance, Fast Retransmit et Fast Recovery.

C’est approximativement la plus petite de `rwnd` et `cwnd` qui s’applique. Une bande passante élevée ne garantit donc pas à elle seule un débit élevé. Sur un chemin présentant un produit bande passante-latence important, une quantité suffisante de données non confirmées doit être autorisée. L’option Window Scale étend la fenêtre de 16 bits ; les Timestamps prennent notamment en charge la mesure RTT et la protection contre les anciens numéros de séquence ([RFC 7323](https://datatracker.ietf.org/doc/html/rfc7323)). Les deux options sont négociées lors du handshake et ne peuvent pas être ajoutées ultérieurement à la même connexion.

Pour les transactions SMTP courtes, le nombre d’allers-retours domine souvent : handshake TCP, handshake TLS facultatif et plusieurs réponses de protocole. Pour les pièces jointes volumineuses et les flux de réplication, RTT, perte, fenêtres, offloading et algorithme de contrôle de congestion deviennent également pertinents. Un problème de débit n’est donc un problème de « bande passante » qu’après avoir exclu le temps de handshake, les pauses applicatives, les Zero Windows et les retransmissions.

## Temporisateurs, keepalives et délais d’inactivité

Lorsqu’aucune réponse immédiate n’arrive, les temporisateurs déterminent l’étape suivante. Le Retransmission Timeout déclenche les répétitions de données, le Persist Timer maintient une connexion malgré une Zero Window et TIME-WAIT empêche les anciens segments d’atteindre une nouvelle connexion. Les systèmes d’exploitation et les applications ajoutent le keepalive ainsi que le User Timeout. Ceux qui ne journalisent que « timeout » perdent ces distinctions et, avec elles, la véritable cause de l’erreur.

Les keepalives TCP sont souvent désactivés par défaut ou ne commencent qu’après de longues périodes d’inactivité. Ils ne remplacent pas un contrôle de santé côté application. Un keepalive réussi indique uniquement que le pair TCP répond ; il ne prouve pas qu’un worker SMTP traite des messages ou qu’une requête LDAP peut être servie. À l’inverse, les pare-feux et équilibreurs de charge coupent souvent les connexions inactives avant les systèmes finaux. Un socket apparemment valide reste alors d’un côté jusqu’à ce que la prochaine tentative de données déclenche un timeout ou un RST.

Pour les connexions IMAP, les pools LDAP ou les API de longue durée, quatre valeurs doivent donc correspondre : délai d’inactivité de l’application, intervalle de keepalive TCP, délai d’inactivité de chaque nœud intermédiaire et logique de reconnexion du client. Le plus petit délai non annoncé détermine le comportement observé.

## Diagnostic du socket au paquet

La recherche d’erreur devient fiable dès lors qu’elle décrit la même connexion que l’application : hôte source, nom cible, adresse résolue, port cible, moment et protocole attendu. Un ping ne remplace pas ces informations. ICMP peut être bloqué alors que TCP fonctionne ; inversement, une réponse Echo ne prouve ni l’existence d’un listener ni l’état d’un pare-feu avec état.

### Accessibilité et listener

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Verbindungstest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection -ComputerName mail.example.ch -Port 25 -InformationLevel Detailed

Get-NetTCPConnection -State Listen -LocalPort 25 |
  Select-Object LocalAddress, LocalPort, OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mail.example.ch 25

sudo ss -ltnp 'sport = :25'</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) combine sous Windows la résolution de nom, l’adresse cible, le test de port et le diagnostic de route facultatif. Sous Unix, [`nc`](https://man.openbsd.org/nc.1) ouvre de manière ciblée des connexions TCP ou UDP ; [`telnet`](https://man.openbsd.org/telnet.1) peut également ouvrir un port TCP, mais traite le protocole Telnet et ne constitue pas un remplacement entièrement transparent. [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) lit les informations de socket du noyau Linux et peut filtrer les états, temporisateurs, files d’attente et processus. Un test de connexion réussi ouvre et ferme une véritable connexion TCP ; sur les services de production, il doit donc être utilisé de manière ciblée et non comme scan de ports à haute fréquence.

### Capture de paquets et filtres d’état

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add MailTcp -t TCP -p 25
pktmon start --capture --pkt-size 0 --file-name C:\Temp\mail-tcp.etl
# Problem reproduzieren, danach:
pktmon stop
pktmon etl2pcap C:\Temp\mail-tcp.etl --out C:\Temp\mail-tcp.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mail-tcp.pcap \
  'host 192.0.2.25 and tcp port 25'

ss -tin state established '( dport = :25 or sport = :25 )'</code></pre>
  </div>
</div>

[`Pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) capture les paquets dans la pile réseau Windows et peut convertir son format ETL en PCAPNG. [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) utilise sous Unix une expression de capture et écrit ici un fichier PCAP pour une analyse ultérieure. Une capture complète conserve la charge utile et peut contenir des identifiants ou le contenu de messages. Elle doit être traitée comme une exportation d’e-mails : filtrage strict, capture brève, stockage protégé et suppression après analyse. Avec TLS, les adresses, ports, tailles de paquets, timing, métadonnées de handshake et signaux TCP restent visibles, mais pas le contenu applicatif chiffré.

### Signaux observés et point de passage le plus probable

| Observation | Vue TCP | Vérification suivante la plus probable |
|---|---|---|
| SYN est répété, aucune réponse | l’établissement de connexion n’atteint aucun pair répondant | cible DNS, route, Security Group, rejet par pare-feu, chemin de retour asymétrique |
| SYN reçoit immédiatement RST | le chemin vers la cible fonctionne, le socket est refusé | listener, adresse de liaison, cible NAT, configuration du port |
| le handshake réussit, pas de bannière | TCP est établi, l’application ne fournit aucune donnée | worker, pool de threads, protocole proxy, journal applicatif, dépendance backend |
| RST immédiatement après les données client | le pair ou une middlebox abandonne la session | protocole incorrect, ACL, TLS sur un port en clair, équipement d’inspection |
| retransmissions et ACK dupliqués en hausse | perte, réordonnancement ou paquets absents de la capture | erreurs d’interface, surcharge, MTU/PMTUD, changement de chemin, capture opposée |
| Zero Window | le récepteur signale un tampon plein | CPU, thread applicatif, stockage, garbage collection, tampon de socket |
| nombreuses anciennes `CLOSE-WAIT` | le pair a terminé, l’application locale ne ferme pas | fuite de descripteurs de fichiers, chemin de code bloqué, pool de connexions |
| les connexions échouent uniquement sous charge | limite d’état ou de capacité | backlog d’écoute, SYN-Cookies, NAT/Conntrack, ports éphémères, limite FD |
| les petites données fonctionnent, les grandes bloquent | erreur de chemin dépendante de la taille | MTU, filtre PMTUD/ICMP, MSS clamping, limites des enregistrements TLS/proxy |

## Capacité et zones de défaillance

Après la connexion individuelle vient la question de la capacité. Chaque socket occupe un état sur le client et le serveur, souvent aussi dans le pare-feu, le NAT, l’équilibreur de charge ou le proxy. Le nombre de connexions supportable n’est donc pas déterminé uniquement par le CPU ou la bande passante, mais par la plus petite table, file d’attente ou quantité de ports le long du chemin réel.

Les administrateurs doivent au minimum distinguer les ressources suivantes :

- **Listen Backlog et SYN Backlog :** limitent respectivement les connexions entièrement et semi-ouvertes ;
- **descripteurs de fichiers et limites de processus :** un port libre ne suffit pas si l’application ne peut plus accepter de socket ;
- **ports éphémères :** les relais et proxys sortants ne peuvent atteindre le même socket cible en parallèle qu’avec des sockets locaux différents ;
- **tables NAT et Conntrack :** conservent leurs propres temporisateurs et peuvent être épuisées malgré des ressources libres sur les systèmes finaux ;
- **mémoire par socket :** les tampons d’envoi/réception, l’état TLS et le contexte applicatif se multiplient avec le nombre de connexions ;
- **répartition de TIME-WAIT :** le côté qui ferme activement porte cet état ; les décisions d’architecture déplacent donc la charge ;
- **affinité et chemin de retour de l’équilibreur de charge :** les équipements avec état doivent voir les deux directions d’une même connexion.

Réduire aveuglément les temporisateurs TIME-WAIT, keepalive ou retransmission résout rarement la cause. Cela modifie la protection du protocole et la sémantique des erreurs. Il faut d’abord mesurer le taux de connexion, la durée de maintien, la distribution des quadruplets et la table réellement épuisée.

## Surveillance des environnements de messagerie

La surveillance découle de ces états. La seule utilisation des interfaces ne détecte pas les handshakes lents, les retransmissions, les tempêtes de resets ni un récepteur dont la fenêtre est fermée. Un tableau de bord utile relie donc les vues de l’hôte, du réseau et de l’application et présente au minimum les séries temporelles suivantes côte à côte :

| Métrique | Indication | Alerte uniquement en relation avec |
|---|---|---|
| taux de réussite des connexions et p95/p99 | accessibilité et durée d’établissement | cible, famille IP, port et emplacement |
| nouvelles connexions par seconde | charge et renouvellement | connexions établies et durée moyenne de maintien |
| `SYN-SENT` / `SYN-RECEIVED` | connexions semi-ouvertes | âge, pertes de backlog, SYN-Cookies |
| RST selon la direction et la phase de vie | interruptions actives | avant/après handshake, nœud émetteur, code applicatif |
| taux de retransmission | signal de perte ou de réordonnancement | RTT, pertes d’interface, position de capture |
| RTT et RTO | latence du chemin et comportement des temporisateurs | région, pair, heure de la journée |
| durée de Zero Window | le récepteur ne lit pas | CPU, stockage, JVM/GC, pools de threads |
| âge de `CLOSE-WAIT` | l’application locale ne ferme pas | processus, version et pool de connexions |
| utilisation NAT/Conntrack | état dans le nœud intermédiaire | délais d’expiration, utilisation des ports, chemin de retour |

Les tests synthétiques de ports doivent être associés à un test de protocole. Pour SMTP, cela signifie : connexion TCP, bannière, EHLO, StartTLS facultatif et transaction contrôlée sont des points de mesure distincts. C’est la seule manière d’identifier si le chemin réseau, TLS ou l’application de messagerie provoque la latence.

## Sécurité et durcissement

TCP fournit l’ordre et la correction d’erreur, mais ni confidentialité ni identité cryptographique. Les applications ajoutent généralement [TLS](/kb/tls), les réseaux éventuellement IPsec et les connexions TCP spéciales TCP-AO. Les numéros de séquence compliquent certes l’injection aveugle, mais ne remplacent aucune authentification. Le durcissement doit également prendre en compte les SYN floods, les resets falsifiés, les files d’attente trop grandes et les listeners susceptibles d’être détournés ([RFC 9293, section 7](https://datatracker.ietf.org/doc/html/rfc9293#section-7)).

Pour les services de messagerie exposés, les contrôles suivants sont pertinents :

- publier uniquement les listeners et familles d’adresses réellement nécessaires ;
- modéliser séparément les rôles serveur et client dans les pare-feux, plutôt que de définir globalement « TCP autorisé » ;
- surveiller le SYN Backlog, les SYN-Cookies et les taux de connexion avant que des limites de débit strictes n’affectent des serveurs de messagerie légitimes ;
- accepter Proxy Protocol uniquement entre pairs explicitement dignes de confiance ;
- documenter les terminaisons TCP et TLS afin que l’IP source, le délai d’expiration et la responsabilité du RST restent traçables ;
- protéger les captures de paquets et les diagnostics de socket comme des données opérationnelles potentiellement sensibles ;
- éviter les manipulations inutiles par des middleboxes sur le MSS, les options et les temporisateurs d’inactivité, et tester chaque écart nécessaire.

## Histoire technique

La séparation aujourd’hui évidente entre IP et TCP n’est apparue qu’au cours des premiers développements d’Internet. [RFC 675](https://datatracker.ietf.org/doc/html/rfc675) décrivait encore en 1974 un Internet Transmission Control Program commun. La séparation ultérieure a attribué à IP le transport par paquets entre réseaux et à TCP la session fiable de bout en bout. [RFC 793](https://datatracker.ietf.org/doc/html/rfc793) a normalisé ce TCP en 1981.

| Année | Évolution |
|---:|---|
| 1974 | RFC 675 décrit l’Internet Transmission Control Program comme précurseur |
| 1981 | RFC 793 définit TCP ; RFC 791 décrit l’Internet Protocol distinct |
| 1989 | RFC 1122 consolide les exigences des hôtes et précise de nombreuses règles TCP |
| 1992 | RFC 1323 introduit les grandes fenêtres et les Timestamps pour les liaisons à hautes performances |
| 2009 | RFC 5681 consolide Slow Start, Congestion Avoidance, Fast Retransmit et Fast Recovery |
| 2011 | RFC 6298 normalise le calcul du Retransmission Timeout |
| 2014 | RFC 7323 remplace RFC 1323 et actualise Window Scale ainsi que Timestamps |
| 2022 | RFC 9293 remplace RFC 793 et réunit à nouveau la norme de base dispersée sur plusieurs décennies |

Les implémentations ont évolué en parallèle : Selective Acknowledgment, Explicit Congestion Notification, algorithmes modernes de contrôle de congestion, TCP Fast Open et offloading du système d’exploitation étendent le noyau. Pour la recherche d’erreurs, le modèle d’origine composé de paires de sockets, d’espace de séquence, de fenêtres, d’états et de temporisateurs reste néanmoins valable.

## Sources

- [RFC Editor – RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/info/rfc9293)
- [RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293) – spécification TCP consolidée, en-tête, états, modèle de séquence, interfaces et sécurité.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – services enregistrés et plages de ports.
- [RFC 6298 – Computing TCP's Retransmission Timer](https://datatracker.ietf.org/doc/html/rfc6298) – algorithme RTT, RTO et backoff.
- [RFC 5681 – TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681) – Slow Start, Congestion Avoidance, Fast Retransmit et Fast Recovery.
- [Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html) – déduction des retransmissions, Zero Window, Out-of-Order et autres signaux d’analyse.
- [RFC 7323 – TCP Extensions for High Performance](https://datatracker.ietf.org/doc/html/rfc7323) – Window Scale, Timestamps, RTTM et PAWS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnostic des ports TCP, de la résolution de noms et des routes sous Windows.
- [OpenBSD Manual – nc(1)](https://man.openbsd.org/nc.1) – connexions TCP/UDP, listeners et tests de port avec Netcat.
- [OpenBSD Manual – telnet(1)](https://man.openbsd.org/telnet.1) – client Telnet, établissement de connexion et comportement de protocole.
- [iproute2 – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – états de socket, temporisateurs, files d’attente et filtres.
- [Microsoft Learn – Pktmon](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – moniteur de paquets Windows et conversion PCAPNG.
- [tcpdump(1) Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – options de capture, filtres et fichiers de sortie.
- [RFC 675 – Specification of Internet Transmission Control Program](https://datatracker.ietf.org/doc/html/rfc675) – précurseur technique de TCP et IP.
- [RFC 793 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793) – spécification TCP originale de 1981.
