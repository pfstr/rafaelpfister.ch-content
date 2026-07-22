---
title: "Serverloser Newsletter: Cloudflare Workers, D1 und ein Deploy-Button"
navTitle: "Newsletter auf Workers"
description: "Ein vollständiges Newsletter-System (Anmeldeformular, One-Click-Abmeldung, Versandseite) läuft serverlos auf dem eigenen Cloudflare-Konto und bleibt für kleine und mittlere Listen im Free Tier. Der Deploy-Button provisioniert Datenbank, Schema und CI in einem Schritt; eine Kommandozeile braucht es nicht."
date: "2026-07-22"
kategorie: "Cloudflare Workers"
timeToRead: "7 min to read"
themen:
  - "cloudflare-workers"
slug: "serverloser-newsletter-cloudflare-workers-d1"
url: "https://rafaelpfister.ch/blog/serverloser-newsletter-cloudflare-workers-d1"
aiPrompt: |
  Du bist mein Cloudflare-Assistent. Hilf mir, das Newsletter-Template Schritt für Schritt auf meinem Cloudflare-Konto in Betrieb zu nehmen:
  1. Deploy über den Deploy-to-Cloudflare-Button (ADMIN_TOKEN, FROM_NAME, FROM_EMAIL setzen).
  2. Eigenen E-Mail-Versand in src/email.ts anbinden (sendEmail() implementieren, Provider-Secret setzen, isEmailConfigured() anpassen) und die Absenderdomain per SPF/DKIM verifizieren.
  3. Optional Double-Opt-in, Turnstile-Bot-Schutz und den RSS-Auto-Versand aktivieren.
  Frage nach den Werten, die nur ich kenne (Domain, E-Mail-Provider, Feed-URL).
---

<!-- NACH MERGE von cloudflare/templates PR #1064: Galerie-Link https://github.com/cloudflare/templates/tree/main/newsletter-template und Live-Demo https://newsletter-template.templates.workers.dev wieder einfügen (Intro, Abschnitt "Ausprobieren", Quellen; DE + EN). Beide URLs waren bei Veröffentlichung noch 404, deshalb entfernt. -->

# Serverloser Newsletter: Cloudflare Workers, D1 und ein Deploy-Button

Newsletter-Dienste rechnen nach Abonnentenzahl ab, und die Empfängerliste liegt beim Anbieter. Die klassische Alternative (ein Newsletter-System auf einem eigenen Server) scheitert in der Praxis selten an der Software, sondern am Betrieb: Ein Server will provisioniert, gepatcht, überwacht und bezahlt werden, für ein Werkzeug, das vielleicht einmal pro Woche eine E-Mail verschickt.

Serverless verändert diese Rechnung grundlegend. Ein Newsletter-Backend besteht aus wenigen HTTP-Endpunkten, einer kleinen Datenbank und einem zeitgesteuerten Job: exakt das Profil, für das Cloudflare Workers und D1 gebaut sind. Ich habe das als offenes Template umgesetzt: ein vollständiges Newsletter-System, das per **Deploy-to-Cloudflare-Button** in einem Schritt auf dem eigenen Cloudflare-Konto landet. Ohne Kommandozeile, ohne Serverbetrieb, und für kleine und mittlere Listen innerhalb des Free Tiers. Der Quellcode ist MIT-lizenziert auf [GitHub](https://github.com/pfstr/newsletter-template) verfügbar; der Aufnahme-PR in die offizielle Cloudflare-Template-Galerie läuft.

<p class="badge-row"><a href="https://deploy.workers.cloudflare.com/?url=https://github.com/pfstr/newsletter-template"><img src="/images/deploy-to-cloudflare.svg" alt="Deploy to Cloudflare" width="184" height="39"></a></p>

![Das gehostete Anmeldeformular des Templates](../images/newsletter-template-signup.png)

## Was das Template kann

- **Anmeldung**: eine gehostete Anmeldeseite, ein einbettbares Formular für die eigene Website und ein JSON-Endpunkt
- **One-Click-Abmeldung**: konform zu RFC 8058, mit individuellem Token pro Abonnent
- **Versand**: eine geschützte Seite, auf der Sie Betreff und HTML einfügen, einen Test an sich selbst schicken und dann an die Liste senden
- **Eigene Daten**: Abonnenten liegen in einer D1-Datenbank auf Ihrem Konto und lassen sich jederzeit exportieren
- **Optional, standardmässig aus**: Double-Opt-in, Bot-Schutz per Turnstile und automatischer Versand neuer Blogartikel aus dem RSS-Feed

## Architektur: ein Worker, eine Datenbank

Das gesamte System ist ein einzelner Cloudflare Worker mit zwei Handlern: `fetch` für HTTP (geroutet mit Hono) und `scheduled` für den Cron-Trigger, plus einer D1-Datenbank. Es gibt keinen zweiten Dienst, keinen Queue-Broker, kein separates Admin-Backend.

| Route | Funktion |
| --- | --- |
| `GET /` | Gehostete Anmeldeseite |
| `GET /embed` | Transparentes Formular zum Einbetten per iframe |
| `POST /api/subscribe` | Anmeldung (CORS-offen für die eigene Website) |
| `GET /confirm` | Bestätigungslink bei Double-Opt-in |
| `GET/POST /unsubscribe` | Abmeldung, One-Click nach RFC 8058 |
| `GET /admin` | Versandseite (Formular) |
| `POST /api/send` | Kampagnenversand, geschützt per Admin-Token |

Das Datenmodell umfasst drei Tabellen: `subscribers` (E-Mail als Primärschlüssel, Name, Status, Abmelde- und Bestätigungs-Token sowie eine JSON-Spalte für selbst definierte Zusatzfelder), `campaigns` als Sendeprotokoll und `sent_posts` für die Deduplizierung des RSS-Versands.

## Deployment ohne Kommandozeile

Der interessanteste Teil ist nicht der Code, sondern der Weg zum laufenden System. Der Deploy-to-Cloudflare-Button liest die Wrangler-Konfiguration des Repositories und erledigt die komplette Einrichtung: Er klont das Repository in den eigenen GitHub-Account, provisioniert die D1-Datenbank, führt die Schema-Migrationen aus und richtet CI ein, sodass jeder Push automatisch deployt. Seit Juli 2025 fragt der Deploy-Flow zusätzlich Umgebungsvariablen und Secrets direkt im Formular ab: im Fall dieses Templates das Admin-Passwort (`ADMIN_TOKEN`), Absendername und -adresse sowie der Double-Opt-in-Schalter.

Das Ergebnis nach einem Klick und einem Formular: Die Anmeldeseite ist unter `https://<worker-name>.workers.dev` live und sammelt Abonnenten. Ein Terminal wird an keiner Stelle geöffnet.

## Abonnenten sammeln

Für die Einbindung auf der eigenen Website gibt es drei Wege, aufsteigend nach Integrationstiefe. Der einfachste: den Link zur gehosteten Anmeldeseite teilen. Der praktischste für Site-Builder (WordPress, Webflow, Squarespace, Framer): ein iframe-Einzeiler in einem beliebigen HTML-Embed-Block.

```html
<iframe
  src="https://<worker-name>.workers.dev/embed"
  style="width:100%;max-width:420px;height:90px;border:0"
></iframe>
```

Wer das Formular im eigenen Design haben will, postet direkt an den Endpunkt:

```html
<form
  onsubmit="event.preventDefault();
  fetch('https://<worker-name>.workers.dev/api/subscribe', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({ email: this.email.value })
  }).then(()=>this.reset());"
>
  <input name="email" type="email" placeholder="you@example.com" required />
  <button>Abonnieren</button>
</form>
```

Das Formular erfasst standardmässig E-Mail und optional den Namen. Weitere Felder (Firma, Land, …) definieren Sie in einer einzigen Datei (`src/fields.ts`); sie erscheinen automatisch auf beiden Formularen und landen als JSON in der Datenbank.

## Versand: eigener Provider statt eingebautem Vendor

Beim E-Mail-Versand trifft das Template eine bewusste Entscheidung: Es ist **provider-agnostisch**. Die Datei `src/email.ts` enthält einen einzelnen `sendEmail()`-Adapter mit einem kommentierten Beispiel für eine generische HTTP-API. Welchen Versanddienst Sie dort anbinden, bleibt Ihre Wahl. Kein Anbieter ist fest verdrahtet, keine Registrierung bei einem bestimmten Dienst wird vorausgesetzt. Das Sammeln von Abonnenten funktioniert bereits komplett ohne Versandkonfiguration; der Versand wird freigeschaltet, sobald der Adapter implementiert und das Provider-Secret gesetzt ist.

Der Versand selbst läuft über die `/admin`-Seite: Betreff und E-Mail-HTML einfügen, Test an die eigene Adresse schicken, dann an alle Abonnenten senden. In den E-Mails stehen die Merge-Tags `{{unsubscribe_url}}`, `{{email}}` und `{{name}}` zur Verfügung. Jede versendete Nachricht trägt die List-Unsubscribe-Header nach RFC 8058: Gmail und Outlook zeigen damit ihren nativen Abmelden-Button, und die Abmeldung funktioniert mit einem Klick ohne Login.

## Optionen: Double-Opt-in, Turnstile, RSS-Automatik

Drei Funktionen sind eingebaut, aber standardmässig deaktiviert, damit das System ohne Konfiguration lauffähig bleibt:

- **Double-Opt-in** (`DOUBLE_OPT_IN = "true"`): Neue Abonnenten werden als `pending` gespeichert und erst nach Klick auf einen Bestätigungslink aktiv. Für die Schweiz (DSG) und die EU ist dieses Verfahren die sauberere Wahl.
- **Bot-Schutz** mit Cloudflare Turnstile: Site- und Secret-Key als Variablen setzen, mehr nicht; das Widget erscheint automatisch auf beiden Formularen, und der Worker verifiziert jede Anmeldung serverseitig. Ohne gültiges Token wird die Anmeldung abgelehnt.
- **RSS-Auto-Versand**: Ein Cron-Trigger prüft alle 15 Minuten den eigenen Blog-Feed (RSS 2.0 oder Atom) und verschickt neue Artikel automatisch an die Liste. Zwei Sicherungen sind eingebaut: Beim allerersten Lauf wird der bestehende Feed nur als Baseline markiert (das Archiv wird also nicht als E-Mail-Flut verschickt), und jede Artikel-ID wird in `sent_posts` festgehalten, sodass kein Beitrag zweimal rausgeht.

## Grenzen

Das Template ist bewusst minimal gehalten. Der Kampagnenversand ist eine einfache Schleife und trägt damit einige hundert Empfänger pro Aussand (Workers-Subrequest-Limits); wer grössere Listen bedient, verlagert den Versand auf Cloudflare Queues. Die Zustellbarkeit hängt, wie bei jedem System, an der eigenen Absenderdomain: SPF und DKIM müssen beim gewählten Versanddienst verifiziert sein, sonst landet der Newsletter im Spam. Und der Single-Opt-in-Default ist der einfachste Start, aber nicht die konservativste Compliance-Variante; dafür gibt es den Schalter.

Zu den Kosten: Workers und D1 haben grosszügige Free-Tier-Kontingente (unter anderem 100'000 Requests pro Tag), die ein Anmeldeformular und wöchentliche Aussände an eine kleine bis mittlere Liste nicht ausschöpfen. Wird ein Limit erreicht, drosselt Cloudflare im Free-Plan, statt eine Rechnung zu stellen.

## Ausprobieren

Der Quellcode inklusive Deploy-Button liegt auf [GitHub](https://github.com/pfstr/newsletter-template); dort steht auch die vollständige Dokumentation der Konfigurationsvariablen.

<p class="badge-row"><a href="https://github.com/pfstr/newsletter-template"><img src="/images/github-newsletter-template.svg" alt="GitHub: pfstr/newsletter-template" width="323" height="28"></a></p>

## Quellen

1.  [pfstr/newsletter-template](https://github.com/pfstr/newsletter-template) — Quellcode des Templates (MIT) mit Deploy-Button und Dokumentation.

2.  [Deploy to Cloudflare buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/) — automatische Provisionierung von Ressourcen, Repo-Klon und CI beim Deploy.

3.  [Deploy buttons: environment variables and secrets](https://developers.cloudflare.com/changelog/post/2025-07-01-workers-deploy-button-supports-environment-variables-and-secrets/) — Secrets und Variablen werden seit Juli 2025 im Deploy-Formular abgefragt.

4.  [Cloudflare D1](https://developers.cloudflare.com/d1/) — serverloses SQLite, hier für Abonnenten, Sendeprotokoll und RSS-Deduplizierung.

5.  [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) — Bot-Schutz ohne Captcha-Rätsel, im Template optional zuschaltbar.

6.  [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058) — Signaling One-Click Functionality for List Email Headers; Grundlage des nativen Abmelden-Buttons in Gmail und Outlook.

7.  [Cloudflare Queues](https://developers.cloudflare.com/queues/) — der dokumentierte Weg für den Versand an grössere Listen.
