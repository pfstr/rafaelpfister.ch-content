---
title: "Home Assistant 2026.11.0: Marketplace statt HACS und neue Entitätsnamen"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 übernimmt HACS als integrierten Marketplace, setzt Entitätsnamen neu aus Gerät und Entität zusammen und bringt zahlreiche Backward-incompatible changes. Prüfen Sie vor dem Update Dashboards, Templates und Anwesenheitserkennung über die FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant und IoT"
timeToRead: "6 Min. Lesezeit"
themen:
  - "smart-home-iot"
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0"
translationId: article-6fe587829541b0ff
url: https://rafaelpfister.ch/blog/home-assistant-2026-11-0
---

# Home Assistant 2026.11.0: Marketplace statt HACS und neue Entitätsnamen

Home Assistant 2026.11.0 ist am 4. November 2026 erschienen und folgt auf 2026.10.4. Das Release integriert den bisherigen Home Assistant Community Store (HACS) als Marketplace direkt in Home Assistant, ändert die Bildung von Entitätsnamen und enthält eine lange Liste von Backward-incompatible changes, darunter Änderungen an den Device Trackern von FRITZ!Box Tools und an der Anmeldung über nicht lokale Adressen. Laut Release-Blogpost kommen 38 neue Integrationen hinzu. Eine Sicherheitskorrektur mit CVE nennen die Release Notes nicht. Das Release richtet sich an alle Betreiber von Home Assistant, besonders an Installationen mit Custom Integrations aus HACS, wie sie in den Artikeln zur [Midea PortaSplit](/blog/midea-portasplit-home-assistant) und zu [digitalSTROM](/blog/digitalstrom-home-assistant) verwendet werden.

## Die Versionen im Überblick

| Version | Datum | Art | Kernpunkte |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Monatsrelease | HACS wird zum Marketplace, neue Entitätsnamen, Device Tracker von Netzwerkscannern immer mit Gerät, Auth-Prüfung bei nicht lokalen Adressen, kein `deps`-Ordner mehr, 38 neue Integrationen |

## Marketplace: HACS wird Teil von Home Assistant

Mit 2026.11 wird HACS laut Release-Blogpost Teil von Home Assistant und heisst nun Marketplace. Administratoren finden ihn unter Einstellungen > Marketplace und installieren dort Integrationen, Dashboard-Karten, Themes und Vorlagen; ein GitHub-Konto ist dafür nicht nötig. Nach der Installation einer Integration entfällt der Neustart, Updates erfordern weiterhin einen Neustart.

Bestehende HACS-Installationen migriert Home Assistant beim ersten Start nach dem Update. Installierte Projekte bleiben installiert und erhalten weiter Updates, Custom Repositories und Update-Entitäten werden übernommen. Laut Hersteller ist dafür kein Eingriff nötig. Der HACS-Installer endet mit Home Assistant 2026.11, eine separate HACS-Installation ist auf dieser Version nicht mehr möglich, und das bisherige HACS erhält keine neuen Releases. Fehler melden Sie künftig im Issue-Tracker von Home Assistant.

Beim ersten Aufruf zeigt der Marketplace einen Hinweis, dass Community-Projekte vom Home-Assistant-Projekt weder getestet noch geprüft oder unterstützt werden. Diesen Hinweis müssen Sie vor dem Installieren oder Aktualisieren bestätigen. Projekte ausserhalb des Katalogs lassen sich über „Add from link" hinzufügen; dafür ist eine GitHub-Verbindung per Device Sign-in nötig, die nur öffentliche Informationen liest.

Für die Midea PortaSplit betrifft das die Custom Integration `Midea AC LAN` und die Karte `apexcharts-card`, sofern beide über HACS installiert sind. Nach den Angaben zur Migration bleiben sie installiert und erhalten weiter Updates. Das Supply-Chain-Risiko von Custom Integrations, das der Artikel zum [Absichern der PortaSplit](/blog/midea-portasplit-home-assistant-absichern) beschreibt, bleibt mit dem Marketplace bestehen, wie der Risikohinweis des Herstellers zeigt. Prüfen Sie nach dem Update unter Einstellungen > Marketplace, ob alle bisher über HACS installierten Projekte und Custom Repositories vorhanden sind.

## Neue Entitätsnamen

Entitätsnamen setzen sich ab 2026.11 aus dem Namen der Entität und ihrem Kontext zusammen, in der Regel dem Gerätenamen. Selbst vergebene Namen werden nach denselben Regeln migriert, der alte Name bleibt als Alias erhalten. Karten, Badges und Zeilen ohne eigenen Namen zeigen nun den Gerätenamen gefolgt vom Entitätsnamen statt des Friendly Name. Entity-IDs, Automationen und Skripte sind laut Release Notes nicht betroffen.

Für Templates empfiehlt der Hersteller die Funktionen `entity_name()`, `device_name()` und `area_name()`; das Attribut `friendly_name` soll später als veraltet markiert werden. Kontrollieren Sie nach dem Update Ihre Dashboards auf geänderte Beschriftungen und passen Sie Templates an, die `friendly_name` auswerten.

## FRITZ!Box Tools und Device Tracker

Netzwerkscanner-Tracker, ausdrücklich genannt sind unter anderem FRITZ!Box Tools, UniFi Network, NETGEAR und Nmap Tracker, haben nun immer ein Gerät. Umbenannte Tracker-Entitäten erhalten den Gerätenamen am Anfang ihres Namens. Neue Tracker dieser Integrationen werden immer deaktiviert angelegt; bisher waren sie aktiv, wenn eine andere Integration die MAC-Adresse des Geräts bereits kannte. Bestehende Tracker behalten ihren Zustand (aktiviert oder deaktiviert).

Für Anwesenheitsautomationen über die FRITZ!Box, wie im [digitalSTROM-Artikel](/blog/digitalstrom-home-assistant) für das automatische Gehen beschrieben, heisst das: Vorhandene Tracker arbeiten weiter. Den Tracker eines neu hinzugekommenen Smartphones müssen Sie dagegen zuerst aktivieren, bevor er in Automationen einen Zustand liefert.

Der Changelog führt für FRITZ!Box Tools zudem mehrere Korrekturen auf, unter anderem für die Erstellung der WLAN-Schalter, das Ausfiltern von FRITZ!Box-Hosts ohne IP-Adresse aus den Device Trackern, eine Neuladeschleife bei Geräten ohne Verkehrsdaten und eine verständlichere Fehlermeldung bei ungültiger Anmeldung im Einrichtungsdialog. Die Integration FRITZ!Box Call Monitor meldet bei fehlenden Berechtigungen nun einen Konfigurationsfehler.

## Midea und digitalSTROM

Die Verbesserungen an Midea im Release-Blogpost betreffen die offizielle Integration `Midea`, die seit Home Assistant 2026.8 Bestandteil von Core ist. Die Custom Integration `Midea AC LAN` (Domain `midea_ac_lan`) ist davon getrennt. Neu sind Steuerungen und ein Fortschrittssensor für Waschmaschinen und Trockner, Türsensoren für Kühlschränke, Wassererwärmer-Entitäten, ein Ein/Aus-Schalter für Luftreiniger und ein Schalter für den Signalton bei Luftentfeuchtern. Für Klimageräte nennt der Changelog eine Schaltfläche `reset_filter`, mit der sich der Filter als gereinigt markieren lässt. Die Integration hat die Qualitätsstufe Platinum erreicht. Laut Integrationsseite laufen Steuerung und Statusabfragen lokal, die Cloud wird nur für Token und Key von V3-Geräten benötigt.

Zu digitalSTROM enthält der Changelog von 2026.11.0 keinen Eintrag. Die im [digitalSTROM-Artikel](/blog/digitalstrom-home-assistant) beschriebene Integration ist eine Custom Integration; ist sie über HACS als Custom Repository eingebunden, gilt dafür die beschriebene Übernahme in den Marketplace.

## Weitere Backward-incompatible changes

Die vollständige Liste steht im Release-Blogpost. Für den Betrieb sind vor allem diese Punkte relevant:

- **Auth:** Bei Zugriff über eine nicht lokale IP-Adresse müssen Schema, Host und Port im Browser mit der Adresse übereinstimmen, die Home Assistant sieht, sonst schlägt die Anmeldung mit „Invalid client id" fehl. Die bisherige Prüfung akzeptierte solche Adressen wegen eines Fehlers, sobald sie einen Port enthielten. Tragen Sie in diesem Fall Ihren Reverse Proxy unter den Trusted Proxies der HTTP-Integration ein oder verwenden Sie einen Domainnamen.
- **Python-Abhängigkeiten:** Home Assistant installiert keine Pakete mehr in den Ordner `deps` im Konfigurationsverzeichnis. Betroffen sind nur Core-Installationen ausserhalb einer virtuellen Umgebung; OS-, Supervised- und Container-Installationen nicht. Nach dem Update lässt sich der Ordner `deps` löschen.
- **SMTP:** Ein nicht lesbarer Anhang führt nun zu einem Fehler; bisher wurde die Nachricht ohne Anhang versendet.
- **Telegram:** Die alte `notify`-Plattform entfällt. Verwenden Sie die Notify-Entitäten des Telegram-Bots und die Aktion `notify.send_message`.
- **Z-Wave:** Vorausgesetzt werden zwave-js-server 3.11.0 oder neuer (Schema 51), die Z-Wave JS App ab 1.9.0 bzw. Z-Wave JS UI ab v11.24.2.
- **Uptime Kuma:** Versionen vor 2.0.0 werden nicht mehr unterstützt.
- **Philips Hue:** Der Lichteffekt `"None"` entfällt, stattdessen gilt `"off"`.
- **Matter und Electra Smart:** `climate.turn_on` setzt den zuletzt aktiven Modus fort. Für einen bestimmten Modus verwenden Sie `climate.set_hvac_mode`.
- **Polling-Intervall:** Bei Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato und Risco wird ein eigenes Intervall ignoriert und der Standard verwendet. Control4 fragt immer alle 5 Sekunden ab; für ein anderes Intervall deaktivieren Sie das Polling und rufen `homeassistant.update_entity` aus einer Automation auf.

Die Integration Flipr wurde entfernt, weil das Unternehmen dahinter liquidiert wurde.

## Neue Funktionen

Neben dem Marketplace nennt der Release-Blogpost unter anderem das Ausschliessen einzelner Entitäten, Geräte, Bereiche, Stockwerke oder Labels aus einem Target, Kopieren und Einfügen von Dashboard-Abschnitten, eine Datumskarte und eine Alert-Karte mit visuellem Editor. Das Energie-Dashboard kann aus einem Leistungssensor einen Energiesensor ableiten. Für die Fehlersuche gibt es einen Filter „Triggered by" in der Aktivität, Live-Logs je Integration und das Löschen von Statistiken in einem Zeitraum. Backups und Medien lassen sich auf lokalen Datenträgern speichern. Neu unterstützt werden Matter-1.5-Kameras mit WebRTC sowie LoRaWAN (ChirpStack, The Things Network, SenseCAP) und Zigbee Green Power.

Bei der Alert-Karte heisst die Option `color` nun `alert_color`; anpassen müssen Sie nur Dashboards, die Sie selbst in YAML angelegt haben. Die neue Integration Debug console führt Python-Code innerhalb von Home Assistant aus, hat vollen Zugriff und muss deshalb in `configuration.yaml` aktiviert werden; nutzen können sie nur Administratoren.

## Empfohlenes Vorgehen

Zielversion ist 2026.11.0. Lesen Sie vor dem Update die Backward-incompatible changes im Release-Blogpost für die bei Ihnen eingesetzten Integrationen. Nach dem Update prüfen Sie im Marketplace, ob alle bisherigen HACS-Projekte übernommen wurden, kontrollieren Dashboards und Templates auf die neuen Entitätsnamen und aktivieren bei Bedarf neu angelegte Device Tracker. Bei Zugriff über einen Reverse Proxy testen Sie die Anmeldung. Die Sicherheitshinweise im GitHub-Repository von Home Assistant Core enthalten zum 5. November 2026 keinen Eintrag, der 2026.11.0 zugeordnet ist; Angaben zu Known Issues enthält der Release-Blogpost nicht.

## Quellen

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Release-Blogpost mit Marketplace, Entitätsnamen, neuen Integrationen und den Backward-incompatible changes.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Vollständiger Changelog, darunter die Einträge zu FRITZ!Box Tools, FRITZ!Box Call Monitor und Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Release-Tag mit Veröffentlichung am 4. November 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Dokumentation der offiziellen Midea-Integration, eingeführt in 2026.8, lokale Steuerung.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Veröffentlichte Sicherheitshinweise, Stand 5. November 2026 ohne Eintrag zu 2026.11.0.

6.  [Midea PortaSplit in Home Assistant: Einrichtung und Dashboard](/blog/midea-portasplit-home-assistant): Installation von Midea AC LAN und apexcharts-card über HACS.

7.  [Midea PortaSplit in Home Assistant absichern](/blog/midea-portasplit-home-assistant-absichern): Supply-Chain-Risiko von Custom Integrations.

8.  [digitalSTROM mit Home Assistant verbinden](/blog/digitalstrom-home-assistant): Lokale digitalSTROM-Integration und Anwesenheit über die FRITZ!Box.
