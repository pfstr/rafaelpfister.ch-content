---
title: "F5 BIG-IP als Outbound-Proxy für den Mail-Massenversand: Persistenz, SNAT, Timeouts und DNS-Auflösung"
navTitle: "F5 Massenversand"
description: "Ein Massenversand mit 1000 Mails pro Minute läuft über eine BIG-IP als ausgehenden Proxy zum Provider-Relay. Der Artikel klärt, warum Sticky Sessions hier nichts bringen, wie der Provider-Hostname sauber per FQDN-Node aufgelöst wird und welche Einstellungen bei SNAT, Timeouts und Verbindungs-Limits den Durchsatz tatsächlich bestimmen."
date: "2026-08-26"
kategorie: "Loadbalancer"
timeToRead: "9 Min. Lesezeit"
themen:
  - "loadbalancer"
  - "smtp-mailflow"
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - "massenmailing-provider-wechsel-checkliste"
  - "mailserver-lastprofil-ermitteln"
slug: "f5-big-ip-outbound-smtp-massenversand"
translationId: "article-ee5e63e82ffd2604"
url: "https://rafaelpfister.ch/blog/f5-big-ip-outbound-smtp-massenversand"
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
---

Ein Rechnungslauf oder Newsletter-Versand mit rund 1000 Mails pro Minute verlässt das Unternehmensnetz, dazwischen steht eine F5 BIG-IP als ausgehender Proxy. Der Provider nimmt die Mails auf einem Relay-Verbund aus mehreren Systemen entgegen, jedes davon mit einem Limit für gleichzeitige Verbindungen. Nach aussen sichtbar ist davon nichts: Der Provider liefert einen einzigen Hostnamen, dahinter steht eine einzelne IP-Adresse, und die Verteilung auf seine Relays macht er selbst auf seiner Seite. Die BIG-IP verteilt also nicht, sie reicht durch. Genau diese Konstellation entscheidet, welche Einstellungen sinnvoll sind und welche vermeintlichen Optimierungen ins Leere laufen.

## Die Architektur in einem Satz

Die Versandsysteme sprechen als Smarthost eine interne Virtual-Server-Adresse auf der BIG-IP an, die BIG-IP übersetzt die Absenderadressen per SNAT auf eine feste öffentliche IP und leitet jede Verbindung an den Provider-Hostnamen weiter. Load Balancing im eigentlichen Sinn findet auf der BIG-IP nicht statt, weil der Pool nur ein Mitglied hat. Das klingt nach einer trivialen Konfiguration, aber die Detailentscheide (Persistenz, Timeouts, SNAT-Typ, DNS-Auflösung) bestimmen, ob der Versand stabil läuft oder unter Last unerklärliche Abbrüche zeigt.

## Sind Sticky Sessions besser? Nein, und zwar aus zwei Gründen

Die Frage nach Session-Persistenz stammt aus der HTTP-Welt, wo ein Benutzer mit Warenkorb oder Login-Session immer auf demselben Backend landen muss. Auf SMTP übertragen ergibt das Konzept keinen Sinn.

Erstens ist SMTP pro Verbindung zustandslos abgeschlossen: Jede Verbindung wickelt eine oder mehrere vollständige Transaktionen ab (MAIL FROM, RCPT TO, DATA) und endet mit QUIT. Es gibt keinen Zustand, der über Verbindungen hinweg auf demselben Zielsystem liegen müsste. Welches der Provider-Relays die nächste Verbindung annimmt, ist für die Zustellung belanglos.

Zweitens gibt es auf dieser BIG-IP schlicht nichts zu persistieren: Der Pool enthält genau ein Mitglied, die eine IP-Adresse des Providers. Ein Persistenzprofil würde nur Speicher für eine Persistenztabelle verbrauchen und bei jeder Verbindung einen Lookup kosten, der immer dasselbe Ergebnis liefert. Die richtige Einstellung ist darum: Default Persistence Profile auf None. Selbst wenn der Provider später mehrere IP-Adressen hinter dem Hostnamen publizieren sollte, wäre Persistenz kontraproduktiv, weil sie die Verteilung auf diese Adressen verhindern und einzelne Relays einseitig belasten würde.

Entscheidend für den Durchsatz beim Massenversand ist das Verbindungsprofil des Senders: wenige langlebige Verbindungen mit vielen Nachrichten pro Verbindung statt einer neuen Verbindung pro Mail; dazu unten mehr.

## Virtual Server: FastL4 statt Full Proxy

Für reines Durchreichen von SMTP ist ein Performance-(Layer-4)-Virtual-Server mit FastL4-Profil die richtige Wahl. Die BIG-IP verarbeitet die Verbindung dann weitgehend in Hardware beziehungsweise im beschleunigten Pfad, ohne die TCP-Verbindung vollständig zu terminieren. Ein Standard-Virtual-Server im Full-Proxy-Modus bringt nur dann einen Mehrwert, wenn Sie auf der BIG-IP wirklich in den Datenstrom eingreifen wollen, etwa mit einem SMTP-Sicherheitsprofil oder iRules auf Protokollebene. Für einen Outbound-Proxy zum eigenen Vertragsprovider ist das unnötig und schafft nur zusätzliche Fehlerquellen.

Wichtig in beiden Fällen: kein Profil aktivieren, das in die Verbindung hineinschreibt. STARTTLS handeln die Versandsysteme direkt mit dem Provider-Relay aus; jede Instanz, die Bytes verändert oder filtert, gefährdet den TLS-Aufbau.

## DNS-Auflösung: der Provider-Hostname gehört als FQDN-Node in den Pool

Der Provider hat einen Hostnamen geliefert, keine IP-Adresse. Der naheliegende Reflex, die IP einmal aufzulösen und statisch als Node einzutragen, ist die schlechteste Variante: Wechselt der Provider die Adresse (Wartung, Umzug, DR-Fall), steht der Versand, bis jemand die BIG-IP-Konfiguration anpasst. Genau dafür gibt es FQDN-Nodes.

Ein FQDN-Node hinterlegt den Hostnamen statt der Adresse. Die BIG-IP löst den Namen selbst auf, legt für jede zurückgelieferte Adresse ein sogenanntes Ephemeral-Node an und aktualisiert diese automatisch, wenn sich die DNS-Antwort ändert. Standardmässig fragt sie den Namen nach Ablauf der DNS-TTL erneut ab; alternativ lässt sich ein festes Abfrageintervall setzen. Mit aktiviertem Autopopulate übernimmt der Pool auch mehrere A-Records automatisch als Mitglieder: Sollte der Provider seine Einlieferung später auf mehrere Adressen erweitern, folgt die BIG-IP ohne Konfigurationsänderung.

Zwei Voraussetzungen werden dabei gerne vergessen. Erstens braucht die BIG-IP dafür funktionierende DNS-Server in der Systemkonfiguration (System, Configuration, Device, DNS); FQDN-Nodes nutzen die System-Resolver, nicht etwa einen DNS-Cache aus einem Listener-Profil. Zweitens sollten diese Resolver aus dem Management- beziehungsweise TMM-Kontext tatsächlich erreichbar sein, sonst bleibt der Node im Status unresolved und der Pool leer.

Die Konfiguration in tmsh sieht so aus (Adressen und Namen sind Beispiele):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

Die Versandsysteme tragen anschliessend 10.0.5.10 als Smarthost ein. Ob Sie Port 25 oder 587 verwenden, gibt der Provider vor; die BIG-IP-Konfiguration ist in beiden Fällen identisch, nur der Port ändert.

## SNAT: feste Adresse statt Automap

Für ausgehenden Mailverkehr gehört die Quelladresse unter Kontrolle. SNAT Automap nimmt die Floating-Self-IP des ausgehenden VLANs, und die kann sich bei Netzwerkänderungen oder Failover-Umbauten unbemerkt ändern. Provider koppeln die Einlieferung aber häufig an ein IP-Allowlisting, und selbst ohne formales Allowlisting hängt Reputation an der Quelladresse. Ein dedizierter SNAT-Pool mit einer fest zugewiesenen Adresse macht die Quell-IP zu einem dokumentierten, stabilen Konfigurationsobjekt.

Zur Kapazität: Eine einzelne SNAT-Adresse bietet gegen ein einzelnes Ziel (eine IP, ein Port) rund 64'000 gleichzeitige Übersetzungen, weil jede Verbindung einen eigenen ephemeren Quellport erhält. Bei dem hier beschriebenen Lastprofil mit wenigen Dutzend gleichzeitigen Verbindungen ist das um Grössenordnungen ausreichend. Port-Erschöpfung wird erst zum Thema, wenn ein fehlkonfigurierter Sender pro Mail eine neue Verbindung öffnet und diese nicht sauber schliesst; dann sammeln sich Übersetzungen im TIME-WAIT-ähnlichen Zustand. Ein solches Verhalten beheben Sie am Sender, nicht mit einer zweiten SNAT-Adresse.

## Timeouts: die häufigste Ursache für Verbindungsabbrüche unter Last

Ein Bulk-Sender hält Verbindungen offen und schiebt Nachricht um Nachricht hindurch. Zwischen zwei Nachrichten können Pausen entstehen: Der Sender generiert den nächsten Block, das Relay verzögert die Annahme (Tarpitting, Greylisting-Reste, interne Queues). Das Idle-Timeout des FastL4-Profils steht standardmässig auf 300 Sekunden. Liegt eine Pause darüber, räumt die BIG-IP die Verbindung ab, und der Sender schreibt in eine Verbindung, die es nicht mehr gibt.

Zwei Einstellungen entschärfen das. Erstens das Idle-Timeout auf einen Wert setzen, der über den realistischen Pausen liegt; für Massenversand sind 600 Sekunden ein vernünftiger Startwert. Beliebig hoch sollte der Wert nicht sein, sonst sammeln sich verwaiste Verbindungen in der Verbindungstabelle. Zweitens im Profil Reset on Timeout aktiviert lassen: Die BIG-IP quittiert das Abräumen dann mit einem TCP-Reset, und der sendende MTA erkennt sofort, dass die Verbindung weg ist, statt in einen Timeout zu laufen und die Nachricht erst nach Minuten neu einzuplanen.

Auf die Timeouts der Gegenseite haben Sie keinen Einfluss, aber sie gehören ins Bild: Wenn das Provider-Relay Verbindungen nach 120 Sekunden Inaktivität schliesst, nützt ein grosszügiges BIG-IP-Timeout nichts. Massgebend ist der kleinste Timeout-Wert auf dem gesamten Pfad; im Zweifel beim Provider nachfragen und diesen Wert als Planungsgrundlage nehmen.

## Verbindungs-Strategie: wenige Verbindungen, viele Nachrichten

Ohne Einliefervorgaben des Providers lohnt sich eine kurze Rechnung. 1000 Mails pro Minute sind rund 17 pro Sekunde. Eine SMTP-Transaktion über eine bereits stehende Verbindung dauert bei normaler Latenz deutlich unter einer halben Sekunde. Mit 10 bis 20 parallelen Verbindungen und beispielsweise 100 Nachrichten pro Verbindung, bevor der Sender sie erneuert, ist die Zielrate bequem erreicht. Der Relay-Verbund des Providers bietet nominell ein Vielfaches davon an Verbindungs-Slots, aber diese Kapazität teilt sich mit allen anderen Kunden. Wenige langlebige Verbindungen mit vielen Transaktionen sind darum nicht nur effizient (der TCP- und TLS-Aufbau entfällt pro Nachricht), sondern auch die verträglichste Art, fremde Infrastruktur zu nutzen.

Die Stellschrauben dafür liegen im Versandsystem, nicht auf der BIG-IP: maximale Nachrichten pro Verbindung, maximale parallele Verbindungen zum Smarthost, Wiederverwendung stehender Verbindungen. Auf der BIG-IP lässt sich das Ganze mit einem Connection Limit auf dem Pool-Member absichern, etwa 200 gleichzeitige Verbindungen: Im Normalbetrieb wird der Wert nie erreicht, aber ein fehlkonfigurierter Sender, der plötzlich pro Mail eine Verbindung öffnet, flutet damit nicht ungebremst das Provider-Relay. Das Limit ist ein Sicherheitsnetz, kein Steuerungsinstrument.

Ob das eingestellte Verbindungsprofil in der Praxis auch ankommt, zeigt die Messung: Verbindungen pro Minute und Nachrichten pro Verbindung lassen sich aus dem Message Tracking beziehungsweise den Connector-Logs auswerten, wie im Artikel [Das Lastprofil eines Mailservers ermitteln](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) beschrieben. Für einen Lasttest mit realistischem Bulk-Lastbild (wenige Sessions, viele Nachrichten pro Session) eignet sich smtp-source aus dem Postfix-Paket besser als HTTP-orientierte Lastwerkzeuge, weil es genau dieses Verbindungsprofil erzeugt.

## Monitoring: den Provider nicht mit Healthchecks belasten

Ein Monitor auf dem Pool-Member ist sinnvoll, damit die BIG-IP einen Ausfall der Provider-Seite erkennt und sauber meldet. Dabei gilt: Jeder Healthcheck ist eine echte Verbindung zum Provider und zählt dort gegen dieselben Limits wie der Nutzverkehr. Ein einfacher TCP-Monitor mit moderatem Intervall (30 Sekunden oder mehr) genügt vollauf. Ein vollständiger SMTP-Monitor, der bis zum Banner oder EHLO prüft, liefert kaum zusätzlichen Erkenntnisgewinn, erzeugt aber auf der Providerseite Logeinträge und im schlechtesten Fall Rückfragen, warum alle 5 Sekunden eine Verbindung ohne Mail eingeht.

## Checkliste

| Einstellung | Empfehlung |
|---|---|
| Persistenzprofil | None; Sticky Sessions bringen bei SMTP nichts und bei einem Ein-Member-Pool erst recht nicht |
| Virtual-Server-Typ | Performance (Layer 4) mit FastL4-Profil, kein Eingriff in den Datenstrom |
| Ziel-Node | FQDN-Node mit Autopopulate statt statischer IP; DNS-Server auf der BIG-IP konfiguriert |
| SNAT | dedizierter SNAT-Pool mit fester, beim Provider bekannter Adresse; kein Automap |
| Idle-Timeout | über den realen Sendepausen, Startwert 600 s; Reset on Timeout aktiv |
| Connection Limit | als Sicherheitsnetz auf dem Pool-Member, z. B. 200 |
| Monitor | TCP, Intervall 30 s oder mehr; kein aggressiver SMTP-Monitor |
| Sender-Konfiguration | wenige parallele Verbindungen, viele Nachrichten pro Verbindung; Wiederverwendung aktiv |

Die kurze Antwort auf die Ausgangsfrage lautet also: Nein, Sticky Sessions sind nicht besser, sie sind in dieser Konstellation wirkungslos bis schädlich. Die Qualität der Lösung entscheidet sich an der DNS-Auflösung des Provider-Hostnamens, an einer stabilen SNAT-Adresse, an zum Lastprofil passenden Timeouts und daran, dass die Versandsysteme ihre 1000 Mails pro Minute über wenige stehende Verbindungen einliefern statt über tausend einzelne.

## Quellen

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Abschnitt 4.5.4 und das Transaktionsmodell zeigen, dass mehrere Mail-Transaktionen über eine Verbindung der vorgesehene Normalfall sind.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5-Grundlagenartikel zu SNAT, SNAT-Pools und der Portübersetzung pro Ziel.

3.  [tmsh-Referenz: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): dokumentiert die FQDN-Optionen (name, autopopulate, interval) für Nodes und damit für Pool-Member.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): Lastgenerator, der das Bulk-Sender-Verbindungsprofil (wenige Sessions, viele Nachrichten) nachbildet.

5.  [Das Lastprofil eines Mailservers ermitteln](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): eigene Anleitung, wie Verbindungen pro Minute und Nachrichten pro Verbindung aus dem Message Tracking ausgewertet werden.
