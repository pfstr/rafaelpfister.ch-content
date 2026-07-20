---
title: "Microsoft Entra Connect Sync 2.6.84.0: Sicherheitsfixes, neue Spielregeln für App-Authentifizierung und PHS – und die Lehre aus dem zurückgezogenen Vorgänger"
navTitle: "Entra Connect 2.6.84"
description: "Microsoft hat am 7. Juli 2026 Entra Connect Sync 2.6.84.0 veröffentlicht – mit Sicherheitsfixes, Passkey-Support im Wizard und etlichen Verhaltensänderungen bei Application-Based Authentication, PowerShell-Cmdlets und Password Hash Sync. Der direkte Vorgänger 2.6.79.0 wurde nach dem Release zurückgezogen. Was im Release steckt, welche Hintergründe dahinterliegen und warum ein kontrolliertes Abwarten trotz Sicherheitsfixes vertretbar ist."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min to read"
themen:
  - "microsoft-entra"
  - "active-directory-ldap"
slug: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/blog/entra-connect-2-6-84-0"
aiPrompt: |
  Du bist mein Entra-Connect-Administrationsassistent. Hilf mir, das Update auf Microsoft Entra Connect Sync 2.6.84.0 sauber zu planen und durchzuführen. Gehe Schritt für Schritt vor:
  1. Aktuelle Version und Supportfrist ermitteln (12-Monats-Retirement-Politik beachten; 2.5.3.0 läuft am 31. Juli 2026 aus).
  2. Prüfen, ob miiserver.exe.config manuell verändert wurde (z. B. FIPS/PHS-Anpassung) und ob der Workaround mit dem bindingRedirect für System.Diagnostics.DiagnosticSource nötig ist.
  3. Prüfen, ob der Server noch das Legacy-Konto (Directory Synchronization Account) oder schon Application-Based Authentication nutzt, und ob Skripte Set-ADSyncAADCompanyFeature oder Set-ADSyncAADPasswordSyncState aufrufen (neu: Pflichtparameter -AADUsername).
  4. Update zuerst auf einem Staging-Mode-Server testen, Konfiguration vorher exportieren, dann Produktion.
  Frage nach den Werten, die nur ich kenne (aktuelle Version, Staging-Server vorhanden, FIPS aktiv, Sovereign Cloud).
---

# Microsoft Entra Connect Sync 2.6.84.0

**Microsoft hat am 7. Juli 2026 die Version 2.6.84.0 von Entra Connect Sync veröffentlicht und stuft sie als Sicherheitsrelease ein: «We recommend upgrading to this version as soon as possible.» Gleichzeitig steht im selben Dokument der Hinweis, dass der direkte Vorgänger 2.6.79.0 nach dem Release zurückgezogen wurde, weil nachträglich ein Problem im Installer gefunden wurde. Beides zusammen ergibt die realistische Einordnung: Das Update ist nötig, aber ein Tag-1-Rollout auf den Produktionsserver ist es nicht – ausser man fällt in eine der unten beschriebenen Ausnahmen.**

## Die Ausgangslage: ein Release mit Vorgeschichte

Die 2.6er-Linie von Entra Connect Sync hat einen holprigen Start hinter sich. Ein kurzer Rückblick, weil er für die Update-Entscheidung relevant ist:

- **2.6.1.0** (Februar 2026) behob unter anderem einen Fehler, bei dem das Bearbeiten der Entra-ID-Connector-Konfiguration im Synchronization Service Manager die Parameter der Application-Based Authentication löschte – mit der Folge, dass Wizard und Zertifikatsrotation fehlschlugen. Für alle 2.5er-Versionen galt deshalb die bemerkenswerte Empfehlung, die Verwaltungsoberfläche des Produkts schlicht nicht zu benutzen.
- **2.6.3.0** (März 2026) war ein Hotfix für ein Problem, bei dem Auto-Upgrade den Entra-Connect-Server unerwartet stoppen konnte. Die Notlösung damals: Auto-Upgrade erkennt manuell veränderte Konfigurationsdateien und überspringt solche Server einfach.
- **2.6.79.0** (Juni 2026) wurde nach der Veröffentlichung komplett zurückgezogen. Der Installer ist nicht mehr verfügbar; wer die Version installiert hat, soll sie laut Microsoft deinstallieren und 2.6.84.0 installieren. Was genau das Problem war, dokumentiert Microsoft nicht.

Version 2.6.84.0 ist Stand heute nur als Download über das Microsoft Entra Admin Center verfügbar («Released for download»). Ein Auto-Upgrade-Rollout wurde noch nicht angekündigt – auch das ist ein Signal: Microsoft selbst verteilt die Version noch nicht flächendeckend auf Bestandsinstallationen.

## Neue Funktionen

### Phishing-resistente Anmeldung im Setup-Wizard (Preview)

Der Setup-Wizard unterstützt neu die Anmeldung mit Passkeys und FIDO2-Security-Keys über den Windows Web Account Manager (WAM). Der Hintergrund: Microsoft erzwingt seit 2024/2025 schrittweise MFA für Anmeldungen an Azure- und Entra-Verwaltungsoberflächen, und viele Organisationen haben ihre Admin-Konten per Conditional Access auf phishing-resistente Methoden (FIDO2, Passkeys, zertifikatsbasierte Authentifizierung) eingeschränkt. Genau diese sauber abgesicherten Konten konnten sich bisher im Entra-Connect-Wizard nicht anmelden, weil der eingebettete Anmeldedialog die Methoden nicht unterstützte. In der Praxis führte das zu unschönen Workarounds – etwa eigenen «Setup-Konten» mit schwächeren Auth-Anforderungen, nur damit der Wizard durchläuft. Diese Lücke schliesst sich jetzt, wenn auch vorerst als Preview.

### Unterstützung für die französische Sovereign Cloud

2.6.84.0 bringt Unterstützung für die französische Sovereign-Cloud-Umgebung, inklusive Pass-through Authentication, Seamless Single Sign-On, Password Writeback und Health-Agent-Monitoring. Passend dazu wurde ein Fehler behoben, bei dem der Application-Proxy-Cloud-Name in der France Cloud nicht korrekt aufgelöst wurde und die PTA-Registrierung mit «EnvironmentName attribute is invalid» scheiterte.

## Verhaltensänderungen im Detail

Der interessanteste Teil des Releases sind nicht die neuen Funktionen, sondern die geänderten Verhaltensweisen. Mehrere davon korrigieren Design-Entscheidungen, die in der Praxis für Überraschungen gesorgt haben.

### Auto-Upgrade zerstört keine angepassten Konfigurationsdateien mehr

Das ist die Änderung mit der längsten Vorgeschichte. Bisher überschrieb Auto-Upgrade die Datei `miiserver.exe.config` beim Update vollständig – manuelle Anpassungen gingen verloren. Das klingt nach einem Randfall, war es aber nicht: Microsoft selbst hatte Administratoren in FIPS-Umgebungen angewiesen, genau diese Datei zu bearbeiten, damit Password Hash Synchronization mit aktiviertem FIPS-Modus funktioniert. Wer der offiziellen Anleitung gefolgt war, hatte also eine «modifizierte» Konfigurationsdatei.

Die Folgen zeigten sich beim Upgrade auf 2.5.190.0 und 2.6.1.0 als bekanntes Problem: Erkennt der Installer eine veränderte `miiserver.exe.config`, lässt er die Datei unangetastet – dann fehlt aber das neue Assembly-Binding, und der Synchronisationsdienst stirbt nach dem Upgrade mit `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. Der dokumentierte Workaround: in der `assemblyBinding`-Sektion von `miiserver.exe.config` (unter `%programfiles%\Microsoft Azure AD Sync\Bin`) manuell einen bindingRedirect nachtragen:

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Danach den ADSync-Dienst neu starten. Der Hotfix 2.6.3.0 entschärfte das Problem nur für Auto-Upgrade – betroffene Server wurden schlicht übersprungen und blieben auf dem alten Stand. Mit 2.6.84.0 kommt die eigentliche Lösung: Der Upgrade-Prozess führt Kundenanpassungen mit der neuen Konfiguration zusammen und validiert das Ergebnis, bevor es angewendet wird. Wer beim manuellen Upgrade von einer betroffenen Version kommt, sollte den Zustand seiner `miiserver.exe.config` trotzdem vorher prüfen und die Datei sichern – der Merge-Mechanismus ist neu und damit selbst noch nicht praxiserprobt.

### Application-Based Authentication: Schluss mit stillem Fallback und stiller Umstellung

Zur Erinnerung: Seit 2.5.76.0 ist die Application-Based Authentication (ABA) generally available und Standard. Statt des alten Directory Synchronization Accounts – ein Cloud-Konto mit gespeichertem Passwort – authentifiziert sich der Sync-Server als Entra-ID-Anwendung mit einem Zertifikat, idealerweise TPM-geschützt. Das ist die deutlich robustere Architektur: kein Passwort, das abfliessen kann, und ein Credential, das an die Maschine gebunden ist.

2.6.84.0 räumt bei zwei Verhaltensweisen auf, die diesen Sicherheitsgewinn unterlaufen haben:

**Kein stiller Fallback mehr.** Schlug die ABA-Einrichtung im Wizard fehl, fiel das Setup bisher kommentarlos auf das Legacy-Konto zurück. Das Ergebnis: Der Administrator glaubte, eine zertifikatsbasierte Anmeldung zu haben, tatsächlich lief der Server mit dem alten Passwort-Konto. Ein klassisches Fail-Open-Muster. Neu bricht der Wizard mit einer klaren Fehlermeldung ab («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), damit die eigentliche Ursache behoben wird, statt sie zu verdecken.

**Keine automatische Umstellung im Hintergrund mehr.** Bisher stellte Entra Connect bestehende Server während des laufenden Sync-Betriebs selbstständig vom Legacy-Konto auf ABA um. Aus Security-Sicht gut gemeint, aus Betriebssicht ein Albtraum: Ein Authentifizierungsverfahren wechselt ungefragt, ohne Change-Fenster, ohne dass jemand davon weiss – und wenn dabei etwas schiefgeht (TPM-Probleme, Conditional-Access-Konflikte, Firewall), steht die Synchronisation. Neu gilt: Nur Neuinstallationen konfigurieren ABA automatisch; Bestandsserver wechseln erst, wenn ein Administrator den Wizard startet und **Configure application-based authentication to Microsoft Entra ID** explizit auswählt. Der Wechsel gehört damit wieder dorthin, wo er hingehört – in einen geplanten Change.

Ergänzend wurde die TPM-Behandlung verbessert: Das Setup testet die Signierfähigkeit eines Zertifikats jetzt vorab und behandelt die TPM-Signaturprüfung korrekt. Auf Servern mit fehlerhafter TPM-Firmware, die keine gültige Signatur erzeugen können, fällt das Setup kontrolliert auf ein softwarebasiertes Zertifikat zurück. Auch das hat Vorgeschichte: TPM-bezogene ABA-Fehlschläge zogen sich durch mehrere frühere Releases (2.5.79.0, 2.5.190.0), unter anderem wegen Inkompatibilitäten zwischen TPM-Implementierungen und dem Standard-Signaturverfahren der MSAL-Bibliothek.

### PowerShell-Cmdlets verlangen jetzt eine explizite Admin-Anmeldung

Eine Änderung, die Skript-Betreiber auf dem Schirm haben müssen: Die Cmdlets `Set-ADSyncAADCompanyFeature` und `Set-ADSyncAADPasswordSyncState`, die Cloud-Konfiguration verändern, verlangen neu den Parameter `-AADUsername` für eine interaktive Admin-Authentifizierung. Auch der Wizard selbst schreibt Cloud-Änderungen nicht mehr mit gespeicherten Dienst-Credentials, sondern über eine interaktive MSAL-Anmeldung. Und der Uninstall-Wizard fragt für das Aufräumen der Cloud-Konfiguration nach Admin-Credentials; überspringt man das, wird nur lokal aufgeräumt.

Der Hintergrund ist derselbe rote Faden wie bei ABA: Aktionen gegen den Tenant sollen einer echten, nachvollziehbaren Administrator-Identität zugeordnet sein statt einem anonymen Dienstkonto. Das passt zu einem Bugfix im selben Release: Bisher protokollierte das Admin-Audit-Logging bei Änderungen an Synchronisationsregeln die Identität des Dienstkontos statt des tatsächlich handelnden Administrators – ein Audit-Trail, der seinen Zweck verfehlt. Beides zusammen ergibt erst ein brauchbares Auditing. Die praktische Konsequenz: Wer diese Cmdlets bisher unbeaufsichtigt in Skripten aufgerufen hat, muss diese Abläufe umbauen – interaktive Authentifizierung und Automation vertragen sich nicht.

### PHS-Self-Healing entfernt

Die unscheinbarste, aber konzeptionell interessanteste Änderung: Password Hash Synchronization reaktiviert ihr Cloud-Feature-Flag nicht mehr selbstständig im Hintergrund. Ist das Flag deaktiviert, muss ein Administrator es explizit wieder einschalten.

Bisher galt: Wurde PHS auf Tenant-Ebene deaktiviert – bewusst oder versehentlich –, «heilte» sich das Feature von selbst und schaltete sich wieder ein. Für Umgebungen, die PHS absichtlich deaktiviert hatten (etwa aus Compliance-Gründen, weil keine Passwort-Hashes in die Cloud fliessen dürfen, oder während einer Migrationsphase), war das ein Feature, das sich über eine dokumentierte Administratorentscheidung hinwegsetzte. Dass ausgerechnet ein Mechanismus, der Passwort-Hashes synchronisiert, sich eigenmächtig reaktiviert, war schwer vermittelbar.

Die Kehrseite darf man aber nicht verschweigen: Das Self-Healing hat auch Umgebungen gerettet, in denen das Flag durch einen Fehler oder ein missglücktes Skript deaktiviert wurde – ohne dass es jemand bemerkte. Diese Absicherung fällt jetzt weg. Wer PHS produktiv nutzt (und sei es nur als Fallback für die Notfall-Anmeldung), sollte den PHS-Status künftig aktiv überwachen, etwa über Entra Connect Health oder einen Blick auf die Heartbeat-Werte der Synchronisation.

### Aktualisierte Komponenten: SQL LocalDB 2022, MSAL, VC++-Runtime

Weniger spektakulär, aber überfällig ist die Modernisierung der mitgelieferten Komponenten:

- **SQL Server LocalDB 2019 → 2022.** Die interne Datenbank von Entra Connect basierte bisher auf SQL Server 2019 Express LocalDB – einer Version, deren Mainstream-Support im Februar 2025 endete. Mit SQL Server 2022 steht die Installation wieder auf einer Version mit laufendem Support.
- **MSAL 4.64.1 → 4.83.3.** Die Microsoft Authentication Library ist die zentrale Komponente für alle Token-Beschaffung (ABA, Wizard-Anmeldung, PowerShell). Der Sprung über rund zwanzig Minor-Versionen bringt die aufgelaufenen Fixes und Verbesserungen der Bibliothek mit.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Bemerkenswert ist hier weniger das Update als die Altlast: Bis zu diesem Release setzte Entra Connect eine Laufzeitumgebung voraus, deren Support im April 2024 ausgelaufen ist. Die VC++-2013-Abhängigkeit ist jetzt komplett entfernt.

Dazu passt der pauschale Hinweis in den Release Notes, es seien «multiple security vulnerabilities in bundled third-party dependencies» behoben worden. Das dürfte der Hauptgrund für die Einstufung als Sicherheitsrelease sein – veraltete Bundled Components sind bei einem Produkt, das mit Domain-Admin-nahen Rechten im Zentrum der Identitätsinfrastruktur läuft, kein kosmetisches Problem.

## Die übrigen Bugfixes

Der Vollständigkeit halber die restlichen Korrekturen:

- **Metaverse-Suche im Synchronization Service Manager** repariert. Nach der Warnung, die Oberfläche in älteren Versionen gar nicht zu benutzen, wird sie jetzt offenbar wieder gepflegt.
- **PowerShell-Diagnose-Report (HTML)** rendert wieder korrekt – relevant für alle, die `Invoke-ADSyncDiagnostics` für Support-Fälle nutzen.
- **Generic-SQL-Connector:** Die Profilerstellung schlug fehl, weil Pflichtparameter bei der Konfiguration nicht befüllt wurden. Betrifft Umgebungen, die per GSQL-Connector zusätzliche Verzeichnisse anbinden.
- **China Cloud:** Der Instanzname wurde von der Discovery-Endpoint-API nicht korrekt aufgelöst, was die Cloud-Instanz-Erkennung scheitern lassen konnte.
- **Admin-Audit-Logging** protokolliert bei Änderungen an Synchronisationsregeln jetzt den tatsächlichen Administrator statt des Dienstkontos (siehe oben).

## Supportfristen: wer jetzt trotzdem handeln muss

Seit März 2023 gilt für Entra Connect Sync 2.x eine strikte Retirement-Politik: Jede Version fällt zwölf Monate nach Erscheinen der Nachfolgeversion aus dem Support. Die aktuellen Fristen:

| Version | Support-Ende |
| --- | --- |
| 2.5.3.0 | **31. Juli 2026** |
| 2.5.76.0 | 1. September 2026 |
| 2.5.79.0 | 23. Oktober 2026 |
| 2.5.190.0 | 2. Februar 2027 |
| 2.6.1.0 | 10. März 2027 |
| 2.6.3.0 | 7. Juli 2027 |

Wer noch auf 2.5.3.0 unterwegs ist, hat also nur noch zwei Wochen Support-Frist – hier ist die Frage nicht ob, sondern nur auf welche Version aktualisiert wird. Microsoft betont zudem, dass ausser Support geratene Versionen «unexpectedly» aufhören können zu funktionieren; bei den abgekündigten 1.x-Versionen ist die Synchronisation inzwischen tatsächlich serverseitig abgeschaltet. Die Mindestvoraussetzungen bleiben .NET Framework 4.7.2 und TLS 1.2; den Installer gibt es ausschliesslich im Entra Admin Center (Entra ID → Entra Connect → Get started), nicht mehr im Download Center.

## Empfehlung: erstmal abwarten

Microsoft empfiehlt, «so schnell wie möglich» zu aktualisieren. Diese Empfehlung stand allerdings wortgleich auch über Version 2.6.79.0 – jener Version, die anschliessend zurückgezogen wurde. Die jüngere Release-Geschichte (zurückgezogener Installer, Hotfix wegen gestoppter Server, UI-Warnungen über mehrere Versionen) rechtfertigt eine nüchterne Abwägung statt eines Reflexes.

Meine Einordnung für typische Umgebungen:

**Einige Wochen warten ist vertretbar**, wenn Sie auf einer noch unterstützten Version (2.5.190.0 oder neuer) laufen, keines der behobenen Probleme Sie akut trifft und keine der neuen Funktionen gebraucht wird. Die behobenen Sicherheitslücken stecken nach Lage der Release Notes in mitgelieferten Drittkomponenten; ein Entra-Connect-Server sollte ohnehin so abgeschottet sein (kein Internetzugriff ausser zu den Microsoft-Endpunkten, keine interaktiven Anmeldungen, Tier-0-Behandlung), dass sich das Zeitfenster verantworten lässt. Bleibt die Version einige Wochen ohne Rückruf und startet Microsoft den Auto-Upgrade-Rollout, ist das ein deutlich besseres Qualitätssignal als jede Ankündigung.

**Zügig handeln sollten Sie**, wenn einer dieser Punkte zutrifft:

- **Sie haben 2.6.79.0 installiert.** Dann ist die Anweisung eindeutig: deinstallieren und 2.6.84.0 installieren – nicht warten.
- **Sie laufen auf 2.5.3.0** (Support-Ende 31. Juli 2026) oder einer noch älteren, bereits abgelaufenen Version.
- **Eines der behobenen Probleme trifft Sie konkret** – etwa die ABA-Einrichtung auf TPM-Servern, der GSQL-Connector oder die Audit-Anforderung, dass Regeländerungen dem richtigen Administrator zugeordnet werden.

Für das Upgrade selbst gilt das übliche Vorgehen, das bei dieser Release-Historie besonders zu empfehlen ist: Konfiguration vorher exportieren (der Wizard bietet **View or export current configuration**), das Update zuerst auf einem Staging-Mode-Server einspielen und dort Sync-Zyklen, Wizard und Zertifikatsrotation prüfen, erst danach der aktive Server. Wer eine angepasste `miiserver.exe.config` hat, sichert sie vor dem Update und kontrolliert danach, ob der neue Merge-Mechanismus die Anpassungen korrekt übernommen hat. Und wer Skripte mit `Set-ADSyncAADCompanyFeature` oder `Set-ADSyncAADPasswordSyncState` betreibt, testet diese vor dem Produktions-Rollout – sie brechen sonst am neuen Pflichtparameter.

## Quellen

1.  [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history) — Offizielle Release Notes zu 2.6.84.0 samt Rückruf-Hinweis zu 2.6.79.0, Retirement-Tabelle und dem bekannten Problem mit modifizierter miiserver.exe.config.

2.  [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version) — Upgrade-Verfahren inklusive Swing-Migration über einen Staging-Mode-Server.

3.  [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id) — Funktionsweise der Application-Based Authentication, die das Legacy-Dienstkonto ablöst.

4.  [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication) — Die neue Passkey-/FIDO2-Anmeldung im Setup-Wizard über den Windows Web Account Manager.

5.  [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade) — Mechanik und Voraussetzungen des Auto-Upgrade, dessen Rollout für 2.6.84.0 noch aussteht.

6.  [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging) — Das Admin-Audit-Logging, dessen Identitätszuordnung bei Synchronisationsregeln in diesem Release korrigiert wurde.

7.  [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019) — Supportdaten zur bisher mitgelieferten LocalDB-Basis, deren Mainstream-Support im Februar 2025 endete.
