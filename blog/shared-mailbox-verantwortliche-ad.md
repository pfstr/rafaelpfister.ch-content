---
title: "Verantwortliche für freigegebene Postfächer im Active Directory hinterlegen und auslesen"
navTitle: "Shared-Mailbox-Verantwortliche"
description: "DLP-Lösungen müssen bei einer blockierten Nachricht aus einem freigegebenen Postfach wissen, wen sie informieren. Welche AD-Attribute dafür taugen, welche nur Zugriffsrechte abbilden, warum Exchange-Online-Berechtigungen nicht im lokalen AD ankommen und wie ein Lieferant die Zuordnung per LDAP oder Microsoft Graph automatisiert ausliest."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 Min. Lesezeit"
themen:
  - "active-directory-entra"
  - "microsoft-365-exchange"
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "shared-mailbox-verantwortliche-ad"
translationId: "article-ed41bd7fefc08361"
url: "https://rafaelpfister.ch/blog/shared-mailbox-verantwortliche-ad"
---

# Verantwortliche für freigegebene Postfächer im Active Directory hinterlegen und auslesen

In einem Data-Loss-Prevention-Projekt kam folgende Anfrage auf meinen Tisch: Sobald eine Nachricht aus einem freigegebenen Postfach (Shared Mailbox) blockiert wird, sollen die verantwortlichen Personen benachrichtigt werden, damit sie reagieren können. Der Lieferant der DLP-Lösung soll diese Zuordnung automatisiert aus dem Active Directory lesen. Die Frage dazu: Wo steht im AD, wer für ein freigegebenes Postfach verantwortlich ist?

Kurzfazit: Das AD kennt nur Zugriffsrechte, keine Verantwortung. Einige Attribute zeigen, wer auf ein Postfach zugreifen oder daraus senden darf, und auch diese nur unvollständig, sobald das Postfach in Exchange Online liegt. Für eine verlässliche Automatisierung muss die verantwortliche Person ausdrücklich in einem Attribut gepflegt werden. Das Standardattribut `manager` eignet sich dafür, wird von Entra Connect synchronisiert und lässt sich per LDAP und Microsoft Graph abfragen.

## Zugriff ist nicht Verantwortung

Ein freigegebenes Postfach ist im AD ein deaktiviertes Benutzerobjekt mit Exchange-Attributen. Wer es benutzt, ergibt sich aus Berechtigungen: Vollzugriff (FullAccess), Senden als (Send As) und Senden im Auftrag von (Send on Behalf). Diese Listen enthalten oft zehn oder zwanzig Personen, dazu Stellvertretungen, Lernende und Dienstkonten. Eine DLP-Meldung an alle Berechtigten erreicht viele Personen, von denen sich niemand zuständig fühlt.

Die Frage nach der Verantwortung ist organisatorisch: Eine Person (oder eine kleine Gruppe) entscheidet, ob eine blockierte Nachricht legitim war, und pflegt die Berechtigungen. Diese Information existiert im AD nur, wenn sie jemand einträgt.

## Welche Attribute im AD in Frage kommen

| Attribut | Inhalt | Eignung für Verantwortliche |
|---|---|---|
| `manager` | DN einer Person, einwertig, Teil der Personen-Klassen und damit am Benutzerobjekt des Postfachs verfügbar | Geeignet. Wird von Entra Connect nach Entra ID synchronisiert und ist per Graph lesbar |
| `managedBy` | DN eines Verwalters, einwertig | Nicht am Benutzerobjekt verfügbar. Laut Schema unter anderem für Gruppen, Kontakte, Computer und OUs definiert, nicht für die Klasse User |
| `msExchDelegateListLink` | DNs der Personen, die Vollzugriff mit Automapping erhalten haben, mehrwertig; Rückverweis `msExchDelegateListLinkBL` am Benutzer | Nur Zugriff, und unvollständig: Vollzugriff ohne Automapping, Rechte über Gruppen und in Exchange Online vergebene Rechte fehlen |
| `publicDelegates` | DNs der Personen mit Senden im Auftrag von, mehrwertig | Nur Zugriff. Wird in Hybrid-Umgebungen von Entra Connect zurückgeschrieben |
| Berechtigungsliste (`nTSecurityDescriptor`) | Erweitertes Recht «Send As» am Benutzerobjekt | Nur lokal vergebenes Senden als. In Exchange Online vergebene Rechte stehen dort nicht |
| `msExchMailboxSecurityDescriptor` | Postfachberechtigungen lokaler Postfächer, binär | Nur Zugriff, nur für Postfächer auf lokalen Exchange-Servern, für Lieferanten schwer auszuwerten |
| `extensionAttribute1` bis `extensionAttribute15` | Freie Textfelder | Möglich, aber nur als Text; keine Referenz auf ein Objekt, keine automatische Bereinigung bei Umbenennung oder Austritt |

Der Hinweis, die Zuordnung stehe «direkt im AD», bezieht sich in der Praxis meist auf `msExchDelegateListLink`: In ADUC oder ADSI-Edit sieht man dort tatsächlich Personen. Das Attribut entsteht jedoch nur als Nebeneffekt des Outlook-Automappings und taugt nicht als Datenquelle für Zuständigkeiten.

## Wo das Postfach liegt, entscheidet über die Datenquelle

Liegen die Postfächer in Exchange Online und die Benutzer werden aus dem lokalen AD synchronisiert (Hybrid), gilt: Berechtigungen, die in Exchange Online vergeben werden, bleiben in Exchange Online. Entra Connect schreibt für Exchange Hybrid nur eine kleine Attributliste zurück (darunter `publicDelegates` und `proxyAddresses`), aber weder `msExchDelegateListLink` noch Send-As-Rechte. Ein lokales AD zeigt in diesem Fall den Stand vor der Migration plus das, was seither lokal nachgeführt wurde.

Den tatsächlichen Stand der Berechtigungen liefert für Exchange Online nur Exchange Online selbst (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Der Lieferant bräuchte dafür Zugriff auf Exchange Online PowerShell, was für eine reine Zuordnung unverhältnismässig ist. Ein gepflegtes Attribut am Benutzerobjekt ist einfacher und funktioniert unabhängig davon, wo das Postfach liegt.

## Empfehlung: `manager` als Verantwortlichen-Attribut

Für jedes freigegebene Postfach wird im Attribut `manager` die verantwortliche Person eingetragen. Gründe:

- Das Attribut ist Teil des Standardschemas, eine Schemaerweiterung entfällt.
- Es enthält eine Referenz (DN) statt Text. Wird die Person umbenannt oder verschoben, bleibt der Bezug gültig.
- Entra Connect synchronisiert `manager` nach Entra ID. Der Lieferant kann wahlweise per LDAP im lokalen AD oder per Microsoft Graph lesen.
- Über den Rückverweis `directReports` sieht jede verantwortliche Person (und jede Auswertung), für welche Postfächer sie zuständig ist.

Die Einschränkung: `manager` ist einwertig. Wo eine Stellvertretung zwingend ist, gibt es zwei Möglichkeiten. Möglichkeit 1: Eine Gruppe pro Postfach, deren Mitglieder die Verantwortlichen sind; ihr Name folgt einer Konvention (zum Beispiel `SM-Verantwortlich-buchhaltung`) oder steht in einem `extensionAttribute`. Möglichkeit 2: Die Stellvertretung wird beim Lieferanten nicht aus dem AD, sondern aus dem `manager` der verantwortlichen Person abgeleitet (Eskalation an die vorgesetzte Person). Möglichkeit 2 kommt ohne zusätzliche Objekte aus und genügt in den meisten Organisationen.

Das Attribut wird nur im führenden System gesetzt: bei synchronisierten Objekten im lokalen AD, bei reinen Cloud-Objekten in Entra ID.

## Bestand erheben: Wer hat heute Zugriff?

Für die Erstbefüllung hilft eine Liste der heutigen Berechtigten. Die verantwortliche Person ist meistens darunter; die Fachabteilung bestätigt die Auswahl. Bei Postfächern in Exchange Online:

```powershell
Connect-ExchangeOnline

$postfaecher = Get-EXOMailbox -RecipientTypeDetails SharedMailbox -ResultSize Unlimited

$bestand = foreach ($pf in $postfaecher) {
    $voll = Get-EXOMailboxPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'FullAccess' -and
            -not $_.IsInherited -and
            $_.User -notlike 'NT AUTHORITY\*'
        }
    $sendas = Get-EXORecipientPermission -Identity $pf.UserPrincipalName |
        Where-Object {
            $_.AccessRights -contains 'SendAs' -and
            $_.Trustee -notlike 'NT AUTHORITY\*'
        }
    [pscustomobject]@{
        Postfach  = $pf.PrimarySmtpAddress
        Vollzugriff = ($voll.User -join '; ')
        SendenAls = ($sendas.Trustee -join '; ')
    }
}

$bestand | Export-Csv -Path .\shared-mailbox-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Nur freigegebene Postfächer, keine Benutzer-, Raum- oder Gerätepostfächer |
| `-ResultSize Unlimited` | Alle Postfächer statt der Standardgrenze von 1000 |
| `-Identity` | Postfach, dessen Berechtigungen abgefragt werden |
| `AccessRights -contains 'FullAccess'` | Filtert auf Vollzugriff; andere Postfachrechte entfallen |
| `-not $_.IsInherited` | Entfernt geerbte Einträge (Administratoren, Systemkonten) |
| `User -notlike 'NT AUTHORITY\*'` | Entfernt den Eintrag `NT AUTHORITY\SELF` des Postfachs selbst |
| `-NoTypeInformation` | Keine Typzeile am Anfang der CSV-Datei |
| `-Encoding UTF8` | Umlaute in Namen bleiben erhalten |

</details>

Ist eine Gruppe berechtigt, erscheint ihr Name in der Liste; die Mitglieder lassen sich mit `Get-DistributionGroupMember` oder `Get-ADGroupMember` auflösen.

## Verantwortliche setzen

Die bestätigte Zuordnung wird als CSV mit den Spalten `Postfach` und `Verantwortlich` (jeweils E-Mail-Adresse) vorbereitet und im lokalen AD gesetzt:

```powershell
Import-Module ActiveDirectory

$zuordnung = Import-Csv -Path .\shared-mailbox-verantwortliche.csv

foreach ($eintrag in $zuordnung) {
    $postfach = Get-ADUser -Filter "mail -eq '$($eintrag.Postfach)'"
    $person = Get-ADUser -Filter "mail -eq '$($eintrag.Verantwortlich)'"
    Set-ADUser -Identity $postfach -Manager $person
}
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Filter "mail -eq '…'"` | Sucht das Objekt anhand der primären E-Mail-Adresse |
| `-Identity` | Benutzerobjekt des freigegebenen Postfachs |
| `-Manager` | Schreibt den DN der verantwortlichen Person in das Attribut `manager` |

</details>

Bei reinen Cloud-Objekten ohne lokales AD setzt `Set-MgUserManagerByRef` aus dem Microsoft-Graph-PowerShell-SDK denselben Wert in Entra ID.

## Auslesen durch den Lieferanten: LDAP

Freigegebene Postfächer lassen sich im AD über `msExchRecipientTypeDetails` erkennen: `4` steht für ein freigegebenes Postfach auf einem lokalen Exchange-Server, `34359738368` für ein freigegebenes Postfach in Exchange Online (RemoteSharedMailbox). Die Abfrage, die der Lieferant in seine Lösung übernehmen kann:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

Der Wert in `manager` ist ein DN. Die E-Mail-Adresse der verantwortlichen Person ergibt eine zweite Abfrage auf diesen DN mit dem Attribut `mail`. Ein Lesekonto ohne weitere Rechte genügt, da `manager` und `mail` für authentifizierte Benutzer lesbar sind. Die Verbindung sollte über LDAPS (Port 636) laufen.

Zum Prüfen der Zuordnung in PowerShell:

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

$verantwortlich = @{
    Name       = 'Verantwortlich'
    Expression = {
        if ($_.manager) {
            (Get-ADUser -Identity $_.manager -Properties mail).mail
        }
    }
}

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Select-Object mail, $verantwortlich |
    Sort-Object mail
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-LDAPFilter` | LDAP-Filter auf die beiden Empfängertypen für freigegebene Postfächer |
| `-Properties mail, manager` | Lädt die beiden Attribute, die `Get-ADUser` standardmässig nicht liefert |
| `Expression = { … }` | Löst den DN in `manager` zur E-Mail-Adresse der verantwortlichen Person auf |

</details>

Für die Nachrichtenzuordnung sollte der Lieferant alle Adressen aus `proxyAddresses` berücksichtigen, nicht nur `mail`: Eine Nachricht kann mit einer sekundären Adresse des Postfachs versendet werden.

## Auslesen durch den Lieferanten: Microsoft Graph

Hat der Lieferant keinen LDAP-Zugang zum internen Netz, bietet sich Microsoft Graph an. Freigegebene Postfächer existieren in Entra ID als (deaktivierte) Benutzerobjekte, `manager` ist dort durch die Synchronisation vorhanden:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

Die App-Registrierung des Lieferanten braucht dafür die Anwendungsberechtigung `User.Read.All`. Eine Liste aller freigegebenen Postfächer liefert Graph nicht direkt, da Entra ID den Postfachtyp nicht kennt. Der Lieferant fragt deshalb gezielt die Absenderadresse der blockierten Nachricht ab, was für den DLP-Anwendungsfall ohnehin genügt.

## Einbindung in die DLP-Benachrichtigung

Wie die Benachrichtigung technisch ausgelöst wird, hängt von der DLP-Lösung ab. Bei Microsoft Purview DLP ist zu beachten: Benachrichtigungen lassen sich zusätzlich an fest eingetragene Personen senden, jedoch nur an einzelne Empfänger, nicht an Gruppen oder Verteilerlisten. Eine dynamische Zuordnung «Postfach zu verantwortlicher Person» ist in der Regel nicht vorgesehen. Bei wenigen Postfächern ist eine Regel pro Postfach denkbar; bei vielen Postfächern braucht es einen nachgelagerten Prozess, der Vorfälle abholt, die Absenderadresse per LDAP oder Graph auflöst und die verantwortliche Person informiert.

Für eine externe Lösung gilt dieselbe Logik: Absenderadresse der blockierten Nachricht ermitteln, Postfach über `mail` oder `proxyAddresses` finden, `manager` auflösen, benachrichtigen. Fehlt `manager`, sollte die Lösung an eine definierte Sammeladresse (Servicedesk oder Datenschutz) melden statt die Meldung zu verwerfen.

## Pflege im Betrieb

Damit die Zuordnung aktuell bleibt, genügen in der Praxis zwei Regeln:

- Neue freigegebene Postfächer werden nur mit eingetragener verantwortlicher Person angelegt; der Antrag enthält das Feld.
- Ein monatlicher Bericht listet Postfächer ohne `manager` und Postfächer, deren verantwortliche Person deaktiviert ist.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

Beim Austritt einer Person zeigt `directReports` an ihrem Objekt, welche Postfächer eine neue verantwortliche Person brauchen. Diese Prüfung lässt sich in den Austrittsprozess aufnehmen.

## Quellen

1.  [Manager attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): einwertiges DN-Attribut der Personen-Klassen, Rückverweis `directReports`.

2.  [Managed-By attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): Liste der Klassen, für die `managedBy` definiert ist; die Klasse User gehört nicht dazu.

3.  [Microsoft Entra Connect Sync: Attributes synchronized to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): synchronisierte Attribute inklusive `manager` sowie die Liste der Exchange-Hybrid-Writeback-Attribute.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): Abfrage der Postfachberechtigungen in Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): Abfrage der Send-As-Berechtigungen in Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): Microsoft-Support-Artikel zum Zusammenhang zwischen Vollzugriff und `msExchDelegateListLink` in Exchange 2016 und 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): Graph-Endpunkt und benötigte Berechtigungen für das Auslesen des Managers.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): Empfänger von DLP-Benachrichtigungen in Microsoft Purview; nur einzelne Empfänger, keine Gruppen.
