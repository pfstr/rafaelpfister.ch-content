---
title: "SSH : transport, identités et administration sécurisée"
blatt: "ssh"
description: "SSH pour les administrateurs d’infrastructure et de messagerie : transport, échange de clés et identité d’hôte, authentification des utilisateurs et des charges de travail, canaux, clés et certificats OpenSSH, agents, bastions et redirection, SFTP/SCP, politique serveur, durcissement, journalisation, Windows OpenSSH et histoire technique."
fakten:
  - label: Rôle système
    wert: couche de transport et de canaux chiffrée, protégée en intégrité et authentifiée pour l’administration
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Port standard
    wert: TCP 22 ; le listener réel et le chemin d’accès dépendent de la configuration
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Pile protocolaire
    wert: Transport Protocol · User Authentication Protocol · Connection Protocol
    href: https://www.rfc-editor.org/rfc/rfc4251.html
  - label: Identité du serveur
    wert: Host Key ou Host Certificate ; à vérifier avant les identifiants utilisateur
    href: https://www.rfc-editor.org/rfc/rfc4253.html
  - label: Authentification utilisateur
    wert: publickey · password · hostbased · keyboard-interactive ; la politique peut exiger plusieurs facteurs
    href: https://www.rfc-editor.org/rfc/rfc4252.html
  - label: Négociation cryptographique
    wert: KEX · algorithme de clé d’hôte · Cipher · MAC/AEAD · compression
    href: https://www.rfc-editor.org/rfc/rfc9142.html
  - label: Types de clés
    wert: Ed25519 · ECDSA · RSA/SHA-2 · variantes FIDO Security Key ; utilisation selon la compatibilité client/serveur
    href: https://www.rfc-editor.org/rfc/rfc8709.html
  - label: Canaux d’accès
    wert: Shell · Exec · Subsystem · redirection TCP/Streamlocal · redirection Agent/X11
    href: https://www.rfc-editor.org/rfc/rfc4254.html
  - label: Transfert de fichiers
    wert: SFTP comme sous-système SSH ; la CLI scp peut utiliser la sémantique SFTP ou SCP historique selon l’implémentation
    href: https://man.openbsd.org/sftp
  - label: Distribution de la confiance
    wert: known_hosts/Pinning · SSHFP avec DNSSEC · Host-CA · certificats utilisateur/hôte OpenSSH
    href: https://www.rfc-editor.org/rfc/rfc4255.html
  - label: État opérationnel
    wert: listener · politique effective · Host Keys · chemin d’authentification · sessions/canaux · journaux · réseau source
    href: https://man.openbsd.org/sshd_config
  - label: Objet de récupération
    wert: identité d’hôte/CA, politique serveur, Authorized Principals/Keys, Break Glass, chemin d’audit et dépendances de bastion
    href: https://man.openbsd.org/ssh-keygen
werbung:
  - newsletter
ctaThemen:
  - claude
  - powershell
  - haertung
translationSourceHash: 9617920ae627ef338448276f4094696352ba3f158be192d73dd10226c8ee296f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-13T11:20:47.841Z
translationReview: automatic
---

# SSH : transport, identités et administration sécurisée

Secure Shell est une famille de protocoles destinée à l’accès distant sécurisé sur un réseau non fiable. SSH réunit trois fonctions : il établit un transport chiffré et protégé en intégrité, authentifie d’abord le serveur puis l’utilisateur ou la charge de travail, et multiplexe enfin shells, commandes individuelles, sous-systèmes et redirections sous forme de canaux distincts. Cet ordre est déterminant pour les administrateurs : une clé utilisateur robuste ne protège pas le client s’il la propose à un serveur non vérifié.

SSH est à la fois un protocole et un plan de contrôle. Un accès peut permettre des commandes root, des transferts de fichiers, des tunnels de base de données, l’accès à un agent et des sauts vers des réseaux internes. L’évaluation de la sécurité ne doit donc pas s’arrêter au port d’écoute ni à `PasswordAuthentication no`. La distribution des Host Keys, la politique Match effective, l’origine des clés, les principals, les droits de redirection, le chemin source, la journalisation des sessions et la récupération des ancres de confiance font partie du même système.

L’explication suit une connexion SSH depuis TCP et l’échange de versions, en passant par la négociation de clés et la vérification de l’hôte, jusqu’à l’authentification utilisateur et au canal. Les redirections, l’exploitation des clés, les certificats et le diagnostic s’appuient sur cette base.

SSH transforme une connexion réseau non fiable en un transport authentifié et chiffré, contenant des canaux distincts pour le shell, les commandes, le transfert de fichiers ou les tunnels. Cet article suit précisément cet ordre : identité du serveur, échange de clés, authentification utilisateur, canaux et exploitation.

## Architecture du protocole

La RFC 4251 divise SSH-2 en trois protocoles :

1. **Transport Layer Protocol :** échange de versions, négociation d’algorithmes, Key Exchange, authentification du serveur, chiffrement et intégrité.
2. **User Authentication Protocol :** authentification d’un utilisateur client auprès du serveur sur le transport protégé.
3. **Connection Protocol :** multiplexage de canaux logiques pour les sessions, le Port Forwarding et d’autres services.

Les couches fonctionnent sur un flux d’octets fiable, généralement [TCP](/kb/tcp). Le port TCP 22 est enregistré auprès de l’IANA sous le nom `ssh` ; les reverse proxies, redirections de ports et bastions peuvent modifier le chemin réel ([RFC 4251 – Architecture du protocole SSH](https://www.rfc-editor.org/rfc/rfc4251.html), [Registre IANA des noms de service et numéros de port](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-ssh.svg?v=20260813" title="Interaktive Infografik: SSH-Verbindungsaufbau von TCP und Versionsaustausch über KEX, Hostidentität und Benutzerautorisierung bis zu Channels, Bastion, Schlüssel- und Zertifikatslebenszyklus sowie Audit und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-ssh.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## Échange de versions et négociation d’algorithmes

Le client et le serveur envoient d’abord des chaînes d’identification telles que `SSH-2.0-...`. Ils échangent ensuite `SSH_MSG_KEXINIT` avec des listes ordonnées pour le Key Exchange, l’algorithme de clé d’hôte, le chiffrement dans les deux directions, le MAC, la compression et la langue. La première option commune de chaque catégorie est sélectionnée. Les listes d’algorithmes constituent donc une politique et un contrat de compatibilité, et non un simple détail d’implémentation ([RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)).

La RFC 9142 actualise les recommandations concernant les algorithmes SSH. Elle distingue `MUST`, `SHOULD`, `MAY`, `SHOULD NOT` et `MUST NOT` pour les implémentations ; une politique de production tient aussi compte de la conformité, du parc de clients, du matériel et du chemin de migration ([RFC 9142 – Recommandations de méthodes de Key Exchange pour SSH](https://www.rfc-editor.org/rfc/rfc9142.html)).

Les lignes de configuration qui imposent une liste de chiffrements personnalisée et figée vieillissent. OpenSSH peut afficher ses algorithmes pris en charge avec `ssh -Q` et montrer la configuration client ou serveur effectivement résolue. Les exploitants retirent de manière ciblée les procédures historiques connues, tout en conservant un chemin de compatibilité testé pour les appliances.

## Key Exchange et clés de session

Le Key Exchange génère un secret commun et un Exchange Hash sans transmettre la clé de session sur le réseau. Le hash lie les chaînes d’identification, les messages KEXINIT, la clé d’hôte du serveur et les valeurs éphémères propres à l’algorithme. Le serveur signe l’Exchange Hash avec sa clé d’hôte ; le client vérifie ainsi simultanément l’identité du serveur et l’échange de clés ([RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html)).

Les méthodes modernes emploient des variantes éphémères de Diffie-Hellman et offrent la confidentialité persistante des clés de session. SSH peut effectuer un rekey pendant une connexion, par exemple après un volume de données ou une durée donnée. Le rekey et les Connection Channels partagent le même transport ; les problèmes apparaissent donc sous forme de gel ou de déconnexion de sessions apparemment actives.

La RFC 8308 définit une négociation générique d’extensions. OpenSSH utilise notamment des extensions pour annoncer des algorithmes de signature serveur ou des mises à jour de clés d’hôte ([RFC 8308 – Négociation d’extensions dans SSH](https://www.rfc-editor.org/rfc/rfc8308.html)). Strict Key Exchange protège plus fortement la séquence initiale contre les messages inattendus et a été standardisé après l’analyse Terrapin ([RFC 9519 – Strict Key Exchange dans SSH](https://www.rfc-editor.org/rfc/rfc9519.html)).

Après le choix de la version et des algorithmes, le client doit d’abord reconnaître le serveur. Ce n’est qu’une fois la Host Key acceptée que l’authentification utilisateur ultérieure a un sens.

## Host Keys : identité du serveur avant identité utilisateur

La Host Key identifie le serveur ou un groupe contrôlé de serveurs. Lors du premier contact, un client ne disposant pas d’une ancre de confiance prédistribuée n’a aucune base cryptographique pour évaluer le fingerprint affiché. « Accept new » automatise le pinning pour les nouveaux hôtes, mais ne résout pas la distribution initiale sécurisée.

OpenSSH stocke les clés et marqueurs connus dans `known_hosts`. Une clé modifiée peut indiquer une rotation légitime, une réinstallation, une modification du pool de load balancers ou une attaque. L’avertissement ne se « corrige » pas en supprimant aveuglément la ligne ; la nouvelle clé est vérifiée via une source indépendante et la cause est documentée ([ssh(1) – vérification de clé d’hôte](https://man.openbsd.org/ssh), [sshd(8)](https://man.openbsd.org/sshd)).

### Modèles de confiance pour l’identité d’hôte

| Modèle | Force | Limite opérationnelle |
|---|---|---|
| fingerprint manuel hors bande | direct et simple pour peu d’hôtes | distribution sécurisée et rotation évoluent mal |
| pinning/TOFU `known_hosts` | détecte les modifications ultérieures | le premier contact reste non prouvé |
| SSHFP avec DNSSEC | DNS publie les fingerprints | le client doit valider DNSSEC ; le chemin du résolveur est une frontière de confiance |
| Host-CA OpenSSH | les clients font confiance à la CA et aux principals | clé de CA, processus de signature, validité et association des principals |
| gestion de configuration | distribuée et auditable de manière centralisée | confiance dans le pipeline et l’enrôlement |

La RFC 4255 définit les enregistrements SSHFP ; sans DNSSEC, ils ne peuvent être qu’une indication et non une ancre de confiance unique ([RFC 4255 – Enregistrements de ressources SSHFP](https://www.rfc-editor.org/rfc/rfc4255.html)). Les certificats OpenSSH lient clé publique, numéro de série, type, principals, validité, Critical Options et Extensions par une signature de CA ([IETF-Draft – Format de certificat OpenSSH](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Hostidentität">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName $env:SSH_HOST -Type SSHFP -DnssecOk
ssh-keyscan -t ed25519,rsa $env:SSH_HOST | Set-Content .\\hostkeys.candidate
ssh-keygen -lf .\\hostkeys.candidate
ssh-keygen -F $env:SSH_HOST</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +dnssec "$SSH_HOST" SSHFP
ssh-keyscan -t ed25519,rsa "$SSH_HOST" &gt; hostkeys.candidate
ssh-keygen -lf hostkeys.candidate
ssh-keygen -F "$SSH_HOST"</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) et [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) interrogent le [DNS](/kb/dns) ; [`ssh-keyscan`](https://man.openbsd.org/ssh-keyscan) collecte les clés proposées, mais **ne les authentifie pas**. [`ssh-keygen`](https://man.openbsd.org/ssh-keygen) calcule les fingerprints et recherche dans `known_hosts`. Les candidats ne sont adoptés qu’après vérification indépendante.

## Cycle de vie et rotation des Host Keys

Un serveur peut disposer de plusieurs Host Keys pour différents algorithmes. Les Host Keys privées ne résident que sur les serveurs ou composants de load balancer contrôlés ; les sauvegardes et clones ne doivent pas entraîner une identité partagée involontaire. En cas d’autoscaling, une Host-CA convient souvent mieux que des Host Keys privées copiées.

Une rotation comprend :

1. générer et protéger une nouvelle clé ou un nouveau certificat d’hôte ;
2. les proposer en parallèle et distribuer le fingerprint ou la confiance CA ;
3. vérifier la compatibilité réelle des clients et le principal d’hôte ;
4. retirer l’ancienne clé de manière contrôlée ;
5. mettre à jour l’inventaire, la révocation et la documentation Break Glass.

OpenSSH peut annoncer des Host Keys supplémentaires après une authentification réussie via `UpdateHostKeys`. Cela facilite la rotation, mais suppose que la confiance existante envers l’hôte soit déjà correcte ([ssh_config(5)](https://man.openbsd.org/ssh_config)).

Avec un transport protégé et un serveur confirmé commence l’authentification de l’utilisateur ou de la machine. Mot de passe, clé publique, certificat et autres méthodes présentent des exigences distinctes d’exploitation et de récupération.

## User Authentication Protocol

Après le transport, le client demande le service `ssh-userauth`. La RFC 4252 définit les méthodes `publickey`, `password`, `hostbased` et `none` ; Keyboard-Interactive est spécifié dans la RFC 4256. Le serveur peut exiger une authentification supplémentaire et indique les méthodes suivantes autorisées ([RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html), [RFC 4256 – Authentification Keyboard Interactive](https://www.rfc-editor.org/rfc/rfc4256.html)).

La **méthode d’authentification** et l’**autorisation** sont distinctes. Une clé publique valide peut être associée à un compte via `authorized_keys`, un service d’annuaire, `AuthorizedKeysCommand`, des principals de certificat ou PAM. Les droits Unix, `sudo`, SELinux/AppArmor, les tokens Windows et la politique applicative déterminent ensuite les droits effectifs.

OpenSSH `AuthenticationMethods` peut imposer des combinaisons comme `publickey,keyboard-interactive:pam`. La question de savoir s’il s’agit de deux facteurs indépendants dépend des sources d’identifiants et de la liaison au terminal. Deux méthodes provenant toutes deux du même agent d’ordinateur portable compromis offrent moins d’indépendance que ne le laisse penser leur syntaxe ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Méthodes à clé publique et types de clés

Lors d’une authentification par clé publique, le client envoie une signature sur l’identifiant de session et la requête d’authentification ; la clé privée ne quitte pas le client. Le serveur vérifie la clé publique, la signature et l’autorisation. La RFC 8332 définit les signatures RSA avec SHA-2 pour SSH ; le type de clé `ssh-rsa` désigne historiquement aussi la sémantique de signature SHA-1 et ne doit pas être assimilé à toute utilisation d’une clé RSA ([RFC 8332 – Clés RSA avec signatures SHA-2](https://www.rfc-editor.org/rfc/rfc8332.html)).

La RFC 8709 standardise Ed25519 et Ed448 pour SSH. OpenSSH prend également en charge les clés `sk-` basées sur FIDO/U2F, dont la clé privée est liée à une Security Key et qui exigent facultativement la présence ou la vérification de l’utilisateur ([RFC 8709 – Ed25519 et Ed448 pour SSH](https://www.rfc-editor.org/rfc/rfc8709.html), [Prise en charge FIDO/U2F d’OpenSSH](https://www.openssh.com/txt/release-8.2)).

Le choix dépend de la compatibilité client/serveur et du modèle de menace. La longueur de clé seule ne détermine pas la sécurité : le stockage privé, la passphrase, l’accès à l’agent, la validité, le principal, la révocation et les options d’autorisation sont tout aussi importants.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Schlüssel- und Agentinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -t ed25519 -a 64 -f $env:USERPROFILE\\.ssh\\id_admin_ed25519
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub
Get-Service ssh-agent
ssh-add -l
ssh-add -T $env:USERPROFILE\\.ssh\\id_admin_ed25519.pub</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -t ed25519 -a 64 -f "$HOME/.ssh/id_admin_ed25519"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519.pub"
ssh-add -l
ssh-add -T "$HOME/.ssh/id_admin_ed25519.pub"
ssh-keygen -y -f "$HOME/.ssh/id_admin_ed25519" | diff - "$HOME/.ssh/id_admin_ed25519.pub"</code></pre>
  </div>
</div>

[`ssh-add`](https://man.openbsd.org/ssh-add) gère les identités dans l’agent et peut tester la capacité de signature d’une clé publique. [`Get-Service`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service) affiche le service d’agent Windows. Les fichiers de clés privées et les sockets d’agent reçoivent des ACL minimales ; les exemples ne génèrent pas de clés de CA ou d’hôte centrales.

## `authorized_keys` comme objet de politique

Une ligne dans `authorized_keys` peut comporter des options avant la clé. `from=`, `command=`, `restrict`, `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty`, `permitopen=` et `environment=` limitent l’utilisation. `restrict` définit un ensemble défensif d’options ; des autorisations explicites peuvent suivre ([sshd(8) – FORMAT DU FICHIER AUTHORIZED_KEYS](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)).

Une clé destinée à la sauvegarde ou au déploiement ne reçoit pas d’accès shell générique. Elle est liée au réseau source, à une commande forcée et aux destinations de redirection nécessaires. Les wrappers valident les arguments comme des entrées structurées et n’utilisent pas de concaténation shell non vérifiée.

`authorized_keys` est une configuration distribuée. L’inventaire comprend le fingerprint, le propriétaire, l’objectif, les comptes cibles, la source, l’expiration, la dernière utilisation et la révocation. Les clés orphelines sont des identifiants durables sans visibilité centrale.

## Certificats OpenSSH et principals

Les certificats OpenSSH ne sont pas des certificats X.509. Une CA SSH signe une clé publique avec un numéro de série, un Key ID, un type, des principals et une fenêtre de validité. Les User Certificates authentifient les utilisateurs auprès des serveurs ; les Host Certificates authentifient les serveurs auprès des clients. Les Critical Options doivent être comprises, sans quoi le certificat est rejeté ; les Extensions accordent des fonctions telles que PTY ou le forwarding ([IETF-Draft – Format de certificat OpenSSH](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)).

Les certificats réduisent la durée de vie des identifiants et centralisent l’émission, mais n’éliminent pas l’autorisation. Les serveurs définissent avec `TrustedUserCAKeys` et Authorized Principals quelles CA et quels principals s’appliquent à quel compte. Les clients lient une Host-CA dans `known_hosts` avec `@cert-authority` à des modèles de noms.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Zertifikatsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">ssh-keygen -L -f $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh-keygen -lf $env:USERPROFILE\\.ssh\\id_admin_ed25519-cert.pub
ssh -G $env:SSH_ALIAS | Select-String 'hostname|user|identityfile|certificatefile|proxyjump'
ssh -vvv $env:SSH_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh-keygen -L -f "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh-keygen -lf "$HOME/.ssh/id_admin_ed25519-cert.pub"
ssh -G "$SSH_ALIAS" | grep -E '^(hostname|user|identityfile|certificatefile|proxyjump) '
ssh -vvv "$SSH_ALIAS"</code></pre>
  </div>
</div>

[`grep`](https://www.gnu.org/software/grep/manual/grep.html) et [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) filtrent uniquement la configuration client effective. La sortie de débogage de `ssh -vvv` peut contenir des noms d’utilisateur, chemins, infrastructures cibles et fingerprints proposés ; elle est traitée comme un artefact de diagnostic sensible.

## ssh-agent et Agent Forwarding

Un agent conserve des clés privées ou des handles de signature déverrouillés et répond aux demandes de signature par un socket local. Les processus ayant accès à ce socket peuvent abuser de l’agent sans exporter la clé privée. La passphrase et la clé matérielle ne limitent donc pas toute compromission de l’agent.

Agent Forwarding transmet le canal de l’agent à un hôte distant. Root ou un processus compromis sur cet hôte peut soumettre des demandes de signature pendant la session. OpenSSH avertit explicitement de ce risque ; `ProxyJump` transporte la connexion via un bastion sans y fournir le socket de l’agent ([ssh(1) – `-A` et `-J`](https://man.openbsd.org/ssh)).

Lorsqu’une cible n’est accessible que par un Jump Host, les modèles privilégiés sont :

- `ProxyJump`/`ProxyCommand` sans stockage d’identifiants sur le bastion ;
- certificats SSH de courte durée avec des principals spécifiques à la cible ;
- clé distincte par domaine de confiance ;
- courtage de session contrôlé avec audit, si l’interactivité doit être relayée de manière centralisée.

Après une authentification réussie, SSH ouvre des canaux logiques au sein de la même connexion. Shell, commande individuelle, SFTP et Port Forwarding partagent ainsi transport et identité, mais restent des opérations distinctes.

## Connection Protocol et canaux

Après une authentification réussie, SSH ouvre des canaux logiques. La RFC 4254 définit le fenêtrage, les données de canal, les requêtes et les types. Une session peut demander un PTY, un shell, une commande `exec` individuelle ou un sous-système tel que SFTP. Le TCP Forwarding ouvre des canaux `direct-tcpip` ou `forwarded-tcpip` ([RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)).

Le multiplexage permet à plusieurs connexions client de partager un transport existant. Cela réduit le coût de KEX et d’authentification, mais augmente l’impact d’un Control Socket compromis. `ControlPath`, les autorisations, la persistance et la liaison à la cible sont définis explicitement.

Les keepalives existent à plusieurs niveaux : TCPKeepAlive vérifie le chemin TCP, `ServerAliveInterval`/`ClientAliveInterval` utilisent des messages SSH. Ils n’empêchent pas les blocages applicatifs et des valeurs agressives peuvent interrompre des sessions lors de brèves perturbations réseau.

## Port Forwarding et frontières de confiance

Le Local Forwarding (`-L`) ouvre un listener côté client et se connecte du réseau serveur vers la cible. Le Remote Forwarding (`-R`) ouvre un listener côté serveur ou distant. Le Dynamic Forwarding (`-D`) fournit un proxy SOCKS. Le forwarding Unix-Socket/Streamlocal complète TCP.

Le forwarding peut contourner la segmentation. Un utilisateur sans privilèges shell peut malgré tout atteindre des interfaces d’administration internes ou des bases de données via une redirection autorisée. Les serveurs contrôlent cela avec `AllowTcpForwarding`, `PermitOpen`, `PermitListen`, `GatewayPorts`, les blocs Match et les options Authorized-Key ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Bastion und Tunnelprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection $env:BASTION_HOST -Port $env:SSH_PORT
ssh -J $env:BASTION_ALIAS -N -L 127.0.0.1:8443:$env:INTERNAL_HOST:443 $env:TARGET_ALIAS
Get-NetTCPConnection -LocalPort 8443 -State Listen
ssh -O check $env:TARGET_ALIAS</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz "$BASTION_HOST" "$SSH_PORT"
ssh -J "$BASTION_ALIAS" -N -L "127.0.0.1:8443:$INTERNAL_HOST:443" "$TARGET_ALIAS"
ss -lntp 'sport = :8443'
ssh -O check "$TARGET_ALIAS"</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient le chemin TCP vers le bastion. [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) confirment le listener local. Le tunnel `-N` n’exécute aucune commande distante ; l’autorisation et le [TLS](/kb/tls) de la cible restent des frontières distinctes.

## SFTP, SCP et droits de fichiers

SFTP est un protocole binaire de transfert de fichiers qui fonctionne habituellement comme sous-système SSH. Ce n’est pas du FTP sur SSH. OpenSSH peut exécuter `internal-sftp` sans binaire de serveur SFTP externe et limiter le transfert de fichiers via `ChrootDirectory` plus `ForceCommand internal-sftp` ([sftp(1)](https://man.openbsd.org/sftp), [sftp-server(8)](https://man.openbsd.org/sftp-server)).

La CLI `scp` utilisait historiquement le protocole distant de type SCP/RCP et utilise par défaut SFTP dans les versions modernes d’OpenSSH ; `-O` force le protocole historique. Les wildcards, le traitement par le shell distant et la compatibilité serveur diffèrent ([scp(1)](https://man.openbsd.org/scp)). Les runbooks indiquent donc explicitement la CLI, le mode de protocole et les métadonnées de fichiers souhaitées.

L’autorisation de transfert de fichiers comprend la propriété du chroot, les chemins cibles, les UID/GID ou ACL Windows, l’umask, les liens symboliques, les quotas et le dépôt atomique. Le chargement dans un répertoire de messagerie ou d’automatisation nécessite souvent une frontière de quarantaine/renommage plutôt que le traitement direct de fichiers partiellement écrits.

## Politique côté serveur et blocs Match

`sshd_config` est évalué de haut en bas ; pour chaque mot-clé, la première valeur obtenue s’applique en principe. `Match` modifie le champ d’application selon l’utilisateur, le groupe, l’hôte, l’adresse, LocalAddress, LocalPort ou RDomain. Les fichiers Include, valeurs par défaut de distribution et options de ligne de commande peuvent compléter le fichier principal visible ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

Une baseline robuste définit :

- les utilisateurs/groupes et réseaux sources autorisés ;
- les méthodes d’authentification requises et l’exception Break Glass ;
- l’accès root/administrateur et l’élévation de privilèges ;
- le forwarding, l’agent, X11, PTY et les sous-systèmes ;
- MaxAuthTries, LoginGraceTime, les limites de session/démarrage et les keepalives ;
- les Host Keys, algorithmes et le rekey ;
- les sources AuthorizedKeys/Principals ;
- la journalisation, les bannières et les hypothèses d’environnement ;
- les écarts spécifiques à Match.

Les « listes de durcissement » générales peuvent exclure des appliances ou être inefficaces si une valeur Match acceptée ultérieurement s’applique différemment. La configuration effective pour une connexion donnée est déterminante.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SSH-Serverpolicy und Logs">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">sshd.exe -t
sshd.exe -T -C user=$env:SSH_USER,host=$env:SSH_HOST,addr=$env:CLIENT_IP
Get-NetTCPConnection -LocalPort $env:SSH_PORT -State Listen
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 100</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sshd -t
sshd -T -C "user=$SSH_USER,host=$SSH_HOST,addr=$CLIENT_IP"
ss -lntp "sport = :$SSH_PORT"
journalctl --unit ssh --unit sshd --since '-60 minutes' --no-pager</code></pre>
  </div>
</div>

`sshd -t` valide la syntaxe et les clés, `sshd -T -C` produit la configuration effective pour les paramètres de test ([sshd(8)](https://man.openbsd.org/sshd)). [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) et [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) lisent les journaux de la plateforme. Un reload réussi et une seconde session d’administration testée précèdent la fermeture de la session existante.

## Brute Force, Rate Limits et frontières réseau

Les services SSH exposés à Internet reçoivent des tentatives automatisées de connexion. Pare-feu source, accès VPN/Zero-Trust, bastion, `AllowUsers`/`AllowGroups`, authentification forte et courte durée de vie des identifiants réduisent la surface d’attaque. Un port différent réduit le bruit des journaux, mais ne modifie ni l’authentification ni la vulnérabilité.

Les outils de blocage périmétrique peuvent limiter les tentatives répétées ; ils traitent des données de journal non fiables et nécessitent leurs propres règles d’allowlist, IPv6, NAT, lockout et récupération. OpenSSH `MaxStartups` limite de manière probabiliste les connexions parallèles non authentifiées, `PerSourceMaxStartups` et `PerSourcePenalties` peuvent prendre en compte le comportement de la source. La capacité et le risque de DoS sont testés dans des conditions réelles de proxy/NAT ([sshd_config(5)](https://man.openbsd.org/sshd_config)).

## Journalisation, audit et preuve de session

Les journaux SSH documentent généralement la source de connexion, l’utilisateur, la méthode d’authentification/le fingerprint, le début/la fin de session et certains événements de sous-système. Ils ne contiennent pas automatiquement chaque commande shell exécutée. L’historique shell est contrôlé par l’utilisateur et ne constitue pas un audit protégé contre la manipulation.

Pour l’administration privilégiée, les éléments suivants sont corrélés :

- identité source, terminal et décision d’accès avant SSH ;
- saut bastion/proxy et hôte cible ;
- décision Host Key/CA du client ;
- principal utilisateur, méthode d’authentification, fingerprint de clé ou numéro de série du certificat ;
- type de session/canal, début, fin et statut de sortie ;
- événements `sudo`/d’élévation de privilèges Windows et audit du système cible ;
- chemin de transfert de fichier, hash ou ticket de changement pour les déploiements sensibles.

L’enregistrement de session modifie la protection des données, la collecte de secrets et le risque opérationnel. Les mots de passe, tokens, contenus de courriel et clés privées ne doivent pas se retrouver sans contrôle dans les transcriptions. La conservation, l’accès et la protection contre la manipulation sont définis avant l’activation.

## Windows OpenSSH

Microsoft propose les clients et serveurs OpenSSH comme fonctionnalités Windows. Le serveur utilise le service `sshd`, la configuration sous `%ProgramData%\ssh\sshd_config`, les comptes Windows et les ACL ; les journaux peuvent apparaître dans le canal `OpenSSH/Operational` ([Microsoft Learn – Présentation d’OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview), [Microsoft Learn – Configuration du serveur OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)).

Selon la configuration, les clés des administrateurs Windows peuvent se trouver dans un fichier central `administrators_authorized_keys` dont l’ACL doit être stricte. Le shell, les sous-systèmes, la valeur de registre DefaultShell et la résolution des chemins diffèrent d’Unix. Une commande client portable n’implique pas une autorisation serveur identique.

## Sauvegarde, récupération et Break Glass

La récupération SSH ne comprend pas seulement `sshd_config` :

- Host Keys ou Host-CA avec clés privées protégées ;
- User-CA, émetteur de certificats, principals et listes de révocation ;
- Authorized Keys/Commands et leur source de service d’annuaire/de gestion de configuration ;
- politique serveur, fichiers Include, dépendances PAM/SSSD/AD et MFA ;
- pare-feu, bastion, DNS/SSHFP et source de temps ;
- compte Break Glass avec identifiant vérifié hors ligne ;
- console hors bande si SSH réseau échoue ;
- journaux d’audit et inventaire des clés/certificats.

La restauration de la même Host Key privée préserve l’identité du serveur, mais peut être dangereuse après une compromission. La récupération choisit donc entre **continuité** et **nouvel enrôlement avec rotation**. Un test de restauration utilise une adresse cible isolée et vérifie le principal d’hôte, les méthodes d’authentification, la politique Match, l’élévation de privilèges et la corrélation des journaux.

Le diagnostic lit l’établissement de connexion dans l’ordre : TCP, échange de versions, Key Exchange, Host Key, authentification utilisateur et ouverture de canal.

## Diagnostic par phase de protocole

SSH se déroule en phases claires. Déterminer d’abord si TCP, la négociation de clés, la vérification d’hôte, l’authentification utilisateur ou le canal est concerné évite des modifications non ciblées des clés et de la configuration serveur.

| Phase | Éléments probants typiques | Classe d’erreur fréquente |
|---|---|---|
| TCP | DNS, route, listener, SYN/ACK | pare-feu, port erroné, proxy/bastion |
| Version | client `-vvv`, journal serveur | implémentation incompatible ou étrangère |
| KEX | algorithmes négociés | aucun KEX/Cipher/MAC commun, problème Strict-KEX |
| Host Key | fingerprint, principal, CA/known_hosts | mauvais hôte, rotation, cible DNS/LB |
| Userauth | méthodes proposées, fingerprint/certificat, PAM | clé non autorisée, principal, ACL, MFA |
| Connection | requête de canal, PTY, sous-système, forwarding | ForceCommand, PermitOpen, SFTP/Chroot, politique |
| Processus distant | statut de sortie, stderr, environnement, droits | chemin shell, sudo, locale, application |

Un `Permission denied (publickey)` intervient après un transport et une confiance d’hôte réussis. Un timeout avant la bannière n’est pas un problème de clé. Un diagnostic orienté phases évite de modifier les identifiants ou listes d’algorithmes sans éléments probants.

## Histoire technique

Tatu Ylönen a développé SSH original en 1995 pour remplacer les outils non chiffrés d’accès distant et de transfert de fichiers. SSH-1 et SSH-2 différaient fondamentalement ; l’IETF SECSH Working Group a ensuite standardisé l’architecture SSH-2 dans les RFC 4251 à 4254 ([OpenSSH – Histoire](https://www.openssh.com/history.html), [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html)).

OpenSSH est né au sein du projet OpenBSD à partir de code SSH libre et a été porté sur d’autres systèmes d’exploitation. Il a réuni client, serveur, outils de clés, agent, SCP et SFTP, et a façonné de nombreuses extensions de facto, telles que son propre format de certificat ([Projet OpenSSH](https://www.openssh.com/)).

La cryptographie a évolué des anciennes combinaisons Diffie-Hellman, DSA, RSA/SHA-1, CBC et MAC séparés vers Curve25519/ECDH, Ed25519, RSA/SHA-2 et les méthodes AEAD. Parallèlement, la confiance opérationnelle est passée de clés individuelles distribuées manuellement à des authentificateurs matériels, des CA hôte/utilisateur, des certificats de courte durée et des bastions fondés sur l’identité. Les couches protocolaire sont restées reconnaissables ; l’exploitation de la confiance et du cycle de vie est devenue plus étendue.

## Liste de contrôle administrateur en un coup d’œil

Cette liste de contrôle résume le chemin de connexion, la confiance d’hôte, les droits utilisateur, l’exploitation des clés et la journalisation. Elle convient à l’approbation et à la vérification régulière d’un accès SSH.

| Question | Preuve opérationnelle |
|---|---|
| Quel chemin ? | client/contexte, DNS, IP source, Bastion/ProxyJump, IP cible, port, processus listener |
| Quelle identité serveur ? | fingerprint Host Key ou certificat d’hôte, principal, CA, validité, rotation |
| Quels algorithmes ? | procédures KEX, Hostkey, Cipher/AEAD et MAC négociées, exceptions Legacy |
| Qui s’authentifie ? | compte, principal, fingerprint de clé publique/numéro de série de certificat, chemin MFA/PAM, appareil source |
| Qu’est-ce qui est autorisé ? | groupes, bloc Match, options Authorized-Key, sudo/token, sous-système/commande |
| Quels canaux ? | Shell/Exec/SFTP, PTY, Local/Remote/Dynamic Forward, Agent/X11, PermitOpen |
| Quelle politique s’applique réellement ? | `sshd -T -C`, `ssh -G`, Includes, valeurs par défaut de distribution et contexte Match |
| Qu’est-ce qui est journalisé ? | connexion/authentification/session/canal, élévation de privilèges, transfert et corrélation centrale |
| Comment effectuer la rotation ? | Host Key/confiance CA parallèle, certificats utilisateur courts, inventaire et révocation |
| Comment restaurer ? | console OOB, CA hôte/utilisateur, politique, principals/keys, DNS/bastion, Break Glass et audit |

## Sources

- [RFC 4251 – Architecture du protocole SSH](https://www.rfc-editor.org/rfc/rfc4251.html)
- [Registre IANA des noms de service et numéros de port](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [RFC 4253 – SSH Transport Layer Protocol](https://www.rfc-editor.org/rfc/rfc4253.html)
- [RFC 9142 – Recommandations de méthodes de Key Exchange pour SSH](https://www.rfc-editor.org/rfc/rfc9142.html)
- [RFC 8308 – Négociation d’extensions dans SSH](https://www.rfc-editor.org/rfc/rfc8308.html)
- [RFC 9519 – Strict Key Exchange dans SSH](https://www.rfc-editor.org/rfc/rfc9519.html)
- [ssh(1)](https://man.openbsd.org/ssh)
- [sshd(8)](https://man.openbsd.org/sshd)
- [RFC 4255 – Enregistrements de ressources SSHFP](https://www.rfc-editor.org/rfc/rfc4255.html)
- [IETF-Draft – Format de certificat OpenSSH](https://datatracker.ietf.org/doc/html/draft-ietf-sshm-cert)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [ssh-keyscan(1)](https://man.openbsd.org/ssh-keyscan)
- [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)
- [ssh_config(5)](https://man.openbsd.org/ssh_config)
- [RFC 4252 – SSH Authentication Protocol](https://www.rfc-editor.org/rfc/rfc4252.html)
- [RFC 4256 – Authentification Keyboard Interactive](https://www.rfc-editor.org/rfc/rfc4256.html)
- [sshd_config(5)](https://man.openbsd.org/sshd_config)
- [RFC 8332 – Clés RSA avec signatures SHA-2](https://www.rfc-editor.org/rfc/rfc8332.html)
- [RFC 8709 – Ed25519 et Ed448 pour SSH](https://www.rfc-editor.org/rfc/rfc8709.html)
- [Prise en charge FIDO/U2F d’OpenSSH](https://www.openssh.com/txt/release-8.2)
- [ssh-add(1)](https://man.openbsd.org/ssh-add)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-service)
- [Manuel GNU grep](https://www.gnu.org/software/grep/manual/grep.html)
- [Microsoft Learn – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [RFC 4254 – SSH Connection Protocol](https://www.rfc-editor.org/rfc/rfc4254.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [sftp(1)](https://man.openbsd.org/sftp)
- [sftp-server(8)](https://man.openbsd.org/sftp-server)
- [scp(1)](https://man.openbsd.org/scp)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – Présentation d’OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn – Configuration du serveur OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenSSH – Histoire](https://www.openssh.com/history.html)
- [Projet OpenSSH](https://www.openssh.com/)
