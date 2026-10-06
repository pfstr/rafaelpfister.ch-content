---
title: "digitalSTROM-Mietwohnung mit Home Assistant verbinden: ein Praxisbericht"
navTitle: "digitalSTROM und HA"
description: "Eine lokale Home-Assistant-Integration für den digitalSTROM-Server, dazu Bewegungslicht, Duschmodus, Musik per 4× Tippen und automatisches Gehen per FRITZ!Box. Mit den Problemen, die in der Praxis aufgetreten sind."
date: "2026-10-06"
kategorie: "Home Assistant und IoT"
timeToRead: "13 Min. Lesezeit"
themen:
  - "smart-home-iot"
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - "midea-portasplit-home-assistant-einrichten"
slug: "digitalstrom-home-assistant"
translationId: "article-271967fa6d61231d"
url: "https://rafaelpfister.ch/blog/digitalstrom-home-assistant"
---

Meine Mietwohnung ist mit digitalSTROM ausgestattet: Lichter, Storen und eine Lüftungsklappe hängen an Klemmen im Sicherungskasten, gesteuert über Taster und einen digitalSTROM-Server (dSS20). Dazu kommen Philips Hue, drei Sonos-Lautsprecher und eine FRITZ!Box. Ziel war, alles lokal in Home Assistant zusammenzuführen, ohne Cloud-Konto und ohne das Passwort des dSS in Home Assistant abzulegen. Daraus ist eine kleine Integration entstanden, die ich zusammen mit den Automationen als [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) unter MIT-Lizenz veröffentlicht habe.

Kurzfazit: Der dSS bietet eine brauchbare JSON-API mit Ereignissen. Wer sie schonend nutzt, bekommt Taster, Szenen und die Aktivitäten Gehen und Kommen ohne Verzögerung nach Home Assistant und kann damit Automationen auslösen, die digitalSTROM allein nicht kennt.

## Wie die Integration arbeitet

Der dSS stellt seine API auf Port 8080 per HTTPS bereit, mit einem selbstsignierten Zertifikat. Für Programme sieht digitalSTROM App-Tokens vor: Das Programm fordert einen Token an, der Benutzer gibt ihn im Configurator unter System > Zugriffsberechtigung frei, danach meldet sich das Programm damit an. Das Passwort des dSS bleibt beim Benutzer.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-s` | keine Fortschrittsanzeige |
| `-k` | selbstsigniertes Zertifikat des dSS akzeptieren |
| `requestApplicationToken` | legt einen App-Token an, der im Configurator freigegeben werden muss |
| `loginApplication` | tauscht den freigegebenen App-Token gegen ein Session-Token |
| `event/subscribe` | abonniert ein Ereignis (`callScene`, `stateChange`, `buttonClick` …) unter einer frei gewählten ID |
| `event/get` mit `timeout` | wartet bis zu 30 Sekunden auf neue Ereignisse (Long-Poll) |

</details>

Die Integration hält eine solche Long-Poll-Verbindung dauerhaft offen. Jede Szene, die ein Taster, die App oder der Configurator auslöst, kommt als Ereignis an. Zustände wie „Licht im Raum an" oder die Anwesenheit stehen im Zwischenspeicher des dSS unter `/usr/states` und lassen sich abfragen, ohne die Klemmen zu belasten.

Direkte Abfragen von Ausgangswerten (`device/getOutputValue`) gehen dagegen über den dS485-Bus bis zur Klemme und dauern pro Wert eine halbe bis ganze Sekunde. Viele solche Abfragen in kurzen Abständen können die Meter überlasten. Die Integration liest deshalb nur Storenpositionen und Joker-Ausgänge direkt, alle 15 Minuten und etwa eine Minute nach einer Fahrt.

| Plattform | Inhalt |
|---|---|
| `light` | ein Licht pro Raum, geschaltet über Raumszenen wie die Taster; Helligkeit bei gedimmten Klemmen, nur Ein/Aus bei geschalteten |
| `cover` | Storen mit Auf, Zu, Stopp und Position |
| `scene` | im Configurator benannte Stimmungen |
| `button` | Gehen (Szene 72) und Kommen (Szene 71) |
| `binary_sensor` | Anwesenheit, Windalarm, Bewegungsmelder, Joker-Ausgänge |
| `sensor` | Gesamtverbrauch der Wohnung |

Zusätzlich meldet die Integration jede Raumszene sowie Gehen und Kommen als Ereignis `digitalstrom_local_event` an Home Assistant. Damit lassen sich 2× oder 4× Tippen auf einem gewöhnlichen Lichtschalter frei belegen.

## Automationen aus der Wohnung

Die folgenden Automationen sind im Repository als Blueprints enthalten und lassen sich über den Import-Knopf im README in Home Assistant übernehmen.

### Bewegungslicht, das von Hand eingeschaltetes Licht nicht löscht

Gang, Reduit, Bad und Büro haben Hue-Bewegungsmelder. Bei Bewegung geht das Licht an, nach einer einstellbaren Zeit ohne Bewegung wieder aus: im Gang 2 Minuten und nur unter 20 Lux, nachts gedimmt auf 20 %; im Reduit 30 Sekunden. Schaltet jemand das Licht am Taster ein, bleibt es an. Ein Hilfsschalter (`input_boolean`) merkt sich dafür, ob die Automation das Licht eingeschaltet hat; nur dann schaltet sie es auch wieder aus.

Ein Detail ist mir erst im Betrieb aufgefallen: Der Auslöser „Melder seit 2 Minuten ruhig" ist ein Zähler, den Home Assistant bei jedem Neustart verwirft. Hat sich der Melder vor dem Neustart zuletzt bewegt, folgt kein neuer Wechsel auf „ruhig", und das Licht bleibt an. Der Blueprint prüft deshalb zusätzlich jede Minute, ob ein von ihm eingeschaltetes Licht brennt, obwohl der Melder lange genug ruhig ist.

### Duschmodus per 2× Tippen

Hinter dem Duschvorhang sieht der Bewegungsmelder niemanden, und nach 5 Minuten wird es dunkel. 2× Tippen auf den Badtaster löst bei digitalSTROM die Stimmung 2 aus (Szene 17). Die Automation erkennt diese Szene im Raum Bad und schaltet einen Duschmodus ein, der das Ausschalten blockiert. Bewegungen in den ersten 2 Minuten werden ignoriert (Ausziehen, Einsteigen). Die erste Bewegung danach beendet den Duschmodus, ab dann gilt wieder die normale Ausschaltzeit. Nach spätestens 60 Minuten endet er von selbst.

### 4× Tippen startet Sonos

Tippt man mehrmals schnell auf einen Taster, schaltet digitalSTROM die Stimmungen der Reihe nach durch: Szene 5, 17, 18 und beim vierten Tippen 19. Im Configurator habe ich Szene 19 bei allen Lampen auf „Ausgang nicht verändern" gestellt; ein Blueprint reagiert auf diese Szene, schaltet das Raumlicht aus und startet den Sonos-Lautsprecher im Raum. In Räumen ohne Lautsprecher spielen alle drei zusammen.

Das Skript dazu achtet auf drei Punkte: Läuft schon ein Lautsprecher, wird der neue Raum dessen Gruppe zugeschaltet, damit die Wiedergabe synchron bleibt. Lässt sich nichts fortsetzen, spielt ein Radiosender aus dem Radio-Browser-Verzeichnis. Und vor jedem Start wird dieselbe Startlautstärke gesetzt, sonst spielt der eine Raum leise und der andere so laut, wie zuletzt jemand Musik gehört hat.

### Gehen und Kommen per FRITZ!Box

Die Integration FRITZ!Box Tools meldet, ob das Handy im WLAN ist; sie wertet die Geräteliste der Box aus und deckt damit 2,4-GHz-Netz, 5-GHz-Netz und LAN gleichzeitig ab. Ist niemand mehr zu Hause und wurde Gehen nicht gedrückt, löst ein Blueprint Gehen aus. Beim Heimkommen folgt Kommen und, wenn es dunkel ist, das Licht im Gang.

Ergänzend schaltet eine Automation bei Gehen alle Hue-Lampen aus und pausiert Sonos. Die digitalSTROM-Lampen schaltet der dSS selbst aus.

## Grundriss als Dashboard

Die Übersicht in Home Assistant ist ein Grundriss mit der eingebauten Karte `picture-elements`. Pro Raum liegt ein transparentes SVG über dem Plan, das bei eingeschaltetem Licht durch ein leicht gelbes ersetzt wird; ein Tippen auf den Raum schaltet das Licht. Lampen, Lautsprecher, Storen und Melder sitzen als Symbole dort, wo sie in der Wohnung stehen. Die Anleitung mit Beispielkonfiguration liegt im Repository unter `docs/floor-plan-dashboard.md`.

## Probleme aus der Praxis

| Problem | Ursache | Lösung |
|---|---|---|
| Gehen-Taster ohne Wirkung | dSS nicht im Netz, Aktivitäten über mehrere Stromkreise laufen über den Server | Netzwerkverbindung des dSS wiederhergestellt; seither funktioniert Gehen |
| Storen reagieren nicht | Windalarm stand seit langem auf „aktiv", obwohl kein Windsensor vorhanden ist | Szene 87 („kein Wind") wohnungsweit ausgelöst |
| Storen fahren bei Gehen nicht hoch | Szene 72 auf „Ausgang nicht verändern" (`dontCare`) | `device/setSceneMode` mit `dontCare=0`; der Wert `false` wurde angenommen, aber ignoriert |
| Bad lässt sich nicht dimmen | Leuchtstoffröhre an einer geschalteten Klemme (Ausgangsmodus 35) | Integration erkennt geschaltete Klemmen und bietet dort nur Ein/Aus an |
| Licht bleibt nach Neustart an | Zähler für „seit X Minuten ruhig" geht beim Neustart verloren | zusätzliche Prüfung jede Minute |
| Handy gilt als abwesend | iPhone nutzt eine wechselnde private WLAN-Adresse und trennt im Ruhezustand kurz das WLAN | Private WLAN-Adresse auf „Fest", Karenzzeit 10 Minuten |
| DECT funkt dauernd | „DECT Eco" ist nicht verfügbar, sobald ein FRITZ!-Smart-Home-Gerät angemeldet ist | Keller-WLAN wird über Powerline statt über eine DECT-Steckdose geschaltet (geplant) |

Eine Gerätesuche an der Hue-Bridge nimmt alle Geräte auf, die sich gerade im Kopplungsmodus befinden. Prüfen Sie danach in der Geräteliste, ob nur die eigenen Lampen dazugekommen sind.

Die Szenen-Tabellen der Klemmen lassen sich über die API auslesen und ändern, etwa mit `device/getSceneMode` und `device/saveScene`. Jeder dieser Aufrufe geht über den Bus. Vor Änderungen habe ich die alten Werte jeweils in eine CSV-Datei gesichert, um sie bei Bedarf wiederherstellen zu können.

## Quellen

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): Integration und Blueprints aus diesem Artikel, MIT-Lizenz.

2.  [digitalSTROM: Handbuch Bedienen und Einstellen](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): Aktivität Gehen (3 Sekunden halten), Einstellung „Ausgang nicht verändern" in Kapitel 3.5.2.

3.  [digitalSTROM: Tastendokumentation der Klemmen](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): Werkseinstellungen der Tasterfunktionen pro Klemmentyp.

4.  [digitalSTROM: Bedienungsanleitungen](https://www.digitalstrom.com/bedienungsanleitungen/): Übersicht der Handbücher, Planer- und Installationsunterlagen.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): Anwesenheitserkennung, WLAN-Schalter, Optionen der Integration.

6.  [Home Assistant: Picture Elements card](https://www.home-assistant.io/dashboards/picture-elements/): Grundlage des Grundriss-Dashboards.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): Import und Verwendung von Blueprints.

8.  [FRITZ! Wissensdatenbank: FRITZ!-Steckdose verliert Verbindung](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): Hinweise zu DECT-Reichweite und -Funkleistung bei Smart-Home-Steckdosen.

9.  [Radio Browser](https://www.radio-browser.info/): freies Verzeichnis von Radiostreams, in Home Assistant als Medienquelle eingebunden.
