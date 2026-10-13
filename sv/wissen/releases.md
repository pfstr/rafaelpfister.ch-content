---
title: "Releaser och uppdateringar: artefakter, utrullning och driftverifiering"
blatt: "releases"
description: "Release- och uppdateringsdrift för meddelande- och infrastruktursystem: versioner, kanaler, livscykel, artefaktintegritet, sårbarhetsprioritering, beroenden, stegvis utrullning, tillståndsmigrering, återställning, verifiering och teknisk utveckling."
fakten:
  - label: Process
    wert: identifiera · prioritera · anskaffa · installera · verifiera installationen
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Releaseenhet
    wert: artefakt + version + målplattform + metadata + integritetsbevis + beroenden
    href: https://theupdateframework.io/specification/latest/
  - label: Version
    wert: tolka endast inom den dokumenterade versionsmodellen; SemVer är inte universellt
    href: https://semver.org/
  - label: Prioritering
    wert: utnyttjande · exponering · påverkan · tillgänglig åtgärd · återställningsförmåga
    href: https://www.cisa.gov/known-exploited-vulnerabilities-catalog
  - label: CVSS
    wert: teknisk allvarlighetsgrad och kontextmått; inte ensamt underlag för patchtidpunkt
    href: https://www.first.org/cvss/v4.0/specification-document
  - label: EPSS
    wert: statistisk sannolikhet för observerat utnyttjande; ingen skadebedömning
    href: https://www.first.org/epss/how-it-works.html
  - label: Integritet
    wert: kontrollera ursprung, hash, signatur och vid behov build-provenance
    href: https://slsa.dev/spec/v1.2/
  - label: Utrullning
    wert: pilot/canary → små vågor → breda vågor; varje våg har avbrottskriterier
    href: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
  - label: Återställning
    wert: planera kodåtergång, dataåtergång och konfigurationsåtergång separat
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Verifiering av meddelandesystem
    wert: inkommande · utgående · Submission · autentisering · kö · policy · lagring · övervakning
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Supportgräns
    wert: bedöm produkt, branch, patchnivå, plattform och beroenden tillsammans
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq
  - label: Driftbevis
    wert: önskad/faktisk version, artefakt-ID, ändring, tid, resultat, avvikelse och ansvarig
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - releases
  - patch-management
  - messaging
translationSourceHash: f5adb284b43cb4bb0db285f310beabb3b7e8b6ac825ea0abf603e08062c0cbbb
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:45:17.093Z
translationReview: automatic
---

# Releaser och uppdateringar: artefakter, utrullning och driftverifiering

En release är en namngiven, publicerad enhet av programvara eller firmware; en uppdatering är den kontrollerade övergången av ett system i drift från ett godkänt tillstånd till ett annat. För administratörer omfattar detta betydligt mer än installationsfilen: målplattform, versions- och supportrelation, beroenden, konfigurations- och datamigreringar, signatur eller annat ursprungsbevis, kända problem, installationsväg, avbrottskriterier och verksamhetsmässig verifiering. NIST definierar Enterprise Patch Management på motsvarande sätt som att identifiera, prioritera, anskaffa, installera och verifiera patchar, uppdateringar och uppgraderingar ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Meddelandesystem skärper denna uppgift. Internetlyssnare behandlar främmande indata, köer bär på ännu inte avslutat ansvar, kluster distribuerar tillstånd och användare förväntar sig obruten leverans. Ett lyckat installationsprogram bevisar därför varken ett fungerande e-postflöde eller en supportad helhetsstack. Releasedrift är en tillståndsändring i en beroendearkitektur – med mätbar utgångspunkt, kontrollerad spridning och verifierbart sluttillstånd.

Förklaringen börjar med den konkreta artefakten och dess ursprung. Därefter följer releasetyp, riskbedömning, test och stegvis utrullning; avbrott, återställning och verifiering avslutar ändringsvägen.

En uppdatering är ett kontrollerat byte från ett känt programvaruläge till en ny artefakt. Det säkra förloppet börjar med ursprung och signatur, går via kvalificering och stegvis utrullning och avslutas först med tekniskt och verksamhetsmässigt godkännande.

## Releaseobjekt och förtroendekedja

En robust frisläppning beskriver det **exakta releaseobjektet**. Marknadsföringsnamnet ”Version 16” räcker inte när utgåva, build, hotfixnivå, plattform, arkitektur eller image-digest avgör den binärfil som faktiskt körs. Inte heller ett filnamn är en identitet: En redan publicerad artefakt får i en ren versionsmodell inte ersättas i det tysta. Semantic Versioning kräver till exempel att innehållet i en publicerad version förblir oförändrat; denna regel gäller dock endast om tillverkaren faktiskt garanterar SemVer ([Semantic Versioning 2.0.0](https://semver.org/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-releases.svg?v=20260813" title="Interaktive Infografik: Releaseobjekt, Vertrauenskette, Risikopriorisierung, Betriebsmodelle, gestufter Rollout, Abbruch, Rollback und Verifikation" loading="lazy">
  <a href="/images/kb-interaktiv-releases.svg?v=20260813">Öppna interaktiv grafik direkt</a>.
</iframe>

Ett releaseobjekt omfattar minst:

| Beståndsdel | Administratörsfråga | Bevis |
|---|---|---|
| Artefakt | Vilka byte installeras eller startas? | Hash, paket-ID, image-digest eller firmwareidentifierare |
| Tillverkare och ursprung | Från vilken godkänd kanal kommer den? | HTTPS-repository, signerade metadata, tillverkarportal, offlineöverföringskedja |
| Version och mål | För vilken produkt, utgåva, operativsystem, arkitektur och utgångsrelease? | Release Notes, kompatibilitetsmatris, uppgraderingsväg |
| Integritet och autenticitet | Är artefakten oförändrad och auktoriserad av förväntad utgivare? | Signatur, kontroll av certifikat/nyckel, hash från autentiserad källa |
| Innehåll och proveniens | Vilka komponenter och vilken build skapade den? | SBOM, build-provenance, attestering, buildsystemets identitet |
| Tillståndsändring | Vilka scheman, index, köer, konfigurationer eller nycklar ändras? | Migrationsplan, irreversibla steg, lagringsbehov och varaktighet |
| Driftsavtal | Vilka tjänster, portar, API:er, standardvärden och beroenden ändrar beteende? | Breaking Changes, utfasningar, kända problem, nya miniminivåer |

The Update Framework (TUF) modellerar säker uppdateringsdistribution med signerade roller för root-, targets-, snapshot- och timestamp-metadata. Därmed ska bland annat komprometterade repositorynycklar, återställning till gamla versioner och frysta metadata begränsas. SLSA beskriver build-provenance som verifierbar information om hur, var och från vilka indata en artefakt skapades. Båda ansatserna visar: En hash bekräftar endast överensstämmelse med ett referensvärde; först det tillförlitliga ursprunget för detta referensvärde besvarar autenticitetsfrågan ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

## Releasevägens teknikstack

Releasedrift är i sig en teknikstack. Den förenar käll- och buildsystem, artefakt- eller paketformat, signatur- och proveniensförfarande, repository eller registry, distributionskanal, installationsprogram respektive orkestrerare, konfigurations- och schemamigrering samt telemetri och återställningsstyrning. TUF beskriver ett repositorys signerade metadataroller; SLSA beskriver proveniensen mellan källäge, buildplattform och artefakt ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

För en målprodukt tillkommer dess egen stack: operativsystem, CPU-arkitektur, runtime, databas, katalogtjänst, proxy, tillägg och klienter. En release är kvalificerad först när dessa båda stackar passar ihop. En giltigt signerad containerimage kan vara oanvändbar på fel arkitektur; en lyckad pakethanterare kan ändå inte återställa en databas­migrering som inte stöds.

## Versioner, kanaler och supportgränser

Versionsnummer är namn inom ett tillverkaravtal, inte en universell riskskala. I SemVer betyder `MAJOR.MINOR.PATCH` inkompatibla API-ändringar, bakåtkompatibel funktion och bakåtkompatibel buggfix. Tillverkare kan dock använda kalender-, build-, firmware-, Cumulative Update-, feature- eller prenumerationsmodeller. Av `3.9.1` följer utan dokumentation varken att `3.10.0` är kompatibel eller att en patch är liten eller låg risk ([Semantic Versioning 2.0.0](https://semver.org/)).

Även ”stable”, ”LTS”, ”extended”, ”preview”, ”early access” och ”general availability” är tillverkarspecifika releasekanaler. Ett administratörsinventarium lagrar därför inte bara den körande versionen, utan även kanal, supportslut, supportade föregångar-/efterföljarevägar och plattformsmatris. Avgörande är **supportmängden** av produktbranch, patchnivå, operativsystem, runtime, databas, katalogtjänst, webbläsare/klient, tillägg och vid behov hårdvarumodell.

Microsofts servicingmodell för Exchange illustrerar denna flerdimensionalitet: Cumulative Updates, Security Updates och Hotfix Updates har olika förutsättningar och relationer; Security Updates är CU-specifika, och dessutom kan manuella åtgärder krävas. Den statiska administratörsregeln är därför inte ”Exchange patchas varje månad”, utan: För varje server kontrolleras supportad CU-/SU-nivå, Windows-nivå, Management Tools och tilläggsåtgärder som hör till det konkreta advisoret tillsammans ([Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)).

När artefakt, kanal och supportstatus har fastställts bedöms ändringsrisken. En hög versionssiffra säger i sig varken något om brådska eller kompatibilitet.

## Prioritering: risk i stället för versionsnummer

Alla nya versioner är inte brådskande, och alla kritiska sårbarheter får inte vänta till nästa rutinmässiga fönster. Prioriteringen sammanför flera oberoende signaler:

1. **Bevis på utnyttjande:** CISA för Known Exploited Vulnerabilities Catalog som en auktoritativ lista över aktivt utnyttjade sårbarheter och rekommenderar den som underlag för prioritering ([CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)).
2. **Teknisk allvarlighetsgrad:** CVSS v4.0 beskriver bas-, hot-, miljö- och kompletterande mått. Poängen är inte ett fullständigt uttalande om den egna tillgången eller en patchtidpunkt ([FIRST – CVSS v4.0](https://www.first.org/cvss/v4.0/specification-document)).
3. **Sannolikhet för utnyttjande:** EPSS uppskattar sannolikheten att en publicerad CVE faktiskt utnyttjas inom ett tidsfönster. EPSS bedömer inte möjlig skada och ersätter inte kunskap om tillgången ([FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)).
4. **Egen exponering:** Internetlyssnare, åtkomligt administrationsgränssnitt, befintlig autentisering, aktiv funktion, version, kompenserande kontroller och datavärde avgör om den beskrivna vägen finns i det egna systemet.
5. **Driftpåverkan:** Avbrott, kötillväxt, datamigrering, klusterförlust eller otestade beroenden kan uppstå genom själva uppdateringen.
6. **Återställningsförmåga:** En testad återställning, ersättningsnod och aktuell [Backup/DR-modell](/kb/backup-dr) möjliggör en annan hastighet än en irreversibel schemauppgradering utan återställningstest.

Resultatet är en motiverad tidsfrist med ansvarig, inte bara ”CVSS 9.8”. En aktivt utnyttjad sårbarhet vid en internetgateway kan kräva omedelbara åtgärder, minskad exponering och ett akut fönster. En oanvänd lokal komponent på ett isolerat system kan planeras in i en reguljär våg med dokumenterad kompensation. Undantag får ett utgångsdatum; annars blir de till osynliga permanenta avvikelser.

## Arkitekturansatser för uppdateringar

Återställningen och avbrottsområdet beror på arkitekturansatsen:

| Driftsmodell | Tillståndsändring | Utrullningsmönster | Central gräns |
|---|---|---|---|
| In-place-server eller appliance | Binärfiler, paket och ofta konfiguration på samma nod | Underhållsläge, uppdatera nod, kontrollera, återuppta | Partiell uppdatering eller misslyckat installationsprogram kan lämna nodens tillstånd oklart |
| Kluster med tillståndsbärande roller | Nod plus replikerings-/quorumtillstånd | Flytta roller, isolera en nod, uppdatera, resynkronisera, sedan nästa våg | Blandad versionsdrift, quorum, rebuildbelastning och versionsordning |
| Immutable image/container | Ny image ersätter gammal instans; konfiguration och data ligger utanför | Canary, RollingUpdate, Blue/Green | Återställning av imagen hjälper inte mot irreversibel datamigrering |
| SaaS / hanterad tjänst | Tillverkaren rullar ut tjänsten; kunden kontrollerar konfiguration och integrationer | Avise­rade vågor, Feature Flags, tenant-/API-kompatibilitet | Ingen binär återställning av kunden; avtal, telemetri och integrationstest är centrala |
| Firmware / hårdvaruappliance | Firmware, bootpartitioner, drivrutiner och vid behov dataformat | Standby-/HA-nod först eller dubbel bank, därefter aktiv sida | Bootbarhet, hårdvarumodell och nedgraderingsväg är tillverkarspecifika |

Kubernetes Deployments visar Rolling-modellen uttryckligen: `maxUnavailable` och `maxSurge` begränsar saknade respektive extra pods under utbytet; utrullningen kan observeras, pausas och återställas till en sparad revision. Detta garanterar endast controllerns tillstånd. En databas­migrering, ett externt kötillstånd eller inkompatibla klienter ligger utanför denna återställning ([Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)).

För meddelandekluster är ”standby först” likaså rätt endast om tillverkaren stöder denna väg. Peer-to-peer-konfigurationskluster, lastbalanserarpooler, databasreplikering och köhög tillgänglighet är olika mekanismer. Före uppdateringen måste det vara klart vilka tillstånd som replikeras, vilka versioner som tillfälligt får samverka och om en uppdaterad nod skriver data i ett format som den gamla noden fortfarande kan läsa.

## Release Notes som tekniskt avtal

Release Notes skummas inte efter nyckelord, utan översätts till en lokal ändringsmatris:

- **Säkerhet:** CVE, berörd komponent/funktion, angreppsväg, privilegium, känt utnyttjande, mitigering och ytterligare åtgärd.
- **Förutsättningar:** tillåtna utgångsversioner, mellanreleaser, ledigt lagringsutrymme, runtime, operativsystem, schema, licens och hårdvarumodell.
- **Beteende:** nya standardvärden, ändrade portar, chiffersviter/protokoll, API-/CLI-fält, autentisering, certifikat, e-postroutning och gränser.
- **Borttagning:** utfasningar, borttagna funktioner, gamla algoritmer, avvecklade plattformar eller administrationsgränssnitt.
- **Migrering:** varaktighet, låsningar, uppbyggnad av kö/index, dataformat, bakåtkompatibilitet och nödvändigt efterarbete.
- **Kända problem:** utlösare, upptäckt, workaround, mål för fix och påverkan på återställning eller HA.
- **Installationsväg:** ordning, omstarter, nodroller, Health Checks och manuella eftersteg.

Tillverkarens dokumentation är primärkällan; forum och erfarenhetsrapporter kan ge hypoteser men ersätter inte en supportad uppgraderingsrelation. NIST SSDF kräver att godkänd programvara och dess komponenter arkiveras, att integriteten skyddas och att information för åtgärdande av sårbarheter tillhandahålls. För operatörer följer av detta: Release Notes, artefakter, hash-/signaturinformation, godkännandebeslut och lokala testresultat bevaras tillsammans som refererbara ([NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

Release Notes beskriver förväntade ändringar; kvalificeringen kontrollerar dem mot den egna topologin, datamängden, integrationerna och återgångsmöjligheterna.

## Kvalificering före underhållsfönstret

Testmiljön behöver inte vara lika stor, men måste vara representativ i de **riskrelevanta egenskaperna**: samma produktutgåva, autentisering, certifikatkedjor, katalog- och databasversion, e-postflödesregler, tillägg, meddelandestorlekar, protokoll och uppgraderingsväg. Ett nyinstallerat labb testar inte ett uppgraderingsfel som bara uppstår med en konfiguration som vuxit fram under åratal.

Före godkännande kontrolleras minst:

1. nedladdning från godkänd källa och integritets-/signaturbevis;
2. förutsättningar och dokumenterad mellanväg;
3. konfigurations- och databackup med känd återställningsväg;
4. uppgradering av en representativ kopia eller pilotnod;
5. verksamhetsmässiga transaktioner inklusive negativfall och policybeslut;
6. prestanda, kö, replikering, index/rebuild och lagringstillväxt;
7. övervaknings- och larmeffekt under avbrott, degradering och återkomst;
8. återställning eller återhämtning fram till definierad beslutspunkt;
9. varaktighet med reserv för avbrott, återhämtning och verifiering.

En snapshot är ingen universell återställning. I distribuerade system kan databas, kö, blob store, katalogtjänst, certifikat och externa peers fortsätta efter snapshoten. Att återställa en enskild nod kan skapa ett inkonsekvent eller redan återbehandlat tillstånd. [Backup- och DR-modellen](/kb/backup-dr) måste ange den gemensamma konsistenspunkten och ansvarsgränserna.

## Stegvis utrullning och avbrottskriterier

En kontrollerad utrullning begränsar skaderadien genom vågor:

1. **Pilot/Canary:** liten, observerbar och representativ enhet utan unik funktion.
2. **Tidig våg:** flera system eller användare så att ovanligare beroenden blir synliga.
3. **Breda vågor:** kapacitet, regioner, roller eller tenants i definierade batcher.
4. **Efterobservation:** minst över relevanta cykler för retry, cache, kö, certifikat och dagsbelastning.

Varje våg har på förhand definierade **Go-, Hold- och Abort-kriterier**: felfrekvens, latens, köålder/-djup, SMTP-4xx/5xx-fördelning, autentiseringsfel, replikeringsfördröjning, resurser, crashloops, dataintegritet och supportärenden. ”Inga ärenden” är inte en tillräcklig Health Check. Nästa våg startar först när mätperiod och kriterier är uppfyllda.

Vid Abort återställs inte automatiskt. Om en återställning är farligare än att hålla läget – exempelvis efter en framåtriktad schemaändring – kan den säkra reaktionen vara: pausa utrullningen, ta bort trafik från den berörda noden, säkra data och följa tillverkarens väg. Beslutsmatrisen skapas före fönstret, inte improviseras under tidspress.

Efter en ren binärändring kan en återställning vara enkel. Så snart schema, index eller lagrade data har ändrats måste det i förväg klargöras om återgång fortfarande är tekniskt möjlig.

## Återställning, roll-forward och datamigrering

”Tillbaka till den gamla versionen” delas upp i minst fyra oberoende frågor om återgång:

- **Kod:** Är det gamla paketet, imagen eller firmwareimagen tillgänglig och signerad? Stöder tillverkaren nedgradering?
- **Konfiguration:** Har formatet ändrats, och förstår den gamla versionen den nya konfigurationen?
- **Data:** Har ett schema eller index migrerats framåt? Kan den gamla programvaran läsa dessa data, eller krävs en återställning?
- **Extern påverkan:** Har meddelanden accepterats, tokens utfärdats, DNS ändrats, certifikat roterats eller API:er använts av andra system?

För tillståndslösa frontends går en imageåterställning ofta snabbt. På en e-postplattform kan samma steg dubbelbehandla meddelanden eller förlora nyligen accepterade tillstånd. Därför finns en **Point of No Return**: Fram till dess är en teknisk återställning testad; därefter planeras roll-forward, återställning eller ett definierat kompatibilitetsläge.

Återställning ersätter dessutom inte orsaksanalys. Efter en återgång förblir defekt artefakt, ofullständig förutsättning, lokal avvikelse, kapacitetsgräns och okänt produktfel olika hypoteser. Releasebedömningen dokumenterar exakt version, värd, fas, felkod, tidpunkt och säkrade loggar och följer den evidensbaserade [felsökningsmetodiken](/kb/troubleshooting).

## Administratörsverktyg för inventering och verifiering

Kommandona ger byggstenar för ett ändringsbevis; de ersätter inte tillverkarens Health Checkers och produktspecifika uppgraderingsanvisningar. Utdata sparas före och efter uppdateringen med tid, värd och Change-ID. Hemligheter eller fullständiga konfigurationer hör inte oskyddade hemma i ärenden.

### Samla in plattforms- och patchinventarium

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ComputerInfo -Property WindowsProductName,WindowsVersion,OsBuildNumber,OsArchitecture
Get-HotFix | Sort-Object InstalledOn,HotFixID</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">uname -a
cat /etc/os-release
rpm -qa --qf '%{NAME}\t%{EPOCHNUM}:%{VERSION}-%{RELEASE}\t%{ARCH}\n' 2&gt;/dev/null \
  || dpkg-query -W -f='${binary:Package}\t${Version}\n'</code></pre>
  </div>
</div>

[`Get-ComputerInfo`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo) och [`Get-HotFix`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix) samlar in Windows- och registrerade uppdateringsuppgifter. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html), [`os-release`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html), [`rpm`](https://rpm.org/docs/6.0.x/man/rpm.8) och [`dpkg-query`](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html) ger operativsystems- och paketdata i Linux. Produktbuild, appliancefirmware, container-digest och tillägg måste dessutom registreras med respektive tillverkarverktyg.

### Kontrollera artefakthash och signatur

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash C:\\Staging\\update.msi -Algorithm SHA256
Get-AuthenticodeSignature C:\\Staging\\update.msi |
  Format-List Status,StatusMessage,SignerCertificate,TimeStamperCertificate</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum update.tar.gz
gpgv release-signature.asc release-manifest.txt</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) jämför byte med ett autentiserat referensvärde. [`Get-AuthenticodeSignature`](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature) och [`gpgv`](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html) kontrollerar signaturer mot den för tillfället konfigurerade förtroendeankaren. Statusen ”giltig” besvarar utgivarfrågan endast om certifikat, nyckel, namn, giltighetstid och anskaffningsväg är förväntade.

### Jämför tjänster och lyssnare före och efter utrullningen

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Dienst- und Listenerprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Service | Sort-Object Name | Select-Object Name,Status,StartType
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort | Select-Object LocalAddress,LocalPort,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">systemctl --failed --no-pager
systemctl list-units --type=service --state=running --no-pager
ss -lntup</code></pre>
  </div>
</div>

[`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service), [`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection), [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) och [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) visar lokalt tjänste- och lyssnartillstånd. En körande process är ännu ingen verksamhetsmässig Health Check; bindningsadress, port, process, TLS, protokollsvar och efterföljande beroenden kontrolleras separat.

### Kontrollera åtkomlighet till en representativ endpoint

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Smoke-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection admin.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
nc -vz -w 5 admin.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) styrker TCP-anslutningen från vald källa. Därefter följer [TLS](/kb/tls)-, [SMTP](/kb/smtp)-, autentiserings- och verksamhetsmässiga e-postflödestester; en öppen port bekräftar inte en fungerande tjänst.

### Säkra fel och varningar inom ändringsfönstret

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T18:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since; Level=1,2,3} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 18:00:00 UTC' \
  --priority warning --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) och [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) begränsar fyndet till underhållsfönstret. Loggar för installationsprogram, produkt, kluster och databas säkras dessutom med exakt källa och tidsbas. Kontrollationen söker inte bara efter nya Errors utan även efter uteblivna förväntade händelser, upprepade omstarter och växande retries.

En oansenlig processstatus räcker inte för meddelandesystem. Mottagande, kö, policy, utgående leverans, klientåtkomst och spårning måste kontrolleras längs den verkliga meddelandevägen.

## Meddelandespecifikt godkännande

Efter uppdateringen skickas inte bara ett testmejl. Godkännandet avbildar plattformens roller:

- **Inbound Relay:** extern DNS-/MX-selektion, TCP/TLS, mottagarkontroll, mottagande efter DATA, innehålls- och domänpolicy, kö och slutlig leverans.
- **Outbound Relay:** connector-/smarthostselektion, käll-IP, TLS-policy, [SPF/DKIM/DMARC](/kb/mail-auth), kö-retry och mottagande av en kontrollerad mottagare.
- **Submission och Access:** autentisering, auktorisering, IMAP/HTTP/API, Send-as/Delegate och mobila/legacy-klienter i den mån de stöds.
- **Identitet:** [LDAP](/kb/ldap)-bindning och -sökning, upplösning av grupper/attribut, [Kerberos](/kb/kerberos), tokens och certifikat.
- **Policy och kryptografi:** malware/DLP, karantän, disclaimer, routning, [S/MIME/OpenPGP/Portal](/kb/verschluesselung) och nyckelåtkomst.
- **Tillstånd och HA:** kö, databas, objektlagring, replikering, index, backupagent, failover och återkomst av den uppdaterade noden.
- **Administration:** GUI, API, CLI, roller, auditlogg, konfigurationsexport och övervakning.

Varje test har förväntat och faktiskt svar, korrelations-ID, tid, källa och mål. En grön Synthetic Check och en liten verklig pilot kompletterar varandra: Synthetic Check är reproducerbar; piloten täcker oväntade kombinationer av klient, policy och belastning. Uppdelningen mellan transport, domänautentisering och innehållsmekanismer följer den tekniska e-postarkitekturen i NIST SP 800-177 Rev. 1 ([NIST – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)).

## Teknisk utveckling av releasedrift

Tidigare distribuerades systemlägen ofta som kompletta medier eller manuellt installerade tillverkarpatchar. Pakethanterare, signerade repositories och kumulativa servicemodeller automatiserade beroenden och distribution; virtuella maskiner, images och containrar flyttade fokus till reproducerbart utbyte av instanser. CI/CD förkortade tiden mellan build och produktion, medan TUF, SLSA och SSDF uttryckligen modellerar förtroendekedjan för källkod, buildsystem, metadata och artefakt ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/), [NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

Den driftsmässiga konsekvensen är inte ”installera allt automatiskt”, utan **identifiera varje tillståndsändring maskinläsbart, kvalificera den reproducerbart, rulla ut den begränsat och verifiera den verksamhetsmässigt**. Automatisering minskar manuella avvikelser; utan vågor och avbrottslogik ökar den samtidigt hastigheten på ett fel. Moderna releaseplattformar behöver därför samma klassiska kontroller – inventarium, godkännande, underhållsfönster och återhämtning – kompletterade med artefaktproveniens, deklarativt måltillstånd, telemetri och progressiv leverans.

## Källor

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [NIST SP 800-177 Rev. 1 – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [The Update Framework – Specification](https://theupdateframework.io/specification/latest/)
- [SLSA – Specification 1.2](https://slsa.dev/spec/v1.2/)
- [Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)
- [CISA – Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [FIRST – CVSS v4.0 Specification](https://www.first.org/cvss/v4.0/specification-document)
- [FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)
- [Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [NIST – SP 800-218, Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [Microsoft Learn – Get-ComputerInfo](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo)
- [Microsoft Learn – Get-HotFix](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix)
- [GNU Coreutils – uname](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html)
- [systemd – os-release](https://www.freedesktop.org/software/systemd/man/latest/os-release.html)
- [RPM – rpm manpage](https://rpm.org/docs/6.0.x/man/rpm.8)
- [Debian – dpkg-query](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-AuthenticodeSignature](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [GnuPG – gpgv](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
