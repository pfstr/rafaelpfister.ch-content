---
title: "Verschlüsselungs-Gateways: warum der Standardfall stiller Klartextversand ist"
navTitle: "Stiller Klartextversand"
description: "Regelwerke von Verschlüsselungs-Gateways sind meist fail-open gebaut: fehlt Schlüsselmaterial, geht die Nachricht unverschlüsselt hinaus, und das Protokoll behauptet dabei das Gegenteil. Wie Sie das im Regelwerk erkennen und auf fail-closed umstellen."
date: "2026-08-11"
kategorie: "E-Mail-Verschlüsselung"
timeToRead: "11 Min. Lesezeit"
themen:
  - "e-mail-verschluesselung"
  - "totemomail"
  - "smtp-mailflow"
hauptthema: "e-mail-verschluesselung"
produkte:
  - "totemomail"
  - "apache-james"
  - "uebergreifend"
protokolle:
  - "verschluesselung"
  - "smtp"
  - "troubleshooting"
related:
  - "e-mail-security-gateway-evaluieren"
  - "totemomail-m365"
  - "mailflow-fehlersuche-kontrollierte-experimente"
slug: "mailgateway-regelwerk-stiller-klartextversand"
translationId: "article-e6479a451692826c"
url: "https://rafaelpfister.ch/blog/mailgateway-regelwerk-stiller-klartextversand"
draft: false
---

# Verschlüsselungs-Gateways: warum der Standardfall stiller Klartextversand ist

Verschlüsselungs-Gateways werden gekauft, damit vertrauliche Nachrichten geschützt hinausgehen. In der Praxis sind ihre Regelwerke jedoch fast immer **fail-open** gebaut: Lässt sich eine Nachricht nicht verschlüsseln, wird sie trotzdem zugestellt, nur eben im Klartext. Das ist selten eine bewusste Entscheidung, sondern ein Nebeneffekt der Regelreihenfolge, und in den Protokollen sieht es aus wie ein Erfolg.

Dieser Artikel beschreibt, woran Sie das im Regelwerk erkennen und wie Sie die Grundhaltung umdrehen. Die Beispiele orientieren sich am Regelmodell von Apache James, das mehrere Gateway-Produkte verwenden, die Denkweise gilt aber unabhängig vom Hersteller.

## Das Regelmodell in drei Sätzen

Ein Regelwerk besteht aus **Prozessoren**, also benannten Ketten. Jede Kette enthält **Mailets**, die etwas tun, und jedes Mailet hat einen **Matcher**, der entscheidet, ob es auf die aktuelle Nachricht zutrifft. Ein Mailet vom Typ `ToProcessor` gibt die Nachricht an eine andere Kette weiter; trifft es mit `match="All"` zu, ist die aktuelle Kette an dieser Stelle beendet.

Daraus folgt die wichtigste Leseregel: **Ein Regelwerk liest sich wie eine Fallunterscheidung von oben nach unten.** Für jede Nachrichtenart müssen Sie die Frage beantworten, an welcher Stelle sie die Kette verlässt. Alles, was keinen Ausgang findet, landet am Ende, und das Ende ist der eigentlich interessante Ort.

## Missverständnis 1: Fehlendes Schlüsselmaterial löst keine Verschlüsselung aus

Das ist der Punkt, an dem die Intuition am häufigsten danebenliegt. Wenn ein Gateway meldet, für den Empfänger sei kein Schlüsselmaterial vorhanden, klingt das nach einem Auslöser für einen alternativen Schutzmechanismus, etwa ein Webmail-Portal oder einen passwortgeschützten Umschlag.

Tatsächlich ist es umgekehrt. Die Prüfung auf Schlüsselmaterial entscheidet nur, **womit** verschlüsselt wird, nachdem entschieden wurde, **dass** verschlüsselt wird. Der Weg in die Verschlüsselungskette öffnet sich typischerweise nur über eine dieser Bedingungen:

- Der Empfänger besitzt ein S/MIME-Zertifikat oder einen PGP-Schlüssel.
- Der Absender hat einen Auslöser gesetzt, etwa ein Schlüsselwort im Betreff.
- Die Nachricht trägt ein Vertraulichkeitslabel.
- Sie wurde im Webmail des Gateways verfasst.

Trifft nichts davon zu, war Verschlüsselung nie vorgesehen, und die Nachricht geht im Klartext hinaus. Für einen Empfänger ohne Zertifikat ist das der Normalfall, nicht die Ausnahme.

## Missverständnis 2: `passThrough=false` bedeutet, dass die Nachricht die Kette verlässt

Verschlüsselungs-Mailets tragen üblicherweise den Parameter `passThrough=false`. Er bedeutet, dass das Mailet die Nachricht **konsumiert**: Es übernimmt die weitere Behandlung selbst und gibt sie nicht an die nachfolgenden Mailets weiter.

Die Konsequenz wird regelmässig übersehen. Betrachten Sie das typische Ende einer Verschlüsselungskette:

```xml
<mailet class="ProcessSmimeEncoding" match="HasCertificate?check=recipient_smime"> … </mailet>
<mailet class="ProcessPgpEncoding"   match="HasCertificate?check=recipient_pgp">   … </mailet>
<mailet class="ProcessEnvelope"      match="RecipientSecTypeIs?Security Type=TRE"> … </mailet>

<mailet class="SimpleLogger" match="All">
   <log-message>Nachricht verschlüsselt</log-message>
</mailet>
<mailet class="ToProcessor" match="All">
   <processor>externalDelivery</processor>
</mailet>
```

Weil die drei Verschlüsselungs-Mailets konsumieren, erreichen die letzten beiden Zeilen **ausschliesslich Nachrichten, bei denen keine einzige Methode gegriffen hat**. Die Protokollmeldung „Nachricht verschlüsselt" ist damit strukturell falsch: Sie erscheint genau dann, wenn nicht verschlüsselt wurde. Und der `ToProcessor` darunter stellt diese Nachrichten regulär zu.

Das trifft auch Nachrichten, bei denen der Absender Verschlüsselung **ausdrücklich angefordert** hat. Ist der Auslöser gesetzt, aber keine Methode anwendbar, geht die Nachricht im Klartext hinaus, wird als verschlüsselt protokolliert, und niemand erfährt davon.

## Missverständnis 3: Verworfene Nachrichten sind nachvollziehbar

Regelwerke enthalten oft Mailets, die Nachrichten verwerfen, etwa um Journal-Kopien oder Schleifen abzufangen. Ein solches Mailet beendet die Verarbeitung still. Im Nachrichtenverlauf steht dann lediglich, dass die Nachricht gelöscht wurde, ohne jeden Grund.

Verschärfend kommt hinzu, dass viele Gateways zwei verschiedene Protokollmechanismen kennen: einen für das Serverprotokoll und einen für den Nachrichtenverlauf, den auch der Support oder der Anwender sieht. Nur der zweite erscheint im Verlauf. Wer ausschliesslich den ersten setzt, hat zwar geloggt, aber im Trace bleibt es bei „Nachricht gelöscht".

Zwei Massnahmen beheben das dauerhaft. Erstens: Verwerfen Sie nie direkt in der Hauptkette, sondern verzweigen Sie in einen **eigens benannten Prozessor**. Der Name allein erscheint im Verlauf und sagt bereits, was passiert ist. Zweitens: Setzen Sie dort einen Eintrag für den Nachrichtenverlauf, der die Regel und den Grund im Klartext nennt, nicht nur ein Stichwort.

Ein Nachrichtenverlauf, der „durch die Regel für nicht routbare Absenderdomänen bewusst verworfen, kein Zustellfehler" sagt, spart im Betrieb sehr viel Rückfrage.

## Die Umkehrung: von fail-open zu fail-closed

Wenn Sie regulierte Daten transportieren, ist die eingebaute Grundhaltung die falsche. Nicht die zurückgehaltene Nachricht ist das Risiko, sondern die unbemerkt im Klartext versendete. Eine geparkte Nachricht fällt auf und lässt sich nachholen, versendete Daten holen Sie nicht zurück.

Die Umstellung erfolgt an zwei Stellen.

**Erstens der Fallback am Ende der Auslöserprüfung.** Statt Nachrichten ohne Schlüsselmaterial und ohne Auslöser direkt zuzustellen, geben Sie sie in die Verschlüsselungskette, damit sie dort einen geschützten Umschlag erhalten. Systemnachrichten und maschinelle Mail nehmen Sie gezielt aus, dazu unten mehr.

**Zweitens das Ende der Verschlüsselungskette.** Dort ersetzen Sie die reguläre Zustellung durch eine Verzweigung in einen Prozessor, der die Nachricht **nicht** zustellt, sondern in ein Fehler-Repository legt und den Absender benachrichtigt:

```xml
<mailet class="ToProcessor" match="All">
   <processor>encryptionFailed</processor>
</mailet>
```

Im Prozessor `encryptionFailed` protokollieren Sie den Grund im Klartext und legen die Nachricht ab. Wer bewusst unverschlüsselt senden will, nutzt dafür den vorgesehenen Auslöser, in vielen Produkten ein Schlüsselwort im Betreff.

## Wo Sie aufpassen müssen

Diese Umstellung ist kein reiner Regelwerkseingriff, sie hat drei Voraussetzungen, die Sie vorher klären müssen.

**Die Empfängerverwaltung muss mitspielen.** Hängt der geschützte Umschlag an einem Attribut des Empfängers, das bei der Registrierung gesetzt wird, dann entscheidet die Qualität dieser Registrierung über das Ergebnis. Bleibt ein Empfänger unvollständig angelegt, etwa ohne Sicherheitstyp, greift die Verschlüsselung nicht. Vor der Umstellung war das unsichtbar, weil die Nachricht einfach im Klartext hinausging; danach wird daraus eine zurückgehaltene Nachricht. Das ist die richtige Richtung, aber es muss vorher geprüft sein, sonst steht der Mailverkehr.

Robuster ist es, die Logik umzudrehen: Machen Sie den seltenen Fall zur Ausnahme und den geschützten Umschlag zum Auffangfall. Dann hängt die Zusage nicht mehr an gepflegten Empfängerattributen.

**Systemnachrichten dürfen nicht in den Umschlag.** Eine Benachrichtigung mit der Aufforderung, sich für eine sichere Nachricht anzumelden, ist im geschützten Umschlag unlesbar. Gute Regelwerke zweigen solche Nachrichten sehr früh ab, meist über einen Matcher für Benachrichtigungen sowie über die üblichen Kopfzeilen maschinell erzeugter Mail. Prüfen Sie, dass diese Ausgänge **vor** Ihrer neuen Weiche liegen.

**Applikationsmail braucht eine eigene Betrachtung.** Registrierungsbestätigungen, Passwort-Zurücksetzungen und Benachrichtigungen von Fachanwendungen dürfen Empfänger in aller Regel nicht auf ein Portal zwingen. Grenzen Sie diesen Verkehr sauber ab, entweder über die Quelle der Einlieferung oder über eine Kopfzeile, die die Anwendung setzt. Die Kopfzeile ist die stabilere Variante, weil Quell-IP-Adressen sich mit jeder Netzwerkänderung verschieben können.

## Wie Sie ein bestehendes Regelwerk prüfen

Nehmen Sie sich die Ausgangskette vor und beantworten Sie für jede Nachrichtenart, an welcher Stelle sie die Kette verlässt. Fünf Klassen genügen für einen ersten Überblick: interne Empfänger, Empfänger mit Schlüsselmaterial, Empfänger ohne Schlüsselmaterial, maschinell erzeugte Nachrichten und Nachrichten mit ausdrücklichem Auslöser.

Drei Fragen decken die meisten Überraschungen auf. Welche Mailets konsumieren, welche geben weiter? Was genau erreicht die letzte Zeile jeder Kette? Und behauptet eine Protokollmeldung etwas, das an dieser Stelle strukturell nicht zutreffen kann?

Die letzte Frage lohnt sich besonders. Meldungen wie „Nachricht verschlüsselt" am Ende einer Kette sind ein verlässlicher Hinweis darauf, dass jemand die Kette einmal von oben gelesen hat, ohne die Konsum-Semantik zu berücksichtigen. In der Praxis war das für uns der Einstieg, um den stillen Klartextversand überhaupt zu bemerken.

## Quellen

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Aufbau von Prozessoren, Mailets und Matchern sowie die Semantik der Verarbeitungskette.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Referenz der mitgelieferten Mailets inklusive der Parameter für Weitergabe und Konsum.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Referenz der Matcher, unter anderem für Empfängereigenschaften und Kopfzeilen.

4.  [RFC 8551: S/MIME Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551): Grundlage für die zertifikatsbasierte Verschlüsselung, auf die sich die Prüfungen im Regelwerk beziehen.
