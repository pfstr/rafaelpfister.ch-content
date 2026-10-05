---
title: "Exchange Hybrid: identitet, samexistens och drift"
blatt: "exchange-hybrid"
description: "Exchange Hybrid förklarat på ett begripligt sätt: förutsättningar, katalogsynkronisering, mottagarauktoritet, Hybrid Configuration Wizard, OAuth, organisationsrelationer, Autodiscover, postlådeöverflyttningar och drift."
fakten:
  - label: Syfte
    wert: Samexistens mellan Exchange Server och Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Gemensam namnrymd
    wert: Postlådor på båda sidor kan använda samma SMTP-domäner
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Katalogsynkronisering
    wert: Microsoft Entra Connect Sync eller Cloud Sync enligt en modell som stöds
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Konfigurationsverktyg
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Lokal konfiguration
    wert: HybridConfiguration-objekt i Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Molnkonfiguration
    wert: Kopplingar, organisationsrelationer och OAuth-förtroende
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Mottagarmodell
    wert: Remote Mailbox lokalt, Exchange Online-postlåda i molnet
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Samexistensfunktioner
    wert: Ledig/upptagen, MailTips, arkiv, sökning och postlådeöverflyttningar beroende på konfiguration
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Postlådemigrering
    wert: Mailbox Replication Service och migreringsslutpunkter
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: E-posttransport
    wert: Certifikatbaserad SMTP/TLS mellan båda organisationerna
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Mottagarhantering
    wert: Exchange Management Tools eller stödd överföring av Cloud-SoA
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnostik
    wert: HCW-logg, Entra-synkroniseringsstatus, OAuth-, organisations- och transportkonfiguration
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:20:45.016Z
translationReview: automatic
---

# Exchange Hybrid: identitet, samexistens och drift

**Exchange Hybrid** kopplar samman en lokal Exchange-organisation med Exchange Online. Användare kan ha postlådor på båda sidor och ändå använda samma SMTP-domäner, en gemensam adressbok och utvalda organisationsövergripande funktioner. Hybrid är därför mer än ett par kopplingar: det sammanför katalogdata, mottagare, autentisering, Autodiscover, kalenderfunktioner, migrering och e-posttransport ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Klargör först tre frågor: **Var finns postlådan?** **Var hanteras dess mottagarobjekt?** Och **vilken tjänst utför den begärda åtgärden?** När dessa tre svar är fastställda blir de många hybridkomponenterna en begriplig kedja.

## Vad Hybrid sammanför för användare

Utan Hybrid är den lokala Exchange-organisationen och Exchange Online två separata system. Hybrid lägger en gemensam användarupplevelse ovanpå dem. Postlådor kan använda samma primära SMTP-domän. Adressboksinformation synkroniseras. Ledig/upptagen-frågor och MailTips kan fungera över organisationsgränserna. Postlådor kan flyttas med Remote Moves som stöds ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Dessa funktioner delar dock inte ett enda gemensamt datalager. En lokal postlåda finns kvar i en lokal ESE-databas; en molnpostlåda finns kvar i Exchange Online. Active Directory och Entra ID håller var sitt katalogobjekt. Organisationsrelationer och OAuth tillåter utvalda frågor över gränsen. SMTP-kopplingar transporterar meddelanden. Det synliga ”enda Exchange” uppstår genom samordnade anslutningar.

För administratörer följer en viktig regel: Ett fungerande e-postflöde bevisar inte att ledig/upptagen fungerar, och en lyckad ledig/upptagen-fråga bevisar inte att en Remote Move är möjlig. Varje funktion har sin egen väg och sina egna belägg.

## Byggstenarna i lämplig ordning

En hybriddistribution börjar med sina förutsättningar, inte med guiden. Den lokala Exchange-organisationen måste ha en version som stöds. Offentliga namn, certifikat, DNS, HTTPS- och SMTP-åtkomst måste stämma. En Microsoft 365-klientorganisation med Exchange Online och en katalogsynkronisering som stöds kopplar sedan samman identiteterna ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Därefter kommer **Hybrid Configuration Wizard**, HCW. Den läser den önskade konfigurationen, skriver ett `HybridConfiguration`-objekt till det lokala Active Directory och konfigurerar lämpliga inställningar lokalt och i Exchange Online. Detta kan omfatta organisationsrelationer, OAuth, Intra-Organization Connectors och transportkopplingar ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Byggsten | Grunduppgift | Vad som först kontrolleras vid störning |
|---|---|---|
| Active Directory | lokala användar- och Exchange-attribut | Objekt, mottagartyp, proxyadresser och ändringstidpunkt |
| Entra-synkronisering | överför identitets- och mottagarattribut som stöds | Exportfel, synkroniseringsstatus och molnobjekt |
| Exchange Online | molnpostlåda och molnkonfiguration | Mottagartyp, licens, postlådestatus och RBAC |
| HCW-konfiguration | samordnar de två Exchange-organisationerna | HCW-logg, urvalsparametrar och objekt som senare ändrats |
| Organisationsrelation och OAuth | organisationsövergripande funktioner | Mål-URI, Autodiscover, certifikat och tokenflöde |
| SMTP-kopplingar | meddelanden mellan båda sidor | Certifikatnamn, käll-/målvärd, TLS och Message Trace |

Tabellen visar också varför ”att köra HCW igen” inte är en universell reparation. Guiden kan samordna dokumenterade hybridobjekt på nytt. Den reparerar varken en felaktig DNS-zon, en blockerad brandvägssökväg eller ett felaktigt underhållet mottagarobjekt.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Öppna den interaktiva Exchange Hybrid-grafiken direkt</a>.
</iframe>

## Teknikstack: protokoll och administrationsverktyg

Hybrid är inte en ytterligare Exchange-serverprocess, utan en koppling av befintliga system. Active Directory och Entra ID håller identiteter och mottagarattribut. Entra-synkronisering överför värden som stöds. HTTPS bär Autodiscover, ledig/upptagen, OAuth-skyddade tjänstanrop och postlådeöverflyttningar. SMTP med TLS transporterar meddelanden. PowerShell, Exchange Admin Center och HCW hanterar de berörda objekten ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Denna uppdelning bestämmer också ordningen vid störningar. Ett objektfel söks i katalogen och synkroniseringen, ett kalenderproblem i HTTPS-/OAuth-vägen, ett e-postproblem i SMTP och kopplingarna. Därmed förblir verktygslådan knuten till den berörda funktionen.

## Katalogsynkronisering och mottagarauktoritet

När plattformarna har kopplats samman blir mottagardatas ursprung den viktigaste driftfrågan. I klassiska hybridmiljöer skapas en användare i det lokala Active Directory. Exchange-verktyg skriver de e-postrelaterade attributen. Entra Connect synkroniserar objektet till molnet, där Exchange Online tillhandahåller motsvarande molnobjekt och vid behov en postlåda ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

En **Remote Mailbox** är då ett lokalt e-postaktiverat objekt som pekar på en Exchange Online-postlåda. Attribut som `remoteRoutingAddress`, `proxyAddresses` och mottagartypen hjälper den lokala organisationen att styra meddelanden och administration till molnsidan. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) skapar respektive aktiverar denna lokala representation; molnpostlådan uppstår först genom synkronisering och licensiering.

Den vanliga administratörsfrågan är alltså: Var måste jag ändra detta värde? Expertfrågan är: Vilket system är auktoritativt för **just detta enskilda attribut**, och vilken synkroniseringskörning överför det? En molnportal kan visa ett synkroniserat värde utan att kunna redigera det permanent.

Microsoft stöder scenarier där endast Exchange Management Tools finns kvar för lokala mottagarattribut. För vissa miljöer finns dessutom en metod för att överföra hanteringen av Exchange-attribut till molnet. Det är olika driftmodeller med förutsättningar; att enbart stänga av den sista servern flyttar inte dataauktoriteten ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover och klientens väg

När mottagarna är korrekta måste en klient hitta postlådans plats. Autodiscover besvarar denna fråga. Lokala Exchange-slutpunkter kan vidarebefordra en klient för en molnpostlåda till Exchange Online; molnslutpunkter levererar inställningarna för onlinepostlådan ([Autodiscover in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Ett problem med Hybrid-Autodiscover visar sig därför ofta som fel plats: Användaren kan i grunden logga in, men hamnar på den lokala slutpunkten, får en oväntad vidarebefordran eller får inställningar för en postlåda som inte längre finns. DNS, SCP:er, virtuella kataloger, certifikat och mottagarattribut kontrolleras i denna ordning.

Först när klienten har nått rätt postlådetjänst är frågor om protokoll och behörighet meningsfulla. Diagnosen förblir därmed begriplig: först hitta, sedan logga in, därefter auktorisera.

## Ledig/upptagen och andra organisationsövergripande funktioner

En gemensam adressbok räcker inte för kalenderfrågor. Ledig/upptagen kräver organisationsrelationer, tillgänglig Autodiscover och en fungerande förtroende- respektive OAuth-konfiguration. Exchange frågar efter information på den andra sidan i stället för att helt kopiera kalenderdata till det egna systemet ([Sharing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

Samma grundmönster gäller för andra hybridfunktioner: En lokal komponent skickar en begäran, motparten autentiserar den, godkänner åtgärden och levererar ett begränsat resultat. Vid felsökning dokumenteras därför källpostlåda, målpostlåda, riktning och slutpunkt. ”Ledig/upptagen fungerar inte” är för brett utan dessa uppgifter.

För experter blir token och mål-URI:er relevanta. HCW konfigurerar organisationsövergripande relationer, men certifikatbyten, manuella ändringar eller föråldrade slutpunkter kan störa den senare driften. Konfigurationen på båda sidor exporteras alltid tillsammans.

## OAuth mellan Exchange-organisationerna

När det är tydligt vilka organisationsövergripande frågor som sker kan deras autentisering placeras i sitt sammanhang. Exchange kan använda OAuth så att den ena organisationen legitimerar ett tjänstanrop mot den andra. Det gäller hybridfunktioner som organisationsövergripande tillgänglighet och utvalda arkiv-, sök- eller migreringsåtgärder; den exakta användningen beror på version och konfiguration ([Configure OAuth authentication](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

Tokenflödet ersätter inte SMTP-TLS. OAuth skyddar programanrop, medan Hybrid-e-posttransport använder egna kopplingar och certifikatkontroller. Denna separation förhindrar det hopp som förvirrar i många förklaringar: Först fastställs funktionen, sedan dess protokoll och först därefter autentiseringen.

För experter hör AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector och Organization Relationship till en gemensam kontrollbild. Ett enskilt objekt kan vara syntaktiskt närvarande medan certifikat, realm eller mål-URI inte längre stämmer överens med motparten.

## Hybrid Modern Authentication är ett eget klienttema

**Hybrid Modern Authentication**, HMA, kommer först nu eftersom det inte förklarar e-posttransport eller mottagarsynkronisering. HMA gör det möjligt för lokala Exchange- och Skype for Business-resurser som stöds att använda Microsoft Entra ID för modern klientautentisering. Klienten får en Entra-token och använder den mot den lokala tjänsten ([Hybrid modern authentication overview](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

Därmed utökar HMA klientåtkomsten med ett molnberoende. Entra-åtkomst, publicerade URL:er, registrerade Service Principal Names och den lokala Exchange-konfigurationen måste stämma överens. Ett fungerande Hybrid-e-postflöde säger inget om denna tokenväg.

Experter hanterar därför HMA i en egen runbook med versioner som stöds, undantag, utrullningsgrupper och återställningsplan. Funktionen läggs inte lättvindigt till i ett routningsalternativ.

## Postlådeöverflyttningar

Samexistens konfigureras ofta för att stegvis flytta postlådor. En Remote Move kopierar postlådedata via Mailbox Replication Service, synkroniserar ändringar och växlar kontrollerat postlådan till målsidan. Mottagarattribut och routning följer med respektive anpassas ([Move mailboxes between on-premises and Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

För avancerade administratörer består förloppet av förberedelse, start, synkronisering, slutförande och efterkontroll. Före slutförandet kontrolleras datamängd, felaktiga objekt, delegeringar, arkiv, klientåtkomst och e-postflöde. Efter slutförandet måste Autodiscover, licens, måladress och det lokala Remote Mailbox-objektet stämma överens.

Experter planerar batchstorlekar, nätverksgenomströmning, MRS-begränsning, Bad Item Limits, stora objekt och återflyttning. Det tekniska förloppsvärdet är i sig ingen acceptans; användaråtkomst, ställföreträdare, mobila klienter och organisationsövergripande funktioner ingår också.

## Hybrid-e-postflöde förblir en egen väg

Hybrid kräver SMTP mellan den lokala organisationen och Exchange Online. Denna e-postväg använder kopplingar, TLS och certifikat. Den är tillräckligt viktig för en egen artikel, eftersom internetpost, Centralized Mail Transport, e-postgatewayer och delade domäner bildar flera varianter ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Artikeln [Hybrid-e-postflöde](/kb/hybrid-mailfluss) utgår från ett konkret meddelande och följer varje hopp. Först där jämförs Centralized Mail Transport, Egress-IP, filtreringsplats och ytterligare köer. Den här artikeln fokuserar på identitet och samexistens.

## Säkerhet och drift

Hybrid utökar de system som är åtkomliga. Offentliga HTTPS- och SMTP-slutpunkter, certifikat, Entra-synkronisering, privilegierade konton och organisationsövergripande förtroendeobjekt måste inventeras gemensamt. HCW kräver omfattande behörigheter på båda sidor; användningen och loggarna måste skyddas och lagras spårbart ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

I vardagen bör varje hybridfunktion ha en ägare och ett test: mottagarsynkronisering, ledig/upptagen i båda riktningarna, Remote Move, Autodiscover och SMTP i båda riktningarna. Ett regelbundet syntetiskt test upptäcker utgångna certifikat eller tyst ändrade slutpunkter tidigare än ett migreringsprojekt.

Vid störningar hjälper en gemensam tidslinje. Entra-synkroniseringshändelser, HCW-logg, Exchange-händelseloggar, OAuth-tester, Message Tracking och Message Trace samlas inte godtyckligt, utan kopplas till den berörda funktionen. Det förkortar diagnosen och förhindrar att ett lyckat test av en annan funktion misstolkas som bevis.

## Säkerhetskopiering, återuppbyggnad och avveckling

Postlådedata skyddas på den sida där de finns: lokala databaser med lokal återställning, molnpostlådor med funktioner i Exchange Online och Purview. Dessutom måste anslutningen kunna återställas. Det omfattar det lokala HybridConfiguration-objektet, certifikat och privata nycklar, kopplings- och organisationskonfiguration, Entra-synkroniseringsregler samt dokumenterade HCW-beslut.

En återuppbyggnad börjar med identitet och namnuppslagning, följt av HTTPS- och SMTP-åtkomst, HCW-konfiguration och funktionstester. Guiden kan skapa konfigurationen på nytt, men utan lämpliga certifikat, DNS och mottagarobjekt uppstår inget fungerande helhetssystem.

Vid avveckling klargörs först vilka hybridfunktioner som fortfarande används. Microsoft skiljer mellan en kvarvarande server, enbart Management Tools och överföring av hanteringen av Exchange-attribut till molnet. Först efter detta beslut tas kopplingar, organisationsrelationer, slutpunkter och servrar bort kontrollerat ([Manage recipients with Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Decommission after Source of Authority transfer](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Teknisk utveckling och gränser

Hybrid uppstod med Exchange Online som ett sätt att kontrollerat utvidga lokala organisationer till molntjänsten. Tidigare generationer använde i högre grad Federation Trusts; nyare Exchange-versioner och HCW-flöden använder OAuth och Intra-Organization Connectors för många organisationsövergripande funktioner ([Create a hybrid deployment](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

Modellen är kraftfull eftersom den möjliggör migrering och varaktig samexistens. Den är krävande eftersom båda Exchange-organisationerna och deras anslutning måste drivas. Den som efter en migrering inte längre har lokala postlådor bör därför medvetet avgöra vilken administrations- eller samexistensfunktion som fortfarande motiverar Hybrid.

## Källor

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
