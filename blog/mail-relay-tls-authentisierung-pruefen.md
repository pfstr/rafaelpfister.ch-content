---
title: "Mailversand über ein Relay: TLS und Authentisierung prüfen"
navTitle: "Relay: TLS prüfen"
description: "Ein Onepager für Application Manager, deren Applikation Mails über ein Relay versendet: Welche drei Einstellungen in der Applikation zählen (Port, TLS-Modus, Anmeldung), wie die Optionen in gängigen Umgebungen heissen und wie eine einzige Testmail per Received-Header belegt, dass die Verbindung tatsächlich verschlüsselt und authentisiert läuft."
date: "2026-08-28"
kategorie: "SMTP und Mailflow"
timeToRead: "5 Min. Lesezeit"
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

Viele Applikationen versenden Mails nicht selbst ins Internet, sondern liefern sie bei einem internen Relay ein: das ERP seine Bestellbestätigungen, das Monitoring seine Alarme, das Ticketsystem seine Benachrichtigungen. Das Relay betreibt das Mail-Team; auf der Applikationsseite ist der Application Manager zuständig. Bei einem Audit oder einer Schutzbedarfsanalyse landet die Frage darum bei ihm: Verbindet sich die Applikation verschlüsselt mit dem Relay, und meldet sie sich sauber an?

Die Antwort steht an zwei Stellen, für die kein Mail-Werkzeug und kein Zugriff auf das Relay nötig ist: in der SMTP-Konfiguration der eigenen Applikation und im Header einer einzigen Testmail. Was das Relay selbst anbietet und wie es die Mails weiter zum Empfänger verschlüsselt, verantwortet das Mail-Team; auf der Applikationsseite genügt es, die eigene Teilstrecke zu belegen.

## Wo die Einstellungen stehen

Die SMTP-Konfiguration findet sich je nach Applikation an einem von drei Orten: in der Administrationsoberfläche (meist unter «E-Mail», «Benachrichtigungen», «SMTP» oder «Ausgangsserver»), in einer Konfigurationsdatei oder in Umgebungsvariablen des Deployments (typisch `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` und Varianten). Gesucht sind immer dieselben Angaben: Servername, Port, eine Verschlüsselungsoption und die Zugangsdaten.

## Die drei Einstellungen, die zählen

**Erstens Port und TLS-Modus.** Beide müssen zusammenpassen, denn hinter den Auswahlwerten stehen zwei verschiedene Verfahren: Bei STARTTLS beginnt die Verbindung im Klartext und schaltet dann auf TLS um, beim impliziten TLS (in den Masken meist «SSL/TLS» oder «SSL» genannt) ist sie von Beginn an verschlüsselt.

| Port | TLS-Einstellung in der Applikation | Bewertung |
|---|---|---|
| 587 | STARTTLS | Sollzustand für die Einlieferung durch Applikationen |
| 465 | SSL/TLS (implizit) | ebenfalls in Ordnung |
| 25 | keine oder STARTTLS | üblich bei Relays mit IP-Freischaltung; TLS-Einstellung trotzdem aktivieren, sofern das Relay STARTTLS anbietet |
| beliebig | «Keine» / «None» | Befund: Versand läuft im Klartext |
| beliebig | «TLS, wenn verfügbar» / opportunistisch | Befund: fällt bei einem Problem stillschweigend auf Klartext zurück; auf erzwungenes TLS umstellen |

Eine Fehlkombination (etwa «SSL/TLS» auf Port 587) führt zu Verbindungsabbrüchen, nicht zu unbemerktem Klartext. Die riskanten Einstellungen sind die beiden letzten Zeilen der Tabelle, denn dort versendet die Applikation ohne Fehlermeldung unverschlüsselt.

**Zweitens die Zertifikatsprüfung.** Viele Applikationen bieten eine Option wie «Zertifikat nicht prüfen», «Allow insecure» oder `verify=false` an, die bei Einführungsprojekten gern gesetzt wird, weil das Relay ein internes Zertifikat trägt. Die Verbindung bleibt damit zwar verschlüsselt, aber die Applikation akzeptiert jede Gegenstelle. Ist die Option gesetzt, gehört sie als Befund in den Bericht; die saubere Lösung ist das Vertrauen auf die interne CA statt das Abschalten der Prüfung.

**Drittens die Anmeldung.** Relays kennen zwei Modelle: SMTP AUTH mit Benutzername und Passwort oder eine IP-Freischaltung ohne Konto. Welche Variante gilt, steht in der Relay-Freigabe des Mail-Teams. Bei SMTP AUTH gehören drei Punkte auf die Checkliste: Die Anmeldung läuft über ein dediziertes Servicekonto der Applikation (nicht über ein persönliches Konto, das beim nächsten Austritt deaktiviert wird), das Passwort ist als Secret hinterlegt statt im Klartext in einer Konfigurationsdatei, und die TLS-Option ist aktiv, denn die gängigen Verfahren PLAIN und LOGIN übertragen die Zugangsdaten sonst im Klartext.

## So heissen die Einstellungen in gängigen Umgebungen

| Umgebung | Verschlüsselung | Anmeldung |
|---|---|---|
| Admin-Oberflächen (ERP, Monitoring, Appliances) | Dropdown «Verschlüsselung»: None / STARTTLS / SSL-TLS | Felder Benutzername/Passwort; leer = keine Anmeldung |
| Java (Jakarta Mail, Spring) | `mail.smtp.starttls.enable=true` plus `mail.smtp.starttls.required=true`; für Port 465 `mail.smtp.ssl.enable=true` | `mail.smtp.auth=true` |
| .NET | `SmtpClient.EnableSsl=true` (macht STARTTLS); MailKit: `SecureSocketOptions.StartTls` | `Credentials` bzw. `Authenticate()` |
| PHP (PHPMailer) | `SMTPSecure='tls'` für 587, `'ssl'` für 465 | `SMTPAuth=true` |
| Python (smtplib) | `starttls()` nach dem Verbindungsaufbau oder `SMTP_SSL` für 465 | `login()` |
| Node.js (Nodemailer) | Port 465: `secure:true`; Port 587: `secure:false` plus `requireTLS:true` | `auth: {user, pass}` |

Zwei Punkte aus dieser Tabelle sind erfahrungsgemäss die häufigsten Befunde: In Java aktiviert `starttls.enable` allein nur opportunistisches TLS, erst `starttls.required` verhindert den Klartext-Rückfall. In Nodemailer bedeutet `secure:false` nicht «unverschlüsselt», sondern «kein implizites TLS»; ohne `requireTLS:true` bleibt STARTTLS aber ebenfalls opportunistisch.

## Gegenprobe: eine Testmail und ihr Received-Header

Die Konfiguration nennt den Sollzustand, beweist aber nicht, was auf dem Draht passiert. Der Beweis steht im Received-Header, den das Relay beim Empfang jeder Mail einträgt. Dazu genügt eine Testmail aus der Applikation an das eigene Postfach; dort den Nachrichtenheader anzeigen lassen (Outlook: Datei, Eigenschaften, Internetkopfzeilen; Gmail: Original anzeigen) und die unterste Received-Zeile lesen, denn Header wachsen von unten nach oben:

```text
Received: from app01.example.com (app01.example.com [10.1.2.3])
        by relay.example.com (Postfix) with ESMTPSA id 4XyZk12Fzq
        (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
        Thu, 28 Aug 2026 09:15:04 +0200
```

Das Schlüsselwort nach `with` ist die Kurzfassung des Prüfergebnisses. Die Kennungen sind standardisiert (IANA-Registry «Mail Transmission Types»):

| Kennung | Bedeutung | Bewertung |
|---|---|---|
| `SMTP` / `ESMTP` | unverschlüsselt, ohne Anmeldung | Handlungsbedarf, falls TLS gefordert ist |
| `ESMTPS` | TLS, ohne Anmeldung | in Ordnung bei IP-Freischaltung |
| `ESMTPA` | angemeldet, aber ohne TLS | kritisch: Zugangsdaten liefen im Klartext |
| `ESMTPSA` | TLS und angemeldet | Sollzustand bei SMTP AUTH |

Postfix und Exchange ergänzen in Klammern TLS-Version und Cipher, womit sich auch veraltete Protokollversionen erkennen lassen. Für die Auswertung längerer Header mit mehreren Stationen nimmt Ihnen der [Mail-Header-Analyzer](https://rafaelpfister.ch/tools/header-analyzer) auf dieser Website die Handarbeit ab; er läuft vollständig lokal im Browser, der Header verlässt Ihren Rechner nicht.

Bleibt der Header unklar oder stempelt ein vorgelagerter Loadbalancer die Verbindung um, ist das der Moment für eine Anfrage an das Mail-Team: Das Relay-Log protokolliert pro Einlieferung, ob TLS ausgehandelt wurde und mit welchem Konto sich die Applikation angemeldet hat.

## Kurz-Checkliste für den Prüfbericht

1. SMTP-Konfiguration der Applikation gefunden (Oberfläche, Konfigurationsdatei oder Umgebungsvariablen) und dokumentiert.
2. Port und TLS-Modus passen zusammen (587/STARTTLS oder 465/SSL-TLS); keine Einstellung «Keine» oder «TLS, wenn verfügbar».
3. Zertifikatsprüfung aktiv; ein gesetztes «Zertifikat nicht prüfen» ist als Befund erfasst.
4. Anmeldemodell geklärt: SMTP AUTH mit Servicekonto und Secret-Ablage, oder IP-Freischaltung laut Relay-Freigabe.
5. Received-Header der Testmail zeigt `ESMTPSA` (mit Konto) bzw. `ESMTPS` (mit IP-Freischaltung); `ESMTPA` und `ESMTP` sind Befunde.
6. Falls die Verschlüsselung bis zum Empfänger gefordert ist: als Anforderung an das Mail-Team adressiert, denn die Strecke ab Relay liegt ausserhalb der Applikation.

## Quellen

1.  [RFC 3207: SMTP Service Extension for Secure SMTP over Transport Layer Security](https://www.rfc-editor.org/rfc/rfc3207): definiert STARTTLS und das Umschalten der Klartextverbindung auf TLS.

2.  [RFC 4954: SMTP Service Extension for Authentication](https://www.rfc-editor.org/rfc/rfc4954): definiert SMTP AUTH und die Verfahren wie PLAIN und LOGIN.

3.  [RFC 8314: Cleartext Considered Obsolete](https://www.rfc-editor.org/rfc/rfc8314): empfiehlt implizites TLS auf Port 465 für die Einlieferung durch Clients.

4.  [IANA: Mail Transmission Types](https://www.iana.org/assignments/mail-parameters/mail-parameters.xhtml#mail-parameters-7): Registry der `with`-Kennungen im Received-Header (ESMTPS, ESMTPA, ESMTPSA).

5.  [Jakarta Mail: Package com.sun.mail.smtp](https://jakarta.ee/specifications/mail/2.1/apidocs/jakarta.mail/com/sun/mail/smtp/package-summary): dokumentiert die Properties mail.smtp.starttls.enable, starttls.required und ssl.enable.
