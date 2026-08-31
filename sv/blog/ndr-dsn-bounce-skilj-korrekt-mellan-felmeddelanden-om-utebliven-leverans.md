---
title: "NDR, DSN, Bounce: skilj korrekt mellan felmeddelanden om utebliven leverans"
navTitle: "NDR & Bounces"
description: "NDR, DSN, Bounce, Reject, Backscatter: Begreppen kring misslyckad leverans används ofta synonymt, men betecknar olika saker. Vad RFC:erna definierar, vem som skapar vilket meddelande, hur en DSN är uppbyggd och varför skillnaden mellan Reject och Bounce avgör Backscatter."
date: "2026-08-28"
kategorie: "SMTP och e-postflöde"
timeToRead: "10 min lästid"
themen:
  - smtp-mailflow
produkte:
  - "uebergreifend"
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-skilj-korrekt-mellan-felmeddelanden-om-utebliven-leverans"
translationId: "article-5c5164049a129fa4"
aiPrompt: |
  Du bist mein Mailflow-Assistent. Ich füge dir gleich eine Unzustellbarkeitsmeldung (NDR/DSN) ein. Analysiere sie Schritt für Schritt: 1. Welcher Server hat die Meldung erzeugt (Reporting-MTA bzw. Generating server)? 2. Wurde die Mail in der SMTP-Session abgewiesen oder nach Annahme zurückgeschickt? 3. Was bedeuten SMTP-Antwortcode und Enhanced Status Code (RFC 3463) konkret? 4. Liegt die Ursache beim Absender, beim Empfänger oder auf dem Transportweg? 5. Welche nächsten Diagnose-Schritte empfiehlst du?
translationOf: ndr-dsn-bounce-unterschiede
url: https://rafaelpfister.ch/sv/blog/ndr-dsn-bounce-skilj-korrekt-mellan-felmeddelanden-om-utebliven-leverans
translationSourceHash: e526de6f4a454b4f4975eac3e8a406ab5b30314c624bf12c69f87bec99fdd0e7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-28T15:34:15.441Z
translationReview: automatic
---

# NDR, DSN, Bounce: skilj korrekt mellan felmeddelanden om utebliven leverans

Ett e-postmeddelande kommer inte fram och i ärendet står det omväxlande «Bounce», «NDR», «Mailer-Daemon» eller «felmeddelande från servern». I administratörsvardagen används dessa begrepp synonymt, trots att de betecknar olika saker: Ett Reject i SMTP-sessionen är inte ett returmeddelande, ett fördröjningsmeddelande är inte ett leveransfel och en läskvitto har inget alls med utebliven leverans att göra. Den som håller isär begreppen hittar orsaken snabbare, eftersom varje typ av meddelande säger något annat om var i transportkedjan problemet finns och vem som kan åtgärda det.

## DSN: samlingsbegreppet från RFC:erna

Det formella samlingsbegreppet heter Delivery Status Notification (DSN) och definieras i RFC 3461 till 3464. En DSN är ett maskingenererat e-postmeddelande som informerar avsändaren om leveransstatusen för meddelandet. Viktigt: En DSN rapporterar inte bara misslyckanden. Fältet `Action` i den maskinläsbara delen har fem värden:

| Action | Betydelse |
|---|---|
| `failed` | Leveransen har misslyckats slutgiltigt; e-postmeddelandet försöks inte igen |
| `delayed` | Leveransen är fördröjd; servern fortsätter att försöka |
| `delivered` | Levererat utan fel (leveranskvitto, endast på uttrycklig begäran) |
| `relayed` | Vidarebefordrat till en server som själv inte skapar några DSN:er |
| `expanded` | Överlämnat till en distributionslista och expanderat |

Meddelandet om utebliven leverans är alltså bara ett specialfall: en DSN med `Action: failed`. Microsoft kallar just detta specialfall för Non-Delivery Report (NDR). Begreppet NDR kommer från Exchange-världen, men används numera oberoende av tillverkare. För att vara exakt: Varje NDR är en DSN, men inte varje DSN är en NDR.

Fördröjningsmeddelandet (`Action: delayed`) förtjänar särskild uppmärksamhet, eftersom det regelbundet missförstås som ett leveransfel inom supporten. Ett typiskt ämne är «Delivery delayed» eller «Leveransen fördröjd». E-postmeddelandet ligger då fortfarande i kön på den sändande servern, som fortsätter att försöka, vanligtvis under en till två dagar. Först när köns livslängd löper ut följer det slutgiltiga NDR:et. En användare som skickar e-postmeddelandet igen efter ett fördröjningsmeddelande skapar dubbletter så snart målsystemet är tillgängligt igen.

## Reject eller Bounce: den viktigaste skillnaden

Innan de övriga begreppen följer behöver den centrala tekniska skiljelinjen förklaras, eftersom den avgör vilken server som skapar ett meddelande.

**Reject (avvisning i sessionen):** Den mottagande servern avvisar e-postmeddelandet redan under SMTP-sessionen, med en 5xx-svarskod på `RCPT TO` eller efter `DATA`. Den tar aldrig emot e-postmeddelandet och skapar själv inget returmeddelande. Ansvaret att informera avsändaren ligger hos den inlämnande servern: Den sändande MTA:n ser 5xx-svaret och skapar därefter NDR:et för sin lokala användare. NDR:et som användaren läser kommer i detta fall från den egna servern, men citerar felmeddelandet från motparten.

**Bounce (mottagning med senare returmeddelande):** Den mottagande servern tar emot e-postmeddelandet med `250 OK` och upptäcker först därefter att den inte kan leverera det, exempelvis eftersom postlådan inte finns, kvoten är full eller en efterföljande server avvisar det. Nu har den ansvaret för meddelandet och måste själv skicka en DSN till avsändaren. Detta efterföljande returmeddelande är Bounce i snäv bemärkelse.

För felsökning kan skillnaden användas direkt: Om den egna servern står som skapande system i NDR:et avvisades e-postmeddelandet i sessionen eller lämnade aldrig den egna miljön. Om en extern server står som avsändare av meddelandet har motparten först tagit emot e-postmeddelandet, och problemet finns efter dess mottagningspunkt, osynligt för avsändaren.

Från marknadsföringsområdet kommer ytterligare två Bounce-begrepp som inte finns i någon RFC: Hard Bounce för slutgiltiga fel (5xx, `Action: failed`) och Soft Bounce för tillfälliga fel (4xx, `Action: delayed`). För e-postplattformar är skillnaden central, eftersom Hard Bounces bör leda till omedelbar rensning av listan. Tekniskt sett är det samma mekanismer som ovan.

## Begreppen i korthet

| Begrepp | Vad det är | Vem skapar meddelandet | Standard |
|---|---|---|---|
| DSN | Samlingsbegrepp: statusmeddelande för leverans (failed, delayed, delivered, relayed, expanded) | Den MTA som bär ansvaret för e-postmeddelandet | RFC 3461 till 3464 |
| NDR | DSN med `Action: failed`; Microsofts benämning på meddelandet om utebliven leverans | Sändande MTA (efter Reject) eller mottagande MTA (efter mottagning) | RFC 3464, Microsoft-dokumentation |
| Reject | 5xx-avvisning under pågående SMTP-session; inget eget e-postmeddelande | Ingen; den sändande MTA:n gör ett NDR av det | RFC 5321 |
| Bounce | Returmeddelande efter att mottagning redan har skett | Mottagande MTA | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Marknadsföringsindelning: slutgiltigt (5xx) kontra tillfälligt (4xx) | som Bounce | ingen RFC |
| Fördröjningsmeddelande | DSN med `Action: delayed`; e-postmeddelandet ligger fortfarande i kön | Sändande eller vidarebefordrande MTA | RFC 3464 |
| Backscatter | NDR:er till förfalskade avsändaradresser, oftast utlösta av spam | Felkonfigurerade mottagande MTA:er | ingen RFC, anti-abuse-begrepp |
| MDN / läskvitto | Meddelande om visning eller borttagning av mottagaren | Mottagarens e-postklient | RFC 8098 |
| Frånvaromeddelande | Automatiskt svar från en nådd postlåda | Postlåde- eller grupprogramvaruserver | RFC 3834 |

## Så är en DSN uppbyggd

Standardkompatibla DSN:er använder MIME-typen `multipart/report; report-type=delivery-status` med tre delar: en förklaring som kan läsas av människor, en maskinläsbar del av typen `message/delivery-status` samt valfritt originalmeddelandet eller dess rubriker. Den maskinläsbara delen är mest värdefull för diagnostik, eftersom dess fält är standardiserade:

```text
Reporting-MTA: dns; mail01.example.net
Received-From-MTA: dns; client.example.org

Final-Recipient: rfc822; max.muster@example.com
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.com
Diagnostic-Code: smtp; 550 5.1.1 <max.muster@example.com>:
    Recipient address rejected: User unknown
```

| Fält | Betydelse |
|---|---|
| `Reporting-MTA` | Servern som skapade denna DSN; första indikatorn på ansvar |
| `Final-Recipient` | Mottagaradressen som statusen avser (ett block per mottagare) |
| `Action` | Ett av de fem statusvärdena (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code enligt RFC 3463, t.ex. `5.1.1` |
| `Remote-MTA` | Motparten som den rapporterande MTA:n kommunicerade med |
| `Diagnostic-Code` | Motpartens ordagranna SMTP-svar; ofta den mest informativa raden |

En DSN skickas alltid med tom Envelope-avsändare (`MAIL FROM:<>`). Det är inte slarv utan ett krav enligt RFC 5321: Den tomma avsändaren förhindrar att en ytterligare DSN följer på en DSN som inte kan levereras och att två servrar skickar felmeddelanden till varandra i all oändlighet. Detta leder till en konfigurationsregel: Ett e-postsystem får inte generellt avvisa e-postmeddelanden med tom Envelope-avsändare, eftersom legitima meddelanden om utebliven leverans då aldrig når de egna användarna.

Exchange och Exchange Online följer standarden för formatet, men paketerar innehållet i en egen presentation: Användaren ser en upparbetad sida med en förklaring i klartext, följt av «Generating server» (motsvarar `Reporting-MTA`) och rådata. För diagnostik är det alltid värt att titta i denna nedre, tekniska del.

## Läs Enhanced Status Codes

I fältet `Status` och oftast även i `Diagnostic-Code` finns en tredelad kod enligt RFC 3463: klass.ämne.detalj. Klassen anger bindningsgraden, ämne och detalj orsaken:

| Kodområde | Betydelse |
|---|---|
| `2.x.x` | Framgång (endast i leveranskvitton) |
| `4.x.x` | Tillfälligt fel; servern försöker igen |
| `5.x.x` | Slutgiltigt fel; inga fler försök |
| `x.1.x` | Adresseringsproblem, t.ex. `5.1.1` okänd mottagare, `5.1.10` domän utan MX |
| `x.2.x` | Postlådeproblem, t.ex. `5.2.2` postlådan är full, `5.2.3` meddelandet är för stort för postlådan |
| `x.3.x` | Problem i målsystemet, t.ex. `4.3.2` systemet tar för närvarande inte emot något |
| `x.4.x` | Nätverk och routning, t.ex. `4.4.1` inget svar, `4.4.7` köns livslängd har löpt ut |
| `x.5.x` | Protokollfel i SMTP-dialogen |
| `x.7.x` | Policy och säkerhet, t.ex. `5.7.1` relay nekad eller policyavvisning, `5.7.26` saknad autentisering (SPF/DKIM/DMARC) |

Den klassiska tresiffriga SMTP-svarskoden (exempelvis `550`) och Enhanced Status Code står ofta tillsammans på en rad: `550 5.7.1 ...`. Den tresiffriga koden styr den sändande serverns protokollbeteende, den utökade koden ger det diagnostiska beskedet. Vid motsägelser mellan kod och fritext är motpartens fritext ofta den mer exakta källan, eftersom många system använder generiska koder och skriver den verkliga orsaken i kommentaren, inklusive referens-ID:n för motpartens support.

Observera: `5.7.x`-avvisningar från rykte- och innehållsfilter säger ofta medvetet väldigt lite. Den som bara tittar på koden här letar på fel ställe; blocklistan eller filtertillverkaren i fritexten leder snabbare till målet.

## Backscatter: den skadliga typen av Bounce

Backscatter uppstår när en server först tar emot spam med förfalskad avsändare och sedan skickar ett NDR till den förfalskade adressen. NDR:et träffar då en utomstående vars adress har missbrukats av spammaren. Vid stora spamvågor får de drabbade tusentals NDR:er för e-postmeddelanden som de aldrig har skickat, och servrar som skapar sådana NDR:er i stor mängd hamnar själva på blocklistor (exempelvis Backscatterer-listan från UCEPROTECT).

Åtgärden följer direkt av skillnaden mellan Reject och Bounce: Allt som kan avvisas ska avvisas i SMTP-sessionen, inte skickas tillbaka efter mottagning. Konkret innebär det mottagarvalidering vid den yttersta mottagningspunkten (edge-gatewayen känner till de giltiga adresserna, genom kataloguppslag eller Recipient Callout, i stället för att ta emot allt och låta det misslyckas internt), avvisning av spam och skadlig programvara under sessionen i stället för NDR:er från karantän, samt att avstå från NDR:er för meddelanden som klassificerats som spam. Ett Reject skapar ingen Backscatter, eftersom 5xx-svaret vid en förfalskad avsändare når spammarens server, som inte skapar något NDR till offret av det.

## Vad som inte är ett meddelande om utebliven leverans

Tre typer av meddelanden hamnar regelbundet i samma kategori i ärenden, men hör inte dit:

**MDN (Message Disposition Notification, RFC 8098):** Läskvittot. Det skapas inte av transportsystemet utan av mottagarens e-postklient och rapporterar visning eller radering av meddelandet, inte dess leverans. MIME-typen heter därför `multipart/report; report-type=disposition-notification`. Ett uteblivet läskvitto säger inget om leveransen; de flesta klienter frågar användaren eller undertrycker MDN:er helt.

**Frånvaromeddelanden och autosvar (RFC 3834):** Ett frånvaromeddelande bevisar motsatsen till ett leveransfel, eftersom det förutsätter att e-postmeddelandet har nått postlådan. I ärendebeskrivningar («jag får ett automatiskt svar, kommer mitt e-postmeddelande fram?») är det värt att fråga vilket meddelande som faktiskt föreligger.

**Karantänaviseringar:** Meddelanden som karantänsammanställningen från Microsoft 365 eller en gateway informerar mottagaren om kvarhållna e-postmeddelanden. De går till mottagaren, inte till avsändaren, och följer ingen DSN-standard. Avsändaren får i detta scenario ofta ingenting alls, vilket förklarar fall där ett e-postmeddelande «försvinner utan felmeddelande».

## Checklista för diagnostik

Om ett meddelande finns, klargör följande i denna ordning:

1. Vilken typ är det: NDR (`Action: failed`), fördröjning (`Action: delayed`), MDN, autosvar eller karantänmeddelande? Vid ett fördröjningsmeddelande: vänta, skicka inte igen.
2. Vem skapade meddelandet (`Reporting-MTA` respektive «Generating server»)? Den egna servern innebär Reject eller internt fel, en extern server innebär mottagning med senare fel på motpartens sida.
3. Vad säger status- och Diagnostic-koden? Klass 4 kontra klass 5 skiljer tillfälligt från slutgiltigt, ämnet (`x.1` adress, `x.2` postlåda, `x.4` nätverk, `x.7` policy) avgränsar orsaken och motpartens fritext ger detaljerna.
4. Om inget meddelande finns trots att e-postmeddelandet inte kommer fram: kontrollera Message Tracking på det egna systemet och tänk på karantän eller tyst filtrering hos motparten.

Hur enskilda leveransvägar sedan kan återskapas riktat visar artiklarna om [Message Tracking och SMTP-diagnostik i kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder) samt [Mail Header Analyzer](https://rafaelpfister.ch/tools/mail-header-analyzer) för utvärdering av transportvägen för ett e-postmeddelande som har kommit fram.

## Källor

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP-utökning som låter avsändare begära och styra DSN:er.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definition av de tredelade statuskoderna (klass.ämne.detalj).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): DSN:ens struktur som multipart/report, fält som Action, Status och Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Grundregler för svarskoder, ansvarsskifte vid mottagning och tom Envelope-avsändare för felmeddelanden.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard för läskvitton, för avgränsning mot DSN:er.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Regler för autosvar såsom frånvaromeddelanden.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/non-delivery-reports-in-exchange-online/non-delivery-reports-in-exchange-online): NDR-struktur och kodlista ur Exchange Online-perspektiv.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blocklista för system som skapar Backscatter; förklarar listningskriterierna.
