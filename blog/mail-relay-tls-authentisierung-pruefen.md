---
title: "Mailversand über ein Relay: TLS und Authentisierung prüfen"
navTitle: "Relay: TLS prüfen"
description: "Ein Onepager für Application Manager, deren Applikation Mails über ein Relay versendet: Bietet das Relay TLS an, verbindet sich die Applikation verschlüsselt und authentisiert, und verschlüsselt das Relay weiter zum Empfänger? Drei Prüfungen mit openssl s_client und dem Received-Header, ohne Zugriff auf das Relay selbst."
date: "2026-08-28"
kategorie: "SMTP und Mailflow"
timeToRead: "7 Min. Lesezeit"
themen:
  - "smtp-mailflow"
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "tls"
  - "troubleshooting"
slug: "mail-relay-tls-authentisierung-pruefen"
translationId: "article-734e79c4a87105e3"
url: "https://rafaelpfister.ch/blog/mail-relay-tls-authentisierung-pruefen"
---
# Mailversand über ein Relay: TLS und Authentisierung prüfen

Viele Applikationen versenden Mails nicht selbst ins Internet, sondern liefern sie bei einem internen Relay ein: das ERP seine Bestellbestätigungen, das Monitoring seine Alarme, das Ticketsystem seine Benachrichtigungen. Das Relay betreibt das Mail-Team, die Applikationsseite konfiguriert der Application Manager. Bei einem Audit oder einer Schutzbedarfsanalyse landet die Frage darum bei ihm: Verlassen diese Mails das System verschlüsselt, und meldet sich die Applikation sauber am Relay an?

Die Frage hat zwei Teilstrecken, denn TLS bei SMTP wirkt pro Verbindung, nicht Ende-zu-Ende. Die erste Strecke führt von der Applikation zum Relay, die zweite vom Relay zum Zielsystem des Empfängers. Beide lassen sich ohne Zugriff auf das Relay prüfen: die erste mit `openssl s_client` und dem Received-Header einer Testmail, die zweite mit dem Header derselben Testmail in einem externen Postfach.

## Die drei Prüfungen im Überblick

1. **Relay-Angebot:** Bietet das Relay auf dem konfigurierten Port TLS an, mit welchem Zertifikat, und welche Authentisierungsverfahren stehen bereit?
2. **Applikationsseite:** Verbindet sich die Applikation tatsächlich verschlüsselt und authentisiert? Der Beweis steht im Received-Header, den das Relay in jede Mail stempelt.
3. **Weiterversand:** Verschlüsselt das Relay die zweite Strecke zum Empfänger? Der Beweis steht im Header der Testmail im Zielpostfach.

Vorab die Portkunde, weil die Applikationskonfiguration daran hängt:

| Port | Verwendung | TLS-Variante |
|---|---|---|
| 587 | Einlieferung durch Clients und Applikationen (Submission) | STARTTLS: Verbindung startet im Klartext, `STARTTLS` schaltet auf TLS um |
| 465 | Einlieferung durch Clients und Applikationen | Implizites TLS: Verbindung ist von Beginn an verschlüsselt |
| 25 | Mailtransport zwischen Servern | STARTTLS, meist opportunistisch (Klartext-Rückfall möglich) |

In Applikationsmasken heisst STARTTLS oft genau so, implizites TLS dagegen «SSL/TLS» oder «SSL». Eine Fehlkombination (z. B. «SSL» auf Port 587) führt zu Verbindungsabbrüchen, keine Kombination zu unbemerktem Klartext; gefährlicher sind Einstellungen wie «TLS, wenn verfügbar», die bei einem Problem stillschweigend auf unverschlüsselt zurückfallen.

## Prüfung 1: Was bietet das Relay an?

`openssl s_client` ist auf Linux vorhanden und auf Windows Teil von Git for Windows (`C:\Program Files\Git\usr\bin\openssl.exe`). Für einen STARTTLS-Port:

```bash
openssl s_client -starttls smtp \
  -connect relay.example.com:587 -crlf
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-starttls smtp` | Baut erst die SMTP-Klartextverbindung auf und schaltet dann per `STARTTLS` auf TLS um |
| `-connect host:port` | Zielhost und Port; für Submission 587, für den Servertransport 25 |
| `-crlf` | Sendet Zeilenenden als CRLF, wie SMTP es verlangt |

</details>

Für Port 465 entfällt `-starttls smtp`, da die Verbindung dort von Beginn an verschlüsselt ist:

```bash
openssl s_client -connect relay.example.com:465 -crlf
```

Drei Dinge sind in der Ausgabe relevant. Erstens der Verbindungserfolg selbst: Erscheint `CONNECTED` mit anschliessendem Zertifikat, spricht das Relay auf diesem Port TLS; bricht der Handshake ab, bietet es dort keines an. Zweitens die Zeilen `Protocol` und `Cipher`: TLS 1.2 oder 1.3 ist der Sollzustand. Drittens die Zertifikatskette samt `Verify return code`: Ein selbstsigniertes oder abgelaufenes Zertifikat funktioniert zwar mit Applikationen, die die Prüfung abgeschaltet haben, aber genau diese Abschaltung sollte im Audit als Befund auftauchen.

Die Sitzung bleibt nach dem Handshake offen und nimmt SMTP-Befehle entgegen. Mit `EHLO test.example.com` antwortet das Relay mit seiner Capability-Liste; dort zeigt eine Zeile wie `250-AUTH LOGIN PLAIN` die angebotenen Authentisierungsverfahren. Viele Relays nennen `AUTH` erst innerhalb der TLS-Sitzung, damit Zugangsdaten nie im Klartext übertragen werden; fehlt die Zeile auch dort, arbeitet das Relay vermutlich mit einer IP-Freischaltung statt mit Konten. Mit `QUIT` endet die Sitzung.

## Prüfung 2: Verbindet sich die Applikation verschlüsselt und authentisiert?

Die Applikationskonfiguration nennt den Sollzustand (Port, TLS-Modus, Konto), beweist aber nicht, was auf dem Draht passiert. Der Beweis steht im ersten Received-Header, den das Relay beim Empfang der Mail einträgt. Dazu genügt eine Testmail aus der Applikation an das eigene Postfach; dort den Nachrichtenheader anzeigen lassen (Outlook: Datei, Eigenschaften, Internetkopfzeilen; Gmail: Original anzeigen) und die unterste Received-Zeile lesen, denn Header wachsen von unten nach oben:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Das Schlüsselwort nach `with` ist die Kurzfassung des Prüfergebnisses. Die Kennungen sind standardisiert (IANA-Registry «Mail Transmission Types»):

| Kennung | Bedeutung | Bewertung |
|---|---|---|
| `SMTP` / `ESMTP` | unverschlüsselt, ohne Authentisierung | Handlungsbedarf, falls TLS gefordert ist |
| `ESMTPS` | TLS, ohne Authentisierung | in Ordnung bei IP-Freischaltung |
| `ESMTPA` | authentisiert, aber ohne TLS | kritisch: Zugangsdaten liefen im Klartext |
| `ESMTPSA` | TLS und authentisiert | Sollzustand bei Konto-Anmeldung |

Postfix und Exchange ergänzen in Klammern TLS-Version und Cipher, womit sich auch veraltete Protokollversionen erkennen lassen. Steht dort `ESMTPA` ohne S, ist das der dringendste Befund der ganzen Prüfung, denn dann überträgt die Applikation Benutzername und Passwort unverschlüsselt.

Zur Authentisierung selbst gehören zwei Konfigurationsfragen, die der Header nicht beantwortet: Läuft die Anmeldung über ein dediziertes Servicekonto der Applikation (und nicht über ein persönliches Konto, das beim nächsten Austritt deaktiviert wird), und ist das Passwort in der Applikation als Secret hinterlegt statt im Klartext in einer Konfigurationsdatei? Beides klärt ein Blick in die Applikationsdoku und die Konfiguration, nicht in den Mailheader.

Bleibt der Received-Header unklar oder stempelt ein vorgelagerter Loadbalancer die Verbindung um, hilft das Mail-Team mit dem Relay-Log weiter; Postfix protokolliert dort pro Einlieferung eine Zeile `TLS connection established from …` samt Version und Cipher, Exchange Verbindungsdetails im SMTP-Receive-Protokoll.

## Prüfung 3: Verschlüsselt das Relay weiter zum Empfänger?

Die zweite Strecke prüft dieselbe Testmail, diesmal an ein externes Postfach geschickt (z. B. ein privates Gmail- oder Outlook.com-Konto). Im Zielpostfach den vollständigen Header anzeigen und die Received-Zeile suchen, in der das externe System die Mail vom Relay entgegennimmt:

```text
Received: from relay.example.com (relay.example.com [203.0.113.10])
        by mx.google.com with ESMTPS id a1b2c3
        (version=TLS1_3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:06 +0200
```

`ESMTPS` samt Versionsangabe belegt die verschlüsselte Übergabe. Gmail zeigt das Ergebnis zusätzlich in der Detailansicht der Mail als «Standardverschlüsselung (TLS)» an. Für die Auswertung längerer Header-Ketten mit mehreren Stationen nimmt Ihnen der [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) auf dieser Website die Handarbeit ab; er läuft vollständig lokal im Browser, der Header verlässt Ihren Rechner nicht.

Eine Einschränkung gehört in jeden Prüfbericht: Zwischen Mailservern ist TLS üblicherweise opportunistisch. Das Relay verschlüsselt, wenn die Gegenstelle es anbietet, und fällt sonst auf Klartext zurück. Die Header-Prüfung belegt also den Regelfall zu diesem einen Empfänger, keine Garantie für alle Empfänger. Ist garantierte Transportverschlüsselung gefordert (etwa zu einem Partner mit Personendaten), gehört das als erzwungenes TLS für die betreffende Domäne auf das Relay; das ist eine Anforderung an das Mail-Team, keine Applikationseinstellung.

## Kurz-Checkliste für den Prüfbericht

1. Relay-Port und TLS-Modus aus der Applikationskonfiguration notiert (587/STARTTLS, 465/implizit); keine Einstellung «TLS, wenn verfügbar».
2. `openssl s_client` gegen den konfigurierten Port: Handshake erfolgreich, TLS 1.2 oder 1.3, Zertifikat gültig.
3. `EHLO` in der TLS-Sitzung: angebotene `AUTH`-Verfahren notiert, oder IP-Freischaltung als Modell bestätigt.
4. Received-Header der Testmail am Relay: `ESMTPSA` (mit Konto) bzw. `ESMTPS` (mit IP-Freischaltung); `ESMTPA` und `ESMTP` sind Befunde.
5. Anmeldung über ein Servicekonto, Passwort als Secret hinterlegt, Zertifikatsprüfung in der Applikation aktiv.
6. Received-Header im externen Zielpostfach: Übergabe mit `ESMTPS` und aktueller TLS-Version.
7. Falls garantierte Verschlüsselung gefordert: erzwungenes TLS pro Zieldomäne beim Mail-Team beantragt.

## Quellen

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definiert STARTTLS und das Umschalten der Klartextverbindung auf TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definiert SMTP AUTH und die Capability-Anzeige im EHLO-Dialog.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): empfiehlt implizites TLS auf Port 465 für die Einlieferung durch Clients.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): Registry der `with`-Kennungen im Received-Header (ESMTPS, ESMTPA, ESMTPSA).

5.  [OpenSSL-Manpage zu s_client](https://docs.openssl.org/master/man1/openssl-s_client/): vollständige Optionsliste inklusive `-starttls` und Zertifikatsprüfung.
