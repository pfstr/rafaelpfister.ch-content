---
title: "Exchange On-Premises: Serverarkitektur og drift"
blatt: "exchange-on-premises"
description: "Exchange Server i eget datasenter: postboks- og Edge-roller, transportpipeline, Active Directory, ESE-databaser, DAG, klienttilgang, sikkerhet, overvåking, sikkerhetskopiering og gjenoppretting."
fakten:
  - label: Produktrolle
    wert: Selvdrevet meldings- og gruppevareplattform
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Serverroller
    wert: Postboks og valgfri Edge Transport
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Operativsystem
    wert: Windows Server i henhold til Exchange-systemkravene
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Katalog
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Postbokslagring
    wert: ESE-database, transaksjonslogger og kontrollpunkt
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: Høy tilgjengelighet
    wert: Database Availability Group og databasekopier
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Transport
    wert: Frontend Transport, Transport Service og Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Transportrobusthet
    wert: Shadow Redundancy og Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Klienttilgang
    wert: HTTPS, MAPI/HTTP, Outlook på nettet, EWS og ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Administrasjon
    wert: Exchange Admin Center og Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Overvåking
    wert: Managed Availability, Health Sets, Event Logs og ytelsesindikatorer
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Gjenoppretting
    wert: Server Recovery, databasegjenoppretting og Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 8d30441ccb794fc2e8228dfe1fa084ee38a525dcb5e59222e182819a74596bbc
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:26:33.323Z
translationReview: automatic
---

# Exchange On-Premises: Serverarkitektur og drift

**Exchange On-Premises** betyr at organisasjonen drifter Exchange Server i sin egen infrastruktur. Den kontrollerer Windows-verter, Active Directory, sertifikater, transporttjenester, køer, postboksdatabaser og gjenoppretting. Microsoft leverer produktkode, dokumentasjon og oppdateringer; tilgjengelighet og sikkert vedlikehold forblir operatørens ansvar ([Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server), [Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Den praktiske forskjellen fra Exchange Online viser seg umiddelbart ved en feil. En On-Prem-administrator kan undersøke en transportkø på en bestemt server, kontrollere statusen til en databasekopi og kontrollert bytte til en annen kopi. Til gjengjeld må vedkommende også forstå hvordan SMTP, Active Directory, ESE, Windows Failover Clustering, IIS og Exchange-tjenester samspiller.

## Postboksserveren er den sentrale byggesteinen

Moderne Exchange-servere bruker **postboksserveren** som en felles byggestein. Den inneholder Client Access-tjenester som mottar og videresender forbindelser, transporttjenester for meldingsflyt samt Information Store med postboksdatabaser. En installasjon kan starte i det små; flere servere og databasekopier utvider den samme grunnmodellen for høy tilgjengelighet ([Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Denne sammenslåingen betyr ikke at alle funksjoner har samme tilstand. En HTTPS-frontend kan være tilgjengelig selv om den aktuelle databasen ikke er montert. SMTP kan motta forbindelser, mens en melding senere venter i en kø. Diagnosen følger derfor den faktiske veien og ikke bare serverens samlede status.

Den valgfrie **Edge Transport-rollen** står vanligvis i perimeternettverket og behandler utelukkende SMTP-trafikk. EdgeSync overfører utvalgt mottaker- og konfigurasjonsinformasjon til en lokal AD-LDS-instans. Edge har ingen postboksdatabase og erstatter ikke de interne postboksserverne ([Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Teknologistakk og avhengigheter

Serverbyggesteinen danner grunnlaget for teknologistakken. Exchange kjører på støttede Windows Server-versjoner og bruker Active Directory til organisasjons-, server- og mottaker­konfigurasjon. IIS tilbyr HTTP-endepunkter. PowerShell utgjør administrasjonsgrensesnittet. ESE lagrer postboks- og kødata i separate databaser ([Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Teknologi | Oppgave i Exchange-drift | Viktig administrasjonsspørsmål |
|---|---|---|
| Windows Server | Prosesser, tjenester, nettverk, sertifikatlager og hendelseslogger | Er verten frisk og riktig oppdatert? |
| Active Directory | Exchange-organisasjon, servere, mottakere, RBAC og rutingsinformasjon | Er den riktige endringen synlig på domenekontrollerne som brukes? |
| IIS og HTTPS | Outlook på nettet, EAC, EWS, ActiveSync, Autodiscover og MAPI/HTTP-frontender | Stemmer navn, sertifikat, autentisering og backendrute? |
| SMTP og TLS | Mottak og videresending av meldinger | Hvilken kobling mottok meldingen, og hvilken neste hopp ble valgt? |
| ESE | Postboksdatabaser, transportkø og transaksjonslogger | Hvilken database og loggsekvens hører sammen? |
| PowerShell | Administrasjon via cmdleter og RBAC | Hvilken rolle, hvilket omfang og hvilken serverkontekst gjelder? |

For eksperter er særlig Active Directory-avhengigheten viktig. Exchange-oppsettet utvider skjemaet og skriver organisasjonskonfigurasjon til Configuration Partition. Mottakerattributter ligger i domene­partisjonen. Replikeringsforsinkelse eller en utilgjengelig domenekontroller kan derfor påvirke ulike funksjoner forskjellig.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Åpne den interaktive Exchange On-Premises-grafikken direkte</a>.
</iframe>

## Transportpipelinen trinn for trinn

Med det tekniske fundamentet kan meldingsveien leses mer nøyaktig. En innkommende SMTP-forbindelse når først Front End Transport Service. Den mottar dialogen og formidler den til Transport Service; den leverer ikke selv meldingen til en postboks ([Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

**Transport Service** lagrer meldingen i sin kødatabase. Deretter kategoriserer den meldingen: mottakere løses opp, regler og Transport Agents kjøres, og rutingen bestemmer neste hopp. For en lokal postboks overleverer Mailbox Transport Delivery meldingen til Store. En melding sendt fra postboksen går tilbake til transporten via Mailbox Transport Submission.

Denne rekkefølgen forklarer typiske observasjoner. En vellykket SMTP-test bekrefter bare mottaket i frontenden. En `RECEIVE`-hendelse i Message Tracking beviser ennå ingen levering. Først de videre hendelsene, køen og eventuelt Store-tilstanden viser hvor forløpet endte ([Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Transport Agents og Mailflow Rules kan avvise, omdirigere, kopiere eller endre meldinger. Fordi det kan oppstå flere transportinstanser, bør søket ikke bare gjøres etter emne. Network Message ID, Internet Message ID, avsender, mottaker, tidspunkt og server gir sammen et mer pålitelig spor.

## Ruting, domener og koblinger

Etter mottaket må Exchange vite om en mottaker er lokal, eller om meldingen skal videresendes. **Accepted Domains** beskriver dette forholdet. Et autoritativt domene forventer alle gyldige mottakere i sin egen organisasjon. Et Internal Relay-domene tillater videresending av ukjente mottakere. External Relay overleverer domenet fullstendig til en annen e-postserver ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Receive Connectors klassifiserer innkommende økter basert på lokal binding, eksternt IP-område, autentisering og tillatelser. Send Connectors velger en utgående vei basert på adresseområde, kostnad, kildeservere og DNS- eller Smart Host-ruting. Flere passende koblinger vurderes etter de dokumenterte rutingsreglene; navnet på en kobling styrer ikke valget ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

For normal drift er en enkel modell nok: Receive Connector forklarer **hvordan en melding kommer inn**; Accepted Domain og mottakeroppløsning forklarer **om Exchange er ansvarlig**; Send Connector og ruting forklarer **hvor den går videre**. Eksperter supplerer med AD-nettsteder, Delivery Groups, DAG-medlemskap, Connector-scoping og transportregler.

## Postboksdatabase, logger og kontrollpunkt

Når transporten overleverer til Store, begynner en annen del av systemet. Exchange lagrer postbokser i ESE-postboksdatabaser. Endringer skrives først i transaksjonslogger og overføres senere til `.edb`-filen. Kontrollpunktfilen registrerer frem til hvilken loggposisjon databasesidene er skrevet ([Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Denne rekkefølgen muliggjør Crash Recovery, men krever filer som hører sammen. En kopiert `.edb` uten passende logger og kjent avslutningstilstand kan ikke automatisk gjenopprettes. På samme måte må en sikkerhetskopi ikke ukontrollert slette loggfiler som fortsatt trengs for gjenoppretting eller replikering.

Transportkøen bruker også ESE, men er en egen database med egne logger. Postboksdatabase og kø overvåkes og gjenopprettes derfor separat. En frisk postboksdatabase løser ikke et blokkert SMTP-neste hopp; en tom kø reparerer ikke en skadet postboksdatabasekopi ([Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group og Active Manager

En enkelt postboksserver forklarer normal drift. For høy tilgjengelighet kobles flere servere sammen i en **Database Availability Group**, DAG. Hver postboksdatabase har nøyaktig én aktiv kopi og kan ha passive kopier på andre DAG-medlemmer. Endringer overføres via logg- og blokkreplikering og spilles av på passive kopier ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

**Active Manager** i Microsoft Exchange Replication Service avgjør hvilken kopi som er aktiv. Best Copy and Server Selection vurderer blant annet kopi- og avspillingstilstand, aktiveringsblokker og serverhelse. En Copy Queue på null er derfor nyttig, men ikke et fullstendig bevis på at en kopi umiddelbart kan aktiveres ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

Høy tilgjengelighet for transport beskytter en annen del av veien. Shadow Redundancy beholder en ekstra kopi mens meldingen er underveis. Safety Net oppbevarer allerede behandlede meldinger for mulig ny levering etter databaseaktivering. DAG, Shadow Redundancy og Safety Net utfyller hverandre; ingen av de tre funksjonene erstatter en sikkerhetskopi mot utilsiktet sletting eller langvarig uoppdaget skade ([Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Klienttilgang og Autodiscover

Databasen kan være frisk selv om en bruker likevel ikke kan åpne Outlook. Client Access Services mottar HTTPS-forbindelser og videresender dem til backend på serveren med den aktive databasen. En lastbalanserer trenger derfor mer enn en åpen TCP-port: navn, sertifikat, protokollendepunkt og backendhelse må samsvare ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover leverer riktige innstillinger til klienten. Klienter innenfor domenet kan bruke Service Connection Points i Active Directory; eksterne og andre klienter følger DNS- og HTTPS-prosedyrer. Feil oppstår ofte på grunn av foreldede SCP-er, motstridende DNS-svar, feil sertifikatnavn eller en frontend som videresender til feil backend ([Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP er den typiske Outlook-transporten. Outlook på nettet, EWS og ActiveSync bruker også HTTPS, men har egne virtuelle kataloger, autentisering og applikasjonsegenskaper. En vellykket OWA-test beviser derfor ikke automatisk en frisk MAPI/HTTP-økt ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory og mottakere

Etter transport og klienttilgang står katalogen igjen som felles grunnlag. Exchange lagrer organisasjons- og serverkonfigurasjon samt mottakerattributter i Active Directory. Cmdleter skriver ikke disse dataene til en privat Exchange-database, men til AD via Exchange-logikk ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Et mottakerproblem undersøkes derfor langs tre spørsmål: Finnes det riktige objektet? Er type, primæradresse, proxyadresser og målattributter korrekte? Har endringen nådd domenekontrolleren som den berørte Exchange-tjenesten bruker? Først deretter lønner det seg å lete i transporten.

For eksperter kommer globale kataloger, AD-nettsteder, Recipient Update, Address Book Policies og hybridattributter i tillegg. Direkte endringer med generiske AD-verktøy omgår Exchange-validering og kan skape konfigurasjoner som er syntaktisk til stede, men faglig inkonsistente.

## Sikkerhet og administrativ kontroll

Exchange publiserer SMTP- og HTTPS-tjenester og behandler katalog- og postboksdata med høye privilegier. Grunnlaget består av raskt installerte Security Updates, minimalt tilgjengelige endepunkter, passende sertifikater, sikrede administrasjonskontoer og sporbare endringer ([Exchange Server Security Updates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC skiller oppgaver gjennom roller, rolleg­rupper og omfang. Postboksrettigheter som Full Access eller Send As holdes atskilt fra dette. Administrator Audit Logging logger cmdlet-endringer, men erstatter ikke operativsystem-, Active Directory- og sikkerhetslogger ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

For eksperter er administrasjonsgrensesnittet selv en del av beskyttelsesmodellen. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP og hypervisortilgang har ulike rettigheter og protokoller. En kompromittert serveradministrator kan utføre tiltak utenfor Exchange-RBAC; tiering og separate privilegerte kontoer forblir derfor viktige.

## Drift: fra symptom til konkret server

Managed Availability kjører Probes, Monitors og Responders. Health Sets samler disse resultatene etter funksjon og kan utløse automatiske gjenopprettingstiltak. De er et godt utgangspunkt, men ikke en fullstendig ende-til-ende-kontroll ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

For meldingsflyt starter lokal diagnose med [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) og [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Antall køer, neste hopp, nytt forsøk-tid og `LastError` hører sammen. For databaser følger [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) og [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) viser Health Sets og Monitors.

Disse cmdletene kjører i Exchange Management Shell på støttede Windows-servere. Nettverks- og DNS-tester kan derimot utføres fra begge administrasjonsplattformene. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) kontrollerer et TCP-endepunkt i Windows; [`nc`](https://man.openbsd.org/nc) utfører den samme porttesten i Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerer DNS. For SMTP med STARTTLS egner [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) seg, og for en kontrollert SMTP-dialog [`swaks`](https://jetmore.org/john/code/swaks/).

Diagnoserekkefølgen er: løs opp offentlig eller internt navn, kontroller forbindelsen til riktig frontend, bekreft mottak i protokolloggen, følg sporingshendelser, kontroller kø og neste hopp, og undersøk først Store og databasen ved lokal levering.

## Sikkerhetskopiering og gjenoppretting

Høy tilgjengelighet holder tjenesten tilgjengelig ved enkeltfeil; gjenoppretting gjenskaper en ønsket tidligere eller tapt tilstand. Exchange dokumenterer Server Recovery, databasegjenoppretting og Recovery Database som ulike prosedyrer ([Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Et gjenopprettbart inventar omfatter minst Active Directory, Exchange-organisasjon og serverkonfigurasjon, sertifikater og private nøkler, postboksdatabaser med logger, connector- og regelkonfigurasjon samt dokumenterte installasjons- og gjenopprettingsparametere. Recovery Database gjør det mulig å montere en gjenopprettet database isolert og overføre innhold til aktive postbokser ([Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Eksperter tester ikke bare om en sikkerhetskopieringsjobb var vellykket. De måler hvor lang tid det faktisk tar å gjenopprette Active Directory, en utgått server, en database og enkelt postboksinnhold. Det kontrolleres hvilke loggsekvenser som kreves, hvilke DNS- og sertifikatavhengigheter som finnes, og om klient- og SMTP-veier fungerer igjen etter gjenoppretting.

## Teknisk utvikling og grenser

Exchange 4.0 kom i 1996. Tidlige versjoner brukte en egen katalog, MAPI og ESE; SMTP og Active Directory ble sentrale plattformkomponenter med Exchange 2000. Exchange 2007 introduserte serverroller og Exchange Management Shell. Exchange 2010 erstattet eldre klyngemodeller med Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Senere versjoner samlet igjen Client Access- og postboksfunksjoner i en felles serverbyggestein. Exchange Server Subscription Edition videreførte den lokale produktlinjen i Modern Lifecycle i 2025. Build-versjoner, støttede oppgraderingsveier og Security Updates kontrolleres i Microsofts løpende dokumentasjon før hver endring ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises passer når organisasjonen trenger kontroll over databasedrift, nettverksveier og lokal integrasjon, og kan levere den nødvendige 24/7-driften. Baksiden er komplekse avhengigheter, kontinuerlig sikkerhetsvedlikehold og ansvar for gjenoppretting. En enkelt server kan se enkel ut; en robust Exchange-tjeneste er alltid også et Active Directory-, nettverks-, sertifikat-, lagrings- og driftsprosjekt.

## Kilder

- [Microsoft Learn – Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Monitor database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Exchange admin interfaces](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
