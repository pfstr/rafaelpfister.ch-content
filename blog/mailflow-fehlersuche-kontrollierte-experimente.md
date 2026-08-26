---
title: "Was wir aus der Naturwissenschaft für die Fehlersuche in der IT lernen können"
navTitle: "Kontrollierte Experimente"
description: "Falsifizierbarkeit, Kontrollgruppe, Störvariablen und Stichprobenverzerrung: Die Methode, mit der Naturwissenschaften seit Jahrhunderten arbeiten, löst genau die Probleme, an denen IT-Fehlersuche regelmässig scheitert. Mit durchgespielten Beispielen aus dem Mailfluss."
date: "2026-08-11"
kategorie: "SMTP / Mailfluss"
timeToRead: "15 Min. Lesezeit"
themen:
  - "smtp-mailflow"
  - "testing"
  - "exchange-onprem-hybrid"
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
  - "exchange"
protokolle:
  - "testing"
  - "smtp"
  - "troubleshooting"
related:
  - "exchange-message-tracking-und-receive-connectoren-analysieren"
  - "einliefernde-ip-adressen-aggregieren"
  - "typische-ursachen-fuer-mail-loops-und-deren-behebung"
slug: "mailflow-fehlersuche-kontrollierte-experimente"
translationId: "article-098ed40e6d027b8b"
url: "https://rafaelpfister.ch/blog/mailflow-fehlersuche-kontrollierte-experimente"
draft: false
---

# Was wir aus der Naturwissenschaft für die Fehlersuche in der IT lernen können

Eine Nachricht kommt nicht an. Das Protokoll liefert eine Fehlermeldung, die sofort eine Erklärung nahelegt. Sie prüfen diese Erklärung, finden Belege, und nach zwei Stunden stellt sich heraus, dass die Erklärung falsch war und die Belege Zufall.

Das ist kein Anfängerfehler, sondern die Regel. Und es ist bemerkenswert, dass unsere Branche für dieses Problem selten eine Methode hat, obwohl eine seit Jahrhunderten existiert und ausgesprochen gut funktioniert. Die Naturwissenschaften haben genau dieselbe Aufgabe: aus Beobachtungen auf Ursachen schliessen, in Systemen, die man nicht vollständig überblickt.

Dieser Artikel überträgt fünf Grundsätze der wissenschaftlichen Methode auf die Fehlersuche im Mailfluss. Die Beispiele stammen aus der Praxis, das Vorgehen ist aber nicht mailspezifisch.

## Warum IT-Fehlersuche systematisch anfällig ist

Der Mailfluss ist eine Kette aus Systemen, die jeweils eine eigene Sicht auf dieselbe Nachricht haben: das Gateway, die Filterebene, der lokale Transportserver, der Cloud-Dienst, das Zielpostfach. Jede Meldung ist aus der Perspektive genau einer Schicht geschrieben.

Dazu kommt: Fehlertexte sind Sammelbegriffe. Derselbe Wortlaut beschreibt oft völlig unterschiedliche Situationen, weil das ablehnende System nur ein grobes Raster kennt. Die erweiterten Statuscodes sind genau dafür gemacht, Klassen zu bilden, nicht Einzelfälle zu benennen.

Ein Beispiel: Ein Cloud-Dienst lehnte eine Nachricht mit dem Hinweis ab, der Absender sei für ausgehende Zustellung nicht zulässig. Derselbe Wortlaut trat in derselben Umgebung in zwei grundverschiedenen Konstellationen auf. Einmal versuchte ein System, über den Dienst an einen fremden Empfänger zuzustellen, also ein echter Weiterleitungsversuch nach aussen. Das andere Mal war der Empfänger ein reguläres Postfach des Dienstes, und beanstandet wurde ausschliesslich die Absenderdomäne.

Wer den Text wörtlich nimmt, sucht in beiden Fällen dasselbe. Und weil das Wort „ausgehend" darin vorkommt, sucht man zuerst am falschen Ende.

## Grundsatz 1: Eine Hypothese muss etwas verbieten

Karl Popper hat die Wissenschaftstheorie um eine Einsicht bereichert, die für die Fehlersuche unmittelbar praktisch ist: **Eine Aussage ist nur dann brauchbar, wenn sie widerlegbar ist.** Eine Erklärung, die zu jedem denkbaren Beobachtungsergebnis passt, erklärt nichts.

Übertragen heisst das: Formulieren Sie Ihre Vermutung so, dass sie eine **Vorhersage** enthält, die falsch sein kann. Nicht „irgendetwas mit der Absenderdomäne stimmt nicht", sondern „wenn ich dieselbe Nachricht mit einer anderen Absenderdomäne über denselben Weg schicke, kommt sie an".

Die zweite Formulierung ist etwas wert, weil sie sich in fünf Minuten widerlegen lässt. Die erste können Sie stundenlang mit Belegen füttern, ohne je schlauer zu werden.

Ein guter Test dafür: Fragen Sie sich vor dem Versuch, welches Ergebnis Ihre Hypothese **widerlegen** würde. Fällt Ihnen keines ein, haben Sie keine Hypothese, sondern eine Stimmung.

## Grundsatz 2: Eine Variable, sonst alles gleich

Der Kern des Experiments ist die Kontrolle der Störvariablen. In der Praxis passiert regelmässig das Gegenteil: Man vergleicht zwei Fälle, die zufällig vorliegen. Und die unterscheiden sich fast immer in mehreren Merkmalen gleichzeitig.

Aus einem realen Fall: Nachrichten von `example-test.com` wurden abgelehnt, Nachrichten von `partner.example` kamen an. Die beiden Domänen unterschieden sich in mindestens vier Merkmalen: Zugehörigkeit zur Organisation, wo die Mail gehostet ist, ob eine strenge Authentifizierungsrichtlinie hinterlegt ist, und der Weg der Einlieferung. Aus zwei Datenpunkten mit vier Unterschieden lässt sich exakt nichts folgern. Jede der vier Erklärungen passt.

Bauen Sie deshalb den Vergleich selbst. Gleicher Einlieferungspunkt, gleicher Empfänger, gleicher Weg, gleiche Zeit, und **genau ein** geändertes Merkmal. Wenn Sie die Absenderdomäne verdächtigen, ändern Sie nur diese.

## Grundsatz 3: Ohne Kontrollversuch ist das Ergebnis wertlos

Das ist der Teil, den man am liebsten weglässt, und der wichtigste. In der klinischen Forschung ist die Kontrollgruppe selbstverständlich; in der IT verzichtet man meist darauf und wundert sich über widersprüchliche Resultate.

**Ihr Testaufbau muss zuerst den Fehler reproduzieren.** Wenn Sie den Fehlerfall nicht mit Ihren eigenen Mitteln erzeugen können, sagt ein erfolgreicher Gegenversuch nichts aus. Vielleicht funktioniert Ihre Testnachricht nur deshalb, weil Sie an anderer Stelle einliefern als das Originalsystem, oder weil eine Prüfung auf Ihrem Weg gar nicht greift.

Ein brauchbarer Test besteht deshalb aus mindestens zwei Nachrichten:

| | Zweck | Erwartung |
|---|---|---|
| Versuch 1 | Kontrolle, repliziert den Originalfall | **muss scheitern** |
| Versuch 2 | Hypothese, eine Variable geändert | soll gelingen |

Scheitert Versuch 1 nicht, ist Ihr Aufbau nicht repräsentativ. Dann haben Sie nichts über den Originalfall gelernt, sondern nur über Ihren Testaufbau, und müssen näher am Original einliefern.

## Ein durchgespieltes Beispiel

Zurück zum Fall oben, anonymisiert. Nachrichten eines Systems erreichten Empfänger in der Cloud nicht, andere Nachrichten an dieselben Empfänger kamen problemlos an. Drei Versuche über denselben Weg, an denselben Empfänger, im Abstand weniger Minuten:

| Versuch | Absenderdomäne | Hypothese, die er prüft | Ergebnis |
|---|---|---|---|
| 1 (Kontrolle) | `example-test.com` | Aufbau ist repräsentativ | Ablehnung, identisch zum Original |
| 2 | `example.com`, eigene Domäne des Ziels | es liegt an der Absenderdomäne | zugestellt |
| 3 | `other-test.com`, fremde Domäne derselben Organisation | es liegt an der Organisationszugehörigkeit | zugestellt |

Versuch 1 reproduzierte den Fehler, der Aufbau war also gültig. Versuch 2 zeigte, dass es an der Absenderdomäne hängt und nicht an Empfänger, Postfach, Routing oder Berechtigungen. Versuch 3 war der eigentlich elegante: Er prüfte gezielt die naheliegendste Alternativerklärung und **widerlegte sie**, denn `other-test.com` gehörte derselben Organisation und kam trotzdem durch.

Drei Nachrichten, zehn Minuten, und die Ursache war belegt statt vermutet. Vorher waren mehrere Stunden in Erklärungsversuche geflossen, von denen sich am Ende keiner hielt.

## Grundsatz 4: Widerlegen ist der eigentliche Fortschritt

Eine widerlegte Hypothese fühlt sich nach Rückschritt an. Tatsächlich ist sie das Einzige, was Sie sicher wissen. Bestätigungen sind schwach, denn eine Beobachtung kann zu mehreren Erklärungen passen. Eine saubere Widerlegung entfernt einen ganzen Ast aus dem Suchraum, und zwar dauerhaft.

Genau hier wirkt der Bestätigungsfehler am stärksten. Haben Sie eine Vermutung, finden Sie fast immer etwas, das dazu passt. In der oben beschriebenen Analyse gab es eine Korrelation zwischen der Ablehnung und der Frage, wo die Absenderdomäne ihre Mail hosten lässt. Sie sah überzeugend aus, beruhte aber auf zwei Datenpunkten, die sich in mehreren Merkmalen unterschieden. Der dritte Versuch hat sie entkräftet.

Notieren Sie deshalb die widerlegten Erklärungen zusammen mit dem Grund, aus dem sie verworfen wurden. Das ist nichts anderes als ein Laborbuch. Es hat zwei Wirkungen: Wer den Fall später übernimmt, läuft nicht dieselben Sackgassen ab. Und Sie selbst merken, wenn Sie im Kreis denken, weil eine schon verworfene Idee unter neuem Namen zurückkehrt.

In der Dokumentation gehören die widerlegten Punkte ausdrücklich neben die belegten. Ein Bericht, der nur die richtige Antwort enthält, verschweigt die Hälfte der Arbeit und lädt dazu ein, sie zu wiederholen.

## Grundsatz 5: Kennen Sie Ihre Stichprobe

Die subtilste Fehlerquelle ist die Stichprobenverzerrung, und sie trifft in der IT vor allem Abfragen, die seitenweise liefern.

Sie fragen sieben Tage Nachrichtenverfolgung ab, filtern lokal nach einem Merkmal und bekommen kein Ergebnis. Der Schluss liegt nahe, dass es diesen Verkehr nicht gab. Tatsächlich haben Sie nur die erste Seite gefiltert, und die deckt bei hohem Aufkommen wenige Minuten ab.

Das korrekte Ergebnis lautet: nicht im Ausschnitt gefunden. Es lautet nicht: existiert nicht. Der Unterschied ist derselbe wie zwischen „in unserer Studie kein Effekt nachweisbar" und „es gibt keinen Effekt".

Zwei Auswege funktionieren. Verkleinern Sie das Zeitfenster so weit, dass eine Seite es vollständig abdeckt, erkennbar am Ausbleiben des Hinweises auf weitere Ergebnisse. Oder blättern Sie durch alle Seiten und werten dann aus.

Und ein dritter, der oft übersehen wird: Für die Frage, ob etwas **nie** vorkommt, ist eine Konfigurationsprüfung jeder Beobachtung überlegen. Wenn ein System keine Route auf ein Ziel besitzt, kann es dorthin nicht zustellen, unabhängig von jedem Beobachtungsfenster. Das ist der Unterschied zwischen einem empirischen und einem strukturellen Argument, und wo Sie das strukturelle haben können, nehmen Sie es.

## Der Übertrag: Beweislast an die Umkehrbarkeit koppeln

Hier endet die Analogie zur Wissenschaft, und die Ingenieurperspektive übernimmt. Forschung will Wahrheit, Betrieb will eine funktionierende Anlage. Daraus folgt ein Massstab, den die Wissenschaft nicht kennt: **Der Aufwand für den Nachweis richtet sich nach der Umkehrbarkeit des Eingriffs.**

Einen Connector zu deaktivieren ist ein Befehl, und das Zurücknehmen ebenfalls. Dafür genügen begründete Indizien, denn ein Irrtum ist in einer Minute behoben und fällt sofort auf. Denselben Connector zu löschen ist nicht umkehrbar; dafür lohnt sich der zusätzliche Nachweis über die Konfiguration der Gegenstelle oder einen serverseitigen Nutzungsbericht.

Dasselbe gilt für Regelwerksänderungen. Eine reine Beobachtungsstufe, die protokolliert und nichts umleitet, dürfen Sie mit dünner Faktenlage einführen. Sie ist folgenlos und beschafft genau die Daten, die für den scharfen Schritt fehlen. Erst die Umstellung, die Nachrichten zurückhalten kann, verlangt belastbare Belege.

Wer diesen Massstab nicht anlegt, macht regelmässig beide Fehler gleichzeitig: Er fordert wochenlange Beweise für eine Änderung, die man in Sekunden zurücknehmen könnte, und schaltet ohne Absicherung etwas scharf, das den Mailverkehr anhalten kann.

## Wann Sie aufhören dürfen

Es gibt einen Punkt, an dem weiteres Graben keinen Wert mehr schafft: wenn die Behebung feststeht, aber der Mechanismus unklar bleibt.

Im Beispiel oben war nach drei Versuchen belegt, dass die Absenderdomäne der Auslöser ist, dass alles andere im Mailweg funktioniert und dass kein breiteres Problem vorliegt. Warum der Cloud-Dienst intern genau so entscheidet, blieb offen. Für die Korrektur war das ohne Bedeutung, denn die lag bei der sendenden Anwendung.

Trennen Sie deshalb bewusst zwei Fragen. Was muss ich ändern, damit es funktioniert? Und warum verhält sich das System so? Die erste müssen Sie beantworten, die zweite dürfen Sie an den Hersteller geben. Ein Support-Fall mit drei kontrollierten Versuchen, Zeitstempeln, Nachrichtenkennungen und einem funktionierenden Gegenbeispiel ist ohnehin um Längen wertvoller als eine Beschreibung des Symptoms.

Das ist übrigens auch der Punkt, an dem sich Wissenschaft und Betrieb sauber trennen lassen. Die Wissenschaft darf die Frage nach dem Mechanismus nicht aufgeben. Der Betrieb muss sie priorisieren.

## Die Kurzfassung

Formulieren Sie Hypothesen so, dass sie scheitern können, und fragen Sie sich vorher, welches Ergebnis sie widerlegen würde. Vergleichen Sie nie zwei zufällig vorliegende Fälle, sondern bauen Sie den Vergleich mit genau einer geänderten Variable. Reproduzieren Sie den Fehler im Kontrollversuch, bevor Sie dem Gegenversuch glauben. Behandeln Sie Widerlegungen als Fortschritt und halten Sie sie schriftlich fest. Prüfen Sie bei jeder Abfrage, ob Sie die Gesamtheit oder eine Stichprobe sehen. Und richten Sie die geforderte Beweistiefe danach aus, wie leicht sich der geplante Eingriff zurücknehmen lässt.

Die konkreten Abfragen dazu stehen in [Exchange-Mailfluss analysieren: Message Tracking, SMTP-Protokolle und Receive-Connectoren](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). Wer die Befehle lieber zusammenklickt als abtippt, findet sie im [Befehls-Generator](https://rafaelpfister.ch/tools/command-builder).

## Quellen

1.  [Karl Popper: Logik der Forschung](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Ursprung des Falsifikationsprinzips, wonach eine Aussage nur dann wissenschaftlich ist, wenn sie widerlegbar bleibt.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): erklärt, warum erweiterte Statuscodes bewusst grobe Klassen sind und denselben Code für verschiedene Ursachen zulassen.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Ereignistypen und Felder, Grundlage für die Bestimmung des letzten Verarbeitungsschritts.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): Seitenlogik der Nachrichtenverfolgung, die Stichprobenfehler begünstigt.
