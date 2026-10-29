---
title: "Redirection d’imprimante RDP : imprimer via le PC local plutôt que via l’ordinateur distant"
navTitle: "Redirection d’imprimante RDP"
description: "Les travaux d’impression de la session RDP doivent être envoyés vers l’imprimante située à côté de l’utilisateur, et non vers l’ordinateur distant. Le paramètre se trouve à trois endroits : dans le client RDP, dans le fichier .rdp et sur le système cible. Cet article explique aussi comment gérer l’avertissement « Éditeur inconnu » et propose une checklist de dépannage."
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
translationSourceHash: 1ca6c78df61e05f90d1aefc5b20b611e5c6a4bf03908d0466da114c2823e31cd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:32:47.035Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/fr/blog/redirection-d-imprimante-rdp-imprimer-via-le-pc-local-plutot-que-via-l-ordinateur-distant
---

# Redirection d’imprimante RDP : imprimer via le PC local plutôt que via l’ordinateur distant

Un utilisateur travaille sur un ordinateur distant via Remote Desktop et souhaite imprimer sur l’imprimante située à côté de lui. C’est précisément le rôle de la redirection d’imprimante : le client RDP annonce les imprimantes locales dans la session, le travail d’impression repasse par le canal RDP vers le client et y est imprimé. Sur le système cible, l’imprimante apparaît avec le suffixe **(redirigée, session n)**. En règle générale, aucun pilote n’est nécessaire sur l’ordinateur distant : Windows utilise le pilote universel **Remote Desktop Easy Print** ; seul le pilote d’imprimante approprié doit être installé sur le client local.

La redirection ne s’applique qu’à l’établissement de la connexion. Après chaque modification des paramètres, la session doit être entièrement déconnectée puis reconnectée ; il ne suffit pas de réduire la fenêtre RDP.

## Côté client : activer la redirection

Le plus simple est d’activer la redirection d’imprimante via l’interface graphique : lancer `mstsc`, cliquer sur **Afficher les options**, ouvrir l’onglet **Ressources locales**, cocher **Imprimantes** et enregistrer la connexion dans l’onglet **Général**. Si vous utilisez des fichiers .rdp, vous pouvez adapter directement la ligne dans le fichier ; les fichiers .rdp sont de simples fichiers texte et peuvent être modifiés avec n’importe quel éditeur :

```text
redirectprinters:i:1
```

Particularité des raccourcis sans fichier .rdp : si la connexion est lancée avec `mstsc /v:hostname`, les paramètres du fichier caché `Default.rdp` dans le dossier Documents de l’utilisateur s’appliquent. Si la ligne `redirectprinters:i:1` y est absente, l’imprimante ne s’affiche pas, même si tout semble correctement configuré. Cet extrait ajoute la ligne de manière idempotente (une valeur `0` existante devient `1`, une ligne manquante est ajoutée) et affiche le résultat à des fins de contrôle :

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

Deux autres sources d’erreur côté client : premièrement, Windows mémorise pour chaque ordinateur cible sous `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices` les redirections que l’utilisateur a autorisées en dernier dans la boîte de dialogue de sécurité ; cette sélection enregistrée remplace la valeur définie dans le fichier .rdp. La suppression de la clé réinitialise l’état. Deuxièmement, la valeur de registre `DisablePrinterRedirection` (DWORD, valeur 1) sous `HKLM\Software\Microsoft\Terminal Server Client` désactive entièrement la redirection d’imprimante sur le client ; sur les appareils gérés, il vaut la peine de la vérifier avant de commencer le dépannage dans la session.

## Côté serveur : autoriser la redirection

Sur le système cible, la stratégie **Ne pas autoriser la redirection d’imprimantes clientes** (Configuration ordinateur → Modèles d’administration → Composants Windows → Services Bureau à distance → Hôte de session Bureau à distance → Redirection d’imprimantes) est déterminante. Si elle est définie sur *Activé*, aucune imprimante cliente n’est créée, quelle que soit la demande du client. Le principe du paramètre le plus restrictif s’applique : si l’un des deux côtés bloque la redirection, elle ne fonctionne pas.

Sans stratégie de groupe, le même mécanisme est contrôlé via le registre : `fDisableCpm` sous `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = redirection autorisée, 1 = bloquée). En outre, le service **Spouleur d’impression** doit être en cours d’exécution sur le système cible ; sans spouleur, les imprimantes redirigées ne sont pas créées non plus.

Dans la même catégorie GPO se trouvent deux autres paramètres utiles : **Utiliser d’abord le pilote d’imprimante Easy Print des Services Bureau à distance** (valeur par défaut et généralement le bon choix) et **Définir l’imprimante par défaut du client comme imprimante par défaut de la session**.

## L’avertissement « Éditeur inconnu »

Lors de l’ouverture d’un fichier .rdp non signé qui demande des redirections de périphériques, le client affiche un avertissement de sécurité avec des cases à cocher pour chaque ressource. Les cases cochées ou décochées à cet endroit ne s’appliquent qu’à ce démarrage de connexion, mais sont enregistrées dans la clé `LocalDevices` mentionnée plus haut et influencent ainsi silencieusement les connexions futures. Si vous vous demandez pourquoi la case de l’imprimante disparaît sans cesse malgré un fichier .rdp correct, la cause se trouve presque toujours là.

Depuis la mise à jour de sécurité d’avril 2026 (CVE-2026-26151), la Connexion Bureau à distance affiche une nouvelle boîte de dialogue de sécurité à chaque ouverture d’un fichier .rdp, avant l’établissement de la connexion. Toutes les redirections demandées par le fichier, y compris les imprimantes, y sont désactivées par défaut et doivent être activées individuellement. Les fichiers non signés affichent en outre l’avertissement « Caution: Unknown remote connection ». Selon Microsoft, les connexions lancées directement avec un nom d’ordinateur dans la Connexion Bureau à distance ne sont pas concernées. Comme solution de contournement temporaire, Microsoft documente la valeur de registre `RedirectionWarningDialogVersion` (DWORD, valeur 1) sous `HKLM\Software\Policies\Microsoft\Windows NT\Terminal Services\Client`, qui rétablit l’ancienne boîte de dialogue ; sa prise en charge pourrait disparaître avec une future mise à jour.

Il existe trois approches pour gérer cet avertissement, par ordre d’effort croissant. Premièrement : lancer la connexion via `mstsc /v:hostname` plutôt qu’avec le fichier .rdp ; sans fichier, il n’y a pas de vérification de l’éditeur et les paramètres proviennent de `Default.rdp`. La boîte de dialogue de sécurité introduite en avril 2026 n’apparaît pas non plus dans ce cas. Deuxièmement : autoriser à l’avance les redirections pour l’ordinateur cible via le registre ; la partie ressources de l’ancienne boîte de dialogue disparaît alors. Microsoft ne précise pas si cette préautorisation s’applique également à la nouvelle boîte de dialogue de sécurité depuis avril 2026 :

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Troisièmement, la solution propre pour les fichiers .rdp distribués dans l’entreprise : signer le fichier avec `rdpsign.exe` et un certificat, puis enregistrer l’empreinte du certificat comme éditeur approuvé via une GPO. L’effort en vaut rarement la peine pour des postes individuels, mais c’est la bonne solution pour des fichiers de connexion distribués de manière centralisée.

## Checklist de dépannage

Si l’imprimante n’apparaît pas dans la session, vérifier dans cet ordre :

1. **Reconnecté ?** La redirection ne s’applique qu’à l’établissement de la connexion, pas dans une session existante.
2. **Bon fichier ?** Pour les raccourcis, vérifiez quel fichier .rdp est réellement ouvert ; avec `mstsc /v:`, c’est `Default.rdp` qui compte.
3. **Sélection enregistrée ?** Vérifiez ou supprimez la clé `LocalDevices` sur le client. En cas de lancement via un fichier .rdp, vérifiez également si l’imprimante a été activée dans la boîte de dialogue de sécurité ; depuis avril 2026, elle y est désactivée par défaut.
4. **Blocage côté client ?** `DisablePrinterRedirection` sous `HKLM\Software\Microsoft\Terminal Server Client` ne doit pas être défini sur 1.
5. **Blocage côté serveur ?** Vérifiez la GPO « Ne pas autoriser la redirection d’imprimantes clientes » ou `fDisableCpm` sur le système cible.
6. **Spouleur ?** Le service Spouleur d’impression doit être en cours d’exécution sur le système cible.
7. **Contrôle dans la session :** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` répertorie les imprimantes redirigées avec leur ID de session.

## Historique des modifications

- **25 octobre 2026 :** Ajouté : depuis la mise à jour de sécurité d’avril 2026 (CVE-2026-26151), une nouvelle boîte de dialogue affiche toutes les redirections, y compris les imprimantes, désactivées par défaut lors de l’ouverture de fichiers .rdp ; ajout également de la valeur de registre RedirectionWarningDialogVersion et d’une étape de vérification supplémentaire.

## Sources

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): référence de toutes les propriétés .rdp, dont redirectprinters avec ses valeurs et sa valeur par défaut.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, configuration GPO et Intune, DisablePrinterRedirection et le test avec Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): référence de commande pour signer des fichiers .rdp à l’aide de l’empreinte d’un certificat.

4.  [Understanding security warnings when opening Remote Desktop (RDP) files](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/understanding-security-warnings): nouvelle boîte de dialogue de sécurité depuis la mise à jour d’avril 2026, redirections désactivées par défaut et valeur de registre RedirectionWarningDialogVersion.
