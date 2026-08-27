---
title: "Intern oder extern? Exchange-Hybrid-Mails im Header einordnen: AuthAs, MessageDirectionality und X-OriginatorOrg"
navTitle: "Hybrid-Header lesen"
description: "In Exchange-Hybrid-Umgebungen entscheidet die Header-Klassifikation, ob eine Mail als intern behandelt wird. Welche Kopfzeilen die Einstufung tragen, wie die Tenant-Zuordnung über Zertifikat und Connector funktioniert und woran eine falsch geroutete Nachricht zu erkennen ist."
date: "2026-08-26"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "10 Min. Lesezeit"
themen:
  - "exchange-onprem-hybrid"
  - "microsoft-365-exchange"
hauptthema: "exchange-onprem-hybrid"
produkte:
  - "exchange-hybrid"
  - "hybrid-mailfluss"
  - "exchange-online"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - "exchange-authmechanism-10-authas-internal"
  - "microsoft-365-compauth-reason-codes"
  - "einliefernde-ip-adressen-aggregieren"
slug: "exchange-hybrid-header-intern-extern"
translationId: "article-c8d7859be8dbfe63"
url: "https://rafaelpfister.ch/blog/exchange-hybrid-header-intern-extern"
---

# Intern oder extern? Exchange-Hybrid-Mails im Header einordnen: AuthAs, MessageDirectionality und X-OriginatorOrg

In einer Hybrid-Umgebung sollen Mails zwischen Exchange OnPrem und Exchange Online wie interne Post behandelt werden: kein Spamfilter dazwischen, kein „Extern"-Hinweis, Zustellung an geschützte Verteiler, aufgelöste Anzeigenamen. Ob das funktioniert, entscheidet nicht die Absenderdomain, sondern eine Handvoll Kopfzeilen, die auf dem Weg zwischen den beiden Welten erhalten bleiben müssen. Wer sie lesen kann, beantwortet die häufigsten Hybrid-Fragen direkt am Header: Kam die Mail über den Hybrid-Connector? Warum wurde sie als extern eingestuft? Und welchem Tenant wurde sie zugeordnet?

## Die beteiligten Kopfzeilen

```text
X-MS-Exchange-Organization-AuthAs: Internal
X-MS-Exchange-Organization-AuthSource: EX01.example.com
X-MS-Exchange-Organization-MessageDirectionality: Originating
X-OriginatorOrg: example.com
```

**`AuthAs`** trägt die Einstufung: `Internal` oder `Anonymous`. Sie ist das Ergebnis der übrigen Signale und der direkteste Indikator, wie Exchange die Nachricht behandelt hat.

**`AuthSource`** nennt den FQDN des Servers, der die Einstufung vorgenommen hat: ein eigener OnPrem-Server, ein Mailbox-Server in Exchange Online oder ein EOP-Frontend. Daran lässt sich ablesen, auf welcher Seite die Bewertung stattfand.

**`MessageDirectionality`** unterscheidet `Originating` (die Nachricht ist innerhalb der Organisation entstanden, in Exchange Online oder über einen authentifizierten Inbound Connector) von `Incoming` (die Nachricht kam von aussen herein).

**`X-OriginatorOrg`** identifiziert die Absender-Organisation aus Sicht von Exchange Online: die Default- oder passende Accepted Domain des sendenden Tenants. Der Header wird beim Versand aus Exchange Online über die XOORG-SMTP-Erweiterung gesetzt und ist an die Kombination aus EOP-TLS-Zertifikat, Connector-Konfiguration und Accepted Domain gebunden. Er lässt sich deshalb nicht durch einfaches Mitsenden fälschen: Ein von aussen eingelieferter `X-OriginatorOrg` ohne die zugehörige Vertrauensstellung wird nicht als solcher anerkannt.

Dazu kommen die `X-MS-Exchange-CrossTenant-*`-Header, die Exchange Online beim Übergang zwischen Tenants stempelt, darunter `X-MS-Exchange-CrossTenant-AuthAs`. Sie spiegeln die Einstufung aus Sicht des empfangenden Tenants.

## Wie die Vertrauensstellung technisch funktioniert

Die Internal-Einstufung über die Organisationsgrenze hinweg beruht auf zwei Bausteinen, die der Hybrid Configuration Wizard einrichtet:

Erstens der **Inbound Connector** in Exchange Online vom Typ OnPremises, der die einliefernde Quelle über das TLS-Zertifikat (`TlsSenderCertificateName`) oder die IP-Adresse identifiziert. Über diese Zuordnung entscheidet Exchange Online auch, welchem Tenant eine Einlieferung zugeschrieben wird (Attribution).

Zweitens das Flag **`CloudServicesMailEnabled`** auf den Connectoren beider Seiten. Es sorgt dafür, dass die `X-MS-Exchange-Organization-*`-Header (Cross-Premises-Header) beim Übergang erhalten bleiben statt wie bei externer Post entfernt zu werden. Fehlt das Flag oder läuft die Mail über einen Weg ohne diese Konfiguration, gehen die Header verloren und die Mail kommt als `Anonymous` an.

Daraus folgt die wichtigste Diagnose-Regel: Eine Hybrid-Mail ist nur dann intern, wenn sie den vom HCW konfigurierten Weg tatsächlich genommen hat.

## Fall 1: Die Mail kommt als Anonymous an, obwohl sie intern sein sollte

Dies ist das häufigste Fehlerbild: Mails von OnPrem-Postfächern erscheinen in Exchange Online als extern, mit Spamprüfung, „Extern"-Kennzeichnung oder Ablehnung an geschützten Verteilern. Die Ursachen in absteigender Häufigkeit:

- **Falsche Route:** Die Mail lief nicht über den Hybrid-Connector, sondern über den MX (also durch EOP wie Internet-Post) oder über ein vorgeschaltetes Gateway, das die Cross-Premises-Header entfernt oder die TLS-Verbindung terminiert. Im Header sichtbar an der `Received`-Kette: Statt der direkten Übergabe OnPrem an `*.mail.protection.outlook.com` über den Connector tauchen Zwischenstationen auf.
- **Zertifikatswechsel:** Das OnPrem-Zertifikat wurde erneuert, der `TlsSenderCertificateName` am Inbound Connector aber nicht nachgeführt. Die Identifikation über das Zertifikat greift nicht mehr.
- **Connector-Konfiguration verändert:** `CloudServicesMailEnabled` wurde beim Troubleshooting deaktiviert oder ein manuell erstellter Connector ersetzt den HCW-Connector ohne die nötigen Einstellungen.

Die Prüfung auf der Exchange-Online-Seite:

```powershell
Get-InboundConnector | Format-List Name, ConnectorType,
  TlsSenderCertificateName, SenderIPAddresses, CloudServicesMailEnabled
```

Im Message Trace zeigt das Feld `ConnectorName`, ob die Nachricht tatsächlich über den erwarteten Connector eingeliefert wurde.

## Fall 2: Die Zuordnung zum falschen Tenant

Exchange Online ordnet jede eingehende Nachricht einem Tenant zu; der Header `X-EOPTenantAttributedMessage` trägt die GUID des attribuierten Tenants. Verwenden zwei Tenants denselben `TlsSenderCertificateName` oder dieselben `SenderIPAddresses` in ihren Inbound Connectoren, etwa bei einem gemeinsamen Gateway-Dienstleister oder nach einer Migration, kann eine Nachricht dem falschen Tenant zugeschrieben werden. Sie taucht dann im Message Trace des eigenen Tenants nicht auf und unterliegt fremden Transportregeln.

Die eigene Tenant-GUID liefert `Get-OrganizationConfig | Select-Object GUID`; stimmt sie nicht mit dem Header überein, gehören die Connector-Identifikatoren getrennt: pro Tenant ein eigenes Zertifikat oder eigene IP-Bereiche.

## Fall 3: Extern eingestufte Mail wird trotzdem als intern behandelt

Der umgekehrte Fall entsteht OnPrem: Ein Receive Connector mit der Option `ExternalAuthoritative` („Externally secured") stuft alles als intern ein, was über ihn eingeliefert wird, erkennbar an `AuthAs: Internal` in Kombination mit `AuthMechanism: 10`. Zeigt ein solcher Connector auf ein Gateway, über das auch Internet-Post läuft, gilt externe Post als intern, mit allen Konsequenzen für Spamfilter und Spoofing-Schutz. Die Details samt Gegenmassnahmen stehen im Artikel [AuthMechanism 10 und AuthAs Internal](/blog/exchange-authmechanism-10-authas-internal).

## Den Header als Ganzes lesen

Für die Einordnung einer konkreten Nachricht braucht es alle Signale zusammen: die `Received`-Kette für den tatsächlichen Weg, `AuthAs`/`AuthSource`/`MessageDirectionality` für die Einstufung, `X-OriginatorOrg` und die CrossTenant-Header für die Herkunfts-Organisation. Der [Mail-Header-Analyzer](/tools/header-analyzer) auf dieser Website wertet diese Felder direkt im Browser aus und markiert Tenant-Übergang und Hybrid-Einstufung im Zustellweg; der Header verlässt den Browser dabei nicht.

## Quellen

1.  [Microsoft Tech Community: Demystifying and troubleshooting hybrid mail flow: when is a message internal?](https://techcommunity.microsoft.com/blog/exchange/demystifying-and-troubleshooting-hybrid-mail-flow-when-is-a-message-internal/1420838): Offizielle Beschreibung der Internal-Einstufung, der beteiligten Header und der Connector-Voraussetzungen.

2.  [Microsoft Tech Community: Advanced Office 365 Routing: Locking Down Exchange On-Premises when MX points to Office 365](https://techcommunity.microsoft.com/blog/exchange/advanced-office-365-routing-locking-down-exchange-on-premises-when-mx-points-to-/609238): Funktionsweise von XOORG und X-OriginatorOrg beim Routing zwischen Exchange Online und OnPrem.

3.  [Microsoft Learn (Archiv): Use headers to determine which Exchange Online tenant a message was attributed to](https://learn.microsoft.com/en-us/archive/blogs/eopfieldnotes/use-headers-to-determine-which-exchange-online-tenant-a-message-was-attributed-to): X-EOPTenantAttributedMessage und das Vorgehen bei falscher Tenant-Zuordnung.

4.  [Microsoft Learn: Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): Referenz zu Inbound-Connector-Typen, TlsSenderCertificateName und Attribution.
