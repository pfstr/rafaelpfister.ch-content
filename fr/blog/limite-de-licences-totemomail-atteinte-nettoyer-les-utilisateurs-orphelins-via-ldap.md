---
title: "Limite de licences Totemomail atteinte : nettoyer les utilisateurs orphelins via LDAP"
navTitle: "Limite de licences atteinte"
description: "Les comptes AD désactivés restent dans totemomail et continuent d’occuper des licences. Avec un accès LDAPS vérifié et l’agent de nettoyage, Active Directory devient la source de référence."
date: "2026-06-26"
kategorie: "Totemomail"
timeToRead: "9 min de lecture"
themen:
  - "totemomail"
slug: "limite-de-licences-totemomail-atteinte-nettoyer-les-utilisateurs-orphelins-via-ldap"
translationOf: "totemomail-licensed-user-limit-ldap-cleanup"
url: "https://rafaelpfister.ch/fr/blog/limite-de-licences-totemomail-atteinte-nettoyer-les-utilisateurs-orphelins-via-ldap"
---

# Limite de licences Totemomail atteinte : nettoyer les utilisateurs orphelins via LDAP

Le message *« The licensed user limit has been reached »* ne signifie pas que le flux de messagerie s’arrête immédiatement. Il indique une sous-licence. Dans les environnements exploités depuis longtemps, la cause n’est généralement pas une croissance soudaine, mais les anciens collaborateurs : le compte AD a été désactivé, mais l’utilisateur interne dans totemomail est resté en place et continue d’occuper une licence.

La solution durable consiste en une synchronisation LDAP régulière avec Active Directory. Les étapes suivantes configurent la connexion et l’agent de nettoyage, puis vérifient l’ensemble du chemin avant la première exécution en production. Les noms d’hôte, DN et comptes de service marqués par `example.com` sont des espaces réservés et doivent être adaptés à votre environnement.

## Quels utilisateurs occupent une licence

Totemomail distingue deux catégories d’utilisateurs. Seuls les utilisateurs internes comptent dans la limite de licences.

| Type d’utilisateur | Description | Soumis à licence |
| --- | --- | --- |
| Internal Users | Utilisateurs de l’organisation qui envoient et reçoivent des messages chiffrés | Oui |
| External Users | Partenaires de communication externes (WebMail, PDF, S/MIME, PGP) | Non |


Un utilisateur interne est créé dès qu’il communique pour la première fois via la passerelle. Cela se produit automatiquement. En revanche, sa suppression ne l’est pas : lorsqu’un collaborateur quitte l’organisation, vous désactivez habituellement son compte AD. L’entrée totemomail reste toutefois présente. Au fil des années, des comptes orphelins s’accumulent ainsi et continuent d’occuper des licences.

### Indicateur de statut

Vous trouverez l’état actuel sous **Settings → Overview → User Information**.

![](../images/953te2zhdJ61lxda1mj04QrlQA.png)

*Available Users est à* `*-17*`*. Les 4017 utilisateurs internes disposent d’un nombre inférieur de licences.*

Les lignes importantes :

-   **Internal users** (`4017`) : utilisateurs internes créés
    
-   **Internal blocked users** (`14`) : bloqués, mais toujours soumis à licence
    
-   **Available Users** (`-17`) : licences disponibles ; une valeur négative signifie une sous-licence
    

Dès que *Available Users* passe sous zéro, l’avertissement apparaît sur la cloche :

![](../images/lcL4owxA3iEdg3L9ZFd2bIioE.png)

*« The licensed user limit has been reached. » Le flux de messagerie continue, mais le message reste affiché en permanence.*

Important : la sous-licence ne bloque pas le flux de messagerie. Il s’agit d’un état lié aux licences, pas d’un problème technique. Vous avez donc le temps de mettre en place une solution propre, mais ne devriez pas ignorer cet état durablement.

## De la mesure immédiate à la solution durable

### Suppression manuelle

Vous pouvez rechercher et supprimer individuellement les utilisateurs internes sous **Internal Users**. Cela corrige la situation immédiate, mais le problème réapparaît après quelques mois. Avec plusieurs milliers de comptes, cette approche n’est pas satisfaisante.

### Connexion LDAP avec agent de nettoyage

La solution viable consiste à connecter Active Directory via LDAP. Un agent compare régulièrement les utilisateurs internes avec l’annuaire et supprime ou désactive les comptes qui n’existent plus dans AD. AD devient ainsi la source de référence, et votre processus de départ dans AD assure en même temps l’hygiène des licences.

## Principes de base LDAP

| Terme | Signification |
| --- | --- |
| DN (Distinguished Name) | Chemin unique vers un objet, par ex. `CN=John Doe,OU=Users,DC=corp,DC=example,DC=com` |
| Base DN / Search Base | Racine de la recherche, par ex. `DC=corp,DC=example,DC=com` |
| Bind DN | Compte avec lequel totemomail s’authentifie auprès d’AD |
| Filtre | Expression de recherche LDAP, par ex. `(&(objectClass=user)(sAMAccountName=jdoe))` |


### Ports

| Port | Protocole | Utilisation |
| --- | --- | --- |
| 389 | LDAP | non chiffré / STARTTLS |
| 636 | LDAPS | LDAP via TLS |
| 3268 | Global Catalog | recherche à l’échelle de la forêt, non chiffrée |
| 3269 | Global Catalog SSL | recherche à l’échelle de la forêt via TLS |


Dans un environnement à domaine unique, le port 636 vers un contrôleur de domaine suffit. Si vous exploitez une forêt avec plusieurs domaines, seul le Global Catalog (port 3269) fournit des résultats à l’échelle de la forêt. Un contrôleur de domaine sur le port 636 ne connaît que les objets de son propre domaine et répond aux recherches hors de sa partition par un referral, un détail souvent négligé dans les environnements multi-domaines.

### userAccountControl

L’état désactivé d’un compte AD se trouve dans le champ de bits `userAccountControl`. L’indicateur `ACCOUNTDISABLE` a la valeur `2`. La règle de correspondance LDAP `1.2.840.113556.1.4.803` (`LDAP_MATCHING_RULE_BIT_AND`) permet d’évaluer des bits individuels :

```text
# Aktive Benutzer
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Deaktivierte Benutzer
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))
```

## Étape 1 : compte de service dans AD

Pour la connexion, créez un compte dédié avec des droits de lecture uniquement. N’utilisez pas de compte administrateur. L’utilisateur de liaison doit uniquement pouvoir lire AD.

```powershell
New-ADUser -Name "svc-totemomail-ldap" `
  -SamAccountName "svc-totemomail-ldap" `
  -UserPrincipalName "svc-totemomail-ldap@corp.example.com" `
  -Path "OU=Comptes de service,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Mot de passe") `
  -PasswordNeverExpires $true `
  -Enabled $true
```

Un utilisateur de domaine standard peut déjà lire AD ; le compte n’a donc besoin d’aucun droit supplémentaire. Pour le mot de passe, utilisez de préférence une valeur longue et aléatoire, stockée dans votre coffre-fort de mots de passe.

Si votre politique de sécurité le prévoit, vous pouvez aussi utiliser un gMSA (Group Managed Service Account). Toutefois, totemomail attend un Bind DN et un mot de passe ; en pratique, un compte de service classique avec `PasswordNeverExpires` est donc généralement utilisé.

## Étape 2 : vérifier la connexion LDAP en ligne de commande

Avant de configurer quoi que ce soit dans totemomail, vérifiez la connexion LDAP en ligne de commande. C’est l’étape que la plupart des gens ignorent. Si `ldapsearch` fonctionne, la connexion dans totemomail fonctionnera également. Si le test échoue, vous saurez au moins où se situe le problème au lieu de devoir deviner dans l’interface de totemomail.

### 2.1 Vérification du port

Sous Linux, par exemple depuis l’appliance totemomail :

```bash
nc -vz dc01.corp.example.com 636
nmap -p 389,636,3268,3269 dc01.corp.example.com
```

Sous Windows avec PowerShell :

```powershell
Test-NetConnection -ComputerName dc01.corp.example.com -Port 636
```

Si aucune connexion ne peut être établie ici, vous avez un problème de pare-feu ou de routage, et non un problème LDAP.

### 2.2 Vérifier le certificat TLS

En pratique, LDAPS échoue le plus souvent à cause du certificat. Examinez donc ce que fournit le contrôleur de domaine :

```bash
openssl s_client -connect dc01.corp.example.com:636 -showcerts </dev/null
```

Vérifiez deux éléments :

-   `**subject=**` **/** `**issuer=**` : le nom d’hôte dans le certificat (CN ou SAN) doit correspondre au nom d’hôte utilisé pour la connexion. Si vous vous connectez via l’adresse IP, la vérification échoue lorsque le certificat ne contient que le FQDN.
    
-   `**Verify return code: 0 (ok)**` : l’autorité de certification émettrice doit être connue de totemomail. Avec une CA d’entreprise interne, vous devez importer son certificat racine ou émetteur dans le magasin de confiance de totemomail.
    

### 2.3 Liaison et recherche avec ldapsearch

`ldapsearch` fait partie de `ldap-utils` (Debian/Ubuntu) ou de `openldap-clients` (RHEL) :

```bash
ldapsearch -x \
  -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Comptes de service,DC=corp,DC=example,DC=com" \
  -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(sAMAccountName=jdoe))" \
  dn sAMAccountName mail userAccountControl
```

| Option | Signification |
| --- | --- |
| `-x` | Authentification simple (Bind DN et mot de passe) |
| `-H` | URI LDAP incluant le schéma (`ldaps://`) et le port |
| `-D` | Bind DN |
| `-W` | Demander le mot de passe de manière interactive |
| `-b` | Base de recherche |
| ensuite | Filtre, puis attributs à renvoyer |


Si la requête renvoie l’objet avec ses attributs, la connexion est établie. Pour déterminer combien de comptes sont désactivés dans AD, utilisez le filtre de bits :

```bash
ldapsearch -x -H ldaps://dc01.corp.example.com:636 \
  -D "CN=svc-totemomail-ldap,OU=Comptes de service,DC=corp,DC=example,DC=com" -W \
  -b "DC=corp,DC=example,DC=com" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName | grep -c sAMAccountName
```

### 2.4 Outils sous Windows

`**ldp.exe**` est l’outil LDAP graphique de Microsoft, présent sur chaque contrôleur de domaine et inclus dans RSAT. Vous vous connectez via `Connection → Connect` (hôte, port 636, activer SSL), vous vous authentifiez avec `Connection → Bind` et parcourez l’arborescence de l’annuaire avec le Base DN via `View → Tree`.

Sans RSAT, vous pouvez utiliser l’ADSI Searcher dans PowerShell :

```powershell
$searcher = [adsisearcher]"(&(objectClass=user)(sAMAccountName=jdoe))"
$searcher.SearchRoot = [adsi]"LDAP://dc01.corp.example.com/DC=corp,DC=example,DC=com"
$searcher.FindOne().Properties
```

Avec RSAT et le module AD, c’est plus court :

```powershell
Get-ADUser -Server dc01.corp.example.com `
  -SearchBase "DC=corp,DC=example,DC=com" `
  -Filter "Enabled -eq '$true'" |
  Measure-Object
```

De manière classique avec `dsquery`, disponible sur chaque contrôleur de domaine :

```bash
dsquery user -disabled -limit 0
```

Ne poursuivez dans totemomail que lorsqu’un de ces tests s’exécute correctement.

## Étape 3 : configurer la connexion LDAP dans totemomail

Créez l’annuaire LDAP dans l’interface d’administration sous **Directories / LDAP**. Reprenez exactement les valeurs que vous avez testées auparavant :

| Champ | Exemple de valeur |
| --- | --- |
| Host / URL | `ldaps://dc01.corp.example.com:636` |
| Bind DN | `CN=svc-totemomail-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com` |
| Bind Password | Mot de passe du compte de service |
| Base DN | `DC=corp,DC=example,DC=com` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` |
| Login Attribute | `sAMAccountName` (ou `mail` ou `userPrincipalName`) |


Si vous utilisez LDAPS avec une CA interne, vous devez importer son certificat racine ou émetteur dans le magasin de confiance de totemomail. Sinon, la négociation TLS échoue avec « certificate verify failed », même si `ldapsearch` avec `-x` fonctionnait auparavant : sous cette forme, `ldapsearch` ne vérifie pas strictement le certificat.

Après l’enregistrement, lancez le test de connexion intégré. Il confirme la liaison.

## Étape 4 : créer l’agent de nettoyage

Sous **Maintenance → Agents → Add**, créez un agent de type **« Check presence of internal users in directories »**.

### 4.1 Onglet « Schedule »

![](../images/oSiutQSlKTW0tMY5HUtWCMGuXQ.png)

*Ici, l’agent s’exécute mensuellement le 1er à 00:30. « Agent runs on server » permet de définir le nœud exécutant dans le cluster.*

| Champ | Recommandation | Justification |
| --- | --- | --- |
| The agent should run | `monthly`, jour `1`, `00:30` | en dehors des heures de bureau ; une exécution mensuelle suffit pour l’hygiène des licences |
| Agent enabled | activer seulement après le test | voir étape 5 |
| Produced emails are not sent but cached in a queue | activer pour la première exécution | exécution de test sans envoi d’e-mails |
| Agent runs on server | un nœud du cluster | la tâche ne doit s’exécuter que sur un seul nœud |


### 4.2 Onglet « Parameters »

![](../images/Y6XzxZWGYIcZoJnZkFL0vUHXxQ.png)

*Les paramètres contrôlent quels utilisateurs internes sont supprimés, désactivés ou créés.*

| Paramètre | Recommandation | Effet |
| --- | --- | --- |
| Delete inactive users that are not found in a directory? | activer | Les utilisateurs internes inactifs sans entrée AD sont supprimés. C’est le cœur du nettoyage des licences. |
| Delete blocked users that are not found in a directory? | activer | Les utilisateurs internes bloqués sans entrée AD sont également supprimés |
| Delete administrators? | laisser vide | Les comptes administrateur ne doivent pas être supprimés automatiquement |
| Only set users found in the defined groups to inactive | facultatif | Les utilisateurs sont définis comme inactifs plutôt que supprimés. Un `!` placé devant exclut les membres du groupe indiqué. Séparez les DN avec `;`. |
| Additional filter attribute | facultatif | attribut supplémentaire pour la recherche dans l’annuaire, par ex. `proxyAddresses` |
| Delete inactive/blocked users that are found in the defined groups | laisser vide | ne s’applique que si le paramètre de groupe est défini |
| Create users based on group membership | facultatif | crée de nouveaux utilisateurs internes à partir de l’appartenance à des groupes AD. Séparez plusieurs groupes avec `;`. |


La négation dans le champ *« Only set users found in the defined groups to inactive »* s’effectue avec un `!` devant un DN de groupe. Les membres de ce groupe sont exclus de l’action :

```text
CN=Mitarbeiter,OU=Groups,DC=corp,DC=example,DC=com;!CN=Dienstkonten,OU=Groups,DC=corp,DC=example,DC=com
```

Dans cet exemple, les utilisateurs du groupe *Collaborateurs* sont définis comme inactifs lorsqu’ils sont absents d’AD, tandis que les membres du groupe *Comptes de service* ne sont pas modifiés.

## Étape 5 : exécution de test et validation

Ne laissez pas l’agent s’exécuter sur les données de production sans test préalable. Procédez plutôt dans cet ordre :

1.  **Activer le mode queue** : via l’option *« Produced emails are not sent but cached in a queue »*. L’agent détermine les actions prévues sans envoyer d’e-mails.
    
2.  **Exécuter manuellement** et analyser le journal de l’agent : combien d’utilisateurs seraient concernés, et des comptes inattendus tels que des boîtes aux lettres fonctionnelles figurent-ils dans la liste ?
    
3.  **Vérifier la cohérence avec** `**ldapsearch**` : le nombre d’utilisateurs non trouvés dans AD doit correspondre à votre requête LDAP manuelle.
    
4.  Si le résultat est correct, désactivez le mode queue, activez *Agent enabled* et mettez la planification en service.
    
5.  Après la première exécution en production, vérifiez à nouveau **Settings → Overview → User Information**. *Available Users* devrait alors repasser dans la plage positive.
    

## Dépannage

| Symptôme | Cause | Mesure |
| --- | --- | --- |
| `Can't contact LDAP server` | Port 636 inaccessible / mauvais hôte | vérifier avec `Test-NetConnection` ou `nc -vz`, contrôler le pare-feu |
| `Invalid credentials (49)` | Bind DN ou mot de passe incorrect | indiquer le Bind DN comme DN complet, pas comme `user@domain` |
| `certificate verify failed` | CA inconnue du magasin de confiance | importer la CA racine ou émettrice |
| Incohérence de nom d’hôte dans TLS | connexion via IP au lieu du FQDN | utiliser le CN/SAN du certificat comme hôte |
| `Referral (10)` | la recherche dépasse les limites du domaine | utiliser le Global Catalog sur le port 3269 au lieu du contrôleur de domaine sur 636 |
| Les utilisateurs désactivés ne sont pas détectés | filtre `userAccountControl` manquant | utiliser la règle de correspondance de bits `:1.2.840.113556.1.4.803:=2` |
| L’agent supprime trop de comptes | filtre trop large / Base DN incorrect | tester en mode queue, restreindre le Base DN |


Avec l’option `-d 1`, `ldapsearch` fournit la sortie de débogage de l’établissement de la connexion :

```bash
ldapsearch -d 1 -x -H ldaps://dc01.corp.example.com:636 ...
```

Vous voyez ainsi si c’est la négociation TLS ou seulement la liaison qui échoue. L’interface de totemomail ne vous montre pas cette distinction derrière son message d’erreur générique.

## Sécurité

-   **Compte de service en lecture seule.** L’utilisateur de liaison a uniquement besoin de droits de lecture.
    
-   **LDAPS plutôt que LDAP.** Utilisez le port 636 ou 3269. LDAP sur le port 389 transmet le mot de passe de liaison en clair. Active Directory impose de toute façon de plus en plus des connexions sécurisées avec LDAP Channel Binding et Signing.
    
-   **Rotation des mots de passe.** `PasswordNeverExpires` est opérationnellement pratique. Documentez le compte et effectuez une rotation du mot de passe selon un plan établi.
    
-   **Surveillance.** Surveillez *Available Users* (idéalement avec des alertes), au lieu d’attendre l’avertissement de la cloche.
    
-   **Première exécution en mode queue.** Un filtre erroné peut affecter un grand nombre de comptes.
    

## La procédure sûre en quatre étapes

L’atteinte de la limite de licences n’est pas un défaut technique, mais la conséquence d’un processus de départ manquant. La solution durable est une synchronisation régulière avec Active Directory comme source de référence. L’ordre est déterminant :

1.  Vérifier la connexion LDAP en ligne de commande (`ldapsearch`, `openssl s_client`, `Test-NetConnection`)
    
2.  Configurer la connexion dans totemomail
    
3.  Valider l’agent en mode queue
    
4.  Mettre l’agent en production
    

En respectant cet ordre, vous résolvez le problème de licence immédiat et empêchez qu’il ne réapparaisse.

## Sources

1.  [totemo / Kiteworks – totemomail (Email Protection Gateway)](https://totemo.com/en/resources/downloads): Documentation produit sur totemomail (modèle de licence, connexion LDAP, agent de nettoyage) ; la technologie est poursuivie chez Kiteworks sous le nom Email Protection Gateway.
    
2.  [Microsoft Learn – « UserAccountControl property flags »](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties): Signification des indicateurs, notamment `ACCOUNTDISABLE` (0x0002) et `NORMAL_ACCOUNT`.
    
3.  [Microsoft Learn – « Search Filter Syntax »](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax): Filtre LDAP bit à bit via l’OID de règle de correspondance `1.2.840.113556.1.4.803` (LDAP\_MATCHING\_RULE\_BIT\_AND).
    
4.  [OpenLDAP – « ldapsearch » (page de manuel)](https://www.openldap.org/software/man.cgi?query=ldapsearch): Options d’appel (`-x`, `-H ldaps://`, `-D`, `-W`, `-b`) pour la liaison et la recherche.
    
5.  [Microsoft Learn – « Service overview and network port requirements »](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements): Ports LDAP 389/636 ainsi que ports Global Catalog 3268/3269.
