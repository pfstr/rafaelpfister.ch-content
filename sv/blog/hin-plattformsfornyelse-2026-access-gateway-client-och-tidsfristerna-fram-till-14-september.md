---
title: "HIN plattformsförnyelse 2026: Access Gateway, Client och tidsfristerna fram till den 26 oktober"
navTitle: "Access Gateway 2026"
description: "Öppning i brandväggen senast den 31 augusti, Access Gateway version 4 från september, SAML-slutpunkter fram till den 12 oktober, hårdvarutoken och HIN Client 4.0 fram till den 26 oktober. Mailgatewayen påverkas inte och ersätts separat."
date: "2026-08-01"
kategorie: "HIN-Gateway"
timeToRead: "5 min läsning"
themen:
  - hin-gateway
  - active-directory-entra
related:
  - hin-mailgateway-backup-disaster-recovery
  - hin-update-issue-version-15.0.5
slug: "hin-plattformsfornyelse-2026-access-gateway-client-och-tidsfristerna-fram-till-14-september"
translationId: "article-106aa61d54408397"
translationOf: hin-plattformerneuerung-2026
translationSourceHash: cd34e6297afca9491bd965e40dbaf55b920296e62e57726754824474fcb27f0d
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:46:59.729Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/hin-plattformsfornyelse-2026-access-gateway-client-och-tidsfristerna-fram-till-14-september
---

# HIN plattformsförnyelse 2026: Access Gateway, Client och tidsfristerna fram till den 26 oktober

HIN förnyar plattformen för identitet och åtkomst 2026. HIN fastställde nya tidsfrister den 27 augusti 2026: den första tidsfristen löpte ut den 31 augusti 2026, de gamla SAML-slutpunkterna kan inte längre användas sedan den 12 oktober 2026 och den stora övergången följer den 26 oktober 2026.

**HIN Access Gateway (AGW), HIN Client och autentiseringsmetoderna påverkas. HIN Mailgateway påverkas inte.** Den ersätts också, men i ett separat projekt med en egen tidsplan.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Din situation</span>
    <span class="choice__title">Endast AGW i drift</span>
    <span class="choice__hint">Tidsfristerna nedan är hela ditt åtgärdsbehov. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Din situation</span>
    <span class="choice__title">Ytterligare migrationsbehov för Mailgateway</span>
    <span class="choice__hint">Då väntar även ersättningen med ”Stargate”, med bred utrullning från Q3 2026. Kostnadsfri kontroll av din miljö. →</span>
  </a>
</div>

## Tidsfristerna

| Datum | Åtgärd | Berör |
|---|---|---|
| 31.08.2026 | Öppning i brandväggen för `idp.id.hin.ch` och `cdn.hin.ch` (443/tcp) | AGW-operatörer |
| från september 2026 | Automatisk installation av AGW version 4 (utom klusterinstallationer) | AGW-operatörer |
| september till 26.10.2026 | Automatisk uppdatering till HIN Client 4.0 i etapper | Alla Client-användare |
| 12.10.2026 | Gamla SAML-slutpunkter kan inte längre användas | Federationer, EPD-anslutningar |
| 26.10.2026 | Hårdvarutoken och testidentiteter för EPD-åtkomst upphör att gälla | Token-användare, integration |
| 26.10.2026 | Tidigare Authenticator App kan användas för sista gången, ny konfiguration från 27.10.2026 | App-användare |
| 26.10.2026 | HIN Client 4.0 krävs | Alla Client-användare |
| från början av november 2026 | Inloggning via webbläsaren i HIN Client | Alla Client-användare |

## Access Gateway är inte Mailgateway

Båda har Gateway i namnet och förväxlas regelbundet. Access Gateway styr åtkomsten till HIN-skyddade applikationer och berör inte e-posttrafiken. Mailgatewayen ligger i e-postflödet och krypterar meddelanden.

## Access Gateway: brandvägg och version 4

Senast den 31 augusti 2026 skulle AGW kunna nå `idp.id.hin.ch` och `cdn.hin.ch` via 443/tcp. HIN rekommenderar att regeln sätts för värdnamnen i stället för IP-adresserna, eftersom IP-adresserna kan ändras. Det är en ändring i brandväggen, inte en inställning i gatewayen, och ligger därför ofta hos den nätverksansvariga i stället för hos gatewayadministratören.

Sedan september 2026 installeras version 4 automatiskt; klusterinstallationer är undantagna och HIN kontaktar dessa kunder direkt. Förutsättningarna är AGW version 3.1.50 eller senare samt att Kerberos är aktiverat som autentiseringsmetod. För anslutningen till Active Directory krävs ett LDAP-konto med läsrättigheter.

Den som inte uppfyller förutsättningarna uppdateras inte, och erfarenhetsmässigt märks det först när ingen längre kan logga in. Kontrollera därför versionsstatus nu i stället för först vid en störning.

## SAML: nya slutpunkter, färre attribut

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

I samband med bytet ändras attributformat och bindningar. Attributuppsättningen reduceras till GLN, namn, födelsedatum och kön.

Detta är punkten där integrationer fallerar. Alla applikationer som använder ytterligare attribut för roller eller separation av klienter får dem inte längre via de nya slutpunkterna; de gamla kan inte längre användas sedan den 12 oktober 2026. Felet visar sig inte som ett inloggningsfel utan som saknade behörigheter i målsystemet.

Testidentiteter för EPD-åtkomst upphör att gälla den 26 oktober 2026 och måste ansökas om på nytt. För EPD-integrationen kräver HIN identiteter som har verifierats genom videoidentifiering; en testmiljö finns ännu inte tillgänglig. Från och med den 26 oktober 2026 kan dessutom assertions utan GLN inkomma.

Den som driver en federation driver nästan alltid också en egen e-postinfrastruktur. För dessa organisationer infaller plattformsförnyelsen samma år som [ersättningen av Mailgateway med ”Stargate”](/stargate): tekniskt oberoende, men konkurrerande om samma personer och underhållsfönster.

## Token, App och HIN Client 4.0

Hårdvarutoken utfärdas inte längre och upphör att gälla den 26 oktober 2026. Alternativ: HIN Client, SMS-kod eller Authenticator App. Den tidigare Authenticator App kan användas till och med den 26 oktober 2026; den nya appen kan konfigureras som andra faktor i självserviceportalen från den 27 oktober 2026. HIN rekommenderar att du i förväg konfigurerar inloggning med lösenord och SMS-kod som nödåtkomst.

HIN Client uppdateras automatiskt i etapper till version 4.0 från september till den 26 oktober 2026. Om Client inte visar version 4.0 den 26 oktober ska den installeras manuellt via `download.hin.ch`. Inloggning via webbläsaren följer från början av november 2026.

Den kritiska punkten är systemkraven: **version 4.0 kräver Windows 11 med TPM eller macOS 14**, och för Intel-Macar dessutom ett Security Chip; utan TPM återstår inloggning med SMS-kod. Äldre enheter måste uppdateras eller ersättas i förväg. För en del mottagningar är tidsfristen därmed inte en mjukvaruuppgift utan en inköpsuppgift. Den som upptäcker detta först strax före den 26 oktober får kämpa med leveranstider och ominstallation av mottagningens programvara.

## Fem frågor för att fastställa nuläget

1. Vilken AGW-version körs, och är Kerberos aktivt?
2. Tillåter brandväggen utgående trafik till `idp.id.hin.ch`?
3. Hur många arbetsplatser kör fortfarande Windows 10 eller macOS 13 och äldre?
4. Hur många hårdvarutoken används, och vad byter de berörda till?
5. Använder någon applikation HIN-attribut som kommer att försvinna?

Svaren på 3 och 5 avgör arbetsinsatsen. Resten är klart på några timmar och dokumenterat av HIN.

## Det andra projektet: ”Stargate”

Oberoende av detta ersätter HIN Mailgatewayen med den nya HIN Gateway, internt projekt ”Stargate”, tekniskt ett Data Mesh-angreppssätt med end-to-end-kryptering och decentraliserad nyckelhantering. Det handlar inte om ett byte av appliance, utan om en arkitekturförändring.

Arbetsinsatsen ligger därmed på en helt annan nivå. Plattformsförnyelsen kräver framför allt att tidsfrister hålls för en brandväggsregel, en versionsstatus och ett enhetsbyte, medan själva det produktiva e-postflödet står på spel med Stargate: det uppbyggda regelverket, nyckelmaterialet, hanteringen av mottagare utan HIN-identitet och frågan om vad man faller tillbaka på om något inte fungerar som förväntat. Eftersom migreringen sker i bokade fyratimmarsfönster och HIN rekommenderar en månads förberedelse, tolererar ett sådant datum inga öppna frågor.

<aside class="offer-box">
  <span class="offer-box__tag">Kostnadsfri kontroll</span>
  <p><strong>Du behöver inte veta var du står. Det är precis därför kontrollen finns.</strong> Jag granskar din befintliga gatewaymiljö och berättar vad som behöver göras före migrationsfönstret, oavsett om du sedan migrerar själv eller får hjälp.</p>
  <a class="offer-box__cta" href="/stargate">Registrera dig nu</a>
</aside>

## Ändringshistorik

- **25 oktober 2026:** Tidsfristerna uppdaterade enligt HIN:s nya fastställande den 27 augusti 2026: öppning i brandväggen (nu även för cdn.hin.ch) senast den 31 augusti, gamla SAML-slutpunkter till den 12 oktober, hårdvarutoken, tidigare Authenticator App och HIN Client 4.0 till den 26 oktober 2026. Tillagt: TPM-krav för HIN Client 4.0 och webbläsarinloggning från början av november 2026.

## Källor

1.  [HIN plattformsförnyelse: Dessa tekniska anpassningar krävs för HIN-medlemmar](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Tidsfrister från augusti till oktober 2026 (status 27 augusti 2026), SAML-slutpunkter, reducerad attributuppsättning, öppningar i brandväggen, Authenticator App och hårdvarutoken.

2.  [Nya HIN Client är här: detta ändras för HIN-medlemmar](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Version 4.0, utrullning till den 26 oktober 2026, krav på operativsystem och TPM, webbläsarbaserad inloggning från november 2026.

3.  [HIN Gateway: Säker kommunikation inom HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Ersättning av Mailgatewayen, arkitektur, driftsmodeller, migrering i bokade tidsfönster.

4.  [Konfiguration av HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): AGW:s roll i åtkomsthanteringen.

5.  [Anslutning till Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos och LDAP-kontot med läsrättigheter.

6.  [HIN AG: ”Från Mailgateway till Data Mesh”](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Bakgrund till ”Stargate”, decentraliserade noder, tidsplan.
