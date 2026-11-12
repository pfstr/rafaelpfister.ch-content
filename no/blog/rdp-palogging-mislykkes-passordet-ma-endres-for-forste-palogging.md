---
title: "RDP-pålogging mislykkes: Passordet må endres før første pålogging"
navTitle: "RDP: Endre passord"
description: "En ny konto med et initialpassord, og tilkoblingen til eksternt skrivebord avbrytes med «You must change your password before logging on the first time». Årsaken er Network Level Authentication, som ikke støtter passordbytte. Passordet kan likevel endres: via en annen Windows-økt, fra egen PC, med PowerShell eller via RD Web Access."
date: "2026-11-11"
kategorie: "Windows-klient"
timeToRead: "6 min lesetid"
themen:
  - windows-client
  - active-directory-entra
produkte:
  - "windows-client"
  - "entra"
protokolle:
  - "troubleshooting"
  - "powershell"
slug: "rdp-palogging-mislykkes-passordet-ma-endres-for-forste-palogging"
translationId: "article-96f19bf85c2f7ccd"
draft: false
translationOf: rdp-kennwort-aendern-nla
url: https://rafaelpfister.ch/no/blog/rdp-palogging-mislykkes-passordet-ma-endres-for-forste-palogging
translationSourceHash: 016256f24e5c0c258d7cbf225d3f62ac577cf37c42844ad4a5f5084f85e7e8bf
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:33:14.557Z
translationReview: automatic
---

# RDP-pålogging mislykkes: Passordet må endres før første pålogging

I en kundes miljø med en nyopprettet konto har jeg igjen opplevd situasjonen: Initialpassordet er riktig, men tilkoblingen til eksternt skrivebord kobles likevel umiddelbart fra. Windows-klienten `mstsc` viser følgende melding:

```text
You must change your password before logging on the first time.
Please update your password or contact your system administrator
or technical support.
```

Andre RDP-klienter og tilkoblingsbehandlere viser samme tilstand med en annen formulering, for eksempel «The user password must be changed before logging on for the first time». Ved et utløpt passord lyder meldingen «This user account's password has expired. The password must change in order to logon». Årsak og løsning er de samme i begge tilfeller.

Kort oppsummert: Passordet kan ikke endres via RDP-tilkoblingen i denne situasjonen. Det må endres på en annen måte på forhånd, og i de fleste tilfeller er nettverkstilkobling til en Domain Controller tilstrekkelig.

## Årsak: NLA autentiserer før økten opprettes

For kontoen er alternativet **Brukeren må endre passord ved neste pålogging** angitt i Active Directory (teknisk: `pwdLastSet` = 0). På en arbeidsstasjon i domenet er dette uproblematisk: Påloggingsskjermen gjenkjenner tilstanden og ber direkte om det nye passordet.

Med RDP mangler dette trinnet så snart Network Level Authentication (NLA) er aktiv. NLA autentiserer brukeren via CredSSP før det opprettes en økt med påloggingsskjerm på målsystemet. Autentisering med et passord som må endres, mislykkes, og det finnes ingen dialog for passordbytte på dette tidspunktet. Klienten kobler fra og viser bare meldingen. NLA er standardinnstillingen i nåværende Windows-versjoner og håndheves med gruppepolicy i herdede miljøer.

## Muligheter for brukeren

Det finnes fire måter å endre passordet på uten den berørte RDP-tilkoblingen. Hvilken som er aktuell, avhenger av hva som er tilgjengelig fra egen arbeidsstasjon.

| Mulighet | Forutsetning |
|---|---|
| 1. Passordportal eller servicedesk | tilbys av driftsansvarlig |
| 2. Eksisterende Windows-økt i domenet | tilgang med en annen konto eller til et annet system |
| 3. Passorddialog på egen PC | Domain Controller tilgjengelig, for eksempel via VPN |
| 4. PowerShell med `NetUserChangePassword` | TCP 445 til Domain Controller tilgjengelig |

### Mulighet 1: Passordportal eller servicedesk

I større miljøer finnes det vanligvis en planlagt løsning for eksterne medarbeidere: en selvbetjeningsportal, en passordfunksjon i VPN- eller Citrix-portalen eller et PAM-system. Hvis ingen av disse finnes, kan servicedesk fjerne flagget eller angi et passord uten krav om endring (se avsnittet for administratorer).

### Mulighet 2: Eksisterende Windows-økt i domenet

Hvis det allerede finnes en økt på et system i domenet, for eksempel på en jumphost eller med en annen konto, kan passordet endres der. I en RDP-økt åpner `Ctrl+Alt+End` sikkerhetsskjermen for det eksterne systemet (`Ctrl+Alt+Del` ville bli behandlet på den lokale PC-en). Under **Endre passord** kan feltet med brukernavnet redigeres: Angi den berørte kontoen der i formen `DOMAIN\benutzer`, deretter initialpassordet og det nye passordet to ganger.

### Mulighet 3: Passorddialog på egen PC

Den samme dialogen fungerer også på en PC som ikke er medlem av domenet, forutsatt at den når en Domain Controller, for eksempel via en VPN-tilkobling. Trykk `Ctrl+Alt+Del`, velg **Endre passord**, og erstatt det forhåndsutfylte lokale brukernavnet med `DOMAIN\benutzer`. PC-en må kunne slå opp domenenavnet via DNS; uten DNS-serverne i målmiljøet finner ikke Windows noen Domain Controller.

### Mulighet 4: PowerShell med NetUserChangePassword

Windows-API-et `NetUserChangePassword` endrer et passord direkte på en Domain Controller. Det kontrollerer det tidligere passordet under selve endringen og forutsetter ingen tidligere pålogging. Derfor fungerer det også når passordet er utløpt eller må endres, og også fra en PC utenfor domenet. Skriptet binder inn funksjonen via P/Invoke og ber om begge passordene skjult:

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
<summary>Forklaring av alternativer</summary>

| Argument | Virkning |
|---|---|
| `domainname` | DNS- eller NetBIOS-navn på en Domain Controller eller domenet. Fra en PC utenfor domenet angir du Domain Controller direkte (navn eller IP-adresse). |
| `username` | Påloggingsnavnet til kontoen (`sAMAccountName`), uten domenedelen. |
| `oldpassword` | Tidligere passord, her initialpassordet. Det må være nøyaktig riktig. |
| `newpassword` | Nytt passord. Det må oppfylle domenets passordpolicy. |

</details>

Returverdien 0 betyr at operasjonen lyktes. De vanligste feilkodene:

| Kode | Betydning |
|---|---|
| 5 | Tilgang nektet, for eksempel fordi Domain Controller ikke tillater kallet uten pålogging |
| 86 | Det tidligere passordet er feil |
| 2221 | Brukernavnet ble ikke funnet |
| 2245 | Det nye passordet oppfyller ikke passordpolicyen (lengde, kompleksitet, historikk, minimumsalder) |
| 2351 | Det angitte servernavnet er ugyldig |

Kallet går via SMB og krever TCP-port 445 til Domain Controller. I sterkt segmenterte nettverk er denne porten ofte blokkert fra klientnettverk eller via VPN; da gjenstår mulighet 1.

`Set-ADAccountPassword` fra Active Directory-modulen hjelper ikke den berørte brukeren i denne situasjonen: Cmdleten krever en autentisert tilkobling til Active Directory Web Services, og nettopp denne påloggingen er ikke mulig med den berørte kontoen. Det er administratorens verktøy.

## Deaktivere NLA i klienten: kun uten påtvunget NLA

Det anbefales ofte å deaktivere CredSSP i .rdp-filen:

```text
enablecredsspsupport:i:0
```

Klienten kobler da til uten NLA og får den klassiske påloggingsskjermen på målsystemet, der passordbytte er mulig. Dette lykkes bare hvis målsystemet godtar tilkoblinger uten NLA. Hvis **Tillat bare tilkoblinger fra datamaskiner som kjører Eksternt skrivebord med godkjenning på nettverksnivå** er aktiv der, avviser det tilkoblingen. Dette er standardinnstillingen i nåværende Windows Server-versjoner.

## Muligheter for administratorer

På administrasjonssiden kan problemet unngås eller løses:

- **Fjern flagget:** `Set-ADUser -Identity m.muster -ChangePasswordAtLogon $false` fjerner kravet om endring. Brukeren kan logge på med initialpassordet og endre det i økten via `Ctrl+Alt+End`. Initialpassordet forblir gyldig frem til da og må derfor overføres via en sikker kanal.
- **Tilbakestill passord:** `Set-ADAccountPassword -Identity m.muster -Reset` angir et nytt passord. Kontroller deretter om flagget fortsatt er satt.
- **RD Web Access:** Hvis rollen finnes, kan den tilby passordbytte for utløpte passord. I IIS Manager under **Sites → Default Web Site → RDWeb → Pages** setter du verdien `PasswordChangeEnabled` til `true` i **Application Settings**. Siden for passordbytte ligger på `/RDWeb/Pages/en-US/password.aspx`.
- **Kontoer for kun RDP-tilgang:** For brukere som arbeider utelukkende via RDP (eksterne tjenesteleverandører, administratorer på jumphoster), kan kravet om endring ved første pålogging ikke oppfylles uten en av metodene ovenfor. Fremgangsmåten for første passordbytte bør derfor være en del av onboardingen.

Å deaktivere NLA på målsystemet løser også symptomet, men fjerner et beskyttelseslag: Uten NLA når enhver klient i nettverket påloggingsskjermen og dermed RDP-tjenesten før den er autentisert. Microsoft fraråder dette.

## Kilder

1.  [Remote Desktop Client Troubleshooting: Disconnect Codes and Reasons](https://learn.microsoft.com/en-us/archive/technet-wiki/37870.remote-desktop-client-troubleshooting-disconnect-codes-and-reasons): Liste over frakoblingskoder for RDP-klienten, inkludert 4615 (passordet må endres før første pålogging) og 3847 (passord utløpt), med meldingenes ordlyd.

2.  [NetUserChangePassword function (lmaccess.h)](https://learn.microsoft.com/en-us/windows/win32/api/lmaccess/nf-lmaccess-netuserchangepassword): API-referanse med parametere, tillatelser og returverdier.

3.  [How to change your own expired password when you can't login to RDP](https://evotec.xyz/how-to-change-your-own-expired-password-when-you-cant-login-to-rdp/): Praktisk rapport fra Evotec om passordbytte via NetUserChangePassword fra en PC utenfor domenet, inkludert informasjon om TCP 445 og om den virkningsløse enablecredsspsupport-oppføringen ved påtvunget NLA.

4.  [Supported RDP properties](https://learn.microsoft.com/en-us/azure/virtual-desktop/rdp-properties): Referanse for .rdp-egenskaper, inkludert enablecredsspsupport med verdier og standardverdi.

5.  [Remote Desktop Web Access troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-web-access-troubleshooting): Avsnitt om passordbytte for utløpte passord via RD Web Access og innstillingen PasswordChangeEnabled.

6.  [Set-ADAccountPassword](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adaccountpassword): Cmdlet-referanse med parameterne Reset, OldPassword, NewPassword, Credential og Server.

7.  [User can't authenticate or must authenticate twice](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice): Microsoft-informasjon om påloggingsfeil med NLA; deaktivering av NLA er der merket som ikke anbefalt.

8.  [Change password in another domain](https://bartwullems.blogspot.com/2019/12/change-password-in-another-domain.html): Blogginnlegg om det redigerbare brukerfeltet i dialogen «Endre passord», som kan brukes til å endre passordet for en konto i et annet domene.

9.  [Allow users to reset expired password via RD Web Access](https://woshub.com/allow-users-to-reset-expired-password-via-rd-webaccess-windows-server-2012/): Veiledning fra Windows OS Hub om innstillingen PasswordChangeEnabled og banen til siden password.aspx.
