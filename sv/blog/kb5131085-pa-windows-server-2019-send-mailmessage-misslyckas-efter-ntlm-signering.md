---
title: "KB5131085 på Windows Server 2019: Send-MailMessage misslyckas efter NTLM-signering"
navTitle: "KB5131085 e-postutskick"
description: "Efter novemberuppdateringen KB5131085 misslyckas e-postutskick via Send-MailMessage mot Postfix-servrar på Windows Server 2019. Orsaken anses vara den nya NTLM-signeringen från KB5135557. Artikeln förklarar varför en SMTP-klient överhuvudtaget använder NTLM, hur du kontrollerar fallet och vilka lösningar som finns: tvinga AUTH LOGIN, stänga av NTLM på e-postservern eller tillfälligt återställa det via registret."
date: "2026-11-17"
kategorie: "Windows och SMTP"
timeToRead: "7 min. lästid"
themen:
  - windows-client
  - smtp-mailflow
produkte:
  - "windows-client"
protokolle:
  - "smtp"
  - "powershell"
  - "troubleshooting"
  - "releases"
slug: "kb5131085-pa-windows-server-2019-send-mailmessage-misslyckas-efter-ntlm-signering"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/sv/blog/kb5131085-pa-windows-server-2019-send-mailmessage-misslyckas-efter-ntlm-signering
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:20:30.421Z
translationReview: automatic
---

# KB5131085 på Windows Server 2019: Send-MailMessage misslyckas efter NTLM-signering

Sedan patchdagen den 10 november 2026 har administratörer rapporterat att skript på Windows Server 2019 inte längre kan skicka e-post. Berörda är säkerhetskopieringsrapporter, övervakningsmeddelanden och liknande uppgifter som i åratal har skickat till en Postfix-server med PowerShell-cmdleten `Send-MailMessage`. Efter avinstallation av den kumulativa uppdateringen KB5131085 (OS Build 17763.9487) fungerar utskicken igen. Günter Born beskrev fallet den 17 november 2026 på Borns IT- och Windows-Blog, utlöst av en läsares rapport.

Microsoft listar ännu inte problemet som ett Known Issue. Den enda Known Issue-posten för KB5131085 gäller aktiveringskommandon med felet `0x80041008`. Orsaken är därför inte officiellt bekräftad, men den går att härleda väl från ändringarna i uppdateringen och beteendet hos .NET-klassen `SmtpClient`.

## Vad KB5131085 ändrar i NTLM

Uppdateringen innehåller skyddsåtgärder mot NTLM-reläattacker, som Microsoft beskriver i KB5135557 (”NTLM signing security improvements in Windows”). De gäller Windows 11, Windows 10 Enterprise LTSC 2021 och Windows Server 2016 till 2025. I ändringsloggen för KB5131085 står det att klienter från och med nu som standard erbjuder NTLM Message Signing för alla icke-anonyma anslutningar.

Tekniskt innebär det att en Windows-klient nu sätter flaggan `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) och det nödvändiga `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`) i NTLM-förhandlingen, även om programmet inte har begärt integritet. Tidigare fick sådana program endast `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, en platshållare utan kryptografiskt skydd. Microsoft distribuerar ändringen i två faser:

| Fas | Datum | Beteende |
|---|---|---|
| 1 | 10 november 2026 | Klienter erbjuder signering på alla anslutningar. Servrar accepterar signerade och osignerade klienter och loggar osignerade. |
| 2 | Maj 2027 | Windows-servrar kräver signering som standard och avvisar klienter utan signering med `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

Enligt Microsoft återgår en Windows-klient utan fel till det gamla beteendet om servern inte bekräftar signering. Alla Windows-servrar gör det. Motparter med en egen NTLM-implementering, till exempel en Linux-e-postserver, omfattas inte av detta påstående.

## Varför en SMTP-klient använder NTLM

`Send-MailMessage` i Windows PowerShell 5.1 använder internt .NET-klassen `System.Net.Mail.SmtpClient`. Den kontrollerar raden `AUTH` i serverns EHLO-svar och provar metoderna i en fast ordning: först `GSSAPI` (Negotiate), sedan `NTLM`, därefter `WDigest`, och sist `LOGIN`. Om servern erbjuder NTLM använder `SmtpClient` NTLM, även om användarnamn och lösenord skickas med `-Credential`.

Postfix implementerar inte själv SMTP-autentisering utan delegerar den till Cyrus SASL eller Dovecot. Båda kan erbjuda NTLM: Cyrus SASL via en egen plugin, Dovecot fram till version 2.3 via `ntlm` i `auth_mechanisms` (Dovecot 2.4 har inte längre NTLM). Om NTLM är aktivt annonserar Postfix exempelvis `AUTH PLAIN LOGIN NTLM`, och varje Windows-klient autentiserar sig med NTLM utan att någon märker det. Med KB5131085 erbjuder klienten dessutom signering och nyckelutbyte i denna förhandling. Om e-postserverns NTLM-implementering inte kan hantera detta misslyckas inloggningen och `Send-MailMessage` avbryts. En kommentator i Born-artikeln anger NTLM-signering som orsak, och fyndet stämmer med detta: läsarens lösning fungerar eftersom den utesluter NTLM.

## Kontrollera fallet

Fastställ först om e-postservern erbjuder NTLM. Från en Linux-dator visar EHLO-svaret efter STARTTLS de erbjudna metoderna:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-starttls smtp` | Upprättar anslutningen i klartext och växlar till TLS via STARTTLS |
| `-crlf` | Skickar radslut som CRLF, enligt SMTP-kraven |
| `-quiet` | Undertrycker certifikat- och sessionsdetaljer |
| `-connect mail.example.com:587` | Målvärd och Submission-port |

</details>

Ange `EHLO test` när anslutningen har upprättats. Om raden `250-AUTH` innehåller posten `NTLM` är servern berörd. På själva Postfix-servern visar två kommandon vilken SASL-backend som är aktiv och vilka metoder Dovecot erbjuder:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

På Windows-servern loggar NTLM-loggen anslutningar med de förhandlade flaggorna från och med Windows Server 2019. Händelse 4022 är klientposten:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Läser den operativa NTLM-loggen |
| `-MaxEvents 50` | Begränsar utdata till de 50 senaste posterna |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtrerar på granskningshändelserna för klient (4022) och server (4023) |
| `Format-List TimeCreated, Id, Message` | Visar tidpunkt, händelse-ID och texten med Negotiated Flags |

</details>

En post vid tidpunkten för utskicksförsöket med e-postservern som mål bekräftar att `SmtpClient` använde NTLM.

## Lösning 1: Tvinga AUTH LOGIN i skriptet

`Send-MailMessage` erbjuder ingen parameter för att ange inloggningsmetod. Klassen `SmtpClient` kan göra detta när den anropas direkt: en `CredentialCache` lämnar endast ut inloggningsuppgifterna för en viss metod. Om bara `Login` har lagts in hoppar `SmtpClient` över NTLM och använder `AUTH LOGIN`. Följande skript bygger på läsarens exempel i Born-artikeln:

```powershell
$server = 'mail.example.com'
$port   = 587
$cred   = Import-Clixml -Path 'C:\Skripte\smtp.cred'

$cache = New-Object System.Net.CredentialCache
$cache.Add($server, $port, 'Login', $cred.GetNetworkCredential())

$smtp = New-Object System.Net.Mail.SmtpClient($server, $port)
$smtp.EnableSsl   = $true
$smtp.Credentials = $cache

$msg = New-Object System.Net.Mail.MailMessage
$msg.From = 'backup@example.com'
$msg.To.Add('admin@example.com')
$msg.Subject      = 'Backup-Report'
$msg.Body         = 'Sicherung abgeschlossen.'
$msg.BodyEncoding = [System.Text.Encoding]::UTF8

try {
    $smtp.Send($msg)
}
finally {
    $msg.Dispose()
    $smtp.Dispose()
}
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `Import-Clixml -Path …` | Läser in ett PSCredential som tidigare sparats med `Get-Credential \| Export-Clixml`; bundet till användare och dator via DPAPI |
| `$cache.Add($server, $port, 'Login', …)` | Lagrar inloggningsuppgifterna endast för värd, port och metoden `Login` |
| `SmtpClient($server, $port)` | Skapar SMTP-klienten för server och Submission-port |
| `EnableSsl = $true` | Växlar till TLS via STARTTLS innan inloggningsuppgifterna överförs |
| `Credentials = $cache` | Överlämnar CredentialCache i stället för en enkel NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Kodar texten som UTF-8 så att specialtecken kommer fram korrekt |

</details>

`EnableSsl` är obligatoriskt här: `AUTH LOGIN` överför användarnamn och lösenord endast Base64-kodat, alltså i klartext utan TLS. Lösningen är en ändring per skript, men ändrar inget i systemkonfigurationen och behåller uppdateringens NTLM-skydd intakt.

## Lösning 2: Stäng av NTLM på e-postservern

Om du själv driver Postfix-servern kan du lösa problemet centralt: ta bort `ntlm` från `auth_mechanisms` i Dovecot-konfigurationen (vanligen `/etc/dovecot/conf.d/10-auth.conf`), så att endast `plain login` återstår, och ladda om Dovecot:

```bash
sudo systemctl reload dovecot
```

För Cyrus SASL begränsar `mech_list: plain login` i filen `smtpd.conf` i SASL-konfigurationskatalogen metoderna på motsvarande sätt. Därefter annonserar Postfix inte längre NTLM, och `Send-MailMessage` återgår till `LOGIN` utan skriptändringar. Kontrollera först om andra klienter är beroende av NTLM, till exempel äldre Outlook-profiler med alternativet ”Säker lösenordsautentisering (SPA)”. För Submission via TLS ger NTLM ingen säkerhetsvinst jämfört med `PLAIN` och `LOGIN`.

## Lösning 3: Registret, endast tillfälligt

KB5135557 dokumenterar ett registervärde som gör att klienten inte längre erbjuder signering:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Nyckel för NTLM-autentiseringspaketet |
| `/v ClientAllowUnsignedNtlm` | Värdets namn; `0` (standard) erbjuder signering, `1` återställer det gamla beteendet |
| `/t REG_DWORD` | Värdets datatyp |
| `/d 1` | Sätter värdet till 1 |
| `/f` | Skriver över utan bekräftelse |

</details>

Enligt Microsoft krävs ingen omstart. Värdet gäller dock alla NTLM-anslutningar på servern, inte bara e-postutskick, och upphäver därmed uppdateringens reläskydd för hela systemet. Microsoft beskriver det som en ren kompatibilitetsåtgärd och förbehåller sig rätten att senare ta bort registervärdena. Det överbryggar högst tiden tills lösning 1 eller 2 har genomförts. Att avinstallera KB5131085 är inget alternativ eftersom alla säkerhetskorrigeringar från november månads patchdag då saknas.

## På medellång sikt: Ersätt Send-MailMessage

Microsoft har i åratal markerat `Send-MailMessage` som föråldrat och avråder från användning eftersom cmdleten inte garanterar säkra anslutningar. Även för `SmtpClient` rekommenderar .NET-dokumentationen andra bibliotek, till exempel MailKit, för ny utveckling. För skript som skickar e-post via Exchange Online är Microsoft Graph API med `Send-MgUserMail` och appregistrering den stödda vägen; installationen beskrivs i artikeln [Åtkomst till Exchange Online med PowerShell: ersätt EWS med Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). Senast med fas 2 i maj 2027 är det värt att inventera vilka skript och tjänster som fortfarande använder NTLM: granskningshändelserna 4022 och 4023 ger listan.

## Källor

1.  [Windows Server 2019: KB5131085 stör e-postutskick via PowerShell (Borns IT- och Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): Läsarrapport från den 17 november 2026 med symptom, avinstallationstest, CredentialCache-skript och hänvisningen till NTLM-signering i kommentarerna.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): Ändringslogg för novemberuppdateringen för Windows Server 2019 och Windows 10 Enterprise LTSC 2019 med NTLM-posten och Known Issue för fel 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): Flaggor, tidsplanen för de två faserna, granskningshändelserna 4022/4023 och registervärdena ClientAllowUnsignedNtlm och ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): Registreringsordning för SMTP-inloggningsmodulerna Negotiate, NTLM, Digest och Login i SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Information om att cmdleten är föråldrad och inte garanterar säkra anslutningar.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Rekommendation att använda MailKit eller andra bibliotek för ny utveckling.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): SMTP-autentisering i Postfix via Cyrus SASL eller Dovecot, inklusive begränsning av de erbjudna metoderna.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): Aktuell lista över SASL-metoder i Dovecot, där NTLM inte längre ingår.
