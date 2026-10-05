---
title: "Exchange Online: arkitektur, e-postflyt og drift"
blatt: "exchange-online"
description: "Exchange Online for meldingsadministratorer: tenant- og mottakermodell, EOP-transport, koblinger, klienttilgang, PowerShell og Graph, Message Trace, oppbevaring, sikkerhet og gjenoppretting."
fakten:
  - label: Produktrolle
    wert: Skybasert e-post-, kalender- og katalogtjeneste
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Plattform
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: Mottak av e-post
    wert: Exchange Online Protection og SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Mottakere
    wert: Postbokser, grupper, kontakter, e-postbrukere og ressurser
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Domener
    wert: Authoritative eller Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Ruting
    wert: Inbound- og Outbound Connectors, regler og MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Klienttilgang
    wert: Outlook, Outlook på nettet, ActiveSync og IMAP/POP-spesialtilfeller
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identitet
    wert: Microsoft Entra ID og moderne godkjenning
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Administrasjon
    wert: Exchange Admin Center og Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph for e-post-, kalender- og administrasjonsfunksjoner
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnostikk
    wert: Message Trace, rapporter og Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Oppbevaring
    wert: Recoverable Items, Retention og Holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:31:05.902Z
translationReview: automatic
---

# Exchange Online: arkitektur, e-postflyt og drift

**Exchange Online** er Exchange-tjenesten i Microsoft 365 som drives av Microsoft. Den tilbyr postbokser, kalendere, kontakter, grupper, SMTP-transport og administrasjonsfunksjoner. Tenantadministratoren bestemmer over mottakere, domener, koblinger, regler, tillatelser og oppbevaring. Microsoft drifter derimot postboksserverne, databasekopiene, interne køer, oppdateringer og failover-prosesser ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Dermed ligner Exchange Online faglig på et eget Exchange-system, men ikke driftsmessig. En lokal administrator kan undersøke en køfil eller aktivere en databasekopi. I Exchange Online ser vedkommende i stedet hendelsene, statusene og konfigurasjonsobjektene som tjenesten tilbyr. Den viktigste ferdigheten er derfor å kartlegge en brukerklage til en tydelig bane: identitet, klienttilgang, mottakerobjekt, transport, filtrering, levering eller oppbevaring.

## Fra tenant til postboks

Tenantet utgjør den organisatoriske rammen. Der administrerer Exchange Online e-postaktiverte mottakere: bruker- og delte postbokser, rom- og utstyrspostbokser, distribusjonslister, Microsoft 365-grupper, kontakter og e-postbrukere. Mottakertypen avgjør om data lagres, hvordan levering skjer og hvilke tillatelser som er tilgjengelige ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

En brukerkonto i Microsoft Entra ID og en Exchange-postboks henger sammen, men er ikke det samme objektet. Lisensiering kan utløse klargjøring av en postboks. Exchange legger da til e-postrelaterte attributter og tjenester. Hvis en administrator fjerner en lisens eller sletter en konto, gjelder ulike oppbevarings- og slettefrister. For drift og offboarding må derfor identitetslivssyklusen, postbokslivssyklusen og compliance-oppbevaring planlegges samlet ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

For eksperter blir attributtenes opprinnelse viktig. I en ren skytenant administreres Exchange-egenskaper på nettet. Ved synkroniserte identiteter kan det lokale miljøet fortsatt være den autoritative kilden for bestemte mottakerattributter. Da vises en verdi i Exchange Online, men må endres lokalt og synkroniseres på nytt. Denne modellen hører hjemme i artikkelen [Exchange Hybrid](/kb/exchange-hybrid), fordi den ikke finnes uten katalogsynkronisering.

## Hvordan en innkommende melding når postboksen

Når mottakeren er forstått, kan e-postveien spores. Det offentlige MX-oppføringen for et domene peker normalt til Exchange Online Protection, EOP. EOP tar imot SMTP-forbindelsen, vurderer avsender og melding, bruker sikkerhets- og transportregler og overleverer en tillatt melding til Exchange Online. For lokale mottakere følger deretter levering til postboksen ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

**Accepted Domain** fastsetter hvordan Exchange Online behandler mottakerdomenet. Ved `Authoritative` forventer tjenesten alle gyldige mottakere i egen organisasjon og avviser ukjente adresser. `Internal Relay` tillater videresending av ukjente mottakere til et annet system. Denne innstillingen er bare fornuftig når neste hopp og mottakeroppløsningen er pålitelig planlagt; ellers oppstår manglende levering eller løkker ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

En intern melding blir ikke automatisk værende «på samme server». Exchange Online løser opp avsender og mottaker, kontrollerer regler og sikkerhetspolicyer og skriver transporthendelser. For administratoren er denne hendelseskjeden avgjørende: `Delivered` betyr at tjenesten har levert til målet; `Filtered`, `Failed`, `Pending` eller `Expanded` beskriver andre trinn. Message Trace gjør disse trinnene synlige, men erstatter ikke kontroll av målpostboksen eller en etterfølgende regel ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Utgående meldinger og koblinger

For utgående meldinger avgjøres det først om Exchange Online sender direkte til målsystemet eller bruker en konfigurert Outbound Connector. En kobling kan rute meldinger til egen infrastruktur, en partner eller en e-postgateway. Valget bygger blant annet på mottakerdomene, koblingsbetingelser og transportregler ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Inbound Connectors beskriver omvendt under hvilke betingelser Exchange Online stoler på et avsendende system. Typiske kriterier er kilde-IP eller et TLS-sertifikat. Disse opplysningene er sikkerhetsrelevante: Et for stort IP-område eller et unøyaktig kontrollert sertifikat kan få ekstern trafikk til å fremstå som intern partnertrafikk.

Hvis en ekstern e-postgateway står foran EOP, ser Microsoft først gatewayens IP-adresse. **Enhanced Filtering for Connectors** kan ta med informasjon om det opprinnelige hoppet i filtervurderingen. Funksjonen er ikke en generell «spamfilterbryter», men må passe til den faktiske banen, koblingene og IP-adressene som hoppes over ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

Ekspertspørsmålet her er: Hvilken motpart godtok faktisk en melding, hvilken identitet ble kontrollert for koblingen, og ved hvilket hopp fant den siste innholdsmessige filtreringen sted? Disse tre svarene hører hjemme i ethvert diagram over e-postflyt.

## Teknisk oppbygning sett fra administratorens ståsted

Exchange Online publiserer ingen serverliste som en tenantadministrator administrerer som en lokal farm. Tjenesten har likevel tydelig gjenkjennelige tekniske byggeklosser. De blir synlige gjennom protokoller og administrasjonsgrensesnitt.

| Byggekloss | Oppgave | Hva tenantadministratoren ser |
|---|---|---|
| Exchange Online Protection | SMTP-mottak, anti-malware, anti-spam og transportbehandling | Karantene, policyer, rapporter og Message Trace |
| Exchange-transport | Mottakeroppløsning, regler, ruting og levering | Koblinger, Accepted Domains, regler og hendelser |
| Postbokstjeneste | Lagring av e-post, kalender, kontakter og mapper | Postboksobjekter, kvoter, tillatelser og klienttilgang |
| Microsoft Entra ID | Bruker-, gruppe-, program- og påloggingsidentiteter | Kontoer, roller, Conditional Access og appregistreringer |
| Exchange Online PowerShell | Exchange-spesifikk administrasjon | Cmdlets, RBAC og reviderbare endringer |
| Microsoft Graph | REST-API for programmer og automatisering | OAuth-tillatelser, ressurser og throttling |

Teknologistakken i ytterkanten består dermed hovedsakelig av SMTP og TLS for e-posttransport samt HTTPS, OAuth, PowerShell og REST for klient- og administrasjonstilgang. De interne implementeringsdetaljene er bare relevante for kunden i den grad Microsoft dokumenterer dem som tjenesteadferd, grense eller diagnosegrensesnitt ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Åpne den interaktive Exchange Online-grafikken direkte</a>.
</iframe>

## Klienttilgang og moderne godkjenning

E-posttransporten ender i postboksen; brukerne får deretter tilgang via klientprotokoller. Outlook, Outlook på nettet, mobilklienter og programmer bruker HTTPS-baserte endepunkter. Autodiscover hjelper klienter med å finne riktig tjeneste. Påloggingen skjer via Microsoft Entra ID, mens Exchange kontrollerer tillatelsen i postboksen ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

Dette skiller to feil som ofte blandes sammen. Hvis påloggingen mislykkes i Entra, når klienten ofte aldri Exchange. Hvis tokenet er gyldig, kan Exchange likevel avvise tilgang på grunn av manglende rolle, postbokstillatelse, klientpolicy eller feil målpostboks. Påloggingslogger og Exchange-diagnostikk må derfor vurderes samtidig.

Programmer får fortrinnsvis tilgang via Microsoft Graph eller støttede Exchange-grensesnitt. En Graph-programtillatelse kan ha bred gyldighet; Exchange RBAC for Applications kan avgrense den tilgjengelige postboksflaten. Et gyldig OAuth-token er altså bare første trinn. Deretter kontrollerer ressurstjenesten hvilken handling som er tillatt på hvilken postboks ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Tillatelser og sporing av endringer

Exchange Online har egne administrasjonsroller. Entra-roller kan gi adgang til Exchange-administrasjon, men de faktiske Exchange-cmdletene og deres virkeområde bestemmes av Exchange-RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

I tillegg finnes postbokstillatelser som Full Access, Send As og Send on Behalf. De styrer ulike handlinger og bør ikke registreres som én felles «delegeringsrett». For programmer kommer OAuth- og Exchange-programroller i tillegg ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

For eksperter er endringenes opprinnelse like viktig som sluttstatusen. Revisjonslogger, Entra-påloggingslogger og konfigurasjonseksporter besvarer hvem som har endret en regel, en kobling eller en tillatelse. En nattlig eksport av sentrale e-postflytobjekter forenkler sammenligninger, men erstatter ikke en beskyttet revisjonskilde.

## Diagnostikk: først DNS, deretter transporthendelser

En analyse av e-postflyt begynner utenfor tenantet. MX viser hvilket system som tar imot e-post fra Internett. Deretter kontrolleres det med Message Trace om Exchange Online har sett den konkrete meldingen og hvordan den er behandlet.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) og [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) viser publisering og oppløsning. De sier ennå ingenting om EOP har godtatt meldingen eller om en postboks har mottatt den.

For neste trinn velges et snevert tidsrom med avsender og mottaker. Den samme Exchange Online PowerShell kjører under Windows og med `pwsh` på støttede Unix-systemer.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) oppretter den godkjente administrasjonsøkten. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) søker etter transporthendelser; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) avgrenser tidsvinduet. For trender brukes rapporter i tillegg, og for Microsoft-feil brukes Service Health. Ett enkelt grønt signal besvarer ikke alle tre spørsmålene ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Oppbevaring, sletting og gjenoppretting

Microsoft beskytter den løpende tjenesten med flere databasekopier, Shadow Redundancy og Safety Net. Disse mekanismene er for tjenestens tilgjengelighet og dataintegritet. De er ikke brukergrensesnittet for å gjenopprette en melding som er slettet ved et uhell ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

For bruker- og compliance-tilfeller brukes andre funksjoner: Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies og Holds. Virkningene deres overlapper, men har ulike formål. En oppbevaringsregel kan beskytte innhold mot endelig sletting; den gir ikke automatisk en separat sikkerhetskopi uavhengig av tenantet med fritt valgt gjenopprettingstidspunkt ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Et robust gjenopprettingskonsept angir derfor hvilke hendelser Microsofts tjenesteberedskap dekker, hvilket innhold som kan hentes tilbake via Exchange- eller Purview-oppbevaring, og hvilke krav som krever en uavhengig kopi. Gjenopprettingstester bør bruke konkrete tilfeller: enkeltmelding, mappe, postboks etter brukersletting, juridisk oppbevart element og tenantomfattende driftsforstyrrelse.

## Sikkerhet og typiske begrensninger

Exchange Online kombinerer flere sikkerhetsområder: e-post fra Internett, EOP, tenantkonfigurasjon, Entra-pålogging, postboksrettigheter og programmer. Beskyttelseseffekten avhenger av at den faktiske meldings- og påloggingsbanen stemmer overens med konfigurasjonen.

For e-postflyt betyr dette at MX, koblingsidentitet, Enhanced Filtering, SPF/DKIM/DMARC og transportregler må kontrolleres som en kjede. For klienttilgang er moderne godkjenning, Conditional Access, Exchange-RBAC og postbokstillatelser separate kontroller. For programmer kommer OAuth-samtykke og tillatt postboksområde i tillegg.

Det dypere administrasjonsspørsmålet er alltid det samme: Hvilket system tok avgjørelsen, hvilke inndata så det, og hvor er resultatet loggført? Uten disse tre opplysningene forblir selv en formelt korrekt policy vanskelig å kontrollere.

## Teknisk utvikling og bevisste avveininger

Exchange Online utviklet seg fra Microsofts hostede Exchange-tilbud og overtok mange konsepter fra serverproduktet: mottakere, postboksdatabaser, transport, DAG-er, Shadow Redundancy og Safety Net. Tjenesten automatiserer driften av denne infrastrukturen og gir tenantadministratorer et høyere administrasjonsnivå ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Gevinsten ligger i outsourcet plattformdrift, global tjenesteintegrasjon og standardiserte administrasjonsgrensesnitt. Prisen er mindre tilgang til enkeltservere, køer og databasekopier, samt sterkere avhengighet av publiserte funksjoner for diagnostikk, eksport og gjenoppretting. For eksperter er oppgaven derfor ikke å gjette den usynlige interne topologien, men å bruke de dokumenterte tenantkontrollene og tjenestesignalene fullt ut.

## Kilder

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
