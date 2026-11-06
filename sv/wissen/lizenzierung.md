---
title: "Licensiering: nyttjanderätter, mått och tekniska räknepunkter"
blatt: "lizenzierung"
description: "Teknisk licensiering för infrastruktur- och meddelandeadministratörer: rättigheter, användar-, enhets-, instans-, core- och kapacitetsmått, utgåvor, abonnemang, aktivering, licensservrar och moln-API:er, identitetskällor, HA/DR, tekniska gränser för efterlevnadskontroll, revisionsdata, öppen källkod och verifiering."
fakten:
  - label: Avgörande
    wert: Avtal, Product Terms och beställning; teknisk visning är ett underlag, inte det juridiska avtalet
    href: https://www.iso.org/standard/52293.html
  - label: Tre mängder
    wert: förvärvade rättigheter · tekniskt tilldelade · faktiskt använda
    href: https://www.iso.org/standard/52293.html
  - label: Mått
    wert: användare · enhet · instans · core/vCPU · kapacitet · transaktion · funktion
    href: https://www.iso.org/standard/52293.html
  - label: Räknepunkt
    wert: omfattning + objektankare + statusfilter + tidsfönster + aggregeringsregel
    href: https://www.iso.org/standard/68531.html
  - label: Programvaru-ID
    wert: SWID-taggar standardiserar produktidentifiering, inte automatiskt behörighet
    href: https://www.iso.org/standard/65666.html
  - label: Identitetslicensiering
    wert: utvärdera direkt och gruppbaserad tilldelning, SKU och serviceplan separat
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA och DR
    wert: passiva noder, Cold Standby, test- och återställningsinstanser får endast bedömas enligt konkreta Terms
    href: https://www.iso.org/standard/52293.html
  - label: Efterlevnadskontroll
    wert: varning, funktionsbegränsning, ingen ny tilldelning, Grace Period eller tjänsteavbrott är produktspecifika
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Offlinedrift
    wert: dokumentera token-/lease-tid, Grace Period, klocka, Trust Store och återställningsväg
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Öppen källkod
    wert: kostnadsfri användning betyder inte frihet från skyldigheter; kontrollera licenstext och distributionsscenario
    href: https://opensource.org/osd
  - label: Maskinläsbart
    wert: SPDX-uttryck modellerar enskilda, alternativa och kombinerade licenser
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Administratörsunderlag
    wert: exportera Entitlement · inventarie · tilldelning · användning · undantag · tid · källa reproducerbart
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: fb38ff7080a593642081c7fa017dd2d33735c4fd96cdf1ae256bebe4be37eb3d
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:44:34.226Z
translationReview: automatic
---

# Licensiering: nyttjanderätter, mått och tekniska räknepunkter

Programvarulicensiering kopplar samman ett juridiskt nyttjandeavtal med tekniskt mätbara objekt. För administratörer är det avgörande att inte blanda ihop dessa nivåer. En licensnyckel, en molnvisning eller en intern användarräknare kan aktivera funktioner och mäta användning, men definierar inte ensamt vad en organisation juridiskt får använda. ISO/IEC 19770-3 behandlar uttryckligen digitala Entitlementdata som en avbildning av nyttjanderätter och klargör att de ursprungliga licensvillkoren har företräde i juridiska syften ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Den operativa uppgiften är därför: **att reproducerbart jämföra förvärvade rättigheter, tekniskt installerade eller tilldelade rättigheter och faktisk användning**. Avvikelser kan vara överanvändning, oanvända kostnader, felaktiga katalogfilter, övergivna konton, passiva klusternoder, utgångna token eller helt enkelt olika definitioner av samma ord, ”användare”. Artikeln beskriver den tekniska synen; avtals- och rättstolkning ligger hos inköp, licenshantering och juridisk rådgivning.

Förklaringen börjar med den avtalsenliga nyttjanderätten och följer hur produkt, mått och teknisk räknepunkt gör den till uppmätt förbrukning. Därefter behandlas tilldelning, efterlevnadskontroll, revision, specialfall och återställning.

En licens är i första hand en nyttjanderätt; den blir tekniskt synlig först genom tilldelning, uppmätt användning och vid behov efterlevnadskontroll. Artikeln skiljer mellan dessa fyra nivåer innan den behandlar produktmått och revisioner.

## Entitlement, tilldelning, användning och efterlevnadskontroll

En korrekt licensavstämning håller isär fyra oberoende tillstånd:

1. **Entitlement:** Vilka nyttjanderätter har förvärvats med vilket avtal, produkt, mått, scope, period och specialrättighet?
2. **Distribution/tilldelning:** På vilka enheter, instanser, användare, tenants eller funktioner är programvara installerad, aktiverad eller tilldelad?
3. **Användning:** Vilka licensrelevanta objekt eller funktioner användes faktiskt under den överenskomna mätperioden?
4. **Efterlevnadskontroll:** Vilken gräns kontrollerar produkten tekniskt, och hur reagerar den vid utebliven anslutning, utgång eller överskridande?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

Ett befintligt Entitlement bevisar inte korrekt tilldelning; en tilldelning bevisar inte användning; låg användning upphäver inte automatiskt en namngiven användarlicens. Omvänt kan en produkt fortsätta fungera tekniskt trots att en abonnemangs- eller supporträttighet har löpt ut. Efterlevnad och teknisk tillgänglighet är därför två olika kontrollmål.

ISO/IEC 19770-1 specificerar krav på ett IT Asset Management-system. ISO/IEC 19770-2 standardiserar Software Identification Tags (SWID), som gör programvara identifierbar, men föreskriver enligt standarden ingen Entitlement-avstämning. ISO/IEC 19770-3 definierar begrepp och ett transportformat för Entitlements och tillhörande mått. Tillsammans tillhandahåller de datanivåerna **inventarie**, **programvaruidentitet** och **nyttjanderätt**, inte en universell licensberäkning ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Licensmått: Vad räknas?

Ett mått kan bara tolkas tillsammans med hela avtalet:

| Mått | Möjligt räkneankare | Administratörsfrågor |
|---|---|---|
| Named User | oföränderligt person-/tenant-ID | räknas inaktiverade, delade, externa, service- eller testkonton; får licensen tilldelas på nytt? |
| Concurrent User/Session | aktiv session eller Checkout-Lease | vilken session börjar/slutar, hur hanteras timeouter, flera enheter och Offline-Leases? |
| Device | maskinvaru-ID, registrerad enhet, klientinstans | räknas VDI, ersättningsenhet, Shared Device och nyinstallerade agenter separat? |
| Server/Instance/Node | VM, värd, Appliance, klusternod, containerinstans | räknas passiva, tillfälliga, autoskalade, test- eller återställningsinstanser? |
| Processor/Core/vCPU | fysisk sockel/core, tilldelad vCPU, minsta antal | hur beräknas Hyperthreading, affinitet, klusterförflyttning, molnstorlekar och minimipaket? |
| Capacity | postlådor, lagring, domäner, meddelanden, genomströmning eller dataposter | gäller toppvärde, genomsnitt, månadsmaximum, Provisioned eller Used; vilket tidsfönster? |
| Feature/Edition | aktiverad serviceplan, modul, databasgräns, API | räcker installation, aktivering, konfiguration eller faktisk användning? |
| Subscription/Consumption | SKU-tilldelning, Credits, Requests, GB-månader | när reserveras, förbrukas, efterdebiteras eller återlämnas det? |

Den tekniska enheten får inte gissas utifrån produktnamnet. ”Per Core” kan betyda fysiska cores på värden, VM:ens vCPU:er eller ett normaliserat core-mått. ”Användare” kan vara en fysisk person, ett aktivt konto, en postlåda, en licensierad identitet eller en sändande avsändare. ”Instans” kan räknas som körande, installerad, registrerad eller per klusternod. ISO/IEC 19770-3 standardiserar en Entitlement-struktur, men ersätter inte den konkreta definitionen i Product Terms och beställning ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Måttet ensamt säger ännu inte var räkningen sker. Först den tekniska räknepunkten förklarar varför tillverkarportalen, det lokala inventariet och fakturavärdet kan avvika från varandra.

## Den tekniska räknepunkten

Varje mått dokumenteras som en reproducerbar räknefunktion:

**Räknare = Scope × objektankare × statusfilter × tidsfönster × aggregeringsregel × undantag**

- **Scope:** organisation, tenant, domän, kluster, Subscription, plats eller avtal.
- **Objektankare:** oföränderligt User-ID, Device-ID, VM-ID, Hostserial, SKU-ID eller hash – inte bara visningsnamn.
- **Statusfilter:** enabled, assigned, provisioned, active, seen, mounted, running eller consumed.
- **Tidsfönster:** referensdatum, kalendermånad, toppvärde, genomsnitt, rullande fönster eller avtalsår.
- **Aggregering:** Distinct, summa, maximum, 95:e percentilen, minimipaket eller nivå.
- **Undantag:** externa användare, systemkonton, passiv DR-instans, Trial, NFR eller avtalsenligt kostnadsfritt Use Case.

Räknepunkten är ofta en **överlämningspunkt**. En LDAP-import kan räkna alla matchande konton även om de aldrig använder en e-postfunktion. En gateway kan lära sig avsändare från trafik och därmed behålla borttagna eller tekniska identiteter. En Cloud-SKU kan tilldelas direkt eller indirekt via en grupp. Microsoft Graphs `licenseDetails`-API ger direkt och via gruppmedlemskap ärvda licenser samt enskilda serviceplaner och provisioneringsstatus; det visar varför en boolean ”har licens” inte räcker för analysen ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Arkitektur för ett licenssystem

Kommersiella produkter använder olika implementationer, men kan tekniskt delas upp i kontrollytor. Den här modellen är en driftsmässig syntes av ISO/IEC-19770-datanivåer och dokumenterade tillverkarmekanismer, inte en universell standardarkitektur:

| Kontrollyta | Funktion | Felområde |
|---|---|---|
| Entitlement Store | avtal/SKU, mängd, period, funktioner, specialrättigheter | felaktig beställning, utgånget, fel tenant/Smart Account |
| Inventory/Identity Source | användare, enheter, instanser, cores, kluster, Software-ID | dubblett, inaktuellt objekt, fel filter eller scope |
| Assignment | tilldelar rättighet till ett objekt eller en serviceplan | direkt kontra gruppbaserad tilldelning, provisioneringsfel |
| Meter | samlar in användning, sessioner, kapacitet eller heartbeats | tid, offlinebuffert, sampling, saknad telemetri |
| Evaluator | tillämpar mått, pool, undantag och period | avtalslogiken överensstämmer inte med teknisk policy |
| License Service | lokal server, Cloud-API, token, lease, certifikat eller key | DNS/TLS/Proxy/klocka/trust, avbrott eller Rate Limit |
| Enforcement | aktiverar Edition/Feature eller begränsar beteende | hård spärr, Grace Period, varning eller Fail-open/Fail-closed |
| Evidence Export | revisions-, användnings- och tilldelningsdata | rotation, dataskydd, saknad historik eller ej reproducerbar export |

Cisco Smart Licensing beskriver centraliserad konto- och licenshantering; Microsoft Graph exponerar förvärvade SKU:er, tilldelningar och serviceplaner via API:er. Sådana system gör licensiering till ett distribuerat beroende av identitet, molntjänst, nätverk, TLS och tid. En produkt kan fortsätta behandla data men inte hämta en ny licens; en annan kan spärra funktioner efter en offlineperiod. Det konkreta beteendet måste hämtas från tillverkarens dokumentation och avtalet ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Identitetsbaserad licensiering

För namngivna användare är katalogen en del av licenssystemet. Ett korrekt filter svarar inte bara på ”personer i OU X”, utan även på:

- vilket attribut som utgör det oföränderliga objektankaret;
- om inaktiverade, spärrade, borttagna eller ännu ej provisionerade konton räknas;
- hur Shared/Resource Mailboxes, servicekonton, gäster, externa partner och testkonton hanteras;
- om direkta och gruppbaserade tilldelningar slås samman;
- när en borttagen licens blir tillgänglig igen;
- om historisk användning eller endast beståndet vid referensdatumet är avgörande;
- vilka attribut produkten cachelagrar lokalt och när den tar bort dem.

LDAP levererar poster och attribut, men ingen universell definition av ”licensierad människa”. [LDAP](/kb/ldap) kan tekniskt implementera scope och filter; måttet kommer från avtalet. Detsamma gäller i molnkataloger: SKU, serviceplan, `assignedLicenses`, provisioneringsstatus och faktiskt arbetsbelastningstillstånd är olika data. Microsoft Graph dokumenterar `subscribedSku` som förvärvad kommersiell Subscription och `licenseDetails` per användare ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

Rensning av övergivna konton får inte styras direkt av licenstryck. Kontrollera först ägare, lagring, e-postrouting, Legal Hold, serviceberoenden och återställning; inaktivera eller ta sedan bort identiteten kontrollerat. En licensrapport är ett underlag i livscykeln, inte en raderingsorder.

## Instans-, core- och kapacitetsmodeller

Virtualisering och kluster gör det fysiska servernamnet otillräckligt. Ett inventarium behöver värd, VM/container, tilldelad vCPU, fysisk CPU/core, Hypervisor-kluster, mobilitetsregler, utgåva och roll. Om en VM kan flyttas mellan värdar kan hela den möjliga värdomfattningen vara relevant enligt avtalet; hård CPU-affinitet kan tekniskt bevisas men är inte automatiskt avtalsmässigt erkänd.

Utgåvor kopplar nyttjanderätt till tekniska begränsningar. Microsoft dokumenterar exempelvis för Exchange utgåvor som bland annat skiljer sig åt i antalet samtidigt monterade databaser; passiva databaskopior kan också räknas som monterade databaser. Product Key anger serverutgåvan. Detta är ett exempel på hur efterlevnadskontroll och kapacitetsarkitektur sammanfaller, inte en allmän licensberäkning för Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Kapacitetsmått kräver en tidsserie. Ett momentant värde visar varken månadstopp eller kortvarig överanvändning. För meddelandehantering är i synnerhet aktiva postlådor, interna avsändare, domäner, dagligt antal meddelanden, throughput, lagring och krypterade användare vanliga tekniska räknare – om de är licensrelevanta avgörs endast av produktvillkoren. Instrumentpaneler sparar därför råvärde, tid, scope, källa och beräkningsregel i stället för bara en indikator.

Så snart instanser eller identiteter räknas påverkar hög tillgänglighet, tester och migreringar också licensmängden. Passiva system är inte automatiskt kostnadsfria; det relevanta avtalet är avgörande.

## HA, Disaster Recovery, test och migrering

Passiva noder, Cold Standby, återställningsinstanser, labb, test, utbildning och tillfälliga parallella miljöer under en [migrering](/kb/migration) är typiska specialfall. Tekniskt ”passiv” kan ändå innebära att programvara är installerad, replikering bearbetas, databaser är monterade eller att en licens checkas ut av servern. Avtalsbegreppen måste avbildas på observerbara tillstånd.

För varje specialfall dokumenteras:

- tillåtet antal och definition av passiva/kalla instanser;
- om test, patchkvalificering, backupåterställning eller DR-test ingår;
- hur länge samtidig gammal/ny drift är tillåten vid migrering;
- om en licens är mobil och vilka krav för omtilldelning/väntetid som gäller;
- om moln- och On-premises-rättigheter är kopplade;
- vilket underlag som skiljer ett verkligt DR-fall från produktiv kontinuerlig belastning;
- hur licenstjänsten fungerar i ett isolerat återställningsnät.

[Backup-och-DR-konceptet](/kb/backup-dr) omfattar därför också licensfiler, aktiveringsdata, licensservrar, offline-token, certifikat, tid, DNS/Proxy och tillverkarkontakter. En tekniskt perfekt återställning som inte kan aktivera sin utgåva eller nödvändiga funktioner uppfyller inte återställningsmålet.

## Utgång, Grace Period och fel i licenstjänsten

För Subscription-, lease- eller molnmodeller är minst följande timers relevanta: Entitlement-slut, tokenutgång, heartbeat, Borrow-/Checkout-tid, offlineperiod, Grace Period och certifikatgiltighet. Administratörsvyn håller reda på **absolut tid, tidszon, synkroniseringskälla och senaste lyckade förnyelse**. En felaktig klocka kan utlösa en skenbar licensutgång eller förhindra validering av en signerad token.

Beteendet vid utgång är produktspecifikt:

- endast varning eller compliancehändelse;
- ingen ny tilldelning, befintlig användning fortsätter;
- inaktiverad premiumfunktion eller återgång till mindre utgåva;
- begränsat antal nya sessioner/användare;
- skrivskyddad drift;
- fullständigt tjänsteavbrott;
- lokal Grace Period när molntjänsten inte kan nås.

Denna reaktion fastställs i en testmiljö eller utifrån en uttrycklig tillverkarkälla, inte genom att prövas i produktionsdriften. Övervakningen varnar för den tidigaste operativa timern, inte först vid avtalsperiodens slut. Cisco dokumenterar egna online-/offlinemekanismer och kontomekanismer för Smart Licensing; andra tillverkare använder lokala licensservrar, signerade filer, USB-donglar, produktnycklar eller SaaS-Entitlements ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Öppen källkod: nyttjanderätt utan teknisk räknare

Öppen källkod innebär inte bara synlig källkod. Open Source Definition kräver bland annat fri vidaregivning, åtkomst till källkod, härledda verk och teknikneutrala rättigheter. Enskilda licenser ställer olika villkor för ändringar, vidaregivning, Notices, tillhandahållande av källkod eller patentlicenser ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

Apache License 2.0 beviljar upphovsrätts- och patentlicenser under villkor och kräver vid vidaregivning bland annat en licenskopia, ändringsnoteringar och bevarande av vissa Notices. Ett nedladdningspris på noll undanröjer inte dessa skyldigheter ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). För sammansatta komponenter kan flera licenser gälla samtidigt eller alternativt.

SPDX License Expressions modellerar sådana fall maskinläsbart: `AND` för kumulativa skyldigheter, `OR` för ett licensval och `WITH` för ett undantag. Ett SPDX-uttryck identifierar den angivna licenssituationen, men utför ingen juridisk kompatibilitetskontroll ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). För administratörer hör licens- och Notice-filer, SBOM/komponentförteckning, distributionsväg och egna ändringar till release- och arkivunderlaget.

## Administratörsverktyg för reproducerbara räknare

Exemplen visar tekniska frågor för inventarium och underlag. Om ett fält är licensrelevant måste härledas från Terms. Exporter kan innehålla personuppgifter och kräver åtkomstskydd, lagringstid och ändamålsbegränsning.

### Registrera CPU-, core- och virtualiseringsinventarium

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

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) läser CIM/WMI-inventariedata; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) samlar CPU-, core-, tråd-, socket- och NUMA-data. I en VM visar de främst gästvyn. Hostscope, klusterförflyttning, molninstanstyp och avtalsenliga corefaktorer dokumenteras separat.

### Räkna katalogobjekt med uttryckligt scope

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

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) och [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) tillhandahåller objektankare och attribut. Search Base, filter, paging, flervärda attribut och inaktiverade objekt dokumenteras. Exporten räknar inga ”licenspliktiga personer” så länge avtalsregeln inte har avbildats exakt på dessa fält.

### Läs ut Cloud-SKU:er och användartilldelningar

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

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) och [`curl`](https://curl.se/docs/manpage.html) läser Microsoft Graph-data. Token lagras inte i skript, ärenden eller shell-historik. `ConsumedUnits`, användartilldelning, serviceplanprovisionering och faktiskt använd arbetsbelastning utvärderas separat.

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) kontrollerar DNS-/TCP-uppbyggnad från vald källa. Ett lyckat resultat bevisar varken Proxy, [TLS](/kb/tls), token, kontotilldelning eller licenstransaktionen. Produktloggar och tillverkarstatus visar nästa tillståndsövergång.

### Skydda revisionsutskrift mot förändring

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

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) upptäcker senare byteändringar. Dessutom dokumenteras skapelsetid, frågescope, verktygs-/API-version, fråga, tidszon, exportör och säker lagring. En hash bekräftar inte fackmässig fullständighet.

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

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) och [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrerar lokala händelser. Tillverkarloggar kan använda strukturerade koder i stället för engelska textmeddelanden; Provider, Event-ID eller definierade fält bör föredras. Tid, rotation och central loggkopia ingår i resultatet.

Entitlement, räknepunkt och drifttillstånd skapar revisionsunderlaget. Det måste reproducerbart visa vad som köptes, tilldelades, installerades och faktiskt användes.

## Revisions- och driftunderlag

Ett reproducerbart licensunderlag innehåller:

- avtals-/beställningsreferens, produkt/SKU, mått, mängd, scope, löptid och specialrättigheter;
- programvaru- och maskinvaruinventarium med oföränderliga ID:n, utgåva, version och klusterrelation;
- direkta, gruppbaserade och automatiska tilldelningar med källa och provisioneringsstatus;
- råa mätvärden och beräkningsregel för varje tidsfönster;
- undantag med motivering, ägare, utgångsdatum och teknisk kontroll;
- HA/DR/test/migrering som egen population;
- produktvisning, API-/CLI-export och oberoende kontrollberäkning;
- tid, tidszon, Query/verktygsversion, hash och skyddad lagring.

Avvikelser klassificeras: **datafel** (dubbletter, gamla konton), **modellfel** (felaktigt mått), **processfel** (licens inte borttagen efter avslut), **tekniskt fel** (synk/token/server) eller **saknat Entitlement**. Först denna uppdelning visar om svaret är rensning, konfiguration, avtalsklarläggande, anskaffning eller Incident Response.

## Teknisk utveckling av licensiering

Lokala produktnycklar och donglar knöt inledningsvis nyttjanderätten nära till en dator. Nätverkslicensservrar införde delade pooler och Concurrent Leases; virtualisering krävde nya regler för värdar, cores och mobilitet. Abonnemang och SaaS flyttade Entitlements till molnkonton, identitetsgrupper och serviceplaner. Cisco Smart Licensing och Microsoft Graph illustrerar centrala konto-/API-modeller, medan ISO/IEC 19770 standardiserar SWID- och Entitlementdata ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallellt gjorde öppen källkod teknisk aktivering överflödig för många komponenter, men inte licensvillkoren. SPDX skapade maskinläsbara kortidentifierare och uttryck för programvaruleveranskedjor. Därmed försköts administratörens uppgift från ”ange key” till ett dataproblem om avtal, identitet, tillgång, löptid, telemetri och programvarusammansättning. Motmodellen är densamma som för övervakning: tydliga objektankare, uttryckliga tidsfönster, rådata och reproducerbara beräkningar.

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
