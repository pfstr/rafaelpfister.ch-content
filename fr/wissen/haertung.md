---
title: "Renforcement : référentiels, limites de confiance et contrôles administratifs"
blatt: "haertung"
description: "Renforcement technique pour les administrateurs de messagerie et d’infrastructure : référentiels de sécurité, fonctionnalité minimale et moindre privilège, plan d’identité et de gestion, limites des réseaux et des protocoles de messagerie, clés, risques liés aux correctifs et à la chaîne d’approvisionnement, journalisation, détection de dérive et vérification."
fakten:
  - label: Objectif
    wert: réduire de manière contrôlée la surface d’attaque, la confiance implicite et le rayon d’impact
    href: https://csrc.nist.gov/pubs/sp/800/123/final
  - label: Principes de conception
    wert: Fail-safe Defaults · Complete Mediation · Least Privilege
    href: https://web.mit.edu/Saltzer/www/publications/protection/Basic.html
  - label: Référentiel
    wert: état cible documenté, approuvé et vérifiable
    href: https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software
  - label: Fonctionnalité minimale
    wert: uniquement les fonctions, ports, protocoles, logiciels et services nécessaires à l’activité
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
  - label: Modèle d’accès
    wert: aucune attribution de confiance implicite fondée uniquement sur l’emplacement réseau ou la propriété
    href: https://csrc.nist.gov/pubs/sp/800/207/final
  - label: Surfaces de contrôle
    wert: hôte · identité · gestion · réseau/protocole · application/données · chaîne d’approvisionnement/télémétrie
    href: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - label: Limites de messagerie
    wert: relais Internet · soumission/accès · distribution interne · administration
    href: https://csrc.nist.gov/pubs/sp/800/177/r1/final
  - label: Accès administrateur
    wert: personnel · fortement authentifié · droits minimaux · traçable
    href: https://www.cisecurity.org/controls/access-control-management
  - label: Réseau de gestion
    wert: zone d’administration restrictive, séparée des flux de données de production
    href: https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure
  - label: Application des correctifs
    wert: identifier · prioriser · obtenir · installer · vérifier l’installation
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Preuve de dérive
    wert: comparaison entre l’état cible et l’état réel de la configuration, des services, des comptes, des règles et des journaux
    href: https://www.cisecurity.org/controls/audit-log-management
  - label: Références
    wert: référentiel du fabricant · CIS Benchmark · BSI IT-Grundschutz · propre validation des risques
    href: https://www.cisecurity.org/cis-benchmarks
werbung:
  - tools
  - newsletter
ctaThemen:
  - haertung
  - messaging
  - security
translationSourceHash: f97c0de00da2f8c1a5d89407341fb9a46b9fc213b01559d4e297feb29c0a3c83
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:32:30.201Z
translationReview: automatic
---

# Renforcement : référentiels, limites de confiance et contrôles administratifs

Le renforcement consiste à amener un système de manière contrôlée vers un **état cible documenté, justifié et vérifiable**. Il limite les fonctions, les accès et les relations de confiance au minimum nécessaire à l’exploitation, sans compromettre de manière incontrôlée le service prévu. Le résultat n’est pas une liste aussi longue que possible d’options de sécurité activées, mais une architecture dans laquelle chaque interface accessible, chaque privilège et chaque flux de données possède une finalité nommée, un propriétaire et une preuve. Le NIST décrit en conséquence la sécurité des serveurs comme la sélection, la mise en œuvre et la maintenance continue de contrôles adaptés ; le CIS Control 4 exige des configurations sécurisées pour les actifs et les logiciels ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)).

Un paramétrage par défaut du produit n’est ni automatiquement non sécurisé ni automatiquement le bon référentiel de production. Les fabricants doivent couvrir de larges plages de fonctionnalités et de compatibilité. L’exploitant, en revanche, connaît l’exposition, les besoins de protection, les dépendances, les capacités de reprise et les risques résiduels acceptés. Un référentiel relie donc les recommandations du fabricant, une référence CIS ou BSI adaptée et sa propre décision d’architecture. Les écarts ne sont pas conservés tacitement, mais documentés avec leur cause, leur risque, leur contrôle compensatoire, leur propriétaire et leur date d’expiration. Les CIS Benchmarks sont des recommandations de configuration fondées sur le consensus ; le BSI distingue les exigences générales pour les serveurs des exigences liées à la messagerie ([CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks), [BSI SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

Le renforcement commence par le service réel et ses voies d’administration, non par une liste arbitraire de valeurs de registre. Les composants, les identités, les données et les chemins réseau sont d’abord inventoriés ; il en découle le référentiel, les exceptions et les contrôles vérifiables.

## De la liste de contrôle au modèle des surfaces de contrôle

Pour les administrateurs, un examen structuré par surfaces de contrôle est plus robuste qu’une unique liste de contrôle d’hôte. Le modèle suivant synthétise les contrôles de NIST SP 800-53 : gestion de configuration et fonctionnalité minimale, accès et authentification, protection des communications, intégrité du système, audit, ainsi que contrôles de chaîne d’approvisionnement et de reprise. Il s’agit d’un modèle de vérification, pas d’une norme supplémentaire ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [NIST SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

| Surface de contrôle | Objet protégé | Valeurs cibles typiques | Preuve opérationnelle |
|---|---|---|---|
| Hôte et environnement d’exécution | Système d’exploitation, conteneurs, services, droits sur les fichiers, fonctions du noyau/de l’environnement d’exécution | paquets minimaux, écouteurs minimaux, processus non privilégiés, droits de fichiers sécurisés | inventaire des services et ports, analyse du référentiel, contrôle d’intégrité |
| Identité et autorisation | Personnes, comptes de service, rôles, jetons, certificats | identité administrateur personnelle, MFA, moindre privilège, identités de service distinctes | revue des comptes/rôles, événements d’authentification et de privilèges |
| Plan de gestion | GUI, API, SSH, PowerShell, SNMP, canal de sauvegarde et de mise à jour | zone d’administration dédiée, protocoles chiffrés, refus par défaut, processus Break-Glass | chemins de gestion accessibles, journaux AAA, modifications de configuration |
| Réseau et protocoles | Écouteurs, sorties, TLS, DNS, chemins de relais, de soumission et d’accès | flux explicites par rôle, aucun protocole en clair inutile, certificats vérifiés | règles de pare-feu, tests de paquets/TLS, surveillance DNS et flux de messagerie |
| Application et données | file d’attente, boîte aux lettres, politique, analyseur, données temporaires, clés | rôles distincts, droits de fichiers restrictifs, paramètres sûrs par défaut, droits d’analyseur et de sortie limités | tests négatifs fonctionnels, journaux de file d’attente/politique, revue des secrets et clés |
| Chaîne d’approvisionnement et télémétrie | Images, paquets, signatures, dépendances, journaux, temps | artefacts pris en charge, provenance vérifiée, processus de correctifs, journaux centraux non altérés | inventaire, hachage/signature, rapport de correctifs et de dérive, test d’alerte |

Le modèle Zero Trust du NIST ajoute une limite importante : un utilisateur, un service ou un appareil ne reçoit pas de confiance uniquement parce qu’il se trouve sur le réseau interne ou appartient à l’organisation. L’authentification et l’autorisation sont évaluées avant l’accès à une ressource. La segmentation reste utile, mais ne remplace ni l’identité, ni la politique, ni une décision continue ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-haertung.svg?v=20260813" title="Interaktive Infografik: Härtungsmodell für Messaging-Systeme mit externen Mailgrenzen, Managementebene, Identität, Host, Anwendung, Daten, Lieferkette, Telemetrie und Verifikationsschleife" loading="lazy">
  <a href="/images/kb-interaktiv-haertung.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

## La pile technologique comme inventaire de renforcement

Le renforcement ne possède pas sa propre pile de langages de programmation ou de produits. Il s’applique à la **pile technologique effectivement exploitée**. L’inventaire comprend donc au minimum le firmware et l’hyperviseur, le système d’exploitation ou la base de conteneurs, l’environnement d’exécution et le langage de programmation, les serveurs web et de messagerie, les bibliothèques et analyseurs, la base de données, la file d’attente et le stockage objet, les composants d’identité et de clés, les protocoles de gestion ainsi que les chemins de journalisation et de mise à jour. NIST CM-8 exige un inventaire des composants du système ; CM-7 relie cet inventaire à la limitation aux fonctions nécessaires ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)).

Pour chaque couche, le fabricant, la provenance, le statut de support, les modules actifs, les privilèges, les écouteurs, les destinations de sortie, la source de configuration, le chemin de correctifs et l’objet de reprise sont consignés. Ce n’est qu’ainsi qu’une recommandation telle que « désactiver les services inutiles » peut être appliquée à un processus concret et à ses dépendances, sans endommager le flux de messagerie ou la capacité de restauration.

## Limites de confiance d’une plateforme de messagerie

Une plateforme de messagerie comporte plusieurs voies d’entrée techniquement différentes. Elles ne doivent pas être traitées par une règle unique telle que « uniquement les connexions authentifiées » :

- **Relais Internet :** un MTA accessible publiquement reçoit sur le port [SMTP](/kb/smtp) 25 des messages provenant de MTA non connus au préalable. Ici, la vérification des destinataires, la politique de relais, les états de protocole, les limites de ressources, la réputation et les contrôles de contenu limitent le risque ; l’authentification d’un utilisateur n’est pas le modèle de confiance général.
- **Soumission de messages :** les utilisateurs et applications remettent de nouveaux messages en tant qu’expéditeurs identifiés. La soumission sépare ce rôle du relais ; l’authentification, l’autorisation, les limites de débit et [TLS](/kb/tls) font partie de la politique.
- **Accès au courrier :** IMAP, POP ou HTTP accèdent aux données existantes des boîtes aux lettres. La RFC 8314 considère le texte en clair pour la soumission et l’accès au courrier comme obsolète et privilégie TLS implicite.
- **Chemins de service internes :** passerelles, annuaires, bases de données, stockages objet, files d’attente et analyseurs communiquent entre eux en tant que services. La localisation réseau seule n’est pas une preuve d’identité ; chaque connexion requiert un chemin de données et d’autorisations minimal et dirigé.
- **Gestion et mises à jour :** GUI d’administration, API, SSH, accès distant, sauvegarde et acquisition de logiciels ont un rayon d’impact plus important qu’un chemin client normal et appartiennent à une zone de gestion et de confiance distincte.

NIST SP 800-177 traite l’authentification de domaine, TLS et la cryptographie de contenu comme des mécanismes de sécurité complémentaires autour de SMTP, qui continue d’être utilisé. La RFC 8314 sépare délibérément le relais de la soumission et de l’accès. Il s’ensuit que le renforcement doit vérifier par rôle **qui est autorisé à initier une connexion, quelle identité elle porte, quelles données elle traite et où elle est autorisée à communiquer ensuite** ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final), [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

L’inventaire montre ce qui doit être protégé. Un référentiel le traduit en paramètres concrets, versionnés, testés et modifiés de manière traçable en cas d’exceptions justifiées.

## Cycle de vie du référentiel et écart contrôlé

Un référentiel efficace suit un cycle de vie :

1. **Inventorier :** recenser le produit, le rôle, la version du logiciel, les modules, les écouteurs, les comptes, les flux de données, les clés et les dépendances.
2. **Choisir la référence :** appliquer au rôle concret le référentiel du fabricant, un CIS Benchmark, un module BSI et les exigences légales.
3. **Adapter :** supprimer les règles non applicables, ajouter des règles plus strictes et justifier les écarts selon les risques.
4. **Piloter :** vérifier la fonction, les performances, le flux de messagerie, la surveillance, la sauvegarde et [Disaster Recovery](/kb/backup-dr) dans un environnement représentatif.
5. **Déployer de manière déclarative :** utiliser une GPO, la gestion de configuration, une image, la politique en tant que code ou l’API du fabricant plutôt que des modifications manuelles isolées.
6. **Vérifier en continu :** détecter les dérives, les nouveaux comptes, écouteurs, paquets, certificats, règles et modifications du référentiel.
7. **Mettre hors service :** supprimer de manière contrôlée les accès, DNS, certificats, clés, données, sauvegardes et la surveillance.

Le Security Compliance Toolkit de Microsoft peut enregistrer, analyser, comparer, modifier et appliquer sous forme de GPO les référentiels Windows recommandés. Il ne remplace pas l’adaptation : un référentiel est d’abord vérifié dans un groupe pilote quant à son fonctionnement et à ses effets secondaires. Il en va de même pour les recommandations CIS et BSI ([Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

## Fonctionnalité minimale : services, ports et logiciels

Le contrôle NIST CM-7 exige de configurer un système selon les capacités nécessaires à l’activité et d’interdire ou de restreindre les fonctions, ports, protocoles, logiciels ou services. La question technique n’est pas « Le port 443 est-il sécurisé ? », mais : **Quel processus écoute sur quelle adresse, pour quel rôle, depuis quelle zone et avec quel modèle de correctifs et d’identité ?** ([NIST SP 800-53 Rev. 5, CM-7](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

Les interfaces web inutiles, les points de terminaison de débogage, les protocoles de découverte, les écouteurs de bases de données locales et les services de gestion hérités sont désactivés. Les services nécessaires ne se lient autant que possible qu’aux interfaces prévues. Un MTA peut écouter publiquement sur SMTP, mais pas sa base de données. Un port d’API d’administration peut être nécessaire, mais n’appartient pas automatiquement à Internet. La CISA recommande, pour l’infrastructure de communication, de désactiver les services inutilisés ou non chiffrés tels que Telnet, FTP, TFTP, HTTP et les anciennes variantes de SNMP, et d’inventorier continuellement les services accessibles publiquement ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Inventorier les écouteurs et services actifs

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Service | Where-Object Status -eq Running |
  Sort-Object Name
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
systemctl list-units --type=service --state=running
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) et [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) affichent les écouteurs et processus locaux. [`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service) et [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) affichent les services actifs. La comparaison entre l’état cible et l’état réel nécessite ensuite une matrice approuvée des ports et services ; un écouteur inconnu est un constat, mais pas encore une analyse de cause.

Après la suppression des fonctions inutiles, restent les comptes et services qui sont réellement autorisés à agir. Leurs droits, voies de connexion et secrets déterminent la plus grande part de la surface d’attaque administrative.

## Identités, comptes et moindre privilège

Les comptes sont séparés par rôle : identité utilisateur normale, identité administrateur personnelle, compte de service non interactif et compte d’urgence strictement contrôlé. Les identifiants administrateur partagés empêchent une attribution fiable. Les comptes quotidiens durablement très privilégiés augmentent le rayon d’impact du phishing et de la compromission du navigateur ou du client. Le CIS Control 5 couvre les comptes utilisateur, administrateur et de service ; le CIS Control 6 couvre l’attribution, la maintenance et le retrait de leurs identifiants et privilèges ([CIS Control 5: Account Management](https://www.cisecurity.org/controls/account-management), [CIS Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)).

Une identité centralisée améliore les processus Joiner/Mover/Leaver, mais ne remplace pas un chemin d’urgence local. Une panne LDAP, Kerberos ou SSO ne doit pas rendre impossible l’accès de reprise autorisé. Les comptes Break-Glass constituent donc une exception volontairement réduite : documentés hors ligne, fortement protégés, non utilisés au quotidien, immédiatement signalés lorsqu’ils sont utilisés et régulièrement testés. Les identités de service ne reçoivent pas de connexion interactive et uniquement les droits, destinations réseau et secrets requis par leur tâche. Lorsque c’est possible, des jetons à courte durée de vie, des identités gérées ou des certificats sont préférés aux mots de passe statiques ; leur cycle de vie et leur reprise restent partie intégrante de l’exploitation.

La MFA réduit le risque lié aux mots de passe volés, mais ne remplace ni les droits minimaux ni une reprise sécurisée. Le modèle Zero Trust du NIST impose une décision d’accès pour le sujet et, le cas échéant, l’appareil avant la session ; la localisation réseau ou la possession seules ne suffisent pas ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

### Vérifier les comptes locaux et privilégiés

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokales Konteninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember -Group Administrators
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
getent passwd
getent group sudo wheel
```

  </div>
</div>

[`Get-LocalUser`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser) et [`Get-LocalGroupMember`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember) lisent les comptes Windows locaux et les appartenances aux groupes. [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) interroge les bases de données de service de noms configurées et peut donc afficher les comptes locaux ainsi que ceux résolus de manière centralisée. Une revue doit en outre recenser les rôles réels dans le produit, les jetons API, les clés SSH, les certificats et le Cloud IAM.

## Plan de gestion et chemins d’administration

Le plan de gestion peut modifier la configuration, les clés, le routage, les mises à jour et les journaux ; il mérite une limite plus stricte que le chemin des données utiles. La CISA recommande un réseau de gestion hors bande, physiquement ou logiquement séparé du flux de données opérationnel, des règles de refus par défaut, des postes de travail administrateur dédiés et une journalisation AAA centralisée. Les connexions de gestion latérales entre appareils doivent également être limitées ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

Pour les systèmes de messagerie, cela signifie :

- Les GUI d’administration, API, SSH et accès distant ne sont accessibles que depuis des zones d’administration définies ou via un chemin de bastion contrôlé.
- Les certificats, comptes et règles de pare-feu de gestion et de flux de messagerie sont gérés séparément.
- Les connexions sortantes du plan de gestion sont limitées aux cibles de mise à jour, d’identité, de temps, de journaux et de sauvegarde.
- Les modifications de configuration nécessitent une identité personnelle, si possible une MFA, un audit et, pour les risques élevés, une approbation selon le principe des quatre yeux.
- Un chemin d’urgence fonctionne sans la plateforme d’identité ou de gestion régulière, mais n’est pas exploité comme accès permanent dissimulé.

SSH n’est qu’un transport pour l’administration ; sa sécurité dépend de l’authentification, des groupes d’utilisateurs autorisés, des algorithmes de clés, du transfert, des droits sur les fichiers et des droits de destination. Vérifier la configuration effective du serveur plutôt que le seul fichier texte permet de détecter les inclusions et les valeurs par défaut.

### Afficher la configuration effective du serveur SSH

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für effektive OpenSSH-Konfiguration">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -T
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sshd -T
```

  </div>
</div>

[`Get-WindowsCapability`](https://learn.microsoft.com/powershell/module/dism/get-windowscapability) affiche le composant OpenSSH installé ; Microsoft documente les chemins et particularités de [`sshd_config`](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration). [`sshd -T`](https://man.openbsd.org/sshd) affiche la configuration effective. Les options ne doivent pas être définies aveuglément selon des listes de contrôle Internet : la disponibilité de l’accès d’urgence, les types de clés utilisés et l’automatisation font partie des tests.

## Chemins réseau : refus par défaut avec direction explicite

Une règle de pare-feu est documentée comme un contrat dirigé : **source, destination, protocole/port, initiateur, identité, finalité, propriétaire et date d’expiration**. « Le serveur de messagerie peut accéder à Internet » n’est pas une spécification technique. Un écouteur SMTP entrant requiert d’autres destinations de sortie qu’un worker de bac à sable antimalware ou l’API d’administration. Le filtrage de sortie limite le command-and-control, l’exfiltration et les téléchargements incontrôlés ; il doit tenir compte consciemment du DNS, du temps, de la vérification des certificats, des mises à jour et des destinations de distribution.

La segmentation réduit l’espace de mouvement après une compromission. Elle est particulièrement importante entre la périphérie Internet, le traitement du courrier, le stockage des boîtes aux lettres/données, l’annuaire, la gestion, la sauvegarde et la surveillance. Le modèle Zero Trust du NIST avertit en même temps de ne pas utiliser la position réseau comme unique fondement de confiance. La CISA recommande un refus par défaut pour les chemins de gestion et une zone séparée du trafic de données clients ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Vérifier le pare-feu hôte et la direction des règles

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Firewall-Regeln">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetFirewallRule -Enabled True |
  Select-Object DisplayName, Direction, Action, Profile
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nft list ruleset
```

  </div>
</div>

[`Get-NetFirewallProfile`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile) et [`Get-NetFirewallRule`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule) affichent les profils Windows et les règles actives. [`nft`](https://netfilter.org/projects/nftables/manpage.html) affiche les règles nftables, y compris les chaînes et la direction. La sortie est vérifiée par rapport à la matrice de flux de données approuvée ; une politique de refus par défaut sans les destinations DNS, de temps ou de certificats nécessaires n’est pas un état de renforcement réussi.

## Protocoles de messagerie et confiance de transport

Le relais, la soumission et l’accès nécessitent des règles TLS et d’authentification distinctes. Pour la soumission et l’accès, la RFC 8314 recommande TLS 1.2 ou supérieur et privilégie TLS implicite ; l’accès en clair ne devrait plus être proposé. Pour le relais SMTP, STARTTLS décrit en revanche une négociation saut par saut. Sans politique supplémentaire, un MTA expéditeur peut poursuivre la distribution en clair si TLS est indisponible. [DANE](/kb/tls) et MTA-STS créent des politiques de transport différentes et plus explicites ([RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

[SPF, DKIM et DMARC](/kb/mail-auth) authentifient les relations de domaine et la politique, et non les comptes utilisateur ou le contenu en tant que tel. [S/MIME et OpenPGP](/kb/verschluesselung) protègent certaines parties du message, mais ne changent rien à un accès administrateur non sécurisé ou à un stockage de clés compromis. Une vérification de renforcement maintient ces prestations de sécurité séparées et contrôle leurs dépendances : DNS, certificats, clés, temps, rapports et règles d’exception. NIST SP 800-177 situe précisément ces mécanismes complémentaires autour de SMTP et DNS ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final)).

### Vérifier l’accessibilité et le comportement TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail- und Management-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) et [`nc`](https://man.openbsd.org/nc) vérifient le chemin TCP. [`curl`](https://curl.se/docs/manpage.html) et [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) affichent STARTTLS, la chaîne de certificats et les erreurs. Seuls la politique MTA et les journaux indiquent si, en cas d’erreur, la distribution est différée, rejetée ou revenue au texte en clair.

## Application, données, analyseurs et clés

Les serveurs de messagerie traitent intentionnellement des formats complexes non fiables. Les messages MIME, archives, documents, images et HTML atteignent des analyseurs, scanners, convertisseurs, prévisualisations et bacs à sable. Le renforcement limite donc non seulement les ports réseau, mais aussi les droits des processus, l’accès au système de fichiers, le stockage temporaire, le CPU/la RAM/la taille de fichier, la récursion, l’exécutabilité et la sortie des composants d’analyse. Un scanner doit accéder à un objet à examiner, mais pas automatiquement à toutes les boîtes aux lettres, aux secrets d’administration ou à l’API de gestion.

Les files d’attente et répertoires temporaires contiennent des contenus confidentiels. Les droits sur les fichiers, le chiffrement, les règles de suppression et de conservation ainsi que les dumps de débogage sont explicitement vérifiés. Les journaux doivent rendre les états et décisions traçables, mais ne doivent pas enregistrer de mots de passe, jetons, clés privées ou contenu de messages inutile. Les clés sont séparées selon leur finalité : TLS, DKIM, S/MIME/OpenPGP, JWT/API et chiffrement des sauvegardes ont des cycles de vie, autorisations et règles de reprise différents. NIST SP 800-53 relie le moindre privilège, l’intégrité du système, la protection des communications et l’audit ; BSI APP.5.3 précise les besoins de protection pour les clients et serveurs de messagerie ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

## Correctifs, images et chaîne d’approvisionnement

La gestion des correctifs est une maintenance préventive, non une urgence occasionnelle. Le NIST définit le processus comme l’identification, la priorisation, l’acquisition, l’installation et la vérification des correctifs, mises à jour et mises à niveau. Pour les composants de messagerie et de gestion exposés, les informations de vulnérabilité, la surface d’attaque accessible, l’exploitation active, la criticité des données et les compensations disponibles doivent influer sur la priorité ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Le chemin de mise à jour est lui-même une limite de confiance. Les paquets, images, conteneurs, plug-ins, signatures antivirus et firmware d’appliance sont obtenus depuis des sources authentifiées et vérifiés au moyen de la signature du fabricant ou d’un hachage publié. Les dépendances et changements de référentiel font partie de l’inventaire. NIST SP 800-161 traite les risques liés aux produits et services dont l’exploitant ne peut qu’avec des limitations consulter ou contrôler le développement, l’intégration et la fourniture ([NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

Une mise à jour de renforcement est testée dans une étape représentative : démarrage, flux de messagerie, file d’attente, TLS, annuaire, politique, surveillance, sauvegarde et retour arrière. « Ne pas appliquer de correctifs parce que la messagerie est critique » échange un risque opérationnel connu contre un risque de sécurité croissant. Une meilleure conception crée de la redondance, des fenêtres de maintenance, des builds reproductibles et des voies de repli testées.

### Intégrité des artefacts et événements pertinents pour la sécurité

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefakt- und Ereignisprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-gateway-update.bin -Algorithm SHA256
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddHours(-4)} |
  Select-Object TimeCreated, Id, ProviderName, Message
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-gateway-update.bin
journalctl --since '-4 hours' --priority=notice..alert
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparent un artefact à un hachage attendu provenant d’une source de fabricant authentifiée. [`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) et [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) lisent les événements ; la détection en production nécessite en outre une politique d’audit correcte, une collecte centralisée, une synchronisation temporelle et des alertes définies.

Une configuration renforcée ne reste efficace que si les modifications, les contrôles échoués et les écarts deviennent visibles. C’est pourquoi la journalisation et la détection de dérive font partie de l’exploitation, et non seulement du contrôle ultérieur.

## Journalisation, télémétrie et dérive

Un état renforcé n’est pas durable sans observation. Les signaux pertinents comprennent notamment :

- les connexions réussies et échouées, l’utilisation de la MFA et de Break-Glass ;
- les modifications de comptes, rôles, jetons, certificats et clés ;
- les modifications de configuration et les écarts par rapport au référentiel ;
- les nouveaux écouteurs, services, paquets, tâches, conteneurs ou destinations sortantes ;
- les rejets du pare-feu, les connexions de sortie inattendues et les accès de gestion ;
- les erreurs de TLS, DNS, politique SMTP, file d’attente et authentification de messagerie ;
- les capteurs désactivés, les lacunes de journaux, les pénuries de stockage et les écarts de temps.

Les journaux sont collectés de manière centralisée et protégés contre les accès afin qu’un hôte compromis ne puisse pas simplement supprimer ses traces avec l’état du système. Le CIS Control 8 exige un processus de gestion des journaux, un stockage suffisant, une heure standardisée, des journaux d’audit détaillés et centralisés ainsi que des revues. Une alerte n’est considérée comme implémentée que lorsqu’un événement contrôlé la déclenche, que le service responsable la voit et qu’un runbook conduit à la réaction ([CIS Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)).

La détection de dérive compare l’état réel au référentiel versionné. La comparaison couvre davantage que les hachages de fichiers : configuration effective, comptes, groupes, rôles IAM, certificats, règles de pare-feu, écouteurs, services, paquets installés, images, tâches planifiées et politique du fournisseur. Les modifications d’urgence sont reportées ou annulées automatiquement ; sinon, l’état d’exception « temporaire » devient la nouvelle valeur par défaut non documentée.

## Évolution technique

Saltzer et Schroeder ont formulé en 1975 des principes fondamentaux de protection tels que des mécanismes petits et simples, des valeurs par défaut sûres, une vérification complète de l’autorisation, la séparation des privilèges et le moindre privilège. Leur point de départ n’était pas un système d’exploitation particulier, mais l’architecture d’une diffusion contrôlée de l’information dans des systèmes multi-utilisateurs ([Saltzer/Schroeder: Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)).

Avec la généralisation des serveurs réseau, le renforcement s’est également déplacé vers les services distants, les protocoles, l’application de correctifs, l’audit et la maintenance sécurisée des configurations. NIST SP 800-123 a synthétisé systématiquement cette pratique des serveurs en 2008. Les CIS Benchmarks fondés sur le consensus, les modules BSI-Grundschutz et les référentiels des fabricants ont rendu les configurations cibles sûres plus reproductibles et comparables ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

Les architectures Cloud, SaaS, API et hybrides ont ensuite affaibli l’hypothèse d’un périmètre interne clairement défini. NIST SP 800-207 a décrit en 2020 Zero Trust comme une architecture orientée ressources sans confiance implicite fondée sur la localisation réseau ou la propriété. En parallèle, la chaîne d’approvisionnement logicielle, la provenance des images et la dérive automatisée des référentiels sont devenues des surfaces de contrôle à part entière. Le renforcement moderne combine donc la minimisation classique des hôtes avec l’identité, la politique de service à service, la configuration déclarative, les preuves de chaîne d’approvisionnement, la télémétrie et la reprise testée ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

## Liste de contrôle administrateur

Le renforcement n’est terminé que lorsque les mesures choisies sont vérifiables en exploitation normale et en cas de reprise. La liste de contrôle relie donc configuration, responsabilité et preuve.

- [ ] Le rôle, les besoins de protection, les flux de données et les limites de confiance du système sont documentés.
- [ ] Les recommandations du fabricant, CIS et BSI ont été mappées sur un référentiel versionné.
- [ ] Chaque écart possède une justification, un contrôle compensatoire, un propriétaire et une date d’expiration.
- [ ] Les écouteurs, services, paquets, modules et destinations sortantes sont réduits au minimum nécessaire.
- [ ] Les identités utilisateur, administrateur, de service et Break-Glass sont séparées et régulièrement vérifiées.
- [ ] Les accès administrateur utilisent une identité personnelle, la MFA, des droits minimaux et un audit centralisé.
- [ ] Le plan de gestion et le flux de messagerie de production se trouvent dans des zones distinctes et restrictives.
- [ ] Les chemins de relais, de soumission, d’accès et de service interne disposent de leurs propres politiques TLS, d’authentification et de limites de débit.
- [ ] Les analyseurs, scanners, données temporaires, files d’attente, clés et secrets ont des droits de processus et de fichiers minimaux.
- [ ] Les correctifs et images proviennent de sources authentifiées ; leur provenance et intégrité sont vérifiées.
- [ ] Les tests du référentiel, du flux de messagerie, de la sauvegarde et du retour arrière sont exécutés avant le déploiement à grande échelle.
- [ ] Les journaux sont centralisés, cohérents dans le temps, protégés contre les modifications et reliés à des alertes testées.
- [ ] La dérive des comptes, de la configuration, des règles, des services, des certificats et des logiciels est détectée automatiquement.
- [ ] La reprise et l’accès d’urgence ont été testés en pratique dans les conditions renforcées.

## Sources

- [NIST – SP 800-123, Guide de la sécurité générale des serveurs](https://csrc.nist.gov/pubs/sp/800/123/final)
- [CIS – Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)
- [CIS – Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [BSI – SYS.1.1 Serveur général](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3)
- [BSI – APP.5.3 Client et serveur de messagerie généraux](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)
- [NIST – SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST – SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)
- [NIST – SP 800-207, Architecture Zero Trust](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST – SP 800-177 Rev. 1, Courrier électronique digne de confiance](https://csrc.nist.gov/pubs/sp/800/177/r1/final)
- [IETF RFC 8314 – TLS pour la soumission et l’accès aux e-mails](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [CIS – FAQ Benchmarks](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)
- [CISA – Conseils pour une visibilité et un renforcement accrus](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Pages de manuel Linux – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [CIS – Control 5: Account Management](https://www.cisecurity.org/controls/account-management)
- [CIS – Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)
- [Microsoft Learn – Get-LocalUser](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser)
- [Microsoft Learn – Get-LocalGroupMember](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember)
- [Pages de manuel Linux – getent](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft Learn – Get-WindowsCapability](https://learn.microsoft.com/powershell/module/dism/get-windowscapability)
- [Microsoft Learn – Configuration du serveur OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenBSD – page de manuel sshd](https://man.openbsd.org/sshd)
- [Microsoft Learn – Get-NetFirewallProfile](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile)
- [Microsoft Learn – Get-NetFirewallRule](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule)
- [Netfilter – page de manuel nft](https://netfilter.org/projects/nftables/manpage.html)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 7672 – Sécurité SMTP via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – Sécurité stricte de transport SMTP MTA](https://datatracker.ietf.org/doc/html/rfc8461)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – page de manuel nc](https://man.openbsd.org/nc)
- [curl – page de manuel de ligne de commande](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [NIST – SP 800-40 Rev. 4, Gestion des correctifs d’entreprise](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [NIST – SP 800-161 Rev. 1, Gestion des risques de cybersécurité de la chaîne d’approvisionnement](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [CIS – Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)
- [Saltzer/Schroeder – Principes fondamentaux de protection de l’information](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)
