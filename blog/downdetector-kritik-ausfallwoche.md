---
title: "Downdetector in der Kritik: Was die US-Ausfallwoche über Störungsmeldungen zeigt"
navTitle: "Downdetector-Kritik"
description: "Zwischen dem 13. und 17. November 2026 meldete Downdetector in den USA so viele Störungen wie seit einem Jahr nicht mehr. Die Ausfälle bei FAA, AWS, Chase, United und Apple waren echt, hatten aber getrennte Ursachen und waren meist nach Stunden behoben. Die Zahlen von Downdetector haben das Bild einer zusammenhängenden Krise verstärkt. Was die Plattform misst, wo die Kritik ansetzt und wie Administratoren Störungen besser einordnen."
date: "2026-11-18"
kategorie: "Betrieb"
timeToRead: "7 Min. Lesezeit"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-kritik-ausfallwoche"
translationId: article-0761d100437e6b25
url: https://rafaelpfister.ch/blog/downdetector-kritik-ausfallwoche
---

# Downdetector in der Kritik: Was die US-Ausfallwoche über Störungsmeldungen zeigt

Vom 13. bis 17. November 2026 fielen in den USA mehrere grosse Systeme aus: die Flugsicherung FAA mit einem landesweiten Startverbot, Teile von AWS in der Region us-east-1, die Bank JPMorgan Chase, die Abfertigung von United Airlines und am Dienstag die iCloud-Anmeldung und weitere Dienste von Apple. Ich habe die Serie in einem [Liveticker](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11) verfolgt. Nach Abschluss der Woche steht fest: Die Ausfälle hatten getrennte Ursachen, Hinweise auf einen Cyberangriff gibt es keine, und Europa war nur am Rand betroffen. In der öffentlichen Wahrnehmung entstand trotzdem das Bild eines Internets, das an allen Ecken gleichzeitig ausfällt. Ein grosser Teil dieses Bildes stammt aus den Diagrammen von Downdetector, die in fast jedem Medienbericht zitiert wurden. Daran setzt seit Montag breite Kritik an.

## Was Downdetector misst

Downdetector gehört zu Ookla und damit seit dem 17. Juni 2026 zu Accenture. Die Plattform sammelt Problemmeldungen von Benutzern, die auf der Seite eines Dienstes ein Problem melden, und ergänzt sie laut eigener Beschreibung um Signale aus sozialen Medien und anderen Online-Quellen. Für jeden Dienst berechnet Downdetector eine Grundlinie: die durchschnittliche Zahl der Meldungen zur selben Tageszeit über das vergangene Jahr. Liegt die aktuelle Zahl deutlich darüber, gilt das als Störung. Wie die Quellen gewichtet werden und ab welchem Faktor ein Ausschlag als Störung zählt, veröffentlicht Downdetector nicht.

Daraus folgen drei Eigenschaften, die in der Berichterstattung oft untergehen:

1.  **Meldungen sind keine betroffenen Benutzer.** Eine Zahl wie «20'000 Meldungen» sagt nichts darüber, wie viele Kunden tatsächlich keinen Zugriff hatten. Sie zählt Menschen, die ein Problem bemerkt und Downdetector aufgerufen haben.

2.  **Benutzer melden den Dienst, den sie sehen.** Fällt ein Zahlungsnetz, ein Cloud-Anbieter oder ein Identitätsdienst aus, erscheinen die Meldungen bei den Banken, Apps und Shops, die davon abhängen. Die Ursache ist im Diagramm nicht erkennbar.

3.  **Die Grundlinie bildet keine Ausnahmewochen ab.** Berichten Medien laufend über Ausfälle, prüfen mehr Menschen bei jeder Verzögerung, ob ein Dienst gestört ist. Jede neue Schlagzeile erhöht die Zahl der Meldungen auch bei Diensten, die normal laufen.

## Die Zahlen der Woche

Laut Axios gingen bei Downdetector in den sieben Tagen bis zum 16. November fast 6 Millionen Problemmeldungen aus den USA ein, der höchste Wochenwert seit einem Jahr. Dieselbe Auswertung zeigt aber auch: Über 30 Tage betrachtet gab es drei grosse Ausfälle, üblich sind fünf. Die Rekordzahl an Meldungen steht damit einer eher unterdurchschnittlichen Zahl tatsächlicher Grossstörungen gegenüber.

Mehrere Fälle der Woche zeigen, wie weit Meldungen und Lage auseinanderliegen können:

- **Banken am 16. November:** Medien schrieben unter Berufung auf Downdetector, dass Benutzer bei «praktisch jeder grossen US-Bank» Probleme meldeten, darunter Chase, Wells Fargo und Capital One. Chase selbst sprach von Transaktionen, die länger dauerten; Kunden hätten weiterhin Zugriff auf Konten und Guthaben. Laut New York Times lag die Ursache in einem Softwarefehler in eigener Hardware von JPMorgan Chase, der andere Banken über gemeinsame Transaktionen mitbetroffen haben könnte. Eigene Störungen haben die anderen Banken nicht bestätigt.

- **Gmail am 16. und 17. November:** Downdetector zeigte über 2'800 Meldungen, vor allem zu nicht zugestellten E-Mails. Google erklärte, es gebe keine Hinweise auf eine breite Störung, das Workspace Status Dashboard blieb ohne Eintrag.

- **Charles Schwab am 17. November:** Laut Wall Street Journal stiegen die Meldungen gegen 9:50 Uhr Ortszeit an und fielen eine halbe Stunde später wieder ab. Eine Störungsmeldung von Schwab dazu gibt es nicht.

- **Exchange Online am 12. November:** Für dieselbe Störung nannte Reuters über 97'000 Meldungen, ein Lokalsender fast 119'000, beide zu unterschiedlichen Zeitpunkten. Die Meldungen stiegen laut Downdetector erst rund 68 Minuten nach dem Beginn, den Microsoft im Admin Center angibt.

Die Ausfälle bei FAA, AWS, Chase, United und Apple waren real und durch die Betreiber bestätigt. Übertrieben war die Verknüpfung: Aus fünf unabhängigen Störungen und einem allgemeinen Anstieg der Meldungen entstand die Erzählung einer koordinierten Attacke. Die Sprecherin des Weissen Hauses sagte am 17. November, es gebe keine Krise, über die sich Amerikaner Sorgen machen müssten. Liz Fong-Jones (Honeycomb) sagte Axios, die Vorfälle hätten sehr wenig gemeinsam.

## Wo die Kritik ansetzt

Die Kritik kommt aus mehreren Richtungen. Betreiber wie Google widersprechen öffentlich, wenn Downdetector eine Störung anzeigt, die sie in ihren eigenen Daten nicht sehen. Fachmedien weisen darauf hin, dass Downdetector Meldungen statt betroffener Benutzer zählt, Störungen verspätet anzeigt und Ausfälle in Komponenten, die Endkunden nicht sehen, gar nicht erfasst. In Diskussionen unter Entwicklern, etwa auf Hacker News, kursiert zudem die Vermutung, dass schon der Aufruf einer Seite wie «Ist Chase gestört?» als implizite Meldung zählt und sich so Ausschläge von einem Dienst auf ähnliche Dienste übertragen. Downdetector hat diese Vermutung nicht bestätigt; belegt ist nur, dass die Plattform neben aktiven Meldungen weitere Online-Quellen auswertet und deren Gewichtung nicht offenlegt.

Die Verteidiger der Plattform halten dagegen, dass Downdetector Störungen oft früher zeigt als die Statusseiten der Anbieter. Beim Apple-Ausfall am 17. November meldete die Apple-Statusseite noch normale Verfügbarkeit, als die Meldungen bereits stark anstiegen. Als Frühwarnung ist der Dienst brauchbar. Als Mass für das Ausmass einer Störung ist er es nicht, und genau so wurde er in dieser Woche verwendet: Schlagzeilen wie «Tausende melden Probleme» standen ohne Einordnung neben bestätigten Ausfällen.

## Störungen besser einordnen

Für Administratoren, die bei einer gemeldeten Störung entscheiden müssen, ob sie Benutzer informieren oder Massnahmen einleiten, ergibt sich eine einfache Reihenfolge:

1.  **Statusseite und Admin-Portal des Anbieters prüfen.** Für Microsoft 365 ist das Service Health im Admin Center, nicht die öffentliche Statusseite. AWS, Google Cloud, Cloudflare und Apple führen eigene Statusseiten mit Zeitstempeln.

2.  **Eigene Messung heranziehen.** Ein einfacher synthetischer Test (Anmeldung, Mailversand, Abruf einer Seite) aus dem eigenen Netz zeigt, ob die eigene Organisation betroffen ist. Mein [M365-Verbindungsmonitor](https://rafaelpfister.ch/liveticker) prüft dafür alle fünf Minuten die Erreichbarkeit der Microsoft-365-Endpunkte aus Europa.

3.  **Downdetector nur als Hinweis lesen.** Ein Ausschlag ist ein Anlass, nachzusehen. Die absolute Zahl und Ausschläge bei mehreren ähnlichen Diensten gleichzeitig sagen wenig über Ursache und Ausmass.

4.  **Regionale Herkunft beachten.** Die meisten Zahlen in Medienberichten beziehen sich auf die US-Seite von Downdetector. Für Benutzer in der Schweiz sind sie nur dann relevant, wenn der Anbieter die Störung auch für europäische Regionen bestätigt.

## Der Liveticker

Den Liveticker zur Ausfallserie habe ich mit diesem Artikel abgeschlossen. Er bleibt als Chronik der Woche online, mit einem Schlussfazit zu den einzelnen Ursachen und dem Hinweis, dass Europa ausser bei Transatlantikflügen und der weltweiten iCloud-Störung nicht betroffen war.

## Quellen

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): fast 6 Millionen Downdetector-Meldungen in sieben Tagen, drei statt üblich fünf grosse Ausfälle in 30 Tagen, Einschätzung von Liz Fong-Jones.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): Funktionsweise, Grundlinie pro Tageszeit, Übernahme durch Accenture, abweichende Zahlen zur Exchange-Online-Störung vom 12. November.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): Downdetector-Meldungen bei mehreren US-Banken, Stellungnahme von Chase.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): zitiert New York Times und Wall Street Journal zu Apple, Charles Schwab und der Ursache bei JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): Downdetector-Meldungen zu Gmail und Dementi von Google.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): Übersicht der Erklärungen der Betreiber.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): Verlauf des Apple-Ausfalls, Statusseite hinter den Meldungen.

8.  [Hacker News: Diskussion zur Methodik von Downdetector](https://news.ycombinator.com/item?id=50521819): Vermutungen zu impliziten Meldungen und Übertragung zwischen ähnlichen Diensten, nicht von Downdetector bestätigt.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): Einschätzung eines Sicherheitsexperten, keine Hinweise auf einen Zusammenhang.
