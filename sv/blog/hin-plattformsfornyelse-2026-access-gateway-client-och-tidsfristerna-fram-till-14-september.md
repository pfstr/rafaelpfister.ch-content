---
title: "HIN-plattformsförnyelse 2026: Access Gateway, Client och tidsfristerna fram till 14 september"
navTitle: "Access Gateway 2026"
description: "Brandväggsöppning senast 14 augusti, Access Gateway version 4 från 17 augusti, SAML-slutpunkter, hårdvarutoken och HIN Client senast 14 september. Mailgatewayet berörs inte och ersätts separat."
date: "2026-08-01"
kategorie: "HIN-Gateway"
timeToRead: "5 min. lästid"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "hin-plattformsfornyelse-2026-access-gateway-client-och-tidsfristerna-fram-till-14-september"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
url: https://rafaelpfister.ch/sv/blog/hin-plattformsfornyelse-2026-access-gateway-client-och-tidsfristerna-fram-till-14-september
translationSourceHash: 1a174bd131b8bb29f9b1e1e793d4cf19b3f732e3c6fd779a25193f151ec8c109
translationModel: gpt-5.6-terra
translatedAt: 2026-08-02T06:17:03.759Z
translationReview: automatic
---

# HIN-plattformsförnyelse 2026: Access Gateway, Client och tidsfristerna fram till 14 september

HIN förnyar 2026 plattformen för identitet och åtkomst. Den första tidsfristen löper ut den 14 augusti 2026, den stora omställningen följer den 14 september 2026.

**HIN Access Gateway (AGW), HIN Client och autentiseringsmetoderna berörs. HIN Mailgateway berörs inte.** Det kommer också att ersättas, men i ett separat projekt med en egen tidsplan.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Din situation</span>
    <span class="choice__title">Endast AGW i drift</span>
    <span class="choice__hint">Tidsfristerna nedan är hela ditt åtgärdsbehov. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Din situation</span>
    <span class="choice__title">Dessutom migrationsbehov för Mailgateway</span>
    <span class="choice__hint">Då står även ersättningen med «Stargate» på tur, med bred utrullning från Q3 2026. Kostnadsfri kontroll av din miljö. →</span>
  </a>
</div>

## Tidsfristerna

| Datum | Åtgärd | Berör |
|---|---|---|
| 14.08.2026 | Brandväggsöppning för `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | AGW-operatörer |
| 17.08.2026 | Automatisk installation av AGW version 4 | AGW-operatörer |
| från mitten av augusti | Manuell installation av HIN Client 4.0 rekommenderas | Alla Client-användare |
| 14.09.2026 | SAML-slutpunkter migrerade | Federationer, EPD-anslutningar |
| 14.09.2026 | Hårdvarutoken och testidentiteter upphör att gälla | Token-användare, integration |
| 14.09.2026 | Konfigurera Authenticator App på nytt | App-användare |
| 14.09.2026 | Tvingande uppdatering till HIN Client 4.0 | Alla Client-användare |

## Access Gateway är inte Mailgateway

Båda har Gateway i namnet och förväxlas regelbundet. Access Gateway styr åtkomsten till HIN-skyddade applikationer och påverkar inte e-posttrafiken. Mailgateway står i e-postflödet och krypterar meddelanden.

## Access Gateway: brandvägg och version 4

Senast den 14 augusti måste AGW kunna nå `idp.id.hin.ch`. Det är en brandväggsändring, inte en inställning i gatewayet, och ligger därför ofta hos den nätverksansvariga snarare än hos gatewayansvarig.

Från den 17 augusti installeras version 4 automatiskt. Förutsättning: AGW på version 3.1.50 eller senare och Kerberos aktiverat som autentiseringsmetod. För anslutningen till Active Directory krävs ett LDAP-konto med läsbehörighet.

Den som inte uppfyller förutsättningarna uppdateras inte, och erfarenhetsmässigt märks det först när ingen längre kan logga in. Därför är det bättre att kontrollera versionen nu än i september.

## SAML: nya slutpunkter, färre attribut

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

I samband med bytet ändras attributformat och bindningar. Attributmängden reduceras till GLN, namn, födelsedatum och kön.

Detta är punkten som bryter integrationer. Varje applikation som använder ytterligare attribut för roller eller klientseparering får dem inte längre efter den 14 september. Felet visar sig inte som ett inloggningsfel, utan som saknad behörighet i målsystemet.

Testidentiteter upphör att gälla samma datum, så den som vill testa omställningen i en integrationsmiljö bör göra det innan dess.

Den som driver en federation driver nästan alltid också en egen e-postinfrastruktur. För dessa organisationer hamnar plattformsförnyelsen och [ersättningen av Mailgateway med «Stargate»](/stargate) under samma år: tekniskt oberoende, men konkurrerande om samma personer och underhållsfönster.

## Token, app och HIN Client 4.0

Hårdvarutoken utfärdas inte längre och upphör att gälla den 14 september. Alternativ: HIN Client, SMS-kod eller Authenticator App. Appen i sig är giltig till den 14 september och måste därefter konfigureras på nytt via självserviceportalen.

HIN Client uppdateras automatiskt till version 4.0 senast den 14 september, med manuell installation från mitten av augusti via `download.hin.ch`. Inloggningen sker nu via webbläsaren.

Den kritiska punkten är systemkraven: **Version 4.0 kräver Windows 11 eller macOS 14.** Äldre enheter måste uppdateras eller bytas ut före dess. För en del mottagningar är tidsfristen därför inte en mjukvaruuppgift, utan en inköpsuppgift. Den som upptäcker detta först i september får kämpa med leveranstider och ominstallation av mottagningens programvara.

## Fem frågor för nulägesbedömning

1. Vilken AGW-version körs, och är Kerberos aktivt?
2. Tillåter brandväggen utgående trafik till `idp.id.hin.ch`?
3. Hur många arbetsplatser kör fortfarande Windows 10 eller macOS 13 och äldre?
4. Hur många hårdvarutoken används, och vad byter de berörda till?
5. Använder en applikation HIN-attribut som kommer att försvinna?

Svaren på 3 och 5 avgör arbetsinsatsen. Resten kan göras på några timmar och är dokumenterat av HIN.

## Det andra projektet: «Stargate»

Oberoende av detta ersätter HIN Mailgateway med det nya HIN Gateway, internt projekt «Stargate», tekniskt en Data-Mesh-metod med end-to-end-kryptering och decentraliserad nyckelhantering. Det handlar inte om ett byte av appliance, utan om en arkitekturförändring.

Arbetsinsatsen ligger därmed på en helt annan nivå. Plattformsförnyelsen kräver framför allt att tidsfristerna hålls för en brandväggsregel, en versionsnivå och ett enhetsbyte, medan själva e-postflödet i produktion står på spel med Stargate: de uppbyggda regelverken, nyckelmaterialet, hanteringen av mottagare utan HIN-identitet och frågan om vad man återgår till om något inte fungerar som förväntat. Eftersom migreringen sker i bokade fyratimmarsfönster och HIN rekommenderar en månads förberedelser, tolererar ett sådant tillfälle inga öppna punkter.

<aside class="offer-box">
  <span class="offer-box__tag">Kostnadsfri kontroll</span>
  <p><strong>Du behöver inte veta var du står. Det är precis vad kontrollen är till för.</strong> Jag granskar din befintliga gatewaymiljö och berättar vad som behöver göras före migrationsfönstret, oavsett om du sedan migrerar själv eller tar hjälp.</p>
  <a class="offer-box__cta" href="/stargate">Registrera dig nu</a>
</aside>

## Källor

1.  [HIN-plattformsförnyelse: Dessa tekniska anpassningar krävs för HIN-medlemmar](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Tidsfrister i augusti och september, SAML-slutpunkter, reducerad attributmängd, brandväggsöppningar.

2.  [Den nya HIN Client är här: detta ändras för HIN-medlemmar](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Version 4.0, operativsystemkrav, webbläsarbaserad inloggning.

3.  [HIN Gateway: Säker kommunikation inom HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Ersättning av Mailgateway, arkitektur, driftmodeller, migrering i bokade tidsfönster.

4.  [Konfiguration av HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): AGW:s roll i åtkomsthanteringen.

5.  [Anslutning till Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos och LDAP-kontot med läsbehörighet.

6.  [HIN AG: «Från Mailgateway till Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Bakgrund om «Stargate», decentraliserade noder, tidsplan.
