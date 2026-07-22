---
title: "Ghost Sender oder Ghost Admin? Ein MX-Record ist keine Firewall!"
navTitle: "Ghost Sender"
description: "Warum Ghost Sender keine neue Sicherheitslücke in Exchange Online ist, sondern die erwartbare Folge eines offenen SMTP-Nebeneingangs."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min to read"
themen:
  - "microsoft-365-exchange"
slug: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/blog/ghost-sender-exchange-online-nebeneingang"
image: "../images/ghost-admin.png"
---

# Ghost Sender oder Ghost Admin? Ein MX-Record ist keine Firewall!

![Ein Ghost-Admin hält im Rechenzentrum die Tür neben dem Security-Gate auf, während E-Mails am Filter vorbei direkt ins Postfach gelangen.](../images/ghost-admin.png)

**InfoGuard Labs hat mit «Ghost-Sender – Universal Email Spoofing against Exchange Online» soviel Besorgnis ausgelöst, dass einige Anfragen zur «Sicherheitslücke» auch auf meinem Tisch gelandet sind. Kurzfazit: Das beschriebene Risiko ist real. Die Einordnung als universelle Sicherheitslücke in Exchange Online ist es jedoch nicht. Wer einen Third-Party-Mailfilter vor Exchange Online stellt, den weiterhin erreichbaren EXO-Endpunkt aber nicht auf diesen Filter beschränkt, hat keine neue Lücke entdeckt, sondern seinen Mailflow nicht fertig konfiguriert.**

Es scheint, als hätte man bei InfoGuard für einen Moment vergessen, wie ein Mail Transfer Agent (MTA) funktioniert. Ein Mailserver, der Postfächer für eine Domain bedient, nimmt grundsätzlich SMTP-Verbindungen aus dem Internet entgegen. Genau dafür ist er da. Ein MX-Record sagt regulären Absendern lediglich, welchen Weg sie für die Zustellung nehmen sollen. Er ist aber weder eine Firewall-Regel noch eine Access Control List.

## Was «Ghost Sender» tatsächlich zeigt

Das von [InfoGuard Labs beschriebene Szenario](https://labs.infoguard.ch/posts/ghost-sender/) sieht so aus:

1. Eine Organisation betreibt ihre Postfächer in Exchange Online.
2. Der öffentliche MX-Record zeigt auf ein vorgeschaltetes Secure Email Gateway.
3. Der Exchange-Online-Endpunkt unter `*.mail.protection.outlook.com` bleibt direkt aus dem Internet erreichbar.
4. Der Administrator hat Exchange Online nicht so eingeschränkt, dass nur das vorgeschaltete Gateway dort zustellen darf.
5. Ein Angreifer ignoriert den MX-Record und liefert seine Nachricht direkt bei Exchange Online ein.

Der vorgesehene Weg lautet also:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Offen geblieben ist jedoch dieser Weg:

```text
Angreifer -> Exchange Online -> Postfach
```

Das ist eine ernst zu nehmende Fehlkonfiguration. Der vorgeschaltete Filter kann auf diesem Weg umgangen werden; gefälschte Absender, Phishing und CEO-Fraud werden dadurch erheblich erleichtert. InfoGuard gebührt Anerkennung dafür, das Problem sichtbar gemacht, seine Verbreitung untersucht und einen einfach nutzbaren Test veröffentlicht zu haben.

Aber wo genau soll hier der Produktfehler sein?

Auch die mediale Zuspitzung hilft bei der Einordnung wenig. [Heise titelt, Exchange Online lasse gefälschte E-Mails «anstandslos durch»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), obwohl eben nur bestimmte, unvollständig gehärtete Drittanbieter- und Hybridkonfigurationen betroffen sind. Deutlich treffender formuliert es [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): kein Sicherheitsloch im engeren Sinn, sondern ein Design- und Konfigurationsproblem.

## «An MTA is doing MTA-Things»

Jeder Exchange-Online-Tenant besitzt einen öffentlichen SMTP-Endpunkt. Dieser Endpunkt ist kein Geheimnis und soll auch keines sein. Microsoft erklärt selbst, dass Exchange Online standardmässig Nachrichten annimmt, die direkt an dort gehostete Postfächer adressiert sind: [Das sei schlicht die Funktionsweise von E-Mail](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

Auch [SMTP selbst beschreibt den MX-Record als Mechanismus zur Ermittlung des regulären Zielsystems](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). Daraus folgt keine Verpflichtung des Zielservers, Verbindungen über jeden anderen erreichbaren Host abzulehnen. Ein Angreifer muss sich nicht an den ausgeschilderten Weg halten. Wenn ein weiterer MTA erreichbar ist, die Empfängerdomain kennt und die Nachricht akzeptiert, dann wird er ausprobiert, ganz ähnlich wie Spammer seit Jahrzehnten schlechter geschützte Backup-MX-Systeme anzusprechen versuchen.

Wer einen Drittfilter vorschaltet, verändert die Standardtopologie. Aus «Exchange Online ist mein Internet-Mailgateway» wird «nur mein Drittanbieter-Gateway darf Internet-Mail an Exchange Online übergeben». Diese neue `Trust-Border` entsteht nicht durch einen DNS-Eintrag. Sie muss am empfangenden System ausdrücklich erzwungen werden.

Genau das dokumentiert Microsoft: Bei einem externen MX soll ein Inbound Connector vom Typ `Partner` angelegt werden, der für `SenderDomains *` nur das Zertifikat oder die Quell-IP-Adressen des vorgeschalteten Dienstes akzeptiert. Direkt am Gateway vorbei zugestellte Nachrichten werden dann abgewiesen. Das steht so 1:1 in Microsofts Anleitung [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Auch Frank Carius beschreibt diesen «Nebeneingang» in der [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) ausführlich.

## SPF, DKIM und DMARC sind kein Türsteher

InfoGuard zeigt Nachrichten, bei denen SPF, DKIM und DMARC fehlschlagen und die dennoch im Postfach landen. Das sieht spektakulär aus, ist aber kein kryptografischer «Bypass» dieser Verfahren. Die Mails schlagen gerade nicht erfolgreich durch. Sie liefern `fail`. Entscheidend ist, welche lokale Aktion das empfangende System aus diesem Ergebnis ableitet.

SPF prüft, ob ein System für den Envelope-Sender senden darf. DKIM prüft eine Signatur. DMARC verbindet diese Ergebnisse mit der sichtbaren Absenderdomain und veröffentlicht eine gewünschte Behandlung. Selbst der aktuelle [DMARC-Standard RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) hält ausdrücklich fest, dass der Empfänger diese gewünschte Behandlung berücksichtigen kann, aber nicht dazu verpflichtet ist. DMARC ist ein wichtiges Signal, jedoch keine Netzwerk-Zugriffskontrolle.

Bei einem vorgeschalteten Gateway kommt hinzu, dass Exchange Online zunächst die IP-Adresse dieses Gateways und nicht jene des ursprünglichen Absenders sieht. Dafür gibt es [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Es rekonstruiert die ursprüngliche Quelle und verbessert SPF-, DKIM-, DMARC-, Anti-Spoofing- und Anti-Phishing-Auswertungen. Enhanced Filtering ist aber ebenfalls kein Türschloss. Es ersetzt den restriktiven Partner-Connector nicht.

Besonders offensichtlich wird die Fehlkonfiguration, wenn ein Administrator die EOP-Prüfung per SCL-Bypass abschwächt oder ganz aus dem Weg nimmt, weil ja bereits das vorgeschaltete Produkt filtern soll, gleichzeitig aber die direkte Internetzustellung offenlässt. Dann hat er nicht einen Schutzmechanismus «umgangen» bekommen, sondern für einen von zwei Eingängen bewusst keinen wirksamen Schutz mehr vorgesehen.

Man kann Microsoft durchaus dafür kritisieren, wenn eine Nachricht trotz deutlich sichtbarem Authentisierungsfehler ohne Warnung im Posteingang landet. Man kann die Semantik der Connector-Typen, die Dokumentation und fehlende Warnungen im Configuration Analyzer kritisieren. All das sind legitime Punkte. Die Existenz eines öffentlich erreichbaren SMTP-Endpunkts ist jedoch keine Sicherheitslücke.

## «Direct Send» ist nicht gleich «Direkte Zustellung»

In der Diskussion werden zwei Dinge vermischt:

- **Direct Send** bezeichnet bei Microsoft anonyme Nachrichten, deren Envelope-Sender (`5321.MailFrom`) eine eigene Accepted Domain des Tenants verwendet.
- **Direkte Zustellung an Exchange Online** bezeichnet allgemein eine SMTP-Nachricht, die den veröffentlichten Drittanbieter-MX ignoriert und unmittelbar beim Exchange-Endpunkt eingeliefert wird. Der Absender kann dabei auch eine beliebige externe Domain verwenden.

Der Schalter

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

ist sinnvoll, wenn Direct Send nicht benötigt wird. Er verhindert internes Domain-Spoofing über diesen Pfad. Er schliesst aber nicht den gesamten Nebeneingang für beliebige externe Absender. Microsoft beschreibt den genauen Geltungsbereich in der [Cmdlet-Dokumentation zu `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Wer «Ghost Sender» vollständig verhindern will, braucht weiterhin die Zugriffsbeschränkung per Partner-Connector oder eine passende Mailflow-Regel.

## Muss Microsoft dem Administrator wirklich alles abnehmen?

Nein. Wer einen zusätzlichen Mailfilter in eine produktive Transportkette einbaut, übernimmt die Verantwortung für diese Transportkette.

Der Anbieter kann nicht zuverlässig erraten, ob neben dem externen MX noch Scanner, Multifunktionsgeräte, SaaS-Dienste, Hybrid-Server, Partner-Relays oder andere legitime Systeme direkt an Exchange Online senden müssen. Ein automatisches «Der MX zeigt woandershin, also blockiere alles andere» würde in zahlreichen realen Umgebungen erwünschte Mailflüsse unterbrechen. Deshalb muss der Administrator die gewünschte Vertrauensgrenze explizit definieren.

Trotzdem darf Microsoft es den Verantwortlichen leichter machen. Ein guter Configuration Analyzer sollte einen externen MX ohne restriktiven Partner-Connector erkennen und deutlich warnen. Der Einrichtungsdialog könnte erklären, dass ein Connector vom Typ «Ihre Organisation» zwar passende Verbindungen identifiziert, unpassende Verbindungen aber nicht automatisch ablehnt. Secure-by-default-Schalter und bessere Betriebsberichte wären ebenfalls willkommen.

Das wäre sinnvolle Produkthärtung. Es ändert aber nichts an der technischen Einordnung: Eine unsichere Sondertopologie bleibt eine unsichere Konfiguration und wird nicht allein durch ihre weite Verbreitung zu einem Zero-Day.

## So wird der Nebeneingang geschlossen

Für Umgebungen mit vorgeschaltetem Filter gehören mindestens diese Punkte auf die Checkliste:

1. **Mailflow vollständig dokumentieren.** Welche Systeme dürfen tatsächlich an Exchange Online zustellen? Dazu gehören auch Hybrid-, Applikations- und Notfallpfade.
2. **Restriktiven Partner-Connector einrichten.** `SenderDomains *` verwenden und die Zustellung auf ein Zertifikat (bevorzugt) oder auf gepflegte Quell-IP-Bereiche beschränken. Ein Connector vom Typ `OnPremises` beziehungsweise «Ihre Organisation» erzwingt diese Default-Deny-Wirkung nicht.
3. **Enhanced Filtering korrekt konfigurieren.** Wenn EOP weiterhin filtern soll, müssen Original-IP und Absenderinformationen sauber rekonstruiert werden. Pauschale SCL-`-1`-Bypässe sind kritisch zu prüfen.
4. **Direct Send deaktivieren, falls unbenutzt.** Vorher mit Message Trace beziehungsweise den verfügbaren Berichten prüfen, ob Scanner oder Anwendungen davon abhängen.
5. **Nicht blind umschalten.** Gateway-IP-Bereiche, Zertifikatswechsel, Hybrid-Mailflow sowie `onmicrosoft.com`-, Teams- und andere Sonderpfade testen und anschliessend überwachen.

Ein vereinfachtes Beispiel für die IP-basierte Variante lautet:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

Wo möglich, ist die Zertifikatsbindung der IP-Allowlist vorzuziehen. Änderungen gehören zuerst in einen kontrollierten Test, denn eine fehlerhafte Allowlist macht aus dem offenen Nebeneingang sehr schnell einen vollständigen Mailausfall.

## Der einfache Selbsttest

Der von InfoGuard (und der MSXFAQ) gezeigte Test ist nützlich:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

Bei einem korrekt beschränkten Partner-Connector ist eine SMTP-Ablehnung wie `5.7.51 TenantInboundAttribution; Rejecting` zu erwarten. Eine alternative Transportregel kann die Nachricht zunächst annehmen und danach in Quarantäne verschieben; deshalb müssen neben der SMTP-Antwort auch Message Trace, Quarantäne und Postfach kontrolliert werden. `Send-MailMessage` (deprecated) dient hier nur der leicht verständlichen Illustration. Jedes kontrollierte SMTP-Testwerkzeug erfüllt denselben Zweck.

## Fazit: Guter Test, falsches Etikett

«Ghost Sender» ist kein neuer SMTP-Exploit. Es ist ein griffiger Name für einen offenen Nebeneingang, dessen Absicherung Microsoft seit langem dokumentiert und den der Administrator offengelassen hat.

Das Ironische daran: InfoGuard bezeichnet das Problem im eigenen Beitrag selbst als «widespread and systematic misconfiguration» und schliesst mit dem Satz «Ghost-Sender is a misconfiguration». Auch Microsofts Security Response Center stufte die Meldung zunächst nicht als Sicherheitslücke ein. Die Fakten sind im Artikel also durchaus vorhanden: nur Titel, Testmail und «Vulnerability»-Branding erzählen leider eine dramatischere Geschichte.

Der sinnvolle Teil der Veröffentlichung ist der Weckruf: Viele Unternehmen haben ihren Mailflow offenbar nicht sauber verriegelt. Der problematische Teil ist die Behauptung, Exchange Online habe dafür eine universelle Sicherheitslücke. Nein: Exchange Online verhält sich hier zunächst wie ein MTA. Unsicher wird es durch die nicht zu Ende konfigurierte Vertrauensgrenze.

Muss man dem Administrator wirklich alles abnehmen? Nein. Aber man muss offenbar immer wieder daran erinnern, dass DNS-Routing keine Zugriffskontrolle ersetzt.

## Quellen

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/) — Die ursprüngliche Untersuchung samt Verbreitungsanalyse und dem selbst gezogenen Fazit «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/) — Der von InfoGuard veröffentlichte Online-Test, um den eigenen Tenant auf den offenen Nebeneingang zu prüfen.

3.  [MSXFAQ: Exchange Online als Nebeneingang für Mailempfang](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm) — Frank Carius' Einordnung: kein Fehler in Exchange Online, sondern eine Fehlkonfiguration des Administrators.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865) — Microsoft erklärt, dass die direkte Annahme von Mail an gehostete Postfächer die Funktionsweise von E-Mail ist, und grenzt Direct Send ab.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud) — Die offizielle Anleitung mit dem eigenen Schritt zum restriktiven Partner-Connector bei externem MX.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors) — Rekonstruiert die ursprüngliche Absenderquelle hinter einem Gateway; verbessert die Auswertung, ersetzt den Connector aber nicht.

7.  [Heise: Ghost-Sender – Exchange Online lässt gefälschte E-Mails anstandslos durch](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html) — Beispiel für die zugespitzte Berichterstattung, die nur bestimmte Fehlkonfigurationen verallgemeinert.

8.  [Crow in the Cloud: Die Geister, die ich nicht rief](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) — Treffende Einordnung als Design- und Konfigurationsproblem samt Schutzmassnahmen.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html) — Beschreibt den MX-Record als Mechanismus zur Ermittlung des regulären Zielsystems, nicht als Zugriffskontrolle.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html) — Hält fest, dass der Empfänger die veröffentlichte DMARC-Behandlung berücksichtigen kann, aber nicht muss.

---

## Ist Ihr Mailflow sicher?

Unsicher, ob Ihr Exchange-Online-Tenant ebenfalls einen offenen Nebeneingang hat? **adeptio** prüft Ihren kompletten Mailflow: von MX-Records, Connectors und Drittanbieter-Gateways bis zu EOP, SPF, DKIM, DMARC und Direct Send. Praxisnah, unabhängig und mit konkreten Empfehlungen.

Wer seinen Mailflow prüfen oder sauber absichern lassen möchte, kann gerne ein unverbindliches Beratungsgespräch vereinbaren:

**[Beratungsgespräch mit adeptio buchen](https://outlook.office.com/book/Erstgesprchadeptio%40adeptio.ch/s/Akxr6wxKAEGw3d5sEmi-AQ2?ismsaljsauthenabled=)**  
[adeptio.ch](https://adeptio.ch/)
