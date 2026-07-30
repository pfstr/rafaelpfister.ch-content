---
title: "Microsoft Entra Domain Services: LDAP og Kerberos for skybaserte miljøer"
navTitle: "Entra Domain Services"
description: "Entra ID snakker ikke LDAP eller Kerberos. Microsoft Entra Domain Services tilbyr et administrert Active Directory-domene som synkroniserer brukere fra Entra ID og tilbyr klassiske protokoller. Funksjon, begrensninger, kostnader og et praktisk eksempel med en e-postgateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min lesetid"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-og-kerberos-for-skybaserte-miljoer"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
url: https://rafaelpfister.ch/no/blog/microsoft-entra-domain-services-ldap-og-kerberos-for-skybaserte-miljoer
translationSourceHash: 00f01b9fa1426d692146e27b2e15e6926e04ea3cccd4855bd0b18c8c10e36e0d
translationModel: gpt-5.6-terra
translatedAt: 2026-07-30T12:23:23.103Z
translationReview: automatic
---

# Microsoft Entra Domain Services: LDAP og Kerberos for skybaserte miljøer

De som har flyttet brukerne sine fullt og helt til Microsoft Entra ID (tidligere Azure Active Directory), merker det senest ved den første appliance-en eller eldre applikasjonen: Entra ID besvarer forespørsler via Microsoft Graph og moderne autentiseringsprotokoller som OAuth og SAML, men ikke via LDAP, Kerberos eller NTLM. En LDAP-bind mot Entra ID er rett og slett ikke mulig. For alt som forventer et klassisk Active Directory, tilbyr Microsoft derfor en egen tjeneste: Microsoft Entra Domain Services, tidligere Azure AD Domain Services.

## Hva tjenesten tilbyr

Entra Domain Services er et administrert Active Directory-domene. Microsoft drifter to Windows-domenecontrollere i et Azure-VNet, tar seg av patching, replikering og sikkerhetskopier, og synkroniserer brukere og grupper automatisk fra Entra ID til domenet. Synkroniseringen går bare én vei, fra Entra ID til det administrerte domenet; endringer direkte i domenet flyter ikke tilbake.

Utad oppfører domenet seg som et vanlig Active Directory: Det besvarer LDAP- og LDAPS-forespørsler, støtter Kerberos- og NTLM-autentisering, tillater at VM-er blir med i domenet og tilbyr begrensede gruppepolicyer. Applikasjoner og enheter trenger ikke tilpasses; de ser en Domain Controller.

## Hva man trenger det til

Tjenesten er rettet mot miljøer som egentlig er skybaserte, men som drifter enkelte komponenter med klassiske katalogkrav:

- **Appliances og fagapplikasjoner med LDAP-integrasjon:** Enheter som søker etter brukere via LDAP, evaluerer gruppemedlemskap eller kontrollerer innlogginger via LDAP-bind.
- **Lift-and-shift-migreringer:** Serverarbeidslaster som må forbli domenetilknyttet (Kerberos, NTLM, domenetilknytning), uten at egne Domain Controllers skal driftes i Azure.
- **Miljøer uten lokalt AD:** Der det aldri har vært et Active Directory, eller der det er avviklet, erstatter det administrerte domenet oppsettet av egne DC-er med tilhørende driftsarbeid.

Viktig avgrensning: De som fortsatt drifter et lokalt Active Directory med Entra Connect-synkronisering, trenger som regel ikke tjenesten; da spør appliance-en det eksisterende AD-et. Entra Domain Services fyller gapet når Entra ID er den eneste brukerkilden.

## Arkitektur og oppsett

Det administrerte domenet distribueres i et eget VNet eller subnett og får to faste Domain Controller-adresser. Arbeidslaster i andre VNet-er når det via VNet-peering; DNS-serverne i de involverte VNet-ene må peke mot Domain Controllers slik at domenenavn og objekter kan løses opp. Tilgang begrenses med Network Security Groups til de faktisk nødvendige kildene og portene.

Noen særtrekk ved det administrerte domenet som er relevante ved tilkobling av applikasjoner:

- Synkroniserte brukere ligger i OU-en **AADDC Users**, og domenet har uten egen konfigurasjon suffikset **onmicrosoft.com**. Search Base og Bind-DN-er må gjenspeile denne strukturen, for eksempel CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Det finnes ingen Domain Administrator. Administrasjonen skjer via den delegerte gruppen AAD DC Administrators; skjemautvidelser er ikke mulig.
- For LDAP-bind-kontoer holder det med en dedikert konto uten privilegier; for rene katalogoppslag i Entra ID rollen Directory Readers.

## Fellen med passordhasher

Ett punkt tar jevnlig tid i tester: Kerberos- og NTLM-innlogginger samt LDAP-bind trenger passordhasher i det administrerte domenet. For cloud-only-kontoer oppretter Entra ID disse hashene først ved neste passordendring etter at tjenesten er aktivert. En nylig synkronisert bruker er altså synlig i katalogen, men kan først logge på etter å ha endret passordet én gang. For hybride kontoer må hashene synkroniseres fra det lokale AD-et via Entra Connect.

## Secure LDAP trinn for trinn

Innenfor domenet kjører LDAP som standard ukryptert over port 389. For innlogginger og all tilgang utenfor strengt isolerte nettverk skal forbindelsen bruke Secure LDAP (LDAPS, port 636); tjenesten tilbyr uansett kun kryptert tilgang utenfra VNet-et. Oppsettet består av fire trinn.

**1. Skaff sertifikat.** Secure LDAP trenger et eget sertifikat som lastes opp som PFX, inkludert privat nøkkel. Subject eller SAN må dekke det administrerte domenet med wildcard (for eksempel *.example.onmicrosoft.com), siden forespørsler kan havne hos hver av de to Domain Controllers. En offentlig CA, egen PKI eller et selvopprettet selvsignert sertifikat kan brukes som kilde. Med selvsignert sertifikat må kjeden være installert som klarert på hvert system som gjør forespørsler; ikke alle appliances tillater dette. De som kan velge, er tryggere med egen PKI eller en offentlig CA.

**2. Aktiver Secure LDAP.** I portalen under Settings > Secure LDAP aktiveres funksjonen, og PFX-filen samt passordet lastes opp. Valgfritt kan tilgang via Internett aktiveres der; det administrerte domenet får da en offentlig IP-adresse.

**3. Nettverk og DNS.** Den eksterne IP-adressen finnes under Properties. Den tilhørende NSG-regelen åpner TCP/636 og bør begrenses til de faktisk nødvendige kilde-IP-adressene, ikke Any. For navneoppløsning peker en DNS-oppføring (for eksempel ldaps.example.com) mot denne IP-adressen; den må samsvare med sertifikatet. Intern tilgang går fortsatt direkte mot Domain Controller-adressene.

**4. Test forbindelsen.** Før applikasjonen legges om, er det verdt å teste med en LDAP-nettleser, ldp.exe eller ldapsearch mot port 636: Bind med tjenestekontoen, deretter et søk under OU-en AADDC Users. Først når bind og søk fungerer korrekt der, er det applikasjonens tur.

For å sette opp selve tjenesten trenger portalkontoen rollene Application Administrator, Domain Services Contributor og Groups Administrator; distribusjonen av det administrerte domenet tar godt over en time. I sikkerhetsinnstillingene kan TLS 1.2 også håndheves som minimum.

## Kostnader

Entra Domain Services er en løpende driftskostnad: Tjenesten faktureres per time etter SKU, i tillegg kommer VNet, peering og eventuelle test-VM-er. For det ene lille LDAP-bruksområdet er det en høy pris; alternativet, å drifte egne Domain Controllers som VM-er, kjøper imidlertid besparelsen tilbake med ansvar for patching, sikkerhetskopiering og tilgjengelighet.

## Praktisk eksempel: E-postgateway med LDAP-integrasjon

Et konkret eksempel på appliance-kategorien er SEPPmail Secure E-Mail Gateway. Den bruker LDAP til brukeropprettelse og tilgangsforespørsler, og siden firmware 15.0.6 også til [innlogging i Admin-GUI](/blog/seppmail-admin-gui-ldap-authentifizierung). En appliance i Azure-VNet-et når det administrerte domenet via VNet-peering med en dedikert bind-konto (Directory Readers), sikret med NSG-er. Senest for innlogging i Admin-GUI-en, der TLS-alternativet er aktivert som standard, skal forbindelsen bruke Secure LDAP.

## Konklusjon

Entra Domain Services er ikke en erstatning for Entra ID, men en bro: Tjenesten oversetter en skybasert brukerbase til et klassisk AD-domene for alt som krever LDAP, Kerberos eller domenetilknytning. De som bare må koble til én enkelt applikasjon, bør veie de løpende kostnadene opp mot en modernisering av applikasjonen. Når tjenesten først er på plass, oppfører appliances og eldre applikasjoner seg som i et velkjent AD-miljø, inkludert de beskrevne særtrekkene ved OU-struktur, tillatelser og passordhasher.

## Kilder

1.  [Microsoft Learn – «Hva er Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Funksjonsomfanget til det administrerte domenet, støttede protokoller og avgrensning mot Entra ID og selvdriftede Domain Controllers.

2.  [Microsoft Learn – «Synkronisering i Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): Enveis synkronisering, OU-struktur og oppførselen til passordhashene for cloud-only- og hybride kontoer.

3.  [Microsoft Learn – «Konfigurere Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS med eget sertifikat for kryptert LDAP-tilgang.

4.  [Koble SEPPmail Admin-GUI til Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): Oppsett av LDAP-innlogging i Admin-GUI fra firmware 15.0.6.
