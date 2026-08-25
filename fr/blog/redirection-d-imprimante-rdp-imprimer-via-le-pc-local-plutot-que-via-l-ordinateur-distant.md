---
title: "Redirection d’imprimante RDP : imprimer via le PC local plutôt que via l’ordinateur distant"
navTitle: "Redirection d’imprimante RDP"
description: "Les travaux d’impression issus de la session RDP doivent sortir sur l’imprimante située à côté de l’utilisateur, et non sur l’ordinateur distant. Le réglage se trouve à trois endroits : dans le client RDP, dans le fichier .rdp et sur le système cible. Avec également la gestion de l’avertissement « Éditeur inconnu » et une liste de contrôle pour le dépannage."
date: "2026-08-24"
kategorie: "Client Windows"
timeToRead: "5 min de lecture"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "redirection-d-imprimante-rdp-imprimer-via-le-pc-local-plutot-que-via-l-ordinateur-distant"
translationId: "article-12521248666e9809"
draft: false
translationOf: rdp-druckerumleitung-lokale-drucker
url: https://rafaelpfister.ch/fr/blog/redirection-d-imprimante-rdp-imprimer-via-le-pc-local-plutot-que-via-l-ordinateur-distant
translationSourceHash: a4f12f591e9dcb86f8ebdd3ff8af1008a130c3ec65424abe789ad4d6446eb4c2
translationModel: gpt-5.6-terra
translatedAt: 2026-08-25T04:14:03.740Z
translationReview: automatic
---

# Redirection d’imprimante RDP : imprimer via le PC local plutôt que via l’ordinateur distant

Un utilisateur travaille sur un ordinateur distant via Remote Desktop et souhaite imprimer sur l’imprimante située à côté de lui. C’est précisément le rôle de la redirection d’imprimante : le client RDP déclare les imprimantes locales dans la session, le travail d’impression repasse par le canal RDP vers le client et y est imprimé. Sur le système cible, l’imprimante apparaît avec le suffixe **(redirection, session n)**. En règle générale, aucun pilote n’est nécessaire sur l’ordinateur distant : Windows utilise le pilote universel **Remote Desktop Easy Print** ; seul le pilote correspondant doit être installé sur le client local.

La redirection ne s’applique qu’à l’établissement de la connexion. Après toute modification des paramètres, la session doit être entièrement déconnectée puis reconnectée ; il ne suffit pas de réduire la fenêtre RDP.

## Côté client : activer la redirection

Le moyen le plus rapide passe par l’interface graphique : lancer `mstsc`, afficher les **Options**, onglet **Ressources locales**, cocher **Imprimantes** et enregistrer la connexion dans l’onglet **Général**. Si vous travaillez plutôt avec des fichiers .rdp, ajoutez directement la ligne correspondante dans le fichier ; les fichiers .rdp sont de simples fichiers texte et peuvent être modifiés avec n’importe quel éditeur :

```text
redirectprinters:i:1
```

Un piège avec les raccourcis sans fichier .rdp : lorsque la connexion est lancée avec `mstsc /v:hostname`, les paramètres du fichier caché `Default.rdp` dans le dossier Documents de l’utilisateur s’appliquent. Si la ligne `redirectprinters:i:1` y manque, l’imprimante reste absente, même si tout semble correctement configuré. Cet extrait ajoute la ligne de manière idempotente (une occurrence existante de `0` devient `1`, une ligne manquante est ajoutée) et affiche le résultat à des fins de vérification :

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

Deux autres pièges côté client : premièrement, Windows mémorise pour chaque ordinateur cible, sous `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices`, les redirections que l’utilisateur a autorisées en dernier dans la boîte de dialogue de sécurité ; cette sélection enregistrée remplace la valeur définie dans le fichier .rdp. La suppression de la clé réinitialise l’état. Deuxièmement, la valeur de registre `DisablePrinterRedirection` (DWORD, valeur 1) sous `HKLM\Software\Microsoft\Terminal Server Client` désactive complètement la redirection d’imprimante sur le client ; sur les appareils gérés, il convient de la vérifier avant de commencer le dépannage dans la session.

## Côté serveur : autoriser la redirection

Sur le système cible, la stratégie **Ne pas autoriser la redirection d’imprimantes clientes** (Configuration ordinateur → Modèles d’administration → Composants Windows → Services Bureau à distance → Hôte de session Bureau à distance → Redirection d’imprimantes) est déterminante. Si elle est définie sur *Activé*, aucune imprimante cliente n’est créée, quelle que soit la demande du client. Le principe du réglage le plus restrictif s’applique : si l’un des deux côtés bloque la redirection, elle ne fonctionne pas.

Sans stratégie de groupe, le même mécanisme est contrôlé via le registre : `fDisableCpm` sous `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirection autorisée, 1 = bloquée). En outre, le service **Spouleur d’impression** doit être en cours d’exécution sur le système cible ; sans spouleur, les imprimantes redirigées ne sont pas créées non plus.

Dans la même catégorie de GPO se trouvent deux paramètres utiles : **Utiliser d’abord le pilote d’impression Remote Desktop Easy Print** (valeur par défaut et généralement le bon choix) et **Définir l’imprimante par défaut du client comme imprimante par défaut de la session**.

## L’avertissement « Éditeur inconnu »

Lors de l’ouverture d’un fichier .rdp non signé qui demande des redirections de périphériques, le client affiche un avertissement de sécurité avec des cases à cocher pour chaque ressource. Les cases cochées ou décochées ne s’appliquent qu’à ce démarrage de connexion, mais sont enregistrées dans la clé `LocalDevices` mentionnée plus haut et influencent ainsi silencieusement les connexions futures. Si vous vous demandez pourquoi la case de l’imprimante disparaît régulièrement malgré un fichier .rdp correct, la cause se trouve presque toujours là.

Il existe trois méthodes pour gérer l’avertissement, par ordre croissant d’effort. Premièrement : lancer la connexion avec `mstsc /v:hostname` plutôt que via le fichier .rdp ; sans fichier, il n’y a pas de vérification de l’éditeur et les paramètres proviennent de `Default.rdp`. Deuxièmement : autoriser à l’avance les redirections pour l’ordinateur cible via le registre ; la partie consacrée aux ressources disparaît alors de la boîte de dialogue :

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Troisièmement, la méthode propre pour les fichiers .rdp distribués dans l’entreprise : signer le fichier avec `rdpsign.exe` et un certificat, puis enregistrer l’empreinte du certificat comme éditeur approuvé via GPO. L’effort en vaut rarement la peine pour des postes individuels, mais c’est la solution appropriée pour des fichiers de connexion distribués de manière centralisée.

## Liste de contrôle pour le dépannage

Si l’imprimante n’apparaît pas dans la session, vérifiez dans cet ordre :

1. **Nouvelle connexion ?** La redirection ne s’applique qu’à l’établissement de la connexion, pas dans une session existante.
2. **Bon fichier ?** Avec les raccourcis, vérifiez quel fichier .rdp est réellement ouvert ; avec `mstsc /v:`, c’est `Default.rdp` qui compte.
3. **Sélection enregistrée ?** Vérifiez ou supprimez la clé `LocalDevices` sur le client.
4. **Blocage côté client ?** `DisablePrinterRedirection` sous `HKLM\Software\Microsoft\Terminal Server Client` ne doit pas être défini sur 1.
5. **Blocage côté serveur ?** Vérifiez la GPO « Ne pas autoriser la redirection d’imprimantes clientes » ou `fDisableCpm` sur le système cible.
6. **Spouleur ?** Le service Spouleur d’impression doit être en cours d’exécution sur le système cible.
7. **Vérification dans la session :** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` répertorie les imprimantes redirigées avec leur ID de session.

## Sources

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): référence de toutes les propriétés .rdp, notamment redirectprinters avec ses valeurs et sa valeur par défaut.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configuration GPO et Intune, DisablePrinterRedirection et le test avec Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): référence de commande pour signer des fichiers .rdp à l’aide de l’empreinte d’un certificat.
