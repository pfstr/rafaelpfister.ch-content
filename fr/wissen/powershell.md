---
title: "PowerShell : pipeline d’objets, environnement d’exécution et administration sécurisée"
blatt: "powershell"
description: "PowerShell pour les administrateurs d’infrastructure et de messagerie : éditions et environnement d’exécution .NET, hôte, moteur, runspace et pipeline d’objets, modules et fournisseurs, accès distant via WSMan et SSH, authentification et JEA, sémantique des erreurs et des modifications, parallélisation, journalisation, histoire technique et automatisation prête pour la production."
fakten:
  - label: Catégorie
    wert: Shell · langage de script · framework d’automatisation et de configuration
    href: https://learn.microsoft.com/en-us/powershell/scripting/overview
  - label: Pile technologique
    wert: C# · .NET · System.Management.Automation · programmes natifs et API du système d’exploitation
    href: https://github.com/PowerShell/PowerShell
  - label: Éditions
    wert: Desktop sur .NET Framework · Core sur .NET moderne
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions
  - label: Processus
    wert: powershell.exe pour Windows PowerShell · pwsh pour PowerShell 6 et versions ultérieures
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh
  - label: Modèle de données
    wert: Objets .NET avec type, propriétés et méthodes plutôt que du texte uniquement
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects
  - label: Exécution
    wert: Hôte → moteur → runspace → pipeline → processeur de commandes
    href: https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples
  - label: Extension
    wert: Modules de script, binaires et manifestes via PSModulePath
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules
  - label: Accès distant
    wert: WSMan/WinRM ou SSH ; le code s’exécute dans un runspace distant
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements
  - label: Objets distants
    wert: généralement des instantanés sérialisés ; les méthodes sont perdues lors de la désérialisation
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output
  - label: Execution Policy
    wert: Fonction de sécurité Windows contre l’exécution involontaire, pas une limite de sécurité
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies
  - label: Délégation
    wert: JEA limite l’identité, l’endpoint et les commandes autorisées
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview
  - label: Preuve d’exploitation
    wert: Version du code · résolution des modules · identité · entrée · modification · résultat · journaux
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/security-features
werbung:
  - newsletter
ctaThemen:
  - powershell
  - microsoft-365-exchange
  - automatisierung
translationSourceHash: a8375bc58323f55f2e87dfe08969d32b9ae12af81cbc67673940011c3245a20a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:35:51.635Z
translationReview: automatic
---

# PowerShell : pipeline d’objets, environnement d’exécution et administration sécurisée

PowerShell est à la fois un shell interactif, un langage de script et un framework d’intégration. Cela la distingue autant d’une simple ligne de commande que d’un langage de programmation ordinaire : les commandes transmettent des objets typés dans un pipeline, les applications peuvent intégrer le moteur à leurs propres processus, et les produits d’administration publient leur interface de gestion sous forme de cmdlets, de fournisseurs ou d’API HTTP. Microsoft décrit donc PowerShell comme un framework multiplateforme d’automatisation et de configuration pour les données structurées, les interfaces REST et les modèles d’objets ([Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)).

Pour l’exploitation, la syntaxe seule n’est donc pas déterminante. L’édition et l’environnement d’exécution .NET concrets, le contexte du processus et du runspace, les modules chargés, l’identité sur le système cible, le transport et l’autorisation, la sémantique des erreurs ainsi que la preuve des modifications réellement effectuées par un script sont décisifs. Une commande qui fonctionne dans une session interactive ne constitue pas encore une automatisation fiable.

L’explication suit une commande à travers l’analyseur, le moteur, le runspace et le pipeline d’objets jusqu’au système concerné. Les modules, fournisseurs, accès distant, parallélisme, sécurité et exploitation sont ensuite rapportés à ce modèle d’exécution.

PowerShell est simultanément un shell, un langage d’automatisation et un environnement d’exécution d’administration. L’élément essentiel est que les cmdlets transmettent des objets .NET par un pipeline et que les modules de produits génèrent à partir de ceux-ci des appels d’API locaux ou distants. Cet article suit ce chemin, de la commande jusqu’à son effet sur le système cible.

## Architecture de l’environnement d’exécution

Une exécution PowerShell se compose de plusieurs couches :

| Couche | Rôle | État pertinent pour l’administration |
|---|---|---|
| Hôte | fournit la console, l’éditeur, le service ou l’application intégrée | nom d’hôte, interactif/non supervisé, entrée/sortie, profil |
| Moteur | analyse le langage, lie les paramètres et coordonne les commandes | édition, version, .NET, Language Mode, paramètres |
| Runspace | contexte d’exécution isolé avec état de session | identité, variables, fonctions, modules, lecteurs, endpoint |
| Pipeline | relie le processeur de commandes et transporte objets/flux | ordre, liaison, énumération, flux d’erreur et d’information |
| Commande | cmdlet, fonction, script, alias ou programme natif | source, version du module, paramètres, effets de bord |
| Adaptateur/fournisseur/API | traduit vers .NET, système d’exploitation, magasin ou service distant | endpoint, authentification, autorisations, limites de débit, domaine de panne |

L’**hôte** n’est pas le moteur. Windows Terminal, la console classique, Visual Studio Code, une tâche planifiée ou une application intégrée en C# peuvent exploiter le même moteur PowerShell de manière différente. Un **runspace** est l’environnement d’exploitation dans lequel s’exécutent les commandes d’un pipeline ; plusieurs runspaces dans un processus peuvent fournir des états de session et du parallélisme différents ([PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary), [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-powershell.svg?v=20260813" title="Interaktive Infografik: PowerShell-Laufzeit von Host, Engine und Runspace über Objektpipeline, Module, Provider und APIs bis zu Remoting, Vertrauensgrenzen und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-powershell.svg?v=20260813">Ouvrir directement le graphique interactif</a>.
</iframe>

Le nom du processus est un indicateur de diagnostic important : `powershell.exe` lance Windows PowerShell, `pwsh` la branche PowerShell basée sur .NET moderne. Les deux peuvent être installés côte à côte sous Windows et voir ainsi des modules, profils, valeurs TLS par défaut et assemblies différents. L’interface en ligne de commande de `pwsh` documente également la liaison des fichiers, commandes, encodages et appels non interactifs ([Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)).

### Inventorier l’environnement d’exécution et le contexte

[$PSVersionTable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) indique l’édition, le moteur et la plateforme ; [`Get-Host`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host) décrit l’hôte, pas nécessairement la version sous-jacente du système d’exploitation. Un rapport d’erreur reproductible doit au minimum inclure le chemin du processus, l’édition, la version, la description .NET, l’architecture, le Language Mode et le chemin des modules.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Laufzeitinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process -Id $PID | Select-Object Path,Id,StartTime
$PSVersionTable
Get-Host
[System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
$ExecutionContext.SessionState.LanguageMode
$env:PSModulePath -split [IO.Path]::PathSeparator</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Process -Id $PID | Select-Object Path,Id,StartTime
  $PSVersionTable
  Get-Host
  [System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
  $ExecutionContext.SessionState.LanguageMode
  $env:PSModulePath -split [IO.Path]::PathSeparator
'</code></pre>
  </div>
</div>

## Éditions, .NET et compatibilité

Depuis PowerShell 5.1, Microsoft distingue les éditions **Desktop** et **Core**. Desktop s’exécute sur le .NET Framework spécifique à Windows ; Core s’exécute sur .NET moderne et est disponible sous Windows, Linux et macOS. Le champ `CompatiblePSEditions` du manifeste d’un module tiers n’est toutefois qu’une métadonnée : PowerShell n’impose pas automatiquement la compatibilité qui y est déclarée ([Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)).

Windows PowerShell 5.1 et PowerShell 7+ sont donc deux produits distincts, et non deux options de démarrage d’une même installation. PowerShell 7 ne peut pas remplacer Windows PowerShell sur place ; les profils, chemins de modules et certaines parties de la surface d’API diffèrent. Le guide de migration de Microsoft recommande de tester explicitement les dépendances et la compatibilité des modules ([Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)).

La décision de compatibilité est prise par charge de travail :

- **Moteur :** le script nécessite-t-il Windows PowerShell/Desktop ou PowerShell/Core ?
- **Système d’exploitation :** appelle-t-il des cmdlets réservées à Windows, COM, le Registre, WMI ou des programmes natifs ?
- **Module :** est-il portable comme module de script, construit comme module binaire pour le bon environnement .NET cible, ou utilisable uniquement via un processus de compatibilité ?
- **Authentification :** l’environnement d’exécution prend-il en charge le flux requis, le magasin de certificats et le cache de jetons ?
- **Sérialisation :** l’objet requis reste-t-il vivant localement, ou provient-il d’un autre processus en tant qu’instantané désérialisé ?
- **Support :** l’éditeur mentionne-t-il précisément cette combinaison de moteur, module et produit ?

PowerShell 7 peut charger certains modules Windows PowerShell via une session implicite d’accès distant/de compatibilité. Cela produit toutefois des objets désérialisés et ajoute une frontière de processus ; cela ne remplace pas une preuve de compatibilité native ([Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)).

Une fois l’édition et l’environnement d’exécution définis, le pipeline d’objets explique la principale différence avec les shells orientés texte : le filtrage, le tri et l’exportation travaillent sur des propriétés plutôt que sur des positions de colonnes.

## Le pipeline d’objets

Un pipeline Unix classique relie des flux d’octets ou de texte. Au sein d’un processus, PowerShell transmet fondamentalement des objets .NET. Un objet possède un type, des propriétés et des méthodes ; le composant de pipeline suivant lie les entrées selon les règles de paramètres et peut accéder aux propriétés sans analyser à nouveau une sortie tabulaire ([Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects), [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)).

Le tableau visible n’est pas l’objet. Le système de mise en forme sélectionne des vues et ne génère les instructions de mise en forme qu’à la fin du pipeline. `Format-Table` et `Format-List` doivent donc être placés à l’extrémité de sortie ; s’ils sont utilisés au milieu d’un pipeline de données, les commandes suivantes ne reçoivent plus les objets d’administration d’origine ([Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Objektpipeline">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$service = Get-Service -Name WinRM
$service.GetType().FullName
$service | Get-Member
$service | Select-Object Name,Status,StartType
$service | ConvertTo-Json -Depth 3</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  $process = Get-Process -Id $PID
  $process.GetType().FullName
  $process | Get-Member
  $process | Select-Object Id,ProcessName,Path
  $process | ConvertTo-Json -Depth 3
'</code></pre>
  </div>
</div>

### Liaison, énumération et flux

La liaison dans le pipeline s’effectue pour chaque paramètre par valeur ou par nom de propriété. Les collections sont généralement énumérées élément par élément ; cela modifie le comportement et les besoins en mémoire par rapport à une liste unique. Pour des fonctions fiables, les paramètres de pipeline sont déclarés explicitement, l’entrée est traitée dans le bloc `process` et la sortie n’est pas involontairement polluée par des valeurs auxiliaires.

PowerShell gère plusieurs flux logiques : Success, Error, Warning, Verbose, Debug, Information et Progress. Seul le flux Success constitue la sortie d’objets normale ; des erreurs peuvent être visibles sans interrompre le pipeline. La redirection et la fusion des flux font partie de l’interface d’appel et doivent être définies consciemment pour les tâches planifiées, l’intégration continue et la supervision ([Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)).

Les programmes natifs restent un point de transition. La citation de leurs arguments, l’encodage des caractères, la sortie standard et les codes de sortie ne suivent pas entièrement le modèle des cmdlets. Un wrapper doit traiter `$LASTEXITCODE`, les codes de sortie attendus, l’encodage et stderr, puis normaliser le résultat dans son propre objet. La seule présence de texte sur stderr ne signifie pas systématiquement un échec selon le programme.

## Résolution des commandes, modules et fournisseurs

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command) indique quel alias, quelle fonction, quelle cmdlet, quel script ou programme natif est effectivement appelé pour un nom. Cela est plus important que l’orthographe visible : un alias ou une fonction de même nom dans le profil peut masquer une cmdlet. L’appel entièrement qualifié `Modulname\\Verb-Noun` rend l’origine explicite.

Les modules regroupent commandes, données de types et de mise en forme ainsi que dépendances. Un module de script utilise généralement `.psm1`, un module binaire une assembly .NET ; un manifeste `.psd1` décrit notamment le module racine, la version, les commandes exportées, les prérequis et les éditions compatibles. Un manifeste n’est pas obligatoire pour le chargement, mais il est requis pour la publication dans la PowerShell Gallery ([Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules), [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)).

Le moteur recherche les modules via `PSModulePath`. Plusieurs versions installées sont possibles ; le chargement automatique et les appels imprécis à `Import-Module` peuvent ainsi sélectionner une autre implémentation sur un serveur que dans l’environnement de test. Un script de production inventorie donc le nom, la version, le chemin et le dépôt du module, et fixe les dépendances lors du déploiement au lieu de les réinstaller sans contrôle pendant chaque exécution ([Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Modulinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Command Get-Mailbox -All |
  Select-Object Name,CommandType,Source,Version,Definition
Get-Module -ListAvailable |
  Sort-Object Name,Version |
  Select-Object Name,Version,Path,CompatiblePSEditions
Get-InstalledModule -ErrorAction SilentlyContinue |
  Select-Object Name,Version,Repository</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -Command '
  Get-Command Invoke-RestMethod -All |
    Select-Object Name,CommandType,Source,Version,Definition
  Get-Module -ListAvailable |
    Sort-Object Name,Version |
    Select-Object Name,Version,Path,CompatiblePSEditions
  Get-InstalledModule -ErrorAction SilentlyContinue |
    Select-Object Name,Version,Repository
'</code></pre>
  </div>
</div>

Les **fournisseurs** proposent des magasins de données hiérarchiques avec une interface comparable à un système de fichiers. `FileSystem:`, `Registry:`, `Certificate:`, `Variable:`, `Function:` ou `WSMan:` ressemblent à des lecteurs, mais disposent de capacités et d’effets de bord différents. Avant d’écrire du code générique, `Get-PSProvider` indique les opérations prises en charge par un fournisseur. La spécification historique du langage cite les fournisseurs et lecteurs comme modèle fondamental pour les données autrement difficiles d’accès ([Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)).

Une commande locale est exécutée dans le processus actuel ; l’accès distant déplace l’exécution dans un runspace sur la cible. L’identité, le chemin réseau, les modules et l’accès aux ressources locales changent alors.

## Accès distant : client local, transport, runspace distant

L’accès distant PowerShell exécute fondamentalement le code du côté cible. Localement, la commande et les arguments sont préparés ; le transport, le listener et la configuration de session authentifient l’appel et créent un runspace distant. Son système d’exploitation, moteur, modules, stratégies, identité et accès réseau déterminent le résultat.

Sous Windows, l’accès distant classique utilise WS-Management via WinRM. PowerShell 7 prend aussi en charge l’accès distant SSH sous Windows, Linux et macOS. Les deux transports véhiculent des données d’accès distant PowerShell, mais n’offrent pas les mêmes fonctions d’endpoint : Microsoft indique par exemple que l’accès distant SSH ne prend pas en charge la configuration des endpoints PowerShell et donc pas JEA ([Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements), [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)).

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Remoting">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-WSMan -ComputerName mail01.example.ch
$session = New-PSSession -ComputerName mail01.example.ch -ConfigurationName Microsoft.PowerShell
Invoke-Command -Session $session -ScriptBlock {
  [pscustomobject]@{
    Host = $env:COMPUTERNAME
    Edition = $PSVersionTable.PSEdition
    Version = $PSVersionTable.PSVersion.ToString()
  }
}
Remove-PSSession $session</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ssh -G admin@mail01.example.ch | grep -E '^(hostname|user|port|identityfile) '
pwsh -NoProfile -Command '
  $s = New-PSSession -HostName mail01.example.ch -UserName admin
  Invoke-Command -Session $s -ScriptBlock {
    [pscustomobject]@{
      Host = [Environment]::MachineName
      Edition = $PSVersionTable.PSEdition
      Version = $PSVersionTable.PSVersion.ToString()
    }
  }
  Remove-PSSession $s
'</code></pre>
  </div>
</div>

Les outils utilisés sont documentés dans [`Test-WSMan`](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman), [`New-PSSession`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession), [`Invoke-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) et la page de manuel OpenSSH consacrée à [`ssh`](https://man.openbsd.org/ssh).

### Sérialisation et perte d’objet

La plupart des objets distants sont sérialisés sur la cible puis reconstruits localement. Le résultat est un instantané avec des propriétés, mais généralement sans les méthodes de l’objet actif d’origine ; le nom de type porte fréquemment le préfixe `Deserialized.`. Les données de mise en forme et Type-Depth déterminent les propriétés transférées. Les modifications de l’instantané local ne changent pas l’objet cible ([Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)).

Les filtres, agrégations et données sensibles sont donc traités autant que possible **sur la cible**, et seul le résultat nécessaire est transféré. Cela réduit la latence, le volume de sérialisation et la fuite de données :

```text
schlecht:  200'000 Remoteobjekte → serialisieren → übertragen → lokal filtern
gut:       remote filtern/projizieren/aggregieren → 200 Ergebnisobjekte übertragen
```

Le « second hop » connu n’est pas une particularité de PowerShell, mais une limite de délégation : l’identité qui s’authentifie depuis le client d’administration auprès du serveur A ne possède pas automatiquement des informations d’identification réutilisables pour le serveur B. Des solutions telles que la délégation contrainte Kerberos, CredSSP ou un compte de service modifient le modèle de confiance et ne sont utilisées qu’avec une analyse explicite des menaces ([Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)).

## Authentification, autorisation et moindre privilège

L’accès distant sépare quatre questions :

1. **Confiance du transport :** à quel hôte le client parle-t-il et comment sa clé ou son certificat est-il vérifié ?
2. **Authentification :** quelle personne, Managed Identity, application, certificat ou clé SSH se connecte ?
3. **Autorisation :** quelle configuration de session, quel rôle et quelle autorisation produit permettent quelles opérations ?
4. **Identité du processus :** sous quel compte local ou virtuel la commande distante s’exécute-t-elle réellement ?

**Just Enough Administration (JEA)** crée des endpoints PowerShell restreints. Les Role Capabilities définissent les cmdlets, fonctions, fournisseurs et programmes externes visibles ; les Session Configurations associent utilisateurs ou groupes aux rôles. Des comptes virtuels ou gMSA peuvent exécuter des actions privilégiées sans donner à l’utilisateur appelant un compte d’administration permanent. Les transcriptions et journaux soutiennent la traçabilité ([Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)).

JEA ne limite pas automatiquement toutes les voies de contournement. Les commandes visibles, paramètres, fournisseurs, accès aux fichiers, types .NET appelables et programmes externes sont examinés conjointement. Un éditeur, compilateur ou bloc de script générique apparemment inoffensif peut étendre l’ensemble de commandes prévu.

## Execution Policy, signatures et contrôle des applications

La PowerShell Execution Policy est sous Windows une fonction de sécurité contre l’exécution involontaire de scripts, **pas une limite de sécurité**. Un utilisateur autorisé à exécuter des commandes peut appeler autrement le contenu d’un script bloqué. Sur les systèmes autres que Windows, la stratégie n’est pas appliquée comme sous Windows ([Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)).

`AllSigned` et `RemoteSigned` utilisent Authenticode pour vérifier l’origine et l’intégrité de certains types de fichiers PowerShell. Une signature valide indique toutefois uniquement que le fichier est resté inchangé depuis la signature et qu’il correspond à une chaîne de certificats fiable ; elle ne prouve pas que le code est sûr ou techniquement correct ([Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)).

Une chaîne de contrôle fiable combine :

- contrôle de source, revue et artefacts de build immuables ;
- signature et vérification de hachage ;
- dépôts de modules contrôlés et versions de dépendances ;
- App Control/System Lockdown ou un contrôle des applications comparable ;
- identité à moindre privilège et endpoints d’accès distant restreints ;
- secrets hors du code source ;
- Script Block/Module Logging, transcriptions et envoi centralisé des journaux.

Microsoft classe Execution Policy comme défense en profondeur, tandis qu’App Control dans System Lockdown peut définir le Language Mode par runspace. Script Block Logging écrit les commandes, fonctions et scripts traités dans le journal PowerShell Operational ; une journalisation plus intensive peut elle-même capturer des secrets ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

### Vérifier la signature et l’artefact

[`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) vérifie sous Windows le statut Authenticode ; [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) et [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparent un artefact à une valeur attendue publiée indépendamment.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$script = 'C:\\Ops\\Deploy-MailPolicy.ps1'
Get-AuthenticodeSignature $script |
  Select-Object Status,StatusMessage,SignerCertificate,TimeStamperCertificate
Get-FileHash $script -Algorithm SHA256
Get-ExecutionPolicy -List</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum /opt/ops/Deploy-MailPolicy.ps1
pwsh -NoProfile -Command '
  Get-FileHash /opt/ops/Deploy-MailPolicy.ps1 -Algorithm SHA256
  Get-ExecutionPolicy -List
'</code></pre>
  </div>
</div>

## Secrets et identités non interactives

Un `SecureString` n’est pas un coffre-fort. Microsoft recommande, pour les nouveaux développements, de remplacer les mots de passe dans la mesure du possible par des certificats, l’authentification Windows ou d’autres méthodes sans mot de passe. Le module SecretManagement fournit une interface de cmdlets uniforme pour les extension vaults ; le coffre enregistré implémente le stockage et l’authentification. L’enregistrement est lié à l’utilisateur, ce qui doit être pris en compte pour les comptes de service et l’automatisation sans interface utilisateur ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features), [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)).

Pour chaque automatisation, les éléments suivants sont documentés séparément :

- **Identité de charge de travail :** compte de service, gMSA, Managed Identity, App Registration ou certificat ;
- **Source des informations d’identification :** coffre, magasin de certificats, fédération de charge de travail ou broker interactif ;
- **Autorisation :** rôle local, rôle RBAC, périmètre d’API ou Application Permission ;
- **Cycle de vie :** émission, rotation, expiration, révocation et Break Glass ;
- **Lieu d’exécution :** agent, tâche planifiée, runner, fonction ou client d’administration ;
- **Chemin réseau :** DNS, proxy, confiance TLS, endpoint et limite de débit ;
- **Ancrage d’audit :** ID d’exécution, ID d’identité, commit de code et ID de modification/ticket.

Les variables d’environnement et de pipeline sont des mécanismes de transport, pas des stockages sûrs. Elles peuvent atteindre des processus enfants, des dumps, des sorties de diagnostic ou des journaux CI. Même l’accès à un coffre est limité au périmètre de secret minimal et à une courte durée de vie.

Une automatisation sécurisée doit reconnaître si une opération a réellement échoué. Un message rouge, une exception, une erreur non terminale et un code de sortie natif sont des signaux distincts.

## Sémantique des erreurs : une ligne rouge visible n’est pas un modèle de sortie

PowerShell distingue les erreurs non terminales, terminales au niveau de l’instruction et terminales au niveau du script. De nombreuses cmdlets écrivent un `ErrorRecord` non terminal pour un objet unique et poursuivent le traitement du reste du pipeline. Un `try/catch` ne capture que les erreurs qui quittent le bloc contrôlé ; `-ErrorAction Stop` peut promouvoir des erreurs non terminales appropriées pour cet appel ([Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling), [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)).

Le code de production définit donc :

- quelles erreurs par objet sont tolérées et consignées ;
- quelles erreurs interrompent l’exécution complète ;
- quel code de sortie le processus transmet au planificateur ou à l’intégration continue ;
- si une nouvelle tentative est sûre et idempotente ;
- quel ID de corrélation relie erreur, objet cible et requête API ;
- quelles ressources sont fermées dans le bloc `finally` ;
- comment les modifications partielles sont détectées et compensées.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Fehlerbehandlung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$ErrorActionPreference = 'Stop'
try {
  $result = Invoke-RestMethod -Uri $endpoint -Method Get -TimeoutSec 30
  $result | ConvertTo-Json -Depth 10
}
catch {
  Write-Error -ErrorRecord $_
  exit 1
}
finally {
  Remove-Variable result -ErrorAction SilentlyContinue
}</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">pwsh -NoProfile -NonInteractive -Command '
  $ErrorActionPreference = "Stop"
  try {
    $result = Invoke-RestMethod -Uri $env:OPS_ENDPOINT -Method Get -TimeoutSec 30
    $result | ConvertTo-Json -Depth 10
  } catch {
    Write-Error -ErrorRecord $_
    exit 1
  } finally {
    Remove-Variable result -ErrorAction SilentlyContinue
  }
'</code></pre>
  </div>
</div>

L’exemple utilise [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), dont le comportement HTTP et les erreurs doivent être intégrés délibérément au traitement des erreurs. Pour les programmes natifs, `$LASTEXITCODE` est également vérifié. La variable de processus `$?` et les flux d’erreurs PowerShell ne remplacent pas un contrat documenté sur les codes de sortie du programme appelé.

## Sémantique sûre des modifications et idempotence

Une automatisation d’administration sépare **l’identification, la planification, la modification et la vérification**. Elle compare l’état réel et l’état souhaité au moyen d’identifiants stables, génère un ensemble de modifications explicite et n’exécute que la différence nécessaire. Elle relit ensuite l’état par l’interface faisant autorité. « La cmdlet n’a renvoyé aucune erreur » ne constitue pas une preuve d’état.

Les fonctions avancées peuvent proposer les paramètres communs `-WhatIf` et `-Confirm` avec `SupportsShouldProcess`. La fonction appelle `$PSCmdlet.ShouldProcess()` immédiatement avant l’effet de bord. Microsoft documente toutefois des limites dans la transmission de `WhatIfPreference` entre modules de script ; un `-WhatIf` réussi est donc un aperçu utile, pas une garantie transactionnelle ([Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess), [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)).

Un processus de modification robuste conserve :

1. l’ID immuable de la cible et l’état antérieur ;
2. la différence attendue et l’approbation métier ;
3. l’identité utilisée, la version de l’endpoint et du module ;
4. l’ID de réponse/requête du système cible ;
5. l’état relu et le test fonctionnel dépendant ;
6. la règle de retour arrière ou de poursuite à partir du point de non-retour.

L’idempotence est une propriété de l’opération et de son modèle d’état, non du langage. `Set-*` peut être idempotent, `Add-*` ne l’est généralement pas ; une opération HTTP PUT peut être répétable, tandis qu’un POST crée un nouvel objet à chaque nouvelle tentative. Avant toute nouvelle tentative automatique, la sémantique de l’API, l’ID de requête et l’état côté serveur sont clarifiés.

## Parallélisme, jobs et runspaces

PowerShell peut paralléliser le travail avec des Background Jobs, Thread Jobs, Remoting Jobs, `ForEach-Object -Parallel` ou ses propres pools de runspaces. Ces modèles diffèrent en matière d’isolation des processus, de sérialisation, de coûts de démarrage et d’état partagé. Les jobs distants et de processus sérialisent les objets ; les Thread Jobs et runspaces partagent un processus et peuvent donc être plus rapides, mais aussi moins isolés ([Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs), [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)).

Le parallélisme optimal est limité par le système cible :

- limite de débit d’API et Retry-After ;
- nombre maximal de shells d’accès distant et d’opérations par utilisateur ;
- pool de connexions LDAP/base de données ;
- limitation Exchange et coût des cmdlets ;
- mémoire par runspace et taille des objets sérialisés ;
- exigences d’ordre et de verrouillage sur la cible ;
- budget d’erreurs et capacité de reprise ciblée.

Davantage de threads n’augmentent pas automatiquement le débit. Un script d’administration mesure la longueur de la file d’attente, la latence, le taux de réussite et la limitation, limite la concurrence et utilise du jitter/backoff. Il enregistre un point de contrôle par objet cible au lieu de répéter aveuglément l’ensemble du lot après une erreur partielle.

## Modules de produits et API HTTP

PowerShell n’est souvent que le côté client d’un plan de contrôle distant. La cmdlet peut valider localement les paramètres puis appeler REST, WSMan, RPC, CIM ou une API propriétaire. L’espace d’erreurs englobe alors le moteur local, le module, l’obtention de jeton, DNS, proxy, [TLS](/kb/tls), version de l’API, autorisation, limitation et le service produit.

Le **Microsoft Graph PowerShell SDK** génère des cmdlets à partir des métadonnées d’API Graph. Les autorisations restent des autorisations Graph ; le nom de la cmdlet ne supprime pas la sémantique Delegated versus Application, le consentement et les restrictions de ressources ([Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)). Le module **ExchangeOnlineManagement** fournit des cmdlets spécifiques à Exchange et utilise des connexions basées sur REST pour de nombreuses opérations ; malgré leurs chevauchements, les interfaces Exchange et Graph ne sont pas des API interchangeables ([Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)).

Avant une exécution en production, les éléments suivants sont enregistrés :

| Niveau | À consigner |
|---|---|
| Client | moteur/édition, processus, système d’exploitation, fuseau horaire |
| Code | dépôt, commit, artefact signé, configuration |
| Dépendance | nom du module, version, chemin, dépôt, hachage |
| Identité | ID d’objet/application, type d’authentification, tenant/realm |
| Autorisation | rôles, scopes, consentement, endpoint/configuration de session |
| Requête | ID cible, opération, ID de corrélation/requête, heure |
| Résultat | statut, classe d’erreur, nouvelle tentative, état modifié |
| Vérification | relecture indépendante ou vérification des journaux |

La recherche d’erreurs commence donc par le contexte d’exécution : édition, hôte, utilisateur, langue, version du module et chemin d’accès distant. Le produit cible n’est examiné qu’ensuite.

## Diagnostic : d’abord le contexte d’exécution, ensuite le produit cible

Un incident PowerShell est étudié selon les dépendances réelles :

1. **Appel :** quel processus, hôte, utilisateur, répertoire de travail, mode profil et NonInteractive ?
2. **Résolution :** quelle commande et quelle version de module sont réellement utilisées ?
3. **Stratégie locale :** Language Mode, App Control, Execution Policy, magasin de confiance et proxy ?
4. **Transport :** DNS, TCP, TLS, WSMan/SSH ou HTTPS sont-ils accessibles ?
5. **Authentification :** quel flux, jeton/clé/certificat, heure et tenant ?
6. **Autorisation :** quel rôle local, quelle capacité JEA, quels scopes API et rôles produit ?
7. **État cible :** l’objet existe-t-il sous l’ID stable utilisé et dans le bon scope ?
8. **Réponse serveur :** code d’erreur, ID de requête, limitation, journal d’audit et journal produit ?

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) vérifie sous Windows l’accessibilité TCP, [`nc`](https://man.openbsd.org/nc) sur les systèmes Unix. Cela ne prouve ni une authentification TLS réussie ni une authentification applicative, mais distingue une erreur de connexion des couches supérieures. [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) lit les journaux d’événements sous Windows ; [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) lit le journal systemd.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für PowerShell-Transport- und Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mail01.example.ch -Port 5986 -InformationLevel Detailed
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' -MaxEvents 100 |
  Select-Object TimeCreated,Id,LevelDisplayName,Message
Get-Command Invoke-Command -All |
  Select-Object CommandType,Source,Version,Definition</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz mail01.example.ch 22
journalctl --since '-30 min' --no-pager -u sshd
pwsh -NoProfile -Command '
  Get-Command Invoke-Command -All |
    Select-Object CommandType,Source,Version,Definition
'</code></pre>
  </div>
</div>

## Journalisation, transcriptions et protection des données

PowerShell peut utiliser Script Block Logging, Module Logging, Protected Event Logging et Transcription. Sous Windows, les événements Script Block apparaissent dans le canal `Microsoft-Windows-PowerShell/Operational`. [`Start-Transcript`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript) enregistre une représentation textuelle d’une session ; il ne s’agit pas d’un journal d’audit structuré complet et il ne capture pas toutes les activités hors de l’hôte ([Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)).

La journalisation accroît la visibilité, mais aussi le risque d’enregistrer des jetons, mots de passe, contenus d’e-mails ou données personnelles. Microsoft recommande Protected Event Logging pour les journaux Script Block sensibles. La rétention, l’accès, le transfert centralisé, la synchronisation de l’heure et le stockage protégé contre les manipulations sont également définis ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

Pour les automatisations internes, les événements structurés sont plus appropriés que les textes de console mis en forme librement. Un événement contient au minimum l’heure en UTC, l’ID d’exécution, la version du code, l’ID cible, l’action, le résultat, la durée, l’ID de requête et une cause d’erreur classifiée. Les secrets et charges utiles inutiles sont supprimés avant la journalisation.

## Contrôle qualité et déploiement

[`Invoke-ScriptAnalyzer`](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) vérifie le code PowerShell avec des règles statiques. Cela détecte de nombreux problèmes de style, de compatibilité et de sécurité, mais ne remplace ni les tests ni une intégration produit. Un pipeline fiable comprend :

- vérification de l’analyseur et de l’analyse statique ;
- tests unitaires pour les fonctions pures et le mappage de données ;
- tests de contrat contre les schémas d’API et les réponses simulées ;
- test d’intégration avec les autorisations minimales ;
- sortie `-WhatIf`/plan, lorsque cela est possible sur le plan métier ;
- test des ensembles vides, doublons, pagination, limitation et erreurs partielles ;
- signature, hachage et artefact immuable ;
- déploiement progressif avec métriques et critères d’arrêt ;
- vérification indépendante par relecture sur le système cible.

Les profils sont généralement exclus de l’automatisation de production avec `-NoProfile`. Les processus non interactifs utilisent `-NonInteractive`, des encodages définis, des chemins explicites et un fuseau horaire fixe. Le déploiement fournit les modules ou utilise un dépôt interne contrôlé ; il n’installe pas sans vérification la dernière dépendance à chaque exécution.

## Histoire technique

Jeffrey Snover a formulé dans le **Monad Manifesto** un modèle d’administration unifié et orienté objet : petits outils, dénomination cohérente, analyseur et pipeline communs ainsi qu’interfaces d’administration via .NET. Microsoft désigne ce document comme l’origine du développement qui est devenu PowerShell ([PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)).

En avril 2006, Microsoft a renommé en **Windows PowerShell** le projet développé sous le nom de code Monad ; la première ligne de produits était étroitement liée à Windows et au .NET Framework ([Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)). Cette branche a vu naître les conventions verbe-nom des cmdlets, le pipeline d’objets, les fournisseurs, l’accès distant, les modules, les workflows et Desired State Configuration.

En 2016, Microsoft a publié PowerShell sous licence MIT et a développé une branche multiplateforme basée sur .NET Core. PowerShell Core 6 a apporté la prise en charge de Linux/macOS et l’accès distant SSH ; PowerShell 7 a poursuivi le nom du produit sans « Core ». La base de code et le développement de PowerShell 7+ se trouvent dans le dépôt public ; les modifications ne sont pas rétroportées vers Windows PowerShell 5.1 ([PowerShell GitHub repository](https://github.com/PowerShell/PowerShell), [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)).

Cette histoire explique de nombreuses limites d’exploitation actuelles : deux éditions sous Windows, des modules dépendant du .NET Framework, différents transports d’accès distant et un mélange d’interfaces locales de gestion Windows et d’API HTTP multiplateformes. Pour les administrateurs, PowerShell est donc moins « un numéro de version » qu’une combinaison concrète de moteur, environnement d’exécution, module, transport et produit cible.

## Ce qu’un administrateur doit retenir en un coup d’œil

Un script n’est exploitable que lorsque, en plus du code, l’environnement d’exécution, les modules, l’identité, le système cible et le traitement des erreurs sont connus. Le tableau résume les preuves minimales requises.

| Question | Preuve minimale |
|---|---|
| Quelle PowerShell s’exécute ? | chemin du processus, PSEdition, PSVersion, .NET, OS/architecture |
| Qu’est-ce qui est exécuté ? | CommandType, Source, Version, Definition résolus |
| Dans quel contexte ? | hôte, runspace, mode de profil, répertoire de travail, Language Mode |
| Avec quelles dépendances ? | chemin du module, manifeste, version, dépôt, hachage |
| Contre quelle cible ? | endpoint, tenant/realm, ID d’objet stable, version d’API/session |
| Avec quelle identité ? | utilisateur/application/Managed Identity, flux d’authentification, ID de certificat/clé |
| Avec quels droits ? | rôle local, endpoint JEA, scopes, RBAC produit |
| Que doit-on modifier ? | différence réel/souhaité, plan, portée, approbation |
| Qu’est-ce qui a réellement changé ? | réponse serveur, ID de requête, relecture et test fonctionnel |
| Comment une erreur partielle est-elle traitée ? | ErrorAction, code de sortie, nouvelle tentative, point de contrôle, compensation |
| Comment l’audit est-il assuré ? | ID d’exécution, journaux structurés, transcript/journal opérationnel, rétention |
| Comment répéter ou annuler ? | clé d’idempotence, point de non-retour, poursuite/retour arrière |

## Sources

- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)
- [Microsoft – PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary)
- [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)
- [Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)
- [Microsoft – about_Automatic_Variables](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables)
- [Microsoft – Get-Host](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host)
- [Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)
- [Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)
- [Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)
- [Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects)
- [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)
- [Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)
- [Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)
- [Microsoft – Get-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command)
- [Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)
- [Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)
- [Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)
- [Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements)
- [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)
- [Microsoft – Test-WSMan](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman)
- [Microsoft – New-PSSession](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession)
- [Microsoft – Invoke-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command)
- [OpenBSD – ssh](https://man.openbsd.org/ssh)
- [Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)
- [Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)
- [Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)
- [Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)
- [Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)
- [Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)
- [Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling)
- [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)
- [Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess)
- [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)
- [Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs)
- [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)
- [Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
- [Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft – Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft – Start-Transcript](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript)
- [Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)
- [Microsoft – PSScriptAnalyzer overview](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview)
- [PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)
- [Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)
- [PowerShell – GitHub repository](https://github.com/PowerShell/PowerShell)
- [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)
