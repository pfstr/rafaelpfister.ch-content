---
title: "RustDesk: konfigurera alternativet med öppen källkod till TeamViewer"
navTitle: "Konfigurera RustDesk"
description: "RustDesk är programvara för fjärrsupport med öppen källkod under AGPL, gratis och möjlig att självhosta. Så här installerar du klienten i Windows (även obevakat via MSI), hur anslutningen upprättas via den offentliga förmedlingsservern, en egen server eller en direktanslutning, vilka funktioner som behövs i det dagliga supportarbetet och var gränserna för kostnadsfri användning går."
date: "2026-09-01"
kategorie: "Fjärrsupport och support"
timeToRead: "9 min läsning"
themen:
  - fernwartung
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-konfigurera-det-oppna-teamviewer-alternativet"
translationId: "article-425ae4b8d562ae41"
translationOf: rustdesk-teamviewer-alternative
translationSourceHash: 3c44a4d5fc03c777c33f77525bcaebe75596ddde249f697041599b54e77cae2b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T10:00:07.786Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/rustdesk-konfigurera-det-oppna-teamviewer-alternativet
---

# RustDesk: konfigurera alternativet med öppen källkod till TeamViewer

TeamViewer och AnyDesk täcker fjärrsupport på ett tillförlitligt sätt, men kräver en licens för kommersiell användning och priserna stiger med antalet enheter som hanteras. RustDesk är ett alternativ under licensen AGPL-3.0: öppen källkod, gratis och utan licenskrav. Klienten körs på Windows, macOS, Linux, Android och iOS samt i webbläsaren. Den är skriven i Rust, med gränssnittet i Flutter.

Den avgörande skillnaden jämfört med de kommersiella produkterna ligger i förmedlingen: RustDesk skiljer klienten från serverinfrastrukturen. Du kan använda den kostnadsfria offentliga förmedlingsservern, driva en egen server eller upprätta en direktanslutning helt utan förmedlingsserver. Därmed kan RustDesk användas från en enskild dator till en självhostad supportplattform, utan att anslutningsdata måste gå via en leverantör.

## De tre anslutningstyperna

Innan du installerar bör du fastställa anslutningstypen, eftersom konfigurationen och öppna portar beror på den.

| Anslutningstyp | Så fungerar den | När den passar |
|---|---|---|
| Offentlig förmedlingsserver | Två klienter hittar varandra via ID:t (ett niosiffrigt nummer) på RustDesk-servern, och anslutningen går direkt eller via en relay | Snabb start, test, tillfällig privat support |
| Egen server (self-hosted) | Du driver serverkomponenterna `hbbs` (förmedling) och `hbbr` (relay) själv, och alla klienter anger deras adress | Kommersiell användning, många enheter, full kontroll över data |
| Direktanslutning (Direct IP Access) | Klienten ansluter direkt till motpartens IP-adress utan förmedlingsserver | Båda enheterna kan nå varandra i samma nätverk eller via VPN |

Den offentliga servern är uttryckligen avsedd för tester och privat bruk. För produktiv, kommersiell drift rekommenderar projektet en egen server, bland annat eftersom den offentliga tjänsten är begränsad och inte har någon tillgänglighetsgaranti.

## Installation i Windows

Du laddar ned installationsprogrammet från den officiella källan, projektets GitHub-releaser (`github.com/rustdesk/rustdesk`). För Windows finns en körbar fil och ett MSI-paket. För interaktiv installation räcker det med ett dubbelklick. Om du vill distribuera RustDesk på flera datorer eller i bakgrunden använder du MSI med en tyst installation. Filnamnet innehåller versionsnumret; sedan den 30 september 2026 är den aktuella versionen 1.5.0:

```powershell
msiexec /i rustdesk-1.5.0-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/i <paket>` | Installerar det angivna MSI-paketet |
| `/qn` | Inget gränssnitt, inga dialogrutor (tyst) |
| `/norestart` | Förhindrar automatisk omstart efter installationen |

</details>

Den tysta installationen konfigurerar tjänsten `RustDesk`, som körs vid systemstart och möjliggör obevakad åtkomst. Efter installationen kan du läsa ut enhetens ID via kommandoraden utan att öppna gränssnittet:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Du anger också ett fast lösenord för obevakad åtkomst via kommandoraden. Välj ett separat och tillräckligt långt lösenord, inte användarens inloggningslösenord:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--get-id` | Skriver ut enhetens niosiffriga RustDesk-ID |
| `--password <wert>` | Anger det fasta lösenordet för obevakad åtkomst |
| `--silent-install` | Installerar den körbara versionen (`.exe`) utan gränssnitt som tjänst |

</details>

## Ange en egen server

Om du driver en egen förmedlingsserver anger klienterna dess adress och den offentliga nyckeln. I gränssnittet finns detta under nätverksinställningarna som ID-server, relay-server och nyckel. För massdistribution kan konfigurationen även anges som fil eller via miljövariabler, så att varje klient startar förkonfigurerad.

En egen server behöver de två komponenterna `hbbs` och `hbbr`, som vanligtvis körs som Docker-containrar. Båda kräver öppna portar så att klienter kan registrera sig och använda en relay.

| Port | Protokoll | Komponent och syfte |
|---|---|---|
| 21114 | TCP | Webbgränssnitt för Pro-versionen (endast där) |
| 21115 | TCP | `hbbs`, test av NAT-typ |
| 21116 | TCP och UDP | `hbbs`, registrering (UDP) och anslutningsupprättande (TCP) |
| 21117 | TCP | `hbbr`, relay-trafik |
| 21118, 21119 | TCP | Stöd för webbklienter |

Öppna bara de portar som din anslutningstyp faktiskt behöver och begränsa åtkomsten via brandväggen till de nätverk från vilka support ges.

## Direktanslutning utan förmedlingsserver

Om båda enheterna kan nå varandra i samma nätverk eller via VPN fungerar RustDesk helt utan förmedlingsserver. Aktivera direktåtkomst på målenheten (i gränssnittet under säkerhet som "Aktivera IP-direktåtkomst", internt reglaget `direct-server`). Klienten lyssnar då på standardporten 21118 (TCP). I anslutningsfönstret anger du motpartens IP-adress i stället för ID:t.

Begränsa direktåtkomsten via brandväggen till nätverket som du ansluter från. Om åtkomsten går via VPN ska du endast öppna porten för VPN-adressområdet, inte för hela internet.

## Funktioner i det dagliga supportarbetet

RustDesk täcker de funktioner som behövs för fjärrsupport i vardagen:

- Skärmöverföring och fjärrstyrning av tangentbord och mus, med val av bildskärm vid flera skärmar.
- Filöverföring i båda riktningarna via ett tvådelat fönster.
- Textchatt under sessionen.
- Obevakad åtkomst med ett fast lösenord, för enheter utan en närvarande användare.
- Sessionsinspelning som videofil, automatiskt om så önskas.
- TCP-tunnlar och vidarebefordran för att nå enskilda tjänster hos motparten lokalt.
- Adressbok och flera sparade enheter, lokalt i gratisversionen och delade på serversidan i Pro-versionen.

För assisterad support är detta viktigt: Som standard frågar RustDesk på motpartens sida om anslutningen ska accepteras och visar under sessionen att åtkomst pågår. Personen vid enheten är alltså informerad. Först ett fast lösenord för obevakad åtkomst tar bort bekräftelsefrågan. Använd obevakad åtkomst endast på enheter vars användare vet att programvaran är installerad och vad den används till.

## Begränsningar och gränser

RustDesk ersätter TeamViewer i många fall, men har begränsningar som du bör känna till före användning:

- Den offentliga förmedlingsservern är begränsad, saknar tillgänglighetsgaranti och är inte avsedd för kontinuerlig kommersiell drift. Den som vill arbeta tillförlitligt hostar själv.
- En egen server innebär driftarbete: containrar, öppna portar, certifikat och uppdateringar är ditt ansvar.
- En adressbok som delas på serversidan, central användarhantering och webbgränssnittet för administration ingår i Pro-versionen, som blir avgiftsbelagd från ett visst antal enheter. Klienten själv och grunddriften förblir gratis.
- Utan fast lösenord är obevakad åtkomst inte möjlig, vilket är korrekt för assisterad support men förhindrar spontan åtkomst till en obemannad enhet.
- Funktionsomfånget och stabiliteten på enskilda plattformar, särskilt på mobila enheter, når inte de kommersiella produkterna i varje detalj. Kontrollera de funktioner som är viktiga för dig före bytet.
- Vissa säkerhetsprogram flaggar fjärrsupportprogram som potentiellt oönskade. Lägg vid behov till ett undantag och dokumentera varför programvaran är installerad.

För privat bruk och support av enskilda enheter räcker gratisversionen med den offentliga servern eller en direktanslutning. Så snart du hanterar många enheter, arbetar kommersiellt eller behöver full kontroll över data behöver du en egen server, med motsvarande driftarbete som motprestation för oberoendet.

## Ändringshistorik

- **25 oktober 2026:** Installationsexemplet uppdaterat till RustDesk 1.5.0 från den 30 september 2026.

## Källor

1.  [RustDesk på GitHub](https://github.com/rustdesk/rustdesk): Källkod, releaser med installationsprogrammen och licensen AGPL-3.0.

2.  [RustDesk-dokumentation](https://rustdesk.com/docs/): Installation, egen server, portar och konfiguration av klienterna.

3.  [rustdesk-server på GitHub](https://github.com/rustdesk/rustdesk-server): Serverkomponenterna `hbbs` och `hbbr` inklusive portöversikten för egen drift.
