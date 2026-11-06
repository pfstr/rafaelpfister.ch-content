---
title: "Exchange On-Premises: serverarkitektur och drift"
blatt: "exchange-on-premises"
description: "Exchange i det egna datacentret: mailbox- och Edge-roller, transportpipeline, Active Directory, ESE-databaser, DAG, klientåtkomst, säkerhet, övervakning, backup och återställning."
fakten:
  - label: Produktroll
    wert: Egendriven plattform för meddelanden och grupprogramvara
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Serverroller
    wert: Mailbox och valfri Edge Transport
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Operativsystem
    wert: Windows Server enligt Exchanges systemkrav
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Katalog
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Mailboxlagring
    wert: ESE-databas, transaktionsloggar och kontrollpunkt
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: Hög tillgänglighet
    wert: Database Availability Group och databaskopior
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Transport
    wert: Frontend Transport, Transport Service och Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Transportresiliens
    wert: Shadow Redundancy och Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Klientåtkomst
    wert: HTTPS, MAPI/HTTP, Outlook på webben, EWS och ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Administration
    wert: Exchange Admin Center och Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Övervakning
    wert: Managed Availability, Health Sets, händelseloggar och prestandaräknare
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Återställning
    wert: Server Recovery, databasåterställning och Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 45315e2f767bcdfb1b554eea6df80335a5ca86bbc102a345b83dd0e54cb0693b
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:35:13.377Z
translationReview: automatic
---

# Exchange On-Premises: serverarkitektur och drift

**Exchange On-Premises** innebär att organisationen driver Exchange i sin egen infrastruktur. Den kontrollerar Windows-värdar, Active Directory, certifikat, transporttjänster, köer, mailboxdatabaser och återställning. Microsoft tillhandahåller produktkod, dokumentation och uppdateringar; tillgänglighet och säkert underhåll förblir operatörens ansvar ([Exchange Server documentation](https://learn.microsoft.com/en-us/exchange/exchange-server), [Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Den praktiska skillnaden mot Exchange Online märks omedelbart vid en störning. En On-Prem-administratör kan undersöka en transportkö på en specifik server, kontrollera statusen för en databaskopia och kontrollerat växla till en annan kopia. Men då måste administratören också förstå hur SMTP, Active Directory, ESE, Windows Failover Clustering, IIS och Exchange-tjänster samverkar.

## Mailboxservern är den centrala byggstenen

Moderna Exchange-servrar använder **Mailboxservern** som en gemensam byggsten. Den innehåller Client Access-tjänster som tar emot och vidarebefordrar anslutningar, transporttjänster för meddelandeflödet samt Information Store med mailboxdatabaser. En installation kan börja smått; flera servrar och databaskopior bygger ut samma grundmodell för hög tillgänglighet ([Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Denna sammanslagning innebär inte att alla funktioner har samma status. Ett HTTPS-frontend kan vara tillgängligt trots att den adresserade databasen inte är monterad. SMTP kan ta emot anslutningar medan ett meddelande senare väntar i en kö. Diagnosen följer därför den faktiska vägen och inte bara serverns övergripande status.

Den valfria **Edge Transport-rollen** finns vanligtvis i perimeternätet och hanterar endast SMTP-trafik. EdgeSync överför utvald mottagar- och konfigurationsinformation till en lokal AD-LDS-instans. Edge har ingen mailboxdatabas och är ingen ersättning för de interna mailboxservrarna ([Edge Transport servers](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Teknikstack och beroenden

Serverbyggstenen ger upphov till teknikstacken. Exchange körs på Windows Server-versioner som stöds och använder Active Directory för organisations-, server- och mottagarkonfiguration. IIS tillhandahåller HTTP-slutpunkter. PowerShell utgör administrationsgränssnittet. ESE lagrar mailbox- och ködata i separata databaser ([Exchange Server system requirements](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Teknik | Uppgift i Exchange-driften | Viktig administratörsfråga |
|---|---|---|
| Windows Server | Processer, tjänster, nätverk, certifikatarkiv och händelseloggar | Är värden frisk och korrekt patchad? |
| Active Directory | Exchange-organisation, servrar, mottagare, RBAC och routningsinformation | Är rätt ändring synlig på de Domain Controllers som används? |
| IIS och HTTPS | Outlook på webben, EAC, EWS, ActiveSync, Autodiscover och MAPI/HTTP-frontends | Stämmer namn, certifikat, autentisering och backendrutt? |
| SMTP och TLS | Mottagning och vidarebefordran av meddelanden | Vilken Connector tog emot och vilken Next Hop valdes? |
| ESE | Mailboxdatabaser, transportkö och transaktionsloggar | Vilken databas och loggsekvens hör ihop? |
| PowerShell | Administration via Cmdlets och RBAC | Vilken roll, vilket scope och vilken serverkontext gäller? |

För experter är framför allt Active Directory-beroendet viktigt. Exchange-installationen utökar schemat och skriver organisationskonfiguration till Configuration Partition. Mottagarattribut finns i domänpartitionen. Replikeringsfördröjning eller en otillgänglig Domain Controller kan därför påverka olika funktioner på olika sätt.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Öppna den interaktiva Exchange On-Premises-grafiken direkt</a>.
</iframe>

## Transportpipelinen steg för steg

Med den tekniska grunden går det att läsa meddelandevägen mer exakt. En inkommande SMTP-anslutning når först Front End Transport Service. Den tar emot dialogen och förmedlar den till Transport Service; den levererar inte själv meddelandet till en mailbox ([Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

**Transport Service** lagrar meddelandet i sin ködatabas. Därefter kategoriserar den det: mottagare löses upp, regler och Transport Agents körs, och routningen bestämmer nästa hopp. För en lokal mailbox överlämnar Mailbox Transport Delivery meddelandet till Store. Ett meddelande som skickas från mailboxen återgår till transporten via Mailbox Transport Submission.

Denna ordning förklarar typiska observationer. Ett lyckat SMTP-test bevisar endast mottagning vid frontend. En `RECEIVE`-händelse i Message Tracking bevisar ännu ingen leverans. Först de följande händelserna, kön och vid behov Store-statusen visar var förloppet slutade ([Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Transport Agents och Mailflow Rules kan avvisa, omdirigera, kopiera eller ändra meddelanden. Eftersom flera transportinstanser kan uppstå bör sökningen inte bara göras efter ämnesrad. Network Message ID, Internet Message ID, avsändare, mottagare, tidpunkt och server ger tillsammans det mer tillförlitliga spåret.

## Routning, domäner och Connectors

Efter mottagningen måste Exchange veta om en mottagare är lokal eller om meddelandet ska vidarebefordras. **Accepted Domains** beskriver detta förhållande. En auktoritativ domän förväntar sig alla giltiga mottagare i den egna organisationen. En Internal Relay-domän tillåter vidarebefordran av okända mottagare. External Relay överlämnar domänen helt till en annan e-postserver ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Receive Connectors klassificerar inkommande sessioner utifrån lokal bindning, intervall för fjärr-IP, autentisering och behörigheter. Send Connectors väljer en utgående väg baserat på adressutrymme, kostnad, källservrar samt DNS- eller Smart Host-routning. Flera matchande Connectors bedöms enligt de dokumenterade routningsreglerna; namnet på en Connector styr inte urvalet ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Mail routing in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

För normal drift räcker en enkel modell: Receive Connector förklarar **hur ett meddelande kommer in**; Accepted Domain och mottagarupplösning förklarar **om Exchange ansvarar**; Send Connector och routning förklarar **vart det går vidare**. Experter kompletterar med AD Sites, Delivery Groups, DAG-medlemskap, Connector-scoping och transportregler.

## Mailboxdatabas, loggar och kontrollpunkt

När transporten överlämnar till Store börjar en annan del av systemet. Exchange lagrar mailboxar i ESE-mailboxdatabaser. Ändringar skrivs först till transaktionsloggar och förs senare in i `.edb`-filen. Kontrollpunktsfilen anger upp till vilken loggposition databassidorna har skrivits ([Transaction logs and checkpoint files](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Denna ordning möjliggör Crash Recovery men kräver sammanhörande filer. En kopierad `.edb` utan matchande loggar och känt avstängningsläge kan inte automatiskt återställas. Inte heller får en backup okontrollerat radera loggfiler som fortfarande behövs för Recovery eller replikering.

Transportkön använder också ESE, men är en egen databas med egna loggar. Mailboxdatabas och kö övervakas och återställs därför separat. En frisk mailboxdatabas åtgärdar inte en blockerad SMTP-Next-Hop; en tom kö reparerar inte en skadad mailboxkopia ([Queues and the queue database](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group och Active Manager

En enskild mailboxserver förklarar normal drift. För hög tillgänglighet kopplas flera servrar samman till en **Database Availability Group**, DAG. Varje mailboxdatabas har exakt en aktiv kopia och kan ha passiva kopior på andra DAG-medlemmar. Ändringar överförs via logg- och blockreplikering och spelas upp på de passiva kopiorna ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Mailbox database copies](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

**Active Manager** i Microsoft Exchange Replication Service avgör vilken kopia som är aktiv. Best Copy and Server Selection utvärderar bland annat kopierings- och uppspelningsstatus, aktiveringsblockeringar och serverhälsa. En Copy Queue på noll är därför användbar, men inte ett fullständigt bevis på att en kopia kan aktiveras omedelbart ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

Transporthög tillgänglighet skyddar en annan del av vägen. Shadow Redundancy håller en extra kopia så länge meddelandet är på väg. Safety Net behåller redan bearbetade meddelanden för en möjlig ny leverans efter databasaktivering. DAG, Shadow Redundancy och Safety Net kompletterar varandra; ingen av de tre funktionerna ersätter en backup mot oavsiktlig radering eller långvariga oupptäckta skador ([Transport high availability](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Klientåtkomst och Autodiscover

Databasen kan vara frisk och ändå kan en användare inte öppna Outlook. Client Access Services tar emot HTTPS-anslutningar och vidarebefordrar dem till backend på servern med den aktiva databasen. En Load Balancer behöver därför mer än en öppen TCP-port: namn, certifikat, protokollslutpunkt och backendhälsa måste stämma överens ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover ger klienten rätt inställningar. Klienter i samma domän kan använda Service Connection Points i Active Directory; externa och andra klienter följer DNS- och HTTPS-förfaranden. Fel uppstår ofta på grund av föråldrade SCP:er, motstridiga DNS-svar, felaktiga certifikatnamn eller ett frontend som vidarebefordrar till fel backend ([Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP är den typiska Outlook-transporten. Outlook på webben, EWS och ActiveSync använder också HTTPS, men har egna virtuella kataloger, autentisering och applikationsegenskaper. Ett lyckat OWA-test bevisar därför inte automatiskt en frisk MAPI/HTTP-session ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory och mottagare

Efter transport och klientåtkomst återstår katalogen som gemensam grund. Exchange lagrar organisations- och serverkonfiguration samt mottagarattribut i Active Directory. Cmdlets skriver inte dessa data till en privat Exchange-databas, utan via Exchange-logik till AD ([Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Ett mottagarproblem undersöks därför utifrån tre frågor: Finns rätt objekt? Är typ, primär adress, proxyadresser och målattribut korrekta? Har ändringen nått den Domain Controller som den berörda Exchange-tjänsten använder? Först därefter är det meningsfullt att söka i transporten.

För experter tillkommer globala kataloger, AD Sites, Recipient Update, Address Book Policies och hybridattribut. Direkta ändringar med generiska AD-verktyg kringgår Exchange-validering och kan skapa konfigurationer som är syntaktiskt närvarande men sakligt inkonsekventa.

## Säkerhet och administrativ kontroll

Exchange publicerar SMTP- och HTTPS-tjänster och behandlar högt privilegierade katalog- och mailboxdata. Grunden består av snabba Security Updates, minimalt tillgängliga slutpunkter, lämpliga certifikat, säkrade administrationskonton och spårbara ändringar ([Exchange Server Security Updates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [TLS certificates in Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC separerar uppgifter genom roller, rollgrupper och scopes. Mailboxbehörigheter som Full Access eller Send As är separata från detta. Administrator Audit Logging loggar Cmdlet-ändringar, men ersätter inte operativsystem-, Active Directory- och säkerhetsloggar ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Administrator audit logging](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

För experter är själva administrationsgränssnittet en del av skyddsmodellen. EAC, Exchange Management Shell, Remote PowerShell, WinRM, RDP och hypervisoråtkomst har olika behörigheter och protokoll. En komprometterad serveradministratör kan utföra åtgärder utanför Exchange-RBAC; Tiering och separata privilegierade konton är därför fortsatt viktiga.

## Drift: från symptom till konkret server

Managed Availability kör Probes, Monitors och Responders. Health Sets sammanfattar dessa resultat per funktion och kan utlösa automatiska Recovery-åtgärder. De är en bra startpunkt, men inte en fullständig end-to-end-kontroll ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

För Mailflow börjar den lokala diagnosen med [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) och [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). Köantal, Next Hop, återförsökstid och `LastError` hör ihop. För databaser följer [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) och [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) visar Health Sets och Monitors.

Dessa Cmdlets körs i Exchange Management Shell på Windows Server-versioner som stöds. Nätverks- och DNS-tester kan däremot utföras från båda administratörsplattformarna. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) kontrollerar en TCP-slutpunkt under Windows; [`nc`](https://man.openbsd.org/nc) utför samma porttest under Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) kontrollerar DNS. För SMTP med STARTTLS passar [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), och för en kontrollerad SMTP-dialog [`swaks`](https://jetmore.org/john/code/swaks/).

Diagnosordningen är: lös upp offentligt eller internt namn, kontrollera anslutningen till rätt frontend, bekräfta mottagning i protokollloggen, följ spårningshändelser, kontrollera kö och Next Hop och undersök Store och databas först vid lokal leverans.

## Backup och Recovery

Hög tillgänglighet håller tjänsten tillgänglig vid enskilda fel; Recovery återställer ett önskat tidigare eller förlorat tillstånd. Exchange dokumenterar Server Recovery, databasåterställning och Recovery Database som olika förfaranden ([Backup, restore, and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Ett återställningsbart inventarium omfattar minst Active Directory, Exchange-organisation och serverkonfiguration, certifikat och privata nycklar, mailboxdatabaser med loggar, Connector- och regelkonfiguration samt dokumenterade installations- och Recovery-parametrar. Recovery Database gör det möjligt att montera en återställd databas isolerat och överföra innehåll till aktiva mailboxar ([Restore data using a recovery database](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Experter testar inte bara om ett backupjobb lyckades. De mäter hur lång tid det faktiskt tar att återställa Active Directory, en felande server, en databas och enskilt mailboxinnehåll. Då kontrolleras vilka loggsekvenser som behövs, vilka DNS- och certifikatberoenden som finns och om klient- och SMTP-vägarna fungerar igen efter återställningen.

## Teknisk utveckling och gränser

Exchange 4.0 lanserades 1996. Tidiga versioner använde en egen katalog, MAPI och ESE; SMTP och Active Directory blev centrala plattformskomponenter med Exchange 2000. Exchange 2007 introducerade serverroller och Exchange Management Shell. Exchange 2010 ersatte äldre klustermodeller med Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Senare versioner sammanförde åter Client Access- och Mailbox-funktioner i en gemensam serverbyggsten. Exchange Server Subscription Edition fortsatte den lokala produktlinjen 2025 inom Modern Lifecycle. Buildnummer, uppgraderingsvägar som stöds och Security Updates kontrolleras före varje ändring i Microsofts löpande dokumentation ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server build numbers and release dates](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises lämpar sig när organisationen behöver kontroll över databasdrift, nätverksvägar och lokal integration och kan upprätthålla den nödvändiga 24/7-driften. Nackdelen är komplexa beroenden, kontinuerligt säkerhetsunderhåll och ansvar för Recovery. En enskild server kan se enkel ut; en robust Exchange-tjänst är alltid också ett Active Directory-, nätverks-, certifikat-, lagrings- och driftprojekt.

## Källor

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
