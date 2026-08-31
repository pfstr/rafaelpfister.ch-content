---
title: "Intern eller ekstern? Klassifiser Exchange-hybrid-e-post i headeren: AuthAs, MessageDirectionality og X-originatorOrg"
navTitle: "Les hybrid-headere"
description: "I Exchange-hybridmiljøer avgjør headerklassifiseringen om en e-post behandles som intern. Hvilke headere som avgjør klassifiseringen, hvordan tenant-tilordningen fungerer via sertifikat og connector, og hvordan du oppdager en feildirigert melding."
date: "2026-08-26"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 min lesetid"
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
slug: "intern-eller-ekstern-klassifiser-exchange-hybrid-e-post-i-headeren-authas-messagedirectionality"
translationId: "article-c8d7859be8dbfe63"
translationOf: exchange-hybrid-header-intern-extern
url: https://rafaelpfister.ch/no/blog/intern-eller-ekstern-klassifiser-exchange-hybrid-e-post-i-headeren-authas-messagedirectionality
translationSourceHash: 5a0eccedd4b1a194461602319f5f1a8f59de204c1710e261c2358591bb720dfb
translationModel: gpt-5.6-terra
translatedAt: 2026-08-29T10:20:06.261Z
translationReview: automatic
---

# Intern eller ekstern? Klassifiser Exchange-hybrid-e-post i headeren: AuthAs, MessageDirectionality og X-originatorOrg

I et hybridmiljø skal e-post mellom Exchange OnPrem og Exchange Online behandles som intern post: ingen spamfiltrering imellom, ingen «Ekstern»-merking, levering til beskyttede distribusjonslister og oppløste visningsnavn. Om dette fungerer, avgjøres ikke av avsenderdomenet, men av en håndfull headere som må bevares på veien mellom de to verdenene. Den som kan lese dem, får svar på de vanligste hybridspørsmålene direkte i headeren: Kom e-posten via hybrid-connectoren? Hvorfor ble den klassifisert som ekstern? Og hvilken tenant ble den tilordnet?

## De involverte headerne

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** angir klassifiseringen: `Internal` eller `Anonymous`. Den er resultatet av de øvrige signalene og den mest direkte indikatoren på hvordan Exchange har behandlet meldingen.

**`AuthSource`** oppgir FQDN-en til serveren som foretok klassifiseringen: en egen OnPrem-server, en postboksserver i Exchange Online eller en EOP-frontend. Dette viser på hvilken side vurderingen fant sted.

**`MessageDirectionality`** skiller mellom `Originating` (meldingen oppsto innenfor organisasjonen, i Exchange Online eller via en autentisert Inbound Connector) og `Incoming` (meldingen kom inn utenfra).

**`X-OriginatorOrg`** identifiserer avsenderorganisasjonen sett fra Exchange Online: standarddomenet eller det passende Accepted Domain for den sendende tenanten. Headeren settes ved sending fra Exchange Online via XOORG-SMTP-utvidelsen og er knyttet til kombinasjonen av EOP-TLS-sertifikat, connector-konfigurasjon og Accepted Domain. Den kan derfor ikke forfalskes ved bare å sende den med: En `X-OriginatorOrg` levert utenfra uten tilhørende tillitsforhold blir ikke anerkjent som sådan.

I tillegg kommer `X-MS-Exchange-CrossTenant-*`-headerne, som Exchange Online setter ved overgangen mellom tenanter, blant annet `X-MS-Exchange-CrossTenant-AuthAs`. De gjenspeiler klassifiseringen sett fra den mottakende tenantens perspektiv.

## Slik fungerer tillitsforholdet teknisk

Internal-klassifiseringen på tvers av organisasjonsgrensen bygger på to komponenter som Hybrid Configuration Wizard konfigurerer:

For det første **Inbound Connector** i Exchange Online av typen OnPremises, som identifiserer den leverende kilden via TLS-sertifikatet (`TlsSenderCertificateName`) eller IP-adressen. Exchange Online bruker også denne tilordningen til å avgjøre hvilken tenant en levering skal tilskrives (attribution).

For det andre flagget **`CloudServicesMailEnabled`** på connectorene på begge sider. Det sørger for at `X-MS-Exchange-Organization-*`-headerne (Cross-Premises-headere) bevares ved overgangen, i stedet for å fjernes som ved ekstern post. Hvis flagget mangler eller e-posten går via en vei uten denne konfigurasjonen, går headerne tapt og e-posten ankommer som `Anonymous`.

Dette gir den viktigste diagnostikkregelen: En hybrid-e-post er bare intern dersom den faktisk har gått via banen som HCW har konfigurert.

## Tilfelle 1: E-posten kommer som Anonymous, selv om den burde være intern

Dette er det vanligste feilbildet: E-post fra OnPrem-postbokser vises i Exchange Online som ekstern, med spamkontroll, «Ekstern»-merking eller avvisning fra beskyttede distribusjonslister. Årsakene, sortert etter hyppighet:

- **Feil rute:** E-posten gikk ikke via hybrid-connectoren, men via MX-en (og dermed gjennom EOP som internettpost) eller via en foranliggende gateway som fjerner Cross-Premises-headerne eller terminerer TLS-forbindelsen. Dette ses i `Received`-kjeden: I stedet for direkte overlevering fra OnPrem til `*.mail.protection.outlook.com` via connectoren, vises mellomliggende stasjoner.
- **Sertifikatbytte:** OnPrem-sertifikatet ble fornyet, men `TlsSenderCertificateName` på Inbound Connector ble ikke oppdatert. Identifiseringen via sertifikatet fungerer ikke lenger.
- **Endret connector-konfigurasjon:** `CloudServicesMailEnabled` ble deaktivert under feilsøking, eller en manuelt opprettet connector erstatter HCW-connectoren uten nødvendige innstillinger.

Kontrollen på Exchange Online-siden:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

I Message Trace viser feltet `ConnectorName`, om meldingen faktisk ble levert via den forventede connectoren.

## Tilfelle 2: Tilordning til feil tenant

Exchange Online tilordner hver innkommende melding til en tenant; headeren `X-EOPTenantAttributedMessage` inneholder GUID-en til den tilskrevne tenanten. Hvis to tenanter bruker samme `TlsSenderCertificateName` eller samme `SenderIPAddresses` i sine Inbound Connectors, for eksempel hos en felles gateway-leverandør eller etter en migrering, kan en melding bli tilskrevet feil tenant. Da vises den ikke i Message Trace for egen tenant og omfattes av andres transportregler.

Egen tenant-GUID hentes med `Get-OrganizationConfig | Select-Object GUID`; hvis den ikke samsvarer med headeren, må connector-identifikatorene skilles: eget sertifikat eller egne IP-områder per tenant.

## Tilfelle 3: Eksternt klassifisert e-post behandles likevel som intern

Det motsatte tilfellet oppstår OnPrem: En Receive Connector med alternativet `ExternalAuthoritative` («Externally secured») klassifiserer alt som leveres via den som internt, synlig gjennom `AuthAs: Internal` i kombinasjon med `AuthMechanism: 10`. Hvis en slik connector peker til en gateway som også mottar internettpost, regnes ekstern post som intern, med alle konsekvensene det har for spamfiltrering og beskyttelse mot spoofing. Detaljene og mottiltakene finnes i artikkelen [AuthMechanism 10 og AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Les headeren som helhet

For å klassifisere en konkret melding må alle signalene ses under ett: `Received`-kjeden for den faktiske veien, `AuthAs`/`AuthSource`/`MessageDirectionality` for klassifiseringen, `X-OriginatorOrg` og CrossTenant-headerne for avsenderorganisasjonen. [Mail Header Analyzer](/tools/header-analyzer) på dette nettstedet analyserer disse feltene direkte i nettleseren og markerer tenant-overgangen og hybridklassifiseringen i leveringsveien; headeren forlater ikke nettleseren.

## Kilder

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Offisiell beskrivelse av Internal-klassifiseringen, de involverte headerne og connector-forutsetningene.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Virkemåten til XOORG og X-OriginatorOrg ved ruting mellom Exchange Online og OnPrem.

3.  [Microsoft Learn (arkiv): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage og fremgangsmåten ved feil tenant-tilordning.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Referanse for Inbound Connector-typer, TlsSenderCertificateName og attribution.
