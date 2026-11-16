---
title: "Migrering: målarkitektur, samexistens och kontrollerad cutover"
blatt: "migration"
description: "Teknisk migrering för administratörer av meddelandesystem och infrastruktur: käll- och målarkitektur, beroenden, identitets- och datamappning, rehost till refactor, samexistens, seed- och deltasynkronisering, pilot, cutover, ansvarsgränser, rollback, godkännande och avveckling."
fakten:
  - label: Utgångspunkt
    wert: verksamhetstransaktion + källarkitektur + tillstånd + ägare
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Mål
    wert: mätbar målarkitektur med gränssnitt, dataansvar, förtroende- och felgränser
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Strategier
    wert: Retain · Retire · Rehost/Relocate · Repurchase · Replatform · Refactor
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Mönster
    wert: Cutover · Pilot/vågor · samexistens · stegvis ersättning
    href: https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html
  - label: Identitet
    wert: avbilda käll-ID, mål-ID, inloggning, SMTP-adresser, grupper, roller och livscykel separat
    href: https://datatracker.ietf.org/doc/html/rfc7643
  - label: Dataväg
    wert: Seed → deltasynkronisering → Freeze/Final Delta → avstämning
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration
  - label: Samexistens
    wert: adressbok · e-postrouting · discovery · autentisering · behörigheter · kalender · support
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: E-postansvar
    wert: 250 efter End-of-Data innebär att den svarande SMTP-hoppen har accepterat meddelandet
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: DNS-cutover
    wert: TTL begränsar cachelivslängden; negativa cachar och lokala åsidosättningar kontrolleras separat
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Rollbackgräns
    wert: fastställ uttryckligen före den första ensamma skrivåtgärden i målet
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Godkännande
    wert: fullständighet · integritet · behörighet · transaktion · prestanda · övervakning
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Avveckling
    wert: hantera routing, identiteter, nycklar, säkerhetskopior, loggar, avtal och datamedier kontrollerat
    href: https://csrc.nist.gov/pubs/sp/800/88/r1/final
werbung:
  - stargate
  - newsletter
ctaThemen:
  - migration
  - messaging
  - exchange-hybrid
translationSourceHash: bdb2fea86e1f811e78f2f210afd26b082f808325322fb675a8ec446e47b38ac3
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:21:52.326Z
translationReview: automatic
---

# Migrering: målarkitektur, samexistens och kontrollerad cutover

En teknisk migrering överför affärsprocesser, gränssnitt och tillstånd från en källarkitektur till en definierad målarkitektur. Det synliga produktbytet är bara en del av detta. Identiteter, adresser, behörigheter, köer, data, nycklar, DNS, certifikat, klienter, automatiseringar, övervakning, säkerhetskopiering och supportprocesser måste fortsatt uppfylla samma verksamhetsmässiga åtaganden – eller medvetet överföras till ett nytt avtal.

För meddelandesystem är det viktigaste åtagandet kontrollerat ansvar för varje meddelande. Under en migrering kan gamla och nya plattformar samtidigt känna till adresser, acceptera anslutningar och ändra data. Utan uttryckligt dataansvar uppstår slingor, split-brain-adressböcker, dubbelleveranser, förlorade behörigheter och en återväg som bara finns i planeringen. En robust migrering börjar därför med arkitektur och tillstånd, inte med datumet för omkopplingen.

Förklaringen börjar med det dokumenterade nuläget och går via målbild, beroenden och dataöverföring till samexistens. Pilot, cutover, återväg och godkännande följer först när det är klart vilka data och funktioner som flyttas.

En migrering är inte en kopieringsprocess utan det kontrollerade bytet från en fungerande källarkitektur till en verifierad målarkitektur. Först registreras tillstånd och beroenden, därefter planeras samexistens, dataförflyttning, cutover och återväg.

## Från käll- till målarkitektur

Före produktvalet dokumenteras den befintliga tjänsten som en graf. Varje nod har en roll; varje kant har initiativtagare, mål, protokoll, identitet, dataklass, felbeteende och ägare. En CMDB-lista över servernamn räcker inte om det är okänt vilken skanner som anropar ett SMTP-hopp, vilken tjänst som expanderar grupper eller vilken nyckel som dekrypterar gamla meddelanden.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-migration.svg?v=20260813" title="Interaktive Infografik: Migration von Ausgangs- zu Zielarchitektur mit Strategie, Zustandsklassen, Koexistenz, Seed und Delta, Pilotwellen, Cutover, Rollbackgrenze, Abnahme und Stilllegung" loading="lazy">
  <a href="/images/kb-interaktiv-migration.svg?v=20260813">Öppna den interaktiva grafiken direkt</a>.
</iframe>

Källdokumentationen omfattar minst:

| Domän | Nuläge som ska registreras | Typiskt dolt avtal |
|---|---|---|
| E-postrouting | MX, Relay-/Submission-lyssnare, anslutningar, smarthosts, accepterade domäner, käll-IP, TLS-policy | Partner tillåtslistar en gammal IP eller förväntar sig ett certifikatnamn |
| Identitet | Katalogkälla, objektankare, inloggning, grupper, roller, tjänstkonton, provisionering | Applikationen använder e-postadressen som oföränderligt ID |
| Data | Brevlåda, arkiv, journaler, kö, karantän, blob, index, konfiguration, granskning | Sökindex kan rekonstrueras, medan journalen måste bevaras |
| Kryptografi | TLS-certifikat, S/MIME-/OpenPGP-nycklar, HSM/KMS, lösenfraser, återställning | Ett gammalt krypterat meddelande kräver en historisk privat nyckel |
| Klienter och API:er | Outlook/IMAP/SMTP/API, agenter, mobila klienter, webhooks, skript | Hårdkodad URL, chiffer, port, header eller API-fält |
| Drift | Övervakning, säkerhetskopiering, återställning, patchning, loggning, support, licenser | Larmet kontrollerar endast processstatus för det gamla systemet |
| Organisation | Ägare, beredskap, godkännande, dataskydd, lagring, avtal | Externt team kontrollerar DNS eller brandvägg med lång ledtid |

Målarkitekturen beskriver samma fält som **målavtal**. Dessutom anges förtroendegränser, dataansvar, felområden, återställningsmål, skalning och det planerade slutläget för det gamla systemet. NIST SP 800-34 kopplar systemutveckling till beredskapsplanering och kräver att återställningsstrategier och tester härleds från verksamhetskrav. För migreringar följer av detta att målarkitektur och återfallsdrift utformas tillsammans ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)).

## Migreringsstrategi: Vad förändras faktiskt?

De av AWS dokumenterade «7 R:en» skiljer mellan Retire, Retain, Rehost, Relocate, Repurchase, Replatform och Refactor/Re-architect. De är en taxonomi för molnmigrering, men kan användas som en precis fråga för varje plattformsmigrering: Finns funktionen kvar, avslutas den, flyttas den oförändrad, ersätts den av en produkt, anpassas den till en ny plattform eller byggs den om verksamhetsmässigt? AWS påpekar att refactoring i stora migreringar är den mest komplexa strategin och att separat modernisering kan minska riskerna ([AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)).

| Strategi | Vad förblir lika? | Vad ändras? | Exempel för meddelandesystem |
|---|---|---|---|
| Retain | Plattform och ansvar | endast dokumenterat undantag/tidsfrist | Legacy-arkiv förblir läsbart tills retentionen löper ut |
| Retire | Inget produktivt | Funktion och åtkomster upphör | Oanvänd POP3-slutpunkt stängs av |
| Rehost / Relocate | Applikation och datamodell i huvudsak | Compute-/virtualiseringsplats | MTA-VM flyttas till annan infrastruktur |
| Repurchase | Verksamhetskrav | Produkt, datamodell, drift och avtal | Appliance-gateway ersätts med SaaS-gateway |
| Replatform | Verksamhetsfunktion och delar av arkitekturen | Runtime, databas eller driftmodell | E-posttjänsten använder ny databas-/containerplattform |
| Refactor | Endast verksamhetsavsikt | Gränssnitt, komponenter och tillstånd | Monolitisk e-postpipeline byggs om händelseorienterat |

Strategin väljs **per komponent och dataklass**. Ett projekt kan replatforma e-postrouting, retaina arkivdata, retirea ett rapporteringsverktyg och repurchasea användargränssnittet. Den som kallar allt för «Lift and Shift» döljer de faktiskt olika riskerna.

## Cutover, vågor, samexistens och stegvis ersättning

Fyra omställningsmönster skiljer sig tekniskt och måste härledas ur dokumenterade produktfunktioner:

- **Engångscutover:** Alla skrivande vägar växlar i ett fönster. Lämpligt när samexistens inte är tekniskt möjlig, tillståndet är litet och avbrottet är acceptabelt. Återvägen måste vara tydlig före den första skrivåtgärden i målet.
- **Pilot och vågor:** En representativ population växlar först; fler batchar följer efter godkännande. Microsofts migreringsbatchar är ett konkret exempel på stegvis flytt av brevlådor ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).
- **Samexistens:** Gammalt och nytt delar tillfälligt namnområde, adressbok, routing eller användarpopulation. Exchange Hybrid dokumenterar bland annat säker e-posttrafik, gemensamt domännamnområde, adressbok och kalenderdelning som samexistensfunktioner ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).
- **Stegvis ersättning:** Ett routing- eller integrationslager fångar upp funktioner och flyttar dem successivt till målet. Martin Fowler beskrev detta mönster 2004 som «Strangler Fig Application» för att minska risken med en enda stor rewrite-cutover ([Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Inget av dessa mönster är generellt säkert. Samexistens minskar cutover-omfattningen men skapar ytterligare tillstånd, routingregler och supportärenden. Dual Write kan öka tillgängligheten, men skapar problem med konfliktlösning och ordningsföljd. En engångscutover är lättare att förstå men ökar den samtidiga skadeomfattningen. Beslutet anger därför produktbevis, tillåten varaktighet, dataansvar och exitkriterier.

Efter valet av strategi måste det för varje dataklass stå klart vilket system som får skriva under varje fas. Två okoordinerade sanningskällor skapar konflikter som ett kopieringsverktyg inte kan lösa.

## Tillståndsklasser och dataansvar

Före varje synkronisering fastställs vilka data som är **auktoritativ källa**, **replikerad kopia**, **härledd projektion** eller **endast historiskt arkiv**:

| Tillståndsklass | Auktoritet under samexistens | Konflikt-/återställningsfråga |
|---|---|---|
| Identitetsmasterdata | Exakt en ledande katalog eller ett HR-system | Vilket objektankare överlever namnbyte och tenantbyte? |
| SMTP-adresser och routingobjekt | En godkänd adresskälla | Hur förhindras dubbletter och catch-all-/aliasregler? |
| Brevlådeinnehåll | Entydigt källa eller mål för varje migrerad brevlåda | Får källbrevlådan fortfarande ta emot skrivningar efter cutover? |
| Kö | Respektive mottagande MTA bär ansvar | Hur töms gamla köer utan att återinjicera dubbelt? |
| Policy och konfiguration | Deklarativt måltillstånd eller angiven master | Vilka regler är semantiskt ekvivalenta, inte bara syntaktiskt kopierade? |
| Nycklar och certifikat | KMS/HSM eller definierad nyckelägare | Vilka historiska privata nycklar måste förbli läsbara? |
| Granskning, journal och retention | Oföränderlig bevarandekälla | Kan tid, fullständighet, Legal Hold och export styrkas? |
| Index/cache | Rekonstruerbar projektion | Hur länge tar ombyggnaden, och när räknas sökningen som fullständig? |

«Dubbelriktat synkroniserad» är ingen datamodell utan konfliktregel. För varje fält dokumenteras riktning, frekvens, transformering, fellagring och beteende vid samtidig ändring. Om verktyget inte kan slå samman ändringar utan förlust görs ena sidan skrivskyddad under den berörda fasen eller ändras endast via en central skrivväg.

## Identitet, adresser och behörigheter

Användarnamn, e-postadress och tekniskt objekt-ID är olika saker. RFC 7643 definierar för SCIM bland annat `id` som en av tjänsteleverantören tilldelad, oföränderlig och unik identifierare inom tenanten; `externalId` kan bära en klienthanterad korrelation. Det illustrerar en central migreringsregel: En föränderlig adress är inget säkert objektankare ([RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643)).

En identitetsmappning innehåller minst käll-ID, mål-ID, `externalId` eller annat korrelationsankare, inloggning/UPN, primär SMTP-adress, alias, grupper, roller, chef-/organisationsattribut, tjänstkonton, autentiseringsmetod och provisioneringsstatus. RFC 7644 beskriver SCIM-operationer för skapande, ändring, sökning och bulkbearbetning; den konkreta målplattformen kan ändå ha avvikande obligatoriska fält, unicitetsregler och rate limits ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)).

Behörigheter migreras som relationer, inte som textlistor. Delegering, Send As, Send on Behalf, Folder ACL, Shared Mailbox, resursbokning, Application Access och administratörsroll kan referera till olika objekttyper. Varje relation behöver ett upplöst målankare; föräldralösa eller cykliska referenser redovisas före vågen. Microsoft dokumenterar exempelvis att migrering av brevlådor mellan tenants kräver mål-`MailUser`-objekt och vissa attribut samt endast överför definierat innehåll och definierade behörigheter ([Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)).

## Dataväg: Seed, delta, freeze och avstämning

Stora tillstånd förladdas vanligtvis före cutover. Det robusta förloppet är:

1. **Seed:** Fullständigt ursprungsbestånd med objektantal, bytevolym, fellista och tidsintervall.
2. **Deltasynkronisering:** Överför ändringar sedan seed upprepade gånger; övervaka lagg och felfrekvens.
3. **Freeze eller skrivomdirigering:** Stoppa exakt definierade källskrivvägar eller led dem till målet.
4. **Final Delta:** Hämta kvarvarande ändringar och säkra en reproducerbar vattennivå.
5. **Cutover:** Växla routing, discovery och användaråtkomst till målet.
6. **Avstämning:** Jämför antal, storlek, nyckelattribut, behörigheter, stickprov och verksamhetstransaktioner.

Microsofts stegvisa e-postmigrering dokumenterar batchar, synkroniseringsstatus och kravet att invänta minst ytterligare en synkronisering efter routingomläggningen innan en batch tas bort. Detta är ett produktrelaterat bevis för seed-/delta-/slutförandefaser, inte ett universellt kommando för andra system ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).

Checksummor bevisar byteidentitet endast vid identisk serialisering. Export/import kan normalisera meddelanden, transformera metadata eller strukturera containrar på annat sätt. Därför kombinerar godkännandet tekniska summor med verksamhetsinvarianter: objektantal per klass, äldsta/nyaste tidsstämpel, total storlek, saknade/större objekt, behörighetsrelationer, slumpmässigt stickprov och avsiktligt svåra fall som stora MIME-meddelanden, kalenderupprepningar, Unicode, kryptering och Legal Hold.

När identitet och dataansvar har definierats kan löpande meddelandetrafik ledas mellan gammalt system och målsystem. Samexistens är då en temporär produktionsarkitektur med egna routing- och felscenarier.

## Samexistensarkitektur för meddelandesystem

En gemensam SMTP-domän på två plattformar kräver ett entydigt mottagar- och routingbeslut. Microsoft beskriver för hybriddrift ett gemensamt domännamnområde, säker e-posttrafik, gemensam adressbok och kalenderfunktioner; för samexistens med tredjepartsprodukter lyfts beslut om identitet, synkronisering och autentisering uttryckligen fram ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid), [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)).

Den lokala samexistensmatrisen besvarar:

- Vilken plattform känner till varje mottagare och med vilken typ?
- Vem ansvarar för okända mottagare och hur förhindras slingor?
- Vilket hopp signerar DKIM, tillämpar DLP/arkiv/disclaimer eller krypterar?
- Hur känner gammalt och nytt igen varandra som internt, autentiserat eller externt hopp?
- Var finns Free/Busy, delegering, gruppexpansion och Autodiscover/Service Discovery?
- Vilken plattform skriver vilka identitets- och adressattribut?
- Hur korreleras studsmeddelanden, karantän, journaler och Message Trace över båda sidorna?
- Vilken supportfunktion har en end-to-end-vy?

Varje övergång görs observerbar med [SMTP](/kb/smtp)-svar, kö-ID och Trace-/Message-ID. RFC 5321 fastställer att den mottagande servern övertar ansvaret efter lyckat slutförande av DATA. En rollback efter ett `250`-svar får inte glömma det accepterade meddelandet eller återinjicera det okontrollerat ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

## DNS, certifikat och externa avtal

DNS är en distribuerad urvalsmekanism, inte en global brytare. RFC 1034 beskriver cachning som en grundläggande del och överlåter TTL-avvägningen mellan aktualitet och kostnad till datakällan. Före en omställning minskas därför relevanta TTL:er i god tid; efter cutover förblir gamla vägar kontrollerade minst så länge som resolvers, negativa cachar, lokala hosts-filer, proxier och externa gateways kan använda gamla data ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).

MX, A/AAAA, CNAME, SRV och TXT samt interna/externa vyer ska kontrolleras. SPF måste auktorisera nya sändande IP-adresser utan att gamla källor tas bort för tidigt; DKIM-selector och DMARC-anpassning kontrolleras med riktiga meddelanden. Certifikat kräver namn, kedja, tillit, privat nyckel, Extended Key Usage och export-/HSM-förmåga. Historiska dekrypteringsnycklar för S/MIME-/OpenPGP följer sin egen livscykel. NIST SP 800-57 behandlar nyckeltillstånd, skydd och arkivering; ett plattformsbyte upphäver inte dessa krav ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Externa avtal omfattar partner-allowlisting, smarthost-autentiseringsuppgifter, API-redirect-URI:er, webhooks, SIEM-källor, backupagenter, övervakningsprober, brandväggar, NAT, reverse DNS och supportåtkomster. Varje avtal har en teknisk ägare och ett test före cutover.

## Pilot, vågor och beslutsgateways

En pilot är representativ, inte bara välvilligt utvald. Den innehåller stora och små brevlådor, delegeringar, Shared/Resource Mailboxes, mobila och äldre klienter som stöds, externa partner, kryptering, arkiv, avancerade användare, olika platser och minst ett kontrollerat negativt fall. Pilotpopulationen får samtidigt inte innehålla någon oåterkallelig unik funktion.

Varje våg har grindar:

- **Ready:** Mappning är fullständig, målobjekt finns, seed ligger inom laggmålet, klient/nät/policy är redo, återställning och support är bemannade.
- **Go:** Sista delta- och integritetskontrollen lyckades, ingen öppen blockerarklass, beroenden och tidsfönster bekräftade.
- **Hold:** Felfrekvens, lagg, kö, replikering eller supportbelastning utanför tröskelvärdet; inga ytterligare objekt startas.
- **Abort:** Dataintegritet, säkerhet, okontrollerbar e-postförlust eller andra i förväg definierade röda kriterier.
- **Accept:** Verksamhetsmässigt godkännande och efterobservationsperiod uppfyllda; objektet tillhör nu entydigt målet.

Vågor delas så att en störning begränsas: exempelvis efter plats, tenant, domän, databas, klienttyp, funktionsgrupp eller beroende. Slumpmässiga mängder kan vara representativa men är operativt svåra att återställa. Varje våg får en oföränderlig objektlista och en resultatpost.

Pilot och vågor ger bevisen för den faktiska cutovern. Återgången är bara realistisk så länge tillståndsändringar fortfarande kan återföras fullständigt till det gamla systemet.

## Cutover och rollbackgräns

Cutover-runbooken innehåller inte bara steg utan **förutsättningar, ägare, förväntat resultat, maximal varaktighet, avbrottskriterium och återväg**. En möjlig ordning är:

1. Frys ändringar, öppna incident-/kommunikationskanal och säkra utgångsmått.
2. Kontrollera Final Delta och mappningsfel; spärra källskrivvägar enligt modellen.
3. Testa åter mållyssnare, certifikat, identitet, policy och nedströmsberoenden.
4. Växla interna routing-/discoverykontroller, därefter externa DNS-/partnervägar.
5. Utför kontrollerade transaktioner för inbound, outbound, Submission, Access, Policy och kryptering.
6. Övervaka kö, SMTP-koder, autentisering, replikering, index, klientfel och övervakning.
7. Besluta Go/Hold/Abort och dokumentera tidpunkten.

**Rollback Point** ligger före tidpunkten då målet övertar ensamt skrivansvar som inte kan återföras utan förlust. Därefter är en blind omkoppling till källan ingen rollback utan en andra migrering. Nya målmeddelanden, kalenderändringar, behörigheter, nycklar och kötillstånd måste då återsynkroniseras, exporteras eller bevaras genom roll-forward. [Backup-och-DR-modellen](/kb/backup-dr) definierar konsistenspunkt, RPO/RTO och återställningsordning.

## Administratörsverktyg för bevis före/efter

Exemplen kontrollerar enskilda avtal. De körs från det faktiskt berörda ursprunget och med godkända testdata. Inloggningsuppgifter, personuppgifter och privata nycklar hör inte hemma oskyddade i ärenden eller shell-historik.

### Jämför DNS före och efter cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName autodiscover.example.ch -Type CNAME -DnsOnly -Server 192.0.2.53
Resolve-DnsName example.ch -Type TXT -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer example.ch MX
dig @192.0.2.53 +noall +answer autodiscover.example.ch CNAME
dig @192.0.2.53 +noall +answer example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) och [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) gör resolver och posttyp explicita. Interna och externa resolvers kontrolleras separat; ett korrekt svar från en offentlig resolver bevisar inte att produkttjänsten använder samma vy.

### Nå nya slutpunkter från det verkliga ursprunget

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection new-mx.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection new-api.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 new-mx.example.ch 25
nc -vz -w 5 new-api.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) och [`nc`](https://man.openbsd.org/nc) styrker endast DNS-/TCP-nåbarhet från det valda ursprunget. Brandvägg, NAT, käll-IP, TLS, autentisering och verksamhetsprotokoll kontrolleras därefter separat.

### Kontrollera TLS-identitet och protokollsvar

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) och [`curl`](https://curl.se/docs/manpage.html) kontrollerar TLS och en kontrollerad dialog. Båda använder sin egen trust-/proxy-/buildkonfiguration; den produktiva klienten måste dessutom testas med sin faktiska policy.

### Exportera och jämför identitetsmappning

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Identitätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -Filter * -Properties ObjectGUID,UserPrincipalName,mail,proxyAddresses,Enabled |
  Select-Object ObjectGUID,SamAccountName,UserPrincipalName,mail,proxyAddresses,Enabled |
  Export-Csv .\\identity-source.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'dc=example,dc=ch' '(objectClass=person)' \
  entryUUID uid userPrincipalName mail proxyAddresses</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) och [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) levererar identitetsattribut från respektive katalog. Export och fråga måste ta hänsyn till scope, paging, borttagna/inaktiverade objekt, flervärdesattribut och åtkomsträttigheter. Målavstämningen använder det överenskomna objektankaret, inte bara visningsnamnet.

### Verifiera exportartefakter byteexakt

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Hashprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\export\\mailbox-001.pst -Algorithm SHA256
Get-FileHash .\\export\\mapping.csv -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./export/mailbox-001.mbox
sha256sum ./export/mapping.csv</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) och [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) upptäcker ändringar i identiska filer. De styrker inte att käll- och målbrevlådan är semantiskt lika; för detta krävs objektantal, attribut, fellistor och verksamhetsmässiga stickprov.

### Säkra loggar i cutover-tidsfönstret

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T20:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 20:00:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) och [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) levererar tidsrefererade systemhändelser. Därtill kommer produkt-, migreringsbatch-, SMTP-, katalog-, databas- och API-loggar med kö-/objekt-/trace-ID. Före omstart eller rotation säkras flyktiga bevis.

## Godkännande och efterfas

Godkännandet kontrollerar inte bara «användaren kan logga in», utan sex dimensioner:

1. **Fullständighet:** Förväntade objekt, tidsintervall, mappar, attribut, regler, nycklar och behörigheter finns.
2. **Integritet:** Ingen otillåten ändring, dubblett, förlorad relation eller inkonsekvent dataansvar.
3. **Behörighet:** Användare, tjänster och administratörer kan utföra exakt de avsedda operationerna – inklusive negativa tester.
4. **Transaktion:** Inbound/outbound, Submission/Access, routing, policy, kryptering, karantän, arkiv och API fungerar end-to-end.
5. **Drift:** Prestanda, kö, replikering, index, säkerhetskopiering, återställning, patchning, support och kapacitet uppfyller målvärdena.
6. **Observerbarhet:** Loggar, mätvärden, traces, Message Tracking, larm och runbooks ser målvägen.

Efterfasen varar över relevanta cykler för TTL, cache, retry, delta, retention och verksamhetsbelastning. Gamla plattformar förblir aktiva endast i den i förväg definierade rollen: exempelvis för att tömma köer eller historisk läsning, inte för att omärkligt fortsätta ta emot nya data. Tillfälliga vidarebefordrare och samexistensdomäner får ett utgångsdatum och övervakning.

Först efter verksamhetsmässigt godkännande och efterfas börjar avvecklingen. Loggar, nycklar, DNS-poster, tjänstkonton och dolda integrationer måste tas bort eller arkiveras kontrollerat.

## Avveckling utan förlorade beroenden

Avveckling är en egen kontrollerad tillståndsändring. Före avstängning kontrolleras sista åtkomster, restköer, aktiva sessioner, jobb, DNS-/brandväggsträffar, tjänstkonton, säkerhetskopior, återställningsskyldigheter, Legal Holds, licenser och supportavtal. Därefter tas routing och discovery bort, åtkomster spärras, hemligheter och certifikat återkallas eller arkiveras, övervakningen anpassas och CMDB uppdateras.

Datamedier och molnresurser raderas eller görs kryptografiskt obrukbara enligt dataklass och avtal. NIST SP 800-88 beskriver Clear, Purge och Destroy som sanitiseringsmetoder och kräver ett beslut som är anpassat till medium och risk ([NIST SP 800-88 Rev. 1](https://csrc.nist.gov/pubs/sp/800/88/r1/final)). Säkerhetskopior och nycklar kan leva längre än produktinstansen; deras bevarande, läsbarhet och slutliga förstöring förblir dokumenterade.

## Migreringars tekniska utveckling

Tidiga infrastrukturmigreringar var ofta en engångsersättning av hårdvara eller programvara med ett stort omkopplingsfönster. Virtualisering och imagekopior underlättade rehost och relocate; standardiserade katalog-, API- och provisioneringsprotokoll möjliggjorde mer detaljerad samexistens. Moln- och SaaS-plattformar gjorde tenant-, identitets- och avtalsgränser till lika viktiga migreringsobjekt som servrar och databaser. 7-R-taxonomin beskriver dessa olika nivåer av ingrepp; Strangler Fig-mönstret beskriver den stegvisa funktionsersättningen ([AWS – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Tekniska framsteg eliminerar inte cutover. De flyttar den till mindre enheter: enskilda identiteter, domäner, API:er, dataklasser eller routingbeslut. Därigenom minskar den samtidiga skadeomfattningen, medan samexistens och dataansvar blir mer krävande. Modern migreringsarkitektur optimerar därför inte bara kopieringshastighet utan **entydighet i ansvaret, observerbarhet för varje övergång och en verklig åter- eller framåtväg**.

## Källor

- [NIST – SP 800-34 Rev. 1, beredskapsplanering](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [AWS Prescriptive Guidance – migreringsstrategier](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)
- [Microsoft – stegvis e-postmigrering](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)
- [Microsoft – Exchange Hybrid-distributioner](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)
- [IETF RFC 7643 – SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643)
- [IETF RFC 7644 – SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
- [Microsoft – migrering av brevlådor mellan tenants](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)
- [Microsoft – samexistens för tredjeparts-e-post](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [NIST – SP 800-57 Part 1 Rev. 5, nyckelhantering](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – man-sida för kommandoraden](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2-verktyg](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [NIST – SP 800-88 Rev. 1, sanering av datamedier](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
