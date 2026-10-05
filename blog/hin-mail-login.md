---
title: "HIN Mail Login: Webmail, Outlook-Token, Passwort vergessen und HIN Mail Global"
navTitle: "HIN Mail Login"
description: "Wo und wie Sie sich bei HIN Mail anmelden: Webmail unter webmail.hin.ch, Mail-Token für Outlook und Smartphone, Login ohne HIN Client per SMS-Code oder Authenticator App, Passwort-Reset und das Öffnen von HIN Mail Global als Empfänger ohne HIN-Konto."
date: "2026-10-05"
kategorie: "HIN-Gateway"
timeToRead: "7 Min. Lesezeit"
themen:
  - "hin-gateway"
produkte:
  - "hin"
  - "outlook"
protokolle:
  - "troubleshooting"
  - "verschluesselung"
related:
  - "hin-plattformerneuerung-2026"
  - "hin-mailgateway-backup-disaster-recovery"
slug: "hin-mail-login"
translationId: "article-6f466c817c7c7c66"
url: "https://rafaelpfister.ch/blog/hin-mail-login"
---

# HIN Mail Login: Webmail, Outlook-Token, Passwort vergessen und HIN Mail Global

Hinter der Suche nach «HIN Mail Login» stehen meist drei verschiedene Situationen: Sie sind HIN-Mitglied und wollen Ihre Mails im Browser lesen, Sie wollen HIN Mail in Outlook oder auf dem Smartphone einrichten, oder Sie haben als Patientin, Patient oder externe Stelle eine verschlüsselte HIN-Mail erhalten und haben gar kein HIN-Konto. Die Anmeldung läuft in allen drei Fällen anders.

| Situation | Adresse | Benutzername | Passwort |
|---|---|---|---|
| Webmail im Browser | `webmail.hin.ch` | HIN ID | HIN-Passwort, abgesichert durch HIN Client, SMS-Code oder Authenticator App |
| Outlook, Thunderbird, Apple Mail, Smartphone | `imap.mail.hin.ch`, `smtp.mail.hin.ch` | HIN ID | Mail-Token aus `apps.hin.ch` |
| Verwaltung, Token, Initialisierungspasswort | `apps.hin.ch` | HIN ID | HIN-Passwort plus zweiter Faktor |
| Empfänger ohne HIN-Konto (HIN Mail Global) | Anhang `secure-email.html` bzw. Link in der Mail | Mobilnummer | SMS-Code |

## HIN Webmail: Login im Browser

Das Webmail erreichen Sie unter `https://webmail.hin.ch`. Sie melden sich mit Ihrer HIN ID (dem HIN-Loginnamen, z. B. `hmuster`) und Ihrem HIN-Passwort an. Den zweiten Faktor liefert eine der folgenden Methoden:

- **HIN Client:** Läuft der Client auf dem Arbeitsplatz und ist angemeldet, öffnet sich das Webmail ohne weitere Abfrage. Mit HIN Client 4.0 erfolgt die Anmeldung direkt im Browser: Sie geben die HIN-geschützte Adresse ein und werden zur Anmeldung weitergeleitet.
- **SMS-Code (mTAN):** Nach dem Passwort erhalten Sie einen Code auf die hinterlegte Mobilnummer. Diese Methode eignet sich für Geräte ohne HIN Client, etwa einen privaten Laptop.
- **HIN Authenticator App:** Die App (App Store und Google Play, «HIN Authenticator») bestätigt die Anmeldung auf dem Smartphone.

SMS-Code und Authenticator App müssen vorgängig aktiviert sein. Fehlen beide und ist der HIN Client nicht verfügbar, ist keine Anmeldung am Webmail möglich. Richten Sie deshalb mindestens eine Alternative ein, solange der Client noch funktioniert.

Im Webmail lassen sich Mails schreiben und verwalten, Kontakte und Adressbücher pflegen, das HIN-Teilnehmerverzeichnis durchsuchen und mehrere Signaturen anlegen. Unter dem Benutzernamen zeigt ein Balken den freien Speicherplatz des Postfachs; ist er rot, müssen Sie Mails löschen oder den Speicher erweitern.

## HIN Mail in Outlook und auf dem Smartphone: Login mit Mail-Token

Mailprogramme melden sich nicht mit dem HIN-Passwort an, sondern mit einem **Mail-Token**. Das ist die häufigste Ursache für eine wiederholte Passwortabfrage in Outlook: Das HIN-Passwort wird beim IMAP-Login abgelehnt.

So erzeugen Sie das Token:

1. `https://apps.hin.ch` öffnen und mit der HIN ID anmelden (HIN Client, SMS-Code oder Authenticator App).
2. Den Bereich «HIN Mail» wählen.
3. «Gerät hinzufügen» anklicken und als Gerätetyp «Mail Clients» wählen.
4. Das angezeigte Token kopieren und im Mailprogramm als Passwort eintragen.

Die Servereinstellungen:

| Einstellung | Posteingang (IMAP) | Postausgang (SMTP) |
|---|---|---|
| Server | `imap.mail.hin.ch` | `smtp.mail.hin.ch` |
| Port | `993` | `587` |
| Verschlüsselung | SSL/TLS | STARTTLS |
| Benutzername | HIN ID | HIN ID |
| Passwort | Mail-Token | Mail-Token |
| Authentifizierung | normal | «Postausgangsserver erfordert Authentifizierung» aktivieren |

Als Benutzername gilt die HIN ID, nicht die E-Mail-Adresse. Outlook schlägt bei der automatischen Einrichtung die E-Mail-Adresse vor; die Einrichtung muss deshalb manuell über «Erweiterte Optionen» bzw. «Manuelle Einrichtung» erfolgen.

Das Token ist an Ihre HIN-Identität gebunden und lässt sich in `apps.hin.ch` jederzeit widerrufen und neu erzeugen. Für jedes Gerät empfiehlt sich ein eigenes Token: Geht ein Smartphone verloren, sperren Sie nur dieses Gerät, und Outlook auf dem Praxis-PC läuft weiter. Ist das Token einmal eingerichtet, braucht Outlook den HIN Client für den Mailabruf nicht mehr.

## Erste Anmeldung ohne HIN Client: Identität aktivieren

Neue HIN-Identitäten lassen sich ohne HIN Client aktivieren. Sie benötigen den Benutzernamen und das Initialisierungspasswort aus den HIN-Unterlagen.

1. Den Aktivierungslink im Servicecenter (`servicecenter.hin.ch`) aufrufen.
2. Benutzernamen und Initialisierungspasswort eingeben.
3. Ein eigenes Passwort setzen: mindestens 10 Zeichen mit Ziffern, Gross- und Kleinbuchstaben sowie Sonderzeichen.
4. Sofort einen zweiten Faktor einrichten (SMS-Code oder Authenticator App).
5. Anschliessend unter `apps.hin.ch` anmelden.

Für die Aktivierung stehen 10 Minuten zur Verfügung. Danach wird das Initialisierungspasswort aus Sicherheitsgründen zurückgesetzt, und Sie müssen ein neues anfordern.

## HIN-Passwort vergessen

Der Reset hängt davon ab, ob eine alternative Anmeldemethode eingerichtet ist.

**Mit SMS-Code oder Authenticator App:**

1. Unter `apps.hin.ch` mit der alternativen Methode anmelden.
2. Den Reiter «HIN Client» wählen und ein neues Initialisierungspasswort erzeugen.
3. Im HIN Client «Registrierung» wählen, Loginnamen und das neue Initialisierungspasswort eingeben.
4. Ein neues Passwort setzen und «Neue HIN Identität registrieren» bestätigen.

**Ohne alternative Methode und ohne angemeldetes Gerät** bleibt nur der telefonische HIN Support (0848 830 740, Montag bis Freitag 08:00 bis 18:00 Uhr). Notieren Sie das neue Initialisierungspasswort, bevor Sie das Fenster schliessen.

## HIN Mail Global: verschlüsselte Mail ohne HIN-Konto öffnen

Arztpraxen, Spitäler und Labors senden Nachrichten an Patientinnen, Patienten oder Stellen ohne HIN-Mitgliedschaft über HIN Mail Global. Die Mail kommt bei Ihnen als normale E-Mail an, der Inhalt liegt jedoch verschlüsselt im Anhang `secure-email.html` bzw. hinter dem Link «Verschlüsselte Nachricht lesen». Ein HIN-Login im eigentlichen Sinn gibt es hier nicht; die Anmeldung erfolgt über Ihre Mobilnummer.

**Beim ersten Mal:**

1. Den Anhang `secure-email.html` herunterladen und öffnen (oder den Link in der Mail anklicken). Bei Webmail-Diensten wie Gmail, Bluewin oder GMX zuerst speichern, dann öffnen.
2. Sprache wählen und die eigene Mobilnummer eingeben.
3. Den per SMS erhaltenen Code eingeben. Die Nachricht wird angezeigt.

**Bei weiteren Nachrichten** genügt das Öffnen des Anhangs; der SMS-Code geht automatisch an die registrierte Nummer.

Zu beachten:

- Die Nachricht ist **30 Tage** lang über den Link abrufbar, danach läuft der Zugriff ab. Speichern oder drucken Sie wichtige Inhalte innerhalb dieser Frist.
- Weitergeleitete Nachrichten kann nur der ursprüngliche Empfänger öffnen.
- Sie können verschlüsselt antworten.
- Ein Gerät lässt sich als vertrauenswürdig markieren; die SMS-Abfrage entfällt dann für ein Jahr.
- Kommt innerhalb von 15 Minuten kein Code an, können Sie einen neuen anfordern.
- Unterstützt werden die zwei aktuellen Versionen von Chrome, Safari, Edge und Firefox.

Bei Problemen mit HIN Mail Global hilft der HIN-Mail-Support unter +41 58 670 48 69. Fragen zum Inhalt der Nachricht beantwortet nur der Absender.

## Häufige Fehler beim HIN Mail Login

| Symptom | Ursache | Lösung |
|---|---|---|
| Outlook fragt ständig nach dem Passwort | HIN-Passwort statt Mail-Token eingetragen | Token in `apps.hin.ch` erzeugen und als Passwort eintragen |
| Login in Outlook schlägt trotz Token fehl | E-Mail-Adresse als Benutzername | HIN ID als Benutzername verwenden |
| Senden schlägt fehl, Empfang funktioniert | SMTP-Authentifizierung nicht aktiviert, falscher Port oder falsche Verschlüsselung | Port `587` mit STARTTLS, Authentifizierung aktivieren |
| Webmail verlangt einen Code, der nie ankommt | Mobilnummer veraltet oder SMS-Code nicht aktiviert | Authenticator App nutzen oder über den HIN Client anmelden und die Nummer aktualisieren |
| Initialisierungspasswort wird abgelehnt | 10-Minuten-Frist überschritten | Neues Initialisierungspasswort anfordern |
| Token funktionierte, plötzlich nicht mehr | Token in `apps.hin.ch` widerrufen oder Gerät entfernt | Neues Token für dieses Gerät erzeugen |
| HIN Mail Global: Link öffnet nichts mehr | 30-Tage-Frist abgelaufen | Absender um erneuten Versand bitten |

## HIN Client 4.0: was sich am Login ändert

HIN verteilt den HIN Client 4.0 bis zum 26. Oktober 2026 automatisch; wer kein automatisches Update erhält, lädt ihn unter `download.hin.ch` herunter. Das bestehende Passwort bleibt gültig und wird bei der ersten Anmeldung mit der neuen Version einmal bestätigt. Die Anmeldung an HIN-geschützten Diensten wie dem Webmail erfolgt danach direkt im Browser.

Version 4.0 setzt Windows 11 mit TPM oder macOS 14 mit Security Chip voraus. Geräte, die diese Anforderungen nicht erfüllen, können den Client nicht mehr nutzen. Der Zugang zum Webmail bleibt dort über SMS-Code oder Authenticator App möglich, der Mailabruf in Outlook über das Mail-Token. Details zu den Fristen der Plattformerneuerung stehen im Artikel [HIN Plattformerneuerung 2026](/blog/hin-plattformerneuerung-2026).

## Praxen und Spitäler mit eigenem Mailserver

Organisationen mit eigenem HIN Mailgateway melden sich nicht bei HIN Mail an. Die Postfächer liegen auf dem eigenen Exchange oder in Exchange Online, und das Gateway (eine SEPPmail-Appliance) verschlüsselt die Mails auf dem Weg zu und von der HIN-Community. Login-Probleme betreffen dort das eigene Mailsystem, nicht die HIN-Plattform. Dieses Gateway wird durch das neue HIN Gateway («Stargate») abgelöst.

<aside class="offer-box">
  <span class="offer-box__tag">Kostenloser Check</span>
  <p><strong>Betreiben Sie ein eigenes HIN Mailgateway?</strong> Ich prüfe Ihre Umgebung und sage Ihnen, was vor der Migration auf «Stargate» zu erledigen ist.</p>
  <a class="offer-box__cta" href="/stargate">Jetzt registrieren</a>
</aside>

## Quellen

1.  [HIN Support: HIN Mail und Mobile](https://support.hin.ch/de/thema/hin-mail-mobile/): Übersicht mit Webmail-Adresse und Anleitungen für Outlook, Apple Mail, Thunderbird, iOS und Android.

2.  [HIN Support: Wie funktioniert das Webmail?](https://support.hin.ch/de/service/hin-mail-und-mobile/wie-funktioniert-das-webmail.cfm): Funktionen des Webmails, Speicheranzeige.

3.  [HIN Support: Anleitung Mail Token Service für Mail Clients](https://support.hin.ch/de/hin-mail-mobile/mail-token-service-mail-clients/): Token-Erzeugung in apps.hin.ch, IMAP- und SMTP-Server, Ports und Verschlüsselung.

4.  [HIN Support: Was ist ein Token?](https://support.hin.ch/de/service/hin-mail-und-mobile/was-ist-ein-token.cfm): Mail-Token und Access-Token, Bindung an die HIN-Identität.

5.  [HIN Support: HIN Identität aktivieren (ohne HIN Client)](https://support.hin.ch/de/hin-2fa/hin-identitaet-aktivieren/): Aktivierung im Servicecenter, 10-Minuten-Frist, Passwortregeln, zweiter Faktor.

6.  [HIN Support: Passwort vergessen](https://support.hin.ch/de/thema/hin-client/passwort-vergessen.cfm): Reset über Initialisierungspasswort, telefonischer Support ohne alternative Methode.

7.  [HIN Support: HIN Mail an Nichtmitglieder](https://support.hin.ch/en/service/hin-mail-to-non-members.cfm): 30 Tage Abrufbarkeit, SMS-Registrierung, vertrauenswürdige Geräte, unterstützte Browser, Support-Telefon.

8.  [HIN: HIN Mail Global](https://www.hin.ch/services/hin-mail/hin-mail-global/): Versand verschlüsselter Mails an Empfänger ohne HIN-Mitgliedschaft.

9.  [HIN Blog: Der neue HIN Client ist da](https://www.hin.ch/de/blog/2026/neuer-hin-client.cfm): Browser-Login, Systemvoraussetzungen mit TPM bzw. Security Chip, Rollout bis 26. Oktober 2026.

10.  [HIN Authenticator im App Store](https://apps.apple.com/ch/app/hin-authenticator/id1535944002): App für den zweiten Faktor.
