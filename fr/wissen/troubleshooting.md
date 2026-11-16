---
title: "Dépannage : preuves, dépendances et diagnostic contrôlé"
blatt: "troubleshooting"
description: "Diagnostic systématique pour les administrateurs de messagerie et d’infrastructure : périmètre, chronologie des preuves, graphes de dépendances, états des protocoles, journaux, métriques, traces, capture de paquets, tests d’hypothèses, stabilisation et correction des causes."
fakten:
  - label: Méthode de base
    wert: Observation + modèle système → hypothèse → test discriminant → constat
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Premier jeu de données
    wert: attendu · réel · reproductible · fenêtre temporelle · périmètre · impact
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Trois perspectives de diagnostic
    wert: graphe de dépendances · état du protocole · chronologie commune
    href: https://opentelemetry.io/docs/concepts/signals/
  - label: Classes de preuves
    wert: métriques · journaux · traces · requêtes d’état · capture de paquets
    href: https://sre.google/workbook/monitoring/
  - label: Corrélation importante
    wert: Toujours associer Message-ID, Queue-ID, Trace-ID, Session-ID et hôte à un horodatage
    href: https://www.w3.org/TR/trace-context/
  - label: Triage
    wert: Limiter l’impact et préserver les preuves ; l’analyse des causes peut suivre
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Limite d’acceptation du courrier
    wert: 250 après End-of-Data transfère la responsabilité SMTP au destinataire
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Classes d’erreurs SMTP
    wert: 4yz temporaire · 5yz permanent ; Enhanced Status Code fournit la sous-classe
    href: https://datatracker.ietf.org/doc/html/rfc3463
  - label: Tests actifs
    wert: Les exécuter depuis l’origine affectée, avec la même identité, le même choix de destination et la même policy
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Temps
    wert: L’état NTP, le fuseau horaire et le décalage d’horloge font partie de tout constat distribué
    href: https://datatracker.ietf.org/doc/html/rfc5905
  - label: Constat négatif
    wert: Documente quelle hypothèse a été réfutée, dans quelles conditions
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Clôture
    wert: Vérifier le service, le chemin des données et la surveillance ; documenter séparément la cause et la prévention
    href: https://csrc.nist.gov/pubs/sp/800/61/r3/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - troubleshooting
  - messaging
  - betrieb
translationSourceHash: 126551190a29508cedf6aa90a1863b0b8a005ef6d04862bac0feb8d9dca5a769
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:24:10.902Z
translationReview: automatic
---

# Dépannage : preuves, dépendances et diagnostic contrôlé

Le dépannage est l’acquisition contrôlée de connaissances sur un système défaillant. Le point de départ n’est pas la cause favorite la plus probable, mais un écart précis entre le comportement attendu et le comportement observé. Les observations et un modèle technique donnent naissance à des hypothèses ; un test doit ensuite permettre de distinguer au moins deux causes possibles. Google SRE décrit cette démarche comme l’application de la méthode hypothético-déductive et souligne qu’une recherche de panne réussie nécessite à la fois une méthodologie générale et la connaissance de l’architecture concrète du système ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Pour les systèmes de messagerie distribués, la formule « vérifier de bas en haut » ne suffit pas. Un message peut avoir été accepté via DNS, TCP et TLS fonctionnels tout en restant bloqué dans une file d’attente, un moteur de politiques, une analyse de contenu, une base de données ou au saut suivant. À l’inverse, une erreur utilisateur dans une interface web peut être causée par l’identité, l’état de session, l’API, le stockage de la boîte aux lettres ou la recherche. Le diagnostic devient solide lorsque l’administrateur combine trois modèles : **le graphe de dépendances de l’opération concernée, les automates d’états des protocoles utilisés et une chronologie alignée sur l’ensemble des composants**.

L’explication commence par le symptôme observable et mène, via le périmètre, la chronologie et les preuves, à une hypothèse vérifiable. La modification, la vérification et le suivi n’interviennent qu’une fois cause et effet distingués.

Le dépannage commence par un écart observable : qui est concerné, depuis quand, sur quel chemin et avec quel résultat ? C’est seulement à partir de ce périmètre qu’une hypothèse vérifiable émerge. Cet article couvre le processus depuis la délimitation, les preuves et les tests jusqu’à la résolution vérifiée.

## Modèle de diagnostic : périmètre, état et preuves

Un bon rapport de problème contient le comportement attendu, le comportement réel et, lorsque cela est possible, une exécution reproductible. Avant même l’analyse technique, le périmètre est déterminé : qui ou quoi est concerné, depuis quand, dans quel sens, par quel point d’entrée et avec quel impact ? Ce triage détermine s’il faut d’abord examiner un cas isolé, rediriger le trafic, annuler une modification défectueuse ou ouvrir un incident plus important. En cas de panne grave, la stabilisation prime sur l’analyse complète des causes ; les journaux volatils et autres preuves doivent néanmoins être préservés ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-troubleshooting.svg?v=20260813" title="Interaktive Infografik: evidenzbasierter Diagnosefluss vom Problembericht über Scope, Zeitachse, Abhängigkeiten, Protokollzustände und Signale bis zum kontrollierten Hypothesentest und verifizierten Abschluss" loading="lazy">
  <a href="/images/kb-interaktiv-troubleshooting.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

### Matrice de périmètre pour le premier constat

La première note d’administration ne devrait pas être « le courrier ne fonctionne pas », mais décrire une transaction reproductible. Pour la messagerie et l’infrastructure, les champs suivants sont au minimum utiles :

| Champ | Contenu solide | Pourquoi cela réduit l’espace des erreurs |
|---|---|---|
| Attendu / réel | « La cible SMTP doit renvoyer 250 après DATA ; 451 4.4.1 est observé » | distingue l’attente fonctionnelle du symptôme |
| Ensemble affecté | utilisateur isolé, domaine expéditeur, tenant, site, protocole ou toutes les transactions | distingue les erreurs d’objet, de politique, de chemin et de plateforme |
| Direction et entrée | entrant/sortant, relais/soumission/accès, GUI/API | désigne l’écouteur, la politique et la limite de responsabilité |
| Identité et autorisation | utilisateur, principal de service, certificat, IP source, rôle | rend l’authentification et l’autorisation vérifiables |
| Période exacte | première/dernière erreur en UTC, fréquence, durée, fuseau horaire | aligne journaux, métriques, traces et changements |
| Valeurs de corrélation | Message-ID, Queue-ID, Trace-ID, Session-ID, hôte et cible | relie les événements entre les composants |
| Reproductibilité | déterministe/intermittente, entrée concrète, cas de comparaison fonctionnel | permet des tests discriminants |
| Impact et solution de contournement | perdu, retardé, dupliqué, affichage uniquement ; contournement et risque | guide le triage et la priorité |
| Dernières modifications | déploiement, configuration, certificat, DNS, pare-feu, migration de données, profil de charge | fournit des hypothèses, mais pas encore une preuve de causalité |

La formulation de chaque entrée distingue **observé**, **déduit** et **inconnu**. « La connexion TCP se termine par un délai d’attente après 21 secondes » est observé. « Le pare-feu bloque » est une hypothèse tant qu’aucun compteur de règle, aucune observation de paquet ou aucun rejet administratif ne l’étaye. Cette discipline linguistique empêche que des suppositions deviennent imperceptiblement des faits.

## Graphe de dépendances plutôt qu’échelle universelle de couches

OSI ou TCP/IP expliquent l’encapsulation des protocoles, mais ne représentent qu’en partie les dépendances opérationnelles. Pour l’opération concrète, un administrateur a besoin d’un graphe orienté : quel composant choisit la destination, lequel ouvre la connexion, lequel vérifie l’identité et la politique, où l’état est durablement pris en charge et quels services en aval peuvent empêcher l’achèvement ? Google SRE recommande d’examiner les entrées et sorties connues à des limites de composants bien définies, et de réduire les grands systèmes linéairement ou par dichotomie ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Pour un flux de courrier sortant, le chemin peut se présenter ainsi :

1. L’application expéditrice crée l’enveloppe et le contenu du message, puis les transmet à un point de terminaison de soumission ou de relais.
2. [DNS](/kb/dns) fournit les données MX et d’adresse ; les résolveurs, caches, suffixes de recherche et le Split DNS peuvent générer des réponses différentes.
3. [TCP](/kb/tcp) relie une adresse source donnée à une adresse cible et à un port donnés.
4. [TLS](/kb/tls) négocie les paramètres et vérifie — selon la politique de transport — le nom, la chaîne de certificats ou d’autres ancres de confiance.
5. [SMTP](/kb/smtp) passe par Greeting, EHLO, l’authentification facultative, l’enveloppe et DATA. Une réponse `250` après le point final transfère la responsabilité au serveur SMTP récepteur ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).
6. Le destinataire inscrit le message dans une file d’attente ou un magasin, applique des règles de routage, de détection des logiciels malveillants, de DLP, d’archivage et de chiffrement, puis tente le saut suivant.
7. Le service de boîte aux lettres, l’index, l’accès client et les notifications déterminent si le message accepté est visible pour l’utilisateur.

Le diagnostic interroge chaque arête sur une **transition vérifiable**. Une connexion TCP réussie prouve uniquement le handshake TCP vers la cible testée. Elle ne prouve ni la confiance TLS ni l’acceptation SMTP. Un `250` SMTP après End-of-Data prouve la prise en charge par ce saut, mais pas le dépôt dans la boîte aux lettres finale. Une sortie réussie de file d’attente ne prouve pas qu’un index de recherche affiche déjà le message.

Outre le chemin direct, les dépendances transversales appartiennent au graphe : heure, résolution de noms, identité, certificats et clés, configuration, base de données, stockage d’objets ou de fichiers, file d’attente, répartiteur de charge, proxy, pare-feu, observabilité et fournisseurs externes. Une erreur dans une telle dépendance peut affecter simultanément plusieurs composants apparemment indépendants.

## États des protocoles et limites de responsabilité

Les protocoles offrent davantage de valeur diagnostique que la simple affirmation « port ouvert ». Le constat pertinent est **quel état a été atteint, quelle partie a envoyé en dernier un message valide et quelle réponse ou quel délai d’attente a suivi**.

- **DNS :** le nom de requête, le type, le résolveur interrogé, le RCODE, Answer/Authority, TTL et l’état DNSSEC forment un tout. La mise en cache fait partie de la conception DNS ; des résolveurs différents peuvent donc temporairement disposer d’états de données différents ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).
- **TCP :** adresse et port source/cible, état de connexion, retransmissions et fermeture distinguent l’absence d’écouteur, l’abandon par politique, le rejet actif et l’interruption après un établissement réussi. La RFC 9293 spécifie les états et le traitement des segments ; un ICMP Echo ne remplace pas le service TCP testé ([RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293), [RFC 4443](https://datatracker.ietf.org/doc/html/rfc4443)).
- **TLS :** ClientHello, ServerHello, chaîne de certificats, nom, magasin de confiance, protocole négocié et alerte localisent l’erreur. TLS 1.3 définit les alertes comme un contenu de protocole distinct ; « Handshake failure » est donc un état, et non simplement un synonyme d’erreur réseau ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).
- **SMTP :** chaque réponse appartient à une commande et à une phase de session. `4yz` signifie une réponse négative temporaire, `5yz` une réponse permanente ; les Enhanced Status Codes répartissent le statut en classe, sujet et détail ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).
- **LDAP :** un établissement TCP/TLS réussi ne dit rien sur Bind, Search Base, Filter, Controls ou les droits d’accès. LDAPResult transmet `resultCode`, `matchedDN`, un message de diagnostic et des referrals facultatifs ([RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).
- **HTTP :** code de statut, méthode, URI cible, sauts proxy/passerelle, en-têtes et corps de réponse constituent un constat. Un `403` ne signifie pas la même chose qu’un échec d’établissement de connexion ; la RFC 9110 définit la sémantique des codes de statut ([RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110)).
- **IMAP :** les réponses étiquetées terminent une commande ; des données non étiquetées peuvent signaler des changements d’état indépendamment. Des états tels que not authenticated, authenticated et selected limitent les commandes autorisées ([RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)).

La question passe ainsi de « le serveur X fonctionne-t-il ? » à « quel est le dernier état confirmé atteint par la transaction Y, avec ces entrées et sur ce chemin ? ».

Lorsque les dépendances et la phase de protocole sont connues, les signaux appropriés sont sélectionnés. Les métriques montrent l’ampleur et le moment, les journaux les décisions, les traces le chemin d’appel et les données de paquets le dialogue réseau réel.

## Signaux : métriques, journaux, traces et données de paquets

Aucun signal individuel n’est la vérité de l’ensemble du système. Les métriques condensent de nombreux événements en séries temporelles, les journaux décrivent des événements discrets, les traces relient les opérations le long d’une requête, les requêtes d’état montrent l’état présent et une capture de paquets documente le trafic réseau observé à un point de mesure. Google SRE cite la surveillance, les journaux et le traçage comme moyens de diagnostic centraux ; OpenTelemetry distingue notamment les traces, les métriques et les journaux comme signaux de télémétrie ([Google SRE – Monitoring](https://sre.google/workbook/monitoring/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)).

| Signal | Répond particulièrement bien à | Limite typique |
|---|---|---|
| Métrique | Quand l’écart a-t-il commencé, quelle est son ampleur, quelle population est concernée ? | L’agrégation masque les cas individuels ; les labels à forte cardinalité sont coûteux |
| Journal structuré | Quel événement s’est produit dans un composant, avec quels champs ? | Le manque de contexte, l’échantillonnage, la rotation ou une horloge erronée compliquent la corrélation |
| Trace | Quels sauts et quelles durées a eus une requête individuelle ? | Ne fonctionne qu’aux limites instrumentées qui propagent le contexte |
| Requête d’état | Que vaut actuellement l’écouteur, la file d’attente, la réplication, le certificat ou la configuration ? | Peut différer du moment de l’erreur et modifier l’état |
| Capture de paquets | Quels paquets ce point de mesure d’interface/hôte a-t-il vus ? | Aucune visibilité sur l’application interne, le contenu chiffré ou un autre chemin |

La norme W3C Trace Context définit `traceparent` et `tracestate`, afin que l’identité de trace puisse être propagée entre les composants HTTP. Il ne s’agit pas d’un identifiant de courrier universel. Dans une plateforme de messagerie, Trace-ID, RFC-Message-ID, Queue-ID des différents MTA, Session-ID et hôte cible doivent donc être explicitement reliés ([W3C Trace Context](https://www.w3.org/TR/trace-context/)). Syslog standardise un format d’événement avec données structurées, mais ne garantit à lui seul ni l’exhaustivité ni la conservation centralisée ; la gestion des journaux comprend la génération, la transmission, le stockage, l’accès et l’élimination ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [NIST SP 800-92](https://csrc.nist.gov/pubs/sp/800/92/final)).

### La chronologie commune

Une chronologie ne contient pas seulement le moment présumé de l’erreur, mais au minimum le premier et le dernier succès connus, la première et la dernière erreur, les déploiements et modifications de configuration pertinents, les changements de certificat ou de jeton, les basculements, les redémarrages ainsi que la base temporelle de chaque preuve. NTP version 4 sert à synchroniser les horloges système ; les administrateurs doivent néanmoins vérifier l’offset, le stratum/la source, l’état Leap/synchronisation, le fuseau horaire et d’éventuels sauts manuels ([RFC 5905](https://datatracker.ietf.org/doc/html/rfc5905)).

« Horodatage 10:14 » n’est pas un point de référence commun sans fuseau horaire, hôte et état de l’horloge. Dans les files d’attente asynchrones, l’heure d’événement de la réception, du traitement et de l’export de journaux peut également diverger. Une note robuste conserve l’horodatage original et ajoute une heure UTC normalisée au lieu d’écraser la valeur originale.

Les signaux seuls n’expliquent pas la cause. Une bonne hypothèse prédit le résultat qui doit se produire lors d’un test ciblé et ce qui la distingue d’une alternative.

## Hypothèses et tests discriminants

Une hypothèse est utile lorsqu’un test à faible risque peut la réfuter ou la rendre plus probable qu’une alternative. « Problème réseau » est trop large. « Le MTA ne peut pas atteindre l’adresse IPv6 sélectionnée par son résolveur sur TCP 25, alors qu’IPv4 fonctionne » indique l’origine, le choix de la cible, le protocole et la comparaison.

Un plan de test consigne avant l’exécution :

1. quelles deux hypothèses ou plus le test distingue ;
2. depuis quel hôte, espace de noms, interface et utilisateur il s’exécute ;
3. quelles configurations DNS, proxy, routage, identité et confiance il utilise réellement ;
4. quel résultat est attendu dans chaque cas ;
5. quels effets secondaires, accès aux données ou charges sont générés ;
6. comment une modification de test est annulée ;
7. quelles données brutes et quels horodatages sont préservés.

Google SRE souligne que les tests peuvent être trompeurs en raison de sources différentes, de politiques de pare-feu, d’identifiants ou d’autres facteurs perturbateurs. Le test lui-même peut aussi influencer le système : une journalisation détaillée modifie la charge et le timing ; un redémarrage efface l’état volatil ; une nouvelle tentative peut déclencher un message en double. Les tests doivent donc s’exécuter depuis l’origine affectée et, si possible, avec le même choix de cible, la même identité et la même politique que le processus défaillant ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

La comparaison avec un cas fonctionnel est particulièrement forte lorsqu’une seule variable indépendante diffère : le même utilisateur sur un autre hôte, un autre utilisateur sur le même hôte, le même message vers un autre domaine ou le même hôte cible via une autre famille d’adresses. Plusieurs modifications de configuration simultanées créent en revanche un nouvel état système inconnu.

## Outils d’administration le long du chemin réel

Les commandes suivantes sont des exemples de diagnostic, et non des contrôles de santé universels. Les espaces réservés doivent être adaptés à l’origine concernée, au résolveur, à l’hôte, au port et à la période. Les identifiants, jetons et contenus de messages ne doivent pas être placés sans expurgation dans les tickets ou l’historique du shell.

### Consigner la réponse DNS et le résolveur sélectionné

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type A -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type AAAA -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer +authority example.ch MX
dig @192.0.2.53 +noall +answer mx1.example.ch A
dig @192.0.2.53 +noall +answer mx1.example.ch AAAA</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) rendent explicites le résolveur et le type d’enregistrement. La comparaison avec un serveur public ou faisant autorité peut révéler les différences de Split DNS et de cache, mais ne doit pas être confondue avec le résolveur réellement utilisé par le service.

### Vérifier la connexion TCP et le chemin depuis l’origine affectée

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP- und Pfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
tracert -d mx1.example.ch</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
traceroute -n mx1.example.ch</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) testent l’établissement TCP ; [`tracert`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert) et [`traceroute`](https://man7.org/linux/man-pages/man8/traceroute.8.html) examinent les limites de sauts et les retours le long d’un chemin. Une étoile dans la route ne prouve pas une perturbation du trafic utile, et un test ICMP Echo réussi ne prouve pas qu’un port TCP est ouvert. Le [`ping`](https://ftp.arl.army.mil/~mike/ping.html) historique reste utile comme sonde de latence/accessibilité, mais ne constitue pas non plus un test applicatif.

### Observer TLS et le dialogue applicatif

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TLS- und Protokolldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affiche le handshake TLS, les certificats et les alertes ; [`curl`](https://curl.se/docs/manpage.html) peut notamment dialoguer en HTTP, SMTP, IMAP, POP3 et LDAP. Tous deux utilisent leur propre configuration de compilation et de confiance, qui peut différer du produit concerné. [`telnet`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet) ou la [page de manuel OpenBSD de `telnet`](https://man.openbsd.org/telnet.1) est historiquement répandu pour les dialogues en clair, mais ne fournit aucune vérification TLS ; pour des sondes TCP simples, `nc` est généralement plus explicite.

### Relever les écouteurs locaux et les états de connexion

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-NetTCPConnection -LocalPort 25,443 -ErrorAction SilentlyContinue |
  Sort-Object LocalPort,State,RemoteAddress |
  Format-Table LocalAddress,LocalPort,RemoteAddress,RemotePort,State,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp '( sport = :25 or sport = :443 )'
ss -nto state syn-sent</code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affichent les points de terminaison locaux et les états TCP. L’adresse d’attachement, l’espace de noms/conteneur, le processus propriétaire et la famille d’adresses sont également déterminants : un écouteur sur `127.0.0.1` ou dans un autre espace de noms réseau n’est pas automatiquement accessible depuis l’extérieur.

### Capture de paquets avec filtre et fenêtre temporelle restreints

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add Mailflow -t TCP SYN ACK RST FIN -p 25
pktmon start --capture --pkt-size 0 --file-name C:\\Temp\\mailflow.etl
pktmon stop
pktmon etl2pcap C:\\Temp\\mailflow.etl --out C:\\Temp\\mailflow.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mailflow.pcap \
  'host 192.0.2.25 and tcp port 25'</code></pre>
  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) et [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturent les paquets au point de mesure sélectionné. La cible, le filtre, la durée, l’espace de stockage et la protection des données doivent être définis au préalable. Une capture peut contenir des identifiants, des contenus personnels et du matériel de clé. De plus, « absent de la capture » prouve uniquement que le paquet n’a pas été observé à ce point de mesure avec ce filtre.

### Sauvegarder les journaux pendant la période exacte de l’erreur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$start = [datetime]'2026-08-08T10:00:00Z'
$end   = [datetime]'2026-08-08T10:15:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$start; EndTime=$end} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 10:00:00 UTC' \
  --until '2026-08-08 10:15:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) et [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) limitent la requête dans le temps. Le constat doit contenir l’ID d’événement/les noms de champs et l’heure brute ; un message d’erreur librement traduit est moins facilement recherchable que le code inchangé. Avant une rotation ou un redémarrage, les journaux pertinents sont exportés et leur provenance documentée.

### Vérifier l’horloge et la source de synchronisation

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">w32tm /query /status
w32tm /query /source
Get-Date -AsUTC -Format o</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">chronyc tracking
chronyc sources -v
date --utc --iso-8601=ns</code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) et [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) affichent la source et l’état de synchronisation ; l’affichage UTC local fournit une référence immédiatement documentable. Une horloge synchronisée ne garantit pas encore qu’une application écrit le même horodatage ou le même fuseau horaire.

Dans la messagerie, ce modèle est appliqué tout au long du message : l’acceptation, la politique, la file d’attente, le saut suivant, la boîte aux lettres et l’accès client disposent chacun de leurs propres preuves.

## Diagnostic de messagerie comme chaîne de responsabilité

Pour un message, la question la plus importante n’est pas « où manque-t-il ? », mais **quel composant a démontrablement pris sa responsabilité et à qui l’a-t-il transmise ?** Chaque saut SMTP possède sa propre session et généralement son propre Queue-ID. L’en-tête RFC-Message-ID est porté par le contenu du message, mais il n’est pas obligatoirement unique ni digne de confiance ; les Queue-ID sont locaux et changent à chaque saut.

Un suivi solide documente donc :

1. l’expéditeur et le destinataire de l’enveloppe, séparément des adresses d’en-tête visibles ;
2. Message-ID, taille et une empreinte respectueuse de la protection des données ou un objet de test ;
3. la session client/de soumission et le premier constat d’acceptation `250` ;
4. chaque association entre Queue-ID entrant et sortant ;
5. la décision de politique, la quarantaine, la réécriture, la séparation ou le bounce ;
6. le saut suivant sélectionné, la réponse DNS, l’IP source/cible, le résultat TLS et SMTP ;
7. l’acceptation finale ou l’erreur permanente pour chaque destinataire ;
8. le constat de boîte aux lettres, d’index et de client, si l’acceptation de transport a réussi.

Un SMTP-`451 4.4.1` n’est pas équivalent à un « 5xx » générique. Le premier chiffre du code SMTP et l’Enhanced Status Code doivent être conservés littéralement. Les erreurs temporaires entraînent habituellement une file d’attente et une nouvelle tentative ; les erreurs permanentes peuvent déclencher un NDR/DSN. La possibilité de renvoyer un message dépend du fait qu’une limite d’acceptation ait déjà été franchie. Répéter aveuglément après la perte d’une réponse peut produire des doublons ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).

## Distinguer triage, résolution et analyse des causes

En cas d’impact élevé, il convient de distinguer trois axes de travail :

- **Stabiliser :** rediriger le trafic, limiter la fonction défectueuse, protéger la capacité, arrêter la corruption de données ou effectuer un rollback sûr connu.
- **Préserver les preuves :** sauvegarder les journaux, l’état des files d’attente, la configuration, les métriques, les traces, les données de paquets, l’état temporel et les références de changements avant leur rotation ou leur disparition lors d’un redémarrage.
- **Diagnostiquer :** prioriser les hypothèses et les réfuter ou les étayer par des tests présentant le moins de risques possible.

Une résolution temporaire ne prouve pas automatiquement la cause. Si un redémarrage met fin à la panne, une fuite mémoire, un interblocage, des pools de connexion épuisés, un cache corrompu, un délai d’attente externe et de nombreuses autres causes restent possibles. Le redémarrage doit être consigné comme intervention avec son moment et son effet ; la prévention et les preuves supplémentaires sont ensuite planifiées.

De même, la « dernière modification » est une source d’hypothèse, non une preuve de causalité. Une corrélation temporelle peut résulter d’un déclencheur commun ou du hasard. Google SRE met explicitement en garde contre les symptômes non pertinents, les théories favorites improbables et les corrélations trompeuses ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Une modification ne clôt l’incident que lorsque le chemin d’origine fonctionne à nouveau, que les effets secondaires sont exclus et que la surveillance confirme l’état stable.

## Vérification et critères de clôture

Un incident n’est pas clos lorsqu’un seul test réussit. La réception vérifie l’opération fonctionnelle initiale, le périmètre affecté et les zones de défaillance adjacentes :

- transactions représentatives dans les deux directions pertinentes ;
- accumulation et vidage de la file d’attente, taux de nouvelles tentatives et d’erreurs ;
- cohérence des données, réplication, recherche et visibilité côté client ;
- authentification, autorisation, chemins de certificats et de clés ;
- surveillance, alertes et tableaux de bord du point de vue de l’utilisateur ;
- nouvelle vérification après au moins un intervalle pertinent de minuterie, de nouvelle tentative ou de cache ;
- aucune nouvelle lacune de données, aucun doublon inattendu ni aucune dégradation cachée.

Le rapport de clôture distingue symptôme, impact, chronologie, cause immédiate, facteurs contributifs, restauration et mesures durables. Les constats négatifs sont conservés : ils documentent quelles hypothèses ont été réfutées dans quelles conditions et empêchent la répétition du travail. NIST SP 800-61 Rev. 3 intègre la réponse aux incidents dans la détection, la réponse, la restauration et l’amélioration à l’échelle de l’organisation ([NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

## Évolution technique des outils de diagnostic

Le diagnostic réseau précoce rendait directement visibles des propriétés de protocoles individuelles. Mike Muuss a écrit `ping` en 1983 pour 4.2a BSD sur la base d’ICMP Echo ; `traceroute` de Van Jacobson a ensuite utilisé l’expiration des limites de sauts IP et les retours ICMP pour rendre visibles les stations du chemin ([Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)). Ces outils restent précieux, mais ne répondent qu’à des questions étroitement définies.

Avec les applications distribuées, le besoin de diagnostic est passé de l’accessibilité des hôtes à la corrélation des transactions. Syslog a reçu un format de message standardisé, le traçage distribué a relié des opérations individuelles, W3C Trace Context a standardisé la transmission HTTP de l’identité de trace et OpenTelemetry a réuni traces, métriques et journaux dans un cadre d’observabilité neutre vis-à-vis des fournisseurs ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [W3C Trace Context](https://www.w3.org/TR/trace-context/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)). Le cœur méthodologique reste le même : un outil n’est valable que par la précision de la question, le point de mesure et l’interprétation documentée de son résultat.

## Sources

- [Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
- [NIST – SP 800-61 Rev. 3, Incident Response](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [IETF RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293)
- [IETF RFC 4443 – ICMPv6](https://datatracker.ietf.org/doc/html/rfc4443)
- [IETF RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [IETF RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 9110 – HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [IETF RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Google SRE Workbook – Monitoring](https://sre.google/workbook/monitoring/)
- [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)
- [W3C – Trace Context](https://www.w3.org/TR/trace-context/)
- [IETF RFC 5424 – The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [NIST – SP 800-92, Log Management](https://csrc.nist.gov/pubs/sp/800/92/final)
- [IETF RFC 5905 – Network Time Protocol Version 4](https://datatracker.ietf.org/doc/html/rfc5905)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – tracert](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert)
- [Linux man-pages – traceroute](https://man7.org/linux/man-pages/man8/traceroute.8.html)
- [Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – telnet](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet)
- [OpenBSD – telnet](https://man.openbsd.org/telnet.1)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Packet Monitor (pktmon)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – Windows Time Service tools](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html)
