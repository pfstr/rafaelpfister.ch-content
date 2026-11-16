---
title: "Registrere og hente ansvarlige for delte postbokser i Active Directory"
navTitle: "Ansvarlige for delte postbokser"
description: "DLP-løsninger må vite hvem de skal varsle når en melding fra en delt postboks blokkeres. Hvilke AD-attributter som egner seg, hvilke som kun avbilder tilgangsrettigheter, hvorfor Exchange Online-tillatelser ikke kommer til det lokale AD, og hvordan en leverandør automatisk henter tilordningen via LDAP eller Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min lesetid"
themen:
  - active-directory-entra
  - microsoft-365-exchange
produkte:
  - "entra"
  - "exchange-online"
  - "exchange"
protokolle:
  - "ldap"
  - "powershell"
slug: "registrere-og-hente-ansvarlige-for-delte-postbokser-i-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/no/blog/registrere-og-hente-ansvarlige-for-delte-postbokser-i-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:54:09.076Z
translationReview: automatic
---

# Registrere og hente ansvarlige for delte postbokser i Active Directory

I et Data Loss Prevention-prosjekt fikk jeg følgende forespørsel: Så snart en melding fra en delt postboks (Shared Mailbox) blokkeres, skal de ansvarlige personene varsles slik at de kan reagere. Leverandøren av DLP-løsningen skal lese denne tilordningen automatisk fra Active Directory. Spørsmålet er: Hvor står det i AD hvem som er ansvarlig for en delt postboks?

Kort konklusjon: AD kjenner bare tilgangsrettigheter, ikke ansvar. Enkelte attributter viser hvem som kan få tilgang til eller sende fra en postboks, og også disse bare ufullstendig når postboksen ligger i Exchange Online. For pålitelig automatisering må den ansvarlige personen registreres eksplisitt i et attributt. Standardattributtet `manager` egner seg til dette, synkroniseres av Entra Connect og kan spørres via LDAP og Microsoft Graph.

## Tilgang er ikke ansvar

En delt postboks er i AD et deaktivert brukerobjekt med Exchange-attributter. Hvem som bruker den, fremgår av tillatelser: full tilgang (FullAccess), send som (Send As) og send på vegne av (Send on Behalf). Disse listene inneholder ofte ti eller tjue personer, i tillegg til stedfortredere, lærlinger og tjenestekontoer. Et DLP-varsel til alle med tillatelser når mange personer, uten at noen føler seg ansvarlig.

Spørsmålet om ansvar er organisatorisk: Én person (eller en liten gruppe) avgjør om en blokkert melding var legitim, og vedlikeholder tillatelsene. Denne informasjonen finnes bare i AD dersom noen registrerer den.

## Hvilke attributter i AD som kan være aktuelle

| Attributt | Innhold | Egnethet for ansvarlige |
|---|---|---|
| `manager` | DN for én person, enkeltverdi, del av personklassene og dermed tilgjengelig på postboksens brukerobjekt | Egnet. Synkroniseres av Entra Connect til Entra ID og kan leses via Graph |
| `managedBy` | DN for én administrator, enkeltverdi | Ikke tilgjengelig på brukerobjektet. Ifølge skjemaet definert blant annet for grupper, kontakter, datamaskiner og OU-er, ikke for klassen User |
| `msExchDelegateListLink` | DN-er for personer som har full tilgang med Automapping, flerverdi; tilbakekobling `msExchDelegateListLinkBL` på brukeren | Kun tilgang, og ufullstendig: full tilgang uten Automapping, rettigheter via grupper og rettigheter tildelt i Exchange Online mangler |
| `publicDelegates` | DN-er for personer med send på vegne av, flerverdi | Kun tilgang. Skrives tilbake av Entra Connect i hybridmiljøer |
| Tillatelsesliste (`nTSecurityDescriptor`) | Utvidet rettighet «Send As» på brukerobjektet | Kun send som tildelt lokalt. Rettigheter tildelt i Exchange Online finnes ikke der |
| `msExchMailboxSecurityDescriptor` | Postbokstillatelser for lokale postbokser, binær | Kun tilgang, kun for postbokser på lokale Exchange-servere, vanskelig for leverandører å evaluere |
| `extensionAttribute1` til `extensionAttribute15` | Frie tekstfelt | Mulig, men bare som tekst; ingen referanse til et objekt, ingen automatisk opprydding ved navneendring eller fratredelse |

Henvisningen til at tilordningen finnes «direkte i AD», viser i praksis oftest til `msExchDelegateListLink`: I ADUC eller ADSI Edit ser man faktisk personer der. Attributtet oppstår imidlertid bare som en bieffekt av Outlook-Automapping og egner seg ikke som datakilde for ansvar.

## Hvor postboksen ligger, avgjør datakilden

Hvis postboksene ligger i Exchange Online og brukerne synkroniseres fra det lokale AD (hybrid), gjelder følgende: Tillatelser som tildeles i Exchange Online, forblir i Exchange Online. Entra Connect skriver bare en liten attributtliste tilbake for Exchange Hybrid (blant annet `publicDelegates` og `proxyAddresses`), men verken `msExchDelegateListLink` eller Send As-rettigheter. Et lokalt AD viser i dette tilfellet statusen før migreringen pluss det som siden har blitt vedlikeholdt lokalt.

Kun Exchange Online selv leverer faktisk status for tillatelsene i Exchange Online (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Leverandøren ville trenge tilgang til Exchange Online PowerShell, noe som er uforholdsmessig for en ren tilordning. Et vedlikeholdt attributt på brukerobjektet er enklere og fungerer uavhengig av hvor postboksen ligger.

## Anbefaling: `manager` som attributt for ansvarlig

For hver delte postboks registreres den ansvarlige personen i attributtet `manager`. Begrunnelser:

- Attributtet er en del av standardskjemaet, så en skjemautvidelse er ikke nødvendig.
- Det inneholder en referanse (DN) i stedet for tekst. Dersom personen får nytt navn eller flyttes, forblir koblingen gyldig.
- Entra Connect synkroniserer `manager` til Entra ID. Leverandøren kan lese enten via LDAP i lokalt AD eller via Microsoft Graph.
- Gjennom tilbakekoblingen `directReports` kan hver ansvarlige person (og hver rapport) se hvilke postbokser vedkommende er ansvarlig for.

Begrensningen: `manager` har én enkeltverdi. Der stedfortreder er påkrevd, finnes det to muligheter. Mulighet 1: En gruppe per postboks, der medlemmene er de ansvarlige; navnet følger en konvensjon (for eksempel `SM-Verantwortlich-buchhaltung`) eller står i et `extensionAttribute`. Mulighet 2: Stedfortrederen utledes hos leverandøren ikke fra AD, men fra `manager` til den ansvarlige personen (eskalering til nærmeste leder). Mulighet 2 krever ingen ekstra objekter og er tilstrekkelig i de fleste organisasjoner.

Attributtet settes bare i det autoritative systemet: i lokalt AD for synkroniserte objekter, i Entra ID for rene skyobjekter.

## Kartlegg eksisterende situasjon: Hvem har tilgang i dag?

For første utfylling er en liste over dagens autoriserte nyttig. Den ansvarlige personen er som regel blant dem; fagavdelingen bekrefter valget. For postbokser i Exchange Online:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Kun delte postbokser, ikke bruker-, rom- eller utstyrspostbokser |
| `-ResultSize Unlimited` | Alle postbokser i stedet for standardgrensen på 1000 |
| `-Identity` | Postboks som tillatelsene hentes for |
| `AccessRights -contains 'FullAccess'` | Filtrerer på full tilgang; andre postbokstillatelser utelates |
| `-not $_.IsInherited` | Fjerner arvede oppføringer (administratorer, systemkontoer) |
| `User -notlike 'NT AUTHORITY\*'` | Fjerner postboksens egen oppføring `NT AUTHORITY\SELF` |
| `-NoTypeInformation` | Ingen typeoverskrift i starten av CSV-filen |
| `-Encoding UTF8` | Spesialtegn i navn beholdes |

</details>

Hvis en gruppe har tillatelse, vises gruppenavnet i listen; medlemmene kan løses opp med `Get-DistributionGroupMember` eller `Get-ADGroupMember`.

## Angi ansvarlige

Den bekreftede tilordningen forberedes som CSV med kolonnene `Postfach` og `Verantwortlich` (begge e-postadresser) og settes i lokalt AD:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-Filter "mail -eq '…'"` | Finner objektet ved hjelp av den primære e-postadressen |
| `-Identity` | Brukerobjektet for den delte postboksen |
| `-Manager` | Skriver DN-en til den ansvarlige personen i attributtet `manager` |

</details>

For rene skyobjekter uten lokalt AD setter `Set-MgUserManagerByRef` fra Microsoft Graph PowerShell SDK den samme verdien i Entra ID.

## Henting for leverandøren: LDAP

Delte postbokser kan gjenkjennes i AD via `msExchRecipientTypeDetails`: `4` står for en delt postboks på en lokal Exchange-server, `34359738368` for en delt postboks i Exchange Online (RemoteSharedMailbox). Forespørselen som leverandøren kan ta inn i sin løsning:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

Verdien i `manager` er en DN. E-postadressen til den ansvarlige personen hentes med en ny forespørsel mot denne DN-en med attributtet `mail`. En lesekonto uten ytterligere rettigheter er tilstrekkelig, siden `manager` og `mail` kan leses av autentiserte brukere. Tilkoblingen bør gå over LDAPS (port 636).

For å kontrollere tilordningen i PowerShell:

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
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-LDAPFilter` | LDAP-filter for de to mottakertypene for delte postbokser |
| `-Properties mail, manager` | Laster inn de to attributtene som `Get-ADUser` ikke leverer som standard |
| `Expression = { … }` | Løser DN-en i `manager` til e-postadressen til den ansvarlige personen |

</details>

For meldingstilordningen bør leverandøren ta hensyn til alle adresser i `proxyAddresses`, ikke bare `mail`: En melding kan sendes med en sekundær adresse til postboksen.

## Henting for leverandøren: Microsoft Graph

Dersom leverandøren ikke har LDAP-tilgang til det interne nettverket, er Microsoft Graph et godt alternativ. Delte postbokser finnes i Entra ID som (deaktiverte) brukerobjekter, og `manager` er tilgjengelig der gjennom synkroniseringen:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

Leverandørens appregistrering trenger apptillatelsen `User.Read.All`. Graph tilbyr ikke direkte en liste over alle delte postbokser, siden Entra ID ikke kjenner postbokstypen. Leverandøren spør derfor målrettet etter avsenderadressen til den blokkerte meldingen, noe som uansett er tilstrekkelig for DLP-brukstilfellet.

## Integrering i DLP-varslingen

Hvordan varslingen utløses teknisk, avhenger av DLP-løsningen. For Microsoft Purview DLP må det tas hensyn til at varsler i tillegg kan sendes til fast angitte personer, men bare til enkeltmottakere, ikke til grupper eller distribusjonslister. En dynamisk tilordning «postboks til ansvarlig person» er som regel ikke planlagt. Ved få postbokser kan én regel per postboks være mulig; ved mange postbokser trengs en etterfølgende prosess som henter hendelser, løser avsenderadressen via LDAP eller Graph og informerer den ansvarlige personen.

For en ekstern løsning gjelder samme logikk: Finn avsenderadressen til den blokkerte meldingen, finn postboksen via `mail` eller `proxyAddresses`, løs opp `manager`, varsle. Hvis `manager` mangler, bør løsningen varsle en definert fellesadresse (servicedesk eller personvern) i stedet for å forkaste varselet.

## Vedlikehold i drift

For å holde tilordningen oppdatert er to regler tilstrekkelige i praksis:

- Nye delte postbokser opprettes bare med en registrert ansvarlig person; forespørselen inneholder feltet.
- En månedlig rapport lister postbokser uten `manager` og postbokser der den ansvarlige personen er deaktivert.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

Ved en persons fratredelse viser `directReports` på vedkommendes objekt hvilke postbokser som trenger en ny ansvarlig person. Denne kontrollen kan tas inn i fratredelsesprosessen.

## Kilder

1.  [Manager attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): enkeltverdi-DN-attributt for personklassene, tilbakekobling `directReports`.

2.  [Managed-By attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): liste over klassene som `managedBy` er definert for; klassen User er ikke blant dem.

3.  [Microsoft Entra Connect Sync: Attributes synchronized to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): synkroniserte attributter inkludert `manager`, samt listen over tilbakeføringsattributter for Exchange Hybrid.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): henting av postbokstillatelser i Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): henting av Send As-tillatelser i Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): Microsoft-støtteartikkel om sammenhengen mellom full tilgang og `msExchDelegateListLink` i Exchange 2016 og 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): Graph-endepunkt og nødvendige tillatelser for å hente manageren.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): mottakere av DLP-varsler i Microsoft Purview; bare enkeltmottakere, ingen grupper.
