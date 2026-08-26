---
title: "RDP-Druckerumleitung: Drucken über den lokalen PC statt über den Remote-Rechner"
navTitle: "RDP-Druckerumleitung"
description: "Druckaufträge aus der RDP-Sitzung sollen auf dem Drucker neben dem Anwender landen, nicht auf dem fernen Rechner. Die Einstellung dafür sitzt an drei Stellen: im RDP-Client, in der .rdp-Datei und auf dem Zielsystem. Dazu der Umgang mit der Warnung «Unbekannter Herausgeber» und eine Troubleshooting-Checkliste."
date: "2026-08-24"
kategorie: "Windows-Client"
timeToRead: "5 Min. Lesezeit"
themen:
  - "windows-client"
produkte:
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "rdp-druckerumleitung-lokale-drucker"
translationId: "article-12521248666e9809"
url: "https://rafaelpfister.ch/blog/rdp-druckerumleitung-lokale-drucker"
draft: false
---

# RDP-Druckerumleitung: Drucken über den lokalen PC statt über den Remote-Rechner

Ein Anwender arbeitet per Remote Desktop auf einem fernen Rechner und will auf dem Drucker drucken, der neben ihm steht. Genau dafür gibt es die Druckerumleitung: Der RDP-Client meldet die lokalen Drucker in der Sitzung an, der Druckauftrag läuft über den RDP-Kanal zurück zum Client und wird dort ausgegeben. Auf dem Zielsystem taucht der Drucker mit dem Zusatz **(umgeleitet, Sitzung n)** auf. Treiber auf dem fernen Rechner braucht es dafür in der Regel keine: Windows verwendet den Universaltreiber **Remote Desktop Easy Print**; der passende Druckertreiber muss nur auf dem lokalen Client installiert sein.

Die Umleitung greift nur beim Verbindungsaufbau. Nach jeder Änderung an den Einstellungen muss die Sitzung vollständig getrennt und neu verbunden werden; ein blosses Minimieren des RDP-Fensters reicht nicht.

## Client-Seite: die Umleitung aktivieren

Am einfachsten lässt sich die Druckerumleitung über die grafische Oberfläche aktivieren: `mstsc` starten, **Optionen einblenden**, Reiter **Lokale Ressourcen**, Haken bei **Drucker** setzen und die Verbindung im Reiter **Allgemein** speichern. Wer mit .rdp-Dateien arbeitet, kann die Zeile direkt in der Datei anpassen; .rdp-Dateien sind einfache Textdateien und lassen sich mit jedem Editor bearbeiten:

```text
redirectprinters:i:1
```

Ein Stolperstein bei Verknüpfungen ohne .rdp-Datei: Wird die Verbindung mit `mstsc /v:hostname` gestartet, gelten die Einstellungen aus der versteckten Datei `Default.rdp` im Dokumente-Ordner des Anwenders. Fehlt dort die Zeile `redirectprinters:i:1`, bleibt der Drucker weg, obwohl scheinbar alles richtig konfiguriert ist. Dieses Snippet trägt die Zeile idempotent nach (vorhandene `0` wird zu `1`, fehlende Zeile wird ergänzt) und zeigt zur Kontrolle das Ergebnis an:

```powershell
$f = "$env:USERPROFILE\Documents\Default.rdp"
if (Test-Path $f) {
    $c = Get-Content $f
    if ($c -match 'redirectprinters') {
        $c -replace 'redirectprinters:i:0', 'redirectprinters:i:1' | Set-Content $f
    } else {
        Add-Content $f 'redirectprinters:i:1'
    }
} else {
    Set-Content $f 'redirectprinters:i:1'
}
Select-String -Path $f -Pattern 'redirectprinters'
```

Zwei weitere Fallen auf der Client-Seite: Erstens merkt sich Windows pro Zielrechner unter `HKCU\Software\Microsoft\Terminal Server Client\LocalDevices`, welche Umleitungen der Anwender im Sicherheitsdialog zuletzt erlaubt hat; diese gespeicherte Auswahl überschreibt die Vorgabe aus der .rdp-Datei. Das Löschen des Schlüssels setzt den Zustand zurück. Zweitens deaktiviert der Registry-Wert `DisablePrinterRedirection` (DWORD, Wert 1) unter `HKLM\Software\Microsoft\Terminal Server Client` die Druckerumleitung auf dem Client komplett; auf verwalteten Geräten lohnt ein Blick darauf, bevor die Fehlersuche in der Sitzung beginnt.

## Server-Seite: Umleitung erlauben

Auf dem Zielsystem entscheidet die Richtlinie **Clientdruckerumleitung nicht zulassen** (Computerkonfiguration → Administrative Vorlagen → Windows-Komponenten → Remotedesktopdienste → Remotedesktop-Sitzungshost → Druckerumleitung). Steht sie auf *Aktiviert*, werden keine Client-Drucker angelegt, egal was der Client anfordert. Es gilt das Prinzip der restriktivsten Einstellung: Sperrt eine der beiden Seiten die Umleitung, findet sie nicht statt.

Ohne Gruppenrichtlinie steuert derselbe Mechanismus über die Registry: `fDisableCpm` unter `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` (0 = Umleitung erlaubt, 1 = gesperrt). Daneben muss auf dem Zielsystem der Dienst **Druckwarteschlange** laufen; ohne Spooler werden auch umgeleitete Drucker nicht angelegt.

In derselben GPO-Kategorie finden sich zwei weitere nützliche Einstellungen: **Zuerst den Remotedesktop-Easy Print-Druckertreiber verwenden** (Standard und meist die richtige Wahl) und **Standarddrucker des Clients als Standarddrucker der Sitzung festlegen**.

## Die Warnung «Unbekannter Herausgeber»

Beim Öffnen einer unsignierten .rdp-Datei, die Geräteumleitungen anfordert, zeigt der Client eine Sicherheitswarnung mit Checkboxen für die einzelnen Ressourcen. Dort gesetzte oder entfernte Haken gelten nur für diesen einen Verbindungsstart, landen aber im oben erwähnten `LocalDevices`-Schlüssel und wirken so still in künftige Verbindungen hinein. Wer sich wundert, warum der Drucker-Haken trotz korrekter .rdp-Datei immer wieder fehlt, findet die Ursache fast immer dort.

Für den Umgang mit der Warnung gibt es drei Wege, in aufsteigendem Aufwand. Erstens: die Verbindung per `mstsc /v:hostname` statt über die .rdp-Datei starten; ohne Datei gibt es keine Herausgeber-Prüfung, die Einstellungen kommen aus der `Default.rdp`. Zweitens: die Umleitungen für den Zielrechner per Registry vorab genehmigen, dann entfällt der Ressourcen-Teil des Dialogs:

```powershell
$key = "HKCU:\Software\Microsoft\Terminal Server Client\LocalDevices"
New-Item -Path $key -Force | Out-Null
Set-ItemProperty -Path $key -Name "hostname-oder-ip" -Value 0xFFFFFFFF -Type DWord
```

Drittens, der saubere Weg für verteilte .rdp-Dateien im Unternehmen: die Datei mit `rdpsign.exe` und einem Zertifikat signieren und den Zertifikats-Fingerabdruck per GPO als vertrauenswürdigen Herausgeber hinterlegen. Für einzelne Arbeitsplätze lohnt sich der Aufwand selten, für zentral verteilte Verbindungsdateien ist er die richtige Lösung.

## Troubleshooting-Checkliste

Wenn der Drucker in der Sitzung nicht auftaucht, in dieser Reihenfolge prüfen:

1. **Neu verbunden?** Die Umleitung greift nur beim Verbindungsaufbau, nicht in einer bestehenden Sitzung.
2. **Richtige Datei?** Bei Verknüpfungen prüfen, welche .rdp-Datei tatsächlich geöffnet wird; bei `mstsc /v:` zählt die `Default.rdp`.
3. **Gespeicherte Auswahl?** `LocalDevices`-Schlüssel auf dem Client kontrollieren oder löschen.
4. **Client-Sperre?** `DisablePrinterRedirection` unter `HKLM\Software\Microsoft\Terminal Server Client` darf nicht auf 1 stehen.
5. **Server-Sperre?** GPO «Clientdruckerumleitung nicht zulassen» bzw. `fDisableCpm` auf dem Zielsystem prüfen.
6. **Spooler?** Dienst Druckwarteschlange auf dem Zielsystem muss laufen.
7. **Kontrolle in der Sitzung:** `Get-Printer | Where-Object DriverName -eq "Remote Desktop Easy Print"` listet die umgeleiteten Drucker samt Sitzungs-ID.

## Quellen

1.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referenz aller .rdp-Eigenschaften, darunter redirectprinters mit Werten und Default.

2.  [Configure printer redirection over the Remote Desktop Protocol](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-printers): Easy Print, GPO- und Intune-Konfiguration, DisablePrinterRedirection und der Test mit Get-Printer.

3.  [rdpsign](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/rdpsign): Kommandoreferenz zum Signieren von .rdp-Dateien per Zertifikats-Fingerabdruck.
