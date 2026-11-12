---
title: "Échec de connexion RDP : le mot de passe doit être modifié avant la première connexion"
navTitle: "RDP : modifier le mot de passe"
description: "Un nouveau compte avec un mot de passe initial, et la connexion Bureau à distance échoue avec « You must change your password before logging on the first time ». La cause est l’authentification au niveau du réseau, qui ne prévoit pas de changement de mot de passe. Il est néanmoins possible de modifier le mot de passe : via une autre session Windows, depuis son propre PC, avec PowerShell ou via RD Web Access."
date: "2026-11-11"
kategorie: "Client Windows"
timeToRead: "6 min de lecture"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "echec-de-connexion-rdp-le-mot-de-passe-doit-etre-modifie-avant-la-premiere-connexion"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/fr/blog/echec-de-connexion-rdp-le-mot-de-passe-doit-etre-modifie-avant-la-premiere-connexion
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:31:55.833Z
translationReview: automatic
---

# Échec de connexion RDP : le mot de passe doit être modifié avant la première connexion

Avec un compte nouvellement configuré dans l’environnement d’un client, j’ai de nouveau rencontré cette situation : le mot de passe initial est correct, mais la connexion Bureau à distance est malgré tout immédiatement interrompue. Le client Windows `mstsc` affiche le message suivant :

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

D’autres clients RDP et gestionnaires de connexions signalent le même état avec leur propre formulation, par exemple « The user password must be changed before logging on for the first time ». Pour un mot de passe expiré, le message est « This user account's password has expired. The password must change in order to logon ». La cause et la solution sont les mêmes dans les deux cas.

En bref : dans cette situation, il n’est pas possible de modifier le mot de passe via la connexion RDP. Il doit être modifié au préalable par un autre moyen, et dans la plupart des cas, une connexion réseau à un contrôleur de domaine suffit.

## Cause : NLA authentifie avant l’établissement de la session

Dans Active Directory, l’option **L’utilisateur doit changer son mot de passe à la prochaine ouverture de session** est activée sur le compte (techniquement : `pwdLastSet` = 0). Sur un poste de travail du domaine, cela ne pose aucun problème : l’écran de connexion détecte cet état et demande directement le nouveau mot de passe.

Avec RDP, cette étape est absente dès que l’authentification au niveau du réseau (NLA) est active. NLA authentifie l’utilisateur via CredSSP avant qu’une session avec écran de connexion ne soit créée sur le système cible. L’authentification avec un mot de passe qui doit être modifié échoue, et aucun dialogue de changement de mot de passe n’est prévu à ce stade. Le client interrompt la connexion et affiche uniquement le message. NLA est le réglage par défaut dans les versions actuelles de Windows et est imposée par stratégie de groupe dans les environnements durcis.

## Options pour l’utilisateur

Il existe quatre possibilités pour modifier le mot de passe sans utiliser la connexion RDP concernée. Le choix dépend de ce qui est accessible depuis son propre poste de travail.

| Possibilité | Condition |
|---|---|
| 1. Portail de mots de passe ou service desk | fourni par l’exploitant |
| 2. Session Windows existante dans le domaine | accès avec un autre compte ou à un autre système |
| 3. Dialogue de mot de passe sur son propre PC | contrôleur de domaine accessible, par exemple via VPN |
| 4. PowerShell avec `NetUserChangePassword` | TCP 445 accessible vers le contrôleur de domaine |

### Possibilité 1 : portail de mots de passe ou service desk

Dans les environnements plus importants, une procédure est généralement prévue pour les collaborateurs externes : un portail libre-service, une fonction de mot de passe dans le portail VPN ou Citrix, ou un système PAM. Si rien de tout cela n’est disponible, le service desk peut supprimer l’indicateur ou définir un mot de passe sans obligation de modification (voir la section destinée aux administrateurs).

### Possibilité 2 : session Windows existante dans le domaine

Si une session existe déjà sur un système du domaine, par exemple sur un hôte de rebond ou avec un second compte, le mot de passe peut y être modifié. Dans une session RDP, `Ctrl+Alt+End` ouvre l’écran de sécurité du système distant (`Ctrl+Alt+Del` serait interprété sur le PC local). Sous **Modifier un mot de passe**, le champ contenant le nom d’utilisateur est modifiable : saisissez-y le compte concerné sous la forme `DOMAIN\benutzer`, puis le mot de passe initial et deux fois le nouveau mot de passe.

### Possibilité 3 : dialogue de mot de passe sur son propre PC

Le même dialogue fonctionne également sur un PC qui n’est pas membre du domaine, à condition qu’il puisse joindre un contrôleur de domaine, par exemple via une connexion VPN. Appuyez sur `Ctrl+Alt+Del`, choisissez **Modifier un mot de passe** et remplacez le nom d’utilisateur local prérempli par `DOMAIN\benutzer`. Le PC doit pouvoir résoudre le nom du domaine via DNS ; sans les serveurs DNS de l’environnement cible, Windows ne trouve aucun contrôleur de domaine.

### Possibilité 4 : PowerShell avec NetUserChangePassword

L’API Windows `NetUserChangePassword` modifie un mot de passe directement sur un contrôleur de domaine. Elle vérifie elle-même l’ancien mot de passe lors de la modification et ne requiert aucune connexion préalable. Elle fonctionne donc aussi lorsque le mot de passe a expiré ou doit être modifié, ainsi que depuis un PC hors du domaine. Le script intègre la fonction via P/Invoke et demande les deux mots de passe de manière masquée :

```powershell
$api = Add-Type -Namespace Win32 -Name NetApi32 -PassThru -MemberDefinition @'
[DllImport("netapi32.dll", CharSet = CharSet.Unicode)]
public static extern uint NetUserChangePassword(
    string domainname, string username,
    string oldpassword, string newpassword);
'@

$dc   = 'dc01.example.com'
$user = 'm.muster'
$alt  = Read-Host -AsSecureString 'Bisheriges Kennwort'
$neu  = Read-Host -AsSecureString 'Neues Kennwort'

$klartext = { param($s) [System.Net.NetworkCredential]::new('', $s).Password }
$rc = $api::NetUserChangePassword(
    $dc, $user, (& $klartext $alt), (& $klartext $neu))

if ($rc -eq 0) { 'Kennwort geändert' } else { "Fehler $rc" }
```

<details class="options-details">
<summary>Explication des options</summary>

| Argument | Effet |
|---|---|
| `domainname` | Nom DNS ou NetBIOS d’un contrôleur de domaine ou du domaine. Depuis un PC hors du domaine, indiquez directement le contrôleur de domaine (nom ou adresse IP). |
| `username` | Nom de connexion du compte (`sAMAccountName`), sans la partie domaine. |
| `oldpassword` | Mot de passe actuel, ici le mot de passe initial. Il doit être exact. |
| `newpassword` | Nouveau mot de passe. Il doit respecter la stratégie de mots de passe du domaine. |

</details>

La valeur de retour 0 signifie que l’opération a réussi. Les codes d’erreur les plus fréquents :

| Code | Signification |
|---|---|
| 5 | Accès refusé, par exemple parce que le contrôleur de domaine n’autorise pas l’appel sans connexion |
| 86 | L’ancien mot de passe est incorrect |
| 2221 | Le nom d’utilisateur est introuvable |
| 2245 | Le nouveau mot de passe ne respecte pas la stratégie de mots de passe (longueur, complexité, historique, âge minimal) |
| 2351 | Le nom du serveur indiqué n’est pas valide |

L’appel passe par SMB et nécessite le port TCP 445 vers le contrôleur de domaine. Dans les réseaux fortement segmentés, ce port est souvent bloqué depuis les réseaux clients ou le VPN ; dans ce cas, seule la possibilité 1 reste disponible.

`Set-ADAccountPassword` du module Active Directory n’aide pas l’utilisateur concerné dans cette situation : le cmdlet nécessite une connexion authentifiée aux Active Directory Web Services, et c’est précisément cette connexion qui n’est pas possible avec le compte concerné. C’est l’outil de l’administrateur.

## Désactiver NLA dans le client : uniquement si NLA n’est pas imposée

Il est souvent recommandé de désactiver CredSSP dans le fichier .rdp :

```text
enablecredsspsupport:i:0
```

Le client se connecte alors sans NLA et obtient l’écran de connexion classique du système cible, sur lequel le changement de mot de passe est possible. Cela ne fonctionne que si le système cible accepte les connexions sans NLA. Si l’option **Autoriser uniquement les connexions depuis des ordinateurs exécutant le Bureau à distance avec authentification au niveau du réseau** y est active, il refuse la connexion. C’est le réglage par défaut dans les versions actuelles de Windows Server.

## Options pour les administrateurs

Côté administration, le problème peut être évité ou résolu :

- **Supprimer l’indicateur :** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` supprime l’obligation de modification. L’utilisateur peut se connecter avec le mot de passe initial et le modifier dans la session avec `Ctrl+Alt+End`. Le mot de passe initial reste valable jusque-là et doit donc être transmis par un canal sécurisé.
- **Réinitialiser le mot de passe :** `Set-ADAccountPassword -Identity m.muster -Reset` définit un nouveau mot de passe. Vérifiez ensuite si l’indicateur est toujours activé.
- **RD Web Access :** si le rôle est présent, il peut proposer le changement de mot de passe pour les mots de passe expirés. Dans le gestionnaire IIS, sous **Sites → Default Web Site → RDWeb → Pages**, définissez dans les **Application Settings** la valeur `PasswordChangeEnabled` sur `true`. La page de changement de mot de passe se trouve sous `/RDWeb/Pages/en-US/password.aspx`.
- **Comptes pour accès RDP uniquement :** pour les utilisateurs qui travaillent exclusivement via RDP (prestataires externes, administrateurs sur des hôtes de rebond), l’obligation de modification lors de la première connexion est impossible à satisfaire sans l’un des moyens mentionnés ci-dessus. La procédure de premier changement de mot de passe devrait donc faire partie de l’intégration.

Désactiver NLA sur le système cible résout également le symptôme, mais supprime une couche de protection : sans NLA, chaque client du réseau atteint l’écran de connexion et donc le service RDP avant de s’être authentifié. Microsoft le déconseille.

## Sources

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): liste des codes de déconnexion du client RDP, dont 4615 (le mot de passe doit être modifié avant la première connexion) et 3847 (mot de passe expiré), avec le texte des messages.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): référence de l’API avec paramètres, autorisations et valeurs de retour.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): retour d’expérience d’Evotec sur le changement de mot de passe via NetUserChangePassword depuis un PC hors du domaine, avec une indication sur TCP 445 et sur l’entrée enablecredsspsupport inefficace lorsque NLA est imposée.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): référence des propriétés .rdp, notamment enablecredsspsupport avec les valeurs et la valeur par défaut.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): section sur le changement de mot de passe pour les mots de passe expirés via RD Web Access et le paramètre PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): référence du cmdlet avec les paramètres Reset, OldPassword, NewPassword, Credential et Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): indications de Microsoft sur les erreurs de connexion avec NLA ; la désactivation de NLA y est indiquée comme non recommandée.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): article de blog sur le champ utilisateur modifiable dans le dialogue « Modifier un mot de passe », qui permet de modifier le mot de passe d’un compte d’un autre domaine.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): guide de Windows OS Hub concernant le paramètre PasswordChangeEnabled et le chemin de la page password.aspx.
