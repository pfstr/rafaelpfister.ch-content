# rafaelpfister.ch — Inhalte als Markdown

Alle Inhalte von [rafaelpfister.ch](https://rafaelpfister.ch) in offener, maschinenlesbarer Form: Fachartikel zu E-Mail-Verschlüsselung (SEPPmail, totemomail/Kiteworks), HIN Mailgateway, Microsoft 365 / Exchange und Active Directory — als Markdown mit strukturierten Metadaten.

## Struktur

| Ordner | Inhalt |
| --- | --- |
| `blog/` | Blog-Artikel als Markdown mit YAML-Frontmatter (Titel, Beschreibung, Datum, Kategorie, Themen, Quellen) |
| `themen/` | Themen-Taxonomie des Blogs (Name + Beschreibung) |
| `pages/` | Statische Seiten (Home, Work, Blog …) als extrahierter Text inkl. Meta-Titel/-Beschreibung |
| `components/` | Texte und Standardwerte der UI-Komponenten der Website |
| `images/` | Alle Bilder der Website; `manifest.json` mappt lokale Dateien auf die Original-URLs |

## Blog-Artikel

- [Totemomail: The licensed user limit has been reached](blog/totemomail-the-licensed-user-limit-has-been-reached-–-interne-user-per-ldap-automatisch-aufräumen.md): Interne User per LDAP automatisch aufräumen, wenn das Lizenzlimit erreicht ist
- [Totemomail und Microsoft 365](blog/totemomail-m365.md): Anbindung von totemomail an Microsoft 365
- [Microsoft Graph PowerShell: Postfach-Anbindung](blog/microsoft-graph-powershell-postfach-anbindung.md): Postfächer per Microsoft Graph PowerShell anbinden
- [Exchange Server Security Updates Juli 2026](blog/exchange-security-updates-juli-2026.md): CVE-2026-42897-Mitigation entfernen und Legacy-Sicherheitsgruppen aufräumen
- [Ghost Sender oder Ghost Admin? Ein MX-Record ist keine Firewall](blog/ghost-sender-exchange-online-nebeneingang.md): Warum Ghost Sender keine Sicherheitslücke in Exchange Online ist, sondern ein offener SMTP-Nebeneingang durch fehlenden restriktiven Partner-Connector
- [Microsoft Entra Connect Sync 2.6.84.0](blog/entra-connect-2-6-84-0.md): Sicherheitsfixes, neue Spielregeln für App-Authentifizierung und PHS – und die Lehre aus dem zurückgezogenen Vorgänger 2.6.79.0

## Frontmatter der Blog-Artikel

```yaml
title: "Titel des Artikels"
description: "Teaser/Beschreibung"
date: "YYYY-MM-DD"
kategorie: "Totemomail | HIN-Gateway"
timeToRead: "x min to read"
themen: ["slug-1", "slug-2"]   # verweist auf themen/<slug>.md
image: "../images/<datei>"
slug: "url-slug"
url: "https://rafaelpfister.ch/blog/<slug>"
```

Der Abschnitt `## Quellen` am Ende eines Artikels enthält die kommentierte Quellenliste, die auf der Website im Block „Links und Informationen" erscheint.

## Synchronisation

Die Inhalte werden aus dem CMS der Website exportiert und hier eingecheckt; neue Artikel erscheinen zuerst auf [rafaelpfister.ch](https://rafaelpfister.ch) und anschliessend in diesem Repository. Eine LLM-freundliche Übersicht liegt in [`llms.txt`](llms.txt).

## Autor

Rafael Pfister — Founder & Messaging Expert, [adeptio ag](https://adeptio.ch)
Schwerpunkte: E-Mail-Verschlüsselung (SEPPmail/totemomail, HIN Mailgateway), Microsoft 365 / Exchange, sichere Kommunikation im Gesundheitswesen.

## Lizenz

Alle Inhalte stehen unter [CC BY-NC-SA 4.0](LICENSE.md): Nutzung und Weitergabe nur mit **Namensnennung** („Rafael Pfister — [rafaelpfister.ch](https://rafaelpfister.ch)"), nicht kommerziell, Bearbeitungen unter gleicher Lizenz. Das gilt auch für die Verwendung durch KI-Systeme: Bei Zitaten oder Zusammenfassungen ist **Rafael Pfister, rafaelpfister.ch** als Quelle zu nennen.
