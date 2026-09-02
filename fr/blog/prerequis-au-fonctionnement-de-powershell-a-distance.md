---
title: "Prérequis au fonctionnement de PowerShell à distance"
navTitle: "PowerShell à distance"
description: "L’accès à distance avec PowerShell échoue rarement à cause de la commande, mais plutôt des prérequis : service WinRM, écouteur, pare-feu, authentification et particularités des comptes locaux. Ce qui doit être configuré côté cible et côté client, comment le vérifier avec Test-WSMan et pourquoi Access denied n’a généralement rien à voir avec le mot de passe."
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
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/fr/blog/prerequis-au-fonctionnement-de-powershell-a-distance
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:45:55.563Z
translationReview: automatic
---

# Prérequis au fonctionnement de PowerShell à distance

`Invoke-Command` et `Enter-PSSession` se saisissent rapidement, mais la connexion ne s’établit que lorsque les prérequis sont remplis des deux côtés. L’accès à distance avec PowerShell repose sur WS-Management (WinRM), un service d’administration basé sur SOAP via HTTP. Lorsqu’une session échoue, le problème vient presque jamais du cmdlet lui-même, mais d’un service absent, d’un port fermé, d’une règle de pare-feu ou de l’authentification. Cet article passe les prérequis en revue dans l’ordre et montre comment les vérifier un par un.

Commençons par les termes : l’ordinateur cible est celui sur lequel les commandes doivent s’exécuter ; le client est l’ordinateur depuis lequel vous vous connectez. Par défaut, WinRM écoute sur le port 5985 (HTTP) et, s’il est configuré, sur le port 5986 (HTTPS). Le trafic HTTP sur le port 5985 est chiffré au niveau des messages dès que l’authentification utilise Kerberos ou NTLM.

## Vue d’ensemble des cmdlets

Voici les cmdlets utilisés dans cet article :

<details class="options-details">
<summary>Vue d’ensemble des options</summary>

| Cmdlet | Objectif |
|---|---|
| `Enable-PSRemoting` | Configure WinRM côté cible : service, écouteur, règle de pare-feu |
| `Test-WSMan` | Vérifie si le service WinRM de l’ordinateur distant répond |
| `Enter-PSSession` | Ouvre une session distante interactive vers un ordinateur |
| `Invoke-Command` | Exécute un bloc de commandes sur un ou plusieurs ordinateurs |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Ajoute des ordinateurs de confiance pour l’authentification hors domaine |
| `Get-Service WinRM` | Affiche l’état et le type de démarrage du service WinRM |

</details>

## Côté cible : configurer WinRM

Sur l’ordinateur cible, une seule commande configure le service, l’écouteur et la règle de pare-feu. Exécutez-la dans une PowerShell avec des droits d’administrateur :

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Force` | Exécute sans demander de confirmation |
| `-SkipNetworkProfileCheck` | Configure aussi l’accès à distance lorsqu’une connexion réseau est considérée comme publique |

</details>

`Enable-PSRemoting` démarre le service WinRM, règle son type de démarrage sur automatique, crée un écouteur HTTP et ajoute la règle de pare-feu appropriée. Une réserve concerne le profil réseau : si une carte réseau est considérée comme publique, la commande refuse par défaut la configuration. Sur des serveurs ou dans des réseaux contrôlés, `-SkipNetworkProfileCheck` permet tout de même d’effectuer la configuration.

La portée de la règle de pare-feu est importante. Pour les profils réseau publics, la règle standard limite l’accès au sous-réseau local. Si vous vous connectez depuis un autre réseau, par exemple via un VPN, cette limitation s’applique et la connexion échoue malgré le service en cours d’exécution. Ouvrez alors la règle spécifiquement pour la plage d’adresses nécessaire, et non globalement pour toutes les adresses :

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Sélectionne, au moyen du modèle de nom, les règles WinRM HTTP créées par Enable-PSRemoting |
| `-RemoteAddress <Bereich>` | Limite les adresses sources autorisées à la plage indiquée (ici un bloc CIDR) ; `Any` autorise toutes les adresses |

</details>

## Côté client : TrustedHosts et service

Sur le client, le service WinRM doit être en cours d’exécution, sans quoi même la définition des paramètres échoue. Vérifiez-le d’abord :

```powershell
Get-Service WinRM
```

Si le service est à l’état Stopped, démarrez-le avec `Start-Service WinRM` (droits d’administrateur requis). Sur les clients, le type de démarrage est souvent manuel ; le service s’arrête donc à nouveau après un redémarrage. Si vous accédez régulièrement à distance depuis cet ordinateur, définissez le type de démarrage sur automatique.

Le deuxième point concerne l’authentification hors domaine. Si vous vous connectez par adresse IP ou dans un groupe de travail, le client ne peut pas vérifier l’ordinateur distant via Kerberos et revient à NTLM. Pour des raisons de sécurité, WinRM refuse cela tant que l’ordinateur distant n’est pas enregistré comme fiable. Ajoutez l’adresse cible aux TrustedHosts (droits d’administrateur requis) :

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Explication des options</summary>

| Option | Effet |
|---|---|
| `-Value <Liste>` | Ordinateurs de confiance (IP ou nom), plusieurs séparés par des virgules, `*` comme caractère générique |
| `-Force` | Définit la valeur sans demander de confirmation |
| `-Concatenate` | Ajoute à la liste existante au lieu de la remplacer |

</details>

TrustedHosts est un paramètre du client, non de l’ordinateur cible, et concerne la sécurité du client : les ordinateurs qui y sont enregistrés sont considérés comme fiables sans que leur identité soit vérifiée cryptographiquement. Saisissez donc des adresses précises, et non le caractère générique `*`. Dans un domaine avec Kerberos, cette entrée n’est pas nécessaire ; hors domaine, la solution propre sans TrustedHosts consiste à utiliser un écouteur HTTPS avec un certificat auquel le client fait confiance.

## Authentification : pourquoi Access denied est rarement lié au mot de passe

Avec les comptes locaux, un message Access denied malgré un mot de passe correct est une erreur fréquente. La raison est le filtrage UAC à distance : pour les comptes locaux (à l’exception de l’administrateur intégré), Windows retire par défaut les droits administratifs lors d’un accès par le réseau. La connexion réussit, mais toute action nécessitant des privilèges élevés est refusée. Si l’ordinateur distant indique Access denied plutôt que des informations d’identification incorrectes, c’en est probablement la cause.

Vous pouvez y remédier sur l’ordinateur cible avec une valeur de Registre qui accorde aux administrateurs locaux tous les droits sur le réseau :

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Il s’agit d’un assouplissement délibéré : les comptes administrateurs locaux obtiennent alors tous les droits sur le réseau. Ne définissez cette valeur que dans des réseaux contrôlés et avec des mots de passe robustes. Dans un domaine, il vaut mieux utiliser un compte de domaine ; la question ne se pose alors pas.

Lors de l’établissement de la connexion, indiquez le nom d’utilisateur des comptes locaux précédé du nom de l’ordinateur, afin que le système cible résolve le compte localement :

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

Dans la boîte de dialogue d’identification, saisissez l’utilisateur sous la forme `RECHNERNAME\Benutzer` ; pour les comptes de domaine, sous la forme `DOMAENE\Benutzer`. Un code PIN de connexion Windows ne fonctionne pas sur le réseau ; le mot de passe du compte est requis. Pour un compte Microsoft, il s’agit de son mot de passe, et le nom de compte peut différer du nom d’affichage.

## Vérifier dans le bon ordre

Isolez les erreurs de bas en haut : vous verrez ainsi rapidement quel prérequis manque.

Commencez par vérifier l’accessibilité du port :

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Si le port ne répond pas, l’écouteur est absent ou le pare-feu bloque. S’il répond, vérifiez le service WinRM de l’ordinateur distant :

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Une réponse indiquant la version du protocole et le fabricant signifie que le service et l’écouteur sont opérationnels. Testez seulement ensuite avec des informations d’identification :

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Si cet appel renvoie le nom de l’ordinateur distant, tous les prérequis sont remplis.

## Erreurs courantes et leur cause

| Message ou symptôme | Cause probable | Piste |
|---|---|---|
| Port 5985 inaccessible | Pas d’écouteur ou pare-feu bloquant | Vérifier `Enable-PSRemoting`, la règle de pare-feu et sa portée |
| WinRM cannot complete the operation | Service arrêté côté cible, ou accès autorisé uniquement depuis le sous-réseau local | Démarrer le service, ouvrir la règle de pare-feu pour la plage d’adresses nécessaire |
| The WinRM client cannot process the request … TrustedHosts | Connexion hors domaine sans entrée TrustedHosts | Ajouter l’adresse cible aux TrustedHosts sur le client, ou utiliser HTTPS |
| Access is denied (malgré un mot de passe correct) | Filtrage UAC à distance avec un compte local | Définir `LocalAccountTokenFilterPolicy` sur 1, ou utiliser un compte de domaine |
| L’accès à une deuxième ressource échoue dans la session | Double saut : les informations d’identification ne sont pas transmises | Exécuter la tâche directement sur la cible, ou utiliser CredSSP ou une authentification déléguée |

## Limites : le problème du double saut

Une limitation subsiste même avec une configuration complète et ne peut être que contournée : par défaut, une session distante ne peut pas transmettre vos informations d’identification à un troisième système. Si, dans une session sur l’ordinateur cible, vous accédez à un partage réseau ou à un autre serveur, cela échoue faute d’informations d’identification. Ce problème de double saut est une caractéristique de sécurité, non une erreur de configuration. Pour la plupart des tâches de support, il suffit d’exécuter la commande directement sur l’ordinateur cible. Lorsque la transmission est réellement nécessaire, CredSSP ou une délégation restreinte entrent en jeu, chacun avec ses propres compromis de sécurité.

## Sources

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Prérequis pour l’accès à distance avec PowerShell, droits et profils réseau.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): Ce que configure la commande, y compris la réserve liée au profil réseau et la règle de pare-feu.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, authentification hors domaine et messages d’erreur fréquents.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): Cause du problème de double saut et approches de résolution avec leurs compromis.
