---
title: "Kerberos: biljetter, nycklar och betrodda tjänster"
blatt: "kerberos"
description: "Kerberos för administratörer: KDC, AS-/TGS-/AP-flöde, principaler och SPN:er, biljetter, sessionsnycklar, keytabs och KVNO, GSS-API, Active Directory och PAC, förtroenderelationer, delegering, kryptering, DNS, tid, drift och diagnostik."
fakten:
  - label: Uppgift
    wert: Nätverksautentisering via biljetter
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.1
  - label: Protokollversion
    wert: Kerberos V5 · RFC 4120
    href: https://datatracker.ietf.org/doc/html/rfc4120
  - label: Betrodd instans
    wert: KDC med Authentication Service och TGS
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-1.2
  - label: Utbyten
    wert: AS · TGS · klient/server-AP
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3
  - label: KDC-port
    wert: 88 över UDP och TCP
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos
  - label: Lösenordstjänst
    wert: kpasswd · port 464
    href: https://datatracker.ietf.org/doc/html/rfc3244
  - label: Tjänstnamn
    wert: service/host@REALM
    href: https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html
  - label: Tjänstnyckel
    wert: Principal · KVNO · Enctype · Key
    href: https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html
  - label: Tidsfönster
    wert: Clock Skew är realm- och klientpolicy
    href: https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3
  - label: Program-API
    wert: GSS-API / SSPI · ofta SPNEGO
    href: https://datatracker.ietf.org/doc/html/rfc4121
  - label: Active Directory
    wert: Domänkontrollanter integrerar KDC
    href: https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview
  - label: AD-auktorisering
    wert: PAC som Ticket-Authorization-Data
    href: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962
werbung:
  - newsletter
ctaThemen:
  - active-directory-entra
translationSourceHash: b5eccfbd21dfaa9b065424961e28940ca63617bba9dbb9f5ab68c1db128df31f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:14:33.079Z
translationReview: required
---

# Kerberos: biljetter, nycklar och betrodda tjänster

Kerberos gör det möjligt för en användare eller process att identifiera sig mot en nätverkstjänst utan att skicka lösenordet till tjänsten. Klient och server förlitar sig då på ett Key Distribution Center (KDC), som utfärdar tidsbegränsade biljetter och sessionsnycklar. Vid lösenordsbaserad inloggning härleds dock fortfarande en långsiktig nyckel från lösenordet. Lösenordskvalitet, pre-authentication och skyddet av klienten förblir därför säkerhetsrelevanta ([RFC 4120, avsnitt 1.1 och 1.2](https://datatracker.ietf.org/doc/html/rfc4120#section-1.1), [RFC 3961, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc3961#section-3)).

För meddelande- och katalogadministratörer ligger Kerberos ofta bredvid [LDAP](/kb/ldap), inte i stället för det. LDAP transporterar katalogåtgärder; Kerberos kan tillhandahålla identiteten för en LDAP-session via SASL. Webbgränssnitt använder HTTP Negotiate, Windows-tjänster SSPI och Unix-program vanligtvis GSS-API. Ett fel kan därför uppstå i DNS, tid, KDC-nåbarhet, realm-mappning, biljettcache, SPN, tjänstkonto, keytab, krypteringstyp, PAC eller delegeringspolicy, även om programmet bara rapporterar «Integrated Authentication failed».

Förklaringen följer en principal från den första kontakten med KDC via TGT och servicebiljett till måltjänsten. Först därefter fördjupas Active Directory-utökningar, delegering, tidsberoende, diagnostik och återställning.

## Arkitekturprincip: betrodd tredje part

Kerberos distribuerar symmetriska nycklar via en betrodd instans. KDC har åtkomst till de långsiktiga nycklarna för principalerna i sin realm och förenar logiskt två tjänster: Authentication Service, AS, utfärdar en Ticket-Granting Ticket, TGT; Ticket-Granting Service, TGS, byter detta TGT mot en biljett för en specifik programtjänst. Klient/server-utbytet, AP, sker därefter mellan klient och tjänst. En tjänst kan normalt kontrollera en servicebiljett med sin egen nyckel utan att anropa KDC på nytt vid varje inloggning ([RFC 4120, avsnitt 1.2 och 3](https://datatracker.ietf.org/doc/html/rfc4120#section-1.2)).

Denna modell minskar vidarebefordran av lösenord och centrala onlinekontroller, men skapar tydliga felområden. Utan en nåbar KDC kan nya biljetter inte utfärdas; redan befintliga biljetter kan fortsätta fungera tills deras giltighet löper ut. En komprometterad KDC-databas eller realm-nyckel äventyrar däremot förtroendekedjan för hela realmen. Kerberos är därför inte en tillståndslös plattform för tokensignering, utan ett distribuerat system av KDC-tillstånd, klientcachar, tjänstnycklar, klockor och namntjänster.

| Nivå | Teknisk komponent | Ansvarigt tillstånd | Administrativt bevis |
|---|---|---|---|
| Program | HTTP, SMB, LDAP, databas, SMTP/IMAP med SASL eller proprietär tjänst | Programmets session och auktorisering | faktiskt valt autentiseringsförfarande och målnamn |
| Integrations-API | GSS-API, Windows SSPI och ofta SPNEGO | Security Context, delegering och Channel Binding | förhandlad mekanism, initiator, acceptor och flaggor |
| Kerberos AP | `KRB_AP_REQ`, valfritt `KRB_AP_REP`, vid behov GSS Wrap/MIC | Servicebiljett, authenticator och sessionsnyckel | målprincipal, biljetttider, enctype och ömsesidig autentisering |
| Kerberos KDC | AS- och TGS-utbyte | Principal-databas, realm-nycklar, policyer och biljettflaggor | KDC-resultat, vald nyckel, KVNO och granskningshändelse |
| Discovery och transport | DNS SRV, UDP/TCP 88, lösenordstjänst 464 | Resolvercache, realm-mappning, routning och paketstorlek | faktiskt vald KDC, transport, svarstid och fallback |
| Persistens | AD DS eller Kerberos-databas, keytabs, credential- och replay-cachar | långsiktiga nycklar, biljetter, PAC, replikering och replaystatus | replikeringsstatus, cacheinnehåll, filbehörigheter och återställningsförfarande |

Kerberos autentiserar principaler och kan tillhandahålla integritet eller sekretess för en GSS-säkerhetskontext. Det krypterar inte automatiskt hela programmets nyttodatastream och erbjuder inte Perfect Forward Secrecy för de sessionsnycklar som distribueras i Kerberos-kärnan. Program kan använda Kerberos för att autentisera en separat skyddad kanal; [TLS](/kb/tls) förblir därför ett eget lager med egen serveridentitet och certifikatvalidering vid HTTPS eller LDAPS ([RFC 4120, avsnitt 10](https://datatracker.ietf.org/doc/html/rfc4120#section-10), [RFC 4121, avsnitt 2 och 4](https://datatracker.ietf.org/doc/html/rfc4121#section-2)).

## Principaler, realmer och nyckelmaterial

En Kerberos-principal är ett namn inom en realm. Användare representeras ofta som `alice@EXAMPLE.CH`, tjänster som `HTTP/intranet.example.ch@EXAMPLE.CH`. Delen före `@` kan innehålla flera komponenter; stora och små bokstäver är i princip betydelsefulla i Kerberos namnmodell. Ett DNS-domännamn och en Kerberos-realm är olika namnrymder, även om realmer vanligen ser ut som DNS-domäner skrivna med versaler. Klienter behöver därför en verifierbar mappning från värdnamn eller DNS-domän till realm ([RFC 4120, avsnitt 6.1 och 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-6.1), [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html#mapping-hostnames-onto-kerberos-realms)).

Långsiktiga nycklar tillhör principaler. Sessionsnycklar skapas för ett begränsat utbyte. En biljett innehåller bland annat klient, server, realm, tidsfält, flaggor, sessionsnyckel och Authorization Data; dess krypterade del skyddas med en nyckel för måltjänsten. Klienten får samma sessionsnyckel i en svarsdelsom är skyddad för klienten. Den kan därför transportera biljettinnehållet, men inte själv ändra den del som är krypterad för tjänsten ([RFC 4120, avsnitt 5.3 och 5.4](https://datatracker.ietf.org/doc/html/rfc4120#section-5.3)).

Key Version Number, KVNO, skiljer generationer av en principal-nyckel åt. Encryption Type, Enctype, fastställer algoritm, nyckellängd, string-to-key och kontrollsumma. En tjänst kan hålla flera keytab-poster för samma principal med olika KVNO:er eller enctypes för att klara en kontrollerad rotation. Om biljettens KVNO och enctype inte matchar någon tillgänglig tjänstnyckel kan tjänsten inte dekryptera biljetten. Att «SPN finns» bevisar därför ännu inte att en passande nyckel finns på målsystemet ([RFC 4120, avsnitt 5.2.9](https://datatracker.ietf.org/doc/html/rfc4120#section-5.2.9), [MIT Kerberos: Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html)).

| Objekt | Plats | Konfidentialitet | Driftgräns |
|---|---|---|---|
| långsiktig principal-nyckel | KDC-databas; hos tjänsten dessutom keytab eller operativsystemets nyckellager | mycket kritisk | rotation, replikering, KVNO och tillåtna enctypes |
| TGT | Klientens credential cache; krypterad biljettdel för `krbtgt/REALM` | nära bearer-token plus sessionsnyckel | lifetime, Forwardable-/Renewable-flaggor, cacheisolering |
| Servicebiljett | Credential cache och därefter hos måltjänsten | kan endast dekrypteras för den namngivna tjänsten | SPN, tjänstkonto, målvärd, PAC och biljettens enctype |
| Authenticator | ny för varje AP Request, skyddad med sessionsnyckel | replay-skydd | klienttid, subkey, sekvens och serverns replay cache |
| Keytab | fil eller annat keytab-lager hos tjänsten | skydda som ett icke-interaktivt lösenord | filbehörigheter, distribution, inventering, rotation och säker radering |
| Replay cache | acceptorns lokala tillstånd | integritetstillstånd | kontrollera per tjänsteinstans, värd och klusterarkitektur |

Active Directory lagrar SPN:er i det flervärda attributet `servicePrincipalName` för ett användar- eller datorkonto. SPN kopplar tjänstnamnet som klienten sätter samman till exakt det konto vars nyckel KDC använder för servicebiljetten. Ett alias, load balancer-namn eller ändrat tjänstkonto kräver därför en medveten SPN-tilldelning. Dubbla SPN:er är tvetydiga inom sitt sökområde; en SPN på fel konto leder normalt till att tjänsten inte kan dekryptera den utfärdade biljetten ([Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

En keytab är inte en exportfil med en identitet som kan återställas godtyckligt, utan en samling verkliga långsiktiga nycklar. Varje post innehåller principal, KVNO, enctype och nyckel. Den som kan läsa filen kan utge sig för att vara denna principal. MIT rekommenderar lokal, restriktiv lagring och ingen oskyddad överföring. Vid Active Directory-interoperabilitet kan [`ktpass`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) koppla samman principal, konto och keytab; de valda parametrarna kan då påverka lösenord, salt, KVNO eller enctype och ska ingå i en testad rotationsprocess, inte i en engångsinstallationsanteckning ([MIT Kerberos: Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-kerberos.svg?v=20260813" title="Interaktive Infografik: Kerberos-Discovery, AS-, TGS- und AP-Fluss, Tickets, Schlüssel, Active-Directory-PAC und Delegationsgrenzen" loading="lazy">
  <a href="/images/kb-interaktiv-kerberos.svg?v=20260813">Öppna infografik om Kerberos-protokollflödet och dess förtroendegränser</a>
</iframe>

När principal, realm och nycklar har klarlagts kan biljettflödet läsas som tre på varandra följande samtal: AS, TGS och slutligen programtjänsten.

## Protokollflöde: AS, TGS och AP

AS-utbytet börjar med `KRB_AS_REQ`. En KDC kan svara med `KDC_ERR_PREAUTH_REQUIRED` och ange de pre-authentication-förfaranden som stöds. Med den vanligt förekommande krypterade tidsstämpeln bevisar klienten kännedom om sin långsiktiga nyckel innan KDC utfärdar ett TGT. `KRB_AS_REP` innehåller TGT:t, krypterat för TGS, och en svarsdel för klienten. PKINIT ersätter detta första bevis med public key-kryptografi och certifikat; Kerberos FAST kan förstärka pre-authentication i en skyddad tunnel ([RFC 4120, avsnitt 3.1 och 5.2.7](https://datatracker.ietf.org/doc/html/rfc4120#section-3.1), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc6113#section-5)).

I TGS-utbytet skickar klienten `KRB_TGS_REQ` med TGT:t, en authenticator och den begärda service-principalen. KDC kontrollerar realm-policy, biljettflaggor, målprincipal och nycklar som stöds och levererar i `KRB_TGS_REP` en servicebiljett samt en ny klient/tjänst-sessionsnyckel. Användarlösenordet behövs då inte igen. Ett redan befintligt TGT kan därför användas för många tjänster tills lifetime, policy eller cachestatus kräver en ny initial autentisering ([RFC 4120, avsnitt 3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3)).

Vid AP-utbytet presenterar klienten `KRB_AP_REQ` för tjänsten: servicebiljetten och en färsk authenticator skyddad med sessionsnyckeln. Tjänsten dekrypterar biljetten med sin långsiktiga nyckel, kontrollerar målprincipal, tider, flaggor och replaystatus och får därigenom sessionsnyckeln. Om klienten begär ömsesidig autentisering svarar tjänsten med `KRB_AP_REP`. Först detta svar bevisar kryptografiskt för klienten att motparten innehar tjänstnyckeln ([RFC 4120, avsnitt 3.2 och 5.5](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2)).

| Utbyte | Request | Framgångssvar | Kritiska indata | Typisk felgräns |
|---|---|---|---|---|
| AS | `KRB_AS_REQ` | `KRB_AS_REP` med TGT | klientprincipal, realm, pre-auth, tillåtna enctypes | okänt konto, pre-auth, tid, saknad klientnyckel |
| TGS | `KRB_TGS_REQ` | `KRB_TGS_REP` med servicebiljett | TGT, authenticator, SPN, flaggor, målnycklar | SPN, realm-referral, policy, delegering, enctype |
| AP | `KRB_AP_REQ` | valfritt `KRB_AP_REP` | servicebiljett, authenticator, tjänstnyckel, replay cache | fel tjänstkonto, keytab/KVNO, tid, replay |
| Fel | en av requesterna | `KRB_ERROR` | Error Code plus valfria e-data och servertid | bevara numerisk kod och berörd protokollfas |

Pre-authentication skyddar inte mot alla offlineattacker, men ändrar förutsättningarna. Konton utan obligatorisk pre-authentication kan ge ett AS-svar vars lösenordsskyddade del kan kontrolleras offline. Servicebiljetter kan också analyseras offline; svaga tjänstkontolösenord och RC4 förvärrar denna risk. Det effektiva skyddet består av pre-authentication, starka slumpmässiga tjänstnycklar eller gMSA, moderna enctypes, begränsade rättigheter och granskningsdata, inte av enbart förekomsten av Kerberos ([RFC 6113, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc6113#section-1), [RFC 8429, avsnitt 5](https://datatracker.ietf.org/doc/html/rfc8429#section-5), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

## Biljetter, tider, flaggor och transport

En biljett har `authtime`, valfritt `starttime`, `endtime` och för förnybara biljetter `renew-till`. TGT:t är inte en universell åtkomsttoken: det riktar sig till TGS och används för att hämta ytterligare biljetter. En servicebiljett riktar sig till exakt serverprincipalen. Lifetime och renewal styrs av KDC-policy och kan begränsas per realm eller konto. Fasta värden som «tio timmar» är därför produktstandarder, inte en egenskap hos Kerberos-standarden ([RFC 4120, avsnitt 2.3 och 5.3](https://datatracker.ietf.org/doc/html/rfc4120#section-2.3)).

Flaggor som `forwardable`, `forwarded`, `proxiable`, `proxy`, `renewable`, `initial`, `pre-authent` och `ok-as-delegate` förändrar en biljetts användbarhet. De är inga dekorativa diagnosfält. Ett double hop kan misslyckas trots att den första servicebiljetten är giltig, eftersom en nödvändig flagga eller KDC-policy saknas. Omvänt utvidgar ett forwardable TGT effekten av en komprometterad tjänst. Biljettflaggor ska därför ingå i varje delegerings- och incidentbevis ([RFC 4120, avsnitt 2](https://datatracker.ietf.org/doc/html/rfc4120#section-2)).

Authenticators och vissa pre-authentication-förfaranden använder tid för att kontrollera färskhet. RFC 4120 lämnar den tillåtna avvikelsen öppen som lokal policy. Active Directory använder som standard fem minuter; denna tolerans ersätter inte exakt tidssynkronisering. Klient, KDC och måltjänst är avgörande: en klient kan få en biljett men ändå misslyckas mot tjänsten med `KRB_AP_ERR_SKEW` om dess klocka avviker ([RFC 4120, avsnitt 3.2.3 och 7.5.1](https://datatracker.ietf.org/doc/html/rfc4120#section-3.2.3), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance), [Microsoft: Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service)).

Kerberos använder port 88 över UDP och TCP. RFC 4120 kräver TCP-stöd och beskriver UDP som valfritt; för stora UDP-svar kan med `KRB_ERR_RESPONSE_TOO_BIG` utlösa ett nytt försök via TCP. PAC, gruppmedlemskap och ytterligare Authorization Data gör biljetterna större. En brandvägg som bara tillåter små UDP-tester eller blockerar TCP 88 kan därför orsaka användar- eller gruppberoende fel ([RFC 4120, avsnitt 7.2.1](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.1), [Microsoft: Kerberos KDC configuration keys](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys)).

Servicebiljetten är bara det kryptografiska beviset. För att HTTP, LDAP eller SMB ska kunna använda den bäddar GSS-API, SSPI och SPNEGO in Kerberos i respektive applikationsprotokoll.

## Inbäddning i applikationsprotokoll

GSS-API ger program en mekanismoberoende Security Context; RFC 4121 definierar Kerberos V5-mekanismen för detta. Windows SSPI fyller en jämförbar roll. SPNEGO förhandlar mellan erbjudna GSS-mekanismer och förekommer ofta i HTTP som `Negotiate`. Det synliga headernamnet bevisar inte automatiskt att Kerberos valdes i stället för NTLM. Diagnostik måste fånga den faktiskt förhandlade mekanismen och det begärda Target Name ([RFC 4121](https://datatracker.ietf.org/doc/html/rfc4121), [RFC 4178](https://datatracker.ietf.org/doc/html/rfc4178), [Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)).

SASL GSS-API bäddar in samma Kerberos-mekanism i applikationsprotokoll. Det är exempelvis möjligt med [LDAP](/kb/ldap), IMAP eller SMTP när klient och server erbjuder mekanismen. Efter en framgångsrik GSS-kontext kan SASL Security Layers tillhandahålla integritet eller sekretess. Om de verkligen används och hur de samspelar med TLS är konfiguration för det specifika programmet; `GSSAPI` i capability-listningen bevisar i sig varken SPN eller Channel Protection ([RFC 4752](https://datatracker.ietf.org/doc/html/rfc4752)).

Klienten bildar tjänstnamnet av Service Class och målvärd. För HTTP är vanligtvis `HTTP/fqdn`, för LDAP `ldap/fqdn` relevant. Alias, CNAME, omvänd uppslagning, proxy, klusternamn och värdnamnet i URL:en kan förändra den identitet som bildas. Klienten måste begära biljetten för samma principal vars nyckel den accepterande tjänsten innehar. En load balancer löser inte denna bindning; alla backendinstanser behöver en konsekvent tjänsteidentitet och nyckelstrategi ([MIT Kerberos: Application servers – DNS](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html#getting-dns-information-correct), [Microsoft: Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)).

## Active Directory som Kerberos-realm

I Active Directory Domain Services är KDC integrerad i domänkontrollanten. Användar-, dator- och tjänstkonton är Kerberos-principaler; katalogen tillhandahåller nycklar, SPN:er, grupper, kontoflaggor och policyer. Kontot `krbtgt` representerar domänens TGS. KDC-tillgänglighet och KDC-datakonsistens följer därmed DC Locator, DNS, AD-replikering, sites och återställningsmodellen för AD DS, inte ett separat Kerberos-klusterprotokoll ([Microsoft: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview), [MS-KILE: Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13), [DC Locator](/kb/ldap#active-directory-als-ldap-serverprofil)).

AD lägger normalt till ett Privilege Attribute Certificate, PAC, som Authorization Data i biljetter. Det kan bland annat innehålla SID:er, gruppmedlemskap, profil- och policyinformation samt signaturer. KDC skapar och signerar dessa data; tjänsten använder dem för Windows-auktorisering eller låter vid behov validera dem. Kerberos kärnautentisering och AD-auktorisering är därför separata påståenden: en kryptografiskt giltig principal har inte automatiskt den önskade behörigheten ([MS-PAC](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962), [MS-KILE: PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909)).

Nyckelmaterial och SPN-attribut replikeras med Active Directory. Efter ändringar av konto, lösenord eller SPN kan olika DC:er därför kortvarigt använda olika tillstånd. Om klienten för TGS och administratören för kontrollen träffar olika DC:er framstår felet som intermittent. Ett tillförlitligt bevis anger utfärdande KDC, mål-DC för katalogfrågan, KVNO, biljettens enctype och replikeringsstatus. Det gäller särskilt vid manuella keytab-rotationer och tjänster över flera platser.

Valet av enctype är snittmängden av klienterbjudande, KDC-policy, målkontots nycklar och tjänstens stöd. AES-kapabel programvara räcker inte om kontot saknar motsvarande nycklar eller om `msDS-SupportedEncryptionTypes` och domänpolicyn utesluter dem. RFC 8429 klassar RC4 och 3DES som föråldrade för Kerberos; Microsoft dokumenterar inventering via Security Events 4768 och 4769. En utfasning börjar med mätning och nyckelgenerering, inte med global avstängning av en bit ([RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [Microsoft: Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)).

För Windows-tjänster minskar Group Managed Service Accounts, gMSA, den manuella lösenords- och SPN-hanteringen. De ersätter dock inte kontrollen av vilken identitet processen faktiskt kör under, vilka värdar som får läsa det hanterade lösenordet och vilka SPN:er som är registrerade på kontot. För apparater eller Unix-tjänster behövs ofta fortsatt en keytab; dess rotation måste synkroniseras med AD-kontot ([Microsoft: Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts)).

Inom en AD-domän är denna väg direkt. Vid åtkomst över realm- eller domängränser tillkommer cross-realm-biljetter och vid behov flera mellansteg.

## Trusts och cross-realm-sökvägar

En realm-trust slår inte samman principal-databaser. Cross-realm-autentisering använder TGT:er för `krbtgt/TARGET@SOURCE` och vid behov flera mellanrealmer. Klienten följer en biljettsökväg tills den når måltjänstens realm. RFC 6806 kompletterar med referrals och namnkanonisering, som särskilt används i AD-miljöer. Sökvägen som är synlig i cachen är därför mer talande än det generella påståendet «trusten är grön» ([RFC 4120, avsnitt 1.1 och 3.3.3](https://datatracker.ietf.org/doc/html/rfc4120#section-3.3.3), [RFC 6806](https://datatracker.ietf.org/doc/html/rfc6806)).

Trust-riktning, transitivitet, Selective Authentication, SID-filtrering, Name Suffix Routing och tillgängliga enctypes begränsar vad ett cross-realm-TGT praktiskt möjliggör. SPN:er måste kunna hittas i rätt forest och vara entydiga. En lokal träff med `setspn -Q` bevisar ingen forestövergripande entydighet; [`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) har forest- och domänalternativ för detta. Diagnostik dokumenterar varje referral-TGT, inte bara det sista servicebiljettfelet ([Microsoft: Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts), [Microsoft: setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)).

En biljett till frontend ger den inte automatiskt rätt att kontakta ett backend i användarens namn. Det är här double hop-problemet börjar.

## Delegering och double hop

Vid normalt AP-utbyte får ett frontend ingen fritt användbar användarnyckel. Om det ska få åtkomst till ett backend i användarens namn behöver det en delegeringsmodell. Forwarded-TGT eller unconstrained delegation överlämnar ett TGT som kan användas vidare till tjänsten och utökar dess verkan långt bortom ett enskilt backend. Ett komprometterat frontend kan använda det för biljetter till andra tjänster; denna form innebär därför en stor utvidgning av förtroendet ([RFC 4120, avsnitt 2.5 och 2.6](https://datatracker.ietf.org/doc/html/rfc4120#section-2.5), [MS-SFU: Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a)).

Microsofts Service-for-User-utökningar delar upp problemet. Med S4U2self kan en tjänst hämta en biljett till sig själv i en användares namn, exempelvis efter en annan frontend-autentisering. Med S4U2proxy begär den, enligt KDC-policy, en biljett till en andra tjänst i användarens namn. Klassisk constrained delegation lagrar tillåtna mål på frontendkontot; resource-based constrained delegation placerar de tillåtna anroparna på resurskontot. I båda fallen ingår mål-SPN, biljettflaggor, kontoinställningar och trust-sökväg i beslutet ([MS-SFU: Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf), [MS-SFU: Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d)).

Delegering är auktorisering för vidarebefordran av identitet, inte bara ett kompatibilitetsalternativ. Administratören inventerar frontendprincipal, backend-SPN:er, tillåtna användarklasser, Protocol Transition, trust-gränser och minsta tekniska målomfång. En lyckad första hop bevisar inte den andra; ett direkt backendtest i användarkontext kringgår omvänt delegeringsgränsen och kan ge ett falskt positivt resultat.

## Driftmodell, övervakning och återställning

Efter biljettflöde och delegering uppstår driftsfrågan: Vilken Kerberos-komponent måste vara tillgänglig, vilket bevis visar dess tillstånd och vad kan återställas i en nödsituation? Tabellen kopplar dessa frågor till de berörda rollerna.

| Roll | Tillstånd att övervaka | Ledande mätvärde eller bevis | Vanlig blind fläck |
|---|---|---|---|
| Klient | realm-mappning, KDC-val, klocka och credential cache | AS-/TGS-latens, cache-lifetime, KDC och Error Code | testet använder annat DNS-namn eller annan användarinloggningssession |
| KDC | principal-nycklar, policy, replikering, trust och audit | 4768/4769/4771, felfrekvens per kod, enctype och utfärdande DC | totalfel utan mål-SPN och klienterbjudande |
| Tjänst | tjänstkonto, SPN, keytab/key store, replay cache och klocka | AP-framgång, KVNO, biljettens enctype, målprincipal | porten är öppen men processen kör under annan identitet |
| Frontend med delegering | S4U-/forwarding-policy och backendmål | första och andra hoppet separat, delegeringssökväg och biljettflaggor | direkt backendtest kringgår double hop |
| Realm-/forest-drift | KDC-databas, realm-nycklar, AD-replikering och återställning | replikerat nyckeltillstånd, säkerhetskopierad konfiguration, testad återställning | tjänstens keytab och KDC-generation glider isär |

Biljettcachar är driftstillstånd. En användarprocess, ett tjänstkonto, en container eller en Windows Logon Session kan se en annan cache än det interaktiva administratörsskalet. Att rensa och hämta en biljett på nytt är ett riktat test, men ingen reparation av den underliggande orsaken i SPN, nyckel eller replikering. Före purge registreras principal, mål-SPN, KDC, KVNO, enctype, flaggor och tidsfält; annars försvinner det bästa felbeviset.

Windows loggar TGT-begäranden under 4768, servicebiljetter under 4769, pre-authentication-fel under 4771 och ytterligare AS-fel under 4772, förutsatt att lämpliga Advanced Audit Policies är aktiva. Volymen är hög på KDC:er. Övervakning behöver därför aggregering efter Result Code, klient, måltjänst, DC och enctype samt baslinjer, i stället för att behandla varje lyckad TGS Request som ett larm ([Microsoft: Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration)).

Återställning beror på KDC-implementeringen. I AD DS ingår KDC-data och `krbtgt` i modellen för System State- och forest-återställning; en godtycklig databasfil eller LDIF-export är ingen giltig Kerberos-säkerhetskopia. Operatörer av fristående MIT-realmer måste säkerhetskopiera KDC-databas, stash-/master-key-material, konfiguration, ACL:er och replikering tillsammans. Tjänst-keytabs ska dessutom inventeras och efter en återställning kontrolleras för KVNO- och nyckelkonsistens ([Microsoft: Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state), [MIT Kerberos: Backups of secure hosts](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html)).

Vid felsökning kontrolleras vägen baklänges: målnamn och SPN, befintlig servicebiljett, TGS-svar, TGT, realmidentifiering, DNS och tid.

## Diagnosverktyg

Diagnosen börjar från samma nätverkszon, med samma målnamn, realm, användar- eller tjänstkontext och samma Logon Session som programmet. Testdata använder reserverade namn. Biljett- och keytab-utdata kan exponera principaler och infrastruktur; nyckelmaterial i sig får aldrig hamna i biljetter, chatt eller processargument.

### Identifiera KDC och lösenordstjänst via DNS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Dienstsuche">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) visar prioritet, vikt, port och target. Därefter ska A-/AAAA-upplösning och nåbarhet för varje levererat target kontrolleras. RFC 4120 definierar DNS-SRV-discovery, men tillåter lokal realm-konfiguration; ett tomt SRV-test bevisar därför bara ett fel när den aktuella klienten använder DNS-discovery ([RFC 4120, avsnitt 7.2.3](https://datatracker.ietf.org/doc/html/rfc4120#section-7.2.3), [RFC 2782](https://datatracker.ietf.org/doc/html/rfc2782)).

### Jämför tid och TCP-nåbarhet

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Zeit- und Portprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) och [`timedatectl`](https://man7.org/linux/man-pages/man1/timedatectl.1.html) visar källa och synkroniseringsstatus; [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) kompletterar med offset- och trackingdata för Chrony. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) respektive [`nc`](https://man.openbsd.org/nc) bevisar endast TCP 88. UDP, KDC-protokoll, realm och pre-authentication kräver ett verkligt AS-/TGS-test.

### Hämta och visa biljetter på nytt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-Ticketprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

Windows-[`klist`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) arbetar i den valda Logon Session-kontexten. Under MIT Kerberos rensar [`kdestroy`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html), hämtar [`kinit`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) och [`kvno`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) samt visar [`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) credentials. [`KRB5_TRACE`](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) gör KDC-val och protokollsökväg synliga. Före `purge` eller `kdestroy` ska en befintlig felbiljett dokumenteras.

### Kontrollera SPN och tjänstnyckel mot varandra

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Kerberos-SPN- und Keytabprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`setspn`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) visar AD-målkontot och söker forestomfattande efter dubbletter med `-X -F`. [`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) läser SPN:er och deklarerade enctypes; ett saknat värde har produktspecifik fallback-semantik och får inte generellt tolkas som «ingen AES». MIT-[`klist`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) visar principal, KVNO och enctype för keytaben; [`kvno -k`](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) begär en biljett och validerar den mot den angivna keytaben.

### Tilldela fel till en protokollfas

| Kod eller symptom | Fas och vanlig gräns | Nästa bevis |
|---|---|---|
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` | AS: klientprincipal okänd i vald realm | realm-mappning, UPN/principal, utfärdande KDC och replikering |
| `KDC_ERR_PREAUTH_FAILED` | AS: nyckel, lösenord, certifikat eller pre-auth-förfarande avvisat | klienttid, pre-auth-typ, kontonyckel och KDC-audit 4771 |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | TGS: mål-SPN hittas inte eller kan inte lösas entydigt | exakt begärd SPN, forestomfattande sökning och målrealm |
| `KDC_ERR_ETYPE_NOSUPP` | AS/TGS: ingen gemensam snittmängd av enctype/nyckel | klienterbjudande, KDC-policy, kontonycklar, keytab och 4768/4769 |
| `KRB_AP_ERR_MODIFIED` | AP: biljetten matchar inte nyckeln hos den svarande tjänsten | SPN-konto, processidentitet, keytab, KVNO, enctype och backendnod |
| `KRB_AP_ERR_SKEW` | AS/AP: tid utanför toleransen | klient-, KDC- och tjänsttid samt respektive tidskälla |
| `KRB_AP_ERR_TKT_EXPIRED` | AP: biljett utanför giltighetsfönstret | cache, `endtime`, renewal, klienttid och ny initiering |
| `KDC_ERR_BADOPTION` | TGS/S4U: flagga, delegering eller policy otillåten | Forwardable-flagga, frontendkonto, backend-SPN och delegeringskonfiguration |
| Kerberos-biljett finns, programmet använder NTLM | Mechanism Negotiation eller Target Name | SPNEGO-resultat, URL/FQDN, zon-/klientpolicy och faktiskt bildad SPN |

Felkoderna standardiseras i RFC 4120; Windows kompletterar med status- och granskningskontext. Automatisering bör bevara den numeriska koden, fasen, KDC, klientprincipal och målprincipal. Enbart fritext är varken stabil eller entydig. För paketanalyser kan [Wireshark](https://www.wireshark.org/docs/dfref/k/kerberos.html) filtrera efter `kerberos`; krypterade biljett delar förblir avsiktligt oläsliga utan passande nycklar ([RFC 4120, avsnitt 7.5.9](https://datatracker.ietf.org/doc/html/rfc4120#section-7.5.9), [Microsoft: Kerberos troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)).

## Teknisk historia

Kerberos uppstod under tidigt 1980-tal i MIT Project Athena. Protokollet bygger konceptuellt på arbeten om betrodd tredje part av Needham och Schroeder samt Denning och Sacco. Version 1 till 4 utvecklades i Athena-miljön; version 4 var den första brett använda utgåvan. Namnet syftar på Kerberos, den flerhövdade väktaren i grekisk mytologi, och står för KDC:s centrala förtroenderoll ([RFC 4120, avsnitt 1](https://datatracker.ietf.org/doc/html/rfc4120#section-1), [MIT Kerberos Consortium: Documentation](https://kerberos.org/docs/index.html)).

Kerberos V5 undanröjde begränsningar i version 4 avseende namngivning, ticket lifetimes, kryptografi, cross-realm och utbyggbarhet. RFC 1510 standardiserade V5 år 1993. RFC 4120 ersatte denna specifikation år 2005 med förtydliganden och en fullständig ASN.1-beskrivning. Protokollfamiljen utökades därefter modulärt, bland annat med PKINIT, Pre-Authentication Framework och FAST, GSS-API, referrals samt nya AES-profiler ([RFC 1510](https://datatracker.ietf.org/doc/html/rfc1510), [RFC 4120](https://datatracker.ietf.org/doc/html/rfc4120), [RFC 4556](https://datatracker.ietf.org/doc/html/rfc4556), [RFC 6113](https://datatracker.ietf.org/doc/html/rfc6113)).

Microsoft gjorde Kerberos V5 till det centrala domänautentiseringsprotokollet med Windows 2000 och kopplade det till AD-principaler, SPN:er, PAC, SSPI, trust-referrals och delegeringsutökningar. Parallellt förblev MIT Kerberos, Heimdal och andra implementationer interoperabla via IETF-protokoll och GSS-API. Kryptografin utvecklades från DES och senare RC4 till AES-profiler; RFC 8429 inledde utfasningen av 3DES och RC4 år 2018. Den tekniska historien förklarar varför äldre enheter, gamla tjänstkonton och trust-nycklar fortfarande synliggör enctype-gränser, utan att artikeln låser fast en flyktig produktversionsnivå ([MS-KILE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/), [RFC 3962](https://datatracker.ietf.org/doc/html/rfc3962), [RFC 8009](https://datatracker.ietf.org/doc/html/rfc8009), [RFC 8429](https://datatracker.ietf.org/doc/html/rfc8429)).

## Källor

- [IETF RFC 3244 – Microsoft Windows 2000 Kerberos Change Password and Set Password Protocols](https://datatracker.ietf.org/doc/html/rfc3244)
- [MIT Kerberos – Mapping Hostnames onto Kerberos Realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/princ_dns.html)
- [IANA – Kerberos service names and ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=kerberos)
- [RFC 4120 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) – arkitektur, meddelanden, biljetter, flaggor, felkoder, transport och säkerhetsmodell.
- [RFC 3961, avsnitt 3](https://datatracker.ietf.org/doc/html/rfc3961)
- [RFC 4121 – Kerberos V5 GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4121) – GSS-kontext, tokens, integritet och sekretess.
- [MIT Kerberos: Mapping hostnames onto realms](https://web.mit.edu/kerberos/krb5-latest/doc/admin/realm_config.html)
- [MIT Kerberos – Keytabs](https://web.mit.edu/kerberos/krb5-latest/doc/basic/keytab_def.html) – keytab-innehåll, KVNO, enctype och nyckel.
- [Microsoft – Service principal names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) – SPN-entydighet och bindning till tjänstkonton.
- [Microsoft – setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn) – SPN-fråga, registrering och sökning efter dubbletter.
- [Microsoft – ktpass](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass) – AD-/keytab-interoperabilitet.
- [MIT Kerberos – Application servers](https://web.mit.edu/kerberos/krb5-latest/doc/admin/appl_servers.html) – keytab-skydd, rotation, klocka och DNS.
- [RFC 4556 – PKINIT](https://datatracker.ietf.org/doc/html/rfc4556) – public key-pre-authentication.
- [RFC 6113 – Generalized Framework for Kerberos Pre-Authentication](https://datatracker.ietf.org/doc/html/rfc6113) – pre-auth-ramverk och FAST.
- [RFC 8429 – Deprecate 3DES and RC4 in Kerberos](https://datatracker.ietf.org/doc/html/rfc8429) – IETF Best Current Practice för föråldrade enctypes.
- [Microsoft – Detect and remediate RC4 usage](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) – enctype-inventering, kontonycklar och händelser.
- [Microsoft – Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance) – Windows-felkoder, tid, SPN och double hop.
- [Microsoft – Windows Time Service](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service) – tidstjänst och Kerberos-beroende.
- [Microsoft – Kerberos KDC registry and protocol settings](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-protocol-registry-kdc-configuration-keys) – UDP-storlek, TCP-fallback och KDC-diagnostik.
- [RFC 4178 – SPNEGO](https://datatracker.ietf.org/doc/html/rfc4178) – förhandling av GSS-mekanismer.
- [Microsoft – Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) – Windows-arkitektur, KDC, SSPI och ömsesidig autentisering.
- [RFC 4752 – SASL GSS-API Mechanism](https://datatracker.ietf.org/doc/html/rfc4752) – inbäddning i LDAP, IMAP, SMTP och andra SASL-protokoll.
- [MS-KILE – Kerberos V5 Synopsis](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/b4af186e-b2ff-43f9-b18e-eedb366abf13) – AS-, TGS- och AP-utbyte i Windows.
- [MS-PAC – Privilege Attribute Certificate](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pac/166d8064-c863-41e1-9c23-edaaa5f36962) – grupper, SID:er, profiler, policyer och signaturer.
- [MS-KILE – PAC Generation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/c25d48df-67f0-4c5f-9e46-27a7d5710909) – skapande av PAC-Authorization-Data.
- [RFC 8009 – AES with HMAC-SHA2 for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc8009) – AES-SHA2-enctypes.
- [Microsoft – Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-service-accounts) – gMSA, lösenords- och SPN-hantering.
- [RFC 6806 – Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://datatracker.ietf.org/doc/html/rfc6806) – referrals och namnkanonisering.
- [Microsoft – Windows Authentication Concepts](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-concepts) – trusts, Protocol Transition och constrained delegation.
- [MS-SFU – Protocol Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/1fb9caca-449f-4183-8f7a-1a5fc7e7290a) – delegeringsflöden och risker med vidarebefordrade TGT:er.
- [MS-SFU – Service for User Overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/36d103d2-61a6-42d5-a725-74de3205cdaf) – S4U2self och S4U2proxy.
- [MS-SFU – Introduction](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/8ee85a47-7526-4184-a7c5-25a5e4155d7d) – Protocol Transition och constrained delegation.
- [Microsoft – Advanced Audit Policy Configuration](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration) – händelserna 4768, 4769, 4771 och 4772.
- [Microsoft – Back up the System State data](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-backing-up-system-state) – säkerhetskopiering av AD DS/KDC.
- [MIT Kerberos – kdb5_util](https://web.mit.edu/kerberos/krb5-latest/doc/admin/admin_commands/kdb5_util.html) – KDC-databas och säkerhetskopiering i MIT Kerberos.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – DNS-SRV-frågor i Windows.
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html) – DNS-SRV-frågor i Unix-system.
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – SRV-prioritet, vikt, port och target.
- [Microsoft – w32tm](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings) – Windows-tidskälla och offsetdiagnostik.
- [Linux man-pages – timedatectl(1)](https://man7.org/linux/man-pages/man1/timedatectl.1.html) – tidssynkroniseringsstatus i Linux.
- [Chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html) – offset-, käll- och trackingdiagnostik.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – TCP-anslutningsdiagnostik i Windows.
- [OpenBSD – nc(1)](https://man.openbsd.org/nc) – TCP-porttest i Unix-system.
- [Microsoft – klist](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/klist) – Windows-biljettcachar och servicebiljettest.
- [MIT Kerberos – kdestroy](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kdestroy.html) – rensa credential cache.
- [MIT Kerberos – kinit](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kinit.html) – hämta TGT och pre-auth-alternativ.
- [MIT Kerberos – kvno](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/kvno.html) – servicebiljett och keytab-validering.
- [MIT Kerberos – klist](https://web.mit.edu/kerberos/krb5-latest/doc/user/user_commands/klist.html) – visning av credential cache och keytab.
- [MIT Kerberos – kerberos environment](https://web.mit.edu/kerberos/krb5-current/doc/user/user_config/kerberos.html) – `KRB5_TRACE` och cache-/keytabvariabler.
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) – SPN- och enctype-attribut för ett tjänstkonto.
- [Wireshark – Kerberos display filter reference](https://www.wireshark.org/docs/dfref/k/kerberos.html) – protokollfält och display-filter.
- [MIT Kerberos Consortium – Documentation](https://kerberos.org/docs/index.html) – ursprung i Project Athena och versionshistoria.
- [RFC 1510 – The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc1510) – historisk V5-kärnspecifikation från 1993.
- [MS-KILE – Kerberos Protocol Extensions](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/) – Active Directory-Kerberos och Microsoft-utökningar.
- [RFC 3962 – AES Encryption for Kerberos 5](https://datatracker.ietf.org/doc/html/rfc3962) – AES-SHA1-profiler.
