---
title: "RDP-inloggning misslyckas: Lösenordet måste ändras före första inloggningen"
navTitle: "RDP: ändra lösenord"
description: "Ett nytt konto med initialt lösenord och fjärrskrivbordsanslutningen avbryts med «You must change your password before logging on the first time». Orsaken är Network Level Authentication, som inte stöder lösenordsbyte. Lösenordet kan ändå ändras: via en annan Windows-session, från den egna datorn, med PowerShell eller via RD Web Access."
date: "2026-11-11"
kategorie: "Windows-klient"
timeToRead: "6 min läsning"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "rdp-inloggning-misslyckas-losenordet-maste-andras-fore-forsta-inloggningen"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/sv/blog/rdp-inloggning-misslyckas-losenordet-maste-andras-fore-forsta-inloggningen
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:32:53.440Z
translationReview: automatic
---

# RDP-inloggning misslyckas: Lösenordet måste ändras före första inloggningen

I en nykonfigurerad kundmiljö stötte jag på situationen igen: Det initiala lösenordet stämmer, men fjärrskrivbordsanslutningen kopplas ändå genast ned. Windows-klienten `mstsc` visar då följande:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Andra RDP-klienter och anslutningshanterare visar samma tillstånd med egen formulering, till exempel «The user password must be changed before logging on for the first time». Vid ett utgånget lösenord lyder meddelandet «This user account's password has expired. The password must change in order to logon». Orsak och lösning är desamma i båda fallen.

Kort sagt: Lösenordet kan inte ändras via RDP-anslutningen i den här situationen. Det måste ändras på annat sätt i förväg, och i de flesta fall räcker en nätverksanslutning till en Domain Controller.

## Orsak: NLA autentiserar innan sessionen etableras

Kontot har alternativet **Användaren måste byta lösenord vid nästa inloggning** aktiverat i Active Directory (tekniskt: `pwdLastSet` = 0). På en arbetsstation i domänen är detta inget problem: Inloggningsskärmen känner igen tillståndet och frågar direkt efter det nya lösenordet.

Med RDP saknas detta steg när Network Level Authentication (NLA) är aktiverat. NLA autentiserar användaren via CredSSP innan en session med inloggningsskärm skapas på målsystemet. Autentisering med ett lösenord som måste ändras misslyckas, och någon dialog för lösenordsbyte är inte avsedd i detta skede. Klienten kopplar ned anslutningen och visar bara meddelandet. NLA är standardinställningen i aktuella Windows-versioner och tvingas fram via grupprincip i härdade miljöer.

## Alternativ för användaren

Det finns fyra sätt att ändra lösenordet utan den berörda RDP-anslutningen. Vilket som är aktuellt beror på vad som kan nås från den egna arbetsstationen.

| Alternativ | Förutsättning |
|---|---|
| 1. Lösenordsportal eller servicedesk | tillhandahålls av operatören |
| 2. Befintlig Windows-session i domänen | åtkomst med ett annat konto eller till ett annat system |
| 3. Lösenordsdialog på den egna datorn | Domain Controller nåbar, till exempel via VPN |
| 4. PowerShell med `NetUserChangePassword` | TCP 445 till Domain Controller nåbar |

### Alternativ 1: Lösenordsportal eller servicedesk

I större miljöer finns det vanligtvis en avsedd lösning för externa medarbetare: en självbetjäningsportal, en lösenordsfunktion i VPN- eller Citrix-portalen eller ett PAM-system. Om inget av detta finns kan servicedesk ta bort flaggan eller ange ett lösenord utan krav på ändring (se avsnittet för administratörer).

### Alternativ 2: Befintlig Windows-session i domänen

Om det redan finns en session på ett system i domänen, exempelvis på en jumphost eller med ett andra konto, kan lösenordet ändras där. I en RDP-session öppnar `Ctrl+Alt+End` säkerhetsskärmen för fjärrsystemet (`Ctrl+Alt+Del` skulle tolkas på den lokala datorn). Under **Ändra lösenord** kan fältet med användarnamnet redigeras: Ange det berörda kontot i formatet `DOMAIN\benutzer` där, följt av det initiala lösenordet och det nya lösenordet två gånger.

### Alternativ 3: Lösenordsdialog på den egna datorn

Samma dialog fungerar även på en dator som inte är medlem i domänen, förutsatt att den når en Domain Controller, till exempel via en VPN-anslutning. Tryck på `Ctrl+Alt+Del`, välj **Ändra lösenord** och ersätt det förifyllda lokala användarnamnet med `DOMAIN\benutzer`. Datorn måste kunna slå upp domännamnet via DNS; utan DNS-servrarna i målmiljön hittar Windows ingen Domain Controller.

### Alternativ 4: PowerShell med NetUserChangePassword

Windows-API:et `NetUserChangePassword` ändrar ett lösenord direkt på en Domain Controller. Det kontrollerar självt det tidigare lösenordet vid bytet och kräver ingen föregående inloggning. Därför fungerar det även när lösenordet har gått ut eller måste ändras, och även från en dator utanför domänen. Skriptet binder in funktionen via P/Invoke och frågar efter båda lösenorden dolt:

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
<summary>Förklarade alternativ</summary>

| Argument | Effekt |
|---|---|
| `domainname` | DNS- eller NetBIOS-namn för en Domain Controller eller domänen. Ange Domain Controller direkt från en dator utanför domänen (namn eller IP-adress). |
| `username` | Kontots inloggningsnamn (`sAMAccountName`), utan domändel. |
| `oldpassword` | Tidigare lösenord, här det initiala lösenordet. Det måste stämma exakt. |
| `newpassword` | Nytt lösenord. Det måste uppfylla domänens lösenordsprincip. |

</details>

Returvärdet 0 betyder att åtgärden lyckades. De vanligaste felkoderna:

| Kod | Betydelse |
|---|---|
| 5 | Åtkomst nekad, till exempel eftersom Domain Controller inte tillåter anropet utan inloggning |
| 86 | Det tidigare lösenordet är fel |
| 2221 | Användarnamnet hittades inte |
| 2245 | Det nya lösenordet uppfyller inte lösenordsprincipen (längd, komplexitet, historik, minimiålder) |
| 2351 | Det angivna servernamnet är ogiltigt |

Anropet sker via SMB och kräver TCP-port 445 till Domain Controller. I starkt segmenterade nät är denna port ofta blockerad från klientnät eller via VPN; då återstår alternativ 1.

`Set-ADAccountPassword` från Active Directory-modulen hjälper inte den berörda användaren i detta läge: Cmdlet:en kräver en autentiserad anslutning till Active Directory Web Services, och just denna inloggning är inte möjlig med det berörda kontot. Det är ett verktyg för administratören.

## Stänga av NLA i klienten: endast utan framtvingad NLA

Det rekommenderas ofta att inaktivera CredSSP i .rdp-filen:

```text
enablecredsspsupport:i:0
```

Klienten ansluter då utan NLA och får målsystemets klassiska inloggningsskärm, där lösenordet kan ändras. Detta fungerar endast om målsystemet accepterar anslutningar utan NLA. Om **Tillåt endast anslutningar från datorer som kör Fjärrskrivbord med autentisering på nätverksnivå** är aktiverat där, avvisas anslutningen. I aktuella Windows Server-versioner är detta standardinställningen.

## Alternativ för administratörer

På administrationssidan kan problemet undvikas eller åtgärdas:

- **Ta bort flaggan:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` tar bort kravet på lösenordsbyte. Användaren kan logga in med det initiala lösenordet och ändra det i sessionen via `Ctrl+Alt+End`. Det initiala lösenordet förblir giltigt tills dess och måste därför överföras via en säker kanal.
- **Återställ lösenordet:** `Set-ADAccountPassword -Identity m.muster -Reset` anger ett nytt lösenord. Kontrollera därefter om flaggan fortfarande är aktiverad.
- **RD Web Access:** Om rollen finns kan den erbjuda lösenordsbyte för utgångna lösenord. I IIS Manager, under **Sites → Default Web Site → RDWeb → Pages**, anger du värdet `PasswordChangeEnabled` till `true` i **Application Settings**. Sidan för lösenordsbyte finns under `/RDWeb/Pages/en-US/password.aspx`.
- **Konton för enbart RDP-åtkomst:** För användare som arbetar uteslutande via RDP (externa tjänsteleverantörer, administratörer på jumphostar) går kravet på lösenordsbyte vid första inloggningen inte att uppfylla utan någon av ovanstående vägar. Vägen för det första lösenordsbytet bör därför ingå i onboardingen.

Att stänga av NLA på målsystemet åtgärdar också symptomet, men tar bort ett skyddslager: Utan NLA når varje klient i nätet inloggningsskärmen och därmed RDP-tjänsten innan autentisering har skett. Microsoft avråder från detta.

## Källor

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): Lista över RDP-klientens frånkopplingskoder, däribland 4615 (lösenordet måste ändras före första inloggningen) och 3847 (lösenordet har gått ut), med meddelandenas formuleringar.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): API-referens med parametrar, behörigheter och returvärden.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): Praktisk rapport från Evotec om lösenordsbyte via NetUserChangePassword från en dator utanför domänen, inklusive information om TCP 445 och den verkningslösa posten enablecredsspsupport vid framtvingad NLA.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referens för .rdp-egenskaper, inklusive enablecredsspsupport med värden och standardvärde.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): Avsnitt om lösenordsbyte för utgångna lösenord via RD Web Access och inställningen PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): Cmdlet-referens med parametrarna Reset, OldPassword, NewPassword, Credential och Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): Microsoft-anvisningar om inloggningsfel med NLA; att stänga av NLA markeras där som inte rekommenderat.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): Blogginlägg om det redigerbara användarfältet i dialogrutan «Ändra lösenord», som gör det möjligt att ändra lösenordet för ett konto i en annan domän.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): Guide från Windows OS Hub om inställningen PasswordChangeEnabled och sökvägen till sidan password.aspx.
