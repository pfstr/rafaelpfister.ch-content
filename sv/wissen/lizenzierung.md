---
title: "Licensiering: nyttjanderätter, mätvärden och tekniska räkneställen"
blatt: "lizenzierung"
description: "Teknisk licensiering för infrastruktur- och meddelandeadministratörer: behörigheter, användar-, enhets-, instans-, kärn- och kapacitetsmått, utgåvor, abonnemang, aktivering, licensservrar och moln-API:er, identitetskällor, HA/DR, tekniska gränser för efterlevnadskontroll, revisionsdata, öppen källkod och verifiering."
fakten:
  - label: Avgörande
    wert: Avtal, Product Terms och beställning; teknisk visning är ett underlag, inte det juridiska avtalet
    href: https://www.iso.org/standard/52293.html
  - label: Tre mängder
    wert: förvärvade rättigheter · tekniskt tilldelade · faktiskt använda
    href: https://www.iso.org/standard/52293.html
  - label: Mätvärden
    wert: användare · enhet · instans · kärna/vCPU · kapacitet · transaktion · funktion
    href: https://www.iso.org/standard/52293.html
  - label: Räkneställe
    wert: scope + objektankare + statusfilter + tidsfönster + aggregeringsregel
    href: https://www.iso.org/standard/68531.html
  - label: Programvaru-ID
    wert: SWID-taggar standardiserar produktidentifiering, inte automatiskt behörighet
    href: https://www.iso.org/standard/65666.html
  - label: Identitetslicensiering
    wert: utvärdera direkta och gruppbaserade tilldelningar, SKU och tjänstplan separat
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA och DR
    wert: passiva noder, Cold Standby, test- och återställningsinstanser ska endast bedömas enligt konkreta villkor
    href: https://www.iso.org/standard/52293.html
  - label: Efterlevnadskontroll
    wert: varning, funktionsbegränsning, ingen ny tilldelning, Grace Period eller tjänsteavbrott är produktspecifikt
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Offlinedrift
    wert: dokumentera token-/lease-tid, Grace Period, klocka, Trust Store och återställningsväg
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Öppen källkod
    wert: kostnadsfri användning innebär inte att den är fri från skyldigheter; kontrollera licenstext och distributionsscenario
    href: https://opensource.org/osd
  - label: Maskinläsbart
    wert: SPDX-uttryck modellerar enskilda, alternativa och kombinerade licenser
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Administrativt underlag
    wert: exportera Entitlement · inventarium · tilldelning · användning · undantag · tid · källa reproducerbart
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: 4abc9d3d627b9f38adb1f2fcb541662b430f07694209251464f0a4b0f9c4351f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T11:36:40.475Z
translationReview: automatic
---

# Licensiering: nyttjanderätter, mätvärden och tekniska räkneställen

Programvarulicensiering kopplar ett juridiskt nyttjandeavtal till tekniskt mätbara objekt. För administratörer är det avgörande att inte blanda ihop dessa nivåer. En licensnyckel, en molnvisning eller en intern användarräknare kan aktivera funktioner och mäta användning, men den definierar inte ensam vad en organisation juridiskt får använda. ISO/IEC 19770-3 behandlar digitala Entitlementdata uttryckligen som en avbildning av nyttjanderätter och klargör att de ursprungliga licensvillkoren har företräde för juridiska ändamål ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Den operativa uppgiften är därför att **reproducerbart jämföra förvärvade rättigheter, tekniskt installerade eller tilldelade rättigheter och faktisk användning**. Avvikelser kan innebära överanvändning, outnyttjade kostnader, felaktiga katalogfilter, övergivna konton, passiva klusternoder, utgångna token eller helt enkelt olika definitioner av samma ord, ”användare”. Artikeln beskriver det tekniska perspektivet; avtals- och rättstolkning hanteras av inköp, licenshantering och juridisk rådgivning.

Förklaringen börjar med den avtalsmässiga nyttjanderätten och följer hur produkt, mätvärde och tekniskt räkneställe gör den till en uppmätt förbrukning. Därefter följer tilldelning, efterlevnadskontroll, revision, specialfall och återställning.

En licens är först och främst en nyttjanderätt; den blir tekniskt synlig först genom tilldelning, uppmätt användning och vid behov efterlevnadskontroll. Artikeln skiljer dessa fyra nivåer åt innan den behandlar produktmått och revisioner.

## Entitlement, tilldelning, användning och efterlevnadskontroll

En korrekt licensavstämning håller reda på fyra oberoende tillstånd:

1. **Entitlement:** Vilka nyttjanderätter har förvärvats med vilket avtal, produkt, mätvärde, scope, period och specialrättighet?
2. **Driftsättning/tilldelning:** På vilka enheter, instanser, användare, tenants eller funktioner är programvara installerad, aktiverad eller tilldelad?
3. **Användning:** Vilka licensrelevanta objekt eller funktioner användes faktiskt under den överenskomna mätperioden?
4. **Efterlevnadskontroll:** Vilken gräns kontrollerar produkten tekniskt, och hur reagerar den vid utebliven anslutning, utgång eller överskridande?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

Ett befintligt Entitlement bevisar inte korrekt tilldelning; en tilldelning bevisar inte användning; låg användning upphäver inte automatiskt en namngiven användarlicens. Omvänt kan en produkt fortsätta fungera tekniskt även om en abonnemangs- eller supporträttighet har löpt ut. Efterlevnad och teknisk tillgänglighet är därför två olika kontrollmål.

ISO/IEC 19770-1 specificerar krav för ett IT-tillgångshanteringssystem. ISO/IEC 19770-2 standardiserar Software Identification Tags (SWID), som gör programvara identifierbar men enligt standarden inte föreskriver någon Entitlement-avstämning. ISO/IEC 19770-3 definierar begrepp och ett transportformat för Entitlements och tillhörande mätvärden. Tillsammans tillhandahåller de datanivåerna **inventarium**, **programvaruidentitet** och **nyttjanderätt**, inte en universell licensberäkning ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Licensmått: Vad räknas?

Ett mätvärde kan endast tolkas tillsammans med hela avtalet:

| Mätvärde | Möjligt räknarankare | Administratörsfrågor |
|---|---|---|
| Named User | oföränderligt person-/tenant-ID | räknas inaktiverade, delade, externa, tjänste- eller testkonton; får licensen omtilldelas? |
| Concurrent User/Session | aktiv session eller checkout-lease | vilken session börjar/slutar, hur hanteras tidsgränser, flera enheter och offlineleaser? |
| Device | maskinvaru-ID, registrerad enhet, klientinstans | räknas VDI, ersättningsenhet, delad enhet och nyinstallerade agenter separat? |
| Server/Instance/Node | VM, värd, appliance, klusternod, containerinstans | räknas passiva, tillfälliga, autoskalade, test- eller återställningsinstanser? |
| Processor/Core/vCPU | fysisk sockel/kärna, tilldelad vCPU, minimiantal | hur beräknas Hyperthreading, affinitet, klusterförflyttning, molnstorlekar och minimipaket? |
| Capacity | postlådor, lagring, domäner, meddelanden, genomströmning eller datamängder | gäller toppvärde, genomsnitt, månadsmaximum, provisioned eller used; vilket tidsfönster? |
| Feature/Edition | aktiverad tjänstplan, modul, databasgräns, API | räcker installation, aktivering, konfiguration eller faktisk användning? |
| Subscription/Consumption | SKU-tilldelning, krediter, requests, GB-månader | när reserveras, förbrukas, efterdebiteras eller återlämnas det? |

Den tekniska enheten får inte gissas utifrån produktnamnet. ”Per Core” kan betyda fysiska kärnor på värden, VM:ens vCPU:er eller ett normaliserat kärnmått. ”Användare” kan betyda fysisk person, aktivt konto, postlåda, licensierad identitet eller sändande avsändare. ”Instans” kan räknas som körande, installerad, registrerad eller per klusternod. ISO/IEC 19770-3 standardiserar en Entitlementstruktur men ersätter inte den konkreta definitionen i Product Terms och beställning ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Mätvärdet i sig säger ännu inte var räknandet sker. Först det tekniska räknestället förklarar varför tillverkarportalen, det lokala inventariet och fakturavärdet kan avvika från varandra.

## Det tekniska räknestället

Varje mätvärde dokumenteras som en reproducerbar räkningsfunktion:

**Räknare = Scope × objektankare × statusfilter × tidsfönster × aggregeringsregel × undantag**

- **Scope:** Organisation, tenant, domän, kluster, Subscription, plats eller avtal.
- **Objektankare:** oföränderligt användar-ID, enhets-ID, VM-ID, värdserial, SKU-ID eller hash – inte bara visningsnamn.
- **Statusfilter:** enabled, assigned, provisioned, active, seen, mounted, running eller consumed.
- **Tidsfönster:** brytdag, kalendermånad, toppvärde, genomsnitt, rullande fönster eller avtalsår.
- **Aggregering:** Distinct, summa, maximum, 95:e percentilen, minimipaket eller nivå.
- **Undantag:** externa användare, systemkonton, passiv DR-instans, Trial, NFR eller avtalsmässigt kostnadsfritt användningsfall.

Räknestället är ofta en **överlämningspunkt**. En LDAP-import kan räkna alla matchande konton, även om de aldrig använder en e-postfunktion. En gateway kan lära sig avsändare från trafiken och därmed behålla borttagna eller tekniska identiteter. En moln-SKU kan tilldelas direkt eller indirekt via en grupp. Microsoft Graphs `licenseDetails`-API tillhandahåller licenser som är direkta och ärvda via gruppmedlemskap samt enskilda tjänstplaner och provisioneringsstatus; detta visar varför ett booleskt värde som ”har licens” inte räcker för analysen ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Arkitektur för ett licenssystem

Kommersiella produkter använder olika implementationer, men kan tekniskt delas upp i kontrolytor. Modellen är en operativ syntes av ISO/IEC-19770-datanivåer och dokumenterade tillverkarmekanismer, inte en universell standardarkitektur:

| Kontrollyta | Funktion | Felområde |
|---|---|---|
| Entitlement Store | avtal/SKU, mängd, period, funktioner, specialrättigheter | felaktig beställning, utgånget, fel tenant/Smart Account |
| Inventory/Identity Source | användare, enheter, instanser, kärnor, kluster, programvaru-ID | dubblett, föråldrat objekt, fel filter eller scope |
| Assignment | tilldelar rättighet till objekt eller tjänstplan | direkt kontra gruppbaserad tilldelning, provisioneringsfel |
| Meter | samlar in användning, sessioner, kapacitet eller heartbeats | tid, offlinebuffert, sampling, saknad telemetri |
| Evaluator | tillämpar mätvärde, pool, undantag och period | avtalslogik stämmer inte med teknisk policy |
| License Service | lokal server, moln-API, token, lease, certifikat eller nyckel | DNS/TLS/proxy/klocka/trust, fel eller Rate Limit |
| Enforcement | aktiverar utgåva/funktion eller begränsar beteende | hård spärr, Grace Period, varning eller Fail-open/Fail-closed |
| Evidence Export | revisions-, användnings- och tilldelningsdata | rotation, dataskydd, saknad historik eller ej reproducerbar export |

Cisco Smart Licensing beskriver centraliserad konto- och licenshantering; Microsoft Graph exponerar förvärvade SKU:er, tilldelningar och tjänstplaner via API:er. Sådana system gör licensiering till ett distribuerat beroende av identitet, molntjänst, nätverk, TLS och tid. En produkt kan fortsätta bearbeta data men inte hämta en ny licens; en annan kan spärra funktioner efter en offlineperiod. Det konkreta beteendet måste hämtas från tillverkardokumentation och avtal ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Identitetsbaserad licensiering

För namngivna användare är katalogen en del av licenssystemet. Ett korrekt filter besvarar inte bara ”personer i OU X”, utan:

- vilket attribut som utgör det oföränderliga objektankaret;
- om inaktiverade, spärrade, borttagna eller ännu ej provisionerade konton räknas;
- hur Shared/Resource Mailboxes, tjänstekonton, gäster, externa partner och testkonton hanteras;
- om direkta och gruppbaserade tilldelningar slås samman;
- när en borttagen licens blir tillgänglig igen;
- om historisk användning eller endast beståndet på brytdagen är avgörande;
- vilka attribut produkten cachar lokalt och när den tar bort dem.

LDAP tillhandahåller poster och attribut, men ingen universell definition av ”licensierad människa”. [LDAP](/kb/ldap) kan tekniskt implementera scope och filter; mätvärdet kommer från avtalet. Samma sak gäller i molnkataloger: SKU, tjänstplan, `assignedLicenses`, provisioneringsstatus och faktisk arbetsbelastningsstatus är olika data. Microsoft Graph dokumenterar `subscribedSku` som en förvärvad kommersiell Subscription och `licenseDetails` per användare ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

Rensning av övergivna konton får inte styras direkt av licenstryck. Kontrollera först ägare, bevarande, e-postdirigering, Legal Hold, tjänsteberoenden och återställning; inaktivera eller ta sedan bort identiteten kontrollerat. En licensrapport är ett underlag i livscykeln, inte en raderingsorder.

## Instans-, kärn- och kapacitetsmodeller

Virtualisering och kluster gör det fysiska servernamnet otillräckligt. Ett inventarium behöver värd, VM/container, tilldelade vCPU:er, fysisk CPU/kärna, hypervisorkluster, mobilitetsregler, utgåva och roll. Om en VM kan flyttas mellan värdar kan hela den möjliga värdscopen vara relevant enligt avtalet; hård CPU-affinitet kan bevisas tekniskt men är inte automatiskt avtalsmässigt erkänd.

Utgåvor kopplar samman nyttjanderätt med tekniska begränsningar. Microsoft dokumenterar exempelvis utgåvor för Exchange Server som bland annat skiljer sig åt i antalet samtidiga monterade databaser; passiva databaskopior kan också räknas som monterade databaser. Product Key anger serverutgåvan. Detta är ett exempel på hur efterlevnadskontroll och kapacitetsarkitektur sammanfaller, inte en allmän licensberäkning för Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Kapacitetsmått kräver en tidsserie. Ett ögonblicksvärde visar varken månadens toppvärde eller kortvarigt överskridande. För meddelandehantering är särskilt aktiva postlådor, interna avsändare, domäner, dagligt antal meddelanden, genomströmning, lagring och krypterade användare vanliga tekniska räknare – endast produktvillkoren avgör om de är licensrelevanta. Instrumentpaneler lagrar därför råvärde, tid, scope, källa och beräkningsregel i stället för bara en indikator.

Så snart instanser eller identiteter räknas påverkar hög tillgänglighet, tester och migreringar också licensmängden. Passiva system är inte automatiskt kostnadsfria; det aktuella avtalet är avgörande.

## HA, Disaster Recovery, test och migrering

Passiva noder, Cold Standby, återställningsinstanser, labb, test, utbildning och tillfälliga parallella miljöer under en [Migration](/kb/migration) är typiska specialfall. Tekniskt ”passiv” kan ändå innebära att programvara är installerad, replikering bearbetas, databaser är monterade eller att en licens checkas ut av servern. Avtalsbegreppen måste avbildas på observerbara tillstånd.

För varje specialfall dokumenteras:

- tillåtet antal och definition av passiva/kalla instanser;
- om test, patchkvalificering, backupåterställning eller DR-test ingår;
- hur länge samtidig gammal/ny drift är tillåten vid migrering;
- om en licens är mobil och vilka villkor för omtilldelning/väntetid som gäller;
- om moln- och on-premises-rättigheter är kopplade;
- vilket underlag som skiljer ett verkligt DR-fall från produktiv permanent belastning;
- hur licenstjänsten fungerar i ett isolerat återställningsnät.

[Backup-och-DR-konceptet](/kb/backup-dr) omfattar därför även licensfiler, aktiveringsdata, licensservrar, offline-token, certifikat, tid, DNS/proxy och tillverkarkontakter. En tekniskt perfekt återställning som inte kan aktivera sin utgåva eller nödvändiga funktioner uppfyller inte återställningsmålet.

## Utgång, Grace Period och fel i licenstjänsten

För Subscription-, lease- eller molnmodeller är minst följande timers relevanta: Entitlementslut, tokenutgång, heartbeat, Borrow-/Checkout-tid, offlinefrist, Grace Period och certifikatgiltighet. Administratörsvyn registrerar **absolut tid, tidszon, synkroniseringskälla och senaste lyckade förnyelse**. Fel klocka kan utlösa en skenbar licensutgång eller förhindra validering av en signerad token.

Beteendet vid utgång är produktspecifikt:

- endast varning eller complianceevent;
- ingen ny tilldelning, befintlig användning kvarstår;
- inaktiverad premiumfunktion eller återgång till mindre utgåva;
- begränsat antal nya sessioner/användare;
- skrivskyddad drift;
- fullständigt tjänsteavbrott;
- lokal Grace Period när molntjänsten inte kan nås.

Denna reaktion fastställs i en testmiljö eller utifrån en uttrycklig tillverkarkälla, inte genom att testas i produktionsdriften. Övervakning varnar för den tidigaste operativa timern, inte först vid avtalslutet. Cisco dokumenterar egna online-/offlinemekanismer och kontomekanismer för Smart Licensing; andra tillverkare använder lokala licensservrar, signerade filer, USB-donglar, produktnycklar eller SaaS-Entitlements ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Öppen källkod: nyttjanderätt utan teknisk räknare

Öppen källkod betyder inte bara synlig källkod. Open Source Definition kräver bland annat fri vidaregivning, tillgång till källkod, härledda verk och teknikneutrala rättigheter. Enskilda licenser ställer olika villkor för ändring, vidaregivning, notices, tillhandahållande av källkod eller patentlicenser ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

Apache License 2.0 beviljar upphovsrätts- och patentlicenser under villkor och kräver bland annat vid vidaregivning en licenskopia, ändringsnoteringar och bevarande av vissa notices. Ett nedladdningspris på noll undanröjer inte dessa skyldigheter ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). För komponentkombinationer kan flera licenser gälla samtidigt eller alternativt.

SPDX License Expressions modellerar sådana fall maskinläsbart: `AND` för kumulativa skyldigheter, `OR` för ett licensval och `WITH` för ett undantag. Ett SPDX-uttryck identifierar den deklarerade licenssituationen, men utför ingen juridisk kompatibilitetskontroll ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). För administratörer hör licens- och noticefiler, SBOM/komponentförteckning, distributionsväg och egna ändringar till release- och arkivunderlaget.

## Administratörsverktyg för reproducerbara räknare

Exemplen visar tekniska frågor för inventarium och underlag. Om ett fält är licensrelevant måste härledas från villkoren. Exporter kan innehålla personuppgifter och kräver åtkomstskydd, bevarande och ändamålsbegränsning.

### Registrera CPU-, kärn- och virtualiseringsinventarium

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-CimInstance Win32_ComputerSystem |
  Select-Object Name,Manufacturer,Model,NumberOfProcessors,NumberOfLogicalProcessors,HypervisorPresent
Get-CimInstance Win32_Processor |
  Select-Object DeviceID,Name,SocketDesignation,NumberOfCores,NumberOfLogicalProcessors</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">lscpu --extended=CPU,NODE,SOCKET,CORE,ONLINE
lscpu</code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) läser CIM/WMI-inventariedata; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) samlar CPU-, kärn-, tråd-, socket- och NUMA-data. I en VM visar de främst gästvyn. Värdscope, klusterförflyttning, molninstanstyp och avtalsmässiga kärnfaktorer dokumenteras separat.

### Räkna katalogobjekt med explicit scope

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -SearchBase 'OU=Licensed,DC=example,DC=ch' -Filter * \
  -Properties ObjectGUID,Enabled,mail,employeeType |
  Select-Object ObjectGUID,SamAccountName,Enabled,mail,employeeType |
  Export-Csv .\\licensed-users.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'ou=Licensed,dc=example,dc=ch' \
  '(&(objectClass=person)(mail=*))' entryUUID uid mail employeeType</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) och [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) tillhandahåller objektankare och attribut. Search Base, filter, paging, flervärdesattribut och inaktiverade objekt dokumenteras. Exporten räknar inga ”licenspliktiga personer” så länge avtalsregeln inte exakt har avbildats på dessa fält.

### Läs ut moln-SKU:er och användartilldelningar

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-MgSubscribedSku -All |
  Select-Object SkuId,SkuPartNumber,ConsumedUnits,PrepaidUnits,CapabilityStatus
Get-MgUserLicenseDetail -UserId 'admin@example.ch' |
  Select-Object SkuId,SkuPartNumber,ServicePlans</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) och [`curl`](https://curl.se/docs/manpage.html) läser Microsoft-Graph-data. Tokens lagras inte i skript, ärenden eller shell-historik. `ConsumedUnits`, användartilldelning, tjänstplaneprovisionering och faktiskt använd arbetsbelastning utvärderas separat.

### Kontrollera licensserver eller molnslutpunkt från produktnätet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection license.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 license.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar DNS-/TCP-upprättande från det valda ursprunget. Ett lyckat resultat bevisar varken proxy, [TLS](/kb/tls), token, kontotilldelning eller licenstransaktionen. Produktloggar och tillverkarstatus visar nästa tillståndsövergång.

### Skydda revisionsexport mot förändring

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\evidence\\license-inventory.csv -Algorithm SHA256
Get-FileHash .\\evidence\\entitlements.pdf -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./evidence/license-inventory.csv
sha256sum ./evidence/entitlements.pdf</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) identifierar senare byteändringar. Dessutom dokumenteras skapandetid, frågescope, verktygs-/API-version, fråga, tidszon, exportör och säker lagring. En hash bekräftar inte saklig fullständighet.

### Kontrollera licens- och aktiveringshändelser under perioden

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-01T00:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=$since} |
  Where-Object Message -Match 'licen[cs]e|activation|entitlement' |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-01 00:00:00 UTC' \
  --output short-iso-precise --no-pager \
  | grep -Ei 'licen[cs]e|activation|entitlement'</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) och [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrerar lokala händelser. Tillverkarloggar kan använda strukturerade koder i stället för engelska textmeddelanden; provider, Event-ID eller definierade fält föredras. Tid, rotation och central loggkopia ingår i bedömningen.

Av Entitlement, räkneställe och driftstatus uppstår revisionsunderlaget. Det måste reproducerbart visa vad som köpts, tilldelats, installerats och faktiskt använts.

## Revisions- och driftunderlag

Ett reproducerbart licensunderlag innehåller:

- avtals-/beställningsreferens, produkt/SKU, mätvärde, mängd, scope, löptid och specialrättigheter;
- programvaru- och maskinvaruinventarium med oföränderliga ID:n, utgåva, version och klusterrelation;
- direkta, gruppbaserade och automatiska tilldelningar med källa och provisioneringsstatus;
- råmätvärden och beräkningsregel per tidsfönster;
- undantag med motivering, ägare, utgångsdatum och teknisk kontroll;
- HA/DR/test/migrering som egen population;
- produktvisning, API-/CLI-export och oberoende motberäkning;
- tid, tidszon, fråga/verktygsversion, hash och skyddad lagring.

Avvikelser klassificeras: **datafel** (dubbletter, gamla konton), **modellfel** (fel mätvärde), **processfel** (licens inte borttagen efter avgång), **tekniskt fel** (synk/token/server) eller **saknat Entitlement**. Först denna uppdelning visar om svaret är rensning, konfiguration, avtalsförtydligande, anskaffning eller incidenthantering.

## Licensieringens tekniska utveckling

Lokala produktnycklar och donglar kopplade till en början nyttjanderätten nära till en dator. Nätverkslicensservrar införde gemensamt använda pooler och Concurrent Leases; virtualisering krävde nya regler för värd, kärna och mobilitet. Abonnemang och SaaS flyttade Entitlements till molnkonton, identitetsgrupper och tjänstplaner. Cisco Smart Licensing och Microsoft Graph illustrerar centrala konto-/API-modeller, medan ISO/IEC 19770 standardiserar SWID- och Entitlementdata ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallellt gjorde öppen källkod teknisk aktivering överflödig för många komponenter, men inte licensvillkoren. SPDX skapade maskinläsbara kortidentifierare och uttryck för programvaruleveranskedjor. Därmed försköts administratörens uppgift från ”ange nyckel” till ett dataproblem om avtal, identitet, tillgång, löptid, telemetri och programvarusammansättning. Motmedlet är detsamma som vid övervakning: tydliga objektankare, explicita tidsfönster, rådata och reproducerbara beräkningar.

## Källor

- [ISO/IEC 19770-3:2016 – Entitlement Schema](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – IT Asset Management Systems](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Software Identification Tag](https://www.iso.org/standard/65666.html)
- [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)
- [Cisco – Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)
- [Microsoft Graph PowerShell – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)
- [Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions)
- [Microsoft – Enter Exchange Server Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)
- [Open Source Initiative – Open Source Definition](https://opensource.org/osd)
- [Apache Software Foundation – Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [SPDX Specification 3.0.1 – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)
- [Microsoft Learn – Get-CimInstance](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance)
- [Linux man-pages – lscpu](https://man7.org/linux/man-pages/man1/lscpu.1.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Graph PowerShell – Get-MgUserLicenseDetail](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [GNU Grep – Manual](https://www.gnu.org/software/grep/manual/grep.html)
