---
title: "F5 BIG-IP som utgående proxy för massutskick av e-post: persistens, SNAT, timeouter och DNS-upplösning"
navTitle: "F5 massutskick"
description: "Ett massutskick med 1 000 e-postmeddelanden per minut går via en BIG-IP som utgående proxy till leverantörens relay. Artikeln förklarar varför sticky sessions inte tillför något här, hur leverantörens värdnamn löses upp korrekt med en FQDN-nod och vilka inställningar för SNAT, timeouter och anslutningsgränser som faktiskt avgör genomströmningen."
date: "2026-08-26"
kategorie: "Lastbalanserare"
timeToRead: "9 min lästid"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-som-utgaende-proxy-for-massutskick-av-e-post-persistens-snat-timeouter-och-dns"
featured: true
translationId: "article-ee5e63e82ffd2604"
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:24:12.193Z
translationReview: required
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/f5-big-ip-som-utgaende-proxy-for-massutskick-av-e-post-persistens-snat-timeouter-och-dns
---

En faktureringskörning eller ett nyhetsbrevsutskick med omkring 1 000 e-postmeddelanden per minut lämnar företagsnätet, med en F5 BIG-IP som utgående proxy till leverantörens inlämningspunkt däremellan. BIG-IP:en distribuerar inte till flera mål, utan vidarebefordrar. Just denna konstellation avgör vilka inställningar som är meningsfulla och vilka påstådda optimeringar som inte har någon effekt.

## Arkitekturen i en mening

Utskickssystemen använder en intern Virtual Server-adress på BIG-IP:en som smarthost, BIG-IP:en översätter avsändaradresserna med SNAT till en fast offentlig IP-adress och vidarebefordrar varje anslutning till leverantörens värdnamn. Lastbalansering i egentlig mening sker inte på BIG-IP:en, eftersom poolen bara har en medlem. Det låter som en trivial konfiguration, men detaljbesluten (persistens, timeouter, SNAT-typ, DNS-upplösning) avgör om utskicket fungerar stabilt eller får oförklarliga avbrott under belastning.

## Är sticky sessions bättre? Nej, av två skäl

Frågan om sessionspersistens kommer från HTTP-världen, där en användare med varukorg eller inloggningssession alltid måste hamna på samma backend. Överfört till SMTP saknar konceptet mening.

För det första avslutas SMTP tillståndslöst per anslutning: Varje anslutning hanterar en eller flera kompletta transaktioner (MAIL FROM, RCPT TO, DATA) och avslutas med QUIT. Det finns inget tillstånd som behöver finnas kvar på samma målsystem mellan anslutningar. Vilket system på leverantörssidan som tar emot nästa anslutning saknar betydelse för leveransen.

För det andra finns det helt enkelt inget att göra persistent på denna BIG-IP: Poolen innehåller exakt en medlem, leverantörens enda IP-adress. En persistensprofil skulle bara använda minne för en persistens-tabell och kosta en uppslagning vid varje anslutning, som alltid ger samma resultat. Rätt inställning är därför: Default Persistence Profile på None. Även om leverantören senare skulle publicera flera IP-adresser bakom värdnamnet vore persistens kontraproduktivt, eftersom den skulle förhindra fördelning över dessa adresser och ensidigt belasta enskilda mål.

Avgörande för genomströmningen vid massutskick är avsändarens anslutningsprofil: få långlivade anslutningar med många meddelanden per anslutning i stället för en ny anslutning per e-postmeddelande; mer om detta nedan.

## Virtual Server: FastL4 i stället för full proxy

För ren vidarebefordran av SMTP är en Performance-(Layer 4)-Virtual Server med FastL4-profil rätt val. BIG-IP:en behandlar då anslutningen till stor del i hårdvara respektive i den accelererade sökvägen, utan att terminera TCP-anslutningen fullt ut. En standard-Virtual Server i full proxy-läge ger bara ett mervärde om du verkligen vill ingripa i dataströmmen på BIG-IP:en, exempelvis med en SMTP-säkerhetsprofil eller iRules på protokollnivå. För en utgående proxy till den egna avtalsleverantören är det onödigt och skapar bara ytterligare felkällor.

Viktigt i båda fallen: aktivera ingen profil som skriver i anslutningen. STARTTLS förhandlas direkt mellan utskickssystemen och leverantörens relay; varje instans som ändrar eller filtrerar byte äventyrar TLS-etableringen.

## DNS-upplösning: leverantörens värdnamn ska vara en FQDN-nod i poolen

Leverantören har angett ett värdnamn, inte en IP-adress. Den närliggande reflexen att lösa upp IP-adressen en gång och registrera den statiskt som nod är den sämsta varianten: Om leverantören byter adress (underhåll, flytt, DR-fall) stannar utskicket tills någon ändrar BIG-IP-konfigurationen. Det är just därför FQDN-noder finns.

En FQDN-nod lagrar värdnamnet i stället för adressen. BIG-IP:en löser själv upp namnet, skapar en så kallad ephemeral node för varje returnerad adress och uppdaterar dessa automatiskt när DNS-svaret ändras. Som standard frågar den efter namnet igen var 3 600:e sekund (en timme); med `interval ttl` följer den i stället TTL:n i DNS-svaret, alternativt kan ett annat fast frågeintervall anges. Med Autopopulate aktiverat tar poolen automatiskt även upp flera A-poster som medlemmar: Om leverantören senare utökar sin inlämning till flera adresser följer BIG-IP:en med utan konfigurationsändring.

Två förutsättningar glöms ofta bort. För det första behöver BIG-IP:en fungerande DNS-servrar i systemkonfigurationen (System, Configuration, Device, DNS); FQDN-noder använder systemets resolvrar, inte exempelvis en DNS-cache från en listener-profil. För det andra bör dessa resolvrar faktiskt vara nåbara från management- respektive TMM-kontexten, annars förblir noden i status unresolved och poolen tom.

Konfigurationen i tmsh ser ut så här (adresser och namn är exempel):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

Utskickssystemen anger sedan 10.0.5.10 som smarthost. Om du använder port 25 eller 587 avgörs av leverantören; BIG-IP-konfigurationen är identisk i båda fallen, endast porten ändras.

## SNAT: fast adress i stället för Automap

För utgående e-posttrafik ska källadressen vara under kontroll. SNAT Automap använder den flytande Self-IP-adressen för det utgående VLAN:et, och den kan ändras obemärkt vid nätverksändringar eller failover-ombyggnader. Leverantörer kopplar dock ofta inlämningen till IP-allowlisting, och även utan formell allowlisting är ryktet knutet till källadressen. En dedikerad SNAT-pool med en fast tilldelad adress gör käll-IP:n till ett dokumenterat och stabilt konfigurationsobjekt.

När det gäller kapacitet erbjuder en enda SNAT-adress mot ett enskilt mål (en IP, en port) omkring 64'000 samtidiga översättningar, eftersom varje anslutning får en egen ephemeral källport. Med den belastningsprofil som beskrivs här, med några få dussin samtidiga anslutningar, är det tillräckligt med mycket god marginal. Portuttömning blir först ett problem när en felkonfigurerad avsändare öppnar en ny anslutning per e-postmeddelande och inte stänger dem korrekt; då samlas översättningar i ett TIME-WAIT-liknande tillstånd. Sådant beteende åtgärdar du hos avsändaren, inte med en andra SNAT-adress.

## Timeouter: den vanligaste orsaken till anslutningsavbrott under belastning

En bulkavsändare håller anslutningar öppna och skickar meddelande efter meddelande genom dem. Pauser kan uppstå mellan två meddelanden: Avsändaren genererar nästa block, reläet fördröjer mottagningen (tarpitting, kvarvarande greylisting, interna köer). Idle-timeouten i FastL4-profilen är som standard 300 sekunder. Om en paus är längre än så rensar BIG-IP:en anslutningen, och avsändaren skriver till en anslutning som inte längre finns.

Två inställningar mildrar detta. För det första ska Idle-timeouten sättas till ett värde som överstiger realistiska pauser; för massutskick är 600 sekunder ett rimligt startvärde. Värdet bör inte vara godtyckligt högt, eftersom övergivna anslutningar annars samlas i anslutningstabellen. För det andra ska Reset on Timeout vara aktiverat i profilen: BIG-IP:en bekräftar då rensningen med en TCP-reset, och den sändande MTA:n märker omedelbart att anslutningen är borta, i stället för att hamna i en timeout och planera om meddelandet först efter flera minuter.

Du har ingen påverkan på motpartens timeouter, men de ingår i helhetsbilden: Om leverantörens relay stänger anslutningar efter 120 sekunders inaktivitet hjälper ingen generös BIG-IP-timeout. Det lägsta timeout-värdet längs hela vägen är avgörande; fråga vid tveksamhet leverantören och använd det värdet som planeringsunderlag.

## Anslutningsstrategi: få anslutningar, många meddelanden

Utan inlämningskrav från leverantören är en kort beräkning värdefull. 1 000 e-postmeddelanden per minut motsvarar omkring 17 per sekund. En SMTP-transaktion över en redan etablerad anslutning tar vid normal latens betydligt mindre än en halv sekund. Med 10 till 20 parallella anslutningar och exempelvis 100 meddelanden per anslutning innan avsändaren förnyar dem nås målhastigheten utan problem. På leverantörssidan finns normalt betydligt större anslutningskapacitet, men den delas med alla andra kunder. Få långlivade anslutningar med många transaktioner är därför inte bara effektivt (TCP- och TLS-etablering bortfaller per meddelande), utan också det mest skonsamma sättet att använda främmande infrastruktur.

Reglagen för detta finns i utskickssystemet, inte på BIG-IP:en: maximalt antal meddelanden per anslutning, maximalt antal parallella anslutningar till smarthosten, återanvändning av etablerade anslutningar. På BIG-IP:en kan helheten säkras med ett Connection Limit på pool-medlemmen, exempelvis 200 samtidiga anslutningar: I normal drift nås värdet aldrig, men en felkonfigurerad avsändare som plötsligt öppnar en anslutning per e-postmeddelande översvämmar då inte leverantörens relay utan begränsning. Gränsen är ett skyddsnät, inte ett styrinstrument.

Om den inställda anslutningsprofilen också uppnås i praktiken visar mätningen: Anslutningar per minut och meddelanden per anslutning kan utvärderas från Message Tracking respektive Connector-loggarna, enligt beskrivningen i artikeln [Fastställa en e-postservers belastningsprofil](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). För ett belastningstest med en realistisk bulkbelastningsprofil (få sessioner, många meddelanden per session) passar smtp-source från Postfix-paketet bättre än HTTP-orienterade belastningsverktyg, eftersom det skapar just denna anslutningsprofil.

## Övervakning: belasta inte leverantören med hälsokontroller

En monitor på pool-medlemmen är meningsfull så att BIG-IP:en upptäcker ett avbrott på leverantörssidan och rapporterar det korrekt. Följande gäller dock: Varje hälsokontroll är en verklig anslutning till leverantören och räknas där mot samma gränser som nyttotrafiken. En enkel TCP-monitor med måttligt intervall (30 sekunder eller mer) är fullt tillräcklig. En fullständig SMTP-monitor som kontrollerar ända till bannern eller EHLO ger knappt någon ytterligare insikt, men skapar loggposter på leverantörssidan och i värsta fall frågor om varför en anslutning utan e-post kommer in var femte sekund.

## Checklista

| Inställning | Rekommendation |
|---|---|
| Persistensprofil | None; sticky sessions tillför inget för SMTP och än mindre i en pool med en medlem |
| Virtual Server-typ | Performance (Layer 4) med FastL4-profil, inget ingrepp i dataströmmen |
| Målnod | FQDN-nod med Autopopulate i stället för statisk IP; DNS-servrar konfigurerade på BIG-IP:en |
| SNAT | dedikerad SNAT-pool med fast adress känd av leverantören; ingen Automap |
| Idle-timeout | över de faktiska sändpauserna, startvärde 600 s; Reset on Timeout aktivt |
| Connection Limit | som skyddsnät på pool-medlemmen, t.ex. 200 |
| Monitor | TCP, intervall 30 s eller mer; ingen aggressiv SMTP-monitor |
| Avsändarkonfiguration | få parallella anslutningar, många meddelanden per anslutning; återanvändning aktiv |

Det korta svaret på utgångsfrågan är alltså: Nej, sticky sessions är inte bättre, de är i denna konstellation verkningslösa till skadliga. Lösningens kvalitet avgörs av DNS-upplösningen av leverantörens värdnamn, en stabil SNAT-adress, timeouter som passar belastningsprofilen och att utskickssystemen lämnar in sina 1 000 e-postmeddelanden per minut via få etablerade anslutningar i stället för tusen enskilda.

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Standardintervallet för namnupplösning av FQDN-noder är 3 600 sekunder, inte DNS-TTL; BIG-IP:en följer endast TTL:n med inställningen interval ttl.

## Källor

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Avsnitt 4.5.4 och transaktionsmodellen visar att flera e-posttransaktioner över en anslutning är det avsedda normalfallet.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5-grundartikel om SNAT, SNAT-pooler och portöversättning per mål.

3.  [tmsh-referens: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): dokumenterar FQDN-alternativen (name, autopopulate, interval) för noder och därmed för pool-medlemmar, inklusive standardintervallet på 3 600 sekunder och värdet `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): belastningsgenerator som efterliknar bulkavsändarens anslutningsprofil (få sessioner, många meddelanden).

5.  [Fastställa en e-postservers belastningsprofil](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): egen guide om hur anslutningar per minut och meddelanden per anslutning utvärderas från Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html): bekräftar 3 600 sekunder som det faktiska standardvärdet för FQDN-frågeintervallet; den tidigare hjälpuppgiften TTL var fel.
