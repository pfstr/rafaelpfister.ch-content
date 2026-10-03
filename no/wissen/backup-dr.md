---
title: "Sikkerhetskopiering og disaster recovery: Tilstander, mål og gjenoppretting"
blatt: "backup-dr"
description: "Sikkerhetskopiering og disaster recovery for meldingsadministratorer: avgrensning mot snapshot, replikering og høy tilgjengelighet, RPO, RTO og MTD, konsistent sikring av postbokser, køer, konfigurasjon, nøkler og indekser, isolert gjenoppretting, oppstartsrekkefølge, gjenopprettingstester og diagnostikk."
fakten:
  - label: Mål
    wert: Tilbakeføre tjenester og data til en kjent, pålitelig tilstand
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Planleggingsgrunnlag
    wert: Business Impact Analysis og avhengighetsinventar
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final
  - label: MTD
    wert: maksimalt tolererbar avbruddstid for forretningsprosessen
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RTO
    wert: maksimal utilgjengelighet for en ressurs før virkningen blir uakseptabel
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: RPO
    wert: tidspunktet data må være gjenopprettet til etter hendelsen
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Sikkerhetskopi
    wert: tidsstemplet, gjenopprettbar kopi; supplerer replikering
    href: https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup
  - label: Konsistens
    wert: Applikasjon, backup-programvare og lagring må koordinere sikringspunktet
    href: https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service
  - label: Datanivåer
    wert: Postboks/blob · metadata · kø · indeks · konfigurasjon · nøkler
    href: https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf
  - label: Cyber recovery
    wert: isolert kopi og gjenoppretting til en pålitelig tilstand
    href: https://cas8.docs.cisecurity.org/en/latest/source/Controls11/
  - label: Uforanderlighet
    wert: Retensjon hindrer sletting eller overskriving av bestemte objektversjoner
    href: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
  - label: Nøkkelregel
    wert: Planlegg gjenoppretting etter nøkkeltype og bruksformål
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
  - label: Akseptkriterium
    wert: vellykket, rettidig gjenoppretting av en forretningstjeneste
    href: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf
werbung:
  - tools
  - newsletter
ctaThemen:
  - backup
  - disaster-recovery
  - messaging
translationSourceHash: dec2b9d3915258979e801cb9d3ea573f054ae89136cd0631af06b681f36c3ca9
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:40:41.185Z
translationReview: required
---

# Sikkerhetskopiering og disaster recovery: Tilstander, mål og gjenoppretting

En vellykket fullført sikkerhetskopieringsjobb beviser først bare at et verktøy har skrevet data. Den sier ennå ikke om sikkerhetskopieringspunktet er fullstendig, applikasjonskonsistent, beskyttet mot samme feil eller kan gjenopprettes som en brukbar tjeneste innen avtalt tid. **Sikkerhetskopi** betegner den gjenopprettbare kopien av en tidligere tilstand; **disaster recovery** omfatter i tillegg mennesker, prioriteringer, målinfrastruktur, avhengigheter, validering og kontrollert tilbakeføring til drift. Derfor skiller NIST mellom sikkerhetskopiering, gjenopprettingsstrategi, recovery-prosedyrer, testing og løpende planvedlikehold ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)).

For meldingsplattformer er «databasen» ikke et fullstendig beskyttelsesobjekt. En brukbar tilstand kan være fordelt på postboks- eller bloblagring, relasjonelle metadata, transportkøer, søkeindekser, konfigurasjoner, rutingsregler, katalogreferanser, sertifikater, private nøkler, DNS, lisenser og automatisering. Noen deler er autoritative, andre er bare projeksjoner, og andre igjen er flyktige tilstander. Recovery-planen må angi for hver del om den **gjenopprettes, bygges opp på nytt, utstedes på nytt eller bevisst forkastes**. I tillegg til brukerdata krever NIST også systemtilstand, programvare, inventar, lisenser og sikkerhetsrelevant dokumentasjon; PostgreSQL påpeker for eksempel uttrykkelig at WAL-arkivering ikke sikkerhetskopierer konfigurasjonsfilene ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [PostgreSQL: Continuous Archiving og PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

Det operative akseptkriteriet er derfor ikke «backup montert», men for eksempel: En ekstern avsender kan levere en melding, riktig policy anvendes, meldingen vises i den tiltenkte postboksen, er søkbar og kan besvares, og overvåking og revisjon registrerer hendelsen. NIST nevner vellykkede og rettidige gjenopprettinger, oppnådde recovery-mål og brukere eller systemer som igjen er tilgjengelige, som målbare recovery-resultater ([NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery), [NIST SP 800-184, Recovery Metrics](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)).

Forklaringen begynner med forretningsprosessen som må fungere igjen etter et avbrudd. Derfra oppstår RTO og RPO, deretter den passende sikkerhetskopieringskjeden; til slutt står ikke backupjobben, men en målt gjenopprettingstest.

## Sikkerhetskopi er ikke høy tilgjengelighet

Sikkerhetskopi, snapshot, replikering og høy tilgjengelighet løser ulike feilklasser. Microsoft beskriver uttrykkelig sikkerhetskopiering og replikering som komplementære: Replikering holder en oppdatert kopi for løpende drift, men overtar også logiske slettinger eller skader; en tidsstemplet sikkerhetskopi muliggjør tilbakesprang til en eldre tilstand ([Microsoft Azure Reliability: Redundancy, Replication and Backup](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)).

| Mekanisme | Primær nytte | Hva den alene ikke beviser |
|---|---|---|
| Sikkerhetskopi | historisk, gjenopprettbar tilstand med retensjon | kort omkoblingstid eller en umiddelbart kjørbar målplattform |
| Storage-snapshot | rask punkt-i-tid-tilstand for et volum | applikasjonskonsistens, separat feilområde eller langtidsoppbevaring |
| Replikering | oppdatert datastatus på et annet mål | beskyttelse mot replikert sletting, kryptering eller stille korrupsjon |
| Høy tilgjengelighet | tjenestekontinuitet ved definerte komponentfeil | historisk tilbakesprang eller gjenoppbygging etter administrator-kompromittering |
| Arkiv | oppbevaring av utvalgte data over lange perioder | fullstendig rekonstruksjon av tjenesten og dens avhengigheter |
| Disaster recovery | koordinert gjenoppretting etter steds-, plattform- eller sikkerhetshendelser | datagjenoppretting uten egnede, testede sikkerhetskopier |

Et snapshot kan være en byggestein i sikkerhetskopieringen. Det blir imidlertid først en robust recovery-kilde gjennom konsistens, eksport eller replikering til uavhengig drevet lagring, retensjon, katalog og gjenopprettingsprosedyre. Kubernetes Snapshot API garanterer for eksempel ikke selv applikasjonskonsistens; applikasjoner må forberedes egnet før snapshotet. Under Windows overtar VSS denne koordineringen bare når requester, writer og provider samspiller korrekt ([Kubernetes: Volume Snapshot og applikasjonskonsistens](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/), [Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)).

## MTD, RTO og RPO hører til forretnings- og systemfunksjoner

**Maximum Tolerable Downtime (MTD)** er den lengste avbruddstiden som forretningsprosessen totalt kan tolerere. **Recovery Time Objective (RTO)** beskriver hvor lenge en konkret systemressurs kan være ute av drift før andre ressurser, den støttede prosessen eller dens MTD påvirkes uakseptabelt. **Recovery Point Objective (RPO)** betegner tidspunktet før hendelsen som data må gjenopprettes til. RTO må vanligvis være kortere enn MTD, fordi data fortsatt må etterbehandles og tjenesten må kontrolleres faglig etter teknisk gjenoppretting ([NIST SP 800-34 Rev. 1, avsnitt 3.2](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

Én enkelt «mail-RTO» skjuler relevante forskjeller. En plattform kan godta SMTP selv om brukertilgang eller søk ennå ikke er tilgjengelig. En gateway kan bufre meldinger selv om den underliggende postbokstjenesten er nede. Omvendt kan et webgrensesnitt være tilgjengelig mens nøkler, katalogoppslag eller utgående konnektorer mangler. Mål bør derfor fastsettes per **forretningsfunksjon og avhengighet**.

| Funksjon | Tilstand som måles | Typisk RPO-spørsmål | RTO er først slutt når |
|---|---|---|---|
| ekstern mottak | MX, TLS, SMTP-lytter, policy og kø | Hvilke mottatte meldinger kan mangle? | mottaket er kontrollert og købehandling kan dokumenteres |
| utgående levering | ruting, DNS, TLS-policy, retry og DSN | Hvilke køoppføringer kan gå tapt? | levering er vellykket eller forsinkelsen er standardmessig korrekt |
| postbokstilgang | identitet, metadata, blob og protokoll | Hvilken siste postbokstilstand kreves? | innlogging samt lesing, skriving og mappeoperasjoner fungerer |
| søk | indeks og projeksjoner | Må indeksen sikres eller opprettes på nytt? | definert dataomfang igjen kan finnes |
| kryptering | policy, sertifikater, nøkler og tillit | Hvilket eldre innhold må fortsatt kunne dekrypteres? | en definert testmelding kan krypteres og dekrypteres |
| administrasjon | control plane, roller, revisjon og overvåking | Hvilken konfigurasjonsendring kan mangle? | autorisert endring, varsling og revisjon kan spores |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-backup-dr.svg?v=20260813" title="Interaktive Infografik: Backup- und Disaster-Recovery-Kette für Messaging-Plattformen von produktiven Zuständen über konsistente, isolierte Sicherungen bis zur geprüften Wiederherstellung" loading="lazy">
  <a href="/images/kb-interaktiv-backup-dr.svg?v=20260813">Åpne interaktiv grafikk direkte</a>.
</iframe>

Når målene er fastsatt, må plattformens faktiske tilstand inventariseres. En postboksdatabase alene gjenoppretter verken ruting, identiteter, nøkler eller søkeindekser.

## Tilstandsinventar for en meldingsplattform

En sikkerhetskopieringspolicy begynner med et tilstands- og avhengighetsinventar, ikke med produktkatalogen til backup-produsenten. For hver tilstand dokumenteres **autoritativ kilde, konsistensmekanisme, RPO, retensjon, beskyttelsesdomene, gjenopprettingsmetode, rekkefølge og kontrolltrinn**. NISTs Business Impact Analysis identifiserer kritiske prosesser, ressurser og deres recovery-prioritet; NIST SP 800-184 supplerer med realistiske scenarioer og avhengigheter som oppdages under gjenopprettingen ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final), [NIST SP 800-184](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)).

| Tilstand | Karakter | Recovery-beslutning | Faglig kontrolltrinn |
|---|---|---|---|
| Postbokser og meldingsblober | autoritative brukerdata | gjenopprett konsistent eller rekonstruer fra uforanderlig kilde | les en kjent melding med MIME-vedlegg |
| Metadatabase | transaksjoner, tilordninger, ACL-er, UID-er | bruk base backup pluss loggreplay eller applikasjonsspesifikk restore | mapper, rettigheter og meldingsreferanser stemmer |
| Transportkø | flyktig, men forretningsrelevant leveringsstatus | sikkerhetskopier, overta ordnet eller lever på nytt bevisst | ingen stille hull og kontrollert håndtering av duplikater |
| Søkeindeks og projeksjoner | som regel avledet og eventuelt konsistent | sikre bare hvis rebuild bryter RTO; ellers reindekser | definert stikkprøve er fullstendig søkbar |
| Konfigurasjon og policy | deklarativ, eksportert eller databasestøttet | sikre versjonert eksport pluss skjema-/produktversjon | ruting, filter, grenser og leietakerseparasjon virker |
| Identiteter og katalogreferanser | ofte eksternt autoritative | gjenopprett katalogen separat; behold bindinger, ID-er og claims | tjeneste- og brukerinnlogging fungerer |
| Sertifikater, nøkler og secrets | svært sensitive, delvis ikke-eksporterbare | sikre, utsted på nytt eller rekonstruer via HSM/KMS etter nøkkeltype | TLS, signatur, dekryptering og rotasjon er kontrollert |
| DNS, tid, nettverk og lastbalanserer | eksternt kontroll- og navnelag | dokumenter som kode/eksport og hos leverandør | navn, porter, sertifikatnavn og tid stemmer |
| Programvare, images, IaC og lisenser | reproduserbart kjøregrunnlag | hold pålitelige artefakter, versjoner og avhengigheter tilgjengelig | identisk eller godkjent kompatibel build starter |
| Logger, revisjon, backup-katalog og runbooks | bevis og styring | hold tilgjengelig utenfor den berørte administrasjonsdomenen | hendelse, restore-punkt og godkjenninger kan spores |

Kø-recovery er et spesialtilfelle. SMTP krever at akseptert ansvar utføres pålitelig, men tillater ved forbindelsesbrudd situasjoner der avsender og mottaker vurderer fullføringen ulikt. En gjenopprettet køtilstand kan derfor levere meldinger på nytt. Runbooks trenger en definert duplikatstrategi, kø-ID-er, tidsvinduer og mottakerkommunikasjon; bare å kopiere en spool-katalog er ikke en standardmessig restore ([RFC 5321, Queuing and Duplicate Messages](https://datatracker.ietf.org/doc/html/rfc5321)).

## Konsistens oppstår i applikasjonslaget

Et **crash-konsistent** sikkerhetskopieringspunkt inneholder tilstanden et system ville se etter plutselig strømbrudd. Filsystem og enkeltblokker kan være konsistente i seg selv, mens tilhørende databaser, blober og køer representerer ulike tidspunkter. Et **applikasjonskonsistent** sikkerhetskopieringspunkt koordinerer skrivebuffere, transaksjonslogger, kontrollpunkter og eventuelt flere volumer, slik at applikasjonen har en definert recovery-bane.

VSS viser denne arkitekturen eksplisitt: Backup-requesteren ber om sikkerhetskopieringen, den applikasjonsspesifikke writeren stiller et konsistent datasett til rådighet, og provideren oppretter Shadow Copy. Exchange stiller til dette formålet en egen VSS Writer til rådighet; en Exchange-bevisst sikkerhetskopi er derfor mer enn et snapshot av databasefilene ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Microsoft: Windows Server Backup for Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)).

PostgreSQL bruker en annen, men sammenlignbar recovery-modell. En base backup gir utgangspunktet, og en sammenhengende sekvens av arkiverte Write-Ahead-Log-segmenter fører den frem til ønsket tidspunkt. En `pg_dump` er en logisk eksport og ikke en erstatning for base-backup-/WAL-kjeden som kreves for PITR. Konfigurasjonsfiler som `postgresql.conf` og `pg_hba.conf` ligger også utenfor denne WAL-recoveryen og trenger en separat sikkerhetskopieringsvei ([PostgreSQL: Continuous Archiving og PITR](https://www.postgresql.org/docs/17/continuous-archiving.html)).

For distribuerte produkter må produktdokumentasjonen besvare om backender skal sikres uavhengig, som en konsistensgruppe eller via applikasjonens egne eksportfunksjoner. Et samtidig storage-snapshot av flere volumer er ikke automatisk et konsistent snitt gjennom database, objektlager, kø og søkeindeks. Administratoren må kjenne **sannhetskilden** og den tillatte rebuild-banen for hver projeksjon.

### Inventariser kapasitet og sikkerhetskopieringsartefakter

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kapazitäts- und Sicherungsinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-Volume | Sort-Object DriveLetter |
  Select-Object DriveLetter, FileSystemLabel, Size, SizeRemaining
Get-ChildItem \\backup.example.ch\mail -File -Recurse |
  Select-Object FullName, Length, LastWriteTime
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
df -hT
find /backup/mail -type f -printf '%TY-%Tm-%TdT%TH:%TM:%TS %s %p\n'
```

  </div>
</div>

[`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) og [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) viser brukt og ledig kapasitet. [`Get-ChildItem`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem) og [`find`](https://www.gnu.org/software/findutils/find) inventariserer artefakter og tidsstempler. Ingen av delene beviser applikasjonskonsistens eller gjenopprettbarhet; det krever katalog-, logg- og restore-bevis.

## Beskyttelsesarkitektur: adskilt, isolert og kontrollerbar

En robust sikkerhetskopieringskjede har minst fire innbyrdes atskilte roller:

1. **Capture:** Applikasjonen eller eksportfunksjonen oppretter en definert tilstand.
2. **Katalog og manifest:** Backup-ID, kilde, tidspunkt, programvareversjon, nødvendige logger, nøkkelreferanser og kontrollsummer gjør settet mulig å finne og kontrollere.
3. **Recovery-lagring:** Versjonerte kopier ligger utenfor primær feil- og helst også administrasjonsdomene.
4. **Recovery-control plane:** Separate identiteter, runbooks, målinfrastruktur og godkjenninger muliggjør gjenoppretting når produksjonen ikke er pålitelig.

CIS Control 11 krever tilsvarende beskyttede recovery-data, en isolert instans, for eksempel offline, i skyen eller utenfor stedet, samt regelmessige restore-tester. CISA anbefaler for ransomware-scenarioer offline eller på annen måte isolerte, krypterte og regelmessig testede sikkerhetskopier samt rene images og et separat recovery-miljø ([CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

**Uforanderlighet** og **isolasjon** er ikke det samme. S3 Object Lock kan beskytte bestemte objektversjoner i WORM-modellen mot sletting og overskriving under en retensjonsperiode eller Legal Hold. Governance- og Compliance-modus har ulike muligheter for omgåelse. Dette beskytter lagrede versjoner, men beviser verken separate tilgangsdata eller et rent restore-endepunkt, en komplett applikasjonskjede eller tilgjengelige dekrypteringsnøkler ([Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

### Kontroller kontrollsummer og manifest

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Prüfsummen von Sicherungsartefakten">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-FileHash .\mail-backup-2026-08-08.tar.zst -Algorithm SHA256
Get-FileHash .\mail-backup-2026-08-08.manifest.json -Algorithm SHA256
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
sha256sum mail-backup-2026-08-08.tar.zst
sha256sum mail-backup-2026-08-08.manifest.json
```

  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) og [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) oppdager endringer i et artefakt når forventet hash stammer fra en pålitelig kilde. En kontrollsum erstatter ikke autentisering av manifestet og heller ikke en gjenopprettingsprøve. NIST nevner kryptografiske hasher og digitale signaturer som mekanismer for integritetsbeskyttelse av backup-informasjon ([NIST SP 800-34 Rev. 1, CP-9](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)).

## Nøkler og secrets er en egen recovery-plan

«Sikkerhetskopier alle private nøkler» er like feil som «sertifikater kan utstedes på nytt». Bruksformålet er avgjørende:

- En tapt **TLS-servernøkkel** kan som regel erstattes av et nytt nøkkelpar og sertifikat; omkoblingen må likevel ligge innenfor RTO, og tilhørende tillits- eller pinning-avhengigheter må kontrolleres.
- En **dekrypteringsnøkkel** for lagrede S/MIME-, OpenPGP- eller backup-data må være tilgjengelig så lenge den beskyttede chifferteksten skal være lesbar.
- Sikkerhetskopiering av en **privat signaturnøkkel** er ifølge NIST generelt ikke ønskelig, fordi gjenbruk kan påvirke signaturens bevisverdi; begrunnede unntak krever særlig sikker recovery og rask utskifting.
- En **ikke-eksporterbar HSM-/KMS-nøkkel** trenger redundans-, backup- eller reprovisioneringsveien systemet har bestemt. Eksport av et sertifikat uten privat nøkkel er ingen nøkkelsikkerhetskopi.
- **Nøkkelen som krypterer sikkerhetskopien** må ikke bare ligge i den krypterte sikkerhetskopien eller i den kompromitterte produksjonsdomenen.

NIST krever en beslutning etter nøkkeltype, tilhørende metadata, en key-recovery-policy samt konfidensialitets-, integritets-, tilgjengelighets- og revisjonskontroller for recovery-materialet. Går en dekrypteringsnøkkel tapt, kan chifferteksten ikke lenger føres tilbake til klartekst ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final), [NIST SP 800-57 Part 1 Rev. 5, Key Recovery](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)).

### Kontroller tid og navneoppløsning før restore

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeit- und DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
w32tm /query /status
Resolve-DnsName -Type MX example.ch
Resolve-DnsName backup.example.ch
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
timedatectl status
dig +short MX example.ch
dig +short A backup.example.ch
```

  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) og [`timedatectl`](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) kontrollerer tidsgrunnlaget for sertifikater, Kerberos, logger og recovery-punkter. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser om MX-, tjeneste- og repository-navn løses opp i recovery-sonen som planlagt. [DNS](/kb/dns)-kilden og dens endringsfullmakt hører selv hjemme i avhengighetsinventaret.

En konsistent sikkerhetskopi er bare halvparten av planen. Ved restore må identitet, DNS, database, kø, nøkler og applikasjoner komme tilbake i en begrunnet rekkefølge.

## Gjenoppretting følger avhengighetsgrafen

En fast produktrekkefølge ville være oppdiktet. Den robuste rekkefølgen oppstår fra BIA, ressursinventar og faktiske avhengigheter. NIST krever en prioritert liste over systemressurser og realistiske testscenarioer; NIST SP 800-184 krever at avhengigheter som oppdages under restore, føres tilbake til dokumentasjonen ([NIST SP 800-34 Rev. 1, Recovery Priorities](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [NIST SP 800-184, Recovery Execution](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). For en typisk meldingsplattform resulterer dette ofte i følgende kjede, som må valideres lokalt:

1. **Avgrens hendelsen:** Skill mellom avbrudd og kompromittering, bevar bevis, fastsett kjent rent recovery-punkt og godkjenning.
2. **Etabler recovery-control plane:** Klargjør separate administratoridentiteter, MFA, runbooks, backup-katalog og dekrypteringstilgang.
3. **Valider grunntjenester:** Kontroller nettverk, ruting, [DNS](/kb/dns), tid, [LDAP](/kb/ldap) eller [Kerberos](/kb/kerberos), PKI/KMS og lastbalanserer i målsonen.
4. **Gjenopprett persistens:** Bygg opp objekt-/postbokslagring, databaser og nødvendige transaksjonslogger i et konsistent snitt.
5. **Start applikasjon og policy:** Legg inn godkjente images, konfigurasjon, secrets, konnektorer og roller; ikke tillat ukontrollert ekstern mailflyt ennå.
6. **Aktiver kø og ruting kontrollert:** Vurder alder, mottakere, retry-status og mulige duplikater; frigjør inn- og utgående trafikk separat.
7. **Bygg projeksjoner på nytt:** Opprett søkeindekser, cacher og rapportering fra autoritative kilder, og overvåk rebuild-forsinkelse.
8. **Godkjenn forretningstransaksjonen:** Test levering, postbokstilgang, søk, [TLS](/kb/tls), kryptering, overvåking og revisjon mot definerte kriterier.

Ved en sikkerhetshendelse er «systemet starter» uttrykkelig ikke nok. NIST beskriver reconstitution til en kjent sikker tilstand med sikre parametere, patcher, konfigurasjon, pålitelig programvare, kjent ren sikkerhetskopi og fullstendig test. CIS formulerer samme mål som gjenoppretting til en «pre-incident and trusted state» ([NIST SP 800-34 Rev. 1, CP-10](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf), [CIS Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)).

### Nå recovery-endepunkter og TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Recovery-Endpunkt- und TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection backup.example.ch -Port 443 -InformationLevel Detailed
curl.exe --verbose https://backup.example.ch/health
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz backup.example.ch 443
openssl s_client -connect backup.example.ch:443 \
  -servername backup.example.ch -verify_return_error
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) og [`nc`](https://man.openbsd.org/nc) kontrollerer TCP-banen. [`curl`](https://curl.se/docs/manpage.html) og [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) viser henholdsvis HTTP- og TLS-atferd. Et tilgjengelig health-endepunkt beviser bare control plane, ikke lesbarheten til alle backup-sett.

Gjenopprettingen er først fullført når en bruker eller motpart faktisk kan benytte tjenesten. Et vellykket lest backupmedium er ikke tilstrekkelig bevis på dette.

## Restore-tester måler tjenesten, ikke mediet

En fullstendig test kjøres i et isolert målmiljø med dokumentert utgangspunkt, tidsmåling og akseptkriterier. Den kontrollerer minst:

- om katalog, credentials, dekrypteringsnøkler og artefakter er tilgjengelige uten produksjonen;
- om et kompatibelt målsystem kan klargjøres fra pålitelige images;
- om base backup, transaksjonslogger, bloblagring og konfigurasjon gir samme faglige tilstand;
- om køer behandles kontrollert og duplikater oppdages;
- om identitet, DNS, TLS, mailflyt, postbokstilgang, søk og overvåking fungerer;
- om målt datatap og målt gjenopprettingstid overholder RPO og RTO;
- om tjenesten kan godkjennes som pålitelig etter sikkerhetshendelser.

CIS Control 11.5 evaluerer et utvalg av gjenopprettede sikkerhetskopier som deretter faktisk fungerer. NIST SP 800-184 måler vellykkede og rettidige gjenopprettinger og krever realistiske scenarioer, ettergjennomgang og planforbedring ([CIS Control 11: Test Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/), [NIST SP 800-184](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)). Testfrekvensen følger risiko, endringshastighet og krav; en årlig fulltest kan suppleres med hyppigere automatiserte stikkprøver og komponentrelaterte restores, men må ikke erstattes av vellykket jobbstatistikk.

### Lyttere og lagringstilstand etter restore

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Listener- und Speicherprüfung nach Restore">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
Get-Volume | Select-Object DriveLetter, FileSystemLabel, SizeRemaining
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -lntup
df -hT
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) og [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) viser lokale lyttere og tilhørende prosesser. [`Get-Volume`](https://learn.microsoft.com/powershell/module/storage/get-volume) og [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) viser ledig lagringsplass. Kontrollen må deretter fortsette på protokollnivå: En lytter på port 25 er ennå ikke en fungerende [SMTP](/kb/smtp)-transaksjon.

## Feilbilder og passende recovery-scope

Et avbrudd avgjør hvor langt en gjenoppretting må rekke. Tabellen knytter derfor den observerte hendelsen til minste meningsfulle recovery-omfang og feilantakelsen som oppstår særlig ofte.

| Hendelse | Primær risiko | Egnet recovery-scope | Vanlig feilantakelse |
|---|---|---|---|
| utilsiktet sletting | liten logisk skade | gjenopprett objekt, postboks, policy eller punkt-i-tid målrettet | rull tilbake hele plattformen og mist nyere korrekte data |
| enkelt node eller datadisk | lokal infrastrukturfeil | HA-failover, replika eller komponentrelatert restore | forveksle failover med historisk backup |
| tap av sted eller leverandør | felles fysisk eller administrativt feilområde | alternativ sone/region/site pluss eksterne kopier og DNS-/nettverksomkobling | betrakte datakopi uten tilgjengelig målkapasitet som DR |
| ransomware eller administrator-kompromittering | data, identiteter, programvare og sikkerhetskopier er ikke pålitelige | isolert recovery-control plane, ren build, kjent restore-punkt | fortsette å bruke kompromittert identitet til å låse opp alle sikkerhetskopier |
| nøkkeltap | chiffertekst permanent uleselig eller identitet kan ikke brukes | nøkkeltypespesifikk recovery, reissue eller HSM/KMS-prosedyrer | forveksle offentlig sertifikat med privat nøkkel |
| feilaktig konfigurasjonsendring | korrekte data, feil atferd | reverser versjonert konfigurasjon og valider målrettet | velge database- eller postboksrestore som første tiltak |

Ved en kompromittering trenger ikke det kjente rene tidspunktet å være det nyeste sikkerhetskopieringspunktet. Nyere sikkerhetskopier kan inneholde angriperens tilstand; eldre kan medføre kjente sårbarheter eller inkompatible programvareversjoner. Recovery forbinder derfor forensikk, patchnivå, konfigurasjonsbaseline, nøkkelrotasjon og faglig datagjenoppretting. CISA anbefaler blant annet rene «golden images», offline oppbevarte infrastrukturdefinisjoner og en recovery-nettsone, slik at systemer ikke infiseres på nytt under gjenoppbyggingen ([CISA: StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)).

## Teknisk utvikling

Magnetbånd ble introdusert tidlig på 1950-tallet som et raskt datalagringsmedium for datamaskiner og er fortsatt et backup-medium på grunn av kostnad, kapasitet og fysisk adskillelse ([IBM: Magnetic Tape](https://www.ibm.com/history/magnetic-tape)). Senere sikkerhetskopieringsarkitekturer skilte i økende grad det logiske sikkerhetskopieringspunktet fra målmediet: Databaser kombinerte base backups med transaksjonslogger og Point-in-Time Recovery; lagringssystemer muliggjorde raske snapshots; deduplisering og objektlagring endret overføring og retensjon.

For Windows-applikasjoner i drift innførte Microsoft VSS en koordinert modell med requester, writer og provider; teknologien kom med Windows XP og Windows Server 2003. I distribuerte og containeriserte plattformer ble snapshot- og orkestrerings-API-er standardisert uten at dette automatisk løste applikasjonskonsistens ([Microsoft: Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service), [Kubernetes: Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)).

Cyber recovery flyttet igjen fokus. Versjonering og offsite-kopier er ikke nok hvis høyt privilegerte identiteter kan slette alle mål eller kompromitterte images kommer tilbake. Isolerte recovery-instanser, separate identiteter, uforanderlige objektversjoner, deklarativ infrastruktur og rene gjenopprettingssoner supplerer klassiske fullstendige, inkrementelle og loggbaserte sikkerhetskopier. Amazon S3 Object Lock ble introdusert i 2018 som WORM-beskyttelse for objektversjoner; funksjonen illustrerer denne overgangen, men erstatter fortsatt verken applikasjonskonsistens eller restore-tester ([AWS: Introduksjon av S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/), [Amazon S3: Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).

## Administrator-sjekkliste

Planleggingen er først robust når mål, kopier, tilganger og tester er dokumentert samlet. Sjekklisten oppsummerer disse avhengighetene for gjennomgang og restore-øvelse.

- [ ] Forretningsprosesser, MTD samt RTO og RPO per systemfunksjon er godkjent.
- [ ] Alle autoritative og avledede tilstander på meldingsplattformen er inventarisert.
- [ ] Applikasjonskonsistens, loggkjede og konsistensgrupper er dokumentert produktspesifikt.
- [ ] Kø-restore, mulige duplikater og gjenåpning av inn- og utgående trafikk er regulert.
- [ ] Konfigurasjon, policyer, DNS, sertifikater, nøkler, secrets, lisenser og runbooks er innenfor scope.
- [ ] Minst én recovery-kopi er adskilt fra produksjon og primære administrasjonskontoer.
- [ ] Uforanderlighet, isolasjon, kryptering og nøkkeltilgang er vurdert separat.
- [ ] Recovery-control plane og målkapasitet fungerer uten den kompromitterte produksjonen.
- [ ] Gjenopprettingsrekkefølgen følger en vedlikeholdt avhengighetsgraf.
- [ ] Tester gjenoppretter en fullstendig forretningsprosess for e-post og måler RPO/RTO.
- [ ] Resultater, nye avhengigheter og avvik føres tilbake til runbook og arkitektur.

## Kilder

- [NIST – SP 800-34 Rev. 1, Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [NIST – SP 800-34 Rev. 1, PDF](https://nvlpubs.nist.gov/nistpubs/legacy/sp/nistspecialpublication800-34r1.pdf)
- [PostgreSQL – Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/17/continuous-archiving.html)
- [NIST – SP 800-184, Guide for Cybersecurity Event Recovery](https://www.nist.gov/publications/guide-cybersecurity-event-recovery)
- [NIST – SP 800-184, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-184.pdf)
- [Microsoft Azure Reliability – Redundancy, Replication and Backup](https://learn.microsoft.com/en-us/azure/reliability/concept-redundancy-replication-backup)
- [Kubernetes – Volume Snapshot and Application Consistency](https://kubernetes.io/blog/2020/12/10/kubernetes-1.20-volume-snapshot-moves-to-ga/)
- [Microsoft Learn – Volume Shadow Copy Service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Microsoft Learn – Windows Server Backup for Exchange](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/windows-server-backup)
- [Microsoft Learn – Get-Volume](https://learn.microsoft.com/powershell/module/storage/get-volume)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [Microsoft Learn – Get-ChildItem](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-childitem)
- [GNU Findutils – find](https://www.gnu.org/software/findutils/find)
- [CIS – Control 11: Data Recovery](https://cas8.docs.cisecurity.org/en/latest/source/Controls11/)
- [CISA – StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide)
- [Amazon S3 – Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – sha256sum](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [NIST – SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST – SP 800-57 Part 1 Rev. 5, PDF](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf)
- [Microsoft Learn – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [systemd – timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [IBM – Magnetic Tape](https://www.ibm.com/history/magnetic-tape)
- [Kubernetes – Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [AWS – Introduksjon av S3 Object Lock](https://aws.amazon.com/about-aws/whats-new/2018/11/s3-object-lock/)
