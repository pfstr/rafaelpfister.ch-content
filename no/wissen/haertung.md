---
title: "Herding: basislinjer, tillitsgrenser og administratorkontroller"
blatt: "haertung"
description: "Teknisk herding for meldings- og infrastruktursadministratorer: sikkerhetsbasislinjer, minste funksjonalitet og minste privilegium, identitets- og administrasjonsplan, nettverks- og e-postprotokollgrenser, nøkler, patch- og forsyningskjederisiko, logging, driftdeteksjon og verifisering."
fakten:
  - label: Mål
    wert: Kontrollert redusere angrepsflate, implisitt tillit og skadeomfang
    href: https://csrc.nist.gov/pubs/sp/800/123/final
  - label: Designprinsipper
    wert: Fail-safe Defaults · Complete Mediation · Least Privilege
    href: https://web.mit.edu/Saltzer/www/publications/protection/Basic.html
  - label: Basislinje
    wert: dokumentert, godkjent og kontrollerbar ønsket tilstand
    href: https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software
  - label: Minste funksjonalitet
    wert: kun forretningsmessig nødvendige funksjoner, porter, protokoller, programvare og tjenester
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf
  - label: Tilgangsmodell
    wert: ingen implisitt tildeling av tillit kun basert på nettverksplassering eller eierskap
    href: https://csrc.nist.gov/pubs/sp/800/207/final
  - label: Kontrollflater
    wert: Vert · identitet · administrasjon · nettverk/protokoll · applikasjon/data · forsyningskjede/telemetri
    href: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - label: E-postgrenser
    wert: Internett-relé · innsending/tilgang · intern levering · administrasjon
    href: https://csrc.nist.gov/pubs/sp/800/177/r1/final
  - label: Administratortilgang
    wert: personlig · sterkt autentisert · minst mulig privilegert · sporbar
    href: https://www.cisecurity.org/controls/access-control-management
  - label: Administrasjonsnettverk
    wert: restriktiv administrasjonssone atskilt fra produktive datastrømmer
    href: https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure
  - label: Patching
    wert: identifisere · prioritere · hente · installere · verifisere installasjonen
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Driftdokumentasjon
    wert: sammenligning av ønsket og faktisk tilstand for konfigurasjon, tjenester, kontoer, regler og logger
    href: https://www.cisecurity.org/controls/audit-log-management
  - label: Referanser
    wert: produsentbasislinje · CIS Benchmark · BSI IT-Grundschutz · egen risikogodkjenning
    href: https://www.cisecurity.org/cis-benchmarks
werbung:
  - tools
  - newsletter
ctaThemen:
  - haertung
  - messaging
  - security
translationSourceHash: f97c0de00da2f8c1a5d89407341fb9a46b9fc213b01559d4e297feb29c0a3c83
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:15:42.071Z
translationReview: automatic
---

# Herding: basislinjer, tillitsgrenser og administratorkontroller

Herding er kontrollert overgang av et system til en **dokumentert, begrunnet og verifiserbar ønsket tilstand**. Den begrenser funksjoner, tilganger og tillitsforhold til det driftsmessige minimumet uten ukontrollert å skade den tiltenkte tjenesten. Resultatet er ikke en lengst mulig liste over aktiverte sikkerhetsalternativer, men en arkitektur der hvert tilgjengelige grensesnitt, hvert privilegium og hver datastrøm har et navngitt formål, en eier og dokumentasjon. NIST beskriver serversikkerhet tilsvarende som valg, implementering og løpende vedlikehold av egnede kontroller; CIS Control 4 krever sikre konfigurasjoner for ressurser og programvare ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)).

En produktstandard er verken automatisk usikker eller automatisk den riktige produksjonsbasislinjen. Produsenter må dekke brede funksjons- og kompatibilitetsområder. Operatøren kjenner derimot eksponering, beskyttelsesbehov, avhengigheter, gjenopprettingsevne og akseptert restrisiko. En basislinje kobler derfor produsentanbefalinger, en egnet CIS- eller BSI-referanse og egen arkitekturbeslutning. Avvik beholdes ikke stilltiende, men dokumenteres med årsak, risiko, kompenserende kontroll, eier og utløpsdato. CIS Benchmarks er konsensusbaserte konfigurasjonsanbefalinger; BSI skiller mellom generelle serverkrav og e-postrelaterte krav ([CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks), [BSI SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

Herding starter med den faktiske tjenesten og dens administrasjonsveier, ikke med en tilfeldig liste over registerverdier. Først inventariseres komponenter, identiteter, data og nettverksveier; derfra oppstår basislinje, unntak og verifiserbare kontroller.

## Fra sjekkliste til kontrollflatemodell

For administratorer er en kontroll inndelt etter kontrollflater mer robust enn én enkelt vertssjekkliste. Følgende modell samler kontroller fra NIST SP 800-53: konfigurasjonsadministrasjon og minste funksjonalitet, tilgang og autentisering, kommunikasjonsbeskyttelse, systemintegritet, revisjon samt forsyningskjede- og gjenopprettingskontroller. Det er en kontrollmodell, ikke en tilleggstandard ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [NIST SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

| Kontrollflate | Beskyttelsesobjekt | Typiske ønskede verdier | Driftsdokumentasjon |
|---|---|---|---|
| Vert og runtime | Operativsystem, containere, tjenester, filrettigheter, kjerne-/runtime-funksjoner | minimale pakker, minimale lyttere, ikke-privilegerte prosesser, sikre filrettigheter | tjeneste- og portinventar, basislinjeskanning, integritetskontroll |
| Identitet og rettigheter | Personer, tjenestekontoer, roller, tokener, sertifikater | personlig administratoridentitet, MFA, minste privilegium, separate tjenesteidentiteter | konto-/roller gjennomgang, autentiserings- og privilegiehendelser |
| Administrasjonsplan | GUI, API, SSH, PowerShell, SNMP, sikkerhetskopierings- og oppdateringskanal | dedikert administrasjonssone, krypterte protokoller, Default Deny, Break-Glass-prosess | tilgjengelige administrasjonsveier, AAA-logger, konfigurasjonsendringer |
| Nettverk og protokoller | Lyttere, utgående trafikk, TLS, DNS, relé-, innsending- og tilgangsveier | eksplisitte flyter per rolle, ingen unødvendige klartekstprotokoller, kontrollerte sertifikater | brannmurregler, pakke-/TLS-tester, DNS- og e-postflytovervåking |
| Applikasjon og data | kø, postboks, policy, parser, midlertidige data, nøkler | separate roller, restriktive filrettigheter, sikre standardinnstillinger, begrensede parser- og utgående rettigheter | faglige negative tester, kø-/policy-logger, gjennomgang av hemmeligheter og nøkler |
| Forsyningskjede og telemetri | Images, pakker, signaturer, avhengigheter, logger, tid | støttede artefakter, verifisert opprinnelse, patchprosess, sentrale uforfalskede logger | inventar, hash/signatur, patch- og driftrapport, alarmtest |

NIST Zero Trust legger til en viktig grense: En bruker, tjeneste eller enhet får ikke tillit bare fordi den befinner seg i det interne nettverket eller tilhører organisasjonen. Autentisering og autorisasjon vurderes før tilgang til en ressurs. Segmentering forblir nyttig, men erstatter ikke identitet, policy og løpende beslutning ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-haertung.svg?v=20260813" title="Interaktive Infografik: Härtungsmodell für Messaging-Systeme mit externen Mailgrenzen, Managementebene, Identität, Host, Anwendung, Daten, Lieferkette, Telemetrie und Verifikationsschleife" loading="lazy">
  <a href="/images/kb-interaktiv-haertung.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

## Teknologistakk som herdingsinventar

Herding har ingen egen programmeringsspråk- eller produktstakk. Den anvendes på den **faktisk driftede teknologistakken**. Inventaret omfatter derfor minst fastvare og hypervisor, operativsystem eller containerbase, runtime og programmeringsspråk, web- og e-postserver, biblioteker og parsere, database, kø og objektlager, identitets- og nøkkelkomponenter, administrasjonsprotokoller samt logging- og oppdateringsveier. NIST CM-8 krever et inventar over systemkomponentene; CM-7 kobler dette inventaret til begrensning til nødvendige funksjoner ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)).

For hvert lag registreres produsent, opprinnelse, supportstatus, aktive moduler, privilegier, lyttere, utgående mål, konfigurasjonskilde, patchvei og gjenopprettingsobjekt. Bare slik kan en anbefaling som «deaktiver unødvendige tjenester» brukes på en konkret prosess og dens avhengigheter uten å skade e-postflyten eller gjenopprettbarheten.

## Tillitsgrenser i en meldingsplattform

En meldingsplattform har flere teknisk ulike inngangsveier. De må ikke behandles med én enkelt regel om «kun autentiserte forbindelser»:

- **Internett-relé:** En offentlig tilgjengelig MTA mottar på [SMTP](/kb/smtp) port 25 meldinger fra MTAs som ikke er kjent på forhånd. Her begrenser mottakerkontroll, relépolicy, protokolltilstander, ressursgrenser, omdømme og innholdskontroller risikoen; brukerinnlogging er ikke den generelle tillitsmodellen.
- **Meldingsinnsending:** Brukere og applikasjoner overleverer nye meldinger som identifiserte avsendere. Innsending skiller denne rollen fra relé; autentisering, autorisasjon, hastighetsgrenser og [TLS](/kb/tls) inngår i policyen.
- **E-posttilgang:** IMAP, POP eller HTTP får tilgang til eksisterende postboksdata. RFC 8314 anser klartekst for innsending og e-posttilgang som foreldet og foretrekker implisitt TLS.
- **Interne tjenesteveier:** Gatewayer, kataloger, databaser, objektlagre, køer og skannere kommuniserer som tjenester med hverandre. Nettverksplassering alene er ikke identitetsbevis; hver forbindelse trenger en minimal, rettet data- og rettighetsvei.
- **Administrasjon og oppdateringer:** Admin-GUI, API, SSH, remoting, sikkerhetskopiering og programvareinnhenting har større skadeomfang enn en vanlig klientvei og hører hjemme i en egen administrasjons- og tillitssone.

NIST SP 800-177 behandler domeneautentisering, TLS og innholdskryptografi som supplerende sikkerhetsmekanismer rundt SMTP, som fortsatt er i bruk. RFC 8314 skiller bevisst relé fra innsending og tilgang. Følgelig må herding kontrollere per rolle **hvem som kan initiere en forbindelse, hvilken identitet den bærer, hvilke data den behandler og hvor den kan kommunisere videre** ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final), [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Inventaret viser hva som må beskyttes. En basislinje oversetter det til konkrete innstillinger som versjoneres, testes og ved begrunnede unntak endres på en sporbar måte.

## Basislinjelivssyklus og kontrollert avvik

En effektiv basislinje går gjennom en livssyklus:

1. **Inventariser:** Registrer produkt, rolle, programvareversjon, moduler, lyttere, kontoer, datastrømmer, nøkler og avhengigheter.
2. **Velg referanse:** Kartlegg produsentbasislinje, CIS Benchmark, BSI-modul og lovkrav mot den konkrete rollen.
3. **Tilpass:** Fjern regler som ikke er relevante, legg til strengere regler og begrunn avvik risikobasert.
4. **Pilotér:** Kontroller funksjon, ytelse, e-postflyt, overvåking, sikkerhetskopiering og [Disaster Recovery](/kb/backup-dr) i et representativt miljø.
5. **Rull ut deklarativt:** Bruk GPO, Configuration Management, image, Policy-as-Code eller produsent-API i stedet for manuelle enkeltendringer.
6. **Kontroller kontinuerlig:** Oppdag drift, nye kontoer, lyttere, pakker, sertifikater, regler og endringer i basislinjen.
7. **Ta ut av drift:** Fjern tilgang, DNS, sertifikater, nøkler, data, sikkerhetskopier og overvåking på en kontrollert måte.

Microsofts Security Compliance Toolkit kan lagre, analysere, sammenligne, redigere og bruke anbefalte Windows-basislinjer som GPO. Det erstatter ikke tilpasning: En basislinje testes først i en pilotgruppe for funksjon og bivirkninger. Det samme gjelder CIS- og BSI-anbefalinger ([Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

## Minste funksjonalitet: tjenester, porter og programvare

NIST-kontroll CM-7 krever at et system konfigureres til forretningsmessig nødvendige funksjoner og at funksjoner, porter, protokoller, programvare eller tjenester forbys eller begrenses. Det tekniske spørsmålet er ikke «Er port 443 sikker?», men: **Hvilken prosess lytter på hvilken adresse, for hvilken rolle, fra hvilken sone og med hvilken patch- og identitetsmodell?** ([NIST SP 800-53 Rev. 5, CM-7](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)).

Unødvendige webgrensesnitt, feilsøkingsendepunkter, oppdagelsesprotokoller, lokale databaselyttere og eldre administrasjonstjenester deaktiveres. Nødvendige tjenester bindes så langt som mulig bare til de tiltenkte grensesnittene. En MTA kan lytte offentlig på SMTP, men ikke databasen sin. En admin-API-port kan være nødvendig, men hører ikke automatisk hjemme på Internett. CISA anbefaler å slå av unødvendige eller ukrypterte tjenester som Telnet, FTP, TFTP, HTTP og eldre SNMP-varianter for kommunikasjonsinfrastruktur, og å inventarisere offentlig tilgjengelige tjenester fortløpende ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Inventariser lyttere og aktive tjenester

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Dienstinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Service | Where-Object Status -eq Running |
  Sort-Object Name
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
systemctl list-units --type=service --state=running
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) og [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) viser lokale lyttere og prosesser. [`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service) og [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) viser aktive tjenester. Sammenligningen mellom ønsket og faktisk tilstand krever deretter en godkjent port- og tjenestematrise; en ukjent lytter er et funn, men ennå ikke en årsaksanalyse.

Etter at unødvendige funksjoner er fjernet, gjenstår kontoene og tjenestene som faktisk kan handle. Deres rettigheter, påloggingsveier og hemmeligheter bestemmer mesteparten av den administrative angrepsflaten.

## Identiteter, kontoer og minste privilegium

Kontoer skilles etter rolle: vanlig brukeridentitet, personlig administratoridentitet, ikke-interaktiv tjenestekonto og strengt kontrollert nødkonto. Delte administratorinnlogginger hindrer pålitelig attribusjon. Vedvarende høyprivilegerte hverdagskontoer øker skadeomfanget ved phishing, nettleser- og klientkompromitteringer. CIS Control 5 omfatter bruker-, administrator- og tjenestekontoer; CIS Control 6 tildeling, vedlikehold og tilbakekalling av deres legitimasjon og privilegier ([CIS Control 5: Account Management](https://www.cisecurity.org/controls/account-management), [CIS Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)).

Sentral identitet forbedrer Joiner/Mover/Leaver-prosesser, men erstatter ikke en lokal nødvei. Et LDAP-, Kerberos- eller SSO-utfall må ikke gjøre autorisert gjenopprettingstilgang umulig. Break-Glass-kontoer er derfor et bevisst lite unntak: dokumentert offline, sterkt beskyttet, ikke brukt til daglig, med umiddelbar alarm ved bruk og regelmessig testing. Tjenesteidentiteter får ingen interaktiv innlogging og bare rettighetene, nettverksmålene og hemmelighetene oppgaven krever. Der det er mulig, foretrekkes kortlivede tokener, Managed Identities eller sertifikater fremfor statiske passord; deres livssyklus og gjenoppretting forblir en del av driften.

MFA reduserer risikoen for stjålne passord, men erstatter ikke minimale rettigheter og sikker gjenoppretting. NIST Zero Trust krever en tilgangsbeslutning for subjektet og eventuelt enheten før økten; nettverksplassering eller eierskap alene er ikke tilstrekkelig ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)).

### Kontroller lokale og privilegerte kontoer

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokales Konteninventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember -Group Administrators
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
getent passwd
getent group sudo wheel
```

  </div>
</div>

[`Get-LocalUser`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser) og [`Get-LocalGroupMember`](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember) leser lokale Windows-kontoer og gruppemedlemskap. [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) spør de konfigurerte navnetjenestedatabasene og kan derfor vise både lokale og sentralt oppløste kontoer. En gjennomgang må i tillegg registrere faktiske roller i produktet, API-tokener, SSH-nøkler, sertifikater og sky-IAM.

## Administrasjonsplan og administrasjonsveier

Administrasjonsplanet kan endre konfigurasjon, nøkler, ruting, oppdateringer og logger og fortjener en strengere grense enn nyttedataveien. CISA anbefaler et Out-of-Band-administrasjonsnettverk som er fysisk eller logisk atskilt fra den operative datastrømmen, Default-Deny-regler, dedikerte administratorarbeidsstasjoner og sentral AAA-logging. Laterale administrasjonsforbindelser mellom enheter bør også begrenses ([CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

For meldingssystemer betyr dette:

- Admin-GUI, API, SSH og remoting er bare tilgjengelige fra definerte administrasjonssoner eller via en kontrollert bastionvei.
- Administrasjons- og e-postflytsertifikater, kontoer og brannmurregler administreres separat.
- Utgående forbindelser fra administrasjonsplanet begrenses til mål for oppdatering, identitet, tid, logging og sikkerhetskopiering.
- Konfigurasjonsendringer krever personlig identitet, helst MFA, revisjon og ved høy risiko godkjenning av to personer.
- En nødvei fungerer uten den vanlige identitets- eller administrasjonsplattformen, men drives ikke som skjult permanent tilgang.

SSH er bare en transport for administrasjon; sikkerheten avhenger av autentisering, tillatte brukergrupper, nøkkelalgoritmer, videresending, filrettigheter og målrettigheter. Kontroll av effektiv serverkonfigurasjon i stedet for bare tekstfilen avdekker inkluderingsfiler og standardinnstillinger.

### Vis effektiv SSH-serverkonfigurasjon

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für effektive OpenSSH-Konfiguration">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
& "$env:WINDIR\System32\OpenSSH\sshd.exe" -T
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sshd -T
```

  </div>
</div>

[`Get-WindowsCapability`](https://learn.microsoft.com/powershell/module/dism/get-windowscapability) viser den installerte OpenSSH-komponenten; Microsoft dokumenterer stier og særegenheter ved [`sshd_config`](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration). [`sshd -T`](https://man.openbsd.org/sshd) viser den gjeldende konfigurasjonen. Alternativer må ikke settes blindt etter sjekklister fra Internett: tilgjengeligheten til nødtilgang, brukte nøkkeltyper og automatisering må inngå i testen.

## Nettverksveier: Default Deny med eksplisitt retning

En brannmurregel dokumenteres som en rettet kontrakt: **kilde, mål, protokoll/port, initiativtaker, identitet, bruksformål, eier og utløpsdato**. «E-postserver kan gå til Internett» er ikke en teknisk spesifikasjon. En innkommende SMTP-lytter trenger andre utgående mål enn en malware-sandkassearbeider eller admin-API-et. Filtrering av utgående trafikk begrenser Command-and-Control, eksfiltrering og ukontrollert nedlasting; DNS, tid, sertifikatkontroll, oppdateringer og leveringsmål må tas bevisst hensyn til.

Segmentering reduserer bevegelsesrommet etter en kompromittering. Den er særlig viktig mellom Internett-edge, e-postbehandling, postboks-/datalager, katalog, administrasjon, sikkerhetskopiering og overvåking. NIST Zero Trust advarer samtidig mot å bruke nettverksposisjon som eneste tillitsgrunnlag. CISA anbefaler Default Deny for administrasjonsveier og en sone atskilt fra kundedatatrafikk ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [CISA: Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)).

### Kontroller vertens brannmur og regelretning

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Firewall-Regeln">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
Get-NetFirewallRule -Enabled True |
  Select-Object DisplayName, Direction, Action, Profile
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nft list ruleset
```

  </div>
</div>

[`Get-NetFirewallProfile`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile) og [`Get-NetFirewallRule`](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule) viser Windows-profiler og aktive regler. [`nft`](https://netfilter.org/projects/nftables/manpage.html) viser nftables-reglene, inkludert kjeder og retning. Utdataene kontrolleres mot den godkjente dataflytmatrisen; en Default-Deny-policy uten nødvendige DNS-, tids- eller sertifikatmål er ikke en vellykket herdingstilstand.

## E-postprotokoller og transporttillit

Relé, innsending og tilgang krever ulike TLS- og autentiseringsregler. RFC 8314 anbefaler TLS 1.2 eller nyere for innsending og tilgang og foretrekker implisitt TLS; klarteksttilgang bør ikke lenger tilbys. For SMTP-relé beskriver STARTTLS derimot forhandling per hopp. Uten ytterligere policy kan en sendende MTA fortsatt levere i klartekst når TLS mangler. [DANE](/kb/tls) og MTA-STS gir ulike, mer eksplisitte transportpolicyer ([RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207), [RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672), [RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)).

[SPF, DKIM og DMARC](/kb/mail-auth) autentiserer domenetilknytninger og policy, ikke brukerkontoer eller selve innholdet. [S/MIME og OpenPGP](/kb/verschluesselung) beskytter meldingsdeler, men endrer ingenting ved usikker administratoradgang eller kompromittert nøkkellagring. En herdingskontroll holder disse sikkerhetsegenskapene atskilt og kontrollerer avhengighetene deres: DNS, sertifikater, nøkler, tid, rapporter og unntaksregler. NIST SP 800-177 plasserer nettopp disse supplerende mekanismene rundt SMTP og DNS ([NIST SP 800-177 Rev. 1](https://csrc.nist.gov/pubs/sp/800/177/r1/final)).

### Kontroller tilgjengelighet og TLS-atferd

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Mail- und Management-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection mx.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://mx.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz mx.example.ch 25
openssl s_client -starttls smtp -connect mx.example.ch:25 \
  -servername mx.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer TCP-veien. [`curl`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser STARTTLS, sertifikatkjede og feil. Først MTA-policy og logger svarer på om en feil fører til forsinkelse, avvisning eller tilbakefall til klartekst.

## Applikasjon, data, parsere og nøkler

E-postservere behandler med hensikt komplekse formater som ikke er til å stole på. MIME-meldinger, arkiver, dokumenter, bilder og HTML når parsere, skannere, konvertere, forhåndsvisninger og sandkasser. Herding begrenser derfor ikke bare nettverksporter, men også prosessrettigheter, filsystemtilgang, midlertidig lagring, CPU/RAM/filstørrelse, rekursjon, kjørbarhet og utgående trafikk fra analysekomponentene. En skanner trenger tilgang til et kontrollobjekt, men ikke automatisk til alle postbokser, administratorhemmeligheter eller administrasjons-API-et.

Køer og midlertidige kataloger inneholder konfidensielt innhold. Filrettigheter, kryptering, slette- og oppbevaringsregler samt feilsøkingsdump kontrolleres uttrykkelig. Logger skal gjøre tilstander og beslutninger sporbare, men ikke registrere passord, tokener, private nøkler eller unødvendig meldingsinnhold. Nøkler skilles etter formål: TLS, DKIM, S/MIME/OpenPGP, JWT/API og sikkerhetskopieringskryptering har ulike livssykluser, rettigheter og gjenopprettingsregler. NIST SP 800-53 knytter minste privilegium, systemintegritet, kommunikasjonsbeskyttelse og revisjon sammen; BSI APP.5.3 konkretiserer beskyttelsesbehovet for e-postklient og -server ([NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), [BSI APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)).

## Patcher, images og forsyningskjede

Patchadministrasjon er forebyggende vedlikehold, ikke en sporadisk nødsituasjon. NIST definerer prosessen som identifikasjon, prioritering, innhenting, installasjon og verifisering av patcher, oppdateringer og oppgraderinger. For eksponerte e-post- og administrasjonskomponenter må sårbarhetsinformasjon, tilgjengelig angrepsflate, aktiv utnyttelse, datakritikalitet og tilgjengelig kompensasjon påvirke prioriteringen ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Oppdateringsveien er selv en tillitsgrense. Pakker, images, containere, utvidelser, virussignaturer og appliance-fastvare hentes fra autentiserte kilder og kontrolleres med produsentsignatur eller publisert hash. Avhengigheter og repositorieskifter hører til i inventaret. NIST SP 800-161 behandler risikoer ved produkter og tjenester der operatøren bare i begrenset grad kan se eller kontrollere utvikling, integrasjon og levering ([NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

En herdingsoppdatering testes i et representativt trinn: oppstart, e-postflyt, kø, TLS, katalog, policy, overvåking, sikkerhetskopiering og tilbakeføring. «Ikke patch fordi e-post er kritisk» bytter en kjent driftsrisiko mot en voksende sikkerhetsrisiko. Det bedre designet skaper redundans, vedlikeholdsvinduer, reproduserbare bygg og testede tilbakefallsveier.

### Artefaktintegritet og sikkerhetsrelevante hendelser

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefakt- und Ereignisprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-gateway-update.bin -Algorithm SHA256
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime=(Get-Date).AddHours(-4)} |
  Select-Object TimeCreated, Id, ProviderName, Message
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-gateway-update.bin
journalctl --since '-4 hours' --priority=notice..alert
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) sammenligner en artefakt med en forventet hash fra en autentisert produsentkilde. [`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) og [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) leser hendelser; produksjonsdeteksjon krever i tillegg riktig revisjonspolicy, sentral innsamling, tidssynkronisering og definerte alarmer.

En herdet konfigurasjon forblir bare effektiv når endringer, mislykkede kontroller og avvik blir synlige. Derfor hører logging og driftdeteksjon til driften, og ikke først til etterkontrollen.

## Logging, telemetri og drift

En herdet tilstand er ikke varig uten observasjon. Relevante signaler omfatter blant annet:

- vellykkede og mislykkede innlogginger, MFA- og Break-Glass-bruk;
- endringer i kontoer, roller, tokener, sertifikater og nøkler;
- konfigurasjonsendringer og avvik fra basislinjen;
- nye lyttere, tjenester, pakker, oppgaver, containere eller utgående mål;
- brannmuravvisninger, uventede utgående forbindelser og administrasjonstilganger;
- feil i TLS, DNS, SMTP-policy, kø og e-postautentisering;
- deaktiverte sensorer, logghull, lagringsmangel og tidsavvik.

Logger samles sentralt og tilgangsbeskyttet slik at en kompromittert vert ikke enkelt kan fjerne sporene sine sammen med systemtilstanden. CIS Control 8 krever en loggadministrasjonsprosess, tilstrekkelig lagring, standardisert tid, detaljerte og sentraliserte revisjonslogger samt gjennomganger. En alarm regnes først som implementert når en kontrollert hendelse utløser den, ansvarlig instans ser den og en runbook leder til respons ([CIS Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)).

Driftdeteksjon sammenligner faktisk tilstand med den versjonerte basislinjen. Sammenligningen omfatter mer enn filhasher: effektiv konfigurasjon, kontoer, grupper, IAM-roller, sertifikater, brannmurregler, lyttere, tjenester, installerte pakker, images, planlagte jobber og leverandørpolicy. Nødendringer følges opp eller tilbakestilles automatisk; ellers blir den «midlertidige» unntakstilstanden den nye, udokumenterte standarden.

## Teknisk utvikling

Saltzer og Schroeder formulerte i 1975 grunnleggende beskyttelsesprinsipper som små og enkle mekanismer, sikre standardinnstillinger, fullstendig autorisasjonskontroll, separasjon av privilegier og minste privilegium. Utgangspunktet deres var ikke et bestemt operativsystem, men arkitekturen for kontrollert informasjonsdeling i flerbrukersystemer ([Saltzer/Schroeder: Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)).

Med utbredte nettverksservere ble herding i tillegg flyttet mot fjerntjenester, protokoller, patching, revisjon og sikker konfigurasjonsforvaltning. NIST SP 800-123 oppsummerte denne serverpraksisen systematisk i 2008. Konsensusbaserte CIS Benchmarks, BSI-Grundschutz-moduler og produsentbasislinjer gjorde sikre ønskekonfigurasjoner mer reproduserbare og sammenlignbare ([NIST SP 800-123](https://csrc.nist.gov/pubs/sp/800/123/final), [CIS Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)).

Sky-, SaaS-, API- og hybridarkitekturer svekket senere antakelsen om en klar intern perimeter. NIST SP 800-207 beskrev Zero Trust i 2020 som en ressursorientert arkitektur uten implisitt tillit basert på nettverksplassering eller eierskap. Parallelt ble programvareforsyningskjede, image-opprinnelse og automatisert basislinjedrift egne kontrollflater. Moderne herding kobler derfor klassisk vertminimering med identitet, tjeneste-til-tjeneste-policy, deklarativ konfigurasjon, bevis for forsyningskjeden, telemetri og testet gjenoppretting ([NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final), [NIST SP 800-161 Rev. 1](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)).

## Administratorsjekkliste

Herding er først fullført når de valgte tiltakene kan verifiseres i normal drift og ved gjenoppretting. Sjekklisten kobler derfor konfigurasjon, ansvar og dokumentasjon.

- [ ] Systemets rolle, beskyttelsesbehov, datastrømmer og tillitsgrenser er dokumentert.
- [ ] Produsent-, CIS- og BSI-anbefalinger er kartlagt mot en versjonert basislinje.
- [ ] Hvert avvik har begrunnelse, kompenserende kontroll, eier og utløpsdato.
- [ ] Lyttere, tjenester, pakker, moduler og utgående mål er redusert til nødvendig minimum.
- [ ] Bruker-, administrator-, tjeneste- og Break-Glass-identiteter er atskilt og gjennomgås regelmessig.
- [ ] Administratoradganger bruker personlig identitet, MFA, minimale rettigheter og sentral revisjon.
- [ ] Administrasjonsplanet og produktiv e-postflyt ligger i separate, restriktive soner.
- [ ] Relé, innsending, tilgang og interne tjenesteveier har egne TLS-, autentiserings- og hastighetsgrensepolicyer.
- [ ] Parsere, skannere, midlertidige data, køer, nøkler og hemmeligheter har minimale prosess- og filrettigheter.
- [ ] Patcher og images kommer fra autentiserte kilder; opprinnelse og integritet verifiseres.
- [ ] Tester av basislinje, e-postflyt, sikkerhetskopiering og tilbakeføring utføres før bred utrulling.
- [ ] Logger er sentrale, tidsmessig konsistente, beskyttet mot endring og koblet til testede alarmer.
- [ ] Drift i kontoer, konfigurasjon, regler, tjenester, sertifikater og programvare oppdages automatisk.
- [ ] Gjenoppretting og nødtilgang er praktisk testet under de herdede betingelsene.

## Kilder

- [NIST – SP 800-123, Guide to General Server Security](https://csrc.nist.gov/pubs/sp/800/123/final)
- [CIS – Control 4: Secure Configuration](https://www.cisecurity.org/controls/secure-configuration-of-enterprise-assets-and-software)
- [CIS – Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [BSI – SYS.1.1 Allgemeiner Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2022/07_SYS_IT_Systeme/SYS_1_1_Allgemeiner_Server_Edition_2022.pdf?__blob=publicationFile&v=3)
- [BSI – APP.5.3 Allgemeiner E-Mail-Client und -Server](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/06_APP_Anwendungen/APP_5_3_Allgemeiner_E-Mail_Client_und_Server_Edition_2023.pdf?__blob=publicationFile&v=3)
- [NIST – SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST – SP 800-53 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)
- [NIST – SP 800-207, Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
- [NIST – SP 800-177 Rev. 1, Trustworthy Email](https://csrc.nist.gov/pubs/sp/800/177/r1/final)
- [IETF RFC 8314 – TLS for Email Submission and Access](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [CIS – Benchmarks FAQ](https://www.cisecurity.org/cis-benchmarks/cis-benchmarks-faq)
- [CISA – Enhanced Visibility and Hardening Guidance](https://www.cisa.gov/resources-tools/resources/enhanced-visibility-and-hardening-guidance-communications-infrastructure)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [CIS – Control 5: Account Management](https://www.cisecurity.org/controls/account-management)
- [CIS – Control 6: Access Control Management](https://www.cisecurity.org/controls/access-control-management)
- [Microsoft Learn – Get-LocalUser](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localuser)
- [Microsoft Learn – Get-LocalGroupMember](https://learn.microsoft.com/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember)
- [Linux man-pages – getent](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft Learn – Get-WindowsCapability](https://learn.microsoft.com/powershell/module/dism/get-windowscapability)
- [Microsoft Learn – OpenSSH Server Configuration](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration)
- [OpenBSD – sshd manpage](https://man.openbsd.org/sshd)
- [Microsoft Learn – Get-NetFirewallProfile](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallprofile)
- [Microsoft Learn – Get-NetFirewallRule](https://learn.microsoft.com/powershell/module/netsecurity/get-netfirewallrule)
- [Netfilter – nft manpage](https://netfilter.org/projects/nftables/manpage.html)
- [IETF RFC 3207 – SMTP STARTTLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [IETF RFC 7672 – SMTP Security via DANE](https://datatracker.ietf.org/doc/html/rfc7672)
- [IETF RFC 8461 – SMTP MTA Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc8461)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [NIST – SP 800-161 Rev. 1, Cybersecurity Supply Chain Risk Management](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [CIS – Control 8: Audit Log Management](https://www.cisecurity.org/controls/audit-log-management)
- [Saltzer/Schroeder – Basic Principles of Information Protection](https://web.mit.edu/Saltzer/www/publications/protection/Basic.html)
