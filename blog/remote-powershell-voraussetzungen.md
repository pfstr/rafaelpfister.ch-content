---
title: "Voraussetzungen, damit Remote-PowerShell funktioniert"
navTitle: "Remote-PowerShell"
description: "PowerShell-Remoting scheitert selten am Befehl, sondern an den Voraussetzungen: WinRM-Dienst, Listener, Firewall, Authentisierung und die Stolperpunkte bei lokalen Konten. Was auf Ziel- und Clientseite eingerichtet sein muss, wie Sie es mit Test-WSMan prüfen, und warum Access denied meist nichts mit dem Passwort zu tun hat."
date: "2026-09-01"
kategorie: "Windows und PowerShell"
timeToRead: "10 Min. Lesezeit"
themen:
  - "windows-client"
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "remote-powershell-voraussetzungen"
translationId: "article-7315c1ae9e67a24d"
url: "https://rafaelpfister.ch/blog/remote-powershell-voraussetzungen"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
---
# Voraussetzungen, damit Remote-PowerShell funktioniert

`Invoke-Command` und `Enter-PSSession` sind schnell getippt, aber die Verbindung steht erst, wenn auf beiden Seiten die Voraussetzungen erfüllt sind. PowerShell-Remoting setzt auf WS-Management (WinRM) auf, einen SOAP-basierten Verwaltungsdienst über HTTP. Scheitert eine Sitzung, liegt es fast nie am Cmdlet selbst, sondern an einem fehlenden Dienst, einem geschlossenen Port, einer Firewallregel oder der Authentisierung. Dieser Beitrag geht die Voraussetzungen der Reihe nach durch und zeigt, wie Sie jede einzeln prüfen.

Die Begriffe zuerst: Der Zielrechner ist der Rechner, auf dem die Befehle laufen sollen; der Client ist der Rechner, von dem aus Sie sich verbinden. WinRM lauscht standardmässig auf Port 5985 (HTTP) und, wenn eingerichtet, auf Port 5986 (HTTPS). Der HTTP-Verkehr auf 5985 ist auf Nachrichtenebene verschlüsselt, sobald die Authentisierung über Kerberos oder NTLM läuft.

## Die Cmdlets im Überblick

Zur Orientierung die Cmdlets, die in diesem Beitrag vorkommen:

<details class="options-details">
<summary>Optionen im Überblick</summary>

| Cmdlet | Zweck |
|---|---|
| `Enable-PSRemoting` | Richtet WinRM auf der Zielseite ein: Dienst, Listener, Firewallregel |
| `Test-WSMan` | Prüft, ob der WinRM-Dienst der Gegenstelle antwortet |
| `Enter-PSSession` | Öffnet eine interaktive Remote-Sitzung zu einem Rechner |
| `Invoke-Command` | Führt einen Befehlsblock auf einem oder mehreren Rechnern aus |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Trägt vertrauenswürdige Gegenstellen für die Authentisierung ausserhalb einer Domäne ein |
| `Get-Service WinRM` | Zeigt Status und Starttyp des WinRM-Dienstes |

</details>

## Zielseite: WinRM einrichten

Auf dem Zielrechner richtet ein einziger Befehl den Dienst, den Listener und die Firewallregel ein. Führen Sie ihn in einer PowerShell mit Administratorrechten aus:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Force` | Führt ohne Rückfragen aus |
| `-SkipNetworkProfileCheck` | Richtet Remoting auch ein, wenn eine Netzwerkverbindung als öffentlich eingestuft ist |

</details>

`Enable-PSRemoting` startet den WinRM-Dienst, setzt seinen Starttyp auf automatisch, erstellt einen HTTP-Listener und legt die passende Firewallregel an. Ein Vorbehalt betrifft das Netzwerkprofil: Ist eine Netzwerkkarte als öffentlich eingestuft, verweigert der Befehl standardmässig die Einrichtung. Auf Servern oder in kontrollierten Netzen hilft `-SkipNetworkProfileCheck`, damit die Einrichtung trotzdem durchläuft.

Wichtig ist der Geltungsbereich der Firewallregel. Für öffentliche Netzwerkprofile beschränkt die Standardregel den Zugriff auf das lokale Subnetz. Verbinden Sie sich über ein anderes Netz, etwa ein VPN, greift diese Begrenzung, und die Verbindung scheitert trotz laufendem Dienst. Öffnen Sie die Regel dann gezielt für den benötigten Adressbereich, nicht pauschal für alle Adressen:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Wählt die von Enable-PSRemoting angelegten WinRM-HTTP-Regeln über das Namensmuster |
| `-RemoteAddress <Bereich>` | Beschränkt die zulässigen Quelladressen auf den angegebenen Bereich (hier ein CIDR-Block); `Any` erlaubt jede Adresse |

</details>

## Clientseite: TrustedHosts und Dienst

Auf dem Client muss der WinRM-Dienst laufen, sonst schlägt schon das Setzen von Einstellungen fehl. Prüfen Sie das zuerst:

```powershell
Get-Service WinRM
```

Steht der Dienst auf Stopped, starten Sie ihn mit `Start-Service WinRM` (Administratorrechte nötig). Der Starttyp ist auf Clients oft manuell, der Dienst also nach einem Neustart wieder gestoppt. Wenn Sie regelmässig von diesem Rechner aus zugreifen, setzen Sie den Starttyp auf automatisch.

Der zweite Punkt betrifft die Authentisierung ausserhalb einer Domäne. Verbinden Sie sich per IP-Adresse oder in einer Arbeitsgruppe, kann der Client die Gegenstelle nicht über Kerberos prüfen und fällt auf NTLM zurück. Aus Sicherheitsgründen verweigert WinRM das, solange die Gegenstelle nicht als vertrauenswürdig eingetragen ist. Tragen Sie die Zieladresse in die TrustedHosts ein (Administratorrechte nötig):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Value <Liste>` | Vertrauenswürdige Gegenstellen (IP oder Name), mehrere durch Komma getrennt, `*` als Platzhalter |
| `-Force` | Setzt den Wert ohne Rückfrage |
| `-Concatenate` | Hängt an die bestehende Liste an, statt sie zu ersetzen |

</details>

TrustedHosts ist eine Einstellung des Clients, nicht des Zielrechners, und betrifft die Sicherheit des Clients: Die eingetragenen Gegenstellen gelten als vertrauenswürdig, ohne dass ihre Identität kryptografisch geprüft wird. Tragen Sie darum konkrete Adressen ein und nicht den Platzhalter `*`. In einer Domäne mit Kerberos ist der Eintrag nicht nötig; der saubere Weg ausserhalb einer Domäne ohne TrustedHosts ist ein HTTPS-Listener mit einem Zertifikat, dem der Client vertraut.

## Authentisierung: warum Access denied selten am Passwort liegt

Ein häufiges Fehlerbild bei lokalen Konten ist die Meldung Access denied, obwohl das Passwort stimmt. Der Grund ist die Remote-UAC-Filterung: Bei lokalen Konten (nicht dem eingebauten Administrator) entzieht Windows dem Zugriff über das Netz standardmässig die administrativen Rechte. Der Login gelingt, aber jede Aktion mit erhöhten Rechten wird abgewiesen. Meldet die Gegenstelle Access denied statt falscher Anmeldedaten, ist das der wahrscheinliche Grund.

Beheben lässt sich das auf dem Zielrechner mit einem Registry-Wert, der lokalen Administratoren über das Netz die vollen Rechte gibt:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Das ist eine bewusste Lockerung: Lokale Administratorkonten erhalten damit über das Netz volle Rechte. Setzen Sie den Wert nur in kontrollierten Netzen und mit starken Passwörtern. In einer Domäne verwenden Sie besser ein Domänenkonto, dann stellt sich die Frage nicht.

Beim Verbindungsaufbau geben Sie den Benutzernamen bei lokalen Konten mit dem Rechnernamen davor an, damit das Zielsystem das Konto lokal auflöst:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

Im Anmeldedialog tragen Sie den Benutzer als `RECHNERNAME\Benutzer` ein, bei Domänenkonten als `DOMAENE\Benutzer`. Eine PIN aus der Windows-Anmeldung funktioniert über das Netz nicht; nötig ist das Kontopasswort. Bei einem Microsoft-Konto ist es dessen Passwort, und der Kontoname kann vom Anzeigenamen abweichen.

## Prüfen in der richtigen Reihenfolge

Grenzen Sie Fehler von unten nach oben ein, dann sehen Sie schnell, welche Voraussetzung fehlt.

Zuerst die Erreichbarkeit des Ports:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Antwortet der Port nicht, fehlt der Listener oder die Firewall blockiert. Antwortet er, prüfen Sie den WinRM-Dienst der Gegenstelle:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Eine Antwort mit Protokollversion und Hersteller bedeutet: Dienst und Listener stehen. Erst danach testen Sie mit Anmeldedaten:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Gibt dieser Aufruf den Rechnernamen der Gegenstelle zurück, sind alle Voraussetzungen erfüllt.

## Typische Fehler und ihre Ursache

| Meldung oder Symptom | Wahrscheinliche Ursache | Ansatz |
|---|---|---|
| Port 5985 nicht erreichbar | Kein Listener oder Firewall blockiert | `Enable-PSRemoting`, Firewallregel und Geltungsbereich prüfen |
| WinRM cannot complete the operation | Dienst auf der Zielseite aus, oder Zugriff nur aus dem lokalen Subnetz erlaubt | Dienst starten, Firewallregel für den benötigten Adressbereich öffnen |
| The WinRM client cannot process the request … TrustedHosts | Nicht-Domänen-Verbindung ohne TrustedHosts-Eintrag | Zieladresse auf dem Client in TrustedHosts eintragen, oder HTTPS nutzen |
| Access is denied (trotz korrektem Passwort) | Remote-UAC-Filterung bei lokalem Konto | `LocalAccountTokenFilterPolicy` auf 1 setzen, oder Domänenkonto verwenden |
| Zugriff auf eine zweite Ressource scheitert in der Sitzung | Double-Hop: Anmeldedaten werden nicht weitergereicht | Aufgabe direkt auf dem Ziel ausführen, oder CredSSP bzw. eine ausgelagerte Anmeldung einsetzen |

## Grenzen: das Double-Hop-Problem

Eine Voraussetzung lässt sich nicht per Konfiguration wegräumen, sondern nur umgehen: Standardmässig kann eine Remote-Sitzung Ihre Anmeldedaten nicht an ein drittes System weiterreichen. Greifen Sie in einer Sitzung auf dem Zielrechner auf eine Netzwerkfreigabe oder einen weiteren Server zu, scheitert das an fehlenden Anmeldedaten. Dieses Double-Hop-Problem ist eine Sicherheitseigenschaft, keine Fehlkonfiguration. Für die meisten Support-Aufgaben genügt es, den Befehl direkt auf dem Zielrechner auszuführen. Wo die Weitergabe wirklich nötig ist, kommen CredSSP oder eine eingeschränkte Delegierung ins Spiel, beide mit eigenen Sicherheitsabwägungen.

## Quellen

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Voraussetzungen für PowerShell-Remoting, Rechte und Netzwerkprofile.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): Was der Befehl einrichtet, inklusive Netzwerkprofil-Vorbehalt und Firewallregel.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, Authentisierung ausserhalb der Domäne und die häufigen Fehlermeldungen.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): Ursache des Double-Hop-Problems und die Lösungsansätze mit ihren Abwägungen.
