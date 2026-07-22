---
title: "HIN Mailgateway auf SEPPmail-Basis: Backup & Disaster Recovery im Cluster (und was sich mit Stargate ändert)"
navTitle: "Backup & Recovery"
description: "Das HIN Mailgateway beruht auf einer SEPPmail-Appliance. Für Backup und Disaster Recovery ist entscheidend, was die Appliance sichert (nur Konfiguration und Schlüsselmaterial, keine Mails), wie die Cluster-Replikation funktioniert und warum sie kein Backup ersetzt."
date: "2026-07-08"
kategorie: "HIN-Gateway"
timeToRead: "15 min to read"
themen:
  - "hin-gateway"
slug: "hin-mailgateway-backup-disaster-recovery"
url: "https://rafaelpfister.ch/blog/hin-mailgateway-backup-disaster-recovery"
---

# HIN Mailgateway auf SEPPmail-Basis: Backup & Disaster Recovery im Cluster (und was sich mit Stargate ändert)

Fast jedes produktive HIN Mailgateway (MGW) läuft im Clusterverbund. Mancher Betreiber verlässt sich stillschweigend darauf, dass diese Redundanz auch die Datensicherung übernimmt. Das ist ein Trugschluss. Ein Cluster schützt vor dem Ausfall eines Knotens, nicht vor einer fehlerhaften Regeländerung, einem gelöschten Zertifikat oder einem korrupten Import, denn [systemrelevante Daten werden zuverlässig auf alle Knoten repliziert](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html), Fehler eingeschlossen.

  

Backup und Disaster Recovery lassen sich nur planen, wenn die technische Basis klar ist. Das HIN Mailgateway beruht auf einer SEPPmail-Appliance mit dem GINA-Verfahren; es gilt damit die dokumentierte SEPPmail-Mechanik, die beim Backup einige Besonderheiten aufweist.

  

## Was das HIN MGW technisch ist

Das Gateway verarbeitet ein- und ausgehende Mails nach einem zentralen Regelwerk und verschlüsselt je nach Empfänger per S/MIME, OpenPGP oder TLS; für Empfänger ohne eigenes Schlüsselmaterial dient das webbasierte GINA-Verfahren. Für das Backup ist entscheidend, dass [Nachrichteninhalte auf dem Gateway nicht persistent gespeichert werden](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Die Appliance verarbeitet Mails im Durchlauf, ohne sie zu archivieren.

  

## Cluster-Architektur: was repliziert wird

SEPPmail kennt mehrere [Cluster-Ausprägungen – High-Availability, Load-Balancing und Geo-Cluster](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html); über alle Knoten synchronisiert werden Systemparameter, Benutzerdaten und Schlüsselmaterial. Beim [Frontend/Backend-Cluster verfügt das Frontend über keine eigene Konfigurationsdatenbank](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html): Es lässt sich in einer DMZ ohne Datenablage betreiben und erhält nur die zur aktuellen Verarbeitung nötigen Daten; die Datenbank samt Schlüsseln liegt auf dem Backend. Für [Large File Transfer (LFT) gilt eine Ausnahme](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html): Jedem Partner (auch Frontends) wird eine gleich grosse Platte zugeordnet, und die LFT-Daten werden auf alle Knoten synchronisiert.

  

## Warum Replikation kein Backup ist

> *Replikation kopiert den aktuellen Zustand, auch den fehlerhaften. Ein Backup bewahrt einen bekannten, funktionierenden Stand.*

Ein fehlerhafter Import, ein gelöschter Schlüssel oder eine deaktivierte Domain wird innerhalb von Sekunden auf den Partnerknoten repliziert. Ohne unabhängiges Backup existiert danach kein Wiederherstellungspunkt mehr. Wie eng Verfügbarkeit und Konsistenz im Cluster zusammenhängen, zeigte sich bei den [Login-Problemen nach dem Update auf 15.0.5](/blog/hin-update-issue-version-15.0.5), die durch gestörte Cluster-Replikation ausgelöst wurden.

  

## Was im Backup enthalten ist und was nicht

Das [SEPPmail-Backup ist bewusst schlank](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html): Es umfasst ausschliesslich Konfiguration und kryptografisches Schlüsselmaterial: [keine Nachrichten, keine Mailqueue und ausdrücklich keine Logs](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) (Logs gehören deshalb per Syslog auf ein externes System). Seit Firmware 14.0.0 erzeugt die Appliance das Backup [automatisch nachts um Mitternacht als backup.tgz](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); abrufen lässt es sich über `Download`, `Send Backup` (E-Mail an die Backup-Gruppe) oder SCP.

| **Im Backup enthalten** | **Nicht im Backup** |
| --- | --- |
| Systemkonfiguration und Regelwerk | E-Mail-Inhalte / Nachrichtentexte |
| Benutzer- und GINA-Konten | Aktuelle Mailqueue |
| Schlüsselmaterial: S/MIME, X.509, OpenPGP | System- und Mail-Logs (extern per Syslog sichern) |
| TLS- und Zertifikatskonfiguration | Betriebssystem / VM-Image |


Daraus folgt: Weil das Betriebssystem nicht im Konfigurations-Backup enthalten ist, benötigt eine vollständige DR-Strategie zusätzlich einen Weg, die Appliance-Basis wiederherzustellen (Neuausrollung aus Hersteller-Image oder VM-Snapshot). Das Konfigurations-Backup ergänzt anschliessend Konfiguration und Schlüssel.

  

## Snapshots sind kein Cluster-Backup

Seit Firmware 14.0.0 legt die Appliance zusätzlich [lokale Snapshots an, allerdings nur, wenn eine LFT-Partition mit Datenbank vorhanden ist](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Sonntags entsteht ein vollständiger, montags bis samstags je ein inkrementeller Snapshot; die Aufbewahrung beträgt 14 Tage.

Entscheidend für die DR-Planung: Im Clusterbetrieb laufen diese Snapshots zwar im Hintergrund, es wird daraus aber kein Restore angeboten. Snapshots sind damit eine lokale Rollback-Hilfe auf Einzelsystemen, kein Cluster-Recovery. Die verlässliche Sicherung bleibt das verschlüsselte Konfigurations-Backup.

  

## Backup einrichten

Voraussetzung für jeden Abrufweg ist ein gesetztes Backup-Passwort unter [Administration › Backup › Change password](https://docs.seppmail.com/de/07_mi_11_adm__administration.html); ohne dieses Passwort wird weder heruntergeladen noch versendet oder per SCP bereitgestellt. Standardmässig geht das nächtliche Backup per E-Mail an die [Gruppe «backup (Backup Operator)»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html); ein dedizierter Backup-Benutzer benötigt eine gültige interne Mailadresse.

-   Backup-Passwort setzen und [getrennt vom Backup aufbewahren](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html): Das Backup enthält private Schlüssel.
    
-   Für die automatisierte Ablage die [Backups per SCP abholen](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html): öffentlichen `SSH-RSA`\-Schlüssel in der Administration hinterlegen und über den OS-Benutzer `backup` die um Mitternacht bereitgestellte `backup.tgz` abziehen.
    
-   Logs separat sichern (externer Syslog), da sie [bewusst nicht Teil des Backups](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) sind.
    

  

## Backup-Strategie im Clusterbetrieb

Im Clusterbetrieb sind eine geordnete Sicherung und eine konsistente Versionsführung entscheidend.

-   **Täglich**: verschlüsseltes Konfigurations-Backup [per SCP](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) abholen und extern versioniert ablegen
    
-   **Wöchentlich**: vollständiges VM- oder System-Backup beider Knoten, zeitlich versetzt statt gleichzeitig (das Betriebssystem ist nicht Teil des Konfigurations-Backups)
    
-   **Vor Wartung oder Update**: Mailannahme über [Preempt anhalten](https://docs.seppmail.com/de/07_mi_11_adm__administration.html): Eingehende Mails werden dann mit einem konfigurierbaren SMTP-Return-Code (Standard `421`) temporär abgewiesen; die Einstellung bleibt auch nach einem Neustart aktiv.
    

  

Zur Versionsführung: SEPPmail aktualisiert im Frontend/Backend-Cluster [das Frontend vor dem Backend](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), und bei mehrstufigen Updates müssen alle Partner denselben Versionsstand aufweisen, bevor auf das nächste Release gehoben wird. Nach einem Major-Update kann eine Neugenerierung des Rulesets nötig sein (Meldung *«Current ruleset created for another version»*).

  

## Restore und Disaster Recovery

Der Grundfall ist unkompliziert: [Import backup file](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), Neustart, anschliessend arbeitet das Gateway mit vollem Funktionsumfang. Zu beachten ist die Versionsregel: Nur das Backup des unmittelbar vorherigen Firmware-Stands lässt sich in den aktuellen einspielen (danach das Ruleset neu generieren); das Backup einer neueren Firmware auf eine ältere Version einzuspielen ist nicht möglich.

Im Cluster gilt eine wichtige Einschränkung:

-   **Einzelnen Knoten nie direkt restaurieren**: Ein [Restore eines einzelnen Cluster-Partners ist nicht vorgesehen](https://docs.seppmail.com/de/07_mi_11_adm__administration.html). Stattdessen die defekte Maschine aus dem Cluster entfernen, eine neue VM aufsetzen und wieder hinzufügen: Konfiguration und Schlüssel kommen automatisch per Replikation vom intakten Partner.
    
-   **Totalverlust auf allen Knoten**: Appliance aus dem Basis-Image neu ausrollen, danach das letzte bekannte, funktionierende Konfigurations-Backup importieren und neu starten.
    

Ein Backup ist nur so verlässlich wie der letzte erfolgreiche Restore-Test. Ein Testrestore sollte mindestens zweimal jährlich in einer isolierten Umgebung erfolgen, nicht gegen den produktiven Cluster.

  

### Restore-Checkliste für den Ernstfall

1.  Defekten Knoten aus dem Cluster entfernen (kein Direkt-Restore eines Partners).
    
2.  Neue VM aufsetzen bzw. bei Totalverlust die Appliance aus Basis-Image/VM-Snapshot bereitstellen.
    
3.  Nur bei Totalverlust: letztes funktionierendes Konfigurations-Backup importieren (Passwort bereithalten, Versionsregel beachten).
    
4.  Knoten isoliert prüfen: SMTP-Annahme, TLS, GINA, Regelwerk.
    
5.  In den Cluster aufnehmen und Replikation beobachten; bei Meldung das Ruleset neu generieren.
    
6.  Vorfall dokumentieren sowie Backup-Intervall und Versionsstände nachziehen.
    

  

Zwei Wartungsaktionen verdienen besondere Vorsicht und immer ein vorheriges Backup: Die [Vergrösserung der LFT-Partition fährt die Appliance herunter](https://docs.seppmail.com/de/07_mi_11_adm__administration.html), und der Factory Reset überschreibt die Festplatte zehnfach (die Sicherheitsabfrage verlangt den Code in umgekehrter Schreibweise).

  

## Was sich mit «Stargate» ändert

HIN ersetzt das bisherige Mailgateway schrittweise durch das [neue HIN Gateway](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) (Projekt «Stargate», bei der Zuger [Vereign AG als «Verimesh»](https://www.vereign.com/) geführt). Es handelt sich nicht um einen 1:1-Ersatz der Appliance, sondern um einen architektonischen Wechsel, der Backup und Disaster Recovery im Kern betrifft:

-   **Von zentral zu dezentral**: Nodes kommunizieren direkt miteinander; ein zentrales Verteilzentrum entfällt.
    
-   **Dezentrales Schlüsselmanagement (DKMS)**: Jede Organisation verwaltet ihre eigene kryptografische Identität, ohne zentrale Certificate Authority.
    
-   **Ende-zu-Ende-Verschlüsselung** mit Fragmentierung der Nachrichten.
    
-   **Resilienz aus dem Netz**: Fällt ein Knoten aus, bleibt das Mesh funktionsfähig.
    
-   **Offene Referenzimplementierung**: Die [Vereign Client Library (vcl)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) ist unter AGPLv3 quelloffen einsehbar.
    

Zeitplan: Die dezentrale Infrastruktur ist im Schweizer Gesundheitswesen [seit April 2025 produktiv im Einsatz](https://www.vereign.com/); für 2026 sind die schrittweise Ablösung der bisherigen Mailgateways und ein breiter Rollout vorgesehen. Organisationen mit HIN-eigenen Domains (`@hin.ch`, `@verband-hin.ch`) laufen auf HIN-Infrastruktur und sind von der Umstellung kaum betroffen.

  

Für das Betriebshandbuch bedeutet das: Die klassische Disziplin «Appliance-Konfiguration und Schlüssel exportieren und auf einen Ersatzknoten zurückspielen» verliert an Bedeutung. An ihre Stelle treten Node-Enrollment, Identitäts- und Schlüsselverwahrung im Mesh sowie die Wiederaufnahme von Knoten in das Netz.

  

## Fazit

Solange das HIN MGW auf SEPPmail-Technik läuft, gilt: Der Cluster kompensiert Hardware-Ausfälle, die Verantwortung für Konfigurations- und Schlüsselintegrität verbleibt jedoch beim Betreiber. Das schlanke Konfigurations-Backup gehört unabhängig vom Cluster gesichert (per SCP, versioniert, mit getrennt verwahrtem Passwort), Snapshots ersetzen es im Cluster nicht, die Versionsstände bleiben synchron, und der Restore wird regelmässig isoliert getestet. Der Umstieg auf Stargate sollte frühzeitig in die DR-Planung einfliessen, da er Resilienz und Schlüsselverwahrung in das dezentrale Netz verlagert.

## Quellen

1.  [SEPPmail-Dokumentation – «Backup / Restore»](https://docs.seppmail.com/de/03_wp_03_sa_07_sm_03_backup-restore.html) — Backup-Inhalt (nur Konfiguration und Schlüsselmaterial), nächtliche Erstellung, automatische Cluster-Wiederherstellung per Replikation.
    
2.  [SEPPmail-Dokumentation – «Administration»](https://docs.seppmail.com/de/07_mi_11_adm__administration.html) — Ausführliche Referenz: Backup-Menü (Download / Send Backup / Change password, `backup.tgz` um Mitternacht), LFT-Snapshots (14 Tage, im Cluster kein Restore), Restore-Regeln und Cluster-Vorgehen, Preempt (SMTP-Return-Code, Standard 421), Device Cloning, Update-Kanäle und Update-Reihenfolge (Frontend vor Backend), Factory Reset, Bulk-Import/-Export.
    
3.  [SEPPmail-Dokumentation – «Backup Benutzer erstellen»](https://docs.seppmail.com/de/04_com_06_bc_03_ism_03_create-backup-user.html) — Gruppe «backup (Backup operator)», Verschlüsselung und Passwortverwaltung.
    
4.  [SEPPmail-Dokumentation – «Backup per SCP kopieren»](https://docs.seppmail.com/de/09_ht_backup_copy-instead-of-sending-mail.html) — Abholung der `backup.tgz` via SCP über den OS-Benutzer `backup` statt Mailversand.
    
5.  [SEPPmail-Dokumentation – «Cluster / Hochverfügbarkeit»](https://docs.seppmail.com/ch/04_com_09_cl_01_general.html) — Cluster-Typen und die über alle Knoten synchronisierten Daten (Systemparameter, Benutzerdaten, Schlüsselmaterial).
    
6.  [SEPPmail-Dokumentation – «Frontend/Backend-Cluster»](https://docs.seppmail.com/de/04_com_09_cl_05_frontend-backend-cluster.html) — Frontend ohne Konfigurationsdatenbank, DMZ-Betrieb, on-demand-Daten; Backend als Datenhalter.
    
7.  [SEPPmail-Dokumentation – «Datenablage im Cluster (LFT)»](https://docs.seppmail.com/ch/09_ht_lft_data-storage-in-cluster.html) — Gleich grosse Zusatzplatte je Partner, Synchronisation der LFT-Daten auf alle Knoten.
    
8.  [HIN AG – «Vom Mailgateway zum Data Mesh»](https://www.hin.ch/de/blog/2025/vom-mailgateway-zum-data-mesh.cfm) — HIN-Kommunikation zum Nachfolger Stargate: dezentrale Nodes, Data-Mesh-Konzept, Zeitplan, Ende-zu-Ende-Verschlüsselung.
    
9.  [Vereign AG – «Verimesh» / Vereign Client Library (vcl, Tag 0.4-rc1)](https://code.vereign.com/code/vcl/-/tree/0.4-rc1) — Technische Grundlage von Stargate: dezentrales Schlüsselmanagement (DKMS), Ende-zu-Ende-Verschlüsselung mit Nachrichtenfragmentierung, quelloffen unter AGPLv3.
