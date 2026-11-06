---
title: "Exchange On-Premises: Serverarkitektur og drift"
blatt: "exchange-on-premises"
description: "Exchange i eget datasenter: postboks- og Edge-roller, transport-pipeline, Active Directory, ESE-databaser, DAG, klienttilgang, sikkerhet, overvåking, sikkerhetskopiering og gjenoppretting."
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
    wert: Managed Availability, Health Sets, hendelseslogger og ytelsesindikatorer
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
translationSourceHash: 45315e2f767bcdfb1b554eea6df80335a5ca86bbc102a345b83dd0e54cb0693b
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:35:54.710Z
translationReview: automatic
---

# Exchange On-Premises: Serverarkitektur og drift

**Exchange On-Premises** betyr at organisasjonen drifter Exchange i sin egen infrastruktur. Den kontrollerer Windows-verter, Active Directory, sertifikater, transporttjenester, køer, postboksdatabaser og gjenoppretting. Microsoft leverer produktkode, dokumentasjon og oppdateringer; tilgjengelighet og sikkert vedlikehold forblir operatørens ansvar ([Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server), [Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Den praktiske forskjellen fra Exchange Online viser seg umiddelbart ved en feil. En On-Prem-administrator kan undersøke en transportkø på en konkret server, kontrollere tilstanden til en databasekopi og kontrollert bytte til en annen kopi. Men administratoren må også forstå hvordan SMTP, Active Directory, ESE, Windows Failover Clustering, IIS og Exchange-tjenester samspiller.

## Postboksserveren er den sentrale byggesteinen

Moderne Exchange-servere bruker **postboksserveren** som felles byggestein. Den inneholder Client Access-tjenester som mottar og videresender forbindelser, transporttjenester for meldingsflyt samt Information Store med postboksdatabaser. En installasjon kan starte i det små; flere servere og databasekopier utvider den samme grunnmodellen for høy tilgjengelighet ([Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Denne sammenslåingen betyr ikke at alle funksjoner har samme tilstand. Et HTTPS-frontend kan være tilgjengelig selv om den adresserte databasen ikke er montert. SMTP kan motta forbindelser mens en melding senere venter i en kø. Diagnosen følger derfor den faktiske veien og ikke bare serverens samlede status.

Den valgfrie **Edge Transport-rollen** plasseres typisk i perimeter-nettverket og behandler utelukkende SMTP-trafikk. EdgeSync overfører utvalgt mottaker- og konfigurasjonsinformasjon til en lokal AD-LDS-instans. Edge har ingen postboksdatabase og er ingen erstatning for de interne postboksserverne ([Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Teknologistack og avhengigheter

Serverbyggesteinen danner grunnlaget for teknologistacken. Exchange kjører på støttede Windows Server-versjoner og bruker Active Directory for organisasjons-, server- og mottakeroppsett. IIS tilbyr HTTP-endepunkter. PowerShell utgjør administrasjonsgrensesnittet. ESE lagrer postboks- og kødata i separate databaser ([Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Teknologi | Oppgave i Exchange-drift | Viktig administrasjonsspørsmål |
|---|---|---|
| Windows Server | Prosesser, tjenester, nettverk, sertifikatlager og hendelseslogger | Er verten sunn og riktig oppdatert? |
| Active Directory | Exchange-organisasjon, servere, mottakere, RBAC og rutinginformasjon | Er den riktige endringen synlig på domenekontrollerne som brukes? |
| IIS og HTTPS | Outlook på nettet, EAC, EWS, ActiveSync, Autodiscover og MAPI/HTTP-frontender | Stemmer navn, sertifikat, autentisering og backend-rute? |
| SMTP og TLS | Mottak og videresending av meldinger | Hvilken connector mottok, og hvilken neste hopp ble valgt? |
| ESE | Postboksdatabaser, transportkø og transaksjonslogger | Hvilken database og loggsekvens hører sammen? |
| PowerShell | Administrasjon via cmdlets og RBAC | Hvilken rolle, hvilket omfang og hvilken serverkontekst gjelder? |

For eksperter er særlig Active Directory-avhengigheten viktig. Exchange-oppsettet utvider skjemaet og skriver organisasjonskonfigurasjon i Configuration Partition. Mottakerattributter ligger i domene-partisjonen. Replikeringsforsinkelse eller en utilgjengelig domenekontroller kan derfor påvirke ulike funksjoner på forskjellige måter.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Åpne interaktiv Exchange On-Premises-grafikk direkte</a>.
</iframe>

## Transport-pipelinen steg for steg

Med det tekniske grunnlaget kan meldingsveien leses mer detaljert. En innkommende SMTP-forbindelse når først Front End Transport Service. Den mottar dialogen og formidler den til Transport Service; den leverer ikke selv meldingen til en postboks ([Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

**Transport Service** lagrer meldingen i sin kødatabase. Deretter kategoriserer den den: mottakere løses opp, regler og Transport Agents kjører, og rutingen bestemmer neste hopp. For en lokal postboks overleverer Mailbox Transport Delivery meldingen til Store. En melding sendt fra postboksen går tilbake til transporten via Mailbox Transport Submission.

Denne rekkefølgen forklarer typiske observasjoner. En vellykket SMTP-test beviser bare mottak i frontend. En `RECEIVE`-hendelse i Message Tracking beviser ennå ikke levering. Først de videre hendelsene, køen og eventuelt Store-tilstanden viser hvor prosessen endte ([Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Transport Agents og Mailflow Rules kan avvise, omdirigere, kopiere eller endre meldinger. Siden flere transportinstanser kan oppstå, bør søket ikke bare baseres på emne. Network Message ID, Internet Message ID, avsender, mottaker, tidspunkt og server gir sammen et mer pålitelig spor.

## Ruting, domener og connectorer

Etter mottak må Exchange vite om en mottaker er lokal eller om meldingen skal videresendes. **Accepted Domains** beskriver dette forholdet. Et autoritativt domene forventer alle gyldige mottakere i egen organisasjon. Et Internal Relay-domene tillater videresending av ukjente mottakere. External Relay overleverer domenet fullstendig til en annen e-postserver ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Receive Connectors klassifiserer innkommende økter basert på lokal binding, eksternt IP-område, autentisering og tillatelser. Send Connectors velger en utgående vei fra adresserom, kostnad, kildeservere og DNS- eller Smart Host-ruting. Flere passende connectorer vurderes etter de dokumenterte rutingreglene; navnet på en connector styrer ikke valget ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

For normal drift er en enkel modell tilstrekkelig: Receive Connector forklarer **hvordan en melding kommer inn**; Accepted Domain og mottakeroppløsning forklarer **om Exchange er ansvarlig**; Send Connector og ruting forklarer **hvor den går videre**. Eksperter supplerer med AD-sites, Delivery Groups, DAG-medlemskap, connector-scoping og transportregler.

## Postboksdatabase, logger og kontrollpunkt

Når transporten overleverer til Store, begynner en annen del av systemet. Exchange lagrer postbokser i ESE-postboksdatabaser. Endringer skrives først til transaksjonslogger og senere til `.edb`-filen. Kontrollpunktfilen registrerer frem til hvilken loggposisjon databasesidene er skrevet ([Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Denne rekkefølgen muliggjør krasjgjenoppretting, men krever tilhørende filer. En kopiert `.edb` uten riktige logger og kjent avslutningstilstand kan ikke automatisk gjenopprettes. Et sikkerhetskopieringssystem må heller ikke ukontrollert slette loggfiler som fortsatt trengs for gjenoppretting eller replikering.

Transportkøen bruker også ESE, men er en egen database med egne logger. Postboksdatabase og kø overvåkes og gjenopprettes derfor separat. En sunn postboksdatabase løser ikke et blokkert SMTP-neste-hopp; en tom kø reparerer ikke en skadet postboksdatabasekopi ([Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group og Active Manager

En enkelt postboksserver forklarer normal drift. For høy tilgjengelighet kobles flere servere sammen i en **Database Availability Group**, DAG. Hver postboksdatabase har nøyaktig én aktiv kopi og kan ha passive kopier på andre DAG-medlemmer. Endringer overføres via logg- og blokk-replikering og spilles av på de passive kopiene ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

**Active Manager** i Microsoft Exchange Replication Service avgjør hvilken kopi som er aktiv. Best Copy and Server Selection vurderer blant annet kopierings- og avspillingstilstand, aktiveringsblokker og serverhelse. En Copy Queue på null er derfor nyttig, men ikke et fullstendig bevis på at en kopi kan aktiveres umiddelbart ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

Transporthøytilgjengelighet beskytter en annen del av veien. Shadow Redundancy beholder en ekstra kopi så lenge meldingen er underveis. Safety Net oppbevarer allerede behandlede meldinger for mulig ny levering etter databaseaktivering. DAG, Shadow Redundancy og Safety Net utfyller hverandre; ingen av de tre funksjonene erstatter en sikkerhetskopi mot utilsiktet sletting eller langvarig uoppdaget korrupsjon ([Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Klienttilgang og Autodiscover

Databasen kan være sunn, og likevel kan en bruker ikke åpne Outlook. Client Access Services mottar HTTPS-forbindelser og videresender dem til backend på serveren med den aktive databasen. En lastbalanserer trenger derfor mer enn en åpen TCP-port: navn, sertifikat, protokollendepunkt og backend-helse må passe sammen ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover leverer de riktige innstillingene til klienten. Klienter innenfor domenet kan bruke Service Connection Points i Active Directory; eksterne og andre klienter følger DNS- og HTTPS-prosedyrer. Feil oppstår ofte på grunn av utdaterte SCP-er, motstridende DNS-svar, feil sertifikatnavn eller et frontend som videresender til feil backend ([Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP er den typiske Outlook-transporten. Outlook på nettet, EWS og ActiveSync bruker også HTTPS, men har egne virtuelle kataloger, autentisering og applikasjonsegenskaper. En vellykket OWA-test beviser derfor ikke automatisk en sunn MAPI/HTTP-økt ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory og mottakere

Etter transport og klienttilgang gjenstår katalogen som felles grunnlag. Exchange lagrer organisasjons- og serverkonfigurasjon samt mottakerattributter i Active Directory. Cmdlets skriver ikke disse dataene til en privat Exchange-database, men via Exchange-logikk til AD ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Et mottakerproblem undersøkes derfor langs tre spørsmål: Finnes riktig objekt? Er type, primæradresse, proxyadresser og målattributter riktige? Har endringen nådd domenekontrolleren som den berørte Exchange-tjenesten bruker? Først deretter er det verdt å lete i transporten.

For eksperter kommer globale kataloger, AD-sites, Recipient Update, Address Book Policies og hybriddattributter i tillegg. Direkte endringer med generiske AD-verktøy omgår Exchange-validering og kan opprette konfigurasjoner som finnes syntaktisk, men er faglig inkonsistente.

## Sikkerhet og administrativ kontroll

Exchange publiserer SMTP- og HTTPS-tjenester og behandler høyt privilegerte katalog- og postboksdata. Grunnlaget består av tidsriktige sikkerhetsoppdateringer, minst mulig tilgjengelige endepunkter, passende sertifikater, sikrede administrasjonskontoer og sporbare endringer ([Exchange Server Security Updates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC skiller oppgaver gjennom roller, rollegrupper og omfang. Postboksrettigheter som Full Access eller Send As holdes atskilt fra dette. Administrator Audit Logging logger cmdlet-endringer, men erstatter ikke operativsystem-, Active Directory- og sikkerhetslogger ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

For eksperter er selve administrasjonsgrensesnittet en del av beskyttelsesmodellen. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP og hypervisor-tilgang har ulike rettigheter og protokoller. En kompromittert serveradministrator kan utføre tiltak utenfor Exchange-RBAC; nivådeling og separate privilegerte kontoer forblir derfor viktig.

## Drift: fra symptom til konkret server

Managed Availability kjører Probes, Monitors og Responders. Health Sets samler disse resultatene etter funksjon og kan utløse automatiske gjenopprettingstiltak. De er et godt utgangspunkt, men ikke en fullstendig ende-til-ende-kontroll ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

For meldingsflyt starter den lokale diagnosen med [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) og [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Antall køer, neste hopp, nytt forsøk-tid og `LastError` hører sammen. For databaser følger [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) og [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) viser Health Sets og Monitors.

Disse cmdlets kjører i Exchange Management Shell på støttede Windows-servere. Nettverks- og DNS-tester kan derimot utføres fra begge administrasjonsplattformer. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) kontrollerer et TCP-endepunkt under Windows; [`nc`](https://man.openbsd.org/nc) utfører den samme porttesten under Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerer DNS. For SMTP med STARTTLS passer [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), og for en kontrollert SMTP-dialog [`swaks`](https://jetmore.org/john/code/swaks/).

Diagnoserekkefølgen er: løs opp offentlig eller internt navn, kontroller forbindelsen til riktig frontend, bekreft mottak i protokolloggen, følg sporingshendelser, kontroller kø og neste hopp, og undersøk Store og database først ved lokal levering.

## Sikkerhetskopiering og gjenoppretting

Høy tilgjengelighet holder tjenesten tilgjengelig ved enkeltfeil; gjenoppretting gjenoppretter en ønsket tidligere eller tapt tilstand. Exchange dokumenterer Server Recovery, databasegjenoppretting og Recovery Database som forskjellige prosedyrer ([Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Et gjenopprettbart inventar omfatter minst Active Directory, Exchange-organisasjon og serverkonfigurasjon, sertifikater og private nøkler, postboksdatabaser med logger, connector- og regelkonfigurasjon samt dokumenterte installasjons- og gjenopprettingsparametere. Recovery Database gjør det mulig å montere en gjenopprettet database isolert og overføre innhold til aktive postbokser ([Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Eksperter tester ikke bare om en sikkerhetskopieringsjobb var vellykket. De måler hvor lang tid det faktisk tar å gjenopprette Active Directory, en utgått server, en database og enkelt innhold i postbokser. Samtidig kontrolleres hvilke loggsekvenser som trengs, hvilke DNS- og sertifikatavhengigheter som finnes, og om klient- og SMTP-veier fungerer igjen etter gjenoppretting.

## Teknisk utvikling og grenser

Exchange 4.0 ble utgitt i 1996. Tidlige versjoner brukte en egen katalog, MAPI og ESE; SMTP og Active Directory ble sentrale plattformkomponenter med Exchange 2000. Exchange 2007 introduserte serverroller og Exchange Management Shell. Exchange 2010 erstattet eldre klyngemodeller med Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Senere versjoner samlet Client Access- og postboksfunksjoner igjen i en felles serverbyggestein. Exchange Server Subscription Edition fortsatte den lokale produktlinjen i Modern Lifecycle i 2025. Build-versjoner, støttede oppgraderingsveier og sikkerhetsoppdateringer kontrolleres i Microsofts løpende dokumentasjon før hver endring ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises egner seg når organisasjonen trenger kontroll over databasedrift, nettverksveier og lokal integrasjon, og kan levere den nødvendige 24/7-driften. Baksiden er komplekse avhengigheter, kontinuerlig sikkerhetsvedlikehold og ansvar for gjenoppretting. En enkelt server kan se enkel ut; en robust Exchange-tjeneste er alltid også et Active Directory-, nettverks-, sertifikat-, lagrings- og driftsprosjekt.

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
