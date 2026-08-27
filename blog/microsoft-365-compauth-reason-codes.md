---
title: "compauth in Microsoft 365: Composite Authentication und alle Reason-Codes"
navTitle: "compauth-Codes"
description: "Microsoft 365 ergänzt SPF, DKIM und DMARC um eine eigene Bewertung: compauth. Was Composite Authentication prüft, was pass, softpass, fail und none bedeuten und welche Ursache hinter jedem Reason-Code steht, von 000 bis 905."
date: "2026-08-26"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 Min. Lesezeit"
themen:
  - "microsoft-365-exchange"
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
protokolle:
  - "mail-auth"
  - "troubleshooting"
related:
  - "exchange-authmechanism-10-authas-internal"
  - "exchange-hybrid-header-intern-extern"
  - "dns-records-e-mail-stolpersteine"
slug: "microsoft-365-compauth-reason-codes"
translationId: "article-a9dceac9ee095bbd"
url: "https://rafaelpfister.ch/blog/microsoft-365-compauth-reason-codes"
---

# compauth in Microsoft 365: Composite Authentication und alle Reason-Codes

In der `Authentication-Results`-Kopfzeile einer in Microsoft 365 empfangenen Mail steht neben den Standardergebnissen für SPF, DKIM und DMARC ein Microsoft-eigenes Feld:

```text
Authentication-Results: spf=pass (sender IP is 192.0.2.10)
  smtp.mailfrom=example.com; dkim=pass (signature was verified)
  header.d=example.com; dmarc=pass action=none header.from=example.com;
  compauth=pass reason=100
```

`compauth` steht für Composite Authentication: Microsoft 365 kombiniert die Ergebnisse von SPF, DKIM und DMARC mit weiteren Signalen der Nachricht zu einer Gesamtbewertung, ob die sichtbare From-Adresse glaubwürdig ist. Bewertungsbasis ist die From-Domain, also die Adresse, die Empfänger im Mailclient sehen. Damit schliesst Microsoft die Lücke, die entsteht, wenn eine Absenderdomain keine oder unvollständige Authentifizierungs-Records publiziert hat: Auch ohne DMARC-Policy wird implizit geprüft, ob die Mail zur behaupteten Domain passt.

## Die vier Ergebnisse

- `compauth=pass`: Die Nachricht hat die explizite (DMARC) oder implizite Authentifizierung bestanden.
- `compauth=softpass`: Die implizite Prüfung wurde mit geringerer Sicherheit bestanden.
- `compauth=fail`: Die Nachricht ist durch die explizite oder implizite Prüfung gefallen.
- `compauth=none`: Es fand keine Composite-Prüfung statt oder sie wurde übersprungen.

Ein `compauth=fail` führt nicht automatisch zu Quarantäne oder Junk-Ordner. Es ist ein Eingangssignal für die Filter-Entscheidung; massgebend für die tatsächliche Behandlung sind `CAT` und weitere Felder im `X-Forefront-Antispam-Report`. Umgekehrt gilt: Wer wissen will, warum compauth so entschieden hat, braucht den `reason`-Code direkt hinter dem Ergebnis.

## Die Reason-Codes im Überblick

Der dreistellige Code nennt die Regel, die zum Ergebnis geführt hat. Die erste Ziffer gruppiert: 0xx und 6xx sind Fehlschläge, 1xx und 7xx sind bestandene Prüfungen, 2xx ist softpass, 3xx, 4xx und 9xx bedeuten keine oder übersprungene Prüfung.

| Code | Bedeutung |
|---|---|
| `000` | Explizit gescheitert: DMARC-Fail bei einer Policy `p=quarantine` oder `p=reject`. |
| `001` | Implizit gescheitert: Die Domain publiziert keine Authentifizierungs-Records oder nur schwache (SPF `~all`/`?all`, DMARC `p=none`). |
| `002` | Die Organisation hat für dieses Absender/Domain-Paar explizit verboten, gespoofte Mails zu senden (manuell gepflegter Eintrag). |
| `010` | DMARC-Fail bei `p=reject`/`p=quarantine`, und die sendende Domain ist eine eigene Accepted Domain (Spoofing der eigenen Organisation). |
| `100` | SPF oder DKIM bestanden, MAIL FROM- und From-Domain sind aligned. |
| `101` | Die Nachricht ist DKIM-signiert von der From-Domain. |
| `102` | MAIL FROM- und From-Domain aligned, SPF bestanden. |
| `103` / `104` | Die From-Domain passt zum PTR-Record (Reverse Lookup) der einliefernden IP-Adresse. |
| `108` | DKIM-Fail durch eine Body-Änderung auf vorherigen legitimen Stationen, etwa im eigenen OnPrem-Umfeld. |
| `109` | Die Domain hat keinen DMARC-Record, die Prüfung wäre aber bestanden worden. |
| `111` | Trotz DMARC-Temp- oder Permerror ist die SPF- oder DKIM-Domain mit der From-Domain aligned. |
| `112` | Ein DNS-Timeout hat das Abrufen des DMARC-Records verhindert. |
| `115` | Die Mail stammt aus einer Microsoft-365-Organisation, in der die From-Domain als Accepted Domain konfiguriert ist. |
| `116` | Der MX-Record der From-Domain passt zum PTR-Record der einliefernden IP. |
| `130` | Ein als vertrauenswürdig konfigurierter ARC-Sealer hat den DMARC-Fail übersteuert. |
| `201` / `202` | Softpass: Die From-Domain passt zum PTR-Record beziehungsweise zu dessen Subnetz. |
| `3xx` / `4xx` / `9xx` | Keine Composite-Prüfung durchgeführt beziehungsweise übersprungen. |
| `501` / `502` | DMARC nicht durchgesetzt, weil es sich um einen gültigen NDR handelt. |
| `601` | Implizit gescheitert: Die sendende Domain ist eine eigene Accepted Domain (Selbst-Spoofing, häufig bei Direct Send). |
| `701`–`704` | DMARC nicht durchgesetzt, weil die Organisation von dieser Infrastruktur nachweislich legitime Mails empfängt. |
| `905` | DMARC nicht durchgesetzt wegen komplexem Routing, etwa Internet-Mails über OnPrem-Exchange oder einen Drittanbieter-Dienst vor Microsoft 365. |

## Die häufigsten Fälle in der Praxis

**`compauth=fail reason=001`** ist der Standardfall bei Domains ohne oder mit schwacher Authentifizierung. Die Behebung liegt beim Absender: SPF mit `-all`, DKIM-Signierung und eine DMARC-Policy publizieren. Solange die Records fehlen, hängt die Zustellbarkeit an Reputationssignalen.

**`compauth=fail reason=601`** taucht auf, wenn Mails mit der eigenen Domain als Absender von aussen eintreffen, klassisch bei Direct Send: Multifunktionsgeräte, Applikationen oder Dienstleister liefern direkt beim MX ein, ohne authentifizierten Connector. Die Behebung führt über einen korrekt konfigurierten Inbound Connector oder die Aufnahme der Quelle ins eigene SPF.

**`compauth=fail reason=000` oder `010`** bedeutet: DMARC hat regulär gegriffen. Steht daneben `action=oreject`, hat Microsoft 365 die Reject-Policy des Absenders in eine Quarantäne-Zustellung übersetzt. Hier ist nichts zu reparieren, ausser der Absender ist legitim und seine Authentifizierung defekt.

**`reason=108`** und **`reason=130`** betreffen Weiterleitungs- und Gateway-Szenarien: Eine Zwischenstation hat die Mail verändert oder ein vertrauenswürdiger ARC-Sealer hat die ursprünglichen Prüfergebnisse konserviert. Wer ein Gateway vor Microsoft 365 betreibt, sollte dessen ARC-Sealing in der Anti-Spam-Konfiguration als vertrauenswürdig hinterlegen, sonst bleiben legitime Mails an DMARC hängen.

## compauth im Header lesen

In der Praxis steht `compauth` selten allein: Erst das Zusammenspiel mit den einzelnen SPF-, DKIM- und DMARC-Ergebnissen, dem Alignment der beteiligten Domains und der `Received`-Kette ergibt das vollständige Bild. Der [Mail-Header-Analyzer](/tools/header-analyzer) auf dieser Website dekodiert `compauth` samt Reason-Code direkt im Browser und stellt die zugehörigen Domains (From, Envelope-From, `d=`) für die Alignment-Bewertung nebeneinander; der eingefügte Header verlässt den Browser nicht.

## Quellen

1.  [Microsoft Learn: Anti-spam message headers](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo): Offizielle Referenz der Authentication-Results-Felder und der vollständigen compauth-Reason-Code-Tabelle.

2.  [Microsoft Learn: Security Operations guide for email authentication](https://learn.microsoft.com/en-us/defender-office-365/email-auth-sec-ops-guide): Vorgehen bei Authentifizierungs-Fehlschlägen aus SecOps-Sicht.

3.  [Microsoft Learn: Configure trusted ARC sealers](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-arc-configure): Einrichtung vertrauenswürdiger ARC-Sealer für Gateway- und Weiterleitungsszenarien (Reason-Code 130).

4.  [Microsoft Learn: Spam confidence level (SCL)](https://learn.microsoft.com/en-us/defender-office-365/anti-spam-spam-confidence-level-scl-about): Abgrenzung zwischen compauth-Signal und der tatsächlichen Filter-Entscheidung.
