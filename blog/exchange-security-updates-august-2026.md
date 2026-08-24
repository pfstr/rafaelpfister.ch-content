---
title: "Exchange-Sicherheitsupdates vom August 2026: Pwn2Own-Lücke geschlossen, OWA Light abgeschaltet"
navTitle: "Exchange SU 08/2026"
description: "Das August-SU schliesst sieben Schwachstellen, darunter den an der Pwn2Own 2026 demonstrierten Exchange-Exploit, und deaktiviert OWA Light endgültig. Dazu erklärt Microsoft, warum Exchange-SUs jetzt monatlich erscheinen und Exchange SE CU1 weiter auf sich warten lässt."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 Min. Lesezeit"
themen:
  - "exchange-updates"
  - "exchange-onprem-hybrid"
produkte:
  - "exchange"
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "exchange-security-updates-august-2026"
translationId: "article-b07bfd4074212673"
url: "https://rafaelpfister.ch/blog/exchange-security-updates-august-2026"
draft: false
---

# Exchange-Sicherheitsupdates vom August 2026: Pwn2Own-Lücke geschlossen, OWA Light abgeschaltet

Microsoft hat am 11. August 2026 Sicherheitsupdates (SUs) für Exchange Server veröffentlicht — bereits den vierten Monat in Folge. Die Updates schliessen sieben Schwachstellen. Keine davon war vorab öffentlich bekannt, keine wird nach aktuellem Stand aktiv ausgenutzt, und Microsoft stuft die Ausnutzung bei allen sieben als «Exploitation Less Likely» ein. Ein Routine-Patchday ist es trotzdem nicht, aus drei Gründen: Das Update schliesst die am Hacking-Wettbewerb Pwn2Own demonstrierte Exchange-Lücke, es schaltet **OWA Light nach fast zwanzig Jahren endgültig ab**, und das Exchange-Team hat im Nachgang erklärt, warum der Monatsrhythmus vorerst der Normalzustand bleibt.

## Für welche Exchange-Versionen das Update verfügbar ist

Die SUs stehen für folgende Versionen bereit:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, Build 15.2.2562.46 — als regulär verfügbares, öffentliches Update.
- **Exchange Server 2019 CU15**: KB5121574, Build 15.2.1748.49 — nur über das **Period-2-ESU-Programm**.
- **Exchange Server 2019 CU14**: KB5121575, Build 15.2.1544.44 — nur über Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, Build 15.1.2507.72 — nur über Period 2 ESU.

Es gilt dieselbe Lage wie im Juli: Exchange 2016 und 2019 sind out of support. Die SUs von Mai bis Oktober 2026 erhält nur, wer im Period-2-ESU-Programm eingeschrieben ist. Alle anderen bleiben ungepatcht auf einem Stand mit inzwischen vierzehn offenen, teils hochbewerteten Schwachstellen — der Wechsel auf Exchange SE duldet dort keinen Aufschub mehr. Exchange Online ist bereits geschützt; in Hybrid-Umgebungen muss das SU trotzdem auf alle Exchange-Server, auch auf reine Management-Server und auf Maschinen, auf denen nur die Exchange Management Tools installiert sind.

Das bekannte Problem der *Wrapper-Nachrichten* in freigegebenen Postfächern von Hybrid-Umgebungen besteht auch mit dem August-SU weiter; der Fix ist laut Microsoft für ein kommendes Update vorgesehen. Immerhin gibt es eine gute Nachricht aus dem Kommentarbereich der Release-Ankündigung: Wer den dokumentierten SettingOverride als Workaround gesetzt hat, muss ihn nach der Installation des August-SUs **nicht** neu anlegen — das Update lässt den Override unangetastet, wie das Exchange-Team dort bestätigt.

## Die sieben Schwachstellen im Überblick

| CVE | Typ | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Remote Code Execution | 8.8 |
| CVE-2026-62911 | Elevation of Privilege | 8.0 |
| CVE-2026-62914 | Spoofing | 7.3 |
| CVE-2026-62910 | Elevation of Privilege | 7.2 |
| CVE-2026-62912 | Denial of Service | 6.5 |
| CVE-2026-62915 | Security Feature Bypass | 6.5 |
| CVE-2026-65813 | Elevation of Privilege | 6.5 |

Drei davon verdienen einen genaueren Blick.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** trägt mit CVSS 8.8 den höchsten Wert des Monats: eine Remote-Code-Execution, die ein authentifizierter Angreifer mit einfachen Rechten ohne jede Benutzerinteraktion auslösen kann. Ein beliebiges kompromittiertes Postfach-Konto genügt als Ausgangspunkt — in Zeiten von Phishing und Credential-Stuffing ist «authentifiziert» keine hohe Hürde.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** ist die einzige Schwachstelle des Monats, die Microsoft als *Critical* einstuft (Elevation of Privilege, CVSS 8.0). Dahinter steckt mehr Geschichte, als die nüchterne Nummer verrät: Auf die Frage, ob der an der **Pwn2Own 2026** von Orange Tsai demonstrierte Exchange-Exploit inzwischen behoben sei, verweist das Exchange-Team im Kommentarbereich der Release-Ankündigung genau auf diese CVE. Der Wettbewerbsfund ist damit geschlossen — ein Grund mehr, das August-SU nicht liegen zu lassen, denn Pwn2Own-Techniken werden nach Ablauf der Sperrfristen üblicherweise im Detail publiziert.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) ist der direkte Anlass für die Abschaltung von OWA Light — dazu gleich mehr.

Die übrigen Lücken: CVE-2026-62910 (EoP, 7.2) setzt bereits hohe Rechte voraus, CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) und CVE-2026-65813 (EoP) liegen bei CVSS 6.5. Die Details stehen wie üblich im Security Update Guide (Filter «Server Software» für Exchange SE bzw. «ESU» für 2016/2019).

## OWA Light: nach fast zwanzig Jahren ist Schluss

### Was das Update ändert

Mit der Installation des August-SUs wird **OWA Light dauerhaft deaktiviert** — auf jedem Server, auf dem das Update (oder ein späteres) landet. Wer die Light-Oberfläche aufruft, landet künftig im normalen Outlook on the web. Die Abschaltung ist Teil des Updates selbst und lässt sich nicht per Schalter rückgängig machen; Microsoft hatte sie einige Wochen vorher in einem eigenen Blogpost angekündigt.

OWA Light stammt aus der Ära von Exchange 2007: eine bewusst schlichte Weboberfläche als Fallback für alte Browser und langsame Verbindungen, seit August 2024 offiziell deprecated. Die Begründung für das Ende ist sicherheitsgetrieben: Ein separater Legacy-Rendering-Pfad neben dem modernen OWA erhöht die Komplexität und damit die Angriffsfläche — CVE-2026-62914 liefert dafür den konkreten Beleg. Wer den [Juli-Artikel](/blog/exchange-security-updates-juli-2026) gelesen hat, erinnert sich zudem: Schon die CVE-2026-42897-Mitigation vom Mai hatte OWA Light nebenbei funktionsunfähig gemacht. Die Oberfläche war also bereits auf Abruf.

### Wer nicht patchen kann: OWA Light manuell abschalten

Wichtig für alle, die das August-SU (noch) nicht installieren können — etwa weil die ESU-Freischaltung fehlt: Microsoft empfiehlt ausdrücklich, OWA Light in diesem Fall **manuell zu deaktivieren**, um CVE-2026-62914 zu entschärfen. Das geht über die OWA-Mailbox-Policy und die Anmeldeseite:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

Der erste Befehl schaltet die Light-Version für alle Postfächer der jeweiligen Policy ab, der zweite entfernt die Auswahl «Light-Version verwenden» von der OWA-Anmeldeseite. Änderungen an der virtuellen OWA-Directory greifen zuverlässig erst nach einem Recycle des OWA-App-Pools bzw. einem `iisreset`.

### Was Admins jetzt prüfen sollten

Die Abschaltung ist technisch trivial, organisatorisch aber nicht immer: OWA Light war der stille Rettungsanker für Nischen-Szenarien. Prüfenswert sind jetzt Lesezeichen und Helpdesk-Anleitungen, die `?layout=light` fest verdrahtet haben, Kiosk- und Terminalgeräte mit alten Browsern sowie interne Anleitungen für Nutzerinnen und Nutzer, die die Light-Version aus Gründen der Barrierefreiheit verwendet haben. Das moderne Outlook on the web läuft in allen aktuellen Browsern und bringt eigene Barrierefreiheits-Funktionen mit — aber wer betroffene Anwender nicht vorab informiert, produziert Tickets.

## Warum jetzt jeden Monat ein SU kommt — und wo Exchange SE CU1 bleibt

Zwei Tage nach dem Release hat das Exchange-Team in einem bemerkenswert offenen Blogpost («Where is Exchange SE CU1 anyway?») die Frage beantwortet, die sich viele Admins stellen. Die Kurzfassung: Microsoft setzt konzernweit KI-Werkzeuge ein, um Schwachstellen in den eigenen Produkten zu finden. Die Teams — Exchange eingeschlossen — arbeiten die gemeldeten Funde derzeit ab: validieren, reproduzieren, fixen, auf Regressionen testen, monatlich ausliefern. Seit Mai 2026 ist so jeden Monat ein Exchange-SU erschienen, und Microsoft sagt ausdrücklich: Dieses erhöhte Tempo wird anhalten.

Das lang erwartete **CU1 für Exchange SE** verzögert sich genau deshalb. Ursprünglich für das erste Halbjahr 2026 angekündigt, dann auf das zweite verschoben, gibt es inzwischen gar kein Zieldatum mehr. Microsoft will CU1 erst veröffentlichen, wenn ein Monat ohne dringende Security-Zulieferung dazwischenliegt — ein CU, das sofort von einem SU überholt wird, würde vielen Organisationen doppelte Update-Arbeit bescheren. Bis dahin fliesst die monatliche Security-Payload laufend in den internen CU1-Build ein.

Für die Praxis heisst das zweierlei. Erstens: Auf CU1 zu warten ist keine Strategie — weder für die Migration auf SE noch für das Einspielen der SUs. Zweitens: Ein **monatliches Wartungsfenster** für Exchange gehört ab sofort fest in den Betriebskalender, so wie es bei Windows-Servern längst selbstverständlich ist.

## Installation und Nachbereitung

Der Ablauf bleibt der bewährte: Zuerst mit dem [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) inventarisieren, welche Server auf welchem CU/SU-Stand sind und ob manuelle Schritte offen sind. Dann das SU installieren (bei veraltetem CU-Stand zeigt der [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) den Pfad), den Server neu starten und kontrollieren, ob alle Exchange-Dienste sauber gestartet sind. Stehen Dienste auf *deaktiviert*, wurde die Installation unterbrochen — dann hilft der dokumentierte Workaround im Microsoft-Support-Artikel zum File-Version-Fehler bzw. das [SetupAssist-Skript](https://aka.ms/ExSetupAssist). Zum Abschluss den Health Checker erneut laufen lassen.

SUs sind kumulativ: Wer das Juli-SU übersprungen hat, installiert direkt das August-SU. Und für Hybrid-Umgebungen gilt der bekannte Zusatz: Wird nach der SU-Installation das Auth-Zertifikat gewechselt, sollte der Hybrid Configuration Wizard erneut ausgeführt werden.

Eine Nacharbeit aus dem Juli bleibt aktuell: Wer die CVE-2026-42897-Mitigation (M2.1.0) immer noch aktiv hat, sollte sie jetzt entfernen — wie das sauber geht, steht im [Artikel zum Juli-SU](/blog/exchange-security-updates-juli-2026).

## Empfohlenes Vorgehen

Kurz zusammengefasst: Das August-SU zeitnah auf allen Exchange-Servern und Management-Tools-Maschinen einspielen — die Pwn2Own-Lücke und die 8.8er-RCE sind Grund genug, nicht auf den nächsten Patchday zu warten. Wer nicht sofort patchen kann, deaktiviert OWA Light manuell als Sofortmassnahme gegen CVE-2026-62914. Vor der OWA-Light-Abschaltung betroffene Nutzergruppen identifizieren und informieren (alte Bookmarks, Kiosk-Browser, Barrierefreiheits-Workflows). Danach Health Checker laufen lassen, offene Nacharbeiten aus dem Juli erledigen — und ein monatliches Exchange-Wartungsfenster einplanen, denn der Rhythmus bleibt.

## Quellen

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Offizielle Release-Ankündigung mit unterstützten Versionen, OWA-Light-Hinweis, Known Issues und FAQ; in den Kommentaren die Bestätigungen zum Pwn2Own-Fix (CVE-2026-62911) und zum fortbestehenden Wrapper-SettingOverride.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): Die Vorankündigung der Abschaltung samt Microsofts Empfehlung, OWA Light bei ausbleibendem Patch manuell zu deaktivieren.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): Das Exchange-Team zur KI-gestützten Schwachstellensuche, zum anhaltenden Monatsrhythmus der SUs und zur CU1-Verzögerung.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referenz für die Build-Nummern der August-SUs.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Bedingungen und Laufzeit (Mai bis Oktober 2026) des ESU-Programms.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Das seit Juni bekannte Hybrid-Problem samt SettingOverride-Workaround.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Deutschsprachige Aufschlüsselung der sieben CVEs mit CVSS-Werten und Builds.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): Der Parameter `OWALightEnabled` zur manuellen Deaktivierung der Light-Version.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventarisierung von CU/SU-Ständen und offenen manuellen Schritten vor und nach der Installation.
