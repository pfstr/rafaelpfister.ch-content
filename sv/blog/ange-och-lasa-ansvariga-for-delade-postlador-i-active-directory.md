---
title: "Ange och läsa ansvariga för delade postlådor i Active Directory"
navTitle: "Ansvariga för delade postlådor"
description: "DLP-lösningar måste veta vem de ska informera när ett meddelande från en delad postlåda blockeras. Vilka AD-attribut som lämpar sig för detta, vilka som bara återspeglar åtkomsträttigheter, varför Exchange Online-behörigheter inte hamnar i det lokala AD och hur en leverantör automatiskt läser ut kopplingen via LDAP eller Microsoft Graph."
date: "2026-11-16"
kategorie: "Active Directory / Entra"
timeToRead: "8 min. lästid"
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
slug: "ange-och-lasa-ansvariga-for-delade-postlador-i-active-directory"
translationId: "article-ed41bd7fefc08361"
translationOf: shared-mailbox-verantwortliche-ad
url: https://rafaelpfister.ch/sv/blog/ange-och-lasa-ansvariga-for-delade-postlador-i-active-directory
translationSourceHash: 54b0d22f6d1fd74974494b96e5566af25ea846062f8f9ce6a255607e40f937e6
translationModel: gpt-5.6-terra
translatedAt: 2026-11-16T08:53:33.348Z
translationReview: automatic
---

# Ange och läsa ansvariga för delade postlådor i Active Directory

I ett Data Loss Prevention-projekt hamnade följande förfrågan på mitt bord: Så snart ett meddelande från en delad postlåda (Shared Mailbox) blockeras ska de ansvariga personerna meddelas så att de kan agera. Leverantören av DLP-lösningen ska läsa denna koppling automatiskt från Active Directory. Frågan är: Var anges i AD vem som ansvarar för en delad postlåda?

Kort slutsats: AD känner bara till åtkomsträttigheter, inte ansvar. Vissa attribut visar vem som kan komma åt eller skicka från en postlåda, och även dessa bara ofullständigt när postlådan finns i Exchange Online. För tillförlitlig automatisering måste den ansvariga personen uttryckligen registreras i ett attribut. Standardattributet `manager` lämpar sig för detta, synkroniseras av Entra Connect och kan efterfrågas via LDAP och Microsoft Graph.

## Åtkomst är inte ansvar

En delad postlåda är i AD ett inaktiverat användarobjekt med Exchange-attribut. Vem som använder den framgår av behörigheter: fullständig åtkomst (FullAccess), Skicka som (Send As) och Skicka för (Send on Behalf). Dessa listor innehåller ofta tio eller tjugo personer, samt ställföreträdare, praktikanter och tjänstkonton. Ett DLP-meddelande till alla behöriga når många personer, utan att någon känner sig ansvarig.

Frågan om ansvar är organisatorisk: En person (eller en liten grupp) avgör om ett blockerat meddelande var legitimt och hanterar behörigheterna. Denna information finns bara i AD om någon anger den.

## Vilka attribut i AD som kan komma i fråga

| Attribut | Innehåll | Lämplighet för ansvariga |
|---|---|---|
| `manager` | DN för en person, enkelvärd, del av personklasserna och därmed tillgängligt på postlådans användarobjekt | Lämpligt. Synkroniseras av Entra Connect till Entra ID och kan läsas via Graph |
| `managedBy` | DN för en förvaltare, enkelvärd | Inte tillgängligt på användarobjektet. Enligt schemat definierat bland annat för grupper, kontakter, datorer och OU:er, inte för klassen User |
| `msExchDelegateListLink` | DN:er för personer som har fullständig åtkomst med Automapping, flervärdigt; bakåtreferens `msExchDelegateListLinkBL` på användaren | Endast åtkomst och ofullständigt: fullständig åtkomst utan Automapping, rättigheter via grupper och rättigheter som tilldelats i Exchange Online saknas |
| `publicDelegates` | DN:er för personer med Skicka för-behörighet, flervärdigt | Endast åtkomst. Skrivs tillbaka av Entra Connect i hybridmiljöer |
| Behörighetslista (`nTSecurityDescriptor`) | Utökad rättighet «Send As» på användarobjektet | Endast lokalt tilldelad Skicka som-behörighet. Rättigheter som tilldelats i Exchange Online finns inte där |
| `msExchMailboxSecurityDescriptor` | Postlådebehörigheter för lokala postlådor, binärt | Endast åtkomst, endast för postlådor på lokala Exchange-servrar, svår för leverantörer att utvärdera |
| `extensionAttribute1` till `extensionAttribute15` | Fria textfält | Möjligt, men endast som text; ingen referens till ett objekt, ingen automatisk rensning vid namnbyte eller avgång |

Hänvisningen till att kopplingen finns «direkt i AD» avser i praktiken oftast `msExchDelegateListLink`: I ADUC eller ADSI Edit syns där faktiskt personer. Attributet uppstår dock endast som en bieffekt av Outlook Automapping och lämpar sig inte som datakälla för ansvarsfördelning.

## Var postlådan finns avgör datakällan

Om postlådorna finns i Exchange Online och användarna synkroniseras från det lokala AD:t (hybrid) gäller följande: Behörigheter som tilldelas i Exchange Online stannar i Exchange Online. Entra Connect skriver bara tillbaka en liten attributlista för Exchange Hybrid (däribland `publicDelegates` och `proxyAddresses`), men varken `msExchDelegateListLink` eller Send As-rättigheter. Ett lokalt AD visar i detta fall läget före migreringen samt det som sedan dess har uppdaterats lokalt.

Det faktiska behörighetsläget för Exchange Online tillhandahålls endast av Exchange Online självt (`Get-EXOMailboxPermission`, `Get-EXORecipientPermission`). Leverantören skulle behöva åtkomst till Exchange Online PowerShell, vilket är oproportionerligt för en ren koppling. Ett underhållet attribut på användarobjektet är enklare och fungerar oberoende av var postlådan finns.

## Rekommendation: `manager` som attribut för ansvarig

För varje delad postlåda anges den ansvariga personen i attributet `manager`. Skälen är:

- Attributet ingår i standardschemat, så någon schemautökning behövs inte.
- Det innehåller en referens (DN) i stället för text. Om personen byter namn eller flyttas förblir kopplingen giltig.
- Entra Connect synkroniserar `manager` till Entra ID. Leverantören kan läsa det antingen via LDAP i det lokala AD:t eller via Microsoft Graph.
- Via bakåtreferensen `directReports` ser varje ansvarig person (och varje rapport) vilka postlådor personen ansvarar för.

Begränsningen är att `manager` är enkelvärdigt. Där en ställföreträdare är nödvändig finns två möjligheter. Möjlighet 1: En grupp per postlåda där medlemmarna är de ansvariga; dess namn följer en konvention (till exempel `SM-Verantwortlich-buchhaltung`) eller anges i ett `extensionAttribute`. Möjlighet 2: Ställföreträdaren härleds hos leverantören inte från AD utan från `manager` för den ansvariga personen (eskalering till överordnad person). Möjlighet 2 kräver inga ytterligare objekt och räcker i de flesta organisationer.

Attributet anges endast i det ledande systemet: för synkroniserade objekt i det lokala AD:t, för rena molnobjekt i Entra ID.

## Inventera befintligt läge: Vem har åtkomst i dag?

För den första ifyllningen hjälper en lista över dagens behöriga. Den ansvariga personen finns oftast med bland dem; verksamhetsavdelningen bekräftar valet. För postlådor i Exchange Online:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-RecipientTypeDetails SharedMailbox` | Endast delade postlådor, inga användar-, rums- eller utrustningspostlådor |
| `-ResultSize Unlimited` | Alla postlådor i stället för standardgränsen på 1000 |
| `-Identity` | Postlåda vars behörigheter efterfrågas |
| `AccessRights -contains 'FullAccess'` | Filtrerar på fullständig åtkomst; andra postlådebehörigheter utelämnas |
| `-not $_.IsInherited` | Tar bort ärvda poster (administratörer, systemkonton) |
| `User -notlike 'NT AUTHORITY\*'` | Tar bort posten `NT AUTHORITY\SELF` för själva postlådan |
| `-NoTypeInformation` | Ingen typrad i början av CSV-filen |
| `-Encoding UTF8` | Umlauter i namn bevaras |

</details>

Om en grupp är behörig visas dess namn i listan; medlemmarna kan lösas upp med `Get-DistributionGroupMember` eller `Get-ADGroupMember`.

## Ange ansvariga

Den bekräftade kopplingen förbereds som en CSV med kolumnerna `Postfach` och `Verantwortlich` (båda e-postadresser) och anges i det lokala AD:t:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-Filter "mail -eq '…'"` | Söker efter objektet utifrån den primära e-postadressen |
| `-Identity` | Användarobjekt för den delade postlådan |
| `-Manager` | Skriver den ansvariga personens DN till attributet `manager` |

</details>

För rena molnobjekt utan lokalt AD anger `Set-MgUserManagerByRef` från Microsoft Graph PowerShell SDK samma värde i Entra ID.

## Läsning hos leverantören: LDAP

Delade postlådor kan identifieras i AD via `msExchRecipientTypeDetails`: `4` står för en delad postlåda på en lokal Exchange-server, `34359738368` för en delad postlåda i Exchange Online (RemoteSharedMailbox). Frågan som leverantören kan använda i sin lösning:

```text
Basis:     DC=example,DC=com (Unterbaum)
Filter:    (&(objectClass=user)(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368)))
Attribute: mail, proxyAddresses, manager
```

Värdet i `manager` är ett DN. E-postadressen till den ansvariga personen fås genom en andra fråga mot detta DN med attributet `mail`. Ett läskonto utan ytterligare rättigheter räcker, eftersom `manager` och `mail` kan läsas av autentiserade användare. Anslutningen bör ske via LDAPS (port 636).

För att kontrollera kopplingen i PowerShell:

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-LDAPFilter` | LDAP-filter för de två mottagartyperna för delade postlådor |
| `-Properties mail, manager` | Läser in de två attribut som `Get-ADUser` inte levererar som standard |
| `Expression = { … }` | Löser upp DN:et i `manager` till e-postadressen för den ansvariga personen |

</details>

För meddelandekopplingen bör leverantören beakta alla adresser i `proxyAddresses`, inte bara `mail`: Ett meddelande kan skickas med en sekundär adress för postlådan.

## Läsning hos leverantören: Microsoft Graph

Om leverantören inte har LDAP-åtkomst till det interna nätet är Microsoft Graph lämpligt. Delade postlådor finns i Entra ID som (inaktiverade) användarobjekt, och `manager` finns där genom synkroniseringen:

```http
GET https://graph.microsoft.com/v1.0/users/buchhaltung@example.com/manager?$select=displayName,mail
```

Leverantörens appregistrering behöver för detta programbehörigheten `User.Read.All`. Graph tillhandahåller inte direkt en lista över alla delade postlådor, eftersom Entra ID inte känner till postlådetypen. Leverantören frågar därför specifikt efter avsändaradressen för det blockerade meddelandet, vilket ändå räcker för DLP-användningsfallet.

## Integrering i DLP-meddelandet

Hur meddelandet utlöses tekniskt beror på DLP-lösningen. För Microsoft Purview DLP bör följande beaktas: Meddelanden kan dessutom skickas till fast angivna personer, men endast till enskilda mottagare, inte till grupper eller distributionslistor. En dynamisk koppling «postlåda till ansvarig person» är normalt inte avsedd. Vid få postlådor kan en regel per postlåda vara tänkbar; vid många postlådor krävs en efterföljande process som hämtar incidenter, löser upp avsändaradressen via LDAP eller Graph och informerar den ansvariga personen.

Samma logik gäller för en extern lösning: Fastställ avsändaradressen för det blockerade meddelandet, hitta postlådan via `mail` eller `proxyAddresses`, lös upp `manager`, meddela. Om `manager` saknas bör lösningen rapportera till en definierad gemensam adress (servicedesk eller dataskydd) i stället för att förkasta meddelandet.

## Förvaltning i drift

För att kopplingen ska hållas aktuell räcker i praktiken två regler:

- Nya delade postlådor skapas endast med en angiven ansvarig person; formuläret innehåller fältet.
- En månatlig rapport listar postlådor utan `manager` och postlådor vars ansvariga person är inaktiverad.

```powershell
$filter = '(|(msExchRecipientTypeDetails=4)(msExchRecipientTypeDetails=34359738368))'

Get-ADUser -LDAPFilter $filter -Properties mail, manager |
    Where-Object {
        -not $_.manager -or
        -not (Get-ADUser -Identity $_.manager).Enabled
    } |
    Select-Object mail, manager
```

När en person slutar visar `directReports` på personens objekt vilka postlådor som behöver en ny ansvarig person. Denna kontroll kan läggas till i avgångsprocessen.

## Källor

1.  [Manager attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-manager): enkelvärdigt DN-attribut för personklasserna, bakåtreferens `directReports`.

2.  [Managed-By attribute (Active Directory Schema)](https://learn.microsoft.com/en-us/windows/win32/adschema/a-managedby): lista över klasser för vilka `managedBy` är definierat; klassen User ingår inte.

3.  [Microsoft Entra Connect Sync: Attributes synchronized to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-sync-attributes-synchronized): synkroniserade attribut inklusive `manager` samt listan över Exchange Hybrid-writeback-attribut.

4.  [Get-EXOMailboxPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exomailboxpermission): fråga om postlådebehörigheter i Exchange Online.

5.  [Get-EXORecipientPermission](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-exorecipientpermission): fråga om Send As-behörigheter i Exchange Online.

6.  [Adding or removing mailbox permission in EAC doesn't address the msExchDelegateListLink attribute](https://support.microsoft.com/en-us/help/4502159): Microsoft-supportartikel om sambandet mellan fullständig åtkomst och `msExchDelegateListLink` i Exchange 2016 och 2019.

7.  [List manager (Microsoft Graph)](https://learn.microsoft.com/en-us/graph/api/user-list-manager): Graph-slutpunkt och nödvändiga behörigheter för att läsa ut chef.

8.  [Send email notifications and show policy tips for DLP policies](https://learn.microsoft.com/en-us/purview/dlp-use-notifications-and-policy-tips): mottagare av DLP-meddelanden i Microsoft Purview; endast enskilda mottagare, inga grupper.
