---
title: "Microsoft Entra Connect Sync 2.6.91.0 und 2.6.92.0: Sicherheitskorrekturen, Cloud-Sync-Migration und Pflicht-Upgrade bis April 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "Version 2.6.91.0 bringt Sicherheitskorrekturen, einen geführten Wechsel zu Cloud Sync und behebt die Known Issues von 2.6.84.0; der Hotfix 2.6.92.0 korrigiert die Pass-through-Authentication-Einrichtung. Bis 7. April 2027 müssen alle Server mindestens 2.6.84.0 mit Application-Based Authentication betreiben."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 Min. Lesezeit"
themen:
  - "microsoft-entra"
  - "active-directory-entra"
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "entra-connect-2-6-92-0"
translationId: article-59a51ef69f2e3538
url: https://rafaelpfister.ch/blog/entra-connect-2-6-92-0
---

# Microsoft Entra Connect Sync 2.6.91.0 und 2.6.92.0: Sicherheitskorrekturen, Cloud-Sync-Migration und Pflicht-Upgrade bis April 2027

Microsoft hat Entra Connect Sync 2.6.91.0 am 16. September 2026 und den Hotfix 2.6.92.0 am 23. September 2026 veröffentlicht, beide nur als Download über das Microsoft Entra Admin Center. Version 2.6.91.0 behebt Sicherheitslücken in mitgelieferten Drittkomponenten, macht die phishing-resistente Anmeldung im Setup-Wizard allgemein verfügbar, bringt einen geführten Migrationsablauf zu Microsoft Entra Cloud Sync und korrigiert die drei Known Issues von 2.6.84.0. Version 2.6.92.0 behebt einen Fehler von 2.6.91.0 bei der Einrichtung von Pass-through Authentication. Gleichzeitig nennt Microsoft eine feste Frist: Bis 7. April 2027 müssen alle Server mindestens 2.6.84.0 betreiben und Application-Based Authentication nutzen, sonst stoppt die Synchronisation. Der Artikel richtet sich an Administratoren, die Entra Connect Sync betreiben. Der vorherige Stand ist im Artikel zu [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0) beschrieben.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Feature- und Sicherheitsrelease (Download) | Sicherheitskorrekturen in Drittkomponenten, Cloud-Sync-Migrationsassistent, Passkey-Anmeldung im Wizard GA, neue Microsoft-Graph-Berechtigungen, Fixes für die Known Issues von 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Hotfix mit Sicherheitskorrekturen (Download) | Behebt fehlschlagende Registrierung des Authentication Agent beim Aktivieren von Pass-through Authentication |

Microsoft stuft 2.6.92.0 in den Release Notes als Release mit Sicherheitskorrekturen ein und empfiehlt, «so schnell wie möglich» zu aktualisieren. Für keine der beiden Versionen ist ein Auto-Upgrade-Rollout angekündigt (Stand 26. Oktober 2026). Auch 2.6.84.0 steht weiterhin nur als Download zur Verfügung.

## Pflicht-Upgrade bis 7. April 2027

Die Versionshistorie und die Upgrade-Anleitung tragen neu einen gleichlautenden Hinweis: Entra Connect Sync muss bis 7. April 2027 auf Version 2.6.84.0 oder neuer aktualisiert und auf Application-Based Authentication (ABA) umgestellt sein. Microsoft stellt die Legacy-Authentifizierung über den Directory Synchronization Account ein; Server, die diese Voraussetzungen nicht erfüllen, synchronisieren nach diesem Datum nicht mehr. Stoppt die Synchronisation, stellt laut Microsoft ein Upgrade auf die aktuelle Version mit anschliessender ABA-Konfiguration den Betrieb wieder her.

Für die Planung ist eine Verhaltensänderung aus 2.6.84.0 entscheidend: Seit dieser Version stellt Entra Connect bestehende Server nicht mehr selbstständig auf ABA um. Nur Neuinstallationen konfigurieren ABA automatisch. Ein Upgrade allein erfüllt die Frist also nicht. Bestandsserver müssen Sie im Wizard über **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID** umstellen.

Welches Verfahren ein Server aktuell nutzt, zeigt der Wizard unter **View or export current configuration**. Microsoft nennt dafür zusätzlich dieses Cmdlet:

```powershell
Get-ADSyncEntraConnectorCredential
```

Der Rückgabewert `ConnectorIdentityType` lautet `ServiceAccount` (Legacy-Konto) oder `Application` (ABA). Steht dort `ServiceAccount`, ist die Umstellung vor dem 7. April 2027 erforderlich. Nach erfolgreicher Umstellung empfiehlt Microsoft, den alten Directory Synchronization Account aus Entra ID zu entfernen.

## Version 2.6.91.0

### Sicherheitskorrekturen

Die Release Notes nennen behobene Sicherheitslücken in mitgelieferten Drittkomponenten, gehärtete Berechtigungen auf lokalen temporären Dateien und abgesicherte Downloads von Profilbildern im SharePoint-Connector. CVE-Nummern, CVSS-Werte oder betroffene Komponenten nennt Microsoft nicht. Eine Einordnung des Schweregrads ist deshalb auf Basis der öffentlichen Quellen nicht möglich.

### Neue Microsoft-Graph-Berechtigungen und Conditional Access

Entra Connect erhält zusätzliche Microsoft-Graph-Berechtigungen. Microsoft weist ausdrücklich darauf hin: Wer app-bezogene Conditional-Access-Richtlinien einsetzt, sollte Richtlinien prüfen, die auf `Microsoft.Azure.SyncFabric` oder den Microsoft 365 Reporting Service zielen. Prüfen Sie diese Richtlinien vor dem Upgrade, damit die neuen Zugriffe nicht blockiert werden.

### Geführte Migration zu Cloud Sync

Neu ist ein geführter Ablauf für den Wechsel von Entra Connect Sync zu Microsoft Entra Cloud Sync. Er umfasst eine Bewertung der bestehenden Konfiguration, die Einrichtung des Provisioning Agent, eine gestufte Aktivierung und eine Validierung. Die Funktion steht nur in der Azure Public Cloud zur Verfügung. Die Upgrade-Anleitung von Microsoft empfiehlt ohnehin, vor einem Upgrade zu prüfen, ob Cloud Sync für die eigene Umgebung in Frage kommt.

### Phishing-resistente Anmeldung im Wizard allgemein verfügbar

Die in 2.6.84.0 als Preview eingeführte Anmeldung über den Windows Web Account Manager ist jetzt generally available und standardmässig aktiv. Der Dialog unterstützt Passkeys, FIDO2-Security-Keys und Passwörter, verwendet eine bestehende Anmeldesitzung für weitere Entra-Dienste wieder und erhält die Kerberos-Schlüsselrotation für Seamless Single Sign-On.

### Geänderte PowerShell-Cmdlets

2.6.84.0 hatte für `Set-ADSyncAADCompanyFeature` und `Set-ADSyncAADPasswordSyncState` den Parameter `-AADUsername` verpflichtend gemacht. Mit 2.6.91.0 ist der Parameter wieder optional: Fehlt er, leitet Entra Connect aus der Connector-Konfiguration einen Anmeldehinweis ab und öffnet eine interaktive Anmeldung. Das gilt für `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` und `Set-ADSyncDirSyncConfiguration`. Die interaktive Anmeldung bleibt damit Teil dieser Cmdlets; Skripte, die sie unbeaufsichtigt aufrufen, sollten Sie nach dem Upgrade testen.

Wer Seamless Single Sign-On mit dem eigenständigen PowerShell-Modul konfiguriert, muss `ADSync.psd1` vor `AzureADSSO.psd1` importieren.

### Weitere Änderungen

- Der Dialog **Select Containers** im Synchronization Service Manager ist jetzt schreibgeschützt. Änderungen an der Container-Auswahl erfolgen über **Customize synchronization options** im Wizard.
- Unterstützung für eine weitere Sovereign-Cloud-Umgebung, inklusive Pass-through Authentication, Seamless Single Sign-On, Password Writeback und Health-Agent-Monitoring. Welche Umgebung gemeint ist, nennen die Release Notes nicht.
- Die mitgelieferte SQL Server 2022 LocalDB wurde von 16.0.4250.1 auf 16.0.4262.2 aktualisiert.

### Fehlerbehebungen

- Installation oder Upgrade mit einer bestehenden ADSync-Datenbank konnte mit Fehler `0xE0474352` fehlschlagen (Known Issue von 2.6.84.0).
- Ein Upgrade aktualisierte eine installierte Visual C++ 2015–2022 Redistributable älter als 14.41 nicht.
- Der Wizard konnte beim Start schliessen, wenn die Richtlinie für Windows PowerShell Transcription ein ungültiges Ausgabeverzeichnis enthielt.
- Die Zertifikatsrotation der Application-Based Authentication konnte bei Replikationsverzögerungen im Verzeichnis den letzten nutzbaren Application Key entfernen.
- Der Synchronization Service Manager konnte beim Auflisten der Verzeichnispartitionen in **Connector Properties** schliessen (Known Issue von 2.6.84.0).
- **Change Credentials** öffnete sich nicht für Connector-Zeilen ausserhalb des sichtbaren Bereichs, wenn viele Active-Directory-Connectoren konfiguriert sind.
- Beim Aufklappen einer zuvor abgewählten Domain auf der Seite **Domain and OU filtering** konnte der Wizard die Domain erneut auswählen und nach Abschluss die Synchronisation für die ganze Domain aktivieren (Known Issue von 2.6.84.0).

Der letzte Punkt verdient Aufmerksamkeit: Wer 2.6.84.0 betreibt und nach dem Upgrade den Wizard erneut durchlaufen hat, sollte die Domain- und OU-Filterung kontrollieren. Eine unbeabsichtigt aktivierte Domain führt zu zusätzlichen Objekten in Entra ID.

## Version 2.6.92.0

Der Hotfix behebt einen Fehler von 2.6.91.0: Das Aktivieren von Pass-through Authentication über den Wizard konnte bei der Registrierung des lokal installierten Microsoft Entra Connect Authentication Agent fehlschlagen. Weitere Änderungen nennen die Release Notes nicht. Microsoft markiert 2.6.92.0 als Release mit Sicherheitskorrekturen; einzelne Lücken sind nicht aufgeführt.

## Bekannte Probleme

| Problem | Betroffene Version | Status |
|---|---|---|
| Installation oder Upgrade mit bestehender ADSync-Datenbank schlägt mit `0xE0474352` fehl | 2.6.84.0 | Behoben in 2.6.91.0 |
| Synchronization Service Manager schliesst bei **Connector Properties** | 2.6.84.0 | Behoben in 2.6.91.0 |
| Wizard wählt abgewählte Domain erneut aus | 2.6.84.0 | Behoben in 2.6.91.0 |
| PTA-Aktivierung im Wizard schlägt bei der Agent-Registrierung fehl | 2.6.91.0 | Behoben in 2.6.92.0 |
| Synchronisation schlägt nach Upgrade fehl (`System.IO.FileLoadException`), wenn `miiserver.exe.config` zuvor verändert wurde | 2.5.190.0, 2.6.1.0 | Weiterhin in der Versionshistorie gelistet; Workaround per manuellem bindingRedirect, siehe [Artikel zu 2.6.84.0](/blog/entra-connect-2-6-84-0) |

Die zurückgezogene Version 2.6.79.0 ist weiterhin nicht verfügbar. Microsoft verweist Betroffene neu auf 2.6.92.0: deinstallieren und die aktuelle Version installieren.

## Supportfristen

Jede 2.x-Version fällt zwölf Monate nach Erscheinen der Nachfolgeversion aus dem Support. Mit den beiden neuen Versionen ergeben sich diese Fristen:

| Version | Erschienen | Support-Ende |
|---|---|---|
| 2.5.79.0 | 1. September 2025 | 23. Oktober 2026 (abgelaufen) |
| 2.5.190.0 | 19. November 2025 | 2. Februar 2027 |
| 2.6.1.0 | 2. Februar 2026 | 10. März 2027 |
| 2.6.3.0 | 10. März 2026 | 7. Juli 2027 |
| 2.6.84.0 | 7. Juli 2026 | 16. September 2027 |
| 2.6.91.0 | 16. September 2026 | 23. September 2027 |
| 2.6.92.0 | 23. September 2026 | noch offen |

Für die Versionen 2.5.190.0 bis 2.6.3.0 gilt zusätzlich die Frist vom 7. April 2027: Sie liegen unter der geforderten Mindestversion 2.6.84.0 und synchronisieren nach diesem Datum nicht mehr, obwohl das reguläre Support-Ende von 2.6.3.0 erst im Juli 2027 liegt. Version 2.5.79.0 ist seit 23. Oktober 2026 ohne Support; laut Microsoft können nicht mehr unterstützte Versionen unerwartet aufhören zu funktionieren.

## Empfohlenes Vorgehen

Die Zielversion ist 2.6.92.0. Sie enthält die Sicherheitskorrekturen beider Releases, behebt alle bekannten Probleme von 2.6.84.0 und 2.6.91.0 und erfüllt die Mindestversion für die Frist vom 7. April 2027. Den Installer gibt es nur im Microsoft Entra Admin Center unter **Microsoft Entra Connect | Get started**, Registerkarte **Manage**; Voraussetzungen sind .NET Framework 4.7.2 und TLS 1.2. Prüfen Sie vor dem Upgrade app-bezogene Conditional-Access-Richtlinien für `Microsoft.Azure.SyncFabric` und den Microsoft 365 Reporting Service sowie Skripte, die die genannten Cloud-Konfigurations-Cmdlets aufrufen. Microsoft empfiehlt für risikoarme Upgrades die Swing Migration: zuerst den Staging-Server aktualisieren, die Konfiguration nach Full Import und Full Synchronization verifizieren, dann die Rollen tauschen und den bisherigen aktiven Server aktualisieren. Wer 12 bis 18 Monate nicht aktualisiert hat, sollte laut Microsoft diesen Weg wählen. Stellen Sie nach dem Upgrade Bestandsserver, die noch `ServiceAccount` melden, auf Application-Based Authentication um, und kontrollieren Sie auf Servern, die zuvor 2.6.84.0 betrieben haben, die Domain- und OU-Filterung.

## Quellen

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Release Notes zu 2.6.91.0 und 2.6.92.0, Known Issues von 2.6.84.0, Retirement-Tabelle und Frist 7. April 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Upgrade-Methoden, Swing Migration über einen Staging-Server und Pflicht-Upgrade-Hinweis.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Manuelle Umstellung bestehender Server auf ABA, `Get-ADSyncEntraConnectorCredential` und Entfernen des Legacy-Kontos.
