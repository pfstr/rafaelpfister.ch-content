---
title: "Lisensiering: bruksrettigheter, måltall og tekniske tellepunkter"
blatt: "lizenzierung"
description: "Teknisk lisensiering for infrastruktur- og meldingsadministratorer: rettigheter, bruker-, enhets-, instans-, kjerne- og kapasitetsmåltall, utgaver, abonnementer, aktivering, lisensservere og sky-API-er, identitetskilder, HA/DR, tekniske håndhevingsgrenser, revisjonsdata, åpen kildekode og verifisering."
fakten:
  - label: Avgjørende
    wert: Kontrakt, Product Terms og bestilling; teknisk visning er et bevis, ikke den juridiske kontrakten
    href: https://www.iso.org/standard/52293.html
  - label: Tre mengder
    wert: ervervede rettigheter · teknisk tildelt · faktisk brukt
    href: https://www.iso.org/standard/52293.html
  - label: Måltall
    wert: bruker · enhet · instans · kjerne/vCPU · kapasitet · transaksjon · funksjon
    href: https://www.iso.org/standard/52293.html
  - label: Tellepunkt
    wert: scope + objektanker + statusfilter + tidsvindu + aggregeringsregel
    href: https://www.iso.org/standard/68531.html
  - label: Programvare-ID
    wert: SWID-tagger standardiserer produktidentifikasjon, ikke automatisk rettigheten
    href: https://www.iso.org/standard/65666.html
  - label: Identitetslisensiering
    wert: vurder direkte og gruppebasert tildeling, SKU og tjenesteplan hver for seg
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA og DR
    wert: passive noder, Cold Standby, test- og gjenopprettingsinstanser må bare vurderes etter konkrete Terms
    href: https://www.iso.org/standard/52293.html
  - label: Håndheving
    wert: advarsel, funksjonsgrense, ingen ny tildeling, Grace Period eller tjenesteavbrudd er produktspesifikt
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Frakoblet drift
    wert: dokumenter token-/lease-varighet, Grace Period, klokke, Trust Store og gjenopprettingsvei
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Åpen kildekode
    wert: gratis bruk betyr ikke uten forpliktelser; kontroller lisenstekst og distribusjonsscenario
    href: https://opensource.org/osd
  - label: Maskinlesbart
    wert: SPDX-uttrykk modellerer enkeltstående, alternative og kombinerte lisenser
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Administratorbevis
    wert: eksporter Entitlement · inventar · tildeling · bruk · unntak · tid · kilde på en reproduserbar måte
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: 4abc9d3d627b9f38adb1f2fcb541662b430f07694209251464f0a4b0f9c4351f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:26:11.935Z
translationReview: automatic
---

# Lisensiering: bruksrettigheter, måltall og tekniske tellepunkter

Programvarelisensiering knytter en juridisk bruksavtale til teknisk målbare objekter. For administratorer er det avgjørende å ikke forveksle disse nivåene. En lisensnøkkel, en skyvisning eller en intern brukerteller kan aktivere funksjoner og måle bruk, men definerer ikke alene hva en organisasjon juridisk har lov til å bruke. ISO/IEC 19770-3 behandler digitale Entitlement-data uttrykkelig som en avbildning av bruksrettigheter og presiserer at de opprinnelige lisensvilkårene har forrang for juridiske formål ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Den operative oppgaven er derfor: **å avstemme ervervede rettigheter, teknisk installerte eller tildelte rettigheter og faktisk bruk på en reproduserbar måte**. Avvik kan være overforbruk, ubrukte kostnader, feil katalogfiltre, foreldreløse kontoer, passive klyngenoder, utløpte token eller ganske enkelt ulike definisjoner av det samme ordet «bruker». Artikkelen beskriver det tekniske perspektivet; kontrakts- og rettstolkning ligger hos innkjøp, lisensadministrasjon og juridisk rådgivning.

Forklaringen begynner med den kontraktsmessige bruksrettigheten og følger hvordan produkt, måltall og teknisk tellepunkt gjør den til et målt forbruk. Deretter følger tildeling, håndheving, revisjon, spesialtilfeller og gjenoppretting.

En lisens er først og fremst en bruksrettighet; teknisk synlig blir den først gjennom tildeling, målt bruk og eventuelt håndheving. Artikkelen skiller disse fire nivåene før den behandler produktmåltall og revisjoner.

## Entitlement, tildeling, bruk og håndheving

En ryddig lisensoversikt holder fire uavhengige tilstander adskilt:

1. **Entitlement:** Hvilke bruksrettigheter er ervervet med hvilken kontrakt, hvilket produkt, måltall, scope, tidsrom og særrettighet?
2. **Distribusjon/tildeling:** På hvilke enheter, instanser, brukere, tenants eller funksjoner er programvare installert, aktivert eller tilordnet?
3. **Bruk:** Hvilke lisensrelevante objekter eller funksjoner ble faktisk brukt i den avtalte måleperioden?
4. **Håndheving:** Hvilken grense kontrollerer produktet teknisk, og hvordan reagerer det ved manglende forbindelse, utløp eller overskridelse?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Åpne den interaktive grafikken direkte</a>.
</iframe>

Et eksisterende Entitlement beviser ikke korrekt tildeling; en tildeling beviser ikke bruk; lav bruk opphever ikke automatisk en navngitt brukerlisens. Omvendt kan et produkt fortsette teknisk selv om en abonnements- eller supportrettighet er utløpt. Samsvar og teknisk tilgjengelighet er derfor to ulike kontrollmål.

ISO/IEC 19770-1 spesifiserer krav til et IT-asset management-system. ISO/IEC 19770-2 standardiserer Software Identification Tags (SWID), som gjør programvare identifiserbar, men som ifølge standarden ikke krever Entitlement-reconciliation. ISO/IEC 19770-3 definerer begreper og et transportformat for Entitlements og tilhørende måltall. Samlet gir de datanivåene **inventar**, **programvareidentitet** og **bruksrettighet**, ikke en universell lisensberegning ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Lisensmåltall: Hva telles?

Et måltall kan bare tolkes sammen med den fullstendige kontrakten:

| Måltall | Mulig telleanker | Administratorspørsmål |
|---|---|---|
| Named User | uforanderlig person-/tenant-ID | telles deaktiverte, delte, eksterne, tjeneste- eller testkontoer; kan lisensen tildeles på nytt? |
| Concurrent User/Session | aktiv sesjon eller checkout-lease | hvilken sesjon starter/avslutter, hvordan behandles tidsavbrudd, flere enheter og frakoblede leases? |
| Device | maskinvare-ID, registrert enhet, klientinstans | telles VDI, erstatningsenhet, delt enhet og nyinstallerte agenter separat? |
| Server/Instance/Node | VM, vert, appliance, klyngenode, containerinstans | telles passive, midlertidige, autoskalerte, test- eller gjenopprettingsinstanser? |
| Processor/Core/vCPU | fysisk sokkel/kjerne, tildelt vCPU, minimumsantall | hvordan beregnes Hyperthreading, affinitet, klyngeflytting, skystørrelser og minimumspakker? |
| Capacity | postbokser, lagring, domener, meldinger, gjennomstrømming eller datasett | gjelder toppverdi, gjennomsnitt, månedsmaksimum, Provisioned eller Used; hvilket tidsvindu? |
| Feature/Edition | aktivert tjenesteplan, modul, databasegrense, API | er installasjon, aktivering, konfigurasjon eller faktisk bruk tilstrekkelig? |
| Subscription/Consumption | SKU-tildeling, kreditter, forespørsler, GB-måneder | når reserveres, forbrukes, etterberegnes eller tilbakeføres det? |

Den tekniske enheten må ikke gjettes ut fra produktnavnet. «Per Core» kan bety fysiske kjerner på verten, vCPU-er i VM-en eller et normalisert kjernemåltall. «Bruker» kan bety fysisk person, aktiv konto, postboks, lisensiert identitet eller avsender som sender. «Instans» kan telles som kjørende, installert, registrert eller per klyngenode. ISO/IEC 19770-3 standardiserer en Entitlement-struktur, men erstatter ikke den konkrete definisjonen i Product Terms og bestillingen ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Måltallet alene sier ennå ikke hvor det telles. Først det tekniske tellepunktet forklarer hvorfor produsentportalen, lokalt inventar og fakturaverdien kan avvike fra hverandre.

## Det tekniske tellepunktet

Hvert måltall dokumenteres som en reproduserbar tellefunksjon:

**Teller = Scope × objektanker × statusfilter × tidsvindu × aggregeringsregel × unntak**

- **Scope:** organisasjon, tenant, domene, klynge, Subscription, sted eller kontrakt.
- **Objektanker:** uforanderlig bruker-ID, enhets-ID, VM-ID, vertsserienummer, SKU-ID eller hash – ikke bare visningsnavn.
- **Statusfilter:** enabled, assigned, provisioned, active, seen, mounted, running eller consumed.
- **Tidsvindu:** referansedato, kalendermåned, toppverdi, gjennomsnitt, rullerende vindu eller kontraktsår.
- **Aggregering:** Distinct, sum, maksimum, 95. persentil, minimumspakke eller trinn.
- **Unntak:** eksterne brukere, systemkontoer, passiv DR-instans, Trial, NFR eller kontraktsmessig gratis Use Case.

Tellepunktet er ofte et **overleveringspunkt**. En LDAP-import kan telle alle matchende kontoer, selv om de aldri bruker en e-postfunksjon. En gateway kan lære avsendere fra trafikken og dermed beholde slettede eller tekniske identiteter. En sky-SKU kan tildeles direkte eller indirekte via en gruppe. Microsoft Graphs `licenseDetails`-API leverer lisenser som er arvet direkte og gjennom gruppemedlemskap, samt enkelte tjenesteplaner og klargjøringsstatus; dette viser hvorfor en boolsk verdi «har lisens» ikke er tilstrekkelig for analysen ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Arkitektur for et lisenssystem

Kommersielle produkter bruker ulike implementasjoner, men kan teknisk deles opp i kontrollflater. Denne modellen er en driftsmessig syntese av ISO/IEC-19770-datanivåer og dokumenterte produsentmekanismer, ikke en universell standardarkitektur:

| Kontrollflate | Funksjon | Feilområde |
|---|---|---|
| Entitlement Store | kontrakt/SKU, mengde, tidsrom, funksjoner, særrettigheter | feil bestilling, utløpt, feil tenant/Smart Account |
| Inventory/Identity Source | brukere, enheter, instanser, kjerner, klynger, programvare-ID | duplikat, utdatert objekt, feil filter eller scope |
| Assignment | tilordner rettighet til et objekt eller en tjenesteplan | direkte kontra gruppebasert tildeling, klargjøringsfeil |
| Meter | samler bruk, sesjoner, kapasitet eller heartbeats | tid, frakoblet buffer, sampling, manglende telemetri |
| Evaluator | anvender måltall, pool, unntak og tidsrom | kontraktslogikken stemmer ikke med teknisk policy |
| License Service | lokal server, sky-API, token, lease, sertifikat eller nøkkel | DNS/TLS/proxy/klokke/trust, avbrudd eller Rate Limit |
| Enforcement | aktiverer Edition/Feature eller begrenser atferd | hard sperre, Grace Period, advarsel eller Fail-open/fail-closed |
| Evidence Export | revisjons-, bruks- og tildelingsdata | rotasjon, personvern, manglende historikk eller ikke-reproduserbar eksport |

Cisco Smart Licensing beskriver sentralisert konto- og lisensadministrasjon; Microsoft Graph eksponerer ervervede SKU-er, tildelinger og tjenesteplaner via API-er. Slike systemer gjør lisensiering til en distribuert avhengighet av identitet, skytjeneste, nettverk, TLS og tid. Et produkt kan fortsatt behandle data, men ikke hente en ny lisens; et annet kan sperre funksjoner etter en frakoblet frist. Den konkrete atferden må hentes fra produsentdokumentasjon og kontrakt ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Identitetsbasert lisensiering

For navngitte brukere er katalogen en del av lisenssystemet. Et korrekt filter svarer ikke bare på «personer i OU X», men også på:

- hvilket attributt som utgjør den uforanderlige objektankeren;
- om deaktiverte, sperrede, slettede eller ennå ikke klargjorte kontoer telles;
- hvordan delte-/ressurspostbokser, tjenestekontoer, gjester, eksterne partnere og testkontoer behandles;
- om direkte og gruppebaserte tildelinger slås sammen;
- når en fjernet lisens blir tilgjengelig igjen;
- om historisk bruk eller bare beholdningen på referansedatoen er avgjørende;
- hvilke attributter produktet cacher lokalt, og når det sletter dem.

LDAP leverer oppføringer og attributter, men ingen universell definisjon av «lisensiert menneske». [LDAP](/kb/ldap) kan implementere scope og filter teknisk; måltallet kommer fra kontrakten. Det samme gjelder i skykataloger: SKU, tjenesteplan, `assignedLicenses`, klargjøringsstatus og faktisk workload-tilstand er ulike data. Microsoft Graph dokumenterer `subscribedSku` som et ervervet kommersielt Subscription og `licenseDetails` per bruker ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

Opprydding av foreldreløse kontoer må ikke styres direkte av lisenspress. Først kontrolleres eiere, oppbevaring, e-postruting, Legal Hold, tjenesteavhengigheter og gjenoppretting; deretter deaktiveres eller slettes identiteten kontrollert. En lisensrapport er et innspill i livssyklusen, ikke en sletteordre.

## Instans-, kjerne- og kapasitetsmodeller

Virtualisering og klynger gjør det fysiske servernavnet utilstrekkelig. Et inventar trenger vert, VM/container, tildelt vCPU, fysisk CPU/kjerne, hypervisorklynge, mobilitetsregler, Edition og rolle. Hvis en VM kan flyttes mellom verter, kan hele det mulige vertsscopet være relevant avhengig av kontrakten; en fast CPU-affinitet kan dokumenteres teknisk, men er ikke automatisk kontraktsmessig anerkjent.

Editioner knytter bruksrettighet til tekniske begrensninger. Microsoft dokumenterer for eksempel for Exchange Server Editioner som blant annet skiller seg i antall samtidig monterte databaser; passive databasekopier kan også telle som monterte databaser. Product Key angir serverens Edition. Dette er et eksempel på hvordan Enforcement og kapasitetsarkitektur faller sammen, ikke en generell Exchange-lisensberegning ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Kapasitetsmåltall trenger et tidsforløp. En øyeblikksverdi viser verken månedstopp eller kortvarig overforbruk. For meldingssystemer er særlig aktive postbokser, interne avsendere, domener, daglig antall meldinger, gjennomstrømming, lagring og krypterte brukere vanlige tekniske tellere – om de er lisensrelevante avgjøres kun av produktrettigheten. Dashbord lagrer derfor råverdi, tid, scope, kilde og beregningsregel i stedet for bare et trafikklys.

Så snart instanser eller identiteter telles, påvirker høy tilgjengelighet, tester og migreringer også lisensmengden. Passive systemer er ikke automatisk kostnadsfrie; den aktuelle kontrakten er avgjørende.

## HA, Disaster Recovery, test og migrering

Passive noder, Cold Standby, gjenopprettingsinstanser, lab, test, opplæring og midlertidige parallelle tilstander under en [migrering](/kb/migration) er typiske spesialtilfeller. Teknisk kan «passiv» likevel bety at programvare er installert, replikering behandles, databaser er montert eller en lisens sjekkes ut av serveren. Kontraktsbegrepene må avbildes på observerbare tilstander.

For hvert spesialtilfelle dokumenteres:

- tillatt antall og definisjon av passive/kalde instanser;
- om test, patchkvalifisering, backupgjenoppretting eller DR-test er inkludert;
- hvor lenge samtidig gammel/ny drift er tillatt under migrering;
- om en lisens er mobil og hvilke Reassignment-/ventebetingelser som gjelder;
- om sky- og On-premises-rettigheter er koblet;
- hvilket bevis som skiller en reell DR-hendelse fra produktiv varig last;
- hvordan lisensetjenesten fungerer i et isolert gjenopprettingsnett.

[Backup- og DR-konseptet](/kb/backup-dr) inneholder derfor også lisensfiler, aktiveringsdata, lisensservere, frakoblede token, sertifikater, klokkeslett, DNS/proxy og produsentkontakter. En teknisk perfekt gjenoppretting som ikke kan aktivere sin Edition eller nødvendige funksjoner, oppfyller ikke gjenopprettingsmålet.

## Utløp, Grace Period og svikt i lisensetjenesten

For Subscription-, lease- eller skymodeller er minst følgende tidtakere relevante: Entitlement-slutt, token-utløp, heartbeat, Borrow-/Checkout-varighet, frakoblet frist, Grace Period og sertifikatgyldighet. Administratorsynet registrerer **absolutt tid, tidssone, synkroniseringskilde og siste vellykkede fornyelse**. En feil klokke kan utløse et tilsynelatende lisensutløp eller hindre validering av et signert token.

Atferden ved utløp er produktspesifikk:

- bare advarsel eller samsvarshendelse;
- ingen ny tildeling, eksisterende bruk fortsetter;
- deaktivert Premiumfeature eller tilbakefall til mindre Edition;
- begrenset antall nye sesjoner/brukere;
- skrivebeskyttet drift;
- fullstendig tjenesteavbrudd;
- lokal Grace Period når skytjenesten ikke er tilgjengelig.

Denne reaksjonen fastslås i et testmiljø eller på grunnlag av en eksplisitt produsentkilde, ikke ved å prøve den ut i produksjon. Overvåking varsler ved den tidligste driftsmessige tidtakeren, ikke først ved kontraktsslutt. Cisco dokumenterer egne online-/offline- og kontomekanismer for Smart Licensing; andre produsenter bruker lokale lisensservere, signerte filer, USB-dongler, produktnøkler eller SaaS-Entitlements ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Åpen kildekode: Bruksrettighet uten teknisk teller

Åpen kildekode betyr ikke bare synlig kildekode. Open Source Definition krever blant annet fri videreformidling, tilgang til kildekode, avledede verk og teknologinøytrale rettigheter. Enkelte lisenser stiller ulike vilkår for endring, videreformidling, Notices, tilgjengeliggjøring av kildekode eller patentlisenser ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

Apache License 2.0 gir opphavsretts- og patentlisenser på vilkår og krever ved videreformidling blant annet en lisenskopi, merking av endringer og bevaring av bestemte Notices. En nedlastingspris på null fjerner ikke disse pliktene ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). I komponentkombinasjoner kan flere lisenser gjelde samtidig eller alternativt.

SPDX License Expressions modellerer slike tilfeller maskinlesbart: `AND` for kumulative plikter, `OR` for et lisensvalg og `WITH` for et unntak. Et SPDX-uttrykk identifiserer den erklærte lisenssituasjonen, men foretar ingen juridisk kompatibilitetskontroll ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). For administratorer hører lisens- og Notice-filer, SBOM/komponentliste, distribusjonsvei og egne endringer til i release- og arkivdokumentasjonen.

## Administratorverktøy for reproduserbare tellere

Eksemplene viser tekniske inventar- og dokumentasjonsspørringer. Om et felt er lisensrelevant, må utledes fra Terms. Eksporter kan inneholde personopplysninger og krever tilgangsbeskyttelse, oppbevaring og formålsbegrensning.

### Registrere CPU-, kjerne- og virtualiseringsinventar

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

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) leser CIM/WMI-inventardata; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) samler CPU-, kjerne-, tråd-, sokkel- og NUMA-data. I en VM viser de primært gjestesynet. Vertsscope, klyngeflytting, skyinstanstype og kontraktsmessige kjernefaktorer dokumenteres separat.

### Telle katalogobjekter med eksplisitt scope

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

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) og [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) leverer objektankere og attributter. Search Base, filter, paging, flerverdiattributter og deaktiverte objekter dokumenteres. Eksporten teller ingen «lisenspliktige personer» så lenge kontraktsregelen ikke er nøyaktig avbildet på disse feltene.

### Lese ut sky-SKU-er og brukertildelinger

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

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) og [`curl`](https://curl.se/docs/manpage.html) leser Microsoft Graph-data. Token lagres ikke i skript, saker eller Shell-historikk. `ConsumedUnits`, brukertildeling, tjenesteplan-klargjøring og faktisk brukt workload vurderes separat.

### Kontrollere lisensserver eller skyendepunkt fra produktnettverket

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

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer DNS-/TCP-oppsett fra valgt opphav. En suksess beviser verken proxy, [TLS](/kb/tls), token, kontotilknytning eller lisenstransaksjonen. Produktlogger og produsentstatus leverer neste tilstandsovergang.

### Sikre revisjonseksport mot endring

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

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) oppdager senere byteendringer. I tillegg dokumenteres opprettelsestid, spørringsscope, verktøy-/API-versjon, Query, tidssone, eksportør og sikker lagring. En hash bekrefter ikke faglig fullstendighet.

### Kontrollere lisens- og aktiveringshendelser i perioden

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

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) og [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtrerer lokale hendelser. Produsentlogger kan bruke strukturerte koder i stedet for engelske tekstmeldinger; Provider, Event-ID eller definerte felter foretrekkes. Tid, rotasjon og sentral loggkopi hører med i funnet.

Av Entitlement, tellepunkt og driftstilstand oppstår revisjonsbeviset. Det må reproduserbart vise hva som ble kjøpt, tildelt, installert og faktisk brukt.

## Revisjons- og driftsbevis

Et reproduserbart lisensbevis inneholder:

- kontrakts-/bestillingsreferanse, produkt/SKU, måltall, mengde, scope, løpetid og særrettigheter;
- programvare- og maskinvareinventar med uforanderlige ID-er, Edition, versjon og klyngerelasjon;
- direkte, gruppebaserte og automatiske tildelinger med kilde og klargjøringsstatus;
- råmåleverdier og beregningsregel per tidsvindu;
- unntak med begrunnelse, Owner, utløpsdato og teknisk kontroll;
- HA/DR/test/migrering som egen populasjon;
- produktvisning, API-/CLI-eksport og uavhengig motberegning;
- tid, tidssone, Query/verktøyversjon, hash og beskyttet lagring.

Avvik klassifiseres: **datafeil** (duplikater, gamle kontoer), **modellfeil** (feil måltall), **prosessfeil** (lisens ikke fjernet etter fratredelse), **teknisk feil** (synkronisering/token/server) eller **manglende Entitlement**. Først dette skillet viser om svaret er opprydding, konfigurasjon, kontraktsavklaring, anskaffelse eller Incident Response.

## Teknisk utvikling av lisensiering

Lokale produktnøkler og dongler knyttet først bruksrettigheten tett til én datamaskin. Nettverkslisensservere innførte delte pooler og Concurrent Leases; virtualisering krevde nye vert-, kjerne- og mobilitetsregler. Abonnementer og SaaS flyttet Entitlements til skykontoer, identitetsgrupper og tjenesteplaner. Cisco Smart Licensing og Microsoft Graph illustrerer sentrale konto-/API-modeller, mens ISO/IEC 19770 standardiserer SWID- og Entitlement-data ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Parallelt gjorde åpen kildekode teknisk aktivering overflødig for mange komponenter, men ikke lisensvilkår. SPDX etablerte maskinlesbare kortidentifikatorer og uttrykk for programvareforsyningskjeder. Dermed flyttet administratoroppgaven seg fra «registrere nøkkel» til et dataproblem om kontrakt, identitet, ressurs, løpetid, telemetri og programvaresammensetning. Mottiltaket er det samme som ved overvåking: klare objektankere, eksplisitte tidsvinduer, rådata og reproduserbare beregninger.

## Kilder

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
