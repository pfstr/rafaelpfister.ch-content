---
title: "Nye Outlook: S/MIME-signatur i sekundær konto kan ikke verifiseres, vedlegg mangler"
navTitle: "S/MIME i sekundærkonto"
description: "Nye Outlook melder for en delt postboks at S/MIME-signaturen ikke kan verifiseres i den sekundære kontoen, og viser ingen vedlegg. Artikkelen forklarer forskjellen mellom Clear Signing og Opaque Signing, hvorfor vedleggene forsvinner i ugjennomsiktig signerte e-poster, hvorfor nye Outlook bare behandler S/MIME i primærkontoen og hvilke alternativer som finnes, inkludert utpakking av smime.p7m med PowerShell eller OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min. lesetid"
themen:
  - outlook
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "outlook"
protokolle:
  - "verschluesselung"
  - "troubleshooting"
related:
  - e-mail-header-analysieren-ohne-upload
slug: "nye-outlook-s-mime-signatur-i-sekundaer-konto-kan-ikke-verifiseres-vedlegg-mangler"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/no/blog/nye-outlook-s-mime-signatur-i-sekundaer-konto-kan-ikke-verifiseres-vedlegg-mangler
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:14:17.489Z
translationReview: automatic
---

# Nye Outlook: S/MIME-signatur i sekundær konto kan ikke verifiseres, vedlegg mangler

Når du åpner en digitalt signert e-post i en delt postboks i nye Outlook for Windows, vises en rød linje: «S/MIME-signaturen kan ikke verifiseres ved visning i den sekundære kontoen.» Selve e-posten vises, men ikke vedleggene, selv om avsenderen har sendt dem med. Kollegene som bruker den samme postboksen som hovedkonto, ser vedleggene uten problemer.

Det skyldes to forhold som forsterker hverandre: Nye Outlook behandler S/MIME bare i primærkontoen, og avsenderen har signert e-posten ugjennomsiktig. Med denne signaturformen ligger hele innholdet, inkludert vedlegg, i én enkelt kryptografisk beholder. Hvis klienten ikke kan åpne beholderen, forblir vedleggene usynlige. Begge deler kan løses hver for seg.

## Hva meldingen betyr

«Sekundær konto» betyr i nye Outlook enhver postboks som ikke er kontoen du er logget på med. Dette gjelder delte postbokser (Shared Mailboxes) som vises via Full Access og Automapping, samt postbokser du har lagt til via «Legg til delt postboks», og delegeringer. S/MIME-behandlingen i nye Outlook er fast knyttet til primærkontoen. Hvis en signert melding kommer til en annen konto, verifiserer ikke klienten signaturen og viser meldingen i stedet.

Dette sier ikke noe om signaturens gyldighet og er ikke et sertifikatproblem hos avsenderen. Den samme e-posten kan verifiseres og åpnes i primærkontoen eller i klassisk Outlook.

## Clear Signing og Opaque Signing

S/MIME-standarden (RFC 8551) har to formater for signerte meldinger. Begge leverer den samme signaturen, men pakker meldingen forskjellig.

| | Clear Signing | Opaque Signing |
|---|---|---|
| MIME-type | `multipart/signed` med `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` med `smime-type=signed-data` |
| Oppbygning | To deler: den lesbare meldingsteksten med vedlegg og den frakoblede signaturen ved siden av | Én del: meldingstekst, vedlegg og signatur samlet i en CMS-SignedData-beholder, Base64-kodet |
| Vedlegg som en klient uten S/MIME ser | `smime.p7s` (bare signaturen, noen få KB) | `smime.p7m` (hele meldingen) |
| Lesbar uten S/MIME-støtte | Ja, tekst og vedlegg vises normalt | Nei, klienten ser bare beholderen |
| Følsomhet under overføring | Signaturen blir ugyldig hvis en e-postserver eller gateway endrer linjeskift, koding eller mellomrom | Beholderen beskytter innholdet mot slike endringer |
| RFC-8551-avsnitt | 3.5.3 | 3.5.2 |

I meldingskilden kan du kjenne igjen de to formatene på overskriften `Content-Type`. En clear-signed e-post begynner slik:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

En opaque-signed e-post slik:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

Forskjellen forklarer oppførselen i nye Outlook fullt ut. For en clear-signed e-post viser klienten tekst og vedlegg selv når den ikke verifiserer signaturen; bare signaturstatusen mangler. For en opaque-signed e-post må klienten først pakke ut beholderen gjennom S/MIME-behandlingen for å få tilgang til tekst og vedlegg. Hvis den nekter dette fordi meldingen ligger i en sekundær konto, forblir beholderen lukket. At teksten likevel er lesbar, skyldes Exchange Online: Tjenesten gjengir tekstdelen på serversiden, men ikke vedleggene fra beholderen.

Begge formatene krypterer ingenting. Også den ugjennomsiktige varianten er bare Base64-kodet og lesbar for alle som får tak i meldingen. Microsoft påpeker dette uttrykkelig i dokumentasjonen for Exchange Online.

## Hvilket format avsenderen velger

I klassisk Outlook styrer alternativet «Send signerte meldinger som klartekst» (Fil > Alternativer > Klareringssenter > E-postsikkerhet) formatet. Det er aktivert som standard, så Outlook signerer clear-signed. Den som slår av alternativet, sender ugjennomsiktig. Nye Outlook og Outlook på nettet tilbyr ikke denne innstillingen.

E-postgatewayer som signerer sentralt, har sin egen innstilling for signaturformatet. Enkelte produkter signerer ugjennomsiktig som standard av robusthetshensyn, fordi signaturen da også forblir gyldig etter endringer utført av etterfølgende systemer. Hvis du regelmessig mottar e-poster med manglende vedlegg fra en bestemt avsender, kan det være verdt å se på gateway-konfigurasjonen deres.

## Hvorfor nye Outlook bare behandler S/MIME i primærkontoen

Microsoft dokumenterer begrensningen, men oppgir ingen teknisk årsak. Årsaken følger av klientens arkitektur.

Nye Outlook er i hovedsak nettklienten til Outlook på nettet i et innebygd native-skall. I nettleseren kan JavaScript ikke få tilgang til Windows-sertifikatlageret. Derfor trengte Outlook på nettet i mange år en separat S/MIME-nettleserutvidelse. Nye Outlook erstatter denne utvidelsen med en innebygd bro mellom nettgrensesnittet og Windows-kryptografien. Denne broen initialiseres når du logger på primærkontoen, og mottar postboksøkten, sertifikatene og S/MIME-innstillingene fra Innstillinger > E-post > S/MIME.

Delte postbokser og sekundærkontoer bruker andre databaner. Sekundærkontoer har egne økter, mens delte postbokser lastes via delegeringen til primærkontoen. Broen har hittil ikke vært koblet til disse banene. Dette gjelder også selve verifiseringen av en signatur, selv om det ikke ville vært nødvendig med en privat nøkkel: Parsing og utpakking av PKCS#7-strukturen går via den samme komponenten.

I klassisk Outlook oppstår ikke problemet, fordi S/MIME-behandlingen der skjer i MAPI-stakken per melding, uavhengig av hvilket lager meldingen kommer fra.

Microsoft legger til den manglende tilkoblingen: Roadmap-oppføring 565861 utvider S/MIME i nye Outlook til delte og delegerte postbokser som er knyttet til primærkontoen. Generell tilgjengelighet er varslet for juli 2026, og utrullingen skjer trinnvis. Hvis du fortsatt ser meldingen, har endringen ennå ikke nådd tenant-en din eller klientversjonen din. Oppføringen dekker ikke separat tilføyde sekundærkontoer med egen pålogging.

## Alternativer

Hvilken vei som passer, avhenger av hvordan postboksen er koblet til og om du må verifisere signaturen eller bare vil ha tilgang til vedleggene.

| Alternativ | Forutsetning | Resultat |
|---|---|---|
| Åpne e-posten i primærkontoen | Du har selv postboksen som hovedkonto, eller e-posten er videresendt til deg | Signaturverifisering og vedlegg |
| Legg til postboksen som en selvstendig konto i nye Outlook (Innstillinger > Kontoer > Legg til konto) | Postboksen har egne påloggingsopplysninger; ikke mulig for delte postbokser uten passord | Signaturverifisering og vedlegg når du bytter til denne kontoen |
| Klassisk Outlook | Fortsatt installert eller mulig å bytte tilbake via bryteren «Nye Outlook»; koble postboksen der som egen konto (Fil > Kontoinnstillinger > Ny) | Signaturverifisering og vedlegg i hvert lager |
| Outlook på nettet | Åpne postboksen direkte (`outlook.office.com/mail/<adresse>`), med S/MIME-utvidelse for Edge eller Chrome installert | Signaturverifisering og vedlegg |
| Be avsenderen om Clear Signing | Avsenderen bruker klassisk Outlook eller en gateway med valgbart format | Vedlegg synlige, men signaturstatusen vises fortsatt ikke i sekundærkontoen |
| Pakk ut beholderen manuelt | Lagre e-posten som `.eml` eller lagre `smime.p7m` | Vedlegg uten signaturverifisering |

## Pakk ut beholderen manuelt

For enkelttilfeller er det nok å åpne beholderen utenfor Outlook. Signaturen verifiseres da matematisk, men sertifikatets tillitskjede kontrolleres ikke. Lagre meldingen som `.eml` eller lagre vedlegget `smime.p7m` i en mappe.

På Windows er PowerShell tilstrekkelig. .NET Framework inkluderer klassen `SignedCms`, som leser PKCS#7-beholderen:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Instruksjon | Virkning |
|---|---|
| `Add-Type -AssemblyName System.Security` | Laster assemblyen med PKCS-klassene (nødvendig i Windows PowerShell 5.1, allerede lastet i PowerShell 7) |
| `[IO.File]::ReadAllBytes(...)` | Leser den binære DER-beholderen; `smime.p7m` som er lagret fra Outlook, er allerede dekodet |
| `$cms.Decode($bytes)` | Parser CMS-SignedData-strukturen |
| `$cms.CheckSignature($true)` | Verifiserer bare signaturen over innholdet (`$true` = verifySignatureOnly); med `$false` ville også sertifikatkjeden blitt kontrollert. Ved ugyldig signatur avsluttes kommandoen med et unntak |
| `$cms.ContentInfo.Content` | Det signerte innholdet: en fullstendig MIME-melding med tekst og vedlegg |
| `[IO.File]::WriteAllBytes(...)` | Skriver denne MIME-meldingen som `.eml`, som du kan åpne i Outlook eller Thunderbird |

</details>

På Linux, macOS eller med Git for Windows er OpenSSL tilgjengelig. Hvis hele e-posten foreligger som `.eml`, håndterer OpenSSL også Base64-dekodingen:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `cms` | CMS-verktøyet i OpenSSL; `smime` fungerer tilsvarende, `cms` er det nyere grensesnittet |
| `-verify` | Verifiserer signaturen og skriver ut det signerte innholdet |
| `-noverify` | Hopper over kontrollen av sertifikatkjeden; selve signaturen verifiseres likevel |
| `-in nachricht.eml` | Hele e-posten i S/MIME-format (Base64 med MIME-overskrifter); legg i tillegg til `smime.p7m` for en lagret `-inform DER` |
| `-out inhalt.eml` | Det utpakkede innholdet som MIME-melding |

</details>

Filen `inhalt.eml` inneholder den opprinnelige meldingsteksten og alle vedlegg som vanlige MIME-deler. Et dobbeltklikk åpner den i Outlook, der du kan lagre vedleggene som vanlig.

## Kilder

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): Tilfellet fra praksis med samme melding i den delte postboksen; svaret bekrefter at oppførselen er kjent og nevner ingen løsning i nye Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Avsnitt 3.5.2 (application/pkcs7-mime med SignedData) og 3.5.3 (multipart/signed) med opplysninger om lesbarhet uten S/MIME og robusthet under overføring.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): Alternativet «Send signerte meldinger som klartekst» i klassisk Outlook, aktivert som standard; finnes ikke i nye Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): S/MIME-innstillinger i nye Outlook under Innstillinger > E-post > S/MIME; sertifikater må installeres manuelt eller via policy.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Merknad om at ugjennomsiktig signerte meldinger bare er Base64-kodet og ikke konfidensielle.

6.  [Microsoft 365 Roadmap, oppføring 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME for delte og delegerte postbokser i nye Outlook for Windows, varslet for juli 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Hvilke kontotyper nye Outlook støtter, og hvordan delte postbokser kobles til.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature og ContentInfo for utpakking av beholderen med PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Alternativene `-verify`, `-noverify`, `-inform` og `-out`.
