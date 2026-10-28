---
title: "Förutsättningar för att fjärr-PowerShell ska fungera"
navTitle: "Fjärr-PowerShell"
description: "PowerShell-fjärrkommunikation misslyckas sällan på grund av kommandot, utan på grund av förutsättningarna: WinRM-tjänst, lyssnare, brandvägg, autentisering och särdragen hos lokala konton. Vad som måste konfigureras på mål- och klientsidan, hur du kontrollerar det med Test-WSMan och varför Access denied oftast inte har med lösenordet att göra."
date: "2026-09-01"
kategorie: "Windows och PowerShell"
timeToRead: "10 min lästid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "forutsattningar-for-att-powershell-fjarrkommunikation-ska-fungera"
translationId: "article-7315c1ae9e67a24d"
translationOf: remote-powershell-voraussetzungen
translationSourceHash: edebb6e6be909c38c0d70f1bf09aaac37328f86bf2def4e23a26a34456b3492f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:58:09.194Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/forutsattningar-for-att-powershell-fjarrkommunikation-ska-fungera
---

# Förutsättningar för att fjärr-PowerShell ska fungera

`Invoke-Command` och `Enter-PSSession` skrivs snabbt, men anslutningen fungerar först när förutsättningarna är uppfyllda på båda sidor. PowerShell-fjärrkommunikation bygger på WS-Management (WinRM), en SOAP-baserad hanteringstjänst över HTTP. Om en session misslyckas beror det nästan aldrig på själva cmdleten, utan på en saknad tjänst, en stängd port, en brandväggsregel eller autentiseringen. Den här artikeln går igenom förutsättningarna i tur och ordning och visar hur du kontrollerar var och en.

Först begreppen: Måldatorn är datorn där kommandona ska köras; klienten är datorn som du ansluter från. WinRM lyssnar som standard på port 5985 (HTTP) och, om det är konfigurerat, på port 5986 (HTTPS). HTTP-trafiken på 5985 krypteras på meddelandenivå när autentiseringen sker via Kerberos eller NTLM.

## Översikt över cmdlets

Här är cmdletarna som förekommer i den här artikeln:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Cmdlet | Syfte |
|---|---|
| `Enable-PSRemoting` | Konfigurerar WinRM på målsidan: tjänst, lyssnare, brandväggsregel |
| `Test-WSMan` | Kontrollerar om WinRM-tjänsten på motparten svarar |
| `Enter-PSSession` | Öppnar en interaktiv fjärrsession till en dator |
| `Invoke-Command` | Kör ett kommando-block på en eller flera datorer |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Lägger till betrodda motparter för autentisering utanför en domän |
| `Get-Service WinRM` | Visar status och starttyp för WinRM-tjänsten |

</details>

## Målsidan: konfigurera WinRM

På måldatorn konfigurerar ett enda kommando tjänsten, lyssnaren och brandväggsregeln. Kör det i PowerShell med administratörsbehörighet:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Force` | Kör utan frågor |
| `-SkipNetworkProfileCheck` | Konfigurerar även fjärrkommunikation när en nätverksanslutning klassas som offentlig |

</details>

`Enable-PSRemoting` startar WinRM-tjänsten, ställer in dess starttyp till automatisk, skapar en HTTP-lyssnare och lägger till rätt brandväggsregel. Det finns ett förbehåll för nätverksprofilen: Om ett nätverkskort klassas som offentligt vägrar kommandot som standard att utföra konfigurationen. På servrar eller i kontrollerade nätverk hjälper `-SkipNetworkProfileCheck` så att konfigurationen ändå slutförs.

Brandväggsregelns omfång är viktigt. För offentliga nätverksprofiler begränsar standardregeln åtkomsten till det lokala undernätet. Om du ansluter via ett annat nätverk, exempelvis ett VPN, gäller denna begränsning och anslutningen misslyckas trots att tjänsten körs. Öppna då regeln specifikt för det behövda adressintervallet, inte generellt för alla adresser:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Väljer de WinRM HTTP-regler som skapats av Enable-PSRemoting via namnmatchningen |
| `-RemoteAddress <Bereich>` | Begränsar tillåtna källadresser till det angivna intervallet (här ett CIDR-block); `Any` tillåter alla adresser |

</details>

## Klientsidan: TrustedHosts och tjänsten

På klienten måste WinRM-tjänsten vara igång, annars misslyckas redan inställningarna. Kontrollera detta först:

```powershell
Get-Service WinRM
```

Om tjänsten har statusen Stopped startar du den med `Start-Service WinRM` (administratörsbehörighet krävs). Starttypen är ofta manuell på klienter, vilket innebär att tjänsten stoppas igen efter en omstart. Om du regelbundet ansluter från den här datorn bör du ställa in starttypen på automatisk.

Den andra punkten gäller autentisering utanför en domän. Om du ansluter via IP-adress eller i en arbetsgrupp kan klienten inte kontrollera motparten via Kerberos och faller tillbaka på NTLM. Av säkerhetsskäl vägrar WinRM detta så länge motparten inte har lagts till som betrodd. Lägg till måladressen i TrustedHosts (administratörsbehörighet krävs):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Value <Liste>` | Betrodda motparter (IP eller namn), flera separerade med kommatecken, `*` som jokertecken |
| `-Force` | Anger värdet utan fråga |
| `-Concatenate` | Lägger till i den befintliga listan i stället för att ersätta den |

</details>

TrustedHosts är en klientinställning, inte en inställning på måldatorn, och påverkar klientens säkerhet: De angivna motparterna betraktas som betrodda utan att deras identitet kontrolleras kryptografiskt. Lägg därför in konkreta adresser och inte jokertecknet `*`. I en domän med Kerberos behövs inte posten; den korrekta lösningen utanför en domän utan TrustedHosts är en HTTPS-lyssnare med ett certifikat som klienten litar på.

## Autentisering: varför Access denied sällan beror på lösenordet

Ett vanligt fel för lokala konton är meddelandet Access denied trots att lösenordet är korrekt. Orsaken är Remote UAC-filtrering: För lokala konton (inte det inbyggda administratörskontot) tar Windows som standard bort administrativa rättigheter vid åtkomst över nätverket. Inloggningen lyckas, men varje åtgärd med förhöjda rättigheter avvisas. Om motparten rapporterar Access denied i stället för felaktiga inloggningsuppgifter är det troligen orsaken.

Det kan åtgärdas på måldatorn med ett registervärde som ger lokala administratörer fullständiga rättigheter över nätverket:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Detta är en medveten lättnad av säkerheten: Lokala administratörskonton får därmed fullständiga rättigheter över nätverket. Ange värdet endast i kontrollerade nätverk och med starka lösenord. I en domän är det bättre att använda ett domänkonto, då uppstår inte frågan.

När du ansluter anger du användarnamnet för lokala konton med datornamnet före, så att målsystemet löser kontot lokalt:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

I inloggningsdialogrutan anger du användaren som `RECHNERNAME\Benutzer` och för domänkonton som `DOMAENE\Benutzer`. En PIN-kod från Windows-inloggningen fungerar inte över nätverket; kontots lösenord krävs. För ett Microsoft-konto är det dess lösenord, och kontonamnet kan skilja sig från visningsnamnet.

## Kontrollera i rätt ordning

Avgränsa fel nedifrån och upp, så ser du snabbt vilken förutsättning som saknas.

Börja med att kontrollera att porten går att nå:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Om porten inte svarar saknas lyssnaren eller brandväggen blockerar. Om den svarar kontrollerar du WinRM-tjänsten hos motparten:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Ett svar med protokollversion och tillverkare betyder att tjänsten och lyssnaren är igång. Först därefter testar du med inloggningsuppgifter:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Om anropet returnerar datornamnet för motparten är alla förutsättningar uppfyllda.

## Vanliga fel och deras orsak

| Meddelande eller symptom | Trolig orsak | Åtgärd |
|---|---|---|
| Port 5985 kan inte nås | Ingen lyssnare eller brandväggen blockerar | `Enable-PSRemoting`, kontrollera brandväggsregeln och dess omfång |
| WinRM cannot complete the operation | Tjänsten på målsidan är avstängd, eller åtkomst tillåts bara från det lokala undernätet | Starta tjänsten, öppna brandväggsregeln för det behövda adressintervallet |
| The WinRM client cannot process the request … TrustedHosts | Anslutning utanför domänen utan TrustedHosts-post | Lägg till måladressen i TrustedHosts på klienten eller använd HTTPS |
| Access is denied (trots korrekt lösenord) | Remote UAC-filtrering för lokalt konto | Ställ in `LocalAccountTokenFilterPolicy` på 1, eller använd ett domänkonto |
| Åtkomst till en andra resurs misslyckas i sessionen | Double hop: inloggningsuppgifter vidarebefordras inte | Kör uppgiften direkt på målet eller använd CredSSP respektive delegerade inloggningsuppgifter |

## Begränsningar: Double hop-problemet

En begränsning kvarstår även med en fullständig konfiguration och kan endast kringgås: Som standard kan en fjärrsession inte vidarebefordra dina inloggningsuppgifter till ett tredje system. Om du i en session på måldatorn öppnar en nätverksresurs eller ansluter till en ytterligare server misslyckas detta på grund av saknade inloggningsuppgifter. Detta Double hop-problem är en säkerhetsfunktion, inte en felkonfiguration. För de flesta supportuppgifter räcker det att köra kommandot direkt på måldatorn. Där vidarebefordran verkligen behövs kan CredSSP eller begränsad delegering användas, båda med egna säkerhetsavvägningar.

## Ändringshistorik

- **25 oktober 2026:** Källänken uppdaterad till den nya adressen på Microsoft Learn.

## Källor

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Förutsättningar för PowerShell-fjärrkommunikation, behörigheter och nätverksprofiler.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): Vad kommandot konfigurerar, inklusive förbehåll för nätverksprofil och brandväggsregel.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autentisering utanför domänen och vanliga felmeddelanden.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/ps-remoting-second-hop): Orsaken till Double hop-problemet och lösningsmetoderna med deras avvägningar.
