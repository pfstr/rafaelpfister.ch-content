---
title: "NDR, DSN, Bounce: skilj korrekt mellan felmeddelanden om utebliven leverans"
navTitle: "NDR och Bounces"
description: "NDR, DSN, Bounce, Reject, Backscatter: Termerna kring misslyckad leverans används ofta synonymt, men betecknar olika saker. Vad RFC:erna definierar, vem som skapar vilket meddelande, hur en DSN är uppbyggd och varför skillnaden mellan Reject och Bounce avgör Backscatter."
date: "2026-08-28"
kategorie: "SMTP och e-postflöde"
timeToRead: "10 min. lästid"
themen:
  - smtp-mailflow
protokolle:
  - "smtp"
  - "troubleshooting"
slug: "ndr-dsn-bounce-skilj-korrekt-mellan-felmeddelanden-om-utebliven-leverans"
translationId: "article-5c5164049a129fa4"
translationOf: ndr-dsn-bounce-unterschiede
translationSourceHash: 1500e27283dc4c53456810f133d12d81e1f60d7e874b6f3450b0a50fb47d4cf2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:21:20.023Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/ndr-dsn-bounce-skilj-korrekt-mellan-felmeddelanden-om-utebliven-leverans
---

# NDR, DSN, Bounce: skilj korrekt mellan felmeddelanden om utebliven leverans

Ett e-postmeddelande kommer inte fram och i ärendet står det omväxlande «Bounce», «NDR», «Mailer-Daemon» eller «felmeddelande från servern». I administratörsvardagen används dessa termer synonymt, trots att de betecknar olika saker: Ett Reject i SMTP-sessionen är inte ett returmeddelande, ett fördröjningsmeddelande är inte ett leveransfel och en läskvitto har överhuvudtaget inget med utebliven leverans att göra. Den som håller isär termerna korrekt hittar orsaken snabbare, eftersom varje meddelandetyp säger något annat om var i transportvägen problemet finns och vem som kan åtgärda det.

## DSN: den övergripande termen från RFC:erna

Den formella övergripande termen är Delivery Status Notification (DSN), definierad i RFC 3461 till 3464. En DSN är ett maskingenererat e-postmeddelande som informerar avsändaren om leveransstatusen för dennes meddelande. Viktigt: En DSN rapporterar inte bara misslyckanden. Fältet `Action` i den maskinläsbara delen har fem värden:

| Action | Betydelse |
|---|---|
| `failed` | Leveransen har slutgiltigt misslyckats; e-postmeddelandet försöks inte igen |
| `delayed` | Leveransen är fördröjd; servern fortsätter att försöka |
| `delivered` | Levererat utan problem (leveranskvitto, endast på uttrycklig begäran) |
| `relayed` | Vidarebefordrat till en server som själv inte skapar DSN:er |
| `expanded` | Överlämnat till en distributionslista och distribuerat |

Meddelandet om utebliven leverans är alltså bara ett specialfall: en DSN med `Action: failed`. Microsoft kallar just detta specialfall för Non-Delivery Report (NDR). Termen NDR kommer från Exchange-världen men används numera oberoende av tillverkare. För att vara exakt: Varje NDR är en DSN, men inte varje DSN är en NDR.

Fördröjningsmeddelandet (`Action: delayed`) förtjänar särskild uppmärksamhet eftersom det regelbundet missförstås som ett leveransfel i supporten. Ett typiskt ämne är «Delivery delayed» eller «Leveransen är fördröjd». E-postmeddelandet ligger då fortfarande i kön på den sändande servern, som fortsätter att försöka, vanligtvis i en till två dagar. Först när köns livslängd löper ut följer den slutgiltiga NDR:en. En användare som skickar e-postmeddelandet igen efter ett fördröjningsmeddelande skapar dubbletter så snart målsystemet blir tillgängligt igen.

## Reject eller Bounce: den viktigaste skillnaden

Innan de övriga termerna följer måste den centrala tekniska skiljelinjen förklaras, eftersom den avgör vilken server som skapar ett meddelande.

**Reject (avvisning i sessionen):** Den mottagande servern avvisar e-postmeddelandet redan under SMTP-sessionen, med en 5xx-svarskod på `RCPT TO` eller efter `DATA`. Den tar aldrig emot e-postmeddelandet och skapar själv inget returmeddelande. Skyldigheten att informera avsändaren ligger hos den inlämnande servern: Den sändande MTA:n ser 5xx-svaret och skapar därefter NDR:en för sin lokala användare. NDR:en som användaren läser kommer i detta fall från den egna servern, men citerar felmeddelandet från motparten.

**Bounce (mottagande med senare returmeddelande):** Den mottagande servern accepterar e-postmeddelandet med `250 OK` och upptäcker först därefter att det inte kan levereras, exempelvis eftersom postlådan inte finns, kvoten är full eller en efterföljande server avvisar det. Nu bär den ansvaret för meddelandet och måste själv skicka en DSN till avsändaren. Detta efterföljande returmeddelande är Bounce i snävare bemärkelse.

För felsökning kan skillnaden användas direkt: Om den egna servern anges som skapande system i NDR:en avvisades e-postmeddelandet under sessionen eller lämnade aldrig systemet. Om en extern server anges som avsändare av meddelandet har motparten först accepterat e-postmeddelandet, och problemet ligger efter dess accepteringspunkt, osynligt för avsändaren.

Från marknadsföringsområdet kommer ytterligare två Bounce-termer som inte förekommer i någon RFC: Hard Bounce för slutgiltiga fel (5xx, `Action: failed`) och Soft Bounce för tillfälliga fel (4xx, `Action: delayed`). För utskicksplattformar är skillnaden central eftersom Hard Bounces bör leda till omedelbar rensning av listan. Tekniskt sett är det samma mekanismer som ovan.

## Termerna i korthet

| Term | Vad det är | Vem skapar meddelandet | Standard |
|---|---|---|---|
| DSN | Övergripande term: statusmeddelande om leverans (failed, delayed, delivered, relayed, expanded) | Den MTA som ansvarar för e-postmeddelandet | RFC 3461 till 3464 |
| NDR | DSN med `Action: failed`; Microsoft-term för meddelande om utebliven leverans | Sändande MTA (efter Reject) eller mottagande MTA (efter mottagande) | RFC 3464, Microsoft-dokumentation |
| Reject | 5xx-avvisning under pågående SMTP-session; inget eget e-postmeddelande | Ingen; den sändande MTA:n gör om detta till en NDR | RFC 5321 |
| Bounce | Returmeddelande efter att meddelandet redan har accepterats | Mottagande MTA | RFC 5321, RFC 3464 |
| Hard/Soft Bounce | Indelning inom marknadsföring: slutgiltigt (5xx) kontra tillfälligt (4xx) | som Bounce | ingen RFC |
| Fördröjningsmeddelande | DSN med `Action: delayed`; e-postmeddelandet ligger fortfarande i kön | Sändande eller vidarebefordrande MTA | RFC 3464 |
| Backscatter | NDR:er till förfalskade avsändaradresser, vanligtvis utlösta av spam | Felkonfigurerade mottagande MTA:er | ingen RFC, term inom anti-missbruk |
| MDN / läskvitto | Meddelande om visning eller radering av mottagaren | Mottagarens e-postklient | RFC 8098 |
| Frånvaromeddelande | Automatiskt svar från en nådd postlåda | Postlåde- eller grupprogramserver | RFC 3834 |

## Så är en DSN uppbyggd

Standardkompatibla DSN:er använder MIME-typen `multipart/report; report-type=delivery-status` med tre delar: en läsbar förklaring, en maskinläsbar del av typen `message/delivery-status` och, valfritt, originalmeddelandet eller dess rubriker. Den maskinläsbara delen är den mest värdefulla för diagnos, eftersom dess fält är standardiserade:

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
| `Reporting-MTA` | Servern som skapade denna DSN; första ledtråden om ansvar |
| `Final-Recipient` | Mottagaradressen som statusen avser (ett block per mottagare) |
| `Action` | Ett av de fem statusvärdena (failed, delayed, delivered, relayed, expanded) |
| `Status` | Enhanced Status Code enligt RFC 3463, t.ex. `5.1.1` |
| `Remote-MTA` | Motparten som den rapporterande MTA:n kommunicerade med |
| `Diagnostic-Code` | Motpartens ordagranna SMTP-svar; ofta den mest informativa raden |

En DSN skickas alltid med tom Envelope-avsändare (`MAIL FROM:<>`). Det är inte slarv utan ett krav i RFC 5321: Den tomma avsändaren förhindrar att en ny DSN följer på en DSN som inte kan levereras och att två servrar skickar felmeddelanden fram och tillbaka i all oändlighet. Därav följer en konfigurationsregel: Ett e-postsystem får inte avvisa e-postmeddelanden med tom Envelope-avsändare generellt, annars når legitima meddelanden om utebliven leverans aldrig de egna användarna.

Exchange och Exchange Online följer standarden för formatet men presenterar innehållet på sitt eget sätt: Användaren ser en formaterad sida med en klartextförklaring, följd av «Generating server» (motsvarar `Reporting-MTA`) och rådata. För diagnos lönar det sig alltid att titta i denna nedre, tekniska del.

## Läs Enhanced Status Codes

I fältet `Status` och oftast även i `Diagnostic-Code` finns en tredelad kod enligt RFC 3463: klass.ämne.detalj. Klassen anger slutgiltigheten, ämne och detalj orsaken:

| Kodintervall | Betydelse |
|---|---|
| `2.x.x` | Framgång (endast i leveranskvitton) |
| `4.x.x` | Tillfälligt fel; servern försöker igen |
| `5.x.x` | Slutgiltigt fel; inga fler försök |
| `x.1.x` | Adresseringsproblem, t.ex. `5.1.1` okänd mottagare, `5.1.10` domän utan MX |
| `x.2.x` | Postlådeproblem, t.ex. `5.2.2` postlådan full, `5.2.3` meddelandet är för stort för postlådan |
| `x.3.x` | Problem i målsystemet, t.ex. `4.3.2` systemet accepterar för närvarande inget |
| `x.4.x` | Nätverk och routning, t.ex. `4.4.1` inget svar, `4.4.7` köns livslängd har löpt ut |
| `x.5.x` | Protokollfel i SMTP-dialogen |
| `x.7.x` | Policy och säkerhet, t.ex. `5.7.1` relä nekas eller policyavvisning, `5.7.26` saknad autentisering (SPF/DKIM/DMARC) |

Den klassiska tresiffriga SMTP-svarskoden (exempelvis `550`) och Enhanced Status Code står ofta tillsammans på en rad: `550 5.7.1 ...`. Den tresiffriga koden styr den sändande serverns protokollbeteende, medan den utökade koden ger diagnosinformationen. Vid motsägelser mellan kod och fritext är motpartens fritext ofta den mer exakta källan, eftersom många system använder generiska koder och skriver den egentliga orsaken i kommentaren, inklusive referens-ID:n för motpartens support.

Observera: `5.7.x`-avvisningar från rykte- och innehållsfilter säger ofta medvetet väldigt lite. Den som bara tittar på koden här letar på fel ställe; blocklistan eller filtertillverkaren i fritexten leder snabbare till målet.

## Backscatter: den skadliga typen av Bounce

Backscatter uppstår när en server först accepterar spam med förfalskad avsändare och sedan skickar en NDR till den förfalskade adressen. NDR:en träffar då en oinblandad person vars adress missbrukats av spammaren. Vid stora spamvågor får de drabbade tusentals NDR:er för e-postmeddelanden som de aldrig har skickat, och servrar som skapar sådana NDR:er i stor mängd hamnar själva på blocklistor (exempelvis Backscatterer-listan från UCEPROTECT).

Åtgärden följer direkt av skillnaden mellan Reject och Bounce: Allt som kan avvisas ska avvisas under SMTP-sessionen, inte skickas tillbaka efter mottagande. Konkret innebär det mottagarvalidering vid den yttersta accepteringspunkten (Edge-gatewayen känner till de giltiga adresserna, genom kataloguppslag eller Recipient Callout, i stället för att acceptera allt och låta det misslyckas internt), avvisning av spam och skadlig programvara under sessionen i stället för karantän-NDR:er samt att avstå från NDR:er för meddelanden som klassificerats som spam. Ett Reject skapar ingen Backscatter, eftersom 5xx-svaret vid förfalskad avsändare hamnar hos spammarens server, som inte skapar någon NDR till offret av det.

## Vad som inte är ett meddelande om utebliven leverans

Tre typer av meddelanden hamnar regelbundet i samma kategori i ärenden, men hör inte dit:

**MDN (Message Disposition Notification, RFC 8098):** Läskvittot. Det skapas inte av transportsystemet utan av mottagarens e-postklient och rapporterar visning eller radering av meddelandet, inte dess leverans. MIME-typen heter följaktligen `multipart/report; report-type=disposition-notification`. Ett uteblivet läskvitto säger inget om leveransen; de flesta klienter frågar användaren eller undertrycker MDN:er helt.

**Frånvaromeddelanden och autosvar (RFC 3834):** Ett frånvaromeddelande bevisar motsatsen till ett leveransfel, eftersom det förutsätter att e-postmeddelandet nått postlådan. I ärendebeskrivningar («jag får ett automatiskt svar, kommer mitt e-postmeddelande fram?») är det värt att fråga vilket meddelande det exakt gäller.

**Karantänmeddelanden:** Meddelanden som karantänsammanställningen från Microsoft 365 eller en gateway informerar mottagaren om kvarhållna e-postmeddelanden. De går till mottagaren, inte till avsändaren, och följer ingen DSN-standard. Avsändaren får ofta ingenting alls i detta scenario, vilket förklarar fall där ett e-postmeddelande «försvinner utan felmeddelande».

## Checklista för diagnos

Om ett meddelande finns, klargör följande i denna ordning:

1. Vilken typ rör det sig om: NDR (`Action: failed`), fördröjning (`Action: delayed`), MDN, autosvar eller karantänmeddelande? Vid ett fördröjningsmeddelande: vänta, skicka inte igen.
2. Vem skapade meddelandet (`Reporting-MTA` respektive «Generating server»)? Den egna servern innebär Reject eller internt fel, en extern server innebär accepterande med senare misslyckande på motpartens sida.
3. Vad säger Status och Diagnostic-Code? Klass 4 kontra klass 5 skiljer tillfälligt från slutgiltigt, ämnet (`x.1` adress, `x.2` postlåda, `x.4` nätverk, `x.7` policy) begränsar orsaken och motpartens fritext ger detaljerna.
4. Om inget meddelande alls saknas trots att e-postmeddelandet inte kommer fram: kontrollera Message Tracking i det egna systemet och tänk på karantän eller tyst filtrering på motpartens sida.

Hur enskilda leveransvägar sedan kan återskapas specifikt visas i artiklarna om [Message Tracking och SMTP-diagnos i kommandogeneratorn](https://rafaelpfister.ch/tools/command-builder) samt [Mail Header Analyzer](https://rafaelpfister.ch/tools/header-analyzer) för utvärdering av transportvägen för ett framkommet e-postmeddelande.

## Ändringshistorik

- **25 oktober 2026:** Korrigerat: Länk till Mail Header Analyzer satt till rätt adress. Källänk uppdaterad till den nya adressen hos Microsoft Learn.

## Källor

1.  [RFC 3461: SMTP Service Extension for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3461): SMTP-utökning som gör det möjligt för avsändare att begära och styra DSN:er.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Definition av de tredelade statuskoderna (klass.ämne.detalj).

3.  [RFC 3464: An Extensible Message Format for Delivery Status Notifications](https://www.rfc-editor.org/rfc/rfc3464): DSN:ens uppbyggnad som multipart/report, fält som Action, Status och Diagnostic-Code.

4.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): Grundregler för svarskoder, ansvarsövergång vid accepterande och tom Envelope-avsändare för felmeddelanden.

5.  [RFC 8098: Message Disposition Notification](https://www.rfc-editor.org/rfc/rfc8098): Standard för läskvitton, för avgränsning från DSN:er.

6.  [RFC 3834: Recommendations for Automatic Responses to Electronic Mail](https://www.rfc-editor.org/rfc/rfc3834): Regler för autosvar som frånvaromeddelanden.

7.  [Microsoft Learn: Email non-delivery reports and SMTP errors in Exchange Online](https://learn.microsoft.com/en-us/troubleshoot/exchange/email-delivery/ndr/non-delivery-reports-in-exchange-online): NDR-struktur och kodlista ur perspektivet Exchange Online.

8.  [UCEPROTECT Backscatterer](https://www.backscatterer.org/): Blocklista för system som skapar Backscatter; förklarar kriterierna för listning.
