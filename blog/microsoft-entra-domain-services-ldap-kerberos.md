---
title: "Microsoft Entra Domain Services: LDAP und Kerberos für Cloud-only-Umgebungen"
navTitle: "Entra Domain Services"
description: "Entra ID spricht kein LDAP und kein Kerberos. Microsoft Entra Domain Services stellt eine verwaltete Active-Directory-Domäne bereit, die Benutzer aus Entra ID synchronisiert und klassische Protokolle anbietet. Funktionsweise, Grenzen, Kosten und ein Praxisfall mit einem E-Mail-Gateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "6 Min. Lesezeit"
themen:
  - "active-directory-entra"
slug: "microsoft-entra-domain-services-ldap-kerberos"
url: "https://rafaelpfister.ch/blog/microsoft-entra-domain-services-ldap-kerberos"
draft: false
---
# Microsoft Entra Domain Services: LDAP und Kerberos für Cloud-only-Umgebungen

Wer seine Benutzer vollständig nach Microsoft Entra ID (früher Azure Active Directory) verlagert hat, merkt es spätestens bei der ersten Appliance oder Legacy-Anwendung: Entra ID beantwortet Abfragen über Microsoft Graph und moderne Authentifizierungsprotokolle wie OAuth und SAML, aber nicht über LDAP, Kerberos oder NTLM. Ein LDAP-Bind gegen Entra ID ist schlicht nicht möglich. Für alles, was ein klassisches Active Directory erwartet, bietet Microsoft dafür einen eigenen Dienst an: Microsoft Entra Domain Services, früher Azure AD Domain Services.

## Was der Dienst bereitstellt

Entra Domain Services ist eine verwaltete Active-Directory-Domäne. Microsoft betreibt dafür zwei Windows-Domain-Controller in einem Azure-VNet, kümmert sich um Patching, Replikation und Backups und synchronisiert Benutzer und Gruppen automatisch aus Entra ID in die Domäne. Die Synchronisierung läuft nur in eine Richtung, von Entra ID in die verwaltete Domäne; Änderungen direkt in der Domäne fliessen nicht zurück.

Nach aussen verhält sich die Domäne wie ein gewöhnliches Active Directory: Sie beantwortet LDAP- und LDAPS-Abfragen, unterstützt Kerberos- und NTLM-Authentifizierung, erlaubt den Domänenbeitritt von VMs und bietet eingeschränkte Gruppenrichtlinien. Anwendungen und Geräte müssen dafür nicht angepasst werden; sie sehen einen Domain Controller.

## Wofür man ihn braucht

Der Dienst zielt auf Umgebungen, die eigentlich cloud-only sind, aber einzelne Komponenten mit klassischen Verzeichnisanforderungen betreiben:

- **Appliances und Fachanwendungen mit LDAP-Anbindung:** Geräte, die Benutzer per LDAP suchen, Gruppenmitgliedschaften auswerten oder Anmeldungen per LDAP-Bind prüfen.
- **Lift-and-Shift-Migrationen:** Server-Workloads, die domänengebunden bleiben müssen (Kerberos, NTLM, Domänenbeitritt), ohne dass eigene Domain Controller in Azure betrieben werden sollen.
- **Umgebungen ohne lokales AD:** Wo nie ein Active Directory stand oder es abgebaut wurde, ersetzt die verwaltete Domäne den Aufbau eigener DCs samt deren Betriebsaufwand.

Wichtig zur Abgrenzung: Wer noch ein lokales Active Directory mit Entra-Connect-Synchronisierung betreibt, braucht den Dienst in der Regel nicht; die Appliance fragt dann das bestehende AD ab. Entra Domain Services füllt die Lücke, wenn Entra ID die einzige Benutzerquelle ist.

## Architektur und Einrichtung

Die verwaltete Domäne wird in einem eigenen VNet oder Subnetz bereitgestellt und erhält zwei feste Domain-Controller-Adressen. Workloads in anderen VNets erreichen sie über VNet-Peering; die DNS-Server der beteiligten VNets müssen auf die Domain Controller zeigen, damit Domänenname und Objekte auflösbar sind. Der Zugriff wird über Network Security Groups auf die tatsächlich benötigten Quellen und Ports eingeschränkt.

Einige Eigenheiten der verwalteten Domäne, die bei der Anbindung von Anwendungen relevant sind:

- Synchronisierte Benutzer liegen in der OU **AADDC Users**, die Domäne trägt ohne eigene Konfiguration das **onmicrosoft.com**-Suffix. Search Base und Bind-DNs müssen diese Struktur abbilden, etwa CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Es gibt keinen Domain Administrator. Die Verwaltung läuft über die delegierte Gruppe AAD DC Administrators; Schema-Erweiterungen sind nicht möglich.
- Für LDAP-Bind-Konten genügt ein dediziertes, unprivilegiertes Konto; für reine Verzeichnisabfragen in Entra ID die Rolle Directory Readers.

## Die Passwort-Hash-Falle

Ein Punkt kostet in Tests regelmässig Zeit: Kerberos- und NTLM-Anmeldungen sowie LDAP-Binds brauchen Passwort-Hashes in der verwalteten Domäne. Für cloud-only-Konten erzeugt Entra ID diese Hashes erst bei der nächsten Passwortänderung nach der Aktivierung des Dienstes. Ein frisch synchronisierter Benutzer ist also im Verzeichnis sichtbar, kann sich aber erst anmelden, nachdem er sein Passwort einmal geändert hat. Bei hybriden Konten müssen die Hashes per Entra Connect aus dem lokalen AD mitsynchronisiert werden.

## Verschlüsselung: Secure LDAP

Innerhalb der Domäne läuft LDAP standardmässig unverschlüsselt über Port 389. Für Anmeldungen und für jeden Zugriff ausserhalb streng abgeschotteter Netze gehört die Verbindung auf Secure LDAP (LDAPS, Port 636) umgestellt. Dafür wird ein eigenes Zertifikat hinterlegt, dessen Kette die anfragende Anwendung validieren kann; die selbstsignierten Zertifikate der Domain Controller taugen dafür nicht. Microsoft beschreibt die Einrichtung in einem eigenen Tutorial.

## Kosten

Entra Domain Services ist ein Dauerbetriebsposten: Der Dienst wird pro Stunde nach SKU abgerechnet, dazu kommen VNet, Peering und allfällige Test-VMs. Für den einen kleinen LDAP-Anwendungsfall ist das ein stolzer Preis; die Alternative, eigene Domain Controller als VMs zu betreiben, kauft sich die Ersparnis allerdings mit Patching, Backup und Verfügbarkeitsverantwortung zurück.

## Praxisfall: E-Mail-Gateway mit LDAP-Anbindung

Ein konkretes Beispiel für die Appliance-Kategorie ist das SEPPmail Secure E-Mail Gateway. Es nutzt LDAP für Benutzeranlage und Berechtigungsabfragen und seit Firmware 15.0.6 auch für die [Anmeldung an der Admin-GUI](/blog/seppmail-admin-gui-ldap-authentifizierung). Der Hersteller beschreibt in einem eigenen Beitrag, wie eine Appliance im Azure-VNet über Entra Domain Services an eine reine Entra-ID-Benutzerbasis angebunden wird: verwaltete Domäne, VNet-Peering, dediziertes Bind-Konto mit Directory Readers, abgesichert über NSGs. Im Herstellerbeitrag läuft die Verbindung noch unverschlüsselt über Port 389; spätestens für die Admin-GUI-Anmeldung, deren TLS-Option standardmässig aktiv ist, ist Secure LDAP die bessere Wahl.

## Fazit

Entra Domain Services ist kein Ersatz für Entra ID, sondern eine Brücke: Der Dienst übersetzt eine Cloud-Benutzerbasis in eine klassische AD-Domäne für alles, was LDAP, Kerberos oder Domänenbeitritt verlangt. Wer nur eine einzelne Anwendung anbinden muss, sollte die laufenden Kosten gegen eine Modernisierung der Anwendung abwägen. Steht der Dienst einmal, verhalten sich Appliances und Legacy-Anwendungen wie in einer gewohnten AD-Umgebung, inklusive der beschriebenen Eigenheiten bei OU-Struktur, Berechtigungen und Passwort-Hashes.

## Quellen

1.  [Microsoft Learn – «Was ist Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Funktionsumfang der verwalteten Domäne, unterstützte Protokolle und Abgrenzung zu Entra ID und selbst betriebenen Domain Controllern.

2.  [Microsoft Learn – «Synchronisierung in Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): Einweg-Synchronisierung, OU-Struktur und das Verhalten der Passwort-Hashes für cloud-only und hybride Konten.

3.  [Microsoft Learn – «Secure LDAP konfigurieren»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS mit eigenem Zertifikat für verschlüsselte LDAP-Zugriffe.

4.  [SEPPmail – «LDAP-Zugriff mit Azure Active Directory ermöglichen»](https://www.seppmail.com/de/seppmail-ldap-zugriff-mit-azure-active-directory-ermoeglichen/): Herstellerbeitrag zur Anbindung der Appliance über Domain Services mit VNet-Peering und Bind-Konto.

5.  [SEPPmail Admin-GUI an Active Directory anbinden](/blog/seppmail-admin-gui-ldap-authentifizierung): Einrichtung der LDAP-Anmeldung an der Admin-GUI ab Firmware 15.0.6.
