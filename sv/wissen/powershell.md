---
title: "PowerShell: objektpipeline, körtid och säker administration"
blatt: "powershell"
description: "PowerShell för infrastruktur- och meddelandeadministratörer: utgåvor och .NET-körtid, värd, motor, runspace och objektpipeline, moduler och providers, fjärrkommunikation via WSMan och SSH, autentisering och JEA, fel- och ändringssemantik, parallellisering, loggning, teknisk historia och produktionsklar automatisering."
fakten:
  - label: Kategori
    wert: Shell · skriptspråk · automatiserings- och konfigurationsramverk
    href: https://learn.microsoft.com/en-us/powershell/scripting/overview
  - label: Teknikstack
    wert: C# · .NET · System.Management.Automation · inbyggda program och operativsystem-API:er
    href: https://github.com/PowerShell/PowerShell
  - label: Utgåvor
    wert: Desktop på .NET Framework · Core på modern .NET
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions
  - label: Process
    wert: powershell.exe för Windows PowerShell · pwsh för PowerShell 6 och senare
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh
  - label: Datamodell
    wert: .NET-objekt med typ, egenskaper och metoder i stället för enbart text
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects
  - label: Körning
    wert: Värd → motor → runspace → pipeline → Command Processor
    href: https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples
  - label: Utökning
    wert: Skript-, binär- och manifestmoduler via PSModulePath
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules
  - label: Fjärrkommunikation
    wert: WSMan/WinRM eller SSH; koden körs i en fjärransluten runspace
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements
  - label: Fjärrobjekt
    wert: oftast serialiserade ögonblicksbilder; metoder går förlorade vid deserialisering
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output
  - label: Execution Policy
    wert: Windows-säkerhetsfunktion mot oavsiktlig körning, ingen säkerhetsgräns
    href: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies
  - label: Delegering
    wert: JEA begränsar identitet, endpoint och tillåtna kommandon
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview
  - label: Driftbevis
    wert: Kodversion · modulupplösning · identitet · indata · ändring · resultat · loggar
    href: https://learn.microsoft.com/en-us/powershell/scripting/security/security-features
werbung:
  - newsletter
ctaThemen:
  - powershell
  - microsoft-365-exchange
  - automatisierung
translationSourceHash: a8375bc58323f55f2e87dfe08969d32b9ae12af81cbc67673940011c3245a20a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:40:32.139Z
translationReview: automatic
---

# PowerShell: objektpipeline, körtid och säker administration

PowerShell är samtidigt ett interaktivt skal, ett skriptspråk och ett inbäddningsramverk. Det skiljer den både från en ren kommandorad och från ett vanligt programmeringsspråk: kommandon levererar typade objekt till en pipeline, program kan bädda in motorn i egna processer och administrationsprodukter publicerar sitt administrationsgränssnitt som cmdlets, providers eller HTTP-API:er. Microsoft beskriver därför PowerShell som ett plattformsoberoende automatiserings- och konfigurationsramverk för strukturerade data, REST-gränssnitt och objektmodeller ([Microsoft – What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview)).

För drift är därför inte bara syntaxen relevant. Avgörande är den konkreta utgåvan och .NET-körtiden, process- och runspace-kontexten, inlästa moduler, identiteten på målsystemet, transport och auktorisering, felsemantik samt beviset på vad ett skript faktiskt har ändrat. Ett kommando som fungerar i en interaktiv session är ännu ingen tillförlitlig automatisering.

Förklaringen följer ett kommando genom parser, motor, runspace och objektpipeline fram till det system som kontaktas. Moduler, providers, fjärrkommunikation, parallellitet, säkerhet och drift relateras därefter till denna körningsmodell.

PowerShell är skal, automatiseringsspråk och administrativ körtid på samma gång. Avgörande är att cmdlets skickar .NET-objekt genom en pipeline och att produktmoduler skapar lokala eller fjärranslutna API-anrop från dem. Artikeln följer denna väg från kommandot till effekten i målsystemet.

## Körtidsarkitektur

En PowerShell-körning består av flera lager:

| Lager | Uppgift | Administrativt relevant tillstånd |
|---|---|---|
| Värd | tillhandahåller konsol, redigerare, tjänst eller inbäddat program | värdnamn, interaktiv/obevakad, in-/utdata, profil |
| Motor | parsar språket, binder parametrar och samordnar kommandon | utgåva, version, .NET, Language Mode, inställningar |
| Runspace | isolerad körningskontext med session state | identitet, variabler, funktioner, moduler, drives, endpoint |
| Pipeline | kopplar samman Command Processor och transporterar objekt/strömmar | ordning, bindning, enumerering, fel- och informationsströmmar |
| Kommando | cmdlet, funktion, skript, alias eller inbyggt program | källa, modulversion, parametrar, sidoeffekter |
| Adapter/provider/API | översätter till .NET, operativsystem, lagring eller fjärrtjänst | endpoint, autentisering, behörigheter, rate limits, felområde |

**Värden** är inte motorn. Windows Terminal, den klassiska konsolen, Visual Studio Code, en Scheduled Task eller ett program inbäddat i C# kan driva samma PowerShell-motor på olika sätt. En **runspace** är driftsmiljön där kommandona i en pipeline körs; flera runspaces inom en process kan tillhandahålla olika session states och parallellitet ([PowerShell Glossary](https://learn.microsoft.com/en-us/powershell/scripting/learn/glossary), [Microsoft – Runspace samples](https://learn.microsoft.com/en-us/powershell/scripting/developer/hosting/runspace-samples)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-powershell.svg?v=20260813" title="Interaktive Infografik: PowerShell-Laufzeit von Host, Engine und Runspace über Objektpipeline, Module, Provider und APIs bis zu Remoting, Vertrauensgrenzen und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-powershell.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

Processnamnet är ett viktigt diagnostiskt kännetecken: `powershell.exe` startar Windows PowerShell, `pwsh` den PowerShell-linje som bygger på modern .NET. Båda kan vara installerade sida vid sida på Windows och därmed se olika moduler, profiler, TLS-standardinställningar och assemblies. Kommandoradsgränssnittet för `pwsh` dokumenterar dessutom hur filer, kommandon, kodningar och icke-interaktiva anrop binds ([Microsoft – about_Pwsh](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pwsh)).

### Inventera körtid och kontext

[$PSVersionTable](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) visar utgåva, motor och plattform; [`Get-Host`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-host) beskriver värden, inte automatiskt den underliggande operativsystemsversionen. En reproducerbar felrapport bör minst innehålla processsökväg, utgåva, version, .NET-beskrivning, arkitektur, Language Mode och modulsökväg.

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

## Utgåvor, .NET och kompatibilitet

Sedan PowerShell 5.1 skiljer Microsoft mellan utgåvorna **Desktop** och **Core**. Desktop körs på det Windows-specifika .NET Framework; Core körs på modern .NET och är tillgänglig på Windows, Linux och macOS. Fältet `CompatiblePSEditions` i manifestet för en extern modul är dock bara metadata: PowerShell tvingar inte automatiskt fram den angivna kompatibiliteten ([Microsoft – about_PowerShell_Editions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_editions)).

Windows PowerShell 5.1 och PowerShell 7+ är därför två separata produkter, inte två startalternativ för samma installation. PowerShell 7 kan inte ersätta Windows PowerShell in-place; profiler, modulsökvägar och delar av API-ytan skiljer sig åt. Microsofts migreringsguide rekommenderar att beroenden och modulkompatibilitet testas uttryckligen ([Microsoft – Migrating from Windows PowerShell 5.1 to PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/migrating-from-windows-powershell-51-to-7)).

Kompatibilitetsbeslutet fattas per arbetsbelastning:

- **Motor:** kräver skriptet Windows PowerShell/Desktop eller PowerShell/Core?
- **Operativsystem:** anropar det Windows-only-cmdlets, COM, registret, WMI eller inbyggda program?
- **Modul:** är det som skriptmodul portabelt, som binärmodul byggt för rätt .NET-målkörtid eller endast användbart via en kompatibilitetsprocess?
- **Autentisering:** stöder körtiden det krävda flödet, certifikatarkivet och token cache?
- **Serialisering:** förblir det nödvändiga objektet lokalt levande, eller kommer det från en annan process som en deserialiserad ögonblicksbild?
- **Support:** anger tillverkaren exakt denna kombination av motor, modul och produkt?

PowerShell 7 kan läsa in utvalda Windows PowerShell-moduler via en implicit remoting-/kompatibilitetssession. Då uppstår dock deserialiserade objekt och en ytterligare processgräns; det ersätter inte ett inbyggt kompatibilitetsbevis ([Microsoft – Using Windows PowerShell compatibility](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_windows_powershell_compatibility)).

När utgåva och körtid har fastställts förklarar objektpipelinen den viktigaste skillnaden mot textorienterade skal: filtrering, sortering och export arbetar på egenskaper i stället för kolumnpositioner.

## Objektpipelinen

En klassisk Unix-pipeline kopplar samman byte- eller textströmmar. PowerShell skickar i grunden vidare .NET-objekt inom en process. Ett objekt har typ, egenskaper och metoder; nästa pipelinekomponent binder indata enligt parameterregler och kan komma åt egenskaper utan att parsa tabellutdata på nytt ([Microsoft – about_Objects](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects), [Microsoft – about_Pipelines](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines)).

Den synliga tabellen är inte objektet. Formateringssystemet väljer vyer och skapar formateringsinstruktioner först i slutet av pipelinen. `Format-Table` och `Format-List` hör därför hemma vid utdataänden; om de används mitt i en datapipeline får efterföljande kommandon inte längre de ursprungliga administrationsobjekten ([Microsoft – about_Format.ps1xml](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml)).

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

### Bindning, enumerering och strömmar

Pipeline-bindning sker per parameter efter värde eller egenskapsnamn. Samlingar enumereras normalt element för element; detta förändrar beteende och minnesbehov jämfört med en enskild lista. För tillförlitliga funktioner deklareras pipelineparametrar uttryckligen, indata bearbetas i `process`-blocket och utdata förorenas inte oavsiktligt av hjälpvärden.

PowerShell hanterar flera logiska strömmar: Success, Error, Warning, Verbose, Debug, Information och Progress. Endast Success-strömmen är den normala objektutdata; fel kan vara synliga utan att avsluta pipelinen. Omdirigering och sammanslagning av strömmarna är en del av anropsgränssnittet och måste fastställas medvetet för Scheduled Tasks, CI och övervakning ([Microsoft – about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)).

Inbyggda program är fortfarande en övergångspunkt. Deras argumentcitering, teckenkodning, standardutdata och exitkoder följer inte fullt ut cmdlet-modellen. En wrapper bör hantera `$LASTEXITCODE`, förväntade exitkoder, kodning och stderr samt normalisera resultatet till ett eget objekt. Att text förekommer på stderr betyder inte för varje program automatiskt ett fel.

## Kommandoresolution, moduler och providers

[`Get-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-command) visar vilket alias, vilken funktion, cmdlet, vilket skript eller inbyggt program som faktiskt anropas för ett namn. Det är viktigare än den synliga skrivformen: ett alias med samma namn eller en funktion i profilen kan dölja en cmdlet. Det fullständigt kvalificerade anropet `Modulname\\Verb-Noun` gör ursprunget explicit.

Moduler paketerar kommandon, typ- och formatdata samt beroenden. En skriptmodul använder oftast `.psm1`, en binärmodul en .NET-assembly; ett manifest `.psd1` beskriver bland annat rotmodul, version, exporterade kommandon, förutsättningar och kompatibla utgåvor. Ett manifest är inte obligatoriskt för inläsning, men krävs för publicering i PowerShell Gallery ([Microsoft – about_Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules), [Microsoft – about_Module_Manifests](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_module_manifests)).

Motorn söker moduler via `PSModulePath`. Flera installerade versioner är möjliga; autoinläsning och oprecisa `Import-Module`-anrop kan därför välja en annan implementation på en server än i testmiljön. Ett produktionsskript inventerar därför modulnamn, version, sökväg och repository samt låser beroenden i deployment i stället för att installera dem okontrollerat vid varje körning ([Microsoft – about_PSModulePath](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_psmodulepath)).

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

**Providers** tillhandahåller hierarkiska datalager med ett filsystemliknande gränssnitt. `FileSystem:`, `Registry:`, `Certificate:`, `Variable:`, `Function:` eller `WSMan:` ser ut som drives, men har olika funktioner och sidoeffekter. Före generisk kod visar `Get-PSProvider`, vilka operationer en provider stöder. Den historiska språkspecifikationen nämner providers och drives som en grundmodell för annars svåråtkomliga data ([Microsoft – PowerShell language specification, basic concepts](https://learn.microsoft.com/en-us/powershell/scripting/lang-spec/chapter-03)).

Ett lokalt kommando körs i den aktuella processen; remoting flyttar körningen till en runspace på målet. Därmed förändras identitet, nätverkssökväg, moduler och åtkomst till lokala resurser.

## Fjärrkommunikation: lokal klient, transport, fjärransluten runspace

PowerShell Remoting kör i grunden kod på målsidan. Lokalt förbereds kommando och argument; transport, listener och Session Configuration autentiserar anropet och skapar en fjärransluten runspace. Dess operativsystem, motor, moduler, policies, identitet och nätverksåtkomst avgör resultatet.

I Windows använder klassisk remoting WS-Management via WinRM. PowerShell 7 stöder dessutom SSH-remoting på Windows, Linux och macOS. Båda transporterna bär PowerShell-remotingdata, men erbjuder inte samma endpointfunktioner: Microsoft påpekar till exempel att SSH-remoting inte har någon PowerShell-endpointkonfiguration och därmed inte stöder JEA ([Microsoft – about_Remote_Requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements), [Microsoft – PowerShell remoting over SSH](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ssh-remoting-in-powershell)).

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

De använda verktygen dokumenteras under [`Test-WSMan`](https://learn.microsoft.com/en-us/powershell/module/microsoft.wsman.management/test-wsman), [`New-PSSession`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession), [`Invoke-Command`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) och OpenSSH-manualsidan för [`ssh`](https://man.openbsd.org/ssh).

### Serialisering och objektförlust

De flesta fjärrobjekt serialiseras på målet och rekonstrueras lokalt. Resultatet är en ögonblicksbild med egenskaper, men vanligtvis utan metoderna hos det ursprungliga liveobjektet; typnamnet har ofta prefixet `Deserialized.`. Formatdata och Type-Depth avgör vilka egenskaper som överförs. Ändringar av den lokala ögonblicksbilden ändrar inte målobjektet ([Microsoft – about_Remote_Output](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_output)).

Filter, aggregering och känsliga data bearbetas därför i möjligaste mån **på målet**, och endast det nödvändiga resultatet överförs. Det minskar latens, serialiseringsvolym och datautflöde:

```text
schlecht:  200'000 Remoteobjekte → serialisieren → übertragen → lokal filtern
gut:       remote filtern/projizieren/aggregieren → 200 Ergebnisobjekte übertragen
```

Den välkända «Second Hop» är ingen egenhet hos PowerShell, utan en delegeringsgräns: identiteten som autentiserar sig från adminklienten till server A har inte automatiskt återanvändbara autentiseringsuppgifter för server B. Lösningar som Kerberos Constrained Delegation, CredSSP eller ett tjänstkonto förändrar förtroendemodellen och används endast med uttrycklig hotanalys ([Microsoft – Making the second hop in PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop)).

## Autentisering, auktorisering och minsta privilegium

Remoting separerar fyra frågor:

1. **Transportförtroende:** Vilken värd talar klienten med, och hur verifieras dess nyckel eller certifikat?
2. **Autentisering:** Vilken person, Managed Identity, app, certifikat eller SSH-nyckel loggar in?
3. **Auktorisering:** Vilken Session Configuration, roll och produktbehörighet tillåter vilka operationer?
4. **Processidentitet:** Under vilket lokalt eller virtuellt konto körs det fjärranslutna kommandot faktiskt?

**Just Enough Administration (JEA)** skapar begränsade PowerShell-endpoints. Role Capabilities definierar synliga cmdlets, funktioner, providers och externa program; Session Configurations tilldelar användare eller grupper till rollerna. Virtuella konton eller gMSA:er kan utföra privilegierade åtgärder utan att ge den anropande användaren ett permanent administratörskonto. Transcripts och loggar stödjer spårbarheten ([Microsoft – JEA overview](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/jea/overview)).

JEA begränsar inte automatiskt varje kringväg. Synliga kommandon, parametrar, providers, filåtkomst, anropsbara .NET-typer och externa program granskas tillsammans. En till synes ofarlig redigerare, kompilator eller generisk skriptblock kan utöka den avsedda kommandouppsättningen.

## Execution Policy, signaturer och programkontroll

PowerShell Execution Policy är på Windows en säkerhetsfunktion mot oavsiktlig skriptkörning, **ingen säkerhetsgräns**. En användare som får köra kommandon kan anropa innehållet i ett blockerat skript på andra sätt. På system som inte är Windows verkställs policyn inte på samma sätt som i Windows ([Microsoft – about_Execution_Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)).

`AllSigned` och `RemoteSigned` använder Authenticode för att kontrollera ursprung och integritet för vissa PowerShell-filtyper. En giltig signatur säger dock endast att filen är oförändrad sedan signeringen och matchar en betrodd certifikatkedja; den bevisar inte att koden är säker eller fackmässigt korrekt ([Microsoft – about_Signing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_signing)).

En tillförlitlig kontrollkedja kombinerar:

- Source Control, granskning och oföränderliga buildartefakter;
- signatur- och hashkontroll;
- kontrollerade modulrepositories och beroendeversioner;
- App Control/System Lockdown eller jämförbar programkontroll;
- identitet med minsta privilegium och begränsade remoting-endpoints;
- hemligheter utanför källkoden;
- Script Block/Module Logging, Transcripts och central loggöverföring.

Microsoft klassificerar Execution Policy som defense-in-depth, medan App Control i System Lockdown kan bestämma Language Mode per runspace. Script Block Logging skriver bearbetade kommandon, funktioner och skript till PowerShell Operational Log; mer omfattande loggning kan i sig registrera hemligheter ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

### Kontrollera signatur och artefakt

[`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) kontrollerar Authenticode-statusen i Windows; [`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) jämför en artefakt med ett oberoende publicerat referensvärde.

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

## Hemligheter och icke-interaktiva identiteter

Ett `SecureString` är inte ett vault. Microsoft rekommenderar för nyutveckling att lösenord om möjligt ersätts med certifikat, Windows-autentisering eller andra lösenordslösa metoder. SecretManagement-modulen tillhandahåller ett enhetligt cmdlet-gränssnitt för Extension Vaults; det registrerade valvet implementerar lagring och autentisering. Registreringen är användarberoende, vilket måste beaktas för tjänstekonton och headless-automatisering ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features), [Microsoft – Understanding SecretManagement](https://learn.microsoft.com/en-us/powershell/utility-modules/secretmanagement/get-started/understanding-secretmanagement)).

För varje automatisering dokumenteras separat:

- **Workload Identity:** tjänstkonto, gMSA, Managed Identity, App Registration eller certifikat;
- **Credential Source:** vault, certifikatarkiv, workload federation eller interaktiv broker;
- **Behörighet:** lokal roll, RBAC-roll, API-scope eller Application Permission;
- **Livscykel:** utfärdande, rotation, utgång, återkallelse och Break Glass;
- **Körplats:** agent, Scheduled Task, runner, funktion eller adminklient;
- **Nätverkssökväg:** DNS, proxy, TLS Trust, endpoint och rate limit;
- **Auditankare:** run-ID, identitets-ID, code commit och change-/ticket-ID.

Miljövariabler och pipelinevariabler är transportmekanismer, inte säkra lagringsplatser. De kan hamna i barnprocesser, dumpfiler, diagnostikutdata eller CI-loggar. Även vaultåtkomst begränsas till minsta möjliga secret scope och kort livslängd.

Säker automatisering måste kunna avgöra om en operation faktiskt misslyckades. Ett rött meddelande, ett undantag, ett icke-terminerande fel och en inbyggd exitkod är olika signaler.

## Felsemantik: synlig röd rad är ingen exitmodell

PowerShell skiljer mellan icke-terminerande, statement-terminerande och script-terminerande fel. Många cmdlets skriver en icke-terminerande `ErrorRecord` för ett enskilt objekt och fortsätter bearbeta resten av pipelinen. Ett `try/catch` fångar bara fel som lämnar det kontrollerade blocket; `-ErrorAction Stop` kan höja lämpliga icke-terminerande fel för detta anrop ([Microsoft – about_Error_Handling](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_error_handling), [Microsoft – about_Try_Catch_Finally](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_try_catch_finally)).

Produktionskod definierar därför:

- vilka fel per objekt som tolereras och loggas;
- vilka fel som avbryter hela körningen;
- vilken exitkod processen levererar till scheduler eller CI;
- om ett retry är säkert och idempotent;
- vilken correlation-ID som kopplar fel, målobjekt och API-request;
- vilka resurser som stängs i `finally`-blocket;
- hur partiella ändringar upptäcks och kompenseras.

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

Exemplet använder [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod), vars HTTP- och felbeteende medvetet måste ingå i felhanteringen. För inbyggda program kontrolleras dessutom `$LASTEXITCODE`. Processvariabeln `$?` och PowerShell-felströmmarna ersätter inte ett dokumenterat avtal om exitkoderna för det anropade programmet.

## Säker ändringssemantik och idempotens

En administrativ automatisering separerar **identifiering, planering, ändring och verifiering**. Den jämför aktuellt och önskat tillstånd med stabila ID:n, skapar en explicit ändringsmängd och utför endast den nödvändiga differensen. Därefter läser den tillståndet igen via det styrande gränssnittet. «Cmdleten returnerade inget fel» är inget tillståndsbevis.

Advanced Functions kan med `SupportsShouldProcess` erbjuda Common Parameters `-WhatIf` och `-Confirm`. Funktionen anropar `$PSCmdlet.ShouldProcess()` direkt före sidoeffekten. Microsoft dokumenterar dock begränsningar vid vidarebefordran av `WhatIfPreference` mellan skriptmoduler; ett lyckat `-WhatIf` är därför en användbar förhandsvisning, ingen transaktionell garanti ([Microsoft – Everything about ShouldProcess](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-shouldprocess), [Microsoft – about_CommonParameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters)).

Ett robust ändringsflöde innehåller:

1. oföränderligt mål-ID och tidigare tillstånd;
2. förväntad differens och fackmässigt godkännande;
3. använd identitet samt endpoint- och modulversion;
4. svars-/request-ID från målsystemet;
5. återläst tillstånd och beroende funktionstest;
6. rollback- eller roll-forward-regel från Point of No Return.

Idempotens är en egenskap hos operationen och dess tillståndsmodell, inte hos språket. `Set-*` kan vara idempotent, `Add-*` är typiskt inte det; en HTTP-PUT-operation kan upprepas, medan en POST skapar ett nytt objekt vid varje retry. Före automatiskt retry klargörs API-semantik, request-ID och serverstatus.

## Parallellitet, jobb och runspaces

PowerShell kan parallellisera arbete med Background Jobs, Thread Jobs, Remoting Jobs, `ForEach-Object -Parallel` eller egna runspace-pooler. Dessa modeller skiljer sig åt i processisolering, serialisering, startkostnad och gemensamt tillstånd. Fjärr- och processjobb serialiserar objekt; Thread Jobs och runspaces delar en process och kan därför vara snabbare men också mindre isolerade ([Microsoft – about_Jobs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs), [Microsoft – ForEach-Object -Parallel](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)).

Optimal parallellitet begränsas av målsystemet:

- API-rate-limit och Retry-After;
- maximalt antal remoting-shells och operationer per användare;
- LDAP-/databasanslutningspool;
- Exchange-throttling och cmdletkostnader;
- minne per runspace och storlek på serialiserade objekt;
- krav på ordning och låsning på målet;
- felbudget och förmåga till riktad återstart.

Fler trådar ökar inte automatiskt genomströmningen. Ett administrativt skript mäter kölängd, latens, framgångsgrad och throttling, begränsar concurrency och använder jitter/backoff. Det sparar en checkpoint per målobjekt i stället för att blint upprepa hela batchen efter ett delfel.

## Produktmoduler och HTTP-API:er

PowerShell är ofta endast klientsidan till en fjärransluten control plane. Cmdleten kan validera parametrar lokalt och därefter anropa REST, WSMan, RPC, CIM eller ett proprietärt API. Felområdet omfattar då lokal motor, modul, tokeninhämtning, DNS, proxy, [TLS](/kb/tls), API-version, auktorisering, throttling och produkttjänsten.

**Microsoft Graph PowerShell SDK** genererar cmdlets från Graph API-metadata. Behörigheter förblir Graph-behörigheter; cmdletnamnet upphäver inte semantiken för Delegated kontra Application, consent och resursbegränsningar ([Microsoft – Microsoft Graph PowerShell overview](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)). Modulen **ExchangeOnlineManagement** tillhandahåller Exchange-specifika cmdlets och använder REST-baserade anslutningar för många operationer; Exchange- och Graph-gränssnitt är trots överlappningar inte utbytbara API:er ([Microsoft – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)).

Före en produktiv körning sparas följande:

| Nivå | Att logga |
|---|---|
| Klient | motor/utgåva, process, operativsystem, tidszon |
| Kod | repository, commit, signerat artefakt, konfiguration |
| Beroende | modulnamn, version, sökväg, repository, hash |
| Identitet | Object-/App-ID, autentiseringstyp, tenant/realm |
| Behörighet | roller, scopes, consent, endpoint/Session Configuration |
| Request | mål-ID, operation, correlation-/request-ID, tidpunkt |
| Resultat | status, felklass, retry, ändrat tillstånd |
| Verifiering | oberoende readback- eller protokollkontroll |

Felsökningen börjar därför med körningskontexten: utgåva, värd, användare, språk, modulversion och remotingväg. Först därefter undersöks målprodukten.

## Diagnos: först körningskontext, sedan målprodukt

En PowerShell-störning undersöks längs de faktiska beroendena:

1. **Anrop:** Vilken process, värd, användare, Working Directory, profil- och NonInteractive-läge?
2. **Upplösning:** Vilket kommando och vilken modulversion används faktiskt?
3. **Lokal policy:** Language Mode, App Control, Execution Policy, Trust Store och proxy?
4. **Transport:** Är DNS, TCP, TLS, WSMan/SSH eller HTTPS tillgängligt?
5. **Autentisering:** Vilket flöde, token/nyckel/certifikat, tid och tenant?
6. **Auktorisering:** Vilken lokal roll, JEA Capability, API-scopes och produktroller?
7. **Måltillstånd:** Finns objektet med det använda stabila ID:t och i rätt scope?
8. **Serversvar:** Felkod, request-ID, throttling, audit- och produktlogg?

[`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) kontrollerar TCP-tillgänglighet i Windows, [`nc`](https://man.openbsd.org/nc) på Unix-system. Det bevisar varken lyckad TLS- eller programautentisering, men skiljer ett anslutningsfel från högre lager. [`Get-WinEvent`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) läser Event Logs i Windows; [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) läser systemd-journalen.

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

## Loggning, Transcripts och dataskydd

PowerShell kan använda Script Block Logging, Module Logging, Protected Event Logging och Transcription. I Windows visas Script Block-händelser i kanalen `Microsoft-Windows-PowerShell/Operational`. [`Start-Transcript`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.host/start-transcript) spelar in textrepresentationen av en session; det är ingen fullständig strukturerad auditlogg och fångar inte all aktivitet utanför värden ([Microsoft – about_Logging](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging)).

Loggning ökar synligheten och samtidigt risken att lagra tokens, lösenord, e-postinnehåll eller personuppgifter. Microsoft rekommenderar Protected Event Logging för känsliga Script Block-loggar. Dessutom definieras bevarande, åtkomst, central vidarebefordran, tidssynkronisering och manipulationsskyddad lagring ([Microsoft – PowerShell security features](https://learn.microsoft.com/en-us/powershell/scripting/security/security-features)).

För egna automatiseringar är strukturerade händelser lämpligare än fritt formaterade konsoltexter. En händelse innehåller minst tid i UTC, run-ID, kodversion, mål-ID, åtgärd, resultat, varaktighet, request-ID och en klassificerad felorsak. Hemligheter och onödiga payloads tas bort före loggning.

## Kvalitetskontroll och deployment

[`Invoke-ScriptAnalyzer`](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview) kontrollerar PowerShell-kod med statiska regler. Det hittar många stil-, kompatibilitets- och säkerhetsproblem, men ersätter varken tester eller produktintegration. En tillförlitlig pipeline omfattar:

- parser- och analysekontroll;
- unit tests för rena funktioner och dataavbildning;
- contract tests mot API-scheman och mock-svar;
- integrationstest med minsta behörighet;
- `-WhatIf`/planutdata där det är fackmässigt möjligt;
- test av nollmängder, dubbletter, pagination, throttling och partiella fel;
- signering, hash och oföränderligt artefakt;
- stegvis rollout med mätvärden och avbrottskriterier;
- oberoende readback-verifiering på målsystemet.

Profiler utesluts i produktiv automatisering vanligen med `-NoProfile`. Icke-interaktiva processer använder `-NonInteractive`, definierade kodningar, explicita sökvägar och en fast tidszon. Deployment levererar moduler med eller använder ett kontrollerat internt repository; den installerar inte utan kontroll den senaste beroendeversionen vid varje körning.

## Teknisk historia

Jeffrey Snover formulerade i **Monad Manifesto** en enhetlig, objektorienterad administrationsmodell: små verktyg, konsekvent namngivning, gemensam parser- och pipelinefunktion samt administrativa gränssnitt via .NET. Microsoft beskriver dokumentet som ursprunget till den utveckling som senare blev PowerShell ([PowerShell Team – Monad Manifesto](https://devblogs.microsoft.com/powershell/monad-manifesto-the-origin-of-windows-powershell/)).

I april 2006 bytte Microsoft namn på projektet som utvecklats under kodnamnet Monad till **Windows PowerShell**; den första produktlinjen var tätt knuten till Windows och .NET Framework ([Microsoft Windows Server Blog – Monad’s new name](https://www.microsoft.com/en-us/windows-server/blog/2006/04/25/monads-new-name-windows-powershell/)). I denna linje uppstod cmdletkonventionerna verb-substantiv, objektpipelinen, providers, remoting, moduler, workflows och Desired State Configuration.

2016 publicerade Microsoft PowerShell under MIT-licensen och utvecklade en plattformsoberoende linje baserad på .NET Core. PowerShell Core 6 gav stöd för Linux/macOS och SSH-remoting; PowerShell 7 fortsatte produktnamnet utan «Core». Kodbasen och utvecklingen för PowerShell 7+ finns i det offentliga repositoryt; ändringar porteras inte tillbaka till Windows PowerShell 5.1 ([PowerShell GitHub repository](https://github.com/PowerShell/PowerShell), [PowerShell Team – Announcing PowerShell 7](https://devblogs.microsoft.com/powershell/announcing-PowerShell-7-0/)).

Denna historia förklarar många av dagens driftsgränser: två utgåvor på Windows, moduler med .NET Framework-beroenden, olika remotingtransporter och en blandning av lokala Windows-administrationsgränssnitt och plattformsoberoende HTTP-API:er. För administratörer är PowerShell därför mindre «ett versionsnummer» än en konkret kombination av motor, körtid, modul, transport och målprodukt.

## Vad en administratör bör notera i korthet

Ett skript är först driftsbart när körtid, moduler, identitet, målsystem och felhantering är kända utöver koden. Tabellen sammanfattar det minsta nödvändiga bevisunderlaget.

| Fråga | Minsta bevis |
|---|---|
| Vilken PowerShell körs? | processsökväg, PSEdition, PSVersion, .NET, OS/arkitektur |
| Vad körs? | upplöst CommandType, Source, Version, Definition |
| I vilken kontext? | värd, runspace, profilläge, Working Directory, Language Mode |
| Med vilka beroenden? | modulsökväg, manifest, version, repository, hash |
| Mot vilket mål? | endpoint, tenant/realm, stabilt objekt-ID, API-/session-version |
| Med vilken identitet? | användare/app/Managed Identity, auth flow, certifikat-/key-ID |
| Med vilka rättigheter? | lokal roll, JEA endpoint, scopes, produkt-RBAC |
| Vad ska ändras? | aktuell/önskad differens, plan, scope, godkännande |
| Vad ändrades faktiskt? | serversvar, request-ID, readback och funktionstest |
| Hur hanteras ett delfel? | ErrorAction, exitkod, retry, checkpoint, kompensation |
| Hur granskas det? | run-ID, strukturerade loggar, transcript/Operational Log, bevarande |
| Hur upprepas eller återställs det? | idempotensnyckel, Point of No Return, roll-forward/-back |

## Källor

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
