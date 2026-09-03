---
title: "Nya Outlook: S/MIME-signatur kan inte verifieras i sekundärt konto, bilagor saknas"
navTitle: "S/MIME i sekundärt konto"
description: "Nya Outlook meddelar i en delad postlåda att S/MIME-signaturen inte kan verifieras i det sekundära kontot och visar inga bilagor. Artikeln förklarar skillnaden mellan Clear Signing och Opaque Signing, varför bilagorna försvinner i ogenomskinligt signerade e-postmeddelanden, varför nya Outlook bara behandlar S/MIME i primärkontot och vilka lösningar som finns, inklusive uppackning av smime.p7m med PowerShell eller OpenSSL."
date: "2026-09-03"
kategorie: "Outlook"
timeToRead: "8 min lästid"
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
slug: "nya-outlook-s-mime-signatur-kan-inte-verifieras-i-sekundart-konto-bilagor-saknas"
translationId: "article-f1e9d4ab5be67349"
aiPrompt: |
  Du bist mein Messaging-Assistent. Hilf mir, das Problem "S/MIME-Signatur kann im sekundären Konto nicht überprüft werden" in Outlook einzuordnen: Prüfe anhand der Nachrichtenquelle, ob die Mail clear-signed (multipart/signed) oder opaque-signed (application/pkcs7-mime) ist, erkläre mir, warum die Anhänge fehlen, und führe mich zu einem Ausweg (Postfach als eigenes Konto, klassisches Outlook, Outlook im Web oder Auspacken der smime.p7m mit PowerShell oder OpenSSL).
translationOf: outlook-smime-sekundaeres-konto-anhaenge-fehlen
url: https://rafaelpfister.ch/sv/blog/nya-outlook-s-mime-signatur-kan-inte-verifieras-i-sekundart-konto-bilagor-saknas
translationSourceHash: ee167a56424fa3ffe1d8e79c62a748cd68c7864d7a95d3d9fdc8989a33cd4283
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:13:38.489Z
translationReview: automatic
---

# Nya Outlook: S/MIME-signatur kan inte verifieras i sekundärt konto, bilagor saknas

I nya Outlook för Windows visas en röd rad när du öppnar ett digitalt signerat e-postmeddelande i en delad postlåda: "S/MIME-signaturen kan inte verifieras vid visning i det sekundära kontot." Själva meddelandet visas, men inte bilagorna, trots att avsändaren har skickat med sådana. Kollegor som använder samma postlåda som primärkonto ser bilagorna utan problem.

Det beror på två saker som förstärker varandra: nya Outlook behandlar S/MIME endast i primärkontot, och avsändaren har signerat meddelandet ogenomskinligt. Med denna signaturform ligger hela innehållet, inklusive bilagor, i en enda kryptografisk behållare. Om klienten inte kan öppna behållaren förblir bilagorna osynliga. Båda problemen kan lösas var för sig.

## Vad meddelandet betyder

"Sekundärt konto" betyder i nya Outlook varje postlåda som inte är det konto som du har loggat in med. Det gäller delade postlådor (Shared Mailboxes) som visas via fullständig åtkomst och automapping, liksom postlådor som du har anslutit via "Lägg till delad postlåda", samt delegeringar. S/MIME-behandlingen i nya Outlook är fast knuten till primärkontot. Om ett signerat meddelande kommer in i ett annat konto kontrollerar klienten inte signaturen utan visar i stället meddelandet.

Detta säger ingenting om signaturens giltighet och är inte ett certifikatproblem hos avsändaren. Samma e-postmeddelande kan kontrolleras och öppnas i primärkontot eller i klassiska Outlook.

## Clear Signing och Opaque Signing

S/MIME-standarden (RFC 8551) har två format för signerade meddelanden. Båda ger samma signatur, men paketerar meddelandet på olika sätt.

| | Clear Signing | Opaque Signing |
|---|---|---|
| MIME-typ | `multipart/signed` med `protocol="application/pkcs7-signature"` | `application/pkcs7-mime` med `smime-type=signed-data` |
| Struktur | Två delar: den läsbara meddelandetexten med bilagor och den separata signaturen bredvid | En del: meddelandetext, bilagor och signatur tillsammans i en CMS-SignedData-behållare, Base64-kodad |
| Bilaga som en klient utan S/MIME ser | `smime.p7s` (endast signaturen, några få KB) | `smime.p7m` (hela meddelandet) |
| Läsbart utan S/MIME-stöd | Ja, text och bilagor visas normalt | Nej, klienten ser endast behållaren |
| Känslighet under överföringen | Signaturen blir ogiltig om en e-postserver eller gateway ändrar radbrytningar, kodning eller blanksteg | Behållaren skyddar innehållet mot sådana ändringar |
| RFC 8551-avsnitt | 3.5.3 | 3.5.2 |

I meddelandekällan känner du igen de två formaten på rubriken `Content-Type`. Ett clear-signed e-postmeddelande börjar så här:

```text
Content-Type: multipart/signed; protocol="application/pkcs7-signature";
    micalg=sha-256; boundary="----=_Part_4711"
```

Ett opaque-signed e-postmeddelande så här:

```text
Content-Type: application/pkcs7-mime; smime-type=signed-data;
    name="smime.p7m"
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="smime.p7m"
```

Skillnaden förklarar helt beteendet i nya Outlook. Vid ett clear-signed e-postmeddelande visar klienten text och bilagor även om den inte kontrollerar signaturen; endast signaturstatusen saknas. Vid ett opaque-signed e-postmeddelande måste klienten först packa upp behållaren via S/MIME-behandlingen för att komma åt text och bilagor. Om den vägrar eftersom meddelandet ligger i ett sekundärt konto förblir behållaren stängd. Att texten ändå är läsbar beror på Exchange Online: tjänsten återger textdelen på serversidan, men inte bilagorna från behållaren.

Inget av formaten krypterar något. Även den ogenomskinliga varianten är bara Base64-kodad och läsbar för alla som får tag i meddelandet. Microsoft påpekar detta uttryckligen i dokumentationen för Exchange Online.

## Vilket format avsändaren väljer

I klassiska Outlook styr alternativet "Skicka signerade meddelanden som klartext" (Arkiv > Alternativ > Säkerhetscenter > E-postsäkerhet) formatet. Det är aktiverat som standard, vilket innebär att Outlook signerar clear-signed. Den som inaktiverar alternativet skickar ogenomskinligt. Nya Outlook och Outlook på webben erbjuder inte denna inställning.

E-postgatewayer som signerar centralt har en egen inställning för signaturformatet. Vissa produkter signerar ogenomskinligt som standard av robusthetsskäl, eftersom signaturen då förblir giltig även efter att efterföljande system har gjort ändringar. Om du regelbundet får e-postmeddelanden med saknade bilagor från en viss avsändare är det värt att kontrollera dess gateway-konfiguration.

## Varför nya Outlook bara behandlar S/MIME i primärkontot

Microsoft dokumenterar begränsningen, men anger ingen teknisk orsak. Orsaken framgår av klientens arkitektur.

Nya Outlook är i grunden webbklienten Outlook på webben i ett inbyggt skal. I webbläsaren får JavaScript inte åtkomst till Windows certifikatarkiv. Därför behövde Outlook på webben under många år ett separat S/MIME-webbläsartillägg. Nya Outlook ersätter detta tillägg med en inbyggd brygga mellan webbgränssnittet och Windows-kryptografi. Bryggan initieras när du loggar in på primärkontot och får dess postlådesession, certifikat och S/MIME-inställningar från Inställningar > E-post > S/MIME.

Delade postlådor och sekundära konton använder andra datavägar. Sekundära konton har egna sessioner, medan delade postlådor läses in via delegeringen från primärkontot. Bryggan har hittills inte anslutits för dessa vägar. Detta gäller även ren kontroll av en signatur, trots att någon privat nyckel inte skulle behövas: tolkningen och uppackningen av PKCS#7-strukturen sker via samma komponent.

I klassiska Outlook uppstår inte problemet eftersom S/MIME-behandlingen där sker i MAPI-stacken per meddelande, oberoende av från vilket Store meddelandet kommer.

Microsoft lägger till den saknade anslutningen: Roadmap-posten 565861 utökar S/MIME i nya Outlook till delade och delegerade postlådor som är kopplade till primärkontot. Allmän tillgänglighet är annonserad för juli 2026 och utrullningen sker stegvis. Om du fortfarande ser meddelandet har ändringen ännu inte nått din tenant eller klientversion. Separat tillagda sekundära konton med egen inloggning omfattas inte av posten.

## Lösningar

Vilken väg som passar beror på hur postlådan är ansluten och om du behöver kontrollera signaturen eller bara vill komma åt bilagorna.

| Väg | Förutsättning | Resultat |
|---|---|---|
| Öppna e-postmeddelandet i primärkontot | Du har själv postlådan som primärkonto eller e-postmeddelandet har vidarebefordrats till dig | Signaturkontroll och bilagor |
| Lägg till postlådan som ett separat konto i nya Outlook (Inställningar > Konton > Lägg till konto) | Postlådan har egna inloggningsuppgifter; inte möjligt för delade postlådor utan lösenord | Signaturkontroll och bilagor så snart du växlar till detta konto |
| Klassiska Outlook | Fortfarande installerat eller möjligt att växla tillbaka via reglaget "Nya Outlook"; anslut postlådan där som ett separat konto (Arkiv > Kontoinställningar > Nytt) | Signaturkontroll och bilagor i varje Store |
| Outlook på webben | Öppna postlådan direkt (`outlook.office.com/mail/<adresse>`), S/MIME-tillägg för Edge eller Chrome installerat | Signaturkontroll och bilagor |
| Be avsändaren om Clear Signing | Avsändaren använder klassiska Outlook eller en gateway med valbart format | Bilagor synliga, men signaturstatusen i sekundärkontot saknas fortfarande |
| Packa upp behållaren manuellt | Spara e-postmeddelandet som `.eml` eller spara `smime.p7m` | Bilagor utan signaturkontroll |

## Packa upp behållaren manuellt

För enstaka fall räcker det att öppna behållaren utanför Outlook. Signaturen kontrolleras då matematiskt, men certifikatets förtroendekedja kontrolleras inte. Spara meddelandet som `.eml` eller spara bilagan `smime.p7m` i en mapp.

I Windows räcker PowerShell. .NET Framework innehåller klassen `SignedCms`, som läser PKCS#7-behållaren:

```powershell
Add-Type -AssemblyName System.Security
$bytes = [IO.File]::ReadAllBytes("C:\Temp\smime.p7m")
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($bytes)
$cms.CheckSignature($true)
[IO.File]::WriteAllBytes("C:\Temp\inhalt.eml", $cms.ContentInfo.Content)
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Instruktion | Effekt |
|---|---|
| `Add-Type -AssemblyName System.Security` | Läser in assemblyn med PKCS-klasserna (krävs i Windows PowerShell 5.1, redan inläst i PowerShell 7) |
| `[IO.File]::ReadAllBytes(...)` | Läser den binära DER-behållaren; `smime.p7m` som sparats från Outlook är redan avkodad |
| `$cms.Decode($bytes)` | Tolkar CMS-SignedData-strukturen |
| `$cms.CheckSignature($true)` | Kontrollerar endast signaturen över innehållet (`$true` = verifySignatureOnly); med `$false` skulle även certifikatkedjan kontrolleras. Vid ogiltig signatur avbryts kommandot med ett undantag |
| `$cms.ContentInfo.Content` | Det signerade innehållet: ett fullständigt MIME-meddelande med text och bilagor |
| `[IO.File]::WriteAllBytes(...)` | Skriver detta MIME-meddelande som `.eml`, som du kan öppna i Outlook eller Thunderbird |

</details>

I Linux, macOS eller med Git för Windows finns OpenSSL tillgängligt. Om hela e-postmeddelandet finns som `.eml` utför OpenSSL även Base64-avkodningen:

```bash
openssl cms -verify -noverify \
  -in nachricht.eml \
  -out inhalt.eml
```

<details class="options-details">
<summary>Förklarade alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `cms` | CMS-verktyget från OpenSSL; `smime` fungerar likvärdigt, `cms` är det nyare gränssnittet |
| `-verify` | Kontrollerar signaturen och skriver ut det signerade innehållet |
| `-noverify` | Hoppar över kontrollen av certifikatkedjan; själva signaturen kontrolleras ändå |
| `-in nachricht.eml` | Hela e-postmeddelandet i S/MIME-format (Base64 med MIME-rubriker); för en sparad `smime.p7m` dessutom `-inform DER` |
| `-out inhalt.eml` | Det uppackade innehållet som MIME-meddelande |

</details>

Filen `inhalt.eml` innehåller den ursprungliga meddelandetexten och alla bilagor som vanliga MIME-delar. Ett dubbelklick öppnar den i Outlook, där du kan spara bilagorna som vanligt.

## Källor

1.  [s/mime sign cannot be verified when viewing in secondary account (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5781907/s-mime-sign-cannot-be-verified-when-viewing-in-sec): Fallet från praktiken med samma meddelande i den delade postlådan; svaret bekräftar att beteendet är känt och nämner ingen lösning i nya Outlook.

2.  [RFC 8551: Secure/Multipurpose Internet Mail Extensions (S/MIME) Version 4.0 Message Specification](https://www.rfc-editor.org/rfc/rfc8551.html): Avsnitten 3.5.2 (application/pkcs7-mime med SignedData) och 3.5.3 (multipart/signed) med uppgifter om läsbarhet utan S/MIME och robusthet under överföringen.

3.  [Secure messages with a digital ID in Outlook (Microsoft Support)](https://support.microsoft.com/en-us/office/secure-messages-with-a-digital-id-in-outlook-549ca2f1-a68f-4366-85fa-b3f4b5856fc6): Alternativet "Skicka signerade meddelanden som klartext" i klassiska Outlook, aktiverat som standard; saknas i nya Outlook.

4.  [Set up Outlook to use S/MIME encryption (Microsoft Support)](https://support.microsoft.com/en-us/outlook/mail/set-up-outlook-to-use-s-mime-encryption): S/MIME-inställningar i nya Outlook under Inställningar > E-post > S/MIME; certifikat måste installeras manuellt eller via princip.

5.  [S/MIME in Exchange Online (Microsoft Learn)](https://learn.microsoft.com/en-us/exchange/security-and-compliance/smime-exo/smime-exo): Information om att ogenomskinligt signerade meddelanden endast är Base64-kodade och inte konfidentiella.

6.  [Microsoft 365 Roadmap, post 565861](https://www.microsoft.com/microsoft-365/roadmap?id=565861): S/MIME för delade och delegerade postlådor i nya Outlook för Windows, annonserat för juli 2026.

7.  [Accounts in the new Outlook for Windows (Microsoft Learn)](https://learn.microsoft.com/en-us/deployoffice/outlook/get-started/supported-account-types): Vilka kontotyper nya Outlook stöder och hur delade postlådor ansluts.

8.  [SignedCms Class (.NET API Reference)](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.pkcs.signedcms): Decode, CheckSignature och ContentInfo för att packa upp behållaren med PowerShell.

9.  [openssl-cms (OpenSSL Manpage)](https://www.openssl.org/docs/man3.0/man1/openssl-cms.html): Alternativen `-verify`, `-noverify`, `-inform` och `-out`.
