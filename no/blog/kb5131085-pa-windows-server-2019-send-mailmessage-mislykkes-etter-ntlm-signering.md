---
title: "KB5131085 på Windows Server 2019: Send-MailMessage mislykkes etter NTLM-signering"
navTitle: "KB5131085 e-postsending"
description: "Etter novemberoppdateringen KB5131085 mislykkes e-postsending via Send-MailMessage mot Postfix-servere på Windows Server 2019. Årsaken anses å være den nye NTLM-signeringen fra KB5135557. Artikkelen forklarer hvorfor en SMTP-klient i det hele tatt bruker NTLM, hvordan du undersøker tilfellet og hvilke løsninger som finnes: tving AUTH LOGIN, deaktiver NTLM på e-postserveren eller tilbakestill midlertidig via registeret."
date: "2026-11-17"
kategorie: "Windows og SMTP"
timeToRead: "7 min. lesetid"
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
slug: "kb5131085-pa-windows-server-2019-send-mailmessage-mislykkes-etter-ntlm-signering"
translationId: "article-ca3e77cefa049fca"
translationOf: kb5131085-send-mailmessage-ntlm
url: https://rafaelpfister.ch/no/blog/kb5131085-pa-windows-server-2019-send-mailmessage-mislykkes-etter-ntlm-signering
translationSourceHash: 6416155fbbec934196848b06f3468ff46dbca040310b92cf8fd3171db1858227
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:21:05.470Z
translationReview: automatic
---

# KB5131085 på Windows Server 2019: Send-MailMessage mislykkes etter NTLM-signering

Siden patchdagen 10. november 2026 har administratorer meldt at skript på Windows Server 2019 ikke lenger sender e-post. Dette gjelder sikkerhetskopirapporter, overvåkingsmeldinger og lignende oppgaver som i årevis har sendt til en Postfix-server med PowerShell-cmdleten `Send-MailMessage`. Etter avinstallering av den kumulative oppdateringen KB5131085 (OS Build 17763.9487) fungerer sendingen igjen. Günter Born beskrev tilfellet 17. november 2026 på Borns IT- und Windows-Blog, utløst av meldingen fra en leser.

Microsoft oppfører foreløpig ikke problemet som et Known Issue. Den eneste Known Issue-oppføringen for KB5131085 gjelder aktiveringskommandoer med feilen `0x80041008`. Årsaken er derfor ikke offisielt bekreftet, men den kan utledes godt fra endringene i oppdateringen og oppførselen til .NET-klassen `SmtpClient`.

## Hva KB5131085 endrer i NTLM

Oppdateringen inneholder beskyttelsestiltak mot NTLM-reléangrep, som Microsoft beskriver i KB5135557 («NTLM signing security improvements in Windows"). De gjelder for Windows 11, Windows 10 Enterprise LTSC 2021 og Windows Server 2016 til 2025. I endringsloggen for KB5131085 står det at klienter fra nå av som standard tilbyr NTLM Message Signing for alle ikke-anonyme tilkoblinger.

Teknisk betyr det: En Windows-klient setter nå flagget `NTLMSSP_NEGOTIATE_SIGN` (`0x00000010`) i NTLM-forhandlingen, samt nødvendig `NTLMSSP_NEGOTIATE_KEY_EXCH` (`0x40000000`), selv om programmet ikke har bedt om integritet. Tidligere fikk slike programmer bare `NTLMSSP_NEGOTIATE_ALWAYS_SIGN`, en plassholder uten kryptografisk beskyttelse. Microsoft ruller ut endringen i to faser:

| Fase | Dato | Atferd |
|---|---|---|
| 1 | 10. november 2026 | Klienter tilbyr signering på alle tilkoblinger. Servere godtar signerte og usignerte klienter og logger usignerte. |
| 2 | Mai 2027 | Windows-servere krever signering som standard og avviser klienter uten signering med `SEC_E_UNSUPPORTED_FUNCTION` (`0x80090302`). |

Ifølge Microsoft faller en Windows-klient tilbake til den gamle atferden uten feil dersom serveren ikke bekrefter signering. Alle Windows-servere gjør det. Motparter med en egen NTLM-implementering, som en Linux-e-postserver, er ikke tatt med i denne uttalelsen.

## Hvorfor en SMTP-klient bruker NTLM

`Send-MailMessage` i Windows PowerShell 5.1 bruker .NET-klassen `System.Net.Mail.SmtpClient` internt. Den kontrollerer `AUTH`-linjen i serverens EHLO-svar og prøver metodene i en fast rekkefølge: først `GSSAPI` (Negotiate), deretter `NTLM`, så `WDigest`, og til slutt `LOGIN`. Dersom serveren tilbyr NTLM, velger `SmtpClient` NTLM, selv når brukernavn og passord overleveres med `-Credential`.

Postfix implementerer ikke SMTP-autentisering selv, men delegerer den til Cyrus SASL eller Dovecot. Begge kan tilby NTLM: Cyrus SASL via en egen plugin-modul, Dovecot frem til versjon 2.3 via `ntlm` i `auth_mechanisms` (Dovecot 2.4 har ikke lenger NTLM). Dersom NTLM er aktivt, annonserer Postfix for eksempel `AUTH PLAIN LOGIN NTLM`, og hver Windows-klient autentiserer seg med NTLM uten at noen merker det. Med KB5131085 tilbyr klienten i denne forhandlingen i tillegg signering og nøkkelutveksling. Hvis e-postserverens NTLM-implementering ikke kan håndtere dette, mislykkes påloggingen, og `Send-MailMessage` avbrytes. En kommentator i Born-artikkelen peker på NTLM-signering som årsak, og funnet passer med dette: Leserens løsning fungerer fordi den utelukker NTLM.

## Undersøk tilfellet

Avklar først om e-postserveren tilbyr NTLM. Fra en Linux-maskin viser EHLO-svaret etter STARTTLS de tilbudte metodene:

```bash
openssl s_client -starttls smtp -crlf -quiet \
  -connect mail.example.com:587
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-starttls smtp` | Oppretter forbindelsen i klartekst og bytter til TLS via STARTTLS |
| `-crlf` | Sender linjeslutt som CRLF, slik SMTP krever |
| `-quiet` | Undertrykker sertifikat- og øktdetaljer |
| `-connect mail.example.com:587` | Målvert og innsendingport |

</details>

Skriv inn `EHLO test` etter at forbindelsen er opprettet. Hvis linjen `250-AUTH` inneholder oppføringen `NTLM`, er serveren berørt. På selve Postfix-serveren viser to kommandoer hvilken SASL-backend som er aktiv, og hvilke metoder Dovecot tilbyr:

```bash
postconf smtpd_sasl_type
doveconf auth_mechanisms
```

På Windows-serveren logger NTLM-loggen fra og med Windows Server 2019 tilkoblinger med de forhandlede flaggene. Hendelse 4022 er klientoppføringen:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-NTLM/Operational' -MaxEvents 50 |
  Where-Object { $_.Id -in 4022, 4023 } |
  Format-List TimeCreated, Id, Message
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `-LogName 'Microsoft-Windows-NTLM/Operational'` | Leser den operative NTLM-loggen |
| `-MaxEvents 50` | Begrenser utdataene til de 50 nyeste oppføringene |
| `Where-Object { $_.Id -in 4022, 4023 }` | Filtrerer etter revisjonshendelsene for klient (4022) og server (4023) |
| `Format-List TimeCreated, Id, Message` | Viser tidspunkt, hendelses-ID og teksten med Negotiated Flags |

</details>

En oppføring på tidspunktet for sendeforsøket, med e-postserveren som mål, bekrefter at `SmtpClient` brukte NTLM.

## Løsning 1: Tving AUTH LOGIN i skriptet

`Send-MailMessage` har ingen parameter for å angi påloggingsmetode. Når klassen `SmtpClient` kalles direkte, kan den gjøre det: En `CredentialCache` leverer påloggingsdataene kun for en bestemt metode. Hvis bare `Login` er lagret, hopper `SmtpClient` over NTLM og bruker `AUTH LOGIN`. Følgende skript er basert på eksempelet fra leseren i Born-artikkelen:

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
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `Import-Clixml -Path …` | Laster en PSCredential som tidligere er lagret med `Get-Credential \| Export-Clixml`; bundet til bruker og maskin via DPAPI |
| `$cache.Add($server, $port, 'Login', …)` | Lagrer påloggingsdataene kun for vert, port og metoden `Login` |
| `SmtpClient($server, $port)` | Oppretter SMTP-klienten for server og innsendingport |
| `EnableSsl = $true` | Bytter til TLS via STARTTLS før påloggingsdataene overføres |
| `Credentials = $cache` | Overleverer CredentialCache i stedet for en enkel NetworkCredential |
| `BodyEncoding = [System.Text.Encoding]::UTF8` | Koder teksten som UTF-8 slik at omlyder kommer riktig frem |

</details>

`EnableSsl` er obligatorisk her: `AUTH LOGIN` overfører brukernavn og passord bare Base64-kodet, altså i klartekst uten TLS. Løsningen er et inngrep per skript, men endrer ingenting i systemkonfigurasjonen og lar NTLM-beskyttelsen fra oppdateringen være intakt.

## Løsning 2: Deaktiver NTLM på e-postserveren

Hvis du selv drifter Postfix-serveren, kan du løse problemet sentralt: Fjern `ntlm` fra `auth_mechanisms` i Dovecot-konfigurasjonen (vanligvis `/etc/dovecot/conf.d/10-auth.conf`), slik at bare `plain login` gjenstår, og last Dovecot på nytt:

```bash
sudo systemctl reload dovecot
```

For Cyrus SASL begrenser `mech_list: plain login` i filen `smtpd.conf` i SASL-konfigurasjonskatalogen metodene tilsvarende. Deretter annonserer Postfix ikke lenger NTLM, og `Send-MailMessage` faller tilbake til `LOGIN` uten endring i skriptet. Kontroller først om andre klienter er avhengige av NTLM, for eksempel eldre Outlook-profiler med alternativet «Sikker passordgodkjenning (SPA)». For innsending via TLS gir NTLM ingen sikkerhetsfordel sammenlignet med `PLAIN` og `LOGIN`.

## Løsning 3: Registeret, kun midlertidig

KB5135557 dokumenterer en registerverdi som gjør at klienten ikke lenger tilbyr signering:

```powershell
reg add 'HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0' /v ClientAllowUnsignedNtlm /t REG_DWORD /d 1 /f
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `HKLM\…\Lsa\MSV1_0` | Nøkkel for NTLM-godkjenningspakken |
| `/v ClientAllowUnsignedNtlm` | Navn på verdien; `0` (standard) tilbyr signering, `1` gjenoppretter den gamle atferden |
| `/t REG_DWORD` | Verditype |
| `/d 1` | Setter verdien til 1 |
| `/f` | Overskriver uten spørsmål |

</details>

Ifølge Microsoft er omstart ikke nødvendig. Verdien gjelder imidlertid for alle NTLM-tilkoblinger på serveren, ikke bare e-postsending, og opphever dermed oppdateringens relébeskyttelse for hele systemet. Microsoft beskriver den som et rent kompatibilitetstiltak og forbeholder seg retten til å fjerne registerverdiene senere. Den kan høyst bygge bro frem til løsning 1 eller 2 er gjennomført. Å avinstallere KB5131085 er ikke et alternativ, fordi alle sikkerhetsrettingene fra november-patchdagen da mangler.

## På mellomlang sikt: Erstatt Send-MailMessage

Microsoft har oppført `Send-MailMessage` som foreldet i mange år og fraråder bruk fordi cmdleten ikke garanterer sikre tilkoblinger. Også for `SmtpClient` anbefaler .NET-dokumentasjonen andre biblioteker som MailKit for nyutvikling. For skript som sender e-post via Exchange Online, er Microsoft Graph API med `Send-MgUserMail` og appregistrering den støttede løsningen; oppsettet er beskrevet i artikkelen [Få tilgang til Exchange Online med PowerShell: Erstatt EWS med Microsoft Graph](/blog/microsoft-graph-powershell-postfach-anbindung). Senest med fase 2 i mai 2027 er det verdt å kartlegge hvilke skript og tjenester som fortsatt bruker NTLM: Revisjonshendelsene 4022 og 4023 gir listen for dette.

## Kilder

1.  [Windows Server 2019: KB5131085 forstyrrer e-postsending med PowerShell (Borns IT- und Windows-Blog)](https://borncity.com/blog/2026/11/17/windows-server-2019-kb5131085-stoert-mail-versand-per-powershell/): Leserinnlegg fra 17. november 2026 med symptom, avinstallasjonstest, CredentialCache-skript og henvisning til NTLM-signering i kommentarene.

2.  [KB5131085 (OS Build 17763.9487)](https://support.microsoft.com/help/5131085): Endringslogg for novemberoppdateringen for Windows Server 2019 og Windows 10 Enterprise LTSC 2019 med NTLM-oppføringen og Known Issue om feil 0x80041008.

3.  [KB5135557: NTLM signing security improvements in Windows](https://support.microsoft.com/en-us/help/5135557): Flagg, tidsplanen for de to fasene, revisjonshendelsene 4022/4023 og registerverdiene ClientAllowUnsignedNtlm og ServerRequireNtlmSign.

4.  [SmtpAuthenticationManager.cs (.NET Framework Reference Source)](https://github.com/microsoft/referencesource/blob/master/System/net/System/Net/mail/SmtpAuthenticationManager.cs): Registreringsrekkefølgen for SMTP-påloggingsmodulene Negotiate, NTLM, Digest og Login i SmtpClient.

5.  [Send-MailMessage (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/send-mailmessage): Merknad om at cmdleten er foreldet og ikke garanterer sikre tilkoblinger.

6.  [SmtpClient Class (Microsoft Learn)](https://learn.microsoft.com/en-us/dotnet/api/system.net.mail.smtpclient): Anbefaling om å bruke MailKit eller andre biblioteker for nyutvikling.

7.  [Postfix SASL Howto](https://www.postfix.org/SASL_README.html): SMTP-autentisering i Postfix via Cyrus SASL eller Dovecot, inkludert begrensning av tilbudte metoder.

8.  [Dovecot: Authentication Mechanisms Overview](https://doc.dovecot.org/main/core/config/auth/mechanisms/overview.html): Aktuell liste over SASL-metodene i Dovecot, der NTLM ikke lenger er inkludert.
