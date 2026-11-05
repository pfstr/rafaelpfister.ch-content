---
title: "Proofpoint Protection Server 8.22.4: Patch kan återställa certifikatkonfigurationen för SMTP Client"
navTitle: "PPS 8.22.4-certifikat"
description: "Om tilldelningen «SMTP Client» i Proofpoint Protection Server under System > Certificates > Services är inställd på «-none-», levererar Sendmail utan TLS-klientcertifikat. Motparter som kontrollerar certifikat avvisar då, till exempel Exchange Online med 5.7.51 eller en smarthost med «Relay access denied». Så kontrollerar och åtgärdar du detta."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min lästid"
themen:
  - proofpoint
  - smtp-mailflow
produkte:
  - "proofpoint-pps"
  - "exchange-online"
protokolle:
  - "tls"
  - "smtp"
  - "troubleshooting"
slug: "proofpoint-protection-server-8-22-4-patch-kan-aterstalla-certifikatkonfigurationen-for-smtp"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
url: https://rafaelpfister.ch/sv/blog/proofpoint-protection-server-8-22-4-patch-kan-aterstalla-certifikatkonfigurationen-for-smtp
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:28:13.208Z
translationReview: automatic
---

Jag undersökte i dag ett avbrott i utgående e-postflöde på en Proofpoint Protection Server (PPS) 8.22.4. Kort slutsats: Efter en automatiskt installerad patch var tilldelningen **SMTP Client** under **System > Certificates > Services** inställd på `-none-`. Sendmail upprättade därefter utgående anslutningar utan TLS-klientcertifikat. Alla motparter som identifierar Proofpoint via detta certifikat avvisade e-postmeddelandena. Vanliga mottagare med opportunistisk TLS fortsatte att ta emot, så felet upptäcktes först när enskilda partner inte längre kunde nås.

Om du använder PPS med automatiska patchar och har partner som kräver ett klientcertifikat bör du kontrollera denna inställning efter varje patch.

## Symptom

Endast anslutningar där motparten kontrollerar Proofpoints klientcertifikat påverkas. Två typiska fall:

- **Exchange Online med partneranslutning**, som är begränsad till ett certifikat via `RestrictDomainsToCertificate`. Avvisningen lyder `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost som endast tillåter relä med klientcertifikat.** Svaret lyder `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Alla andra mål fortsätter att ta emot sina e-postmeddelanden. Testmatrisen ger därför en ojämn bild: e-post till externa mottagare går igenom, medan e-post till enskilda partner eller via en viss smarthost misslyckas.

I agentens e-postlogg står ett av följande två meddelanden vid varje utgående anslutning. Om filen saknas helt:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Om filen finns men är tom:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Direkt därefter upprättar Sendmail ändå anslutningen med TLS (`verify=OK`), men utan klientcertifikat. För motparten ser detta ut som en okänd avsändare.

## Orsak

PPS tilldelar varje tjänst ett certifikat per server. Under **System > Certificates > Services** visas bland annat `SMTP Server`, `SMTP Client`, `Administration Server` och `End User Web Application / Encryption` per server. Utifrån tilldelningen för `SMTP Client` skapar PPS filen `/etc/pki/tls/certs/sendmail-client.pem`, som Sendmail använder som klientcertifikat. Om den är inställd på `-none-` skriver PPS filen tom eller inte alls.

I den undersökta miljön var tilldelningen på alla servrar inställd på `-none-` efter en automatisk patch. Under **System > Config History** syntes patchen som posten `configuration changes for patch-…` med skaparen `system`, och filerna under `/etc/pki/tls/certs/` hade samma tidsstämpel. Det framgår inte av de offentligt tillgängliga versionsinformationen om Proofpoint införde tilldelningen med denna patch eller om patchen återställde en befintlig tilldelning. Jag uppdaterar artikeln så snart Proofpoint uttalar sig om detta.

En återställning under **Config History** till patchens ögonblicksbild hjälpte inte. PPS skrev sedan om filen, men med 0 byte.

## Kontrollera

I administrationsgränssnittet på mastern går du till **System > Certificates > Services**, väljer varje agent som levererar utgående e-post under **Server** högst upp och granskar raden `SMTP Client`.

På själva agenten visar två kommandon statusen:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `-l` | Långt format med behörigheter, ägare, storlek och datum |
| `--time-style=full-iso` | Tidsstämpel med datum, tid och tidszon för jämförelse med tidpunkten för patchen |
| `grep -E` | Utökade reguljära uttryck så att båda söktermerna kan kombineras med `\|` |
| `tail -5` | Visa endast de fem senaste träffarna |

</details>

Om `sendmail-client.pem` saknas eller filen har 0 byte, medan `sendmail.pem` är flera tusen byte stor, är inget certifikat tilldelat tjänsten `SMTP Client`.

## Åtgärda

Under **System > Certificates > Services** väljer du ett giltigt certifikat för `SMTP Client` på varje berörd agent och sparar med **Save Changes**. Vanligtvis är det samma offentliga certifikat som står vid `SMTP Server`.

Urvalslistan visar alla importerade certifikat, även utgångna certifikat med samma namn. De kan endast skiljas åt genom serienumret inom parentes. Jämför serienumret före sparandet med listan under **System > Certificates > Certificates**, där utfärdande- och utgångsdatum anges.

Efter sparandet distribuerar PPS konfigurationen till agenterna. Kontroll på agenten:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `ls -l` | Visar filens storlek och behörigheter; efter korrigeringen är den inte längre tom |
| `grep -E 'STARTTLS=client\|stat='` | Filtrerar TLS-meddelanden för utgående anslutningar och leveransresultatet |
| `tail -20` | Visa endast de tjugo senaste raderna |

</details>

Meddelandena `SSL_CTX_… failed` och `unsafe` får inte längre visas efter sparandet, och leveranser till partner måste avslutas med `stat=Sent`. E-post som tidigare köats med en 4xx-kod levererar Sendmail själv från kön. E-post som avvisats med en 5xx-kod måste avsändarna skicka på nytt.

Att manuellt kopiera `sendmail.pem` till `sendmail-client.pem` överbryggar också avbrottet. PPS skriver dock över filen vid nästa konfigurationspush så länge tilldelningen saknas i gränssnittet.

## Kontrollera efter varje patch

Tills det är klarlagt varför patchen ändrar tilldelningen rekommenderar jag att sidan **Services** kontrolleras efter varje patch. Ännu enklare är övervakning som larmar vid meddelandet `sendmail-client.pem` i agenternas e-postlogg. Under **Licenses and Updates > Settings** kan du även ange när patchar ska installeras. Ett tidsfönster då någon kan kontrollera e-postflödet förkortar ett sådant avbrott avsevärt.

## Client Authentication försvinner från offentliga certifikat

Oberoende av patchen påverkar en andra förändring just dessa anslutningar. Offentliga TLS-certifikat har hittills oftast innehållit båda användningsändamålen i tillägget `Extended Key Usage`: `TLS Web Server Authentication` och `TLS Web Client Authentication`. E-postgateways använder därför även sitt servercertifikat som klientcertifikat.

Chrome Root Program kräver att offentliga certifikat som utfärdas från och med 15.03.2027 endast innehåller `serverAuth`. DigiCert utfärdar sedan 01.10.2025 offentliga TLS-certifikat utan Client Authentication som standard; på begäran är ändamålet fortfarande möjligt till och med 01.03.2027. Andra CA:er har jämförbara tidsplaner. OpenSSL visar om ett certifikat innehåller klientändamålet:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Alternativ förklaras</summary>

| Alternativ | Effekt |
|---|---|
| `x509` | Underkommando för att visa och bearbeta X.509-certifikat |
| `-in zertifikat.pem` | Certifikatfil i PEM-format |
| `-noout` | Skriv inte ut själva certifikatet igen |
| `-subject` | Visa innehavaren (CN och ytterligare fält) |
| `-enddate` | Visa utgångsdatumet |
| `-ext extendedKeyUsage` | Visa endast tillägget med användningsändamålen |

</details>

I Windows visar ett dubbelklick på filen samma sak: fliken **Details**, fältet **Enhanced Key Usage**.

Vad detta innebär för utgående anslutningar beror på motparten:

| Motparten kontrollerar | Följd med ett certifikat utan Client Authentication |
|---|---|
| Namn i certifikatet (t.ex. Exchange Online via `TlsSenderCertificateName`) | Det är inte dokumenterat om Exchange Online kontrollerar användningsändamålet; testa före certifikatbytet |
| Klientcertifikatets användningsändamål | Avvisning, även om namn och CA stämmer |
| Klientcertifikatets fingeravtryck | Avvisning vid varje certifikatbyte tills det nya fingeravtrycket har lagts in |

Därtill kommer den kortare giltighetstiden för offentliga certifikat. Enligt CA/Browser Forum-beslutet SC-081v3 minskar den stegvis från 398 dagar till 47 dagar fram till mars 2029, och den första nivån med 200 dagar gäller sedan mars 2026. Varje certifikatbyte är en tidpunkt då en motpart kanske inte känner till det nya certifikatet.

För anslutningar där partner identifierar Proofpoint via ett klientcertifikat rekommenderar jag därför:

1.  **Kontrollera befintliga certifikat:** Innehåller dagens gatewaycertifikat fortfarande klientändamålet, och när löper de ut?

2.  **Planera ett eget klientcertifikat** som varaktigt innehåller klientändamålet, exempelvis från en privat PKI eller från ett erbjudande från CA:n för server-till-server-anslutningar. Partnerna måste känna till det.

3.  **Fråga partnerna vad de kontrollerar:** namn, användningsändamål, utfärdare eller fingeravtryck. Detta avgör om ett certifikatbyte innebär en ändring för dem.

4.  **Hantera certifikatbyten som en ändring**, med en lista över alla motparter som måste känna till certifikatet och ett test per motpart.

## Källor

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parametrarna `RestrictDomainsToCertificate` och `TlsSenderCertificateName` för partneranslutningar i Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Avsnitt 1.3.2 om dedikerade TLS-serverhierarkier; certifikat från och med 15.03.2027 endast med `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Standard utan Client Authentication sedan 01.10.2025, på begäran till och med 01.03.2027, alternativ för server-till-server-anslutningar.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Översikt över de uppskjutna datumen för Client Authentication och giltighetsstegen hos DigiCert och Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Beslut om stegvis förkortning av giltighetstiden för offentliga TLS-certifikat fram till mars 2029.
