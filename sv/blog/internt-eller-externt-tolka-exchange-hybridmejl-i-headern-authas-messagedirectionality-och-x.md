---
title: "Internt eller externt? Tolka Exchange-hybridmejl i headern: AuthAs, MessageDirectionality och X-originatorOrg"
navTitle: "Läsa hybridheaders"
description: "I Exchange-hybridmiljöer avgör headerklassificeringen om ett mejl behandlas som internt. Vilka headers som styr klassificeringen, hur tenant-tilldelningen fungerar via certifikat och connector samt hur man identifierar ett felroutat meddelande."
date: "2026-08-26"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min lästid"
themen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-authmechanism-10-authas-internal
  - microsoft-365-compauth-reason-codes
  - einliefernde-ip-adressen-aggregieren
slug: "internt-eller-externt-tolka-exchange-hybridmejl-i-headern-authas-messagedirectionality-och-x"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/sv/blog/internt-eller-externt-tolka-exchange-hybridmejl-i-headern-authas-messagedirectionality-och-x
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:19:47.876Z
translationReview: automatic
---

# Internt eller externt? Tolka Exchange-hybridmejl i headern: AuthAs, MessageDirectionality och X-originatorOrg

I en hybridmiljö ska mejl mellan Exchange OnPrem och Exchange Online behandlas som intern post: inget spamfilter emellan, ingen markering som ”Extern”, leverans till skyddade distributionslistor och upplösta visningsnamn. Om detta fungerar avgörs inte av avsändardomänen, utan av en handfull headers som måste bevaras på vägen mellan de två världarna. Den som kan läsa dem kan besvara de vanligaste hybridfrågorna direkt i headern: Kom mejlet via hybrid-connectorn? Varför klassificerades det som externt? Och vilken tenant tilldelades det?

## Berörda headers

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** anger klassificeringen: `Internal` eller `Anonymous`. Den är resultatet av de övriga signalerna och den mest direkta indikatorn på hur Exchange har behandlat meddelandet.

**`AuthSource`** anger FQDN för servern som gjorde klassificeringen: en egen OnPrem-server, en mailbox-server i Exchange Online eller ett EOP-frontend. Detta visar på vilken sida bedömningen gjordes.

**`MessageDirectionality`** skiljer mellan `Originating` (meddelandet skapades inom organisationen, i Exchange Online eller via en autentiserad Inbound Connector) och `Incoming` (meddelandet kom in utifrån).

**`X-OriginatorOrg`** identifierar avsändarorganisationen ur Exchange Onlines perspektiv: den sändande tenantens standarddomän eller matchande Accepted Domain. Headern sätts vid sändning från Exchange Online via XOORG-SMTP-tillägget och är kopplad till kombinationen av EOP-TLS-certifikat, connector-konfiguration och Accepted Domain. Den kan därför inte förfalskas genom att bara skickas med: en `X-OriginatorOrg` som levereras utifrån utan motsvarande förtroenderelation erkänns inte som sådan.

Därtill kommer `X-MS-Exchange-CrossTenant-*`-headers som Exchange Online stämplar vid övergångar mellan tenants, däribland `X-MS-Exchange-CrossTenant-AuthAs`. De återspeglar klassificeringen ur den mottagande tenantens perspektiv.

## Så fungerar förtroenderelationen tekniskt

Internal-klassificeringen över organisationsgränsen bygger på två komponenter som Hybrid Configuration Wizard konfigurerar:

För det första **Inbound Connector** i Exchange Online av typen OnPremises, som identifierar den levererande källan via TLS-certifikatet (`TlsSenderCertificateName`) eller IP-adressen. Med denna tilldelning avgör Exchange Online även vilken tenant en leverans tillskrivs (attribution).

För det andra flaggan **`CloudServicesMailEnabled`** på connectorerna på båda sidor. Den ser till att `X-MS-Exchange-Organization-*`-headers (Cross-Premises-headers) bevaras vid övergången i stället för att tas bort som vid extern post. Om flaggan saknas eller om mejlet går via en väg utan denna konfiguration försvinner headers och mejlet kommer fram som `Anonymous`.

Därav följer den viktigaste diagnosregeln: Ett hybridmejl är bara internt om det faktiskt har gått den väg som HCW har konfigurerat.

## Fall 1: Mejlet kommer som Anonymous trots att det borde vara internt

Detta är det vanligaste felmönstret: MejI från OnPrem-postlådor visas som externa i Exchange Online, med spamkontroll, markering som ”Extern” eller avslag till skyddade distributionslistor. Orsakerna i fallande frekvens:

- **Felaktig rutt:** Mejlet gick inte via hybrid-connectorn, utan via MX (alltså genom EOP som internetpost) eller via en mellanliggande gateway som tar bort Cross-Premises-headers eller terminerar TLS-anslutningen. Detta syns i headern i `Received`-kedjan: I stället för direkt överlämning från OnPrem till `*.mail.protection.outlook.com` via connectorn visas mellanliggande stationer.
- **Certifikatbyte:** OnPrem-certifikatet har förnyats, men `TlsSenderCertificateName` på Inbound Connector har inte uppdaterats. Identifieringen via certifikatet fungerar då inte längre.
- **Ändrad connector-konfiguration:** `CloudServicesMailEnabled` har inaktiverats vid felsökning eller en manuellt skapad connector ersätter HCW-connectorn utan nödvändiga inställningar.

Kontrollen på Exchange Online-sidan:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

I Message Trace visar fältet `ConnectorName`, om meddelandet faktiskt levererades via den förväntade connectorn.

## Fall 2: Tilldelning till fel tenant

Exchange Online tilldelar varje inkommande meddelande till en tenant; headern `X-EOPTenantAttributedMessage` innehåller GUID för den tillskrivna tenanten. Om två tenants använder samma `TlsSenderCertificateName` eller samma `SenderIPAddresses` i sina Inbound Connectors, exempelvis hos en gemensam gateway-leverantör eller efter en migrering, kan ett meddelande tillskrivas fel tenant. Det visas då inte i den egna tenantens Message Trace och omfattas av andra transportregler.

Den egna tenantens GUID hämtas med `Get-OrganizationConfig | Select-Object GUID`; om den inte stämmer överens med headern måste connector-identifikatorerna separeras: ett eget certifikat eller egna IP-intervall per tenant.

## Fall 3: MejI som klassificerats som externt behandlas ändå som internt

Det omvända fallet uppstår OnPrem: En Receive Connector med alternativet `ExternalAuthoritative` (”Externally secured”) klassificerar allt som levereras via den som internt, vilket syns på `AuthAs: Internal` i kombination med `AuthMechanism: 10`. Om en sådan connector pekar mot en gateway som även hanterar internetpost betraktas extern post som intern, med alla följder för spamfilter och skydd mot spoofing. Detaljerna och motåtgärderna finns i artikeln [AuthMechanism 10 och AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Läs headern som en helhet

För att klassificera ett specifikt meddelande behövs alla signaler tillsammans: `Received`-kedjan för den faktiska vägen, `AuthAs`/`AuthSource`/`MessageDirectionality` för klassificeringen, `X-OriginatorOrg` och CrossTenant-headers för ursprungsorganisationen. [Mail Header Analyzer](/tools/header-analyzer) på denna webbplats analyserar dessa fält direkt i webbläsaren och markerar tenant-övergången och hybridklassificeringen i leveransvägen; headern lämnar inte webbläsaren.

## Källor

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Officiell beskrivning av Internal-klassificeringen, berörda headers och connector-förutsättningarna.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Så fungerar XOORG och X-OriginatorOrg vid routning mellan Exchange Online och OnPrem.

3.  [Microsoft Learn (arkiv): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage och förfarandet vid felaktig tenant-tilldelning.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Referens för Inbound Connector-typer, TlsSenderCertificateName och attribution.
