---
title: "HIN Plattformerneuerung 2026: Access Gateway, Client und die Fristen bis 26. Oktober"
navTitle: "Access Gateway 2026"
description: "Firewall-Freigabe bis 31. August, Access Gateway Version 4 ab September, SAML-Endpunkte bis 12. Oktober, Hardware-Token und HIN Client 4.0 bis 26. Oktober. Das Mailgateway ist nicht betroffen und wird separat abgelöst."
date: "2026-08-01"
aktualisiert: "2026-10-25"
kategorie: "HIN-Gateway"
timeToRead: "5 Min. Lesezeit"
themen:
  - "hin-gateway"
  - "active-directory-entra"
produkte:
  - "hin"
protokolle:
  - "tcp"
  - "migration"
related:
  - "hin-mailgateway-backup-disaster-recovery"
  - "hin-update-issue-version-15.0.5"
slug: "hin-plattformerneuerung-2026"
translationId: "article-106aa61d54408397"
url: "https://rafaelpfister.ch/blog/hin-plattformerneuerung-2026"
---

# HIN Plattformerneuerung 2026: Access Gateway, Client und die Fristen bis 26. Oktober

HIN erneuert 2026 die Plattform für Identität und Zugriff. HIN hat die Fristen am 27. August 2026 neu festgelegt: Die erste Frist lief am 31. August 2026 ab, die alten SAML-Endpunkte sind seit dem 12. Oktober 2026 nicht mehr nutzbar, die grosse Umstellung folgt am 26. Oktober 2026.

**Betroffen sind das HIN Access Gateway (AGW), der HIN Client und die Authentisierungsmittel. Das HIN Mailgateway ist nicht betroffen.** Es wird ebenfalls abgelöst, aber in einem separaten Vorhaben mit eigenem Fahrplan.

<div class="choice-row">
  <a class="choice" href="#die-fristen">
    <span class="choice__label">Ihre Situation</span>
    <span class="choice__title">Nur AGW im Betrieb</span>
    <span class="choice__hint">Die Fristen unten sind Ihr vollständiger Handlungsbedarf. →</span>
  </a>
  <a class="choice" href="/stargate">
    <span class="choice__label">Ihre Situation</span>
    <span class="choice__title">Zusätzlich Migrationsbedarf beim Mailgateway</span>
    <span class="choice__hint">Dann steht auch die Ablösung durch «Stargate» an, breiter Rollout ab Q3 2026. Kostenloser Check Ihrer Umgebung. →</span>
  </a>
</div>

## Die Fristen

| Datum | Massnahme | Betrifft |
|---|---|---|
| 31.08.2026 | Firewall-Freigabe für `idp.id.hin.ch` und `cdn.hin.ch` (443/tcp) | AGW-Betreiber |
| ab September 2026 | Automatische Installation AGW Version 4 (ohne Cluster-Installationen) | AGW-Betreiber |
| September bis 26.10.2026 | Automatisches Update auf HIN Client 4.0 in Tranchen | Alle Client-Nutzer |
| 12.10.2026 | Alte SAML-Endpunkte nicht mehr nutzbar | Föderationen, EPD-Anbindungen |
| 26.10.2026 | Hardware-Token und Test-Identitäten für den EPD-Zugang verfallen | Token-Nutzer, Integration |
| 26.10.2026 | Bisherige Authenticator App letztmals nutzbar, Neueinrichtung ab 27.10.2026 | App-Nutzer |
| 26.10.2026 | HIN Client 4.0 erforderlich | Alle Client-Nutzer |
| ab Anfang November 2026 | Login über den Browser im HIN Client | Alle Client-Nutzer |

## Access Gateway ist nicht Mailgateway

Beide tragen Gateway im Namen und werden regelmässig verwechselt. Das Access Gateway steuert den Zugriff auf HIN-geschützte Anwendungen und berührt den Mailverkehr nicht. Das Mailgateway steht in der Mailstrecke und verschlüsselt Nachrichten.

## Access Gateway: Firewall und Version 4

Bis 31. August 2026 musste das AGW `idp.id.hin.ch` und `cdn.hin.ch` über 443/tcp erreichen können. HIN empfiehlt, die Regel auf die Hostnamen statt auf IP-Adressen zu setzen, weil sich die IP-Adressen ändern können. Das ist eine Firewall-Änderung, keine Einstellung im Gateway, und liegt damit oft beim Netzwerkverantwortlichen statt beim Gateway-Betreuer.

Seit September 2026 wird Version 4 automatisch installiert; Cluster-Installationen sind ausgenommen, mit diesen Kunden nimmt HIN direkt Kontakt auf. Voraussetzung: AGW auf Version 3.1.50 oder höher, und Kerberos als Authentisierungsmethode aktiv. Für die Anbindung ans Active Directory braucht es ein LDAP-Konto mit Leserechten.

Wer die Voraussetzungen nicht erfüllt, wird nicht aktualisiert, und das fällt erfahrungsgemäss erst auf, wenn sich niemand mehr anmelden kann. Den Versionsstand prüfen Sie deshalb besser jetzt als erst bei einer Störung.

## SAML: neue Endpunkte, weniger Attribute

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Mit dem Wechsel ändern sich Attributformate und Bindings. Die Attributmenge wird auf GLN, Name, Geburtsdatum und Geschlecht reduziert.

Das ist der Punkt, an dem Integrationen scheitern. Jede Anwendung, die weitere Attribute für Rollen oder Mandantentrennung verwertet, bekommt sie über die neuen Endpunkte nicht mehr; die alten sind seit dem 12. Oktober 2026 nicht mehr nutzbar. Der Fehler zeigt sich nicht als Anmeldefehler, sondern als fehlende Berechtigung im Zielsystem.

Test-Identitäten für den EPD-Zugang verlieren am 26. Oktober 2026 ihre Gültigkeit und müssen neu beantragt werden. Für die EPD-Integration verlangt HIN per Videoidentifikation geprüfte Identitäten, eine Testumgebung ist noch nicht verfügbar. Ab dem 26. Oktober 2026 können zudem Assertions ohne GLN eintreffen.

Wer eine Föderation betreibt, betreibt fast immer auch eigene Mailinfrastruktur. Für diese Organisationen fällt die Plattformerneuerung mit der [Mailgateway-Ablösung durch «Stargate»](/stargate) ins selbe Jahr: technisch unabhängig, aber um dieselben Personen und Wartungsfenster konkurrierend.

## Token, App und HIN Client 4.0

Hardware-Token werden nicht mehr ausgegeben und verfallen am 26. Oktober 2026. Alternativen: HIN Client, SMS-Code oder Authenticator App. Die bisherige Authenticator App ist bis zum 26. Oktober 2026 einsetzbar; die neue App lässt sich ab dem 27. Oktober 2026 im Self-Service-Portal als zweiter Faktor einrichten. HIN empfiehlt, vorher das Login mit Passwort und SMS-Code als Notfallzugang einzurichten.

Der HIN Client wird von September bis 26. Oktober 2026 tranchenweise automatisch auf Version 4.0 aktualisiert. Zeigt der Client am 26. Oktober nicht Version 4.0 an, ist er manuell über `download.hin.ch` zu installieren. Der Login über den Browser folgt ab Anfang November 2026.

Der kritische Punkt sind die Systemvoraussetzungen: **Version 4.0 setzt Windows 11 mit TPM oder macOS 14 voraus**, bei Intel-Macs zusätzlich einen Security Chip; ohne TPM bleibt das Login mit SMS-Code. Ältere Geräte müssen vorher aktualisiert oder ersetzt werden. Für einen Teil der Praxen ist die Frist damit keine Software-, sondern eine Beschaffungsaufgabe. Wer das erst kurz vor dem 26. Oktober merkt, kämpft mit Lieferzeiten und Neuinstallation der Praxissoftware.

## Fünf Fragen zur Standortbestimmung

1. Welche AGW-Version läuft, und ist Kerberos aktiv?
2. Lässt die Firewall ausgehend `idp.id.hin.ch` zu?
3. Wie viele Arbeitsplätze laufen noch auf Windows 10 oder macOS 13 und älter?
4. Wie viele Hardware-Token sind im Einsatz, und worauf wechseln die Betroffenen?
5. Verwertet eine Anwendung HIN-Attribute, die künftig wegfallen?

Die Antworten auf 3 und 5 bestimmen den Aufwand. Der Rest ist in wenigen Stunden erledigt und von HIN dokumentiert.

## Das zweite Vorhaben: «Stargate»

Unabhängig davon ersetzt HIN das Mailgateway durch das neue HIN Gateway, intern Projekt «Stargate», technisch ein Data-Mesh-Ansatz mit Ende-zu-Ende-Verschlüsselung und dezentraler Schlüsselverwaltung. Es handelt sich dabei nicht um einen Austausch der Appliance, sondern um einen Architekturwechsel.

Der Aufwand liegt damit auf einer ganz anderen Ebene. Die Plattformerneuerung verlangt vor allem Termintreue bei einer Firewall-Regel, einem Versionsstand und einem Gerätetausch, während bei Stargate die produktive Mailstrecke selbst zur Disposition steht: das gewachsene Regelwerk, das Schlüsselmaterial, die Behandlung von Empfängern ohne HIN-Identität und die Frage, worauf zurückgefallen wird, wenn etwas nicht wie erwartet läuft. Da die Migration in gebuchten Vierstundenfenstern abläuft und HIN einen Monat Vorbereitung empfiehlt, verzeiht ein solcher Termin keine offenen Punkte.

<aside class="offer-box">
  <span class="offer-box__tag">Kostenloser Check</span>
  <p><strong>Sie müssen nicht wissen, wo Sie stehen. Genau dafür ist der Check da.</strong> Ich sehe mir Ihre bestehende Gateway-Umgebung an und sage Ihnen, was vor dem Migrationsfenster zu erledigen ist, unabhängig davon, ob Sie danach selbst migrieren oder sich begleiten lassen.</p>
  <a class="offer-box__cta" href="/stargate">Jetzt registrieren</a>
</aside>

## Quellen

1.  [HIN Plattformerneuerung: Diese technischen Anpassungen sind für HIN Mitglieder erforderlich](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Fristen von August bis Oktober 2026 (Stand 27. August 2026), SAML-Endpunkte, reduzierte Attributmenge, Firewall-Freigaben, Authenticator App und Hardware-Token.

2.  [Der neue HIN Client ist da: das ändert sich für HIN Mitglieder](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Version 4.0, Rollout bis 26. Oktober 2026, Betriebssystem- und TPM-Voraussetzungen, browserbasierte Anmeldung ab November 2026.

3.  [HIN Gateway: Sichere Kommunikation innerhalb der HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Ablösung des Mailgateway, Architektur, Betriebsmodelle, Migration in gebuchten Zeitfenstern.

4.  [Konfiguration des HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): Rolle des AGW im Zugriffsmanagement.

5.  [Anbindung Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos und das LDAP-Konto mit Leserechten.

6.  [HIN AG: «Vom Mailgateway zum Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Hintergrund zu «Stargate», dezentrale Nodes, Zeitplan.
