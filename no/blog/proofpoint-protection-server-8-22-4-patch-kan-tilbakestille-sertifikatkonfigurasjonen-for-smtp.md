---
title: "Proofpoint Protection Server 8.22.4: Patch kan tilbakestille sertifikatkonfigurasjonen for SMTP-klient"
navTitle: "PPS 8.22.4-sertifikat"
description: "Hvis tilordningen «SMTP Client» under System > Certificates > Services i Proofpoint Protection Server står på «-none-», leverer Sendmail uten TLS-klientsertifikat. Motparter med sertifikatkontroll avviser da, for eksempel Exchange Online med 5.7.51 eller en smarthost med «Relay access denied». Slik kontrollerer og løser du det."
date: "2026-11-05"
kategorie: "Proofpoint"
timeToRead: "7 min lesetid"
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
slug: "proofpoint-protection-server-8-22-4-patch-kan-tilbakestille-sertifikatkonfigurasjonen-for-smtp"
translationId: "article-6f99807d289cc394"
warnung: true
featured: "2026-11-05"
translationOf: proofpoint-smtp-client-zertifikat
url: https://rafaelpfister.ch/no/blog/proofpoint-protection-server-8-22-4-patch-kan-tilbakestille-sertifikatkonfigurasjonen-for-smtp
translationSourceHash: a6ce9c6e7e8b62910100ae9ff2323179510735b20326f5b916838ea5df74de02
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:29:01.682Z
translationReview: automatic
---

Jeg undersøkte i dag et utfall i utgående e-postflyt på en Proofpoint Protection Server (PPS) 8.22.4. Kort oppsummert: Etter en automatisk installert patch stod tilordningen **SMTP Client** under **System > Certificates > Services** på `-none-`. Sendmail opprettet deretter utgående forbindelser uten TLS-klientsertifikat. Alle motparter som identifiserer Proofpoint via dette sertifikatet, avviste e-postene. Vanlige mottakere med opportunistisk TLS fortsatte å ta imot, og feilen ble derfor først oppdaget da enkelte partnere ikke lenger kunne nås.

Hvis du kjører PPS med automatiske patcher og har partnere som krever et klientsertifikat, må du kontrollere denne innstillingen etter hver patch.

## Symptom

Bare forbindelser der motparten kontrollerer klientsertifikatet til Proofpoint, er berørt. To typiske tilfeller:

- **Exchange Online med partnerkobling**, som er begrenset til et sertifikat via `RestrictDomainsToCertificate`. Avvisningen lyder `550 5.7.51 TenantInboundAttribution; Rejecting. Recipient has a partner connector with RestrictDomainsToIPAddresses or RestrictDomainsToCertificate set.`
- **Smarthost som bare tillater relay med klientsertifikat.** Svaret lyder `554 5.7.1 <empfaenger@example.com>: Relay access denied`.

Alle andre mål mottar fortsatt e-postene sine. Testmatrisen gir derfor et uensartet bilde: E-post til eksterne mottakere går gjennom, mens e-post til enkelte partnere eller via en bestemt smarthost feiler.

I e-postloggen til agenten står én av de følgende to meldingene ved hver utgående forbindelse. Hvis filen mangler helt:

```text
sendmail[...]: STARTTLS=client: file /etc/pki/tls/certs/sendmail-client.pem unsafe:
  No such file or directory
```

Hvis filen finnes, men er tom:

```text
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_PrivateKey_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_use_certificate_chain_file(/etc/pki/tls/certs/sendmail-client.pem) failed
sendmail[...]: STARTTLS=client, error:
  SSL_CTX_check_private_key failed(/etc/pki/tls/certs/sendmail-client.pem): 0
```

Rett etterpå oppretter Sendmail likevel forbindelsen med TLS (`verify=OK`), men uten klientsertifikat. For motparten ser dette ut som en ukjent avsender.

## Årsak

PPS tilordner hvert tjeneste et sertifikat per server. Under **System > Certificates > Services** finnes blant annet `SMTP Server`, `SMTP Client`, `Administration Server` og `End User Web Application / Encryption` per server. PPS genererer filen `/etc/pki/tls/certs/sendmail-client.pem` fra tilordningen ved `SMTP Client`, som Sendmail bruker som klientsertifikat. Hvis den står på `-none-`, skriver PPS filen tom eller oppretter den ikke i det hele tatt.

I miljøet som ble undersøkt, stod tilordningen på alle servere på `-none-` etter en automatisk patch. Under **System > Config History** var patchen synlig som oppføringen `configuration changes for patch-…` med oppretteren `system`, og filene under `/etc/pki/tls/certs/` hadde samme tidsstempel. Om Proofpoint innførte tilordningen på nytt med denne patchen, eller om patchen tilbakestilte en eksisterende tilordning, er ikke beskrevet i de offentlig tilgjengelige utgivelsesnotatene. Jeg oppdaterer artikkelen så snart Proofpoint uttaler seg om dette.

En rollback under **Config History** til patchens snapshot hjalp ikke. PPS skrev deretter filen på nytt, men med 0 byte.

## Kontrollere

I admin-GUI-en på masteren, under **System > Certificates > Services**, velger du øverst under **Server** etter tur hver agent som leverer utgående e-post, og ser på raden `SMTP Client`.

På selve agenten viser to kommandoer tilstanden:

```bash
ls -l --time-style=full-iso /etc/pki/tls/certs/
grep -E 'sendmail-client.pem|SSL_CTX_' /var/log/maillog | tail -5
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `-l` | Langt format med rettigheter, eier, størrelse og dato |
| `--time-style=full-iso` | Tidsstempel med dato, klokkeslett og tidssone for å sammenligne det med patchtidspunktet |
| `grep -E` | Utvidede regulære uttrykk, slik at begge søkeordene kan kombineres med `\|` |
| `tail -5` | Vis bare de siste fem treffene |

</details>

Hvis `sendmail-client.pem` mangler eller filen har 0 byte, mens `sendmail.pem` er noen tusen byte stor, er det ikke tilordnet noe sertifikat til tjenesten `SMTP Client`.

## Løse problemet

Under **System > Certificates > Services** velger du et gyldig sertifikat for `SMTP Client` på hver berørte agent og lagrer med **Save Changes**. Vanligvis er dette det samme offentlige sertifikatet som står ved `SMTP Server`.

Utvalgslisten viser alle importerte sertifikater, også utløpte med samme navn. De kan bare skilles fra hverandre via serienummeret i parentes. Sammenlign serienummeret med listen under **System > Certificates > Certificates** før du lagrer; der står utstedelses- og utløpsdato.

Etter lagring distribuerer PPS konfigurasjonen til agentene. Kontrollen på agenten:

```bash
ls -l /etc/pki/tls/certs/sendmail-client.pem
grep -E 'STARTTLS=client|stat=' /var/log/maillog | tail -20
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `ls -l` | Viser filens størrelse og rettigheter; etter korrigeringen er den ikke lenger tom |
| `grep -E 'STARTTLS=client\|stat='` | Filtrerer TLS-meldingene for utgående forbindelser og leveringsresultatet |
| `tail -20` | Vis bare de siste tjue linjene |

</details>

Meldingene `SSL_CTX_… failed` og `unsafe` må ikke lenger vises etter lagring, og leveringene til partnerne må ende med `stat=Sent`. E-poster som tidligere ble utsatt med en 4xx-kode, leverer Sendmail selv fra køen. E-poster som ble avvist med en 5xx-kode, må avsenderne sende på nytt.

En manuell kopi av `sendmail.pem` til `sendmail-client.pem` kan også midlertidig løse utfallet. PPS overskriver imidlertid filen ved neste konfigurasjons-push så lenge tilordningen mangler i GUI-en.

## Kontroller etter hver patch

Inntil det er avklart hvorfor patchen endret tilordningen, anbefaler jeg å kontrollere siden **Services** etter hver patch. Enklere er overvåking som varsler ved meldingen `sendmail-client.pem` i e-postloggen på agentene. Under **Licenses and Updates > Settings** kan du også angi når patcher installeres. Et tidsvindu der noen kan kontrollere e-postflyten, forkorter et slikt utfall betydelig.

## Client Authentication forsvinner fra offentlige sertifikater

Uavhengig av patchen påvirker en annen endring nettopp disse forbindelsene. Offentlige TLS-sertifikater har hittil som regel inneholdt begge bruksformålene i utvidelsen `Extended Key Usage`: `TLS Web Server Authentication` og `TLS Web Client Authentication`. E-postgatewayer bruker derfor også serversertifikatet sitt som klientsertifikat.

Chrome Root Program krever at offentlige sertifikater utstedt fra 15.03.2027 bare inneholder `serverAuth`. DigiCert utsteder offentlige TLS-sertifikater som standard uten Client Authentication siden 01.10.2025; på forespørsel er formålet fortsatt mulig frem til 01.03.2027. Andre CA-er har tilsvarende tidsplaner. OpenSSL viser om et sertifikat inneholder klientformålet:

```bash
openssl x509 -in zertifikat.pem -noout \
  -subject -enddate -ext extendedKeyUsage
```

<details class="options-details">
<summary>Alternativer forklart</summary>

| Alternativ | Virkning |
|---|---|
| `x509` | Underkommando for visning og behandling av X.509-sertifikater |
| `-in zertifikat.pem` | Sertifikatfil i PEM-format |
| `-noout` | Ikke skriv ut selve sertifikatet på nytt |
| `-subject` | Vis innehaver (CN og flere felt) |
| `-enddate` | Vis utløpsdato |
| `-ext extendedKeyUsage` | Vis bare utvidelsen med bruksformålene |

</details>

I Windows viser et dobbeltklikk på filen det samme: fanen **Details**, feltet **Enhanced Key Usage**.

Hva dette betyr for utgående forbindelser, avhenger av motparten:

| Motparten kontrollerer | Konsekvens med et sertifikat uten Client Authentication |
|---|---|
| Navn i sertifikatet (f.eks. Exchange Online via `TlsSenderCertificateName`) | Det er ikke dokumentert om Exchange Online kontrollerer bruksformålet; test før sertifikatbyttet |
| Bruksformålet til klientsertifikatet | Avvisning, selv om navn og CA stemmer |
| Fingeravtrykket til klientsertifikatet | Avvisning ved hvert sertifikatbytte, til det nye fingeravtrykket er registrert |

I tillegg kommer den kortere gyldighetstiden for offentlige sertifikater. Etter CA/Browser Forum-vedtaket SC-081v3 reduseres den trinnvis fra 398 dager til 47 dager innen mars 2029, og første trinn med 200 dager gjelder siden mars 2026. Hvert sertifikatbytte er et tidspunkt der en motpart kanskje ikke kjenner det nye sertifikatet.

For forbindelser der partnere identifiserer Proofpoint via et klientsertifikat, anbefaler jeg derfor:

1.  **Kontroller eksisterende sertifikater:** Inneholder dagens gateway-sertifikater fortsatt klientformålet, og når utløper de?

2.  **Planlegg et eget klientsertifikat** som permanent inneholder klientformålet, for eksempel fra en privat PKI eller fra et CA-tilbud for server-til-server-forbindelser. Partnerne må kjenne det.

3.  **Spør partnerne hva de kontrollerer:** Navn, bruksformål, utsteder eller fingeravtrykk. Dette avgjør om et sertifikatbytte innebærer en endring for dem.

4.  **Håndter sertifikatbytter som en endring**, med en liste over alle motparter som må kjenne sertifikatet, og en test per motpart.

## Kilder

1.  [Set-InboundConnector (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/exchange/set-inboundconnector): Parameterne `RestrictDomainsToCertificate` og `TlsSenderCertificateName` for partnerkoblinger i Exchange Online.

2.  [Chrome Root Program Policy](https://googlechrome.github.io/chromerootprogram/crp/policy): Avsnitt 1.3.2 om dedikerte TLS-serverhierarkier; sertifikater fra 15.03.2027 kun med `serverAuth`.

3.  [DigiCert: Removing the client authentication EKU from public TLS certificates](https://knowledge.digicert.com/alerts/sunsetting-client-authentication-eku-from-digicert-public-tls-certificates): Standard uten Client Authentication siden 01.10.2025, på forespørsel frem til 01.03.2027, alternativer for server-til-server-forbindelser.

4.  [GeoCerts Change Log](https://www.geocerts.com/docs/change-log/): Oversikt over utsatte datoer for Client Authentication og gyldighetstrinnene hos DigiCert og Sectigo.

5.  [CA/Browser Forum: Ballot SC081v3](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/): Vedtak om trinnvis forkorting av gyldighetstiden for offentlige TLS-sertifikater frem til mars 2029.
