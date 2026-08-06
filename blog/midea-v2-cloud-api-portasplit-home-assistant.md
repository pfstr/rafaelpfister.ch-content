---
title: "Midea V2, V3 und Cloud-API: Was für die PortaSplit tatsächlich gemeint ist"
navTitle: "Midea V2 Cloud-API"
description: "Lokales Geräteprotokoll, private App-Endpunkte und offizielle Partner-API verwenden ähnliche Versionsnamen. Die Quellenanalyse trennt diese Ebenen und ordnet die Abschaltungswarnung ein."
date: "2026-07-25"
kategorie: "Home Assistant und IoT"
timeToRead: "11 Min. Lesezeit"
themen:
  - "smart-home-iot"
produkte:
  - "home-assistant"
protokolle:
  - "apis"
related:
  - "midea-portasplit-home-assistant"
  - "midea-portasplit-home-assistant-einrichten"
draft: false
slug: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: "article-f504b2af00493864"
url: "https://rafaelpfister.ch/blog/midea-v2-cloud-api-portasplit-home-assistant"
---

Im Umfeld der Midea PortaSplit bezeichnet «V2» mehrere voneinander unabhängige Dinge. Es gibt ein lokales V2-Geräteprotokoll, Versionsnummern in privaten App-Endpunkten und eine offizielle Cloud-to-Cloud API V2 für Partner. Wer diese Ebenen gleichsetzt, kommt zwangsläufig zu falschen Schlüssen über die lokale Steuerung.

Das Projekt `Midea AC LAN` warnt in seiner [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) davor, dass bisherige Token-Schnittstellen geschlossen und durch eine cloudbasierte V2-API ersetzt würden. Eine Prüfung der Diskussionen, des aktuellen Codes und der offiziellen Midea-Unterlagen ergibt ein differenzierteres Bild:

> Eine offizielle Midea Cloud-to-Cloud API V2 existiert. Sie ist aber nicht identisch mit der von Home Assistant verwendeten Token-Schnittstelle und auch nicht mit dem lokalen V2- oder V3-Geräteprotokoll. Eine offiziell angekündigte Abschaltung der lokalen PortaSplit-Steuerung mit einem konkreten Termin ist nicht dokumentiert. Im Juni 2026 wurde zudem nachgewiesen, dass die vermeintlich abgeschaltete SmartHome-Token-API weiterhin funktionierte – der bisherige Request der Community-Bibliothek war lediglich unvollständig.

Stand dieses Artikels ist der 25. Juli 2026.

## Weshalb die frühere Einordnung korrigiert werden muss

Im [ersten Artikel zur Cloud-Token-Frage](/blog/midea-portasplit-home-assistant) hatte ich die Warnung aus dem Projekt `Midea AC LAN` sinngemäss als angekündigte Abschaltung der Cloud-Schnittstellen wiedergegeben. Das entsprach dem Wortlaut der Projekt-README, war aber als Tatsachenbehauptung zu stark formuliert.

Die Warnung ist als Risikohinweis weiterhin relevant. Sie ist jedoch keine veröffentlichte Midea-Roadmap. Vor allem ist inzwischen neues technisches Material verfügbar, das einen wesentlichen Teil der bisherigen Interpretation infrage stellt.

## Wie die lokale PortaSplit-Steuerung funktioniert

Die Home-Assistant-Integration `Midea Smart AC` beschreibt ihre Architektur ausdrücklich als lokale Steuerung. Bei neueren V3-Geräten wird die Midea-Cloud lediglich während der Einrichtung verwendet, um einen gerätespezifischen Token und Key zu beziehen. Danach speichert die Integration beide Werte lokal und benötigt für die eigentliche Steuerung keine weitere Cloud-Verbindung. Das dokumentiert das Projekt unter [„Note On Cloud Usage"](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Vereinfacht sieht der Ablauf so aus:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

Für manuell konfigurierte V3-Geräte verlangt `Midea Smart AC` Geräte-ID, IP-Adresse, Port, Token und Key. Der dokumentierte Standardport ist `6444/TCP`; Token und Key werden als 128 beziehungsweise 64 Hexadezimalzeichen angegeben. Diese Angaben stehen in der [Dokumentation zur manuellen Konfiguration](https://github.com/mill1000/midea-ac-py#manual-configuration).

Eine PortaSplit wurde im Issue-Tracker von `Midea AC LAN` beispielsweise als Gerätetyp `0xAC`, Modell `00000Q1D` und Protokollversion 3 erkannt. Derselbe Nutzer konnte sie anschliessend über NetHome Plus in Home Assistant aufnehmen. Der konkrete Verlauf ist in [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607) dokumentiert.

Entscheidend ist die Trennung:

- Der Cloud-Dienst wird zur Beschaffung der lokalen Zugangsdaten verwendet.
- Die spätere Steuerung erfolgt direkt im LAN.
- Eine Störung des Token-Dienstes verhindert daher vor allem neue Einrichtungen.
- Sie beendet nicht automatisch eine bereits eingerichtete lokale Verbindung.

Letzteres entspricht auch der ausdrücklichen Beschreibung von [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## Woher die Abschaltungswarnung stammt

Der heute sichtbare Warntext wurde am 19. Mai 2025 mit [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578) in die Dokumentation aufgenommen.

Die Begründung lautet zusammengefasst:

- Die lokalen Tokens hätten kein Ablaufdatum.
- Verschiedene Home-Assistant-Projekte verwendeten nachgebildete oder extrahierte App-Verschlüsselung.
- Daraus ergebe sich ein Sicherheitsproblem.
- Midea werde deshalb die bisherigen Token-Dienste schrittweise schliessen.
- Langfristig solle die lokale V1-Steuerung durch eine cloudbasierte V2-API verdrängt werden.

Im Juli 2025 wurde die Dokumentation über [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639) nochmals angepasst. Anstelle der SmartHome-Cloud wurde nun NetHome Plus als vorübergehend verwendete Token-Quelle genannt. Die eigentliche Abschaltungswarnung blieb bestehen.

Die zugrunde liegende Diskussion ist allerdings vorsichtiger formuliert als die README.

Im [Kommentar des Midea-AC-LAN-Maintainers](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) heisst es sinngemäss, NetHome Plus sei möglicherweise nur eine temporäre Lösung und Midea verfüge nach seinem Verständnis über einen neuen, vollständig cloudbasierten V2-Dienst.

Der Maintainer von `midea-msmart` antwortete darauf, er habe die Existenz einer neuen V2-API ebenfalls vermutet, könne sie mangels eigener Midea-Geräte aber nur eingeschränkt untersuchen. Das steht im [direkten Antwortkommentar](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Damit ist die Quellenlage klarer:

- Die Warnung stammt von erfahrenen Community-Entwicklern.
- Sie basiert auf beobachteten Änderungen und deren technischer Einschätzung.
- Einer der Maintainer bezeichnet die V2-Migration ausdrücklich als sein Verständnis.
- Der andere spricht von einer Vermutung.
- Weder der Pull Request noch die Diskussion verlinken eine offizielle Midea-Abschaltungsankündigung oder einen Termin.

Das macht die Warnung nicht wertlos. Es macht sie aber zu einer Risikoanalyse und nicht zu einer bestätigten Hersteller-Roadmap.

## Der entscheidende neue Befund von Juni 2026

Am 15. Juni 2026 wurde in der Bibliothek `midea-local` ein Fix übernommen, der die bisherige Interpretation wesentlich verändert.

Der Ausgangspunkt war der Fehler:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Dieser Fehler war bei der Abfrage von Token und Key über die SmartHome-Cloud aufgetreten. Login und Geräteliste funktionierten weiterhin, aber der Aufruf von `/v1/iot/secure/getToken` wurde abgewiesen.

Zunächst sah das nach einer abgeschalteten oder unbrauchbar gemachten Schnittstelle aus. Eine Analyse des Requests der offiziellen SmartHome-App zeigte jedoch eine andere Ursache: Die App sendete zusätzlich zum `udpid` das Feld `applianceCodes`. Die Community-Bibliothek hatte dieses Feld nicht mitgeschickt.

Der korrigierte Request enthält nun:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

Der Entwickler testete die Änderung mit einem realen SmartHome-Konto und vier V3-Klimageräten vom Typ `0xAC`:

- Ohne `applianceCodes` antwortete der Server mit Fehler 3004.
- Mit `applianceCodes` lieferte er gültige Token und Keys.
- Die zurückgelieferten Werte funktionierten anschliessend für die lokale V3-Authentifizierung.

Die vollständige Untersuchung, die Testergebnisse und der Code-Diff sind in [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470) dokumentiert. Der zugehörige unveränderliche Commit ist [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

Auch im aktuellen Quellcode wird weiterhin genau dieser Endpunkt verwendet:

```text
/v1/iot/secure/getToken
```

Zusätzlich wird nun `applianceCodes` mitgesendet. Das ist im aktuellen [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) direkt nachvollziehbar.

Die gegenwärtige Version von `Midea AC LAN` bindet `midea-local==6.11.0` ein und deklariert sich weiterhin als `local_push`-Integration. Beides steht im aktuellen [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json).

Die pauschale Aussage, die SmartHome-Token-API sei geschlossen worden, ist damit mindestens für die im Juni 2026 getesteten Konten und Geräte widerlegt. Korrekt wäre:

> Die bisherige Token-Abfrage funktionierte nach einer Änderung des erwarteten Request-Formats nicht mehr. Nach Anpassung an das von der offiziellen App verwendete Format lieferte derselbe V1-Endpunkt wieder gültige lokale Zugangsdaten.

Regionale Unterschiede, abweichende Konten oder nicht unterstützte Gerätetypen sind damit nicht ausgeschlossen. Eine globale Abschaltung war es aber offensichtlich nicht.

## Warum „V2" hier so leicht missverstanden wird

Im Midea-Umfeld werden mindestens drei voneinander unabhängige Versionsbezeichnungen verwendet.

| Begriff | Bedeutung |
| --- | --- |
| Lokales V2-/V3-Protokoll | Generation der direkten Kommunikation zwischen Integration und Gerät |
| V1-/V2-App-Endpunkt | Versionsnummer eines einzelnen HTTP-Endpunkts im Backend der Midea-Apps |
| Cloud-to-Cloud API V2 | Offizielle Partner-API für autorisierte Drittunternehmen |

### Lokales V2 und V3

Beim lokalen Geräteprotokoll bezeichnet V2 beziehungsweise V3 die Kommunikationsgeneration des Geräts. Neuere V3-Geräte benötigen Token und Key für die lokale Authentifizierung. `Midea Smart AC` dokumentiert diese Voraussetzung in seiner [Konfigurationsanleitung](https://github.com/mill1000/midea-ac-py#manual-configuration).

Diese Protokollversion hat nichts mit der offiziellen Cloud-to-Cloud API V2 zu tun.

### V1 und V2 in App-URLs

Auch innerhalb derselben App können gleichzeitig Endpunkte mit unterschiedlichen Versionsnummern verwendet werden. Ein `/v2/` im URL-Pfad bedeutet deshalb nicht, dass die gesamte Plattform auf eine neue Architektur umgestellt wurde.

Der aktuelle `midea-local`-Code verwendet für Token und Key weiterhin [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py). Andere Funktionen können trotzdem unter anders versionierten Pfaden liegen.

### Offizielle Cloud-to-Cloud API V2

Midea dokumentiert tatsächlich eine [offizielle Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Diese verwendet unter anderem:

- OAuth 2.0
- `client_id` und `client_secret`
- kurzlebige Access-Tokens und Refresh-Tokens
- HMAC-SHA256-Signaturen
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- cloudbasierte Statusabfragen und Steuerbefehle

Das ist eine kontrollierte Partnerschnittstelle. Der benötigte `client_secret` wird einem Drittanbieter von Midea zugeteilt. Ein normaler Besitzer einer PortaSplit erhält ihn nicht einfach über sein MSmartHome-Konto. Die Anforderungen und Signaturregeln sind in der [offiziellen V2-Dokumentation](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html) beschrieben.

Diese API ist ausserdem nicht erst 2025 entstanden. Die Dokumentation enthält Request-Beispiele mit Zeitstempeln aus 2018 und einen Java-Kommentar vom 18. April 2019. Die V2-Partnerschnittstelle existierte somit bereits deutlich vor der Warnung in `Midea AC LAN`.

## Midea ersetzt tatsächlich eine V1-API – aber eine andere

Midea führt auch eine ältere offizielle Cloud-to-Cloud-Schnittstelle unter `/v1/open/...`. Deren Dokumentation trägt ausdrücklich den Hinweis, dass sie nicht mehr empfohlen werde, zukünftig abgeschaltet werden könne und die neue V2-Dokumentation verwendet werden solle. Das steht in Mideas [Dokumentation der alten Cloud-to-Cloud API](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Dieser Hinweis ist eine echte offizielle V1-zu-V2-Migration. Er betrifft aber die Partner-Endpunkte:

```text
/v1/open/...
           ↓
/v2/open/...
```

Die von den Home-Assistant-Bibliotheken verwendete Token-Abfrage lautet dagegen:

```text
/v1/iot/secure/getToken
```

Und die lokale PortaSplit-Verbindung läuft anschliessend gar nicht mehr über einen solchen Cloud-URL, sondern direkt im Heimnetz.

Die drei Schnittstellen nur aufgrund der Versionsnummer „V1" gleichzusetzen, wäre daher technisch nicht gerechtfertigt.

## Gibt es bereits eine vollständig cloudbasierte Home-Assistant-Integration?

Mit [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) existiert inzwischen eine Community-Integration, die Midea-Geräte über die Cloud statt direkt über das LAN steuert.

Auch das ist jedoch kein Beleg dafür, dass die offizielle Partner-V2-API die lokale Steuerung bereits ersetzt hätte. Der aktuelle Quellcode von `Midea Auto Cloud` verwendet unter anderem:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Diese Endpunkte sind im aktuellen [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) einsehbar.

Die Integration bildet damit private App- beziehungsweise Consumer-Cloud-Funktionen nach. Sie verwendet nicht einfach die dokumentierte `/v2/open/...`-Partnerschnittstelle.

Eine cloudbasierte Alternative existiert also bereits. Sie bringt aber auch die üblichen Abhängigkeiten einer Cloudintegration mit: Internetzugriff, funktionierendes Benutzerkonto, verfügbare Midea-Server und weiterhin kompatible private Endpunkte.

## Was bedeutet das konkret für PortaSplit-Besitzer?

### Bereits eingerichtete lokale Steuerung

Für eine bereits konfigurierte PortaSplit ist die Lage relativ entspannt. `Midea Smart AC` speichert Token und Key nach der Einrichtung lokal und benötigt laut eigener [Cloud-Dokumentation](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) für die weitere Steuerung keine Cloud-Verbindung.

Eine Abschaltung des reinen Token-Abrufs würde daher nicht automatisch die vorhandene lokale Verbindung beenden.

### Neueinrichtung oder Wiederherstellung

Grösser ist das Risiko bei:

- einer neuen Home-Assistant-Installation
- dem Wechsel auf eine andere Integration
- einem verlorenen oder beschädigten Backup
- einem Austausch des WLAN-Moduls
- Änderungen an der Gerätezuordnung
- einer erneuten Kopplung, falls sich dabei die Gerätezugangsdaten ändern

Für solche Fälle muss die Integration Token und Key erneut beschaffen oder der Benutzer muss sie manuell angeben. Dass `Midea Smart AC` eine manuelle Konfiguration unterstützt, ist in dessen [Konfigurationsdokumentation](https://github.com/mill1000/midea-ac-py#manual-configuration) beschrieben.

Ob ein Werksreset oder eine erneute Kopplung bei jeder PortaSplit zwingend neue Zugangsdaten erzeugt, ist nicht offiziell dokumentiert und sollte deshalb nicht pauschal behauptet werden.

### Eine echte Abschaltung der LAN-Steuerung

Damit eine bereits eingerichtete PortaSplit ihre lokal gespeicherten Zugangsdaten nicht mehr akzeptiert, müsste sich zusätzlich das Verhalten des Geräts oder WLAN-Moduls ändern – etwa durch eine neue Firmware oder ein geändertes Authentifizierungsverfahren.

Eine blosse Abschaltung des Cloud-Endpunkts `/v1/iot/secure/getToken` entfernt nicht automatisch die bereits im Gerät und in Home Assistant vorhandenen Zugangsdaten. Das folgt aus der von [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage) dokumentierten Trennung zwischen einmaligem Cloud-Abruf und anschliessender LAN-Steuerung.

Eine solche zukünftige Geräteänderung ist technisch möglich. Eine konkrete Ankündigung oder einen Abschalttermin speziell für die PortaSplit habe ich in den öffentlich zugänglichen Midea-Unterlagen jedoch nicht gefunden.

## Was ich weiterhin empfehlen würde

Trotz der relativierenden Erkenntnisse bleibt ein Backup sinnvoll.

Für V3-Geräte empfiehlt `Midea AC LAN` ausdrücklich, die erzeugte JSON-Konfiguration ausserhalb von HAOS zu sichern. Die aktuelle Empfehlung steht direkt in der [Projekt-README](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Dabei gilt:

- Token und Key wie Passwörter behandeln.
- Die JSON-Datei nicht in ein öffentliches Git-Repository laden.
- Keine ungeschwärzten Debug-Logs veröffentlichen.
- Das Backup verschlüsseln.
- Zusätzlich ein vollständiges Home-Assistant-Backup erstellen.
- Vor Firmware- und Integrationsupdates die aktuelle Funktion prüfen.
- Nach Updates die lokale Steuerung erneut testen.

Ein Backup ist eine vernünftige Absicherung gegen Cloudänderungen, Integrationsprobleme und eigene Fehler. Es ist aber kein Hinweis darauf, dass eine Abschaltung unmittelbar bevorsteht. Wie sich eine PortaSplit sauber einrichten und im Heimnetz absichern lässt, steht im [praktischen Teil zur Einrichtung](/blog/midea-portasplit-home-assistant-einrichten).

## Einordnung auf Basis der verfügbaren Belege

Die Warnung von `Midea AC LAN` sollte ernst genommen, aber korrekt eingeordnet werden.

Sie dokumentiert ein plausibles langfristiges Risiko: Midea könnte nicht ablaufende lokale Tokens als Sicherheitsproblem betrachten, die Beschaffung solcher Tokens weiter einschränken oder zukünftige Geräte stärker an die Cloud binden.

Nicht belegt ist dagegen eine offiziell angekündigte, terminierte Abschaltung der lokalen PortaSplit-Steuerung.

Der aktuelle technische Stand zeigt sogar das Gegenteil einer bereits vollzogenen Abschaltung: Im Juni 2026 lieferte der weiterhin verwendete V1-Token-Endpunkt gültige Zugangsdaten, nachdem der Request an das Format der offiziellen SmartHome-App angepasst worden war. Der entsprechende Fix ist heute Bestandteil der von `Midea AC LAN` eingesetzten Bibliothek.

Auch die offizielle Midea Cloud-to-Cloud API V2 existiert. Sie ist aber eine ältere, zugangsbeschränkte Partnerschnittstelle und nicht automatisch der Nachfolger des lokalen PortaSplit-Protokolls.

Die nüchterne Schlussfolgerung lautet deshalb:

> Backup erstellen, Integrationen beobachten und Cloudabhängigkeiten im Hinterkopf behalten – aber die lokale PortaSplit-Steuerung nicht aufgrund einer unbestätigten Abschaltungsannahme vorschnell abschreiben.

## Quellen

1.  [Midea AC LAN: aktuelle README und Abschaltungswarnung](https://github.com/wuwentao/midea_ac_lan#1-important-notice): Wortlaut der Warnung, Empfehlung zum Backup und Unterscheidung zwischen älteren V2- und neueren V3-Geräten.

2.  [Midea AC LAN PR #578 vom 19. Mai 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): Einführung der Warnung vor der schrittweisen Abschaltung der Token-Dienste und der behaupteten Migration zu einer cloudbasierten V2-API.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): Umstellung der dokumentierten Token-Quelle auf NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): Diskussion über die fehlerhafte SmartHome-Token-Abfrage und die vorübergehende Verwendung von NetHome Plus.

5.  [Kommentar des Midea-AC-LAN-Maintainers zur vermuteten V2-Migration](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): Kennzeichnet die Aussage über die neue V2-Cloud ausdrücklich als eigenes Verständnis.

6.  [Antwort des midea-msmart-Maintainers](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): Beschreibt die Existenz einer neuen V2-API als Vermutung und weist auf die eingeschränkten Reverse-Engineering-Möglichkeiten hin.

7.  [midea-local PR #470 vom 15. Juni 2026](https://github.com/midea-lan/midea-local/pull/470): Analyse des Fehlers 3004, Mitschnitt des offiziellen App-Requests, Ergänzung von `applianceCodes` und erfolgreicher Test mit vier V3-Klimageräten.

8.  [Unveränderlicher Commit des SmartHome-getToken-Fixes](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): Exakter Code-Diff des übernommenen Fixes.

9.  [Aktueller midea-local Cloud-Code](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): Weiterhin verwendeter Endpunkt `/v1/iot/secure/getToken` und aktuelles Request-Feld `applianceCodes`.

10.  [Aktuelles Manifest von Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): Verwendete Version von `midea-local` und Einstufung als lokale Push-Integration.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): Dokumentation der lokalen Steuerung, des einmaligen Cloud-Abrufs für V3-Geräte und der manuellen Konfiguration mit Token und Key.

12.  [Midea AC LAN Issue #607 zur PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): Konkretes PortaSplit-Beispiel mit Gerätetyp `0xAC`, Modell `00000Q1D`, Protokollversion 3 und erfolgreicher Einrichtung über NetHome Plus.

13.  [Offizielle Midea Cloud-to-Cloud API V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, Client-ID, Client-Secret, Access- und Refresh-Tokens, Signaturverfahren und `/v2/open/...`-Endpunkte.

14.  [Offizielle Midea Cloud-to-Cloud API V1](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): Offizieller Hinweis, dass die alte `/v1/open/...`-Partnerschnittstelle nicht mehr empfohlen wird und zukünftig abgeschaltet werden könnte.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) und [aktueller Cloud-Code](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): Community-Integration für vollständige Cloudsteuerung und die dabei tatsächlich verwendeten privaten V1-App-Endpunkte.
