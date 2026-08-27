---
title: "F5 BIG-IP som utgående proxy för massutskick av e-post: persistens, SNAT, timeouter och DNS-upplösning"
navTitle: "F5 massutskick"
description: "Ett massutskick med 1 000 e-postmeddelanden per minut går via en BIG-IP som utgående proxy till leverantörens relä. Artikeln förklarar varför Sticky Sessions inte tillför något här, hur leverantörens värdnamn löses upp korrekt via en FQDN-nod och vilka inställningar för SNAT, timeouter och anslutningsgränser som faktiskt avgör genomströmningen."
date: "2026-08-26"
kategorie: "Lastbalanserare"
timeToRead: "9 min. lästid"
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
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/sv/blog/f5-big-ip-som-utgaende-proxy-for-massutskick-av-e-post-persistens-snat-timeouter-och-dns
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:30:07.300Z
translationReview: required
---

En faktureringskörning eller ett nyhetsbrevsutskick med omkring 1 000 e-postmeddelanden per minut lämnar företagsnätverket, med en F5 BIG-IP som utgående proxy till leverantörens inlämningspunkt däremellan. BIG-IP belastningsfördelar inte mellan flera mål, utan vidarebefordrar trafiken. Just denna konstellation avgör vilka inställningar som är meningsfulla och vilka påstådda optimeringar som inte ger någon effekt.

## Arkitekturen i en mening

Utskicksystemen använder en intern Virtual Server-adress på BIG-IP som Smarthost, BIG-IP översätter avsändaradresserna via SNAT till en fast offentlig IP-adress och vidarebefordrar varje anslutning till leverantörens värdnamn. Load Balancing i egentlig mening sker inte på BIG-IP, eftersom poolen bara har en medlem. Det låter som en trivial konfiguration, men detaljbesluten (persistens, timeouter, SNAT-typ, DNS-upplösning) avgör om utskicken fungerar stabilt eller uppvisar oförklarliga avbrott under belastning.

## Är Sticky Sessions bättre? Nej, av två skäl

Frågan om sessionspersistens kommer från HTTP-världen, där en användare med kundvagn eller inloggningssession alltid måste hamna på samma backend. Överfört till SMTP saknar konceptet mening.

För det första avslutas SMTP tillståndslöst per anslutning: Varje anslutning hanterar en eller flera fullständiga transaktioner (MAIL FROM, RCPT TO, DATA) och avslutas med QUIT. Det finns inget tillstånd som måste ligga kvar på samma målsystem mellan anslutningar. Vilket system på leverantörssidan som accepterar nästa anslutning saknar betydelse för leveransen.

För det andra finns det helt enkelt inget att persistera på denna BIG-IP: Poolen innehåller exakt en medlem, leverantörens enda IP-adress. En persistensprofil skulle bara förbruka minne för en persistens-tabell och kosta en uppslagning vid varje anslutning, som alltid ger samma resultat. Rätt inställning är därför: Default Persistence Profile på None. Även om leverantören senare skulle publicera flera IP-adresser bakom värdnamnet vore persistens kontraproduktivt, eftersom den skulle förhindra fördelningen över dessa adresser och belasta enskilda mål ensidigt.

Avgörande för genomströmningen vid massutskick är avsändarens anslutningsprofil: få långlivade anslutningar med många meddelanden per anslutning i stället för en ny anslutning per e-postmeddelande; mer om detta längre ned.

## Virtual Server: FastL4 i stället för Full Proxy

För ren vidarebefordran av SMTP är en Performance-(Layer 4)-Virtual Server med FastL4-profil rätt val. BIG-IP behandlar då anslutningen till stor del i hårdvara eller i den accelererade sökvägen, utan att terminera TCP-anslutningen fullständigt. En standard-Virtual Server i Full Proxy-läge ger bara ett mervärde om du faktiskt vill ingripa i dataströmmen på BIG-IP, till exempel med en SMTP-säkerhetsprofil eller iRules på protokollnivå. För en Outbound Proxy till den egna avtalsleverantören är detta onödigt och skapar bara ytterligare felkällor.

Viktigt i båda fallen: aktivera ingen profil som skriver in i anslutningen. STARTTLS förhandlas direkt mellan utskicksystemen och leverantörens relä; varje instans som ändrar eller filtrerar byte äventyrar TLS-etableringen.

## DNS-upplösning: leverantörens värdnamn ska finnas som FQDN-nod i poolen

Leverantören har lämnat ett värdnamn, inte en IP-adress. Den närliggande reflexen att lösa upp IP-adressen en gång och registrera den statiskt som en nod är den sämsta varianten: Om leverantören byter adress (underhåll, flytt, DR-fall) stannar utskicket tills någon anpassar BIG-IP-konfigurationen. Det är precis därför FQDN-noder finns.

En FQDN-nod lagrar värdnamnet i stället för adressen. BIG-IP löser själv upp namnet, skapar en så kallad Ephemeral Node för varje returnerad adress och uppdaterar dessa automatiskt när DNS-svaret ändras. Som standard frågar den namnet igen när DNS-TTL har löpt ut; alternativt kan ett fast frågeintervall anges. Med Autopopulate aktiverat tar poolen även automatiskt emot flera A-records som medlemmar: Om leverantören senare utökar sin inlämning till flera adresser följer BIG-IP med utan någon konfigurationsändring.

Två förutsättningar glöms ofta bort. För det första behöver BIG-IP fungerande DNS-servrar i systemkonfigurationen (System, Configuration, Device, DNS); FQDN-noder använder systemets resolvers, inte någon DNS-cache från en Listener-profil. För det andra bör dessa resolvers faktiskt vara nåbara från management- respektive TMM-kontexten, annars förblir noden i status unresolved och poolen tom.

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

Utskicksystemen anger därefter 10.0.5.10 som Smarthost. Om du använder port 25 eller 587 bestäms av leverantören; BIG-IP-konfigurationen är identisk i båda fallen, endast porten ändras.

## SNAT: fast adress i stället för Automap

För utgående e-posttrafik måste källadressen vara under kontroll. SNAT Automap använder den flytande Self-IP-adressen för det utgående VLAN:et, och den kan ändras obemärkt vid nätverksändringar eller failover-ombyggnader. Leverantörer kopplar dock ofta inlämning till IP-allowlisting, och även utan formell allowlisting är ryktet knutet till källadressen. En dedikerad SNAT-pool med en fast tilldelad adress gör käll-IP:n till ett dokumenterat, stabilt konfigurationsobjekt.

När det gäller kapacitet: En enda SNAT-adress erbjuder cirka 64 000 samtidiga översättningar mot ett enda mål (en IP, en port), eftersom varje anslutning får en egen ephemeral källport. Med den belastningsprofil som beskrivs här, med några dussin samtidiga anslutningar, är det mer än tillräckligt. Portutarmning blir först ett problem när en felkonfigurerad avsändare öppnar en ny anslutning per e-postmeddelande och inte stänger den korrekt; då samlas översättningar i ett TIME-WAIT-liknande tillstånd. Ett sådant beteende åtgärdas hos avsändaren, inte med en andra SNAT-adress.

## Timeouter: den vanligaste orsaken till anslutningsavbrott under belastning

En bulkavsändare håller anslutningar öppna och skickar meddelande efter meddelande. Mellan två meddelanden kan pauser uppstå: Avsändaren genererar nästa block, reläet fördröjer godkännandet (Tarpitting, rester av Greylisting, interna köer). Idle-timeouten i FastL4-profilen är som standard 300 sekunder. Om en paus är längre rensar BIG-IP bort anslutningen, och avsändaren skriver till en anslutning som inte längre finns.

Två inställningar mildrar detta. För det första ska Idle-timeouten sättas till ett värde som överstiger realistiska pauser; för massutskick är 600 sekunder ett rimligt startvärde. Värdet bör inte vara godtyckligt högt, eftersom övergivna anslutningar annars samlas i anslutningstabellen. För det andra ska Reset on Timeout förbli aktiverat i profilen: BIG-IP bekräftar då rensningen med en TCP-reset, och den sändande MTA:n upptäcker omedelbart att anslutningen är borta, i stället för att hamna i en timeout och schemalägga om meddelandet först efter flera minuter.

Du har ingen påverkan på motpartens timeouter, men de ingår i helhetsbilden: Om leverantörens relä stänger anslutningar efter 120 sekunders inaktivitet hjälper inte en generös BIG-IP-timeout. Det minsta timeout-värdet längs hela vägen är avgörande; fråga vid tvekan leverantören och använd detta värde som planeringsunderlag.

## Anslutningsstrategi: få anslutningar, många meddelanden

Utan inlämningskrav från leverantören är en kort beräkning värdefull. 1 000 e-postmeddelanden per minut motsvarar cirka 17 per sekund. En SMTP-transaktion över en redan etablerad anslutning tar vid normal latens betydligt mindre än en halv sekund. Med 10 till 20 parallella anslutningar och exempelvis 100 meddelanden per anslutning innan avsändaren förnyar dem uppnås målhastigheten utan problem. På leverantörssidan finns normalt betydligt större anslutningskapacitet, men den delas med alla andra kunder. Få långlivade anslutningar med många transaktioner är därför inte bara effektivt (TCP- och TLS-etableringen bortfaller per meddelande), utan också det mest skonsamma sättet att använda andras infrastruktur.

Reglagen för detta finns i utskicksystemet, inte på BIG-IP: maximalt antal meddelanden per anslutning, maximalt antal parallella anslutningar till Smarthost och återanvändning av etablerade anslutningar. På BIG-IP kan helheten säkras med ett Connection Limit på poolmedlemmen, exempelvis 200 samtidiga anslutningar: Vid normal drift nås värdet aldrig, men en felkonfigurerad avsändare som plötsligt öppnar en anslutning per e-postmeddelande översvämmar då inte leverantörens relä obegränsat. Gränsen är ett skyddsnät, inte ett styrinstrument.

Mätningen visar om den inställda anslutningsprofilen också fungerar i praktiken: Anslutningar per minut och meddelanden per anslutning kan utvärderas från Message Tracking eller Connector-loggarna, enligt artikeln [Fastställ en e-postservers belastningsprofil](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln) beskriver. För ett belastningstest med en realistisk bulkbelastningsprofil (få sessioner, många meddelanden per session) passar smtp-source från Postfix-paketet bättre än HTTP-orienterade belastningsverktyg, eftersom det skapar just denna anslutningsprofil.

## Övervakning: belasta inte leverantören med hälsokontroller

En monitor på poolmedlemmen är meningsfull, så att BIG-IP upptäcker ett avbrott på leverantörssidan och rapporterar det korrekt. Följande gäller dock: Varje healthcheck är en verklig anslutning till leverantören och räknas där mot samma gränser som nyttotrafiken. En enkel TCP-monitor med måttligt intervall (30 sekunder eller mer) räcker gott. En fullständig SMTP-monitor som kontrollerar ända till bannern eller EHLO ger knappt någon ytterligare insikt, men skapar loggposter hos leverantören och i värsta fall frågor om varför en anslutning utan e-post kommer in var femte sekund.

## Checklista

| Inställning | Rekommendation |
|---|---|
| Persistensprofil | None; Sticky Sessions tillför inget för SMTP och ännu mindre för en pool med en enda medlem |
| Virtual Server-typ | Performance (Layer 4) med FastL4-profil, inget ingrepp i dataströmmen |
| Målnod | FQDN-nod med Autopopulate i stället för statisk IP; DNS-servrar konfigurerade på BIG-IP |
| SNAT | dedikerad SNAT-pool med fast adress känd av leverantören; ingen Automap |
| Idle-timeout | över de faktiska sändpauserna, startvärde 600 s; Reset on Timeout aktivt |
| Connection Limit | som skyddsnät på poolmedlemmen, t.ex. 200 |
| Monitor | TCP, intervall 30 s eller mer; ingen aggressiv SMTP-monitor |
| Avsändarkonfiguration | få parallella anslutningar, många meddelanden per anslutning; återanvändning aktiv |

Det korta svaret på den ursprungliga frågan är alltså: Nej, Sticky Sessions är inte bättre, de är i denna konstellation verkningslösa eller skadliga. Lösningens kvalitet avgörs av DNS-upplösningen av leverantörens värdnamn, av en stabil SNAT-adress, av timeouter som passar belastningsprofilen och av att utskicksystemen lämnar in sina 1 000 e-postmeddelanden per minut över få etablerade anslutningar i stället för över tusen enskilda.

## Källor

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Avsnitt 4.5.4 och transaktionsmodellen visar att flera e-posttransaktioner över en anslutning är det avsedda normalfallet.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): F5:s grundläggande artikel om SNAT, SNAT-pooler och portöversättning per mål.

3.  [tmsh-referens: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): dokumenterar FQDN-alternativen (name, autopopulate, interval) för noder och därmed för poolmedlemmar.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): belastningsgenerator som återskapar bulkavsändarens anslutningsprofil (få sessioner, många meddelanden).

5.  [Fastställ en e-postservers belastningsprofil](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): egen handledning om hur anslutningar per minut och meddelanden per anslutning utvärderas från Message Tracking.
