---
title: "Microsoft Exchange: Produktfamilie og driftsmodeller"
blatt: "exchange"
description: "Microsoft Exchange som produktfamilie: felles begreper og protokoller, forskjeller mellom Exchange Online og Exchange Server samt oppgavene til hybriddrift og hybrid e-postflyt."
fakten:
  - label: Produktfamilie
    wert: Exchange Online og Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Kjerneoppgaver
    wert: E-post, kalender, kontakter, adressebok og policyer
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Skydrift
    wert: Exchange Online i Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Egendrift
    wert: Exchange på Windows Server og Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Sameksistens
    wert: Exchange Hybrid kobler lokal organisasjon og Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: E-posttransport
    wert: SMTP, koblinger, regler, køer og levering
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Klienttilgang
    wert: HTTPS, MAPI/HTTP, Outlook på nettet og mobile klienter
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Mottakere
    wert: Postbokser, grupper, kontakter, e-postbrukere og ressurser
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Kataloger
    wert: Active Directory lokalt, Microsoft Entra ID i skyen
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Administrasjon
    wert: Exchange Admin Center, PowerShell og rollebaserte rettigheter
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Skydiagnostikk
    wert: Message Trace, rapporter og Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Serverdiagnostikk
    wert: Køer, Message Tracking, Health Sets og databasekopier
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: 120d6e74efed65ae14cfc80b5e66597bd4ea19799d6b0c4d9e22fce0152b8d30
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:31:46.060Z
translationReview: automatic
---

# Microsoft Exchange: Produktfamilie og driftsmodeller

Navnet **Microsoft Exchange** betegner i dag to nært beslektede, men ulikt drevne plattformer. Med **Exchange Online** drifter Microsoft serverne, databasekopiene og den interne transportinfrastrukturen. Med **Exchange Server** er verter, Active Directory, databaser, køer, sertifikater og gjenoppretting eget ansvar. **Exchange Hybrid** kobler begge organisasjonene når postbokser eller funksjoner er fordelt på begge sider ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Dette skillet er utgangspunktet for alle videre spørsmål. En postboks kan ligge i skyen eller i eget datasenter. Den synlige avsenderen, SMTP-domenet og adresseboken kan likevel brukes felles. Først når plasseringen av postboksen, opprinnelsen til mottakerattributtene og den faktiske meldingsveien er fastslått, kan levering, rettigheter og feil undersøkes på en meningsfull måte.

## Avklar først: Hvor ligger postboksen?

Exchange administrerer e-post, kalender, kontakter, oppgaver, adressebokobjekter og tilgangsrettigheter. For brukerne ser dette stort sett likt ut. For administratorer endrer plasseringen av postboksen derimot nesten alle verktøy og ansvarsområder.

| Spørsmål | Exchange Online | Exchange On-Premises | Exchange Hybrid |
|---|---|---|---|
| Hvem drifter postboksserverne? | Microsoft | egen organisasjon | begge sider for sine respektive postbokser |
| Hvor redigeres mottakere? | Exchange Online og Entra ID | Exchange Server og Active Directory | vanligvis opprettet lokalt og synkronisert til Entra ID; den nøyaktige modellen må dokumenteres |
| Hvor spores en melding? | Message Trace | Message Tracking Logs og køer | på begge sider, koblet sammen via tid, avsender, mottaker og meldings-ID-er |
| Hvem kan aktivere databasekopier? | Microsoft | egen Exchange-administrasjon | driftsansvarlig for den berørte siden |
| Hva kobler begge sider? | ikke aktuelt | ikke aktuelt | katalogsynkronisering, organisasjonsrelasjoner, OAuth, Autodiscover og SMTP-koblinger |

Tabellen er bare oversikten. De fire fordypningene behandler driftsmodellene separat:

- [Exchange Online](/kb/exchange-online) forklarer leierobjekter, EOP, koblinger, Message Trace, oppbevaring og drift av en skytjeneste.
- [Exchange On-Premises](/kb/exchange-on-premises) følger transportpipelinen, ESE-databaser, DAG-er, Active Directory og gjenoppretting i eget datasenter.
- [Exchange Hybrid](/kb/exchange-hybrid) behandler katalogsynkronisering, mottakeradministrasjon, OAuth, organisasjonsrelasjoner, Autodiscover og Hybrid Configuration Wizard.
- [Hybrid e-postflyt](/kb/hybrid-mailfluss) følger meldinger mellom Internett, Exchange Online, lokal organisasjon og en valgfri e-postgateway.

## Felles teknisk struktur: Hva som er likt for alle Exchange-varianter

Når driftsstedet er avklart, er det nyttig å se på den felles faglige modellen. Exchange kjenner **mottakere**, **meldinger**, **postbokser**, **transportregler**, **domener** og **koblinger**. Disse begrepene forekommer i skyen og på egne servere, selv om de underliggende systemene er ulikt tilgjengelige ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

En mottaker er først og fremst et e-postaktivert katalogobjekt. Det har adresser og en type, for eksempel brukerpostboks, delt postboks, gruppe, kontakt eller e-postbruker. Objektet besvarer spørsmålet om **hvem** en adresse representerer og **hvor** Exchange skal levere. Postboksen lagrer deretter de faktiske elementene. Derfor kan et feilaktig mottakerobjekt og en frisk postboks eksistere samtidig, eller omvendt.

Meldingsveien følger også de samme grove trinnene på begge plattformer: Exchange mottar en SMTP-melding, løser opp mottakerne, bruker transportregler og beskyttelsesfunksjoner, velger neste mål og leverer enten til en postboks eller til et videre SMTP-hopp. Den nøyaktige implementeringen er forskjellig. På egne servere kan administratoren se køer og lokale sporingslogger; i Exchange Online finnes Message Trace, rapporter og Service Health ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Åpne den interaktive Exchange-oversikten direkte</a>.
</iframe>

## Fra adresse til meldingsvei

Det felles SMTP-domenet fører ofte til den feilaktige antakelsen at alle meldinger går samme vei. I praksis avgjør en kombinasjon av DNS, Accepted Domains, mottakerobjekt, koblinger og regler hvor en melding skal videre.

Et **Accepted Domain** forteller Exchange hvordan et domene behandles. For et autoritativt domene forventer Exchange alle gyldige mottakere i sin egen katalog. For et Internal-Relay-domene kan ukjente mottakere sendes videre til et annet system. Denne innstillingen er derfor ikke bare en inventarlinje, men en del av leveringsbeslutningen ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

**Koblinger** bestemmer deretter hvilke systemer Exchange mottar meldinger fra, og hvilke systemer det sender dem til. I Exchange Server arbeider Receive og Send Connectors med lokale bindinger, adresserom, kildeservere, smarthoster og rettigheter. Exchange Online bruker Inbound og Outbound Connectors for forbindelser til egen infrastruktur eller partnere ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Først her blir hybridvarianter viktige. Centralized Mail Transport, en foranliggende gateway eller et delt mottakerdomene endrer ytterligere hopp og ansvarsområder. De hører derfor hjemme i den egne artikkelen [Hybrid e-postflyt](/kb/hybrid-mailfluss), ikke mellom identitets- eller klienttemaer.

## Fra pålogging til postboks

Meldingsveien forklarer ikke ennå hvordan Outlook finner postboksen sin. Til det bruker Exchange **Autodiscover**. En klient starter med brukerens identitet og finner derfra det passende tjenesteendepunktet. Lokalt kan Active Directory Service Connection Points og DNS være involvert; i Exchange Online leder Microsoft 365-endepunkter til skytjenesten ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Etter at endepunktet er funnet, bruker moderne Outlook-tilgang HTTPS, særlig med MAPI over HTTP. Outlook på nettet, Exchange ActiveSync og ulike API-er bruker også HTTPS, men med egne applikasjonsprotokoller og rettigheter. En vellykket pålogging i Microsoft 365-portalen beviser derfor ikke automatisk at Autodiscover, Outlook-protokollen eller tilgangen til den konkrete postboksen fungerer ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

I hybriddrift kommer en ytterligere avgjørelse: Ligger postboksen lokalt eller på nett? Autodiscover og mottakerattributtene må lede klienten til riktig side. Først deretter spiller funksjoner på tvers av organisasjoner, som ledig/opptatt eller postboksflyttinger, en rolle. Denne rekkefølgen behandles trinn for trinn i artikkelen [Exchange Hybrid](/kb/exchange-hybrid).

## Administrasjon og rettigheter

Når data- og tilgangsveier er forstått, følger spørsmålet om hvem som kan endre dem. Exchange bruker rollebasert tilgangskontroll. Roller inneholder cmdlets og parametere, rollegrupper eller rolletildelinger kobler dem til administratorer, og omfang begrenser området. Exchange Online og Exchange Server har beslektede RBAC-konsepter, men separate konfigurasjoner ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

I praksis betyr det: En Entra-administratorrolle, en Exchange-rollegruppe og en postbokstillatelse er ikke det samme. **Full Access** tillater åpning av en postboks, **Send As** sending som mottaker og **Send on Behalf** synlig sending på vegne av. Ingen av disse tillatelsene forklarer alene om en applikasjon kan få tilgang via Microsoft Graph eller EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Ekspertspørsmålet er derfor ikke «Er brukeren administrator?», men: Hvilken identitet logger på, hvilken rolle gjelder i hvilken Exchange-organisasjon, hvilket objekt adresseres, og hvilken ytterligere postboks- eller applikasjonstillatelse kontrolleres?

## Drift og feilsøking starter på riktig side

En fornuftig diagnose starter med tre opplysninger: **berørt bruker eller mottaker, nøyaktig tidspunkt og plasseringen av postboksen**. Deretter følges banen i rekkefølgen DNS eller Autodiscover, pålogging, Exchange-endepunkt, mottakeroppløsning, transporthendelse og postbokslevering.

For Exchange Online gir Message Trace og Microsoft 365 Service Health kundesynlig tjenestestatus. For Exchange Server kommer lokale køer, Message Tracking Logs, Health Sets, Event Logs og databasekopier i tillegg. I hybriddrift samles bevisene fra begge sider på en felles tidslinje ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

De fordypende artiklene inneholder passende Windows-/Unix-diagnoseblokker og den offisielle dokumentasjonen for verktøyene som brukes. Her er den viktigste driftsregelen tilstrekkelig: Bestem først stedet og veien, velg deretter verktøyet.

## Datalagring, tilgjengelighet og gjenoppretting

Forskjellen mellom sky og egendrift er tydeligst ved gjenoppretting. Exchange Server lagrer postbokser i ESE-databaser med transaksjonslogger. Database Availability Groups replikerer databasekopier og muliggjør aktivering på andre servere. Organisasjonen er likevel ansvarlig for sikkerhetskopieringsstrategi, gjenopprettbarhet og Active Directory-avhengigheten ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online drifter database- og transportredundans som en del av tjenesten. Leieradministratorer arbeider i stedet med slettede elementer, Single Item Recovery, oppbevaring, holds og eventuelt eksterne krav til sikkerhetskopiering. Microsofts tjenestemotstandskraft og en faglig oppbevaringsregel besvarer ulike spørsmål: Den ene beskytter den løpende tjenesten, den andre bestemmer hvilket innhold som beholdes etter sletting eller av compliance-hensyn ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

I hybriddrift må begge gjenopprettingsmodellene dokumenteres side om side. I tillegg trengs katalogsynkronisering, sertifikater, OAuth-konfigurasjon og koblinger for å gjenopprette forbindelsen etter et utfall. Disse konfigurasjonene inneholder riktignok ikke postboksinnhold, men avgjør om de to Exchange-organisasjonene samarbeider igjen.

## Teknisk utvikling

Exchange 4.0 kom i 1996 som etterfølger til tidligere Microsoft-e-postplattformer. De første versjonene brukte en egen katalog, MAPI og ESE-databasefamilien; Internettprotokoller ble gradvis viktigere. Med Exchange 2000 ble Active Directory og SMTP sentrale byggesteiner. Exchange 2007 innførte tydelige serverroller og Exchange Management Shell, og Exchange 2010 Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Parallelt utviklet Microsoft hostede Exchange-tilbud til Exchange Online. Dermed oppstod Hybrid ikke som et enkelt produkt, men som forbindelsen mellom to selvstendige Exchange-organisasjoner. Exchange Server Subscription Edition videreførte denne lokale utviklingslinjen fra 2025 i Modern Lifecycle. Konkrete opplysninger om bygg og oppdateringer kontrolleres før endringer i Microsoft-dokumentasjonen som vedlikeholdes løpende ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Kilder

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server-arkitektur](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange-hybriddistribusjoner](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Mottakere](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – E-postflyt og transportpipelinen](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Spor en e-postmelding](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Vanlige spørsmål om Message Trace](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Accepted Domains i Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Accepted Domains i Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Koblinger på Exchange-servere](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Konfigurer koblinger for å rute e-post](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Protokollarkitektur for klienttilgang](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory i Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Tillatelser i Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Tillatelser i Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Administrer tillatelser for mottakere i Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – Serverhelse og ytelse](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Database Availability Groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Sikkerhetskopiering, gjenoppretting og katastrofegjenoppretting](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Dataresiliens for Exchange Online](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Oppbevaringspolicyer for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – En kort tidshistorie](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Omdesign av transport i Exchange Server 2007](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Versjonsmerknader for Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
