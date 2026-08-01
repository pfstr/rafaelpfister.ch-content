---
title: "HIN Plattformerneuerung 2026: Access Gateway, Client und die Fristen bis 14. September"
navTitle: "Access Gateway 2026"
description: "Firewall-Freigabe bis 14. August, Access Gateway Version 4 ab 17. August, SAML-Endpunkte, Hardware-Token und HIN Client bis 14. September. Das Mailgateway ist nicht betroffen und wird separat abgelöst."
date: "2026-08-01"
kategorie: "HIN-Gateway"
timeToRead: "5 Min. Lesezeit"
themen:
  - "hin-gateway"
  - "active-directory-entra"
related:
  - "hin-mailgateway-backup-disaster-recovery"
  - "hin-update-issue-version-15.0.5"
slug: "hin-plattformerneuerung-2026"
translationId: "article-106aa61d54408397"
url: "https://rafaelpfister.ch/blog/hin-plattformerneuerung-2026"
---

# HIN Plattformerneuerung 2026: Access Gateway, Client und die Fristen bis 14. September

HIN erneuert 2026 die Plattform für Identität und Zugriff. Die erste Frist läuft am 14. August 2026 ab, die grosse Umstellung folgt am 14. September 2026.

**Betroffen sind das HIN Access Gateway (AGW), der HIN Client und die Authentisierungsmittel. Das HIN Mailgateway ist nicht betroffen.** Es wird ebenfalls abgelöst, aber in einem separaten Vorhaben mit eigenem Fahrplan.

Kurze Einordnung vorab:

- **Kein eigenes Mailgateway im Betrieb?** Dann ist die Tabelle unten Ihr vollständiger Handlungsbedarf.
- **Eigenes Mailgateway im Betrieb?** Dann steht zusätzlich die Ablösung durch «Stargate» an, breiter Rollout ab Q3 2026. Wer den Migrationspfad vorab geklärt haben will, kann sich für einen kostenlosen Check der bestehenden Umgebung registrieren.

<aside class="offer-box">
  <span class="offer-box__tag">Angebot</span>
  <p><strong>Vorbereitung auf die Stargate-Migration.</strong> Kostenloser Check Ihrer bestehenden Gateway-Umgebung, mit Empfehlung für den Migrationspfad.</p>
  <a class="offer-box__cta" href="/stargate">Kostenlos registrieren</a>
</aside>

## Die Fristen

| Datum | Massnahme | Betrifft |
|---|---|---|
| 14.08.2026 | Firewall-Freigabe für `idp.id.hin.ch` (`185.154.38.46`, `193.168.215.45`) | AGW-Betreiber |
| 17.08.2026 | Automatische Installation AGW Version 4 | AGW-Betreiber |
| ab Mitte August | Manuelle Installation HIN Client 4.0 empfohlen | Alle Client-Nutzer |
| 14.09.2026 | SAML-Endpunkte migriert | Föderationen, EPD-Anbindungen |
| 14.09.2026 | Hardware-Token und Testidentitäten verfallen | Token-Nutzer, Integration |
| 14.09.2026 | Authenticator App neu konfigurieren | App-Nutzer |
| 14.09.2026 | Zwangsupdate auf HIN Client 4.0 | Alle Client-Nutzer |

## Access Gateway ist nicht Mailgateway

Beide tragen Gateway im Namen und werden regelmässig verwechselt. Das Access Gateway steuert den Zugriff auf HIN-geschützte Anwendungen und berührt den Mailverkehr nicht. Das Mailgateway steht in der Mailstrecke und verschlüsselt Nachrichten.

Eine Praxis kann also von sämtlichen Fristen betroffen sein, ohne je ein Mailgateway betrieben zu haben, während umgekehrt ein Mailgateway-Betreiber beim Access Gateway völlig unberührt bleiben kann.

## Access Gateway: Firewall und Version 4

Bis 14. August muss das AGW `idp.id.hin.ch` erreichen können. Das ist eine Firewall-Änderung, keine Einstellung im Gateway, und liegt damit oft beim Netzwerkverantwortlichen statt beim Gateway-Betreuer.

Ab 17. August wird Version 4 automatisch installiert. Voraussetzung: AGW auf Version 3.1.50 oder höher, und Kerberos als Authentisierungsmethode aktiv. Für die Anbindung ans Active Directory braucht es ein LDAP-Konto mit Leserechten.

Wer die Voraussetzungen nicht erfüllt, wird nicht aktualisiert, und das fällt erfahrungsgemäss erst auf, wenn sich niemand mehr anmelden kann. Den Versionsstand prüfen Sie deshalb besser jetzt als im September.

## SAML: neue Endpunkte, weniger Attribute

```text
Föderationsdienst
  broker.hin.ch/realms/HINBroker/protocol/saml/descriptor

EPD-Zugang
  idp.id.hin.ch/auth/realms/hinid/protocol/saml/descriptor
```

Mit dem Wechsel ändern sich Attributformate und Bindings. Die Attributmenge wird auf GLN, Name, Geburtsdatum und Geschlecht reduziert.

Das ist der Punkt, der Integrationen bricht. Jede Anwendung, die weitere Attribute für Rollen oder Mandantentrennung verwertet, bekommt sie nach dem 14. September nicht mehr. Der Fehler zeigt sich nicht als Anmeldefehler, sondern als fehlende Berechtigung im Zielsystem.

Testidentitäten verfallen am selben Datum, wer die Umstellung also in einer Integrationsumgebung erproben möchte, sollte das vorher tun.

Wer eine Föderation betreibt, betreibt fast immer auch eigene Mailinfrastruktur. Für diese Organisationen fällt die Plattformerneuerung mit der [Mailgateway-Ablösung durch «Stargate»](/stargate) ins selbe Jahr: technisch unabhängig, aber um dieselben Personen und Wartungsfenster konkurrierend.

## Token, App und HIN Client 4.0

Hardware-Token werden nicht mehr ausgegeben und verfallen am 14. September. Alternativen: HIN Client, SMS-Code oder Authenticator App. Die App selbst ist bis zum 14. September gültig und danach über das Self-Service-Portal neu zu konfigurieren.

Der HIN Client wird spätestens am 14. September automatisch auf Version 4.0 aktualisiert, manuelle Installation ab Mitte August über `download.hin.ch`. Der Login läuft neu über den Browser.

Der kritische Punkt sind die Systemvoraussetzungen: **Version 4.0 setzt Windows 11 oder macOS 14 voraus.** Ältere Geräte müssen vorher aktualisiert oder ersetzt werden. Für einen Teil der Praxen ist die Frist damit keine Software-, sondern eine Beschaffungsaufgabe. Wer das erst im September merkt, kämpft mit Lieferzeiten und Neuinstallation der Praxissoftware.

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
  <p><strong>Ihr Migrationspfad sollte vor der Terminbuchung feststehen.</strong> Ich sehe mir Ihre bestehende Gateway-Umgebung an und gebe Ihnen eine Empfehlung dazu, unabhängig davon, ob Sie die Migration danach selbst durchführen oder sich begleiten lassen.</p>
  <a class="offer-box__cta" href="/stargate">Jetzt registrieren</a>
</aside>

## Quellen

1.  [HIN Plattformerneuerung: Diese technischen Anpassungen sind für HIN Mitglieder erforderlich](https://www.hin.ch/de/blog/2026/technische-anpassungen.cfm): Fristen im August und September, SAML-Endpunkte, reduzierte Attributmenge, Firewall-Freigaben.

2.  [Der neue HIN Client ist da: das ändert sich für HIN Mitglieder](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Version 4.0, Betriebssystemvoraussetzungen, browserbasierte Anmeldung.

3.  [HIN Gateway: Sichere Kommunikation innerhalb der HIN Community](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm): Ablösung des Mailgateway, Architektur, Betriebsmodelle, Migration in gebuchten Zeitfenstern.

4.  [Konfiguration des HIN Access Gateway](https://cdn.hin.ch/agw/manual/DE/4-konfiguration-des-hin-access-gateway.html): Rolle des AGW im Zugriffsmanagement.

5.  [Anbindung Active Directory](https://cdn.hin.ch/agw/manual/DE/5-anbindung-active-directory.html): Kerberos und das LDAP-Konto mit Leserechten.

6.  [HIN AG: «Vom Mailgateway zum Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm): Hintergrund zu «Stargate», dezentrale Nodes, Zeitplan.
