---
title: "Conditions requises pour le fonctionnement de PowerShell à distance"
navTitle: "PowerShell à distance"
description: "L’accès à distance avec PowerShell échoue rarement à cause de la commande, mais plutôt des prérequis : service WinRM, écouteur, pare-feu, authentification et particularités des comptes locaux. Ce qui doit être configuré côté cible et côté client, comment le vérifier avec Test-WSMan, et pourquoi Access denied n’a généralement rien à voir avec le mot de passe."
date: "2026-09-01"
kategorie: "Windows et PowerShell"
timeToRead: "10 min de lecture"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "prerequis-au-fonctionnement-de-powershell-a-distance"
translationId: "article-7315c1ae9e67a24d"
translationOf: remote-powershell-voraussetzungen
translationSourceHash: edebb6e6be909c38c0d70f1bf09aaac37328f86bf2def4e23a26a34456b3492f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:56:56.533Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/prerequis-au-fonctionnement-de-powershell-a-distance
---

# Conditions requises pour le fonctionnement de PowerShell à distance

`Invoke-Command` et `Enter-PSSession` se tapent rapidement, mais la connexion ne s’établit que lorsque les prérequis sont remplis des deux côtés. L’accès à distance avec PowerShell repose sur WS-Management (WinRM), un service d’administration basé sur SOAP via HTTP. Lorsqu’une session échoue, le problème vient presque jamais du cmdlet lui-même, mais d’un service absent, d’un port fermé, d’une règle de pare-feu ou de l’authentification. Cet article passe en revue les prérequis dans l’ordre et montre comment les vérifier un par un.

Commençons par les termes : l’ordinateur cible est celui sur lequel les commandes doivent s’exécuter ; le client est l’ordinateur depuis lequel vous vous connectez. Par défaut, WinRM écoute sur le port 5985 (HTTP) et, s’il est configuré, sur le port 5986 (HTTPS). Le trafic HTTP sur le port 5985 est chiffré au niveau des messages dès lors que l’authentification utilise Kerberos ou NTLM.

## Vue d’ensemble des cmdlets

Voici les cmdlets utilisés dans cet article :

<details class="options-details">
<summary>Vue d’ensemble des options</summary>

| Cmdlet | Objectif |
|---|---|
| `Enable-PSRemoting` | Configure WinRM côté cible : service, écouteur et règle de pare-feu |
| `Test-WSMan` | Vérifie si le service WinRM de l’hôte distant répond |
| `Enter-PSSession` | Ouvre une session distante interactive vers un ordinateur |
| `Invoke-Command` | Exécute un bloc de commandes sur un ou plusieurs ordinateurs |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Ajoute des hôtes approuvés pour l’authentification hors domaine |
| `Get-Service WinRM` | Affiche l’état et le type de démarrage du service WinRM |

</details>

## Côté cible : configurer WinRM

Sur l’ordinateur cible, une seule commande configure le service, l’écouteur et la règle de pare-feu. Exécutez-la dans une console PowerShell avec des droits d’administrateur :

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Force` | Exécute l’action sans demander de confirmation |
| `-SkipNetworkProfileCheck` | Configure également l’accès à distance lorsqu’une connexion réseau est classée comme publique |

</details>

`Enable-PSRemoting` démarre le service WinRM, règle son type de démarrage sur automatique, crée un écouteur HTTP et ajoute la règle de pare-feu appropriée. Une réserve concerne le profil réseau : si une carte réseau est classée comme publique, la commande refuse par défaut la configuration. Sur des serveurs ou dans des réseaux contrôlés, `-SkipNetworkProfileCheck` permet néanmoins de terminer la configuration.

L’étendue de la règle de pare-feu est importante. Pour les profils réseau publics, la règle par défaut limite l’accès au sous-réseau local. Si vous vous connectez depuis un autre réseau, par exemple via un VPN, cette restriction s’applique et la connexion échoue malgré le service en cours d’exécution. Ouvrez alors la règle de manière ciblée pour la plage d’adresses nécessaire, et non globalement pour toutes les adresses :

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Sélectionne, à l’aide du modèle de nom, les règles WinRM HTTP créées par Enable-PSRemoting |
| `-RemoteAddress <Bereich>` | Limite les adresses sources autorisées à la plage indiquée (ici un bloc CIDR) ; `Any` autorise toutes les adresses |

</details>

## Côté client : TrustedHosts et service

Sur le client, le service WinRM doit être en cours d’exécution, sans quoi même la définition des paramètres échoue. Vérifiez-le d’abord :

```powershell
Get-Service WinRM
```

Si le service est à l’état Stopped, démarrez-le avec `Start-Service WinRM` (droits d’administrateur requis). Sur les clients, le type de démarrage est souvent manuel ; le service est donc de nouveau arrêté après un redémarrage. Si vous accédez régulièrement à des systèmes depuis cet ordinateur, définissez le type de démarrage sur automatique.

Le second point concerne l’authentification hors domaine. Si vous vous connectez avec une adresse IP ou dans un groupe de travail, le client ne peut pas vérifier l’hôte distant via Kerberos et bascule vers NTLM. Pour des raisons de sécurité, WinRM le refuse tant que l’hôte distant n’est pas enregistré comme approuvé. Ajoutez l’adresse cible à TrustedHosts (droits d’administrateur requis) :

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Value <Liste>` | Hôtes approuvés (IP ou nom), plusieurs séparés par des virgules, `*` comme caractère générique |
| `-Force` | Définit la valeur sans demander de confirmation |
| `-Concatenate` | Ajoute à la liste existante au lieu de la remplacer |

</details>

TrustedHosts est un paramètre du client, et non de l’ordinateur cible ; il concerne la sécurité du client : les hôtes enregistrés sont considérés comme fiables sans que leur identité soit vérifiée cryptographiquement. Indiquez donc des adresses précises plutôt que le caractère générique `*`. Dans un domaine avec Kerberos, cette entrée n’est pas nécessaire ; hors domaine, une solution propre sans TrustedHosts consiste à utiliser un écouteur HTTPS avec un certificat auquel le client fait confiance.

## Authentification : pourquoi Access denied est rarement lié au mot de passe

Avec les comptes locaux, un message Access denied apparaît fréquemment alors que le mot de passe est correct. La raison est le filtrage UAC à distance : pour les comptes locaux (à l’exception de l’administrateur intégré), Windows retire par défaut les privilèges administratifs lors des accès réseau. La connexion réussit, mais toute action nécessitant des droits élevés est refusée. Si l’hôte distant renvoie Access denied plutôt que des informations d’identification incorrectes, c’est probablement la cause.

Vous pouvez y remédier sur l’ordinateur cible avec une valeur de registre qui accorde aux administrateurs locaux tous les droits via le réseau :

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Il s’agit d’un assouplissement délibéré : les comptes d’administrateur locaux obtiennent ainsi tous les droits via le réseau. Définissez cette valeur uniquement dans des réseaux contrôlés et avec des mots de passe robustes. Dans un domaine, utilisez plutôt un compte de domaine ; la question ne se pose alors pas.

Lors de la connexion, indiquez le nom d’utilisateur des comptes locaux précédé du nom de l’ordinateur afin que le système cible résolve le compte localement :

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

Dans la boîte de dialogue d’authentification, saisissez l’utilisateur sous la forme `RECHNERNAME\Benutzer` ; pour les comptes de domaine, sous la forme `DOMAENE\Benutzer`. Un code PIN utilisé pour la connexion à Windows ne fonctionne pas via le réseau ; le mot de passe du compte est requis. Pour un compte Microsoft, il s’agit de son mot de passe, et le nom du compte peut différer du nom d’affichage.

## Vérifier dans le bon ordre

Isolez les erreurs de bas en haut : vous verrez ainsi rapidement quel prérequis manque.

Commencez par vérifier l’accessibilité du port :

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Si le port ne répond pas, l’écouteur est absent ou le pare-feu bloque l’accès. S’il répond, vérifiez le service WinRM de l’hôte distant :

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Une réponse contenant la version du protocole et le fabricant signifie que le service et l’écouteur sont opérationnels. Testez ensuite seulement avec des informations d’identification :

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Si cet appel renvoie le nom de l’ordinateur distant, tous les prérequis sont remplis.

## Erreurs fréquentes et leurs causes

| Message ou symptôme | Cause probable | Piste |
|---|---|---|
| Port 5985 inaccessible | Aucun écouteur ou pare-feu bloquant | Vérifier `Enable-PSRemoting`, la règle de pare-feu et son étendue |
| WinRM cannot complete the operation | Service désactivé côté cible, ou accès autorisé uniquement depuis le sous-réseau local | Démarrer le service, ouvrir la règle de pare-feu pour la plage d’adresses nécessaire |
| The WinRM client cannot process the request … TrustedHosts | Connexion hors domaine sans entrée TrustedHosts | Ajouter l’adresse cible à TrustedHosts sur le client ou utiliser HTTPS |
| Access is denied (malgré un mot de passe correct) | Filtrage UAC à distance pour un compte local | Définir `LocalAccountTokenFilterPolicy` sur 1, ou utiliser un compte de domaine |
| L’accès à une seconde ressource échoue dans la session | Double saut : les informations d’identification ne sont pas transmises | Exécuter la tâche directement sur la cible, ou utiliser CredSSP ou une authentification déléguée |

## Limites : le problème du double saut

Une limitation subsiste même avec une configuration complète et ne peut être que contournée : par défaut, une session distante ne peut pas transmettre vos informations d’identification à un troisième système. Si, depuis une session sur l’ordinateur cible, vous accédez à un partage réseau ou à un autre serveur, l’opération échoue faute d’informations d’identification. Ce problème de double saut est une caractéristique de sécurité, et non une erreur de configuration. Pour la plupart des tâches de support, il suffit d’exécuter la commande directement sur l’ordinateur cible. Lorsque la transmission est réellement nécessaire, CredSSP ou une délégation restreinte entrent en jeu, chacun impliquant ses propres compromis de sécurité.

## Historique des modifications

- **25 octobre 2026 :** lien des sources mis à jour vers la nouvelle adresse de Microsoft Learn.

## Sources

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): prérequis pour l’accès à distance avec PowerShell, droits et profils réseau.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): ce que configure la commande, y compris la réserve liée au profil réseau et la règle de pare-feu.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, authentification hors domaine et messages d’erreur fréquents.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop): cause du problème de double saut et solutions avec leurs compromis.
