---
title: "RDP-Anmeldung scheitert: Kennwort muss vor der ersten Anmeldung geändert werden"
navTitle: "RDP: Kennwort ändern"
description: "Ein neues Konto mit Initialkennwort, und die Remotedesktopverbindung bricht mit «You must change your password before logging on the first time» ab. Die Ursache ist die Network Level Authentication, die keinen Kennwortwechsel kennt. Das Kennwort lässt sich trotzdem ändern: über eine andere Windows-Sitzung, vom eigenen PC aus, per PowerShell oder über RD Web Access."
date: "2026-11-11"
kategorie: "Windows-Client"
timeToRead: "6 Min. Lesezeit"
themen:
  - "windows-client"
  - "active-directory-entra"
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "rdp-kennwort-aendern-nla"
translationId: "article-96f19bf85c2f7ccd"
url: "https://rafaelpfister.ch/blog/rdp-kennwort-aendern-nla"
draft: false
---

# RDP-Anmeldung scheitert: Kennwort muss vor der ersten Anmeldung geändert werden

Bei einem frisch eingerichteten Konto in einer Kundenumgebung ist mir die Situation wieder begegnet: Das Initialkennwort stimmt, die Remotedesktopverbindung wird trotzdem sofort getrennt. Der Windows-Client `mstsc` meldet dazu:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Andere RDP-Clients und Verbindungsmanager zeigen denselben Zustand mit eigenem Wortlaut an, zum Beispiel «The user password must be changed before logging on for the first time». Bei einem abgelaufenen Kennwort lautet die Meldung «This user account's password has expired. The password must change in order to logon». Ursache und Lösung sind in beiden Fällen dieselben.

Kurzfazit: Das Kennwort lässt sich in dieser Situation nicht über die RDP-Verbindung ändern. Es muss vorher auf einem anderen Weg geändert werden, und dafür genügt in den meisten Fällen die Netzwerkverbindung zu einem Domain Controller.

## Ursache: NLA authentisiert vor dem Aufbau der Sitzung

Auf dem Konto ist in Active Directory die Option **Benutzer muss Kennwort bei der nächsten Anmeldung ändern** gesetzt (technisch: `pwdLastSet` = 0). An einem Arbeitsplatz in der Domäne ist das unproblematisch: Der Anmeldebildschirm erkennt den Zustand und fragt direkt nach dem neuen Kennwort.

Bei RDP fehlt dieser Schritt, sobald die Network Level Authentication (NLA) aktiv ist. NLA authentisiert den Benutzer über CredSSP, bevor auf dem Zielsystem eine Sitzung mit Anmeldebildschirm entsteht. Die Authentisierung mit einem Kennwort, das geändert werden muss, schlägt fehl, und ein Dialog für den Kennwortwechsel ist an dieser Stelle nicht vorgesehen. Der Client trennt die Verbindung und zeigt nur die Meldung an. NLA ist bei aktuellen Windows-Versionen die Voreinstellung und in gehärteten Umgebungen per Gruppenrichtlinie erzwungen.

## Möglichkeiten für den Benutzer

Es gibt vier Möglichkeiten, das Kennwort ohne die betroffene RDP-Verbindung zu ändern. Welche davon in Frage kommt, hängt davon ab, was vom eigenen Arbeitsplatz aus erreichbar ist.

| Möglichkeit | Voraussetzung |
|---|---|
| 1. Kennwortportal oder Servicedesk | vom Betreiber bereitgestellt |
| 2. Bestehende Windows-Sitzung in der Domäne | Zugang mit einem anderen Konto oder zu einem anderen System |
| 3. Kennwortdialog am eigenen PC | Domain Controller erreichbar, zum Beispiel über VPN |
| 4. PowerShell mit `NetUserChangePassword` | TCP 445 zum Domain Controller erreichbar |

### Möglichkeit 1: Kennwortportal oder Servicedesk

In grösseren Umgebungen gibt es für externe Mitarbeitende meist einen vorgesehenen Weg: ein Self-Service-Portal, eine Kennwortfunktion im VPN- oder Citrix-Portal oder ein PAM-System. Ist nichts davon vorhanden, kann der Servicedesk das Flag entfernen oder ein Kennwort ohne Änderungszwang setzen (siehe Abschnitt für Administratoren).

### Möglichkeit 2: Bestehende Windows-Sitzung in der Domäne

Besteht bereits eine Sitzung auf einem System der Domäne, etwa auf einem Jumphost oder mit einem zweiten Konto, lässt sich das Kennwort dort ändern. In einer RDP-Sitzung öffnet `Ctrl+Alt+End` den Sicherheitsbildschirm des entfernten Systems (`Ctrl+Alt+Del` würde auf dem lokalen PC ausgewertet). Unter **Kennwort ändern** ist das Feld mit dem Benutzernamen editierbar: Tragen Sie dort das betroffene Konto in der Form `DOMAIN\benutzer` ein, darunter das Initialkennwort und zweimal das neue Kennwort.

### Möglichkeit 3: Kennwortdialog am eigenen PC

Derselbe Dialog funktioniert auch an einem PC, der nicht Mitglied der Domäne ist, sofern er einen Domain Controller erreicht, zum Beispiel über eine VPN-Verbindung. `Ctrl+Alt+Del` drücken, **Kennwort ändern** wählen und den vorausgefüllten lokalen Benutzernamen durch `DOMAIN\benutzer` ersetzen. Der PC muss den Domänennamen per DNS auflösen können; ohne die DNS-Server der Zielumgebung findet Windows keinen Domain Controller.

### Möglichkeit 4: PowerShell mit NetUserChangePassword

Die Windows-API `NetUserChangePassword` ändert ein Kennwort direkt auf einem Domain Controller. Sie prüft das bisherige Kennwort beim Wechsel selbst und setzt keine vorherige Anmeldung voraus. Deshalb funktioniert sie auch dann, wenn das Kennwort abgelaufen ist oder geändert werden muss, und auch von einem PC ausserhalb der Domäne. Das Skript bindet die Funktion per P/Invoke ein und fragt beide Kennwörter verdeckt ab:

```powershell
$api = Add-Type -Namespace Win32 -Name NetApi32 -PassThru -MemberDefinition @'
[DllImport("netapi32.dll", CharSet = CharSet.Unicode)]
public static extern uint NetUserChangePassword(
    string domainname, string username,
    string oldpassword, string newpassword);
'@

$dc   = 'dc01.example.com'
$user = 'm.muster'
$alt  = Read-Host -AsSecureString 'Bisheriges Kennwort'
$neu  = Read-Host -AsSecureString 'Neues Kennwort'

$klartext = { param($s) [System.Net.NetworkCredential]::new('', $s).Password }
$rc = $api::NetUserChangePassword(
    $dc, $user, (& $klartext $alt), (& $klartext $neu))

if ($rc -eq 0) { 'Kennwort geändert' } else { "Fehler $rc" }
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Argument | Wirkung |
|---|---|
| `domainname` | DNS- oder NetBIOS-Name eines Domain Controllers oder der Domäne. Von einem PC ausserhalb der Domäne den Domain Controller direkt angeben (Name oder IP-Adresse). |
| `username` | Anmeldename des Kontos (`sAMAccountName`), ohne Domänenanteil. |
| `oldpassword` | Bisheriges Kennwort, hier das Initialkennwort. Es muss exakt stimmen. |
| `newpassword` | Neues Kennwort. Es muss die Kennwortrichtlinie der Domäne erfüllen. |

</details>

Der Rückgabewert 0 bedeutet Erfolg. Die häufigsten Fehlercodes:

| Code | Bedeutung |
|---|---|
| 5 | Zugriff verweigert, zum Beispiel weil der Domain Controller den Aufruf ohne Anmeldung nicht zulässt |
| 86 | Das bisherige Kennwort ist falsch |
| 2221 | Der Benutzername wurde nicht gefunden |
| 2245 | Das neue Kennwort erfüllt die Kennwortrichtlinie nicht (Länge, Komplexität, Historie, Mindestalter) |
| 2351 | Der angegebene Servername ist ungültig |

Der Aufruf läuft über SMB und benötigt TCP-Port 445 zum Domain Controller. In stark segmentierten Netzen ist dieser Port von Client-Netzen oder aus dem VPN oft gesperrt; dann bleibt Möglichkeit 1.

`Set-ADAccountPassword` aus dem Active-Directory-Modul hilft dem betroffenen Benutzer in dieser Lage nicht: Das Cmdlet braucht eine authentisierte Verbindung zu den Active Directory Web Services, und genau diese Anmeldung ist mit dem betroffenen Konto nicht möglich. Es ist das Werkzeug für den Administrator.

## NLA im Client abschalten: nur ohne erzwungene NLA

Häufig wird empfohlen, CredSSP in der .rdp-Datei zu deaktivieren:

```text
enablecredsspsupport:i:0
```

Der Client verbindet sich dann ohne NLA und erhält den klassischen Anmeldebildschirm des Zielsystems, auf dem der Kennwortwechsel möglich ist. Das gelingt nur, wenn das Zielsystem Verbindungen ohne NLA annimmt. Ist dort **Verbindungen nur von Computern zulassen, auf denen Remotedesktop mit Authentifizierung auf Netzwerkebene ausgeführt wird** aktiv, lehnt es die Verbindung ab. Bei aktuellen Windows-Server-Versionen ist das die Voreinstellung.

## Möglichkeiten für Administratoren

Auf der Administrationsseite lässt sich das Problem vermeiden oder beheben:

- **Flag entfernen:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` hebt den Änderungszwang auf. Der Benutzer kann sich mit dem Initialkennwort anmelden und es in der Sitzung über `Ctrl+Alt+End` ändern. Das Initialkennwort bleibt bis dahin gültig und muss deshalb auf einem sicheren Kanal übermittelt werden.
- **Kennwort zurücksetzen:** `Set-ADAccountPassword -Identity m.muster -Reset` setzt ein neues Kennwort. Prüfen Sie danach, ob das Flag noch gesetzt ist.
- **RD Web Access:** Ist die Rolle vorhanden, kann sie den Kennwortwechsel für abgelaufene Kennwörter anbieten. Im IIS-Manager unter **Sites → Default Web Site → RDWeb → Pages** in den **Application Settings** den Wert `PasswordChangeEnabled` auf `true` setzen. Die Seite für den Kennwortwechsel liegt unter `/RDWeb/Pages/en-US/password.aspx`.
- **Konten für reinen RDP-Zugriff:** Für Benutzer, die ausschliesslich per RDP arbeiten (externe Dienstleister, Administratoren auf Jumphosts), ist der Änderungszwang bei der ersten Anmeldung ohne einen der oben genannten Wege nicht erfüllbar. Der Weg für den ersten Kennwortwechsel sollte deshalb Teil des Onboardings sein.

NLA auf dem Zielsystem abzuschalten, behebt das Symptom ebenfalls, entfernt aber eine Schutzschicht: Ohne NLA erreicht jeder Client im Netz den Anmeldebildschirm und damit den RDP-Dienst, bevor er sich authentisiert hat. Microsoft rät davon ab.

## Quellen

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): Liste der Trenncodes des RDP-Clients, darunter 4615 (Kennwort muss vor der ersten Anmeldung geändert werden) und 3847 (Kennwort abgelaufen), mit dem Wortlaut der Meldungen.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): API-Referenz mit Parametern, Berechtigungen und Rückgabewerten.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): Praxisbericht von Evotec zum Kennwortwechsel per NetUserChangePassword von einem PC ausserhalb der Domäne, inklusive Hinweis auf TCP 445 und auf den wirkungslosen enablecredsspsupport-Eintrag bei erzwungener NLA.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referenz der .rdp-Eigenschaften, darunter enablecredsspsupport mit Werten und Default.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): Abschnitt zum Kennwortwechsel für abgelaufene Kennwörter über RD Web Access und die Einstellung PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): Cmdlet-Referenz mit den Parametern Reset, OldPassword, NewPassword, Credential und Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): Microsoft-Hinweise zu Anmeldefehlern mit NLA; das Abschalten von NLA ist dort als nicht empfohlen gekennzeichnet.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): Blogbeitrag zum editierbaren Benutzerfeld im Dialog «Kennwort ändern», mit dem sich das Kennwort eines Kontos einer anderen Domäne ändern lässt.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): Anleitung von Windows OS Hub zur Einstellung PasswordChangeEnabled und zum Pfad der Seite password.aspx.
