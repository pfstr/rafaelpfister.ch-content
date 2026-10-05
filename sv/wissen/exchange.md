---
title: "Microsoft Exchange: produktfamilj och driftmodeller"
blatt: "exchange"
description: "Microsoft Exchange som produktfamilj: gemensamma begrepp och protokoll, skillnader mellan Exchange Online och Exchange Server samt Hybrid-driftens och Hybrid-e-postflödets uppgifter."
fakten:
  - label: Produktfamilj
    wert: Exchange Online och Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Kärnuppgifter
    wert: E-post, kalender, kontakter, adressbok och principer
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Molndrift
    wert: Exchange Online inom Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Egen drift
    wert: Exchange Server på Windows Server och Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Samexistens
    wert: Exchange Hybrid förbinder lokal organisation och Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: E-posttransport
    wert: SMTP, anslutningar, regler, köer och leverans
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Klientåtkomst
    wert: HTTPS, MAPI/HTTP, Outlook på webben och mobila klienter
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Mottagare
    wert: Postlådor, grupper, kontakter, e-postanvändare och resurser
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Kataloger
    wert: Active Directory lokalt, Microsoft Entra ID i molnet
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Administration
    wert: Exchange Admin Center, PowerShell och rollbaserade behörigheter
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Molndiagnostik
    wert: Message Trace, rapporter och Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Serverdiagnostik
    wert: Köer, Message Tracking, Health Sets och databaskopior
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: f76fa79338367d134f158edf1b05fbee051e68cffacc657f421ca0bbdd03dc2f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:16:23.416Z
translationReview: automatic
---

# Microsoft Exchange: produktfamilj och driftmodeller

Namnet **Microsoft Exchange** avser i dag två nära besläktade men olika driftade plattformar. I **Exchange Online** driver Microsoft servrarna, databaskopiorna och den interna transportinfrastrukturen. I **Exchange Server** ligger värdar, Active Directory, databaser, köer, certifikat och återställning under den egna organisationens ansvar. **Exchange Hybrid** förbinder båda organisationerna när postlådor eller funktioner är fördelade mellan båda sidorna ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Denna skillnad är utgångspunkten för varje vidare fråga. En postlåda kan finnas i molnet eller i det egna datacentret. Den synliga avsändaren, SMTP-domänen och adressboken kan ändå användas gemensamt. Först när postlådans plats, ursprunget för dess mottagarattribut och det faktiska meddelandeflödet är fastställda går det att undersöka leverans, behörigheter och fel på ett meningsfullt sätt.

## Klargör först: Var finns postlådan?

Exchange hanterar e-post, kalender, kontakter, uppgifter, adressboksobjekt och åtkomsträttigheter. För användare ser detta i stort sett likadant ut. För administratörer ändrar postlådans plats däremot nästan varje verktyg och ansvarsområde.

| Fråga | Exchange Online | Exchange On-Premises | Exchange Hybrid |
|---|---|---|---|
| Vem driver postlåde-servrarna? | Microsoft | den egna organisationen | båda sidor för sina respektive postlådor |
| Var redigeras mottagare? | Exchange Online och Entra ID | Exchange Server och Active Directory | skapas vanligtvis lokalt och synkroniseras till Entra ID; den exakta modellen måste dokumenteras |
| Var spåras ett meddelande? | Message Trace | Message Tracking Logs och köer | på båda sidor, sammankopplade genom tid, avsändare, mottagare och meddelande-ID:n |
| Vem kan aktivera databaskopior? | Microsoft | den egna Exchange-administrationen | respektive operatör för den berörda sidan |
| Vad förbinder båda sidor? | ej tillämpligt | ej tillämpligt | katalogsynkronisering, organisationsrelationer, OAuth, Autodiscover och SMTP-anslutningar |

Tabellen är endast en översikt. De fyra fördjupningarna behandlar driftmodellerna var för sig:

- [Exchange Online](/kb/exchange-online) förklarar klientobjekt, EOP, anslutningar, Message Trace, bevarande och driften av en molntjänst.
- [Exchange On-Premises](/kb/exchange-on-premises) följer transportpipelinen, ESE-databaser, DAG:ar, Active Directory och återställning i det egna datacentret.
- [Exchange Hybrid](/kb/exchange-hybrid) behandlar katalogsynkronisering, mottagarhantering, OAuth, organisationsrelationer, Autodiscover och Hybrid Configuration Wizard.
- [Hybrid-e-postflöde](/kb/hybrid-mailfluss) följer meddelanden mellan internet, Exchange Online, den lokala organisationen och en valfri e-postgateway.

## Gemensam teknisk struktur: Vad som förblir lika i alla Exchange-varianter

När driftplatsen har klarlagts är det värt att titta på den gemensamma fackmodellen. Exchange känner till **mottagare**, **meddelanden**, **postlådor**, **transportregler**, **domäner** och **anslutningar**. Dessa begrepp förekommer i molnet och på egna servrar, även om de bakomliggande systemen är olika tillgängliga ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

En mottagare är först och främst ett e-postaktiverat katalogobjekt. Det har adresser och en typ, till exempel användarpostlåda, delad postlåda, grupp, kontakt eller e-postanvändare. Objektet besvarar frågan **vem** en adress representerar och **vart** Exchange ska leverera. Postlådan lagrar därefter själva objekten. Därför kan ett felaktigt mottagarobjekt och en frisk postlåda finnas samtidigt, eller tvärtom.

Även meddelandeflödet följer på båda plattformarna samma grova steg: Exchange tar emot ett SMTP-meddelande, löser upp mottagarna, tillämpar transportregler och skyddsfunktioner, väljer nästa mål och levererar antingen till en postlåda eller till ett ytterligare SMTP-hopp. Den exakta implementeringen skiljer sig åt. På egna servrar kan administratören se köer och lokala spårningsloggar; i Exchange Online finns Message Trace, rapporter och Service Health tillgängliga ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Öppna den interaktiva Exchange-översikten direkt</a>.
</iframe>

## Från adress till meddelandeflöde

Den gemensamma SMTP-domänen leder ofta till det felaktiga antagandet att alla meddelanden tar samma väg. I själva verket avgör en kombination av DNS, Accepted Domains, mottagarobjekt, anslutningar och regler vart ett meddelande sedan går.

En **Accepted Domain** anger för Exchange hur en domän ska hanteras. För en auktoritativ domän förväntar sig Exchange alla giltiga mottagare i den egna katalogen. För en Internal-Relay-domän får okända mottagare skickas vidare till ett annat system. Denna inställning är därför inte en inventarierad utan en del av leveransbeslutet ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

**Anslutningar** anger därefter från vilka system Exchange tar emot meddelanden och till vilka system det skickar dem. I Exchange Server arbetar Receive- och Send Connectors med lokala bindningar, adressutrymmen, källservrar, smarta värdar och behörigheter. Exchange Online använder Inbound- och Outbound Connectors för relationer till den egna infrastrukturen eller till partner ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Först här blir hybridvarianter viktiga. Centralized Mail Transport, en föregående gateway eller en delad mottagardomän förändrar ytterligare hopp och ansvarsområden. De hör därför hemma i den egna artikeln [Hybrid-e-postflöde](/kb/hybrid-mailfluss), inte mellan identitets- eller klientämnen.

## Från inloggning till postlåda

Meddelandeflödet förklarar ännu inte hur Outlook hittar sin postlåda. För detta använder Exchange **Autodiscover**. En klient börjar med användarens identitet och fastställer utifrån den rätt tjänsteslutpunkt. Lokalt kan Active Directory Service Connection Points och DNS vara inblandade; i Exchange Online leder Microsoft 365-slutpunkter till molntjänsten ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Efter att slutpunkten har fastställts sker modern Outlook-åtkomst via HTTPS, särskilt med MAPI over HTTP. Outlook på webben, Exchange ActiveSync och olika API:er använder också HTTPS, men med egna applikationsprotokoll och behörigheter. En lyckad inloggning på Microsoft 365-portalen bevisar därför inte automatiskt att Autodiscover, Outlook-protokollet eller åtkomsten till den specifika postlådan fungerar ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

I hybriddrift tillkommer ytterligare ett beslut: Finns postlådan lokalt eller online? Autodiscover och mottagarattributen måste leda klienten till rätt sida. Först därefter spelar organisationsövergripande funktioner som ledig/upptagen eller postlådeflyttar en roll. Denna ordning behandlas steg för steg i artikeln [Exchange Hybrid](/kb/exchange-hybrid).

## Administration och behörigheter

När data- och åtkomstvägarna är förstådda följer frågan om vem som får ändra dem. Exchange använder rollbaserad åtkomstkontroll. Roller innehåller cmdlets och parametrar, rollgrupper eller rolltilldelningar kopplar dem till administratörer och scopes begränsar området. Exchange Online och Exchange Server har besläktade RBAC-koncept, men separata konfigurationer ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

I vardagen innebär detta: En Entra-administratörsroll, en Exchange-rollgrupp och en postlådebehörighet är inte samma sak. **Full Access** tillåter att en postlåda öppnas, **Send As** att skicka som mottagare och **Send on Behalf** att skicka synligt på uppdrag av någon. Ingen av dessa behörigheter förklarar ensam om ett program får åtkomst via Microsoft Graph eller EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Expertfrågan är därför inte ”Är användaren administratör?”, utan: Vilken identitet loggar in, vilken roll gäller i vilken Exchange-organisation, vilket objekt adresseras och vilken ytterligare postlåde- eller programbehörighet kontrolleras?

## Drift och felsökning börjar på rätt sida

En meningsfull diagnos börjar med tre uppgifter: **berörd användare eller mottagare, exakt tidpunkt och postlådans plats**. Därefter följs vägen i ordningen DNS respektive Autodiscover, inloggning, Exchange-slutpunkt, mottagarupplösning, transporthändelse och postlådeleverans.

För Exchange Online visar Message Trace och Microsoft 365 Service Health det tjänstetillstånd som kunden kan se. För Exchange Server tillkommer lokala köer, Message Tracking Logs, Health Sets, Event Logs och databaskopior. I hybriddrift sammanförs bevisen från båda sidor på en gemensam tidslinje ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

De fördjupande artiklarna innehåller vardera passande diagnostikblock för Windows/Unix och den officiella dokumentationen för de använda verktygen. Här räcker den viktigaste driftregeln: Fastställ först platsen och vägen, välj sedan verktyget.

## Datalagring, tillgänglighet och återställning

Skillnaden mellan moln och egen drift är tydligast vid återställning. Exchange Server lagrar postlådor i ESE-databaser med transaktionsloggar. Database Availability Groups replikerar databaskopior och möjliggör aktivering på andra servrar. Organisationen ansvarar dock fortfarande för backupstrategi, återställbarhet och Active Directory-beroendet ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online driver databas- och transportredundans som en del av tjänsten. Klientadministratörer arbetar i stället med borttagna objekt, Single Item Recovery, bevarande, Holds och vid behov externa säkerhetskopieringskrav. Microsofts tjänsteresiliens och en verksamhetsmässig bevaranderegel besvarar olika frågor: den ena skyddar den löpande tjänsten, den andra avgör vilket innehåll som bevaras efter radering eller för efterlevnadsändamål ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

I hybriddrift måste båda återställningsmodellerna dokumenteras parallellt. Dessutom krävs katalogsynkronisering, certifikat, OAuth-konfiguration och anslutningar för att återställa förbindelsen efter ett avbrott. Dessa konfigurationer innehåller visserligen inget postlådeinnehåll, men avgör om de två Exchange-organisationerna kan samarbeta igen.

## Teknisk utveckling

Exchange 4.0 lanserades 1996 som efterföljare till tidigare e-postplattformar från Microsoft. De första versionerna använde en egen katalog, MAPI och ESE-databasfamiljen; internetprotokoll blev gradvis viktigare. Med Exchange 2000 blev Active Directory och SMTP centrala byggstenar. Exchange 2007 introducerade tydliga serverroller och Exchange Management Shell, Exchange 2010 Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Parallellt utvecklade Microsoft värdbaserade Exchange-erbjudanden till Exchange Online. Hybrid uppstod därmed inte som en enskild produkt, utan som en förbindelse mellan två självständiga Exchange-organisationer. Exchange Server Subscription Edition fortsatte denna lokala utvecklingslinje från 2025 i Modern Lifecycle. Konkreta uppgifter om builds och uppdateringar kontrolleras före ändringar i den löpande uppdaterade Microsoft-dokumentationen ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Källor

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server-arkitektur](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
