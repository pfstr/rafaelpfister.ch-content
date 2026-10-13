---
title: "PowerShell: objektpipeline, kjøretid og sikker administrasjon"
blatt: "powershell"
description: "PowerShell for infrastruktur- og meldingsadministratorer: utgaver og .NET-kjøretid, vert, motor, runspace og objektpipeline, moduler og providere, remoting via WSMan og SSH, autentisering og JEA, feil- og endringssemantikk, parallellisering, logging, teknisk historie og produksjonsklar automatisering."
fakten:
  - label: Kategori
    wert: Shell · skriptspråk · rammeverk for automatisering og konfigurasjon
    href: https://learn.microsoft.com/en-us/powershell/scripting/overview
  - label: Teknologistakk
    wert: C# · .NET · System.Management.Automation · native programmer og operativsystem-API-er
    href: https://github.com/PowerShell/PowerShell
  - label: Utgaver
    wert: Desktop på .NET Framework · Core på moderne .NET
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions
  - label: Prosess
    wert: powershell.exe for Windows PowerShell · pwsh for PowerShell 6 og nyere
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh
  - label: Datamodell
    wert: .NET-objekter med type, egenskaper og metoder i stedet for utelukkende tekst
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects
  - label: Kjøring
    wert: Vert → motor → runspace → pipeline → kommandoprosessor
    href: https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples
  - label: Utvidelse
    wert: Skript-, binær- og manifestmoduler via PSModulePath
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules
  - label: Remoting
    wert: WSMan/WinRM eller SSH; kode kjører i en ekstern runspace
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements
  - label: Eksterne objekter
    wert: vanligvis serialiserte øyeblikksbilder; metoder går tapt ved deserialisering
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output
  - label: Execution Policy
    wert: Windows-sikkerhetsfunksjon mot utilsiktet kjøring, ingen sikkerhetsgrense
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies
  - label: Delegering
    wert: JEA begrenser identitet, endepunkt og tillatte kommandoer
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview
  - label: Driftsdokumentasjon
    wert: Kodeversjon · moduloppløsning · identitet · inndata · endring · resultat · logger
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/security-features
werbung:
  - newsletter
ctaThemen:
  - powershell
  - microsoft-365-exchange
  - automatisierung
translationSourceHash: a8375bc58323f55f2e87dfe08969d32b9ae12af81cbc67673940011c3245a20a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:43:06.128Z
translationReview: automatic
---

# PowerShell: objektpipeline, kjøretid og sikker administrasjon

PowerShell er samtidig et interaktivt shell, et skriptspråk og et rammeverk for innebygging. Det skiller det både fra en ren kommandolinje og fra et vanlig programmeringsspråk: Kommandoer leverer typede objekter inn i en pipeline, applikasjoner kan bygge inn motoren i egne prosesser, og adminprodukter publiserer administrasjonsgrensesnittet sitt som cmdlets, providere eller HTTP-API-er. Microsoft beskriver derfor PowerShell som et plattformuavhengig rammeverk for automatisering og konfigurasjon av strukturerte data, REST-grensesnitt og objektmodeller ([Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)).

For drift er derfor ikke bare syntaksen relevant. Avgjørende er den konkrete utgaven og .NET-kjøretiden, prosess- og runspace-konteksten, innlastede moduler, identiteten på målsystemet, transport og autorisasjon, feilsemantikk samt dokumentasjon av hva et skript faktisk har endret. En kommando som fungerer i en interaktiv økt, er ennå ikke pålitelig automatisering.

Forklaringen følger en kommando gjennom parser, motor, runspace og objektpipeline frem til systemet som kontaktes. Moduler, providere, remoting, parallellitet, sikkerhet og drift knyttes deretter til denne kjøretidsmodellen.

PowerShell er shell, automatiseringsspråk og administrasjonskjøretid samtidig. Det avgjørende er at cmdlets sender .NET-objekter gjennom en pipeline, og at produktmoduler genererer lokale eller eksterne API-kall fra dem. Artikkelen følger denne veien fra kommandoen til virkningen i målsystemet.

## Arkitektur for kjøretiden

En PowerShell-kjøring består av flere lag:

| Lag | Oppgave | Tilstand som er relevant for administratorer |
|---|---|---|
| Vert | stiller til rådighet konsoll, editor, tjeneste eller innebygd applikasjon | vertsnavn, interaktiv/ubevoktet, inndata/utdata, profil |
| Motor | parser språket, binder parametere og koordinerer kommandoer | utgave, versjon, .NET, Language Mode, innstillinger |
| Runspace | isolert kjørekontekst med session state | identitet, variabler, funksjoner, moduler, drives, endepunkt |
| Pipeline | kobler kommandoprosessorer og transporterer objekter/strømmer | rekkefølge, binding, enumerering, feil- og informasjonsstrømmer |
| Kommando | cmdlet, funksjon, skript, alias eller native program | kilde, modulversjon, parametere, sideeffekter |
| Adapter/provider/API | oversetter til .NET, operativsystem, lager eller ekstern tjeneste | endepunkt, autentisering, rettigheter, rate limits, feilområde |

**Verten** er ikke motoren. Windows Terminal, den klassiske konsollen, Visual Studio Code, en Scheduled Task eller en applikasjon innebygd i C# kan kjøre den samme PowerShell-motoren på ulike måter. En **runspace** er driftsmiljøet der kommandoene i en pipeline kjører; flere runspaces i én prosess kan tilby ulike session states og parallellitet ([PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary), [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-powershell.svg?v=20260813" title="Interaktive Infografik: PowerShell-Laufzeit von Host, Engine und Runspace über Objektpipeline, Module, Provider und APIs bis zu Remoting, Vertrauensgrenzen und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-powershell.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

Prosessnavnet er et viktig diagnostisk kjennetegn: `powershell.exe` starter Windows PowerShell, `pwsh` den PowerShell-linjen som er basert på moderne .NET. Begge kan være installert side om side på Windows og kan derfor se ulike moduler, profiler, TLS-standardinnstillinger og assemblies. Kommandolinjegrensesnittet til `pwsh` dokumenterer også hvordan filer, kommandoer, enkodinger og ikke-interaktive kall bindes ([Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)).

### Inventariser kjøretid og kontekst

[$PSVersionTable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) viser utgave, motor og plattform; [`Get-Host`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host) beskriver verten, ikke automatisk den underliggende operativsystemversjonen. En reproduserbar feilrapport må minst inneholde prosessbane, utgave, versjon, .NET-beskrivelse, arkitektur, Language Mode og modulbane.

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

## Utgaver, .NET og kompatibilitet

Siden PowerShell 5.1 skiller Microsoft mellom utgavene **Desktop** og **Core**. Desktop kjører på det Windows-spesifikke .NET Framework; Core kjører på moderne .NET og er tilgjengelig på Windows, Linux og macOS. Feltet `CompatiblePSEditions` i manifestet til en fremmed modul er imidlertid bare metadata: PowerShell håndhever ikke automatisk kompatibiliteten som angis der ([Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)).

Windows PowerShell 5.1 og PowerShell 7+ er derfor to separate produkter, ikke to oppstartsalternativer for samme installasjon. PowerShell 7 kan ikke erstatte Windows PowerShell in-place; profiler, modulbaner og deler av API-overflaten er forskjellige. Microsofts migreringsveiledning anbefaler eksplisitt testing av avhengigheter og modulkompatibilitet ([Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)).

Kompatibilitetsbeslutningen tas per arbeidslast:

- **Motor:** Krever skriptet Windows PowerShell/Desktop eller PowerShell/Core?
- **Operativsystem:** Kaller det Windows-only-cmdlets, COM, Registry, WMI eller native programmer?
- **Modul:** Er det portabelt som skriptmodul, bygget som binærmodul for riktig .NET-målkjøretid eller bare brukbart via en kompatibilitetsprosess?
- **Autentisering:** Støtter kjøretiden den nødvendige flyten, sertifikatlageret og tokencachen?
- **Serialisering:** Forblir det nødvendige objektet levende lokalt, eller kommer det fra en annen prosess som et deserialisert øyeblikksbilde?
- **Støtte:** Angir produsenten nøyaktig denne kombinasjonen av motor, modul og produkt?

PowerShell 7 kan laste utvalgte Windows PowerShell-moduler gjennom en implisitt remoting-/kompatibilitetsøkt. Dette gir imidlertid deserialiserte objekter og en ytterligere prosessgrense; det er ingen erstatning for en innfødt kompatibilitetsbekreftelse ([Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)).

Når utgave og kjøretid er fastslått, forklarer objektpipelinen den viktigste forskjellen fra tekstorienterte shell: Filtrering, sortering og eksport arbeider med egenskaper i stedet for kolonneposisjoner.

## Objektpipelinen

En klassisk Unix-pipeline kobler byte- eller tekststrømmer. PowerShell sender som hovedregel .NET-objekter videre innenfor én prosess. Et objekt har type, egenskaper og metoder; den neste pipelinekomponenten binder inndata etter parameterregler og kan få tilgang til egenskaper uten å parse tabellutdata på nytt ([Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects), [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)).

Den synlige tabellen er ikke objektet. Formateringssystemet velger visninger og genererer formateringsinstruksjoner først på slutten av pipelinen. `Format-Table` og `Format-List` hører derfor hjemme ved utdataenden; brukes de midt i en datapipeline, mottar ikke etterfølgende kommandoer de opprinnelige administrasjonsobjektene lenger ([Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)).

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

### Binding, enumerering og strømmer

Pipeline-binding skjer per parameter etter verdi eller egenskapsnavn. Samlinger enumereres vanligvis elementvis; dette endrer oppførsel og minnebehov sammenlignet med én enkelt liste. For pålitelige funksjoner deklareres pipelineparametere eksplisitt, inndata behandles i `process`-blokken, og utdata forurenses ikke utilsiktet av hjelpeverdier.

PowerShell har flere logiske strømmer: Success, Error, Warning, Verbose, Debug, Information og Progress. Bare Success-strømmen er vanlig objektutdata; feil kan være synlige uten å stanse pipelinen. Omdirigering og sammenslåing av strømmene er en del av kallsgrensesnittet og må fastsettes bevisst for Scheduled Tasks, CI og overvåking ([Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)).

Native programmer forblir et overgangspunkt. Argumentkvotering, tegnkoding, standardutdata og exitkoder følger ikke fullt ut cmdlet-modellen. En wrapper bør håndtere `$LASTEXITCODE`, forventede exitkoder, enkoding og stderr, og normalisere resultatet som et eget objekt. At det finnes tekst på stderr, betyr ikke alltid at et program har feilet.

## Kommandooppløsning, moduler og providere

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command) viser hvilket alias, hvilken funksjon, cmdlet, hvilket skript eller native program som faktisk kalles for et navn. Dette er viktigere enn den synlige skrivemåten: Et alias eller en funksjon med samme navn i profilen kan skjule en cmdlet. Det fullstendig kvalifiserte kallet `Modulname\\Verb-Noun` gjør opprinnelsen eksplisitt.

Moduler pakker kommandoer, type- og formatdata samt avhengigheter. En skriptmodul bruker vanligvis `.psm1`, en binærmodul en .NET-assembly; et manifest `.psd1` beskriver blant annet rotmodul, versjon, eksporterte kommandoer, krav og kompatible utgaver. Et manifest er ikke obligatorisk for lasting, men kreves for publisering i PowerShell Gallery ([Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules), [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)).

Motoren søker etter moduler via `PSModulePath`. Flere installerte versjoner er mulig; automatisk lasting og upresise `Import-Module`-kall kan derfor velge en annen implementering på en server enn i testmiljøet. Et produksjonsskript inventariserer derfor modulnavn, versjon, bane og repository og fester avhengigheter i deployment i stedet for å etterinstallere dem ukontrollert ved hver kjøring ([Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)).

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

**Providere** stiller hierarkiske datalagre med et filsystemlignende grensesnitt til rådighet. `FileSystem:`, `Registry:`, `Certificate:`, `Variable:`, `Function:` eller `WSMan:` ser ut som drives, men har ulike egenskaper og sideeffekter. Før generisk kode brukes `Get-PSProvider` til å vise hvilke operasjoner en provider støtter. Den historiske språkspesifikasjonen nevner providere og drives som en grunnleggende modell for ellers vanskelig tilgjengelige data ([Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)).

En lokal kommando kjører i den aktuelle prosessen; remoting flytter kjøringen til en runspace på målet. Dermed endres identitet, nettverksbane, moduler og tilgang til lokale ressurser.

## Remoting: lokal klient, transport, ekstern runspace

PowerShell Remoting utfører i utgangspunktet kode på målsiden. Lokalt forberedes kommandoen og argumentene; transport, listener og Session Configuration autentiserer kallet og oppretter en ekstern runspace. Dets operativsystem, motor, moduler, retningslinjer, identitet og nettverkstilgang bestemmer resultatet.

På Windows bruker klassisk remoting WS-Management via WinRM. PowerShell 7 støtter i tillegg SSH-remoting på Windows, Linux og macOS. Begge transportene bærer PowerShell-remotingdata, men tilbyr ikke de samme endepunktfunksjonene: Microsoft påpeker for eksempel at SSH-remoting ikke har PowerShell-endepunktkonfigurasjon og dermed ikke støtter JEA ([Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements), [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)).

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

Verktøyene som brukes, er dokumentert under [`Test-WSMan`](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman), [`New-PSSession`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession), [`Invoke-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) og OpenSSH-manualsiden for [`ssh`](https://man.openbsd.org/ssh).

### Serialisering og objekttap

De fleste eksterne objekter serialiseres på målet og rekonstrueres lokalt. Resultatet er et øyeblikksbilde med egenskaper, men vanligvis uten metodene til det opprinnelige liveobjektet; typenavnet har ofte prefikset `Deserialized.`. Formateringsdata og Type-Depth avgjør hvilke egenskaper som overføres. Endringer i det lokale øyeblikksbildet endrer ikke målobjektet ([Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)).

Filtrering, aggregering og sensitive data behandles derfor så langt som mulig **på målet**, og bare det nødvendige resultatet overføres. Dette reduserer ventetid, serialiseringsvolum og dataavrenning:

```text
schlecht:  200'000 Remoteobjekte → serialisieren → übertragen → lokal filtern
gut:       remote filtern/projizieren/aggregieren → 200 Ergebnisobjekte übertragen
```

Den kjente «Second Hop» er ikke en særegenhet ved PowerShell, men en delegeringsgrense: Identiteten som autentiserer seg fra adminklienten mot server A, har ikke automatisk gjenbrukbare påloggingsopplysninger for server B. Løsninger som Kerberos Constrained Delegation, CredSSP eller en tjenestekonto endrer tillitsmodellen og brukes bare med eksplisitt trusselanalyse ([Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)).

## Autentisering, autorisasjon og minste privilegium

Remoting skiller mellom fire spørsmål:

1. **Transporttillit:** Hvilken vert snakker klienten med, og hvordan kontrolleres nøkkelen eller sertifikatet?
2. **Autentisering:** Hvilken person, Managed Identity, app, sertifikat eller SSH-nøkkel logger på?
3. **Autorisasjon:** Hvilken Session Configuration, rolle og produktrettighet tillater hvilke operasjoner?
4. **Prosessidentitet:** Hvilken lokal eller virtuell konto kjører den eksterne kommandoen faktisk under?

**Just Enough Administration (JEA)** oppretter begrensede PowerShell-endepunkter. Role Capabilities definerer synlige cmdlets, funksjoner, providere og eksterne programmer; Session Configurations tilordner brukere eller grupper til roller. Virtuelle kontoer eller gMSA-er kan utføre privilegerte handlinger uten å gi brukeren som kaller, en varig administratorkonto. Transkripsjoner og logger støtter sporbarhet ([Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)).

JEA begrenser ikke automatisk enhver omgåelsesvei. Synlige kommandoer, parametere, providere, filtilgang, kallbare .NET-typer og eksterne programmer må kontrolleres samlet. En tilsynelatende harmløs editor, kompilator eller generisk skriptblokk kan utvide det tiltenkte kommandoomfanget.

## Execution Policy, signaturer og applikasjonskontroll

PowerShell Execution Policy er på Windows en sikkerhetsfunksjon mot utilsiktet skriptkjøring, **ikke en sikkerhetsgrense**. En bruker som har tillatelse til å kjøre kommandoer, kan kalle innholdet i et blokkert skript på andre måter. På systemer som ikke er Windows, håndheves ikke policyen på samme måte som i Windows ([Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)).

`AllSigned` og `RemoteSigned` bruker Authenticode for å kontrollere opprinnelse og integritet for bestemte PowerShell-filtyper. En gyldig signatur sier imidlertid bare at filen ikke er endret siden signeringen og passer til en klarert sertifikatkjede; den beviser ikke at koden er sikker eller faglig riktig ([Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)).

En robust kontrollkjede kombinerer:

- Source Control, gjennomgang og uforanderlige byggeartefakter;
- signatur- og hashkontroll;
- kontrollerte modul-repositories og avhengighetsversjoner;
- App Control/System Lockdown eller tilsvarende applikasjonskontroll;
- identitet med minste privilegium og begrensede remoting-endepunkter;
- hemmeligheter utenfor kildekoden;
- Script Block/Module Logging, transkripsjoner og sentral loggforsendelse.

Microsoft klassifiserer Execution Policy som Defense-in-Depth, mens App Control i System Lockdown kan fastsette Language Mode per runspace. Script Block Logging skriver behandlede kommandoer, funksjoner og skript til PowerShell Operational Log; mer omfattende logging kan i seg selv registrere hemmeligheter ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

### Kontroller signatur og artefakt

[`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) kontrollerer Authenticode-statusen under Windows; [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) sammenligner en artefakt med en forventet verdi som er publisert uavhengig.

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

## Hemmeligheter og ikke-interaktive identiteter

En `SecureString` er ikke et hvelv. Microsoft anbefaler at passord om mulig erstattes av sertifikater, Windows-autentisering eller andre passordløse metoder i ny utvikling. SecretManagement-modulen tilbyr et enhetlig cmdlet-grensesnitt for Extension Vaults; det registrerte hvelvet implementerer lagring og autentisering. Registreringen er brukeravhengig, noe som må tas hensyn til for tjenestekontoer og headless-automatisering ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features), [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)).

For hver automatisering dokumenteres separat:

- **Arbeidslastidentitet:** Tjenestekonto, gMSA, Managed Identity, App Registration eller sertifikat;
- **Kilde for legitimasjon:** Hvelv, sertifikatlager, workload-federation eller interaktiv broker;
- **Rettighet:** Lokal rolle, RBAC-rolle, API-scope eller Application Permission;
- **Livssyklus:** Utstedelse, rotasjon, utløp, tilbakekalling og Break Glass;
- **Kjøringssted:** Agent, Scheduled Task, runner, Function eller adminklient;
- **Nettverksbane:** DNS, proxy, TLS Trust, endepunkt og rate limit;
- **Revisjonsanker:** Run-ID, identitets-ID, kodecommit og endrings-/ticket-ID.

Miljøvariabler og pipelinevariabler er transportmekanismer, ikke sikre lagre. De kan havne i underprosesser, dumps, diagnoseutdata eller CI-logger. Også tilgang til et hvelv begrenses til minst mulig secret-scope og kort levetid.

Sikker automatisering må oppdage om en operasjon faktisk har feilet. En rød melding, et unntak, en ikke-terminerende feil og en native exitkode er ulike signaler.

## Feilsemantikk: Synlig rød linje er ikke en exitmodell

PowerShell skiller mellom ikke-terminerende, statement-terminerende og skript-terminerende feil. Mange cmdlets skriver en ikke-terminerende `ErrorRecord` for ett enkelt objekt og fortsetter å behandle resten av pipelinen. En `try/catch` fanger bare feil som forlater den kontrollerte blokken; `-ErrorAction Stop` kan eskalere egnede ikke-terminerende feil for dette kallet ([Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling), [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)).

Produksjonskode definerer derfor:

- hvilke feil per objekt som tolereres og logges;
- hvilke feil som avbryter hele kjøringen;
- hvilken exitkode prosessen leverer til scheduler eller CI;
- om retry er sikkert og idempotent;
- hvilken korrelasjons-ID som knytter feil, målobjekt og API-forespørsel sammen;
- hvilke ressurser som lukkes i `finally`-blokken;
- hvordan delvise endringer oppdages og kompenseres.

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

Eksempelet bruker [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), der HTTP- og feilatferd bevisst må inngå i feilhåndteringen. For native programmer kontrolleres også `$LASTEXITCODE`. Prosessvariabelen `$?` og PowerShell-feilstrømmene erstatter ikke en dokumentert kontrakt om exitkodene til det kalte programmet.

## Sikker endringssemantikk og idempotens

En automatisering for administratorer skiller mellom **innhenting, planlegging, endring og verifisering**. Den sammenligner faktisk og ønsket tilstand ved hjelp av stabile ID-er, oppretter et eksplisitt endringssett og utfører bare den nødvendige differansen. Deretter leser den tilstanden på nytt via det relevante grensesnittet. «Cmdleten returnerte ingen feil» er ikke tilstandsbevis.

Advanced Functions kan med `SupportsShouldProcess` tilby Common Parameters `-WhatIf` og `-Confirm`. Funksjonen kaller `$PSCmdlet.ShouldProcess()` rett før sideeffekten. Microsoft dokumenterer imidlertid begrensninger ved videresending av `WhatIfPreference` mellom skriptmoduler; en vellykket `-WhatIf` er derfor en nyttig forhåndsvisning, ikke en transaksjonell garanti ([Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess), [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)).

En robust endringsprosess beholder:

1. uforanderlig mål-ID og tidligere tilstand;
2. forventet differanse og faglig godkjenning;
3. brukt identitet, endepunkt- og modulversjon;
4. svar-/forespørsels-ID fra målsystemet;
5. tilstand lest på nytt og avhengig funksjonstest;
6. rollback- eller roll-forward-regel fra Point of No Return.

Idempotens er en egenskap ved operasjonen og tilstandsmodellen dens, ikke ved språket. `Set-*` kan være idempotent, `Add-*` vanligvis ikke; en HTTP-PUT-operasjon kan gjentas, mens en POST oppretter et nytt objekt ved hvert retry. Før automatisk retry avklares API-semantikk, request-ID og serverstatus.

## Parallellitet, jobber og runspaces

PowerShell kan parallellisere arbeid gjennom Background Jobs, Thread Jobs, Remoting Jobs, `ForEach-Object -Parallel` eller egne runspace-pooler. Disse modellene er ulike når det gjelder prosessisolasjon, serialisering, oppstartskostnad og delt tilstand. Eksterne jobber og prosessjobber serialiserer objekter; Thread Jobs og runspaces deler en prosess og kan dermed være raskere, men også mindre isolerte ([Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs), [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)).

Optimal parallellitet begrenses av målsystemet:

- API-rate-limit og Retry-After;
- maksimalt antall remoting-sheller og operasjoner per bruker;
- LDAP-/databaseforbindelsespool;
- Exchange-throttling og cmdlet-kostnader;
- minne per runspace og størrelse på serialiserte objekter;
- krav til rekkefølge og låsing på målet;
- feilbudsjett og evne til målrettet omstart.

Flere tråder øker ikke automatisk gjennomstrømmingen. Et adminskript måler kølengde, ventetid, suksessrate og throttling, begrenser samtidighet og bruker jitter/backoff. Det lagrer et sjekkpunkt per målobjekt i stedet for å gjenta hele batchen blindt etter en delfeil.

## Produktmoduler og HTTP-API-er

PowerShell er ofte bare klientsiden av en ekstern control plane. Cmdleten kan validere parametere lokalt og deretter kontakte REST, WSMan, RPC, CIM eller et proprietært API. Feilområdet omfatter da lokal motor, modul, tokeninnhenting, DNS, proxy, [TLS](/kb/tls), API-versjon, autorisasjon, throttling og produkttjenesten.

**Microsoft Graph PowerShell SDK** genererer cmdlets fra Graph-API-metadataene. Rettigheter forblir Graph-rettigheter; cmdletnavnet opphever ikke Delegated-versus-Application-semantikk, consent og ressursbegrensninger ([Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)). **ExchangeOnlineManagement**-modulen tilbyr Exchange-spesifikke cmdlets og bruker REST-baserte forbindelser for en rekke operasjoner; Exchange- og Graph-grensesnittene er ikke utskiftbare API-er til tross for overlappinger ([Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)).

Før en produksjonskjøring lagres følgende:

| Nivå | Skal logges |
|---|---|
| Klient | motor/utgave, prosess, operativsystem, tidssone |
| Kode | repository, commit, signert artefakt, konfigurasjon |
| Avhengighet | modulnavn, versjon, bane, repository, hash |
| Identitet | Object-/App-ID, autentiseringstype, Tenant/Realm |
| Rettighet | roller, scopes, consent, Endpoint/Session Configuration |
| Forespørsel | mål-ID, operasjon, Correlation-/Request-ID, tidspunkt |
| Resultat | status, feilklasse, retry, endret tilstand |
| Verifisering | uavhengig readback- eller protokollkontroll |

Feilsøkingen starter derfor med kjørekonteksten: utgave, vert, bruker, språk, modulversjon og remotingbane. Først deretter undersøkes målproduktet.

## Diagnose: først kjørekontekst, deretter målprodukt

En PowerShell-feil undersøkes langs de faktiske avhengighetene:

1. **Kall:** Hvilken prosess, vert, bruker, Working Directory, profil- og NonInteractive-modus?
2. **Oppløsning:** Hvilken kommando og modulversjon brukes faktisk?
3. **Lokal policy:** Language Mode, App Control, Execution Policy, Trust Store og proxy?
4. **Transport:** Er DNS, TCP, TLS, WSMan/SSH eller HTTPS tilgjengelig?
5. **Autentisering:** Hvilken flyt, token/nøkkel/sertifikat, klokkeslett og tenant?
6. **Autorisasjon:** Hvilken lokal rolle, JEA Capability, API-scopes og produktroller?
7. **Måltilstand:** Finnes objektet under den brukte stabile ID-en og i riktig scope?
8. **Serversvar:** Feilkode, request-ID, throttling, revisjons- og produktlogg?

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) kontrollerer TCP-tilgjengelighet på Windows, [`nc`](https://man.openbsd.org/nc) på Unix-systemer. Dette beviser verken vellykket TLS- eller applikasjonsautentisering, men skiller en tilkoblingsfeil fra høyere lag. [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) leser Event Logs i Windows; [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leser systemd-journalen.

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

## Logging, transkripsjoner og personvern

PowerShell kan bruke Script Block Logging, Module Logging, Protected Event Logging og Transcription. I Windows vises Script Block-hendelser i kanalen `Microsoft-Windows-PowerShell/Operational`. [`Start-Transcript`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript) registrerer tekstrepresentasjonen av en økt; det er ikke en fullstendig strukturert revisjonslogg og registrerer ikke all aktivitet utenfor verten ([Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)).

Logging øker synligheten, men også risikoen for å lagre tokens, passord, e-postinnhold eller personopplysninger. Microsoft anbefaler Protected Event Logging for sensitive Script Block-logger. I tillegg defineres oppbevaring, tilgang, sentral videresending, tidssynkronisering og manipulasjonssikker lagring ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

For egne automatiseringer er strukturerte hendelser bedre egnet enn fritt formaterte konsolltekster. En hendelse inneholder minst tidspunkt i UTC, Run-ID, kodeversjon, mål-ID, handling, resultat, varighet, request-ID og en klassifisert feilårsak. Hemmeligheter og unødvendige payloads fjernes før logging.

## Kvalitetskontroll og deployment

[`Invoke-ScriptAnalyzer`](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) kontrollerer PowerShell-kode med statiske regler. Det finner mange stil-, kompatibilitets- og sikkerhetsproblemer, men erstatter verken tester eller produktintegrasjon. En robust pipeline omfatter:

- parser- og analysekontroll;
- unit tests for rene funksjoner og dataavbildning;
- contract tests mot API-skjemaer og mock-svar;
- integrasjonstest med minst mulige rettigheter;
- `-WhatIf`/planutdata der det er faglig mulig;
- test av tomme mengder, duplikater, paginering, throttling og delvise feil;
- signering, hash og uforanderlig artefakt;
- trinnvis utrulling med måleverdier og avbruddskriterier;
- uavhengig readback-verifisering på målsystemet.

Profiler utelates som regel i produksjonsautomatisering med `-NoProfile`. Ikke-interaktive prosesser bruker `-NonInteractive`, definerte enkodinger, eksplisitte baner og en fast tidssone. Deployment leverer moduler med eller bruker et kontrollert internt repository; det installerer ikke ukritisk den nyeste avhengigheten ved hver kjøring.

## Teknisk historie

Jeffrey Snover formulerte i **Monad Manifesto** en enhetlig, objektorientert administrasjonsmodell: små verktøy, konsekvent navngivning, felles parser- og pipelinefunksjon samt administrative grensesnitt via .NET. Microsoft beskriver dokumentet som utgangspunktet for utviklingen som senere ble PowerShell ([PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)).

I april 2006 ga Microsoft prosjektet, som ble utviklet under kodenavnet Monad, navnet **Windows PowerShell**; den første produktlinjen var tett bundet til Windows og .NET Framework ([Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)). I denne linjen oppstod cmdlet-konvensjonene med verb-substantiv, objektpipelinen, providere, remoting, moduler, workflows og Desired State Configuration.

I 2016 publiserte Microsoft PowerShell under MIT-lisensen og utviklet en plattformuavhengig linje basert på .NET Core. PowerShell Core 6 ga støtte for Linux/macOS og SSH-remoting; PowerShell 7 fortsatte produktnavnet uten «Core». Kodebasen og utviklingen for PowerShell 7+ ligger i det offentlige repositoryet; endringer tilbakeporteres ikke til Windows PowerShell 5.1 ([PowerShell GitHub repository](https://github.com/PowerShell/PowerShell), [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)).

Denne historien forklarer mange av dagens driftsgrenser: to utgaver på Windows, moduler med .NET Framework-avhengigheter, ulike remotingtransporter og en blanding av lokale Windows-administrasjonsgrensesnitt og plattformuavhengige HTTP-API-er. For administratorer er PowerShell derfor mindre «ett versjonsnummer» enn en konkret kombinasjon av motor, kjøretid, modul, transport og målprodukt.

## Hva en administrator bør dokumentere på et øyeblikk

Et skript er først driftsklart når kjøretid, moduler, identitet, målsystem og feilhåndtering er kjent i tillegg til koden. Tabellen oppsummerer minste dokumentasjon som trengs.

| Spørsmål | Minste dokumentasjon |
|---|---|
| Hvilken PowerShell kjører? | prosessbane, PSEdition, PSVersion, .NET, OS/arkitektur |
| Hva kjøres? | løst CommandType, Source, Version, Definition |
| I hvilken kontekst? | vert, runspace, profilmodus, Working Directory, Language Mode |
| Med hvilke avhengigheter? | modulbane, manifest, versjon, repository, hash |
| Mot hvilket mål? | endepunkt, Tenant/Realm, stabil objekt-ID, API-/Session-versjon |
| Med hvilken identitet? | bruker/app/Managed Identity, authflow, sertifikat-/key-ID |
| Med hvilke rettigheter? | lokal rolle, JEA Endpoint, scopes, produkt-RBAC |
| Hva skal endres? | faktisk/ønsket differanse, plan, scope, godkjenning |
| Hva ble faktisk endret? | serversvar, request-ID, readback og funksjonstest |
| Hvordan håndteres en delfeil? | ErrorAction, exitkode, retry, sjekkpunkt, kompensasjon |
| Hvordan revideres det? | Run-ID, strukturerte logger, Transcript/Operational Log, oppbevaring |
| Hvordan gjentas eller rulles det tilbake? | idempotensnøkkel, Point of No Return, roll-forward/-back |

## Kilder

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
