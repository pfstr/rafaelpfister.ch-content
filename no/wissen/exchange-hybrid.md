---
title: "Exchange Hybrid: identitet, sameksistens og drift"
blatt: "exchange-hybrid"
description: "Exchange Hybrid forklart på en forståelig måte: forutsetninger, katalogsynkronisering, mottakerautoritet, Hybrid Configuration Wizard, OAuth, organisasjonsrelasjoner, Autodiscover, postkasseflyttinger og drift."
fakten:
  - label: Formål
    wert: Sameksistens mellom Exchange Server og Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Felles navneområde
    wert: Postkasser på begge sider kan bruke de samme SMTP-domenene
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Katalogsynkronisering
    wert: Microsoft Entra Connect Sync eller Cloud Sync i henhold til støttet modell
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Konfigurasjonsverktøy
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Lokal konfigurasjon
    wert: HybridConfiguration-objekt i Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Skynkonfigurasjon
    wert: Koblinger, organisasjonsrelasjoner og OAuth-tillit
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Mottakermodell
    wert: Remote Mailbox lokalt, Exchange Online-postkasse i skyen
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Sameksistensfunksjoner
    wert: Ledig/opptatt, MailTips, arkiv, søk og postkasseflyttinger avhengig av konfigurasjon
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Postkassemigrering
    wert: Mailbox Replication Service og migreringsendepunkter
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: E-posttransport
    wert: Sertifikatbasert SMTP/TLS mellom de to organisasjonene
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Mottakeradministrasjon
    wert: Exchange Management Tools eller støttet sky-SoA-overføring
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnose
    wert: HCW-logg, Entra-synkroniseringsstatus, OAuth-, organisasjons- og transportkonfigurasjon
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:21:28.905Z
translationReview: automatic
---

# Exchange Hybrid: identitet, sameksistens og drift

**Exchange Hybrid** kobler en lokal Exchange-organisasjon med Exchange Online. Brukere kan ha postkasser på begge sider og likevel bruke de samme SMTP-domenene, en felles adressebok og utvalgte funksjoner på tvers av organisasjonen. Hybrid er derfor mer enn et par med koblinger: Det kobler katalogdata, mottakere, autentisering, Autodiscover, kalenderfunksjoner, migrering og e-posttransport ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Avklar først tre spørsmål: **Hvor ligger postkassen?** **Hvor administreres mottakerobjektet?** Og **hvilken tjeneste utfører den forespurte handlingen?** Når disse tre svarene er fastslått, blir de mange hybridkomponentene en forståelig kjede.

## Hva Hybrid samler for brukerne

Uten Hybrid er den lokale Exchange-organisasjonen og Exchange Online to separate systemer. Hybrid legger en felles brukeropplevelse over dem. Postkasser kan bruke samme primære SMTP-domene. Adressebokinformasjon synkroniseres. Ledig/opptatt-oppslag og MailTips kan fungere på tvers av organisasjonen. Postkasser kan flyttes med støttede eksterne flyttinger ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Disse funksjonene deler imidlertid ikke ett felles datalager. En lokal postkasse forblir i en lokal ESE-database; en skypostkasse forblir i Exchange Online. Active Directory og Entra ID har hver sine katalogobjekter. Organisasjonsrelasjoner og OAuth tillater utvalgte oppslag over grensen. SMTP-koblinger transporterer meldinger. Det synlige «ene Exchange» oppstår gjennom samordnede forbindelser.

For administratorer følger en viktig regel av dette: En grønn e-postflyt beviser ikke at ledig/opptatt fungerer, og et vellykket ledig/opptatt-oppslag beviser ikke at en ekstern flytting er mulig. Hver funksjon har sin egen bane og sine egne bevis.

## Byggesteinene i riktig rekkefølge

En hybridimplementering starter med forutsetningene, ikke med veiviseren. Den lokale Exchange-organisasjonen må være på en støttet versjon. Offentlige navn, sertifikater, DNS, HTTPS- og SMTP-tilgjengelighet må være riktige. En Microsoft 365-leier med Exchange Online og støttet katalogsynkronisering kobler deretter identitetene ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Dette er grunnlaget for **Hybrid Configuration Wizard**, HCW. Den leser den ønskede konfigurasjonen, skriver et `HybridConfiguration`-objekt til det lokale Active Directory og konfigurerer riktige innstillinger lokalt og i Exchange Online. Dette kan omfatte organisasjonsrelasjoner, OAuth, Intra-Organization Connectors og transportkoblinger ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Byggestein | Hovedoppgave | Hva som først kontrolleres ved feil |
|---|---|---|
| Active Directory | lokale bruker- og Exchange-attributter | Objekt, mottakertype, proxyadresser og endringstidspunkt |
| Entra-synkronisering | overfører støttede identitets- og mottakerattributter | Eksportfeil, synkroniseringsstatus og skyobjekt |
| Exchange Online | skypostkasse og skykonfigurasjon | Mottakertype, lisens, postkassetilstand og RBAC |
| HCW-konfigurasjon | samordner de to Exchange-organisasjonene | HCW-logg, valgparametere og objekter som senere er endret |
| Organisasjonsrelasjon og OAuth | funksjoner på tvers av organisasjoner | Mål-URI, Autodiscover, sertifikater og tokenflyt |
| SMTP-koblinger | meldinger mellom begge sider | Sertifikatnavn, kilde-/målvert, TLS og Message Trace |

Tabellen viser også hvorfor «kjør HCW på nytt» ikke er en universell reparasjon. Veiviseren kan samordne dokumenterte hybridobjekter på nytt. Den reparerer ikke en feil DNS-sone, en blokkert brannmursti eller et feil vedlikeholdt mottakerobjekt.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Åpne interaktiv Exchange Hybrid-grafikk direkte</a>.
</iframe>

## Teknologistakk: protokoller og administrasjonsverktøy

Hybrid er ikke en ekstra Exchange-serverprosess, men en kobling mellom eksisterende systemer. Active Directory og Entra ID har identiteter og mottakerattributter. Entra-synkronisering overfører støttede verdier. HTTPS brukes til Autodiscover, ledig/opptatt, OAuth-beskyttede tjenestekall og postkasseflyttinger. SMTP med TLS transporterer meldinger. PowerShell, Exchange Admin Center og HCW administrerer de involverte objektene ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Denne inndelingen angir også rekkefølgen ved feil. En objektfeil søkes i katalog og synkronisering, et kalenderproblem i HTTPS-/OAuth-banen, og et e-postproblem i SMTP og koblinger. Slik forblir verktøykassen knyttet til den berørte funksjonen.

## Katalogsynkronisering og mottakerautoritet

Så snart plattformene er koblet sammen, blir opprinnelsen til mottakerdataene det viktigste driftsspørsmålet. I klassiske hybridmiljøer opprettes en bruker i det lokale Active Directory. Exchange-verktøy skriver de e-postrelaterte attributtene. Entra Connect synkroniserer objektet til skyen, der Exchange Online klargjør det tilsvarende skyobjektet og eventuelt en postkasse ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

En **Remote Mailbox** er et lokalt e-postaktivert objekt som peker til en Exchange Online-postkasse. Attributter som `remoteRoutingAddress`, `proxyAddresses` og mottakertypen hjelper den lokale organisasjonen med å føre meldinger og administrasjon til skysiden. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) oppretter eller aktiverer denne lokale representasjonen; skypostkassen oppstår først gjennom synkronisering og lisensiering.

Det vanlige administrasjonsspørsmålet er altså: Hvor må jeg endre denne verdien? Ekspertspørsmålet er: Hvilket system er autoritativt for **dette enkelte attributtet**, og hvilken synkroniseringskjøring overfører det? En skyportal kan vise en synkronisert verdi uten å kunne redigere den permanent.

Microsoft støtter scenarioer der bare Exchange Management Tools beholdes for lokale mottakerattributter. For bestemte miljøer finnes det dessuten en prosedyre for å overføre administrasjonen av Exchange-attributter til skyen. Dette er ulike driftsmodeller med forutsetninger; å slå av den siste serveren alene flytter ikke dataautoriteten ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover og klientens bane

Når mottakerne er riktige, må en klient finne postkassens plassering. Autodiscover besvarer dette spørsmålet. Lokale Exchange-endepunkter kan omdirigere en klient for en skypostkasse til Exchange Online; skyendepunkter leverer innstillingene for nettpostkassen ([Autodiscover in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Et Hybrid-Autodiscover-problem viser seg derfor ofte som feil sted: Brukeren kan i utgangspunktet logge på, men havner på det lokale endepunktet, får en uventet omdirigering eller får innstillinger for en postkasse som ikke lenger finnes. DNS, SCP-er, virtuelle kataloger, sertifikater og mottakerattributter kontrolleres i denne rekkefølgen.

Først etter at klienten har kommet frem til riktig postkassetjeneste, er spørsmål om protokoll og tillatelser relevante. Dermed blir diagnosen forståelig: først finne, så logge på, deretter autorisere.

## Ledig/opptatt og andre funksjoner på tvers av organisasjoner

En felles adressebok er ikke nok for kalenderoppslag. Ledig/opptatt krever organisasjonsrelasjoner, tilgjengelig Autodiscover og en fungerende tillits- eller OAuth-konfigurasjon. Exchange henter informasjon fra den andre siden i stedet for å kopiere kalenderdata fullstendig til sitt eget system ([Sharing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

Det samme grunnmønsteret gjelder andre hybridfunksjoner: En lokal komponent sender en forespørsel, motparten autentiserer den, autoriserer handlingen og leverer et begrenset resultat. Ved feilsøking registreres derfor kildepostkasse, målpostkasse, retning og endepunkt. «Ledig/opptatt fungerer ikke» er for bredt uten disse opplysningene.

For eksperter blir tokener og mål-URI-er relevante. HCW konfigurerer relasjoner på tvers av organisasjoner, men sertifikatbytter, manuelle endringer eller utdaterte endepunkter kan forstyrre den senere driften. Konfigurasjonen på begge sider eksporteres alltid sammen.

## OAuth mellom Exchange-organisasjonene

Når det er klart hvilke oppslag på tvers av organisasjoner som foretas, kan autentiseringen av dem settes i sammenheng. Exchange kan bruke OAuth slik at den ene organisasjonen identifiserer et tjenestekall overfor den andre. Dette gjelder hybridfunksjoner som tilgjengelighet på tvers av organisasjoner og utvalgte arkiv-, søke- eller migreringsoperasjoner; den nøyaktige bruken avhenger av versjon og konfigurasjon ([Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

Tokenflyten er ikke en erstatning for SMTP-TLS. OAuth beskytter applikasjonskall, mens hybrid e-posttransport bruker egne koblinger og sertifikatkontroller. Dette skillet forhindrer spranget som forvirrer i mange forklaringer: Først bestemmes funksjonen, deretter protokollen og først til slutt autentiseringen.

For eksperter hører AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector og Organization Relationship til ett samlet kontrollbilde. Ett enkelt objekt kan være syntaktisk til stede, mens sertifikat, realm eller mål-URI ikke lenger passer til motparten.

## Hybrid Modern Authentication er et eget klienttema

**Hybrid Modern Authentication**, HMA, kommer først nå, fordi den ikke forklarer e-posttransporten eller mottakersynkroniseringen. HMA gjør det mulig for støttede lokale Exchange- og Skype for Business-ressurser å bruke Microsoft Entra ID for moderne klientautentisering. Klienten får et Entra-token og bruker det mot den lokale tjenesten ([Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

Dermed utvider HMA klienttilgangen med en skyavhengighet. Entra-tilgjengelighet, publiserte URL-er, registrerte Service Principal Names og den lokale Exchange-konfigurasjonen må passe sammen. En fungerende hybrid e-postflyt sier ingenting om denne tokenbanen.

Eksperter behandler derfor HMA i et eget runbook med støttede versjoner, unntak, utrullingsgrupper og tilbakefallsplan. Funksjonen kobles ikke tilfeldig til et rutealternativ.

## Postkasseflyttinger

Sameksistens etableres ofte for å flytte postkasser trinnvis. En ekstern flytting kopierer postkassedata via Mailbox Replication Service, synkroniserer endringer fortløpende og bytter postkassen kontrollert til målsiden. Mottakerattributter og ruting følger med eller justeres ([Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

For avanserte administratorer består prosessen av forberedelse, start, synkronisering, fullføring og etterkontroll. Før fullføring kontrolleres datamengde, feilaktige elementer, delegeringer, arkiver, klienttilgang og e-postflyt. Etter fullføring må Autodiscover, lisens, måladresse og det lokale Remote Mailbox-objektet passe sammen.

Eksperter planlegger batchstørrelser, nettverksgjennomstrømning, MRS-begrensning, Bad Item Limits, store objekter og tilbakeflytting. Den tekniske fremdriftsverdien alene er ikke en godkjenning; brukertilgang, stedfortredere, mobile klienter og funksjoner på tvers av organisasjoner hører også med.

## Hybrid e-postflyt forblir en egen bane

Hybrid krever SMTP mellom lokal organisasjon og Exchange Online. Denne e-postbanen bruker koblinger, TLS og sertifikater. Den er viktig nok for en egen artikkel fordi Internett-e-post, Centralized Mail Transport, e-postgatewayer og delte domener utgjør flere varianter ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Artikkelen [Hybrid e-postflyt](/kb/hybrid-mailfluss) starter med en konkret melding og følger hvert hopp. Først der sammenlignes Centralized Mail Transport, egress-IP, filtreringssted og ytterligere køer. Denne artikkelen konsentrerer seg om identitet og sameksistens.

## Sikkerhet og drift

Hybrid utvider systemene som er tilgjengelige. Offentlige HTTPS- og SMTP-endepunkter, sertifikater, Entra-synkronisering, privilegerte kontoer og tillitsobjekter på tvers av organisasjoner må inventariseres samlet. HCW krever omfattende rettigheter på begge sider; bruk og logger må beskyttes og lagres på en sporbar måte ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

I hverdagen bør hver hybridfunksjon ha en eier og en test: mottakersynkronisering, ledig/opptatt i begge retninger, ekstern flytting, Autodiscover og SMTP i begge retninger. En regelmessig syntetisk test oppdager utløpte sertifikater eller stille endrede endepunkter tidligere enn et migreringsprosjekt.

Ved feil hjelper en felles tidslinje. Entra-synkroniseringshendelser, HCW-logg, Exchange-hendelseslogger, OAuth-tester, meldingssporing og Message Trace samles ikke vilkårlig, men knyttes til den berørte funksjonen. Dette forkorter diagnosen og hindrer at en vellykket test av en annen funksjon misforstås som bevis.

## Sikkerhetskopiering, gjenoppbygging og avvikling

Postkassedataene beskyttes på siden der de ligger: lokale databaser med lokal gjenoppretting, skypostkasser med funksjoner i Exchange Online og Purview. I tillegg må forbindelsen kunne gjenopprettes. Dette omfatter det lokale HybridConfiguration-objektet, sertifikater og private nøkler, koblings- og organisasjonskonfigurasjon, Entra-synkroniseringsregler samt dokumenterte HCW-beslutninger.

En gjenoppbygging begynner med identitet og navneoppløsning, deretter følger HTTPS- og SMTP-tilgjengelighet, HCW-konfigurasjon og funksjonstester. Veiviseren kan opprette konfigurasjon på nytt, men uten riktige sertifikater, DNS og mottakerobjekter oppstår det ikke et fungerende helhetssystem.

Ved avvikling avklares først hvilke hybridfunksjoner som fortsatt brukes. Microsoft skiller mellom en gjenværende server, rene Management Tools og overføring av Exchange-attributtadministrasjonen til skyen. Først etter denne beslutningen fjernes koblinger, organisasjonsrelasjoner, endepunkter og servere kontrollert ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Teknisk utvikling og begrensninger

Hybrid oppsto med Exchange Online som en måte å utvide lokale organisasjoner kontrollert til skytjenesten på. Tidligere generasjoner brukte i større grad Federation Trusts; nyere Exchange-versjoner og HCW-prosesser bruker OAuth og Intra-Organization Connectors for mange funksjoner på tvers av organisasjoner ([Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

Modellen er kraftig fordi den tillater migrering og varig sameksistens. Den er krevende fordi begge Exchange-organisasjonene og forbindelsen mellom dem må driftes. Den som ikke lenger har lokale postkasser etter en migrering, bør derfor bevisst avgjøre hvilken administrasjons- eller sameksistensfunksjon Hybrid fortsatt rettferdiggjør.

## Kilder

- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools)
- [Microsoft Learn – Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)
- [Microsoft Learn – Autodiscover service](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Sharing in Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)
- [Microsoft Learn – Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)
- [Microsoft Learn – Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)
- [Microsoft Learn – Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)
- [Microsoft Learn – Cross-tenant mailbox migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants)
