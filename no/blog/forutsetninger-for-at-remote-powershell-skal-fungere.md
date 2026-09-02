---
title: "Forutsetninger for at Remote PowerShell skal fungere"
navTitle: "Remote PowerShell"
description: "PowerShell-Remoting mislykkes sjelden på grunn av kommandoen, men på grunn av forutsetningene: WinRM-tjeneste, lytter, brannmur, autentisering og særegenhetene ved lokale kontoer. Hva som må være konfigurert på mål- og klientsiden, hvordan du kontrollerer det med Test-WSMan, og hvorfor Access denied som regel ikke har noe med passordet å gjøre."
date: "2026-09-01"
kategorie: "Windows og PowerShell"
timeToRead: "10 min lesetid"
themen:
  - windows-client
produkte:
  - "windows-client"
protokolle:
  - "powershell"
  - "haertung"
slug: "forutsetninger-for-at-remote-powershell-skal-fungere"
translationId: "article-7315c1ae9e67a24d"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, PowerShell-Remoting (WinRM) zwischen zwei Rechnern einzurichten und Fehler einzugrenzen: Dienst und Listener auf der Zielseite, Firewall, TrustedHosts auf der Clientseite, Authentisierung bei Domänen- und lokalen Konten, und die Prüfung mit Test-WSMan.
translationOf: remote-powershell-voraussetzungen
url: https://rafaelpfister.ch/no/blog/forutsetninger-for-at-remote-powershell-skal-fungere
translationSourceHash: 2969f02b5e677daaea867ea7c19fe929dc58f628cc4e47f3b165e85329836464
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:47:58.218Z
translationReview: automatic
---

# Forutsetninger for at Remote PowerShell skal fungere

`Invoke-Command` og `Enter-PSSession` er raske å skrive, men forbindelsen opprettes først når forutsetningene er oppfylt på begge sider. PowerShell-Remoting bygger på WS-Management (WinRM), en SOAP-basert administrasjonstjeneste over HTTP. Hvis en økt mislykkes, skyldes det nesten aldri selve cmdleten, men en manglende tjeneste, en lukket port, en brannmurregel eller autentiseringen. Denne artikkelen går gjennom forutsetningene i rekkefølge og viser hvordan du kan kontrollere hver enkelt.

Først begrepene: Målmaskinen er maskinen kommandoene skal kjøres på; klienten er maskinen du kobler til fra. WinRM lytter som standard på port 5985 (HTTP) og, hvis konfigurert, på port 5986 (HTTPS). HTTP-trafikken på 5985 er kryptert på meldingsnivå så snart autentiseringen skjer via Kerberos eller NTLM.

## Oversikt over cmdletene

Her er cmdletene som brukes i denne artikkelen:

<details class="options-details">
<summary>Oversikt over alternativer</summary>

| Cmdlet | Formål |
|---|---|
| `Enable-PSRemoting` | Konfigurerer WinRM på målsiden: tjeneste, lytter, brannmurregel |
| `Test-WSMan` | Kontrollerer om WinRM-tjenesten på motparten svarer |
| `Enter-PSSession` | Åpner en interaktiv ekstern økt mot en maskin |
| `Invoke-Command` | Kjører en kommandoblokk på én eller flere maskiner |
| `Set-Item WSMan:\localhost\Client\TrustedHosts` | Legger til klarerte motparter for autentisering utenfor et domene |
| `Get-Service WinRM` | Viser status og oppstartstype for WinRM-tjenesten |

</details>

## Målsiden: Konfigurer WinRM

På målmaskinen konfigurerer én enkelt kommando tjenesten, lytteren og brannmurregelen. Kjør den i PowerShell med administratorrettigheter:

```powershell
Enable-PSRemoting -Force
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Force` | Kjører uten spørsmål |
| `-SkipNetworkProfileCheck` | Konfigurerer også Remoting når en nettverkstilkobling er klassifisert som offentlig |

</details>

`Enable-PSRemoting` starter WinRM-tjenesten, setter oppstartstypen til automatisk, oppretter en HTTP-lytter og legger til den riktige brannmurregelen. Ett forbehold gjelder nettverksprofilen: Hvis et nettverkskort er klassifisert som offentlig, nekter kommandoen som standard å konfigurere dette. På servere eller i kontrollerte nettverk hjelper `-SkipNetworkProfileCheck`, slik at konfigurasjonen likevel fullføres.

Det er viktig å være oppmerksom på brannmurregelens virkeområde. For offentlige nettverksprofiler begrenser standardregelen tilgangen til det lokale delnettet. Kobler du til via et annet nettverk, for eksempel et VPN, gjelder denne begrensningen, og forbindelsen mislykkes selv om tjenesten kjører. Åpne da regelen målrettet for det nødvendige adresseområdet, ikke generelt for alle adresser:

```powershell
Set-NetFirewallRule -Name 'WINRM-HTTP-In-TCP*' -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Name 'WINRM-HTTP-In-TCP*'` | Velger WinRM-HTTP-reglene opprettet av Enable-PSRemoting via navnemønsteret |
| `-RemoteAddress <Bereich>` | Begrenses tillatte kildeadresser til det angitte området (her en CIDR-blokk); `Any` tillater alle adresser |

</details>

## Klientsiden: TrustedHosts og tjeneste

På klienten må WinRM-tjenesten kjøre, ellers mislykkes allerede innstilling av konfigurasjon. Kontroller dette først:

```powershell
Get-Service WinRM
```

Hvis tjenesten står som Stopped, starter du den med `Start-Service WinRM` (administratorrettigheter kreves). Oppstartstypen er ofte manuell på klienter, slik at tjenesten er stoppet igjen etter omstart. Hvis du jevnlig får tilgang fra denne maskinen, setter du oppstartstypen til automatisk.

Det andre punktet gjelder autentisering utenfor et domene. Kobler du til via IP-adresse eller i en arbeidsgruppe, kan ikke klienten kontrollere motparten med Kerberos og faller tilbake til NTLM. Av sikkerhetsgrunner nekter WinRM dette så lenge motparten ikke er registrert som klarert. Legg målmaskinens adresse til i TrustedHosts (administratorrettigheter kreves):

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '100.105.207.14' -Force
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Effekt |
|---|---|
| `-Value <Liste>` | Klarerte motparter (IP eller navn), flere atskilt med komma, `*` som jokertegn |
| `-Force` | Angir verdien uten spørsmål |
| `-Concatenate` | Legger til i den eksisterende listen i stedet for å erstatte den |

</details>

TrustedHosts er en innstilling på klienten, ikke målmaskinen, og gjelder klientens sikkerhet: De oppførte motpartene anses som klarerte uten at identiteten deres kontrolleres kryptografisk. Legg derfor inn konkrete adresser, ikke jokertegnet `*`. I et domene med Kerberos er oppføringen ikke nødvendig; den ryddige løsningen utenfor et domene uten TrustedHosts er en HTTPS-lytter med et sertifikat klienten stoler på.

## Autentisering: hvorfor Access denied sjelden skyldes passordet

Et vanlig feilbilde med lokale kontoer er meldingen Access denied, selv om passordet stemmer. Årsaken er ekstern UAC-filtrering: For lokale kontoer (ikke den innebygde Administrator-kontoen) fjerner Windows som standard administratorrettighetene ved tilgang over nettverket. Innloggingen lykkes, men enhver handling med utvidede rettigheter avvises. Hvis motparten melder Access denied i stedet for feil påloggingsinformasjon, er dette den sannsynlige årsaken.

Dette kan løses på målmaskinen med en registerverdi som gir lokale administratorer fullstendige rettigheter over nettverket:

```powershell
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LocalAccountTokenFilterPolicy -Value 1 -Type DWord
```

Dette er en bevisst oppmykning: Lokale administratorkontoer får dermed fullstendige rettigheter over nettverket. Angi verdien kun i kontrollerte nettverk og med sterke passord. I et domene er det bedre å bruke en domenekonto; da oppstår ikke spørsmålet.

Når du oppretter forbindelsen, oppgir du brukernavnet for lokale kontoer med maskinnavnet foran, slik at målsystemet løser kontoen lokalt:

```powershell
$cred = Get-Credential
Enter-PSSession -ComputerName 100.105.207.14 -Credential $cred
```

I påloggingsdialogen angir du brukeren som `RECHNERNAME\Benutzer` for lokale kontoer, og som `DOMAENE\Benutzer` for domenekontoer. En PIN-kode fra Windows-påloggingen fungerer ikke over nettverket; du trenger kontopassordet. For en Microsoft-konto er dette passordet til kontoen, og kontonavnet kan avvike fra visningsnavnet.

## Kontroller i riktig rekkefølge

Avgrens feil nedenfra og opp, så ser du raskt hvilken forutsetning som mangler.

Kontroller først at porten er tilgjengelig:

```powershell
Test-NetConnection -ComputerName 100.105.207.14 -Port 5985
```

Hvis porten ikke svarer, mangler lytteren eller brannmuren blokkerer. Hvis den svarer, kontrollerer du WinRM-tjenesten på motparten:

```powershell
Test-WSMan -ComputerName 100.105.207.14
```

Et svar med protokollversjon og produsent betyr at tjenesten og lytteren er på plass. Først deretter tester du med påloggingsinformasjon:

```powershell
Invoke-Command -ComputerName 100.105.207.14 -Credential $cred -ScriptBlock { $env:COMPUTERNAME }
```

Hvis dette kallet returnerer maskinnavnet til motparten, er alle forutsetningene oppfylt.

## Vanlige feil og årsakene til dem

| Melding eller symptom | Sannsynlig årsak | Tiltak |
|---|---|---|
| Port 5985 ikke tilgjengelig | Ingen lytter eller brannmuren blokkerer | `Enable-PSRemoting`, kontroller brannmurregel og virkeområde |
| WinRM cannot complete the operation | Tjenesten på målsiden er av, eller tilgang er bare tillatt fra det lokale delnettet | Start tjenesten, åpne brannmurregelen for det nødvendige adresseområdet |
| The WinRM client cannot process the request … TrustedHosts | Tilkobling utenfor domene uten TrustedHosts-oppføring | Legg målmaskinens adresse til i TrustedHosts på klienten, eller bruk HTTPS |
| Access is denied (til tross for korrekt passord) | Ekstern UAC-filtrering for lokal konto | Sett `LocalAccountTokenFilterPolicy` til 1, eller bruk en domenekonto |
| Tilgang til en annen ressurs mislykkes i økten | Double-hop: Påloggingsinformasjon videresendes ikke | Utfør oppgaven direkte på målet, eller bruk CredSSP eller separat pålogging |

## Begrensninger: Double-hop-problemet

En begrensning består også ved fullstendig konfigurasjon og kan bare omgås: Som standard kan ikke en ekstern økt videresende påloggingsinformasjonen din til et tredje system. Hvis du i en økt på målmaskinen får tilgang til en nettverksressurs eller en annen server, mislykkes dette på grunn av manglende påloggingsinformasjon. Dette double-hop-problemet er en sikkerhetsegenskap, ikke en feilkonfigurasjon. For de fleste supportoppgaver er det nok å kjøre kommandoen direkte på målmaskinen. Der videresending faktisk er nødvendig, kan CredSSP eller begrenset delegering brukes, begge med egne sikkerhetsavveininger.

## Kilder

1.  [about_Remote_Requirements (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements): Forutsetninger for PowerShell-Remoting, rettigheter og nettverksprofiler.

2.  [Enable-PSRemoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting): Hva kommandoen konfigurerer, inkludert forbeholdet for nettverksprofil og brannmurregel.

3.  [about_Remote_Troubleshooting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting): TrustedHosts, autentisering utenfor domenet og vanlige feilmeldinger.

4.  [Making the second hop in PowerShell Remoting (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop): Årsaken til double-hop-problemet og løsningsmetodene med deres avveininger.
