---
title: "RustDesk: konfigurera det öppna TeamViewer-alternativet"
navTitle: "Konfigurera RustDesk"
description: "RustDesk är programvara för fjärrsupport med öppen källkod under AGPL, gratis och möjlig att självhosta. Så installerar du klienten i Windows (även obevakat via MSI), hur anslutningen upprättas via den offentliga förmedlingsservern, en egen server eller en direktanslutning, vilka funktioner som behövs i det dagliga supportarbetet och var gränserna för kostnadsfri användning går."
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
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
translationOf: rustdesk-teamviewer-alternative
url: https://rafaelpfister.ch/sv/blog/rustdesk-konfigurera-det-oppna-teamviewer-alternativet
translationSourceHash: f812fc4b04abe0aa92cca47b285a30a18f5cd1e99ab328593b224ee26051a7f3
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T08:49:58.536Z
translationReview: automatic
---

# RustDesk: konfigurera det öppna TeamViewer-alternativet

TeamViewer och AnyDesk täcker fjärrsupport på ett tillförlitligt sätt, men kräver en licens för kommersiell användning och priserna stiger med antalet enheter som hanteras. RustDesk är ett alternativ under licensen AGPL-3.0: öppen källkod, gratis och utan licenskrav. Klienten körs på Windows, macOS, Linux, Android och iOS samt i webbläsaren. Den är skriven i Rust och gränssnittet i Flutter.

Den avgörande skillnaden jämfört med de kommersiella produkterna ligger i förmedlingen: RustDesk skiljer klienten från serverinfrastrukturen. Du kan använda den kostnadsfria offentliga förmedlingsservern, driva en egen server eller upprätta en direktanslutning helt utan förmedlingsserver. Därmed kan RustDesk användas från en enskild arbetsplats till en självhostad supportplattform, utan att anslutningsdata behöver gå via en leverantör.

## De tre anslutningssätten

Innan du installerar bör du fastställa anslutningssättet, eftersom konfigurationen och öppna portar beror på det.

| Anslutningssätt | Så fungerar det | När det är lämpligt |
|---|---|---|
| Offentlig förmedlingsserver | Två klienter hittar varandra via ID:t (niiffrigt nummer) på RustDesk-servern, anslutningen går direkt eller via en relay | Snabb start, test, privat tillfällig support |
| Egen server (självhostad) | Du driver serverkomponenterna `hbbs` (förmedling) och `hbbr` (relay) själv, och alla klienter anger deras adress | Kommersiell användning, många enheter, full kontroll över data |
| Direktanslutning (Direct IP Access) | Klienten ansluter direkt till motpartens IP-adress utan förmedlingsserver | Båda enheterna kan nå varandra i samma nätverk eller via ett VPN |

Den offentliga servern är uttryckligen avsedd för tester och privat användning. För produktiv, kommersiell drift rekommenderar projektet en egen server, även eftersom den offentliga tjänsten är begränsad och inte ger någon tillgänglighetsgaranti.

## Installation i Windows

Installationsprogrammet laddar du ned från den officiella källan, projektets GitHub-releaser (`github.com/rustdesk/rustdesk`). För Windows finns en körbar fil och ett MSI-paket. För interaktiv installation räcker det att dubbelklicka. Om du vill distribuera RustDesk på flera datorer eller i bakgrunden använder du MSI med en tyst installation:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `/i <paket>` | Installerar det angivna MSI-paketet |
| `/qn` | Inget gränssnitt, inga dialogrutor (tyst) |
| `/norestart` | Förhindrar en automatisk omstart efter installationen |

</details>

Den tysta installationen konfigurerar tjänsten `RustDesk`, som körs vid systemstart och möjliggör obevakad åtkomst. Efter installationen kan du läsa ut enhetens ID via kommandoraden, utan att öppna gränssnittet:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Du kan också ange ett fast lösenord för obevakad åtkomst via kommandoraden. Ange ett separat, tillräckligt långt lösenord, inte användarens inloggningslösenord:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Funktion |
|---|---|
| `--get-id` | Visar enhetens niosiffriga RustDesk-ID |
| `--password <wert>` | Anger det fasta lösenordet för obevakad åtkomst |
| `--silent-install` | Installerar den körbara versionen (`.exe`) utan gränssnitt som en tjänst |

</details>

## Ange egen server

Om du driver en egen förmedlingsserver anger klienterna dess adress och den offentliga nyckeln. I gränssnittet finns detta i nätverksinställningarna som ID-server, relay-server och nyckel. För massdistribution kan konfigurationen även anges som fil eller via miljövariabler, så att varje klient startar förkonfigurerad.

En egen server behöver de två komponenterna `hbbs` och `hbbr`, som oftast körs som Docker-containrar. Båda kräver öppna portar så att klienter kan registrera sig och använda en relay.

| Port | Protokoll | Komponent och syfte |
|---|---|---|
| 21114 | TCP | Webbgränssnitt för Pro-versionen (endast där) |
| 21115 | TCP | `hbbs`, test av NAT-typ |
| 21116 | TCP och UDP | `hbbs`, registrering (UDP) och anslutningsupprättande (TCP) |
| 21117 | TCP | `hbbr`, relay-trafik |
| 21118, 21119 | TCP | Stöd för webbklienter |

Öppna endast de portar som ditt anslutningssätt faktiskt behöver och begränsa åtkomsten via brandväggen till de nätverk från vilka support ges.

## Direktanslutning utan förmedlingsserver

Om båda enheterna kan nå varandra i samma nätverk eller via ett VPN fungerar RustDesk helt utan förmedlingsserver. Aktivera då direktåtkomst på målenheten (i gränssnittet under säkerhet som "Aktivera direkt IP-åtkomst", internt växeln `direct-server`). Klienten lyssnar då på standardporten 21118 (TCP). I anslutningsfönstret anger du motpartens IP-adress i stället för ID:t.

Begränsa direktåtkomsten via brandväggen till nätverket som du ansluter från. Om åtkomsten sker via ett VPN ska porten endast öppnas för VPN-adressområdet, inte för hela internet.

## Funktioner i det dagliga supportarbetet

RustDesk täcker de funktioner som behövs för fjärrsupport i vardagen:

- Skärmöverföring och fjärrstyrning av tangentbord och mus, med val av skärm vid flera bildskärmar.
- Filöverföring i båda riktningarna via ett delat fönster.
- Textchatt under sessionen.
- Obevakad åtkomst via ett fast lösenord, för enheter utan närvarande användare.
- Sessionsinspelning som videofil, automatiskt vid behov.
- TCP-tunnel och vidarebefordran för att lokalt nå enskilda tjänster hos motparten.
- Adressbok och flera sparade enheter, lokalt i gratisversionen och delat på serversidan i Pro-versionen.

För support med användaren på plats är följande viktigt: Som standard frågar RustDesk på motpartssidan om anslutningen ska accepteras och visar under sessionen att åtkomst pågår. Personen vid enheten är alltså informerad. Först ett fast lösenord för obevakad åtkomst tar bort förfrågan. Använd endast obevakad åtkomst på enheter vars användare vet att programvaran är installerad och vad den används till.

## Begränsningar och gränser

RustDesk ersätter TeamViewer i många fall, men har begränsningar som du bör känna till före användning:

- Den offentliga förmedlingsservern är begränsad, saknar tillgänglighetsgaranti och är inte avsedd för kontinuerlig kommersiell drift. Den som vill arbeta tillförlitligt hostar själv.
- En egen server innebär driftarbete: containrar, öppna portar, certifikat och uppdateringar är ditt ansvar.
- En adressbok som delas på serversidan, central användarhantering och webbgränssnittet för administration ingår i Pro-versionen, som blir avgiftsbelagd från ett visst antal enheter. Själva klienten och grunddriften förblir gratis.
- Utan fast lösenord är obevakad åtkomst inte möjlig, vilket är korrekt för support med användaren på plats men förhindrar spontan åtkomst till en obemannad enhet.
- Funktionsomfånget och stabiliteten på enskilda plattformar, särskilt mobila enheter, når inte de kommersiella produkterna i varje detalj. Kontrollera de funktioner som är viktiga för dig innan du byter.
- Vissa säkerhetsprogram rapporterar fjärrsupportprogram som potentiellt oönskade. Lägg vid behov till ett undantag och dokumentera varför programvaran är installerad.

För privat användning och support av enskilda enheter räcker gratisversionen med den offentliga servern eller en direktanslutning. Så snart du hanterar många enheter, arbetar kommersiellt eller behöver full kontroll över data behöver du en egen server, med motsvarande driftarbete som motprestation för oberoendet.

## Källor

1.  [RustDesk på GitHub](https://github.com/rustdesk/rustdesk): Källkod, releaser med installationsprogrammen och licensen AGPL-3.0.

2.  [RustDesk-dokumentation](https://rustdesk.com/docs/): Installation, egen server, portar och klientkonfiguration.

3.  [rustdesk-server på GitHub](https://github.com/rustdesk/rustdesk-server): Serverkomponenterna `hbbs` och `hbbr` inklusive portöversikten för egen drift.
