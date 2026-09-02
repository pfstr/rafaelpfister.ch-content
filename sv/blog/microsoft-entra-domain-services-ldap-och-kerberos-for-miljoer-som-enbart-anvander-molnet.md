---
title: "Microsoft Entra Domain Services: LDAP och Kerberos för molnbaserade miljöer"
navTitle: "Entra Domain Services"
description: "Entra ID talar inte LDAP eller Kerberos. Microsoft Entra Domain Services tillhandahåller en hanterad Active Directory-domän som synkroniserar användare från Entra ID och erbjuder klassiska protokoll. Funktion, begränsningar, kostnader och ett praktiskt exempel med en e-postgateway."
date: "2026-07-30"
kategorie: "Active Directory / Entra"
timeToRead: "8 min lästid"
themen:
  - active-directory-entra
slug: "microsoft-entra-domain-services-ldap-och-kerberos-for-miljoer-som-enbart-anvander-molnet"
translationId: "article-9c271900a94406b8"
draft: false
translationOf: microsoft-entra-domain-services-ldap-kerberos
translationSourceHash: 6360f60ed2e92d286f0e279f487b62a86fa9a987c2f574b0a53af0d31f0d736b
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:21:39.742Z
translationReview: automatic
url: https://rafaelpfister.ch/sv/blog/microsoft-entra-domain-services-ldap-och-kerberos-for-miljoer-som-enbart-anvander-molnet
---

# Microsoft Entra Domain Services: LDAP och Kerberos för molnbaserade miljöer

Den som helt har flyttat sina användare till Microsoft Entra ID (tidigare Azure Active Directory) märker det senast vid den första appliance-lösningen eller äldre applikationen: Entra ID besvarar förfrågningar via Microsoft Graph och moderna autentiseringsprotokoll som OAuth och SAML, men inte via LDAP, Kerberos eller NTLM. En LDAP-bindning mot Entra ID är helt enkelt inte möjlig. För allt som förväntar sig ett klassiskt Active Directory erbjuder Microsoft därför en egen tjänst: Microsoft Entra Domain Services, tidigare Azure AD Domain Services.

## Vad tjänsten tillhandahåller

Entra Domain Services är en hanterad Active Directory-domän. Microsoft driver två Windows-domänkontrollanter i ett Azure-VNet, hanterar patchning, replikering och säkerhetskopiering samt synkroniserar automatiskt användare och grupper från Entra ID till domänen. Synkroniseringen sker endast i en riktning, från Entra ID till den hanterade domänen; ändringar direkt i domänen synkroniseras inte tillbaka.

Utåt fungerar domänen som ett vanligt Active Directory: den besvarar LDAP- och LDAPS-frågor, stöder Kerberos- och NTLM-autentisering, tillåter att virtuella datorer ansluts till domänen och erbjuder begränsade grupprinciper. Applikationer och enheter behöver inte anpassas för detta; de ser en domänkontrollant.

## Vad den behövs för

Tjänsten riktar sig till miljöer som i grunden är helt molnbaserade men som kör enskilda komponenter med klassiska katalogkrav:

- **Appliances och verksamhetsapplikationer med LDAP-anslutning:** Enheter som söker efter användare via LDAP, utvärderar gruppmedlemskap eller kontrollerar inloggningar via LDAP-bindning.
- **Lift-and-shift-migreringar:** Serverarbetslaster som måste förbli domänanslutna (Kerberos, NTLM, domänanslutning), utan att egna domänkontrollanter ska behöva drivas i Azure.
- **Miljöer utan lokalt AD:** Där det aldrig har funnits ett Active Directory eller där det har avvecklats ersätter den hanterade domänen uppbyggnaden av egna DC:er och tillhörande driftarbete.

En viktig avgränsning: Den som fortfarande driver ett lokalt Active Directory med Entra Connect-synkronisering behöver i regel inte tjänsten; appliance-lösningen frågar då det befintliga AD:t. Entra Domain Services fyller luckan när Entra ID är den enda användarkällan.

## Arkitektur och konfiguration

Den hanterade domänen distribueras i ett eget VNet eller undernät och får två fasta adresser till domänkontrollanter. Arbetslaster i andra VNet når den via VNet-peering; DNS-servrarna i de berörda VNet måste peka på domänkontrollanterna så att domännamn och objekt kan lösas upp. Åtkomsten begränsas med Network Security Groups till de källor och portar som faktiskt behövs.

Några särdrag hos den hanterade domänen som är relevanta vid anslutning av applikationer:

- Synkroniserade användare finns i organisationsenheten **AADDC Users**, och domänen har utan egen konfiguration suffixet **onmicrosoft.com**. Search Base och Bind-DN måste återspegla denna struktur, exempelvis CN=svc-ldap,OU=AADDC Users,DC=example,DC=onmicrosoft,DC=com.
- Det finns ingen Domain Administrator. Administrationen sker via den delegerade gruppen AAD DC Administrators; schemautökningar är inte möjliga.
- För LDAP-bindningskonton räcker ett dedikerat, oprivilegierat konto; för rena katalogfrågor i Entra ID krävs rollen Directory Readers.

## Problemet med lösenordshashar

En punkt kostar regelbundet tid vid tester: Kerberos- och NTLM-inloggningar samt LDAP-bindningar behöver lösenordshashar i den hanterade domänen. För konton som endast finns i molnet skapar Entra ID dessa hashar först vid nästa lösenordsändring efter att tjänsten har aktiverats. En nyss synkroniserad användare är alltså synlig i katalogen, men kan inte logga in förrän lösenordet har ändrats en gång. För hybrida konton måste hasharna synkroniseras från det lokala AD:t via Entra Connect.

## Secure LDAP steg för steg

Inom domänen körs LDAP som standard okrypterat över port 389. För inloggningar och all åtkomst utanför strikt isolerade nät ska anslutningen använda Secure LDAP (LDAPS, port 636); tjänsten erbjuder ändå endast krypterad åtkomst utanför VNet. Konfigurationen består av fyra steg.

**1. Skaffa ett certifikat.** Secure LDAP kräver ett eget certifikat som laddas upp som PFX inklusive privat nyckel. Subject eller SAN måste täcka den hanterade domänen med wildcard (exempelvis *.example.onmicrosoft.com), eftersom förfrågningar kan hamna hos någon av de två domänkontrollanterna. En offentlig CA, den egna PKI:n eller ett separat skapat självsignerat certifikat kan användas som källa. Vid självsignering måste certifikatkedjan vara betrodd på varje frågande system; alla appliances tillåter inte detta. Den som kan välja är tryggare med en egen PKI eller en offentlig CA.

**2. Aktivera Secure LDAP.** I portalen under Settings > Secure LDAP aktiveras funktionen och PFX-filen samt lösenordet laddas upp. Där kan åtkomst via internet även aktiveras som tillval; den hanterade domänen får då en offentlig IP-adress.

**3. Nätverk och DNS.** Den externa IP-adressen finns under Properties. Den tillhörande NSG-regeln öppnar TCP/636 och bör begränsas till de käll-IP-adresser som faktiskt behövs, inte till Any. För namnuppslagning pekar en DNS-post (exempelvis ldaps.example.com) på denna IP-adress; den måste stämma överens med certifikatet. Intern åtkomst går fortsatt direkt mot domänkontrollanternas adresser.

**4. Testa anslutningen.** Innan applikationen ställs om är det värt att testa med en LDAP-webbläsare, ldp.exe eller ldapsearch mot port 636: bindning med tjänstekontot, därefter en sökning under organisationsenheten AADDC Users. Först när bindning och sökning fungerar korrekt där är det dags för applikationen.

För att konfigurera själva tjänsten behöver portalkontot rollerna Application Administrator, Domain Services Contributor och Groups Administrator; distributionen av den hanterade domänen tar drygt en timme. I säkerhetsinställningarna kan TLS 1.2 dessutom tvingas som lägstanivå.

## Kostnader

Entra Domain Services är en löpande driftkostnad: Tjänsten debiteras per timme enligt SKU, och dessutom tillkommer VNet, peering och eventuella test-VM:ar. För ett enda litet LDAP-användningsfall är det ett högt pris; alternativet att driva egna domänkontrollanter som VM:ar innebär dock att besparingen vägs upp av ansvar för patchning, säkerhetskopiering och tillgänglighet.

## Praktiskt exempel: E-postgateway med LDAP-anslutning

Ett konkret exempel på appliance-kategorin är SEPPmail Secure E-Mail Gateway. Den använder LDAP för användarupplägg och behörighetsfrågor samt sedan firmware 15.0.6 även för [inloggning till Admin-GUI](/blog/seppmail-admin-gui-ldap-authentifizierung). En appliance i Azure-VNet når den hanterade domänen via VNet-peering med ett dedikerat bindningskonto (Directory Readers), skyddat med NSG:er. Senast för inloggning till Admin-GUI, där TLS-alternativet är aktivt som standard, ska anslutningen använda Secure LDAP.

## Slutsats

Entra Domain Services är inte en ersättning för Entra ID, utan en brygga: tjänsten översätter en molnbaserad användarbas till en klassisk AD-domän för allt som kräver LDAP, Kerberos eller domänanslutning. Den som bara behöver ansluta en enskild applikation bör väga de löpande kostnaderna mot en modernisering av applikationen. När tjänsten väl är på plats fungerar appliances och äldre applikationer som i en välbekant AD-miljö, inklusive de beskrivna särdragen för OU-struktur, behörigheter och lösenordshashar.

## Källor

1.  [Microsoft Learn – «Vad är Microsoft Entra Domain Services?»](https://learn.microsoft.com/de-de/entra/identity/domain-services/overview): Funktionsomfång för den hanterade domänen, protokoll som stöds och avgränsning mot Entra ID och egenhanterade domänkontrollanter.

2.  [Microsoft Learn – «Synkronisering i Entra Domain Services»](https://learn.microsoft.com/de-de/entra/identity/domain-services/synchronization): Enkelriktad synkronisering, OU-struktur och beteendet för lösenordshashar för molnbaserade och hybrida konton.

3.  [Microsoft Learn – «Konfigurera Secure LDAP»](https://learn.microsoft.com/de-de/entra/identity/domain-services/tutorial-configure-ldaps): LDAPS med eget certifikat för krypterad LDAP-åtkomst.

4.  [Anslut SEPPmail Admin-GUI till Active Directory](/blog/seppmail-admin-gui-ldap-authentifizierung): Konfiguration av LDAP-inloggning till Admin-GUI från firmware 15.0.6.
