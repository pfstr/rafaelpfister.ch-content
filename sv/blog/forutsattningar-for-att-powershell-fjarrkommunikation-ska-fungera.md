---
title: "Förutsättningar för att PowerShell-fjärrkommunikation ska fungera"
navTitle: "PowerShell-fjärrkommunikation"
description: "PowerShell-fjärrkommunikation misslyckas sällan på grund av kommandot, utan på grund av förutsättningarna: WinRM-tjänst, lyssnare, brandvägg, autentisering och särdragen hos lokala konton. Vad som måste konfigureras på mål- och klientsidan, hur du kontrollerar det med Test-WSMan och varför Access denied oftast inte har med lösenordet att göra."
date: "2026-09-01"
kategorie: "Windows och PowerShell"
timeToRead: "10 min. lästid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "forutsattningar-for-att-powershell-fjarrkommunikation-ska-fungera"
translationId: "article-7315c1ae9e67a24d"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/sv/blog/forutsattningar-for-att-powershell-fjarrkommunikation-ska-fungera
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:47:25.924Z
translationReview: automatic
---

# Förutsättningar för att PowerShell-fjärrkommunikation ska fungera

`Invoke-Command` och `Enter-PSSession` skrivs snabbt, men anslutningen upprättas först när förutsättningarna är uppfyllda på båda sidor. PowerShell-fjärrkommunikation bygger på WS-Management (WinRM), en SOAP-baserad hanteringstjänst över HTTP. När en session misslyckas beror det nästan aldrig på själva cmdleten, utan på en saknad tjänst, en stängd port, en brandväggsregel eller autentiseringen. Den här artikeln går igenom förutsättningarna i tur och ordning och visar hur du kontrollerar var och en.

Först begreppen: Måldatorn är datorn där kommandona ska köras; klienten är datorn som du ansluter från. WinRM lyssnar som standard på port 5985 (HTTP) och, om den har konfigurerats, på port 5986 (HTTPS). HTTP-trafiken på 5985 är krypterad på meddelandenivå så snart autentiseringen sker via Kerberos eller NTLM.

## Översikt över cmdleterna

Som orientering följer de cmdletar som förekommer i den här artikeln:

<details class="options-details">
<summary>Översikt över alternativ</summary>

| Cmdlet | Syfte |
|---|---|
| `Enable-PSRemoting` | Konfigurerar WinRM på målsidan: tjänst, lyssnare, brandväggsregel |
| `Test-WSMan` | Kontrollerar om WinRM-tjänsten på motparten svarar |
| `Enter-PSSession` | Öppnar en interaktiv fjärrsession till en dator |
| `Invoke-Command` | Kör ett kommandoblock på en eller flera datorer |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Lägger till betrodda motparter för autentisering utanför en domän |
| `Get-Service WinRM` | Visar WinRM-tjänstens status och starttyp |

</details>

## Målsidan: konfigurera WinRM

På måldatorn konfigurerar ett enda kommando tjänsten, lyssnaren och brandväggsregeln. Kör det i PowerShell med administratörsbehörighet:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Force` | Kör utan att fråga efter bekräftelse |
| `-SkipNetworkProfileCheck` | Konfigurerar även fjärrkommunikation när en nätverksanslutning klassificeras som offentlig |

</details>

`Enable-PSRemoting` startar WinRM-tjänsten, sätter dess starttyp till automatisk, skapar en HTTP-lyssnare och lägger till rätt brandväggsregel. Ett förbehåll gäller nätverksprofilen: Om ett nätverkskort klassificeras som offentligt vägrar kommandot som standard att konfigurera det. På servrar eller i kontrollerade nätverk hjälper `-SkipNetworkProfileCheck` så att konfigurationen ändå slutförs.

Det viktiga är brandväggsregelns omfattning. För offentliga nätverksprofiler begränsar standardregeln åtkomsten till det lokala subnätet. Om du ansluter via ett annat nät, till exempel ett VPN, gäller denna begränsning och anslutningen misslyckas trots att tjänsten körs. Öppna då regeln specifikt för det adressintervall som behövs, inte generellt för alla adresser:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Väljer de WinRM HTTP-regler som skapats av Enable-PSRemoting via namnmatchningen |
| `-RemoteAddress <Bereich>` | Begränsar tillåtna källadresser till det angivna intervallet (här ett CIDR-block); `Any` tillåter alla adresser |

</details>

## Klientsidan: TrustedHosts och tjänst

På klienten måste WinRM-tjänsten köras, annars misslyckas redan inställningen av konfigurationen. Kontrollera det först:

```powershell
Get-Service WinRM
```

Om tjänsten har status Stopped startar du den med `Start-Service WinRM` (administratörsbehörighet krävs). Starttypen är ofta manuell på klienter, vilket innebär att tjänsten åter är stoppad efter en omstart. Om du regelbundet ansluter från den här datorn, ställ in starttypen på automatisk.

Den andra punkten gäller autentisering utanför en domän. Om du ansluter via IP-adress eller i en arbetsgrupp kan klienten inte kontrollera motparten med Kerberos och faller tillbaka på NTLM. Av säkerhetsskäl vägrar WinRM detta så länge motparten inte har lagts till som betrodd. Lägg till måladressen i TrustedHosts (administratörsbehörighet krävs):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `-Value <Liste>` | Betrodda motparter (IP eller namn), flera separerade med kommatecken, `*` som jokertecken |
| `-Force` | Anger värdet utan att fråga efter bekräftelse |
| `-Concatenate` | Lägger till i den befintliga listan i stället för att ersätta den |

</details>

TrustedHosts är en inställning på klienten, inte på måldatorn, och påverkar klientens säkerhet: De registrerade motparterna betraktas som betrodda utan att deras identitet kontrolleras kryptografiskt. Ange därför specifika adresser och inte jokertecknet `*`. I en domän med Kerberos behövs inte posten; det korrekta sättet utanför en domän utan TrustedHosts är en HTTPS-lyssnare med ett certifikat som klienten litar på.

## Autentisering: varför Access denied sällan beror på lösenordet

Ett vanligt fel vid lokala konton är meddelandet Access denied trots att lösenordet är korrekt. Orsaken är Remote UAC-filtrering: För lokala konton (inte det inbyggda administratörskontot) tar Windows som standard bort de administrativa rättigheterna vid åtkomst över nätverket. Inloggningen lyckas, men varje åtgärd som kräver förhöjda rättigheter nekas. Om motparten rapporterar Access denied i stället för felaktiga inloggningsuppgifter är detta den sannolika orsaken.

Det kan åtgärdas på måldatorn med ett registervärde som ger lokala administratörer fullständiga rättigheter över nätverket:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Detta är en medveten lättnad av säkerheten: Lokala administratörskonton får därmed fullständiga rättigheter över nätverket. Ange värdet endast i kontrollerade nätverk och med starka lösenord. I en domän bör du hellre använda ett domänkonto, då uppstår inte frågan.

När du ansluter anger du användarnamnet för lokala konton med datornamnet framför, så att målsystemet löser kontot lokalt:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

I inloggningsdialogen anger du användaren som `RECHNERNAME\Benutzer` och domänkonton som `DOMAENE\Benutzer`. En PIN-kod från Windows-inloggningen fungerar inte över nätverket; kontots lösenord krävs. För ett Microsoft-konto är det dess lösenord, och kontonamnet kan skilja sig från visningsnamnet.

## Kontrollera i rätt ordning

Avgränsa fel nedifrån och upp, så ser du snabbt vilken förutsättning som saknas.

Kontrollera först att porten går att nå:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Om porten inte svarar saknas lyssnaren eller så blockeras den av brandväggen. Om den svarar kontrollerar du WinRM-tjänsten hos motparten:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Ett svar med protokollversion och tillverkare betyder att tjänsten och lyssnaren är igång. Testa först därefter med inloggningsuppgifter:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Om detta anrop returnerar datornamnet på motparten är alla förutsättningar uppfyllda.

## Vanliga fel och deras orsaker

| Meddelande eller symptom | Trolig orsak | Åtgärd |
|---|---|---|
| Port 5985 kan inte nås | Ingen lyssnare eller brandväggen blockerar | `Enable-PSRemoting`, kontrollera brandväggsregeln och dess omfattning |
| WinRM cannot complete the operation | Tjänsten är avstängd på målsidan eller åtkomst tillåts endast från det lokala subnätet | Starta tjänsten, öppna brandväggsregeln för det adressintervall som behövs |
| The WinRM client cannot process the request … TrustedHosts | Anslutning utanför domänen utan TrustedHosts-post | Lägg till måladressen i TrustedHosts på klienten eller använd HTTPS |
| Access is denied (trots korrekt lösenord) | Remote UAC-filtrering för lokalt konto | Sätt `LocalAccountTokenFilterPolicy` till 1 eller använd ett domänkonto |
| Åtkomst till en andra resurs misslyckas i sessionen | Double hop: inloggningsuppgifter vidarebefordras inte | Kör uppgiften direkt på målet eller använd CredSSP respektive delegerad autentisering |

## Begränsningar: double-hop-problemet

En begränsning kvarstår även med fullständig konfiguration och kan bara kringgås: Som standard kan en fjärrsession inte vidarebefordra dina inloggningsuppgifter till ett tredje system. Om du i en session på måldatorn försöker komma åt en nätverksresurs eller en annan server misslyckas detta på grund av saknade inloggningsuppgifter. Detta double-hop-problem är en säkerhetsegenskap, inte en felkonfiguration. För de flesta supportuppgifter räcker det att köra kommandot direkt på måldatorn. Där vidarebefordran verkligen behövs kommer CredSSP eller begränsad delegering i fråga, båda med egna säkerhetsavvägningar.

## Källor

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Förutsättningar för PowerShell-fjärrkommunikation, behörigheter och nätverksprofiler.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): Vad kommandot konfigurerar, inklusive förbehåll för nätverksprofil och brandväggsregel.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autentisering utanför domänen och vanliga felmeddelanden.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): Orsaken till double-hop-problemet och lösningsmetoderna med deras avvägningar.
