---
title: "Kiteworks: Vendor Recommends Shutdown on September 26—What Is Known So Far"
navTitle: "Kiteworks Shutdown"
description: "Kiteworks asked its customers by email to shut down all systems on Saturday, September 26, 2026, from 4:00 a.m. to 10:00 a.m. The reason is a warning from law enforcement agencies about a possible attack. Since September 27, the recommendation has been lifted; there is no CVE or new patch. TotemoMail is not affected."
date: "2026-09-25"
kategorie: "Kiteworks / TotemoMail"
timeToRead: "9 min read"
themen:
  - totemomail
  - sicherheitsluecken
produkte:
  - "totemomail"
protokolle:
  - "verschluesselung"
  - "haertung"
  - "smtp"
hauptthema: "totemomail"
slug: "kiteworks-vendor-recommends-shutdown-on-september-26-what-is-known-so-far"
featured: "2026-09-27"
translationId: "article-38fbaa0e9095957a"
aiPrompt: |
  Du bist mein Exchange- und Mailflow-Assistent. Kiteworks empfiehlt, alle Systeme am 26.09.2026 von 04:00 bis 10:00 Uhr herunterzufahren. Hilf mir zu ermitteln, welche Connectoren, Transportregeln und MX-Einträge in meiner Umgebung Mails über das Gateway leiten, wie ich den Mailflow für die Dauer der Abschaltung umleite oder kontrolliert anhalte und wie ich den ursprünglichen Zustand danach wiederherstelle. Frage zuerst nach meinem Aufbau (Exchange Online, Exchange Server oder Hybrid, Richtung des Mailflows, Position des Gateways).
translationOf: kiteworks-zero-day-abschaltung
translationSourceHash: 93bc9f973258d524a87baa5fe75957444b339bcac669281db049e3f1e5817813
translationModel: gpt-5.6-terra
translatedAt: 2026-09-28T09:56:12.554Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/kiteworks-vendor-recommends-shutdown-on-september-26-what-is-known-so-far
---

# Kiteworks: Vendor Recommends Shutdown on September 26—What Is Known So Far

On September 25, 2026, Kiteworks asked its customers by email to shut down all Kiteworks systems on Saturday, September 26, from 4:00 a.m. to 10:00 a.m. (Central European Time). According to the letter from CISO Frank Balonis, the vendor has received indications from law enforcement agencies that an attack on Kiteworks systems may be imminent that weekend. Customer support cited protection against potential zero-day attacks as the reason for the shutdown. heise online confirmed the authenticity of the message with support by phone.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Emergency assistance with switching mail flow</p>
<p>If you need help redirecting mail flow before the shutdown and switching it back afterward, please use the <a href="https://adeptio.ch/">contact form at adeptio.ch</a>. I can also respond at short notice.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update from September 28, 2026: Kiteworks lifts shutdown recommendation</p>
<p>Kiteworks added a notice to the press release: Since September 27, the shutdown recommendation no longer applies to any customers.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Anyone who has not yet restarted their systems can now do so. Anyone self-hosting Advanced Forms should contact Kiteworks Support before restarting. Instances hosted by Kiteworks are running again. There is still no CVE number, no new version beyond 9.5.1, no indicators of compromise, and no information on whether an attack was attempted or what prompted the warning. The Security Updates page and GitHub advisories remain unchanged.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update from September 25, 2026: Statement from Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>TotemoMail is not affected.</strong> It remains unclear whether Kiteworks EPG (Email Protection Gateway) is affected.</p>
</div>

## Timeline

All times are Central European Summer Time (CEST). Where no time is given, no reliable time information is available.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Advisory to customers</p>
<p>CISO Frank Balonis informs customers by email about indications from law enforcement agencies of a possible attack that weekend and recommends a six-hour shutdown. According to the advisory, all known vulnerabilities are fixed in version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Initial media reports</p>
<p>heise online reports that Kiteworks Support confirmed the authenticity of the message and cited protection against potential zero-day attacks as the reason for the shutdown. TechCrunch, BleepingComputer, Computer Weekly, and others follow shortly afterward.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25, 5:41 p.m.</p>
<p class="timeline__titel">BKA does not comment</p>
<p>heise adds: The BKA declines to comment for investigative reasons. The BSI does not respond, while the FBI declines to comment to TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Statement and press release</p>
<p>Kiteworks describes the shutdown as a precautionary measure with no known compromise. The press release cites “federal intelligence authorities” as the source and lists unaffected subsidiaries, including totemo. The vendor shuts down instances hosted by Kiteworks itself.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sat, September 26, 4:00 a.m. to 10:00 a.m.</p>
<p class="timeline__titel">Shutdown window</p>
<p>The window takes place simultaneously worldwide: 2:00 a.m. to 8:00 a.m. UTC, 12:00 p.m. to 6:00 p.m. in Sydney, and Friday 10:00 p.m. to Saturday 4:00 a.m. in New York.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sat, September 26, 10:00 a.m.</p>
<p class="timeline__titel">End of the window</p>
<p>The window stated in the customer email ends. The formal lifting of the recommendation follows on September 27.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sun, September 27</p>
<p class="timeline__titel">Recommendation lifted</p>
<p>Kiteworks adds to the press release: The shutdown recommendation is lifted for all customers, and systems may resume operation. Hosted instances are back in service. Customers self-hosting Advanced Forms should contact support.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">As of Mon, September 28</p>
<p class="timeline__titel">Still unresolved</p>
<p>No public advisory, no CVE number, no new version, no indicators, no information about the vulnerability, and no reports of a successful or attempted attack.</p>
</li>
</ol>

## What is known

The recommendation applies worldwide; the email gives the time window for all time zones from AEST to PDT. Kiteworks advises shutting down systems before the start of the window, including systems that are not reachable from the internet.

Almost everything else remains unclear: there is no public security advisory, no CVE number, no patch, and no indication of which products or versions are affected. The press release cites “federal intelligence authorities” as the source, presumably U.S. federal agencies; which ones are unknown. As of September 28, there is no entry under Security Updates or in Kiteworks' GitHub advisories; the most recent GitHub entry is dated May 27, 2026. Publicly available are the statement quoted above and the press release from September 25.

Kiteworks CISO Frank Balonis gave TechCrunch the same statement verbatim. The BKA declined to comment to heise for investigative reasons, and the BSI did not respond. The FBI declined to comment to TechCrunch, while a CISA spokesperson declined to comment publicly. According to TechCrunch, a customer in the healthcare sector immediately took its server offline, causing noticeable operational disruptions: doctors could reach their patients only with delays at times. According to a security researcher quoted by TechCrunch, at least 1,000 Kiteworks systems are reachable from the internet; BornCity reports that more than 1,000 organizations received the warning.

The press release differs from the customer email on one point: it refers to a nine-hour shutdown window, while the customer advisory refers to six hours. According to the press release, the recommendation applies only to self-operated installations (on-premises, AWS, Azure). According to the vendor, its subsidiaries Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai, and 123FormBuilder are not affected.

## The advisory to customers

In addition to the warning, the customer email from September 25 includes a schedule for each time zone and instructions for clusters. Converting the times to UTC yields the same 2:00 a.m. to 8:00 a.m. UTC window for all regions.

| Time zone | City | Start | End |
|---|---|---|---|
| AEST (UTC+10) | Sydney | Sat, 12:00 p.m. | Sat, 6:00 p.m. |
| SGT (UTC+8) | Singapore | Sat, 10:00 a.m. | Sat, 4:00 p.m. |
| IDT (UTC+3) | Tel Aviv | Sat, 5:00 a.m. | Sat, 11:00 a.m. |
| CEST (UTC+2) | Amsterdam, Zurich | Sat, 4:00 a.m. | Sat, 10:00 a.m. |
| BST (UTC+1) | London | Sat, 3:00 a.m. | Sat, 9:00 a.m. |
| EDT (UTC−4) | New York | Fri, 10:00 p.m. | Sat, 4:00 a.m. |
| CDT (UTC−5) | Chicago | Fri, 9:00 p.m. | Sat, 3:00 a.m. |
| MDT (UTC−6) | Denver | Fri, 8:00 p.m. | Sat, 2:00 a.m. |
| PDT (UTC−7) | San Francisco | Fri, 7:00 p.m. | Sat, 1:00 a.m. |

For clusters with multiple servers, Kiteworks specifies a fixed sequence:

1.  **Enable maintenance mode** under System Setup > Maintenance Mode so users can no longer access the system.

2.  **Create a backup:** Take a snapshot of each node or back up the Kiteworks database (System Setup > Cluster Configuration > System Configuration). Only one database backup is retained; each new backup replaces the previous one.

3.  **Record roles:** Under System Setup > Locations, the Assigned Roles column shows which nodes have the Application role; the primary Application node is marked with an asterisk. Record the nodes and their IP addresses, as they are needed for the restart.

4.  **Shut down in this order:** First all nodes without the Application role, then the remaining Application nodes, and finally the primary Application node. This can be done through the Shut Down tab of the respective node or through the hypervisor console (such as VMware or AWS) if the Kiteworks interface is no longer accessible.

5.  **Restart in reverse order** through the hypervisor, because the admin console becomes accessible only once enough nodes are running (Appendix E of the Administrator Guide): first the primary Application node, then the remaining Application nodes one at a time and only after the preceding node is fully running, so that the database servers can form a quorum. Then restart the storage servers, followed by the remaining roles (Repositories Gateway, Search, SFTP, Antivirus), and finally the web servers.

6.  **Disable maintenance mode** once all nodes are green in the Cluster Health Dashboard on the admin console status page.

When asked, Kiteworks Support also confirmed that none of Kiteworks' subsidiaries are affected.

## Possible causes: theories

Until Kiteworks publishes details, the cause remains unclear. The following explanations are hypotheses that can be derived from the known facts; some are also discussed in comments on the heise report. None has been confirmed.

Three facts narrow the possibilities. First, the warning names a fixed time window rather than an indefinite shutdown until a patch is available. Second, even systems not reachable from the internet are to be taken offline. Third, the window falls at the same time worldwide (2:00 a.m. to 8:00 a.m. UTC), rather than during local nighttime in each region. A classic vulnerability exploitable over the internet would not explain the first two points: disconnecting the system from the internet would protect against it, and should continue until a patch is available.

### 1. Authorities know of a planned date

Law enforcement agencies occasionally learn of the timing of a planned campaign in advance, for example from monitored communications of an attacker group or seized infrastructure. Mass exploitation of file-sharing products typically occurs in a short, coordinated window, often on weekends or holidays when fewer staff are on duty. Kiteworks is the successor to Accellion, whose File Transfer Appliance was attacked in exactly this way in 2020 and 2021, at the time attributed to the Clop group: data was exfiltrated through several vulnerabilities, after which the affected organizations were extorted.

The narrowly defined weekend window supports this theory. Against it is the objection raised by several commenters at heise: the warning went to all customers, so attackers are likely to know about it and can simply postpone the attack. However, a delay would give the vendor time to develop a patch.

### 2. The vendor does not yet know the vulnerability itself

It is also possible that, apart from the tip from authorities, Kiteworks has no technical details, meaning it knows neither the affected component nor a patch or configuration change it can recommend. In that case, shutting down is the only measure that works without knowledge of the vulnerability, and the fixed end time is a compromise customers are more likely to accept. heise commenters speculate that the vendor could leave individual systems online as bait during the window to observe the attack. There is no evidence of this.

Supporting this theory is that neither an advisory nor a mitigation has been named. Against it is that Kiteworks says it works with Mandiant and, when authorities issue a warning, indicators are generally at least available.

### 3. A previously planted backdoor with a time trigger

The recommendation to shut down internal systems as well fits a scenario in which the attack does not come from outside but has already been prepared on the appliances: for example, a backdoor from an earlier compromise that activates at a fixed time or contacts a command-and-control server. A system that is switched off cannot execute anything at that time.

Supporting this theory is that internet reachability plays no role in this scenario. Against it is that, in such a case, a vendor would be more likely to recommend checking for compromise and reinstalling than restarting after six hours.

### 4. Compromise on the vendor side

Another path to internal systems is through connections initiated by the appliance to the vendor, such as for updates, license verification, or remote maintenance. If such a channel is compromised, a firewall does not protect against inbound traffic. In this scenario, the shutdown would give the vendor a window to clean up its own infrastructure, replace keys or certificates, and permit connections again only afterward.

The uniform worldwide time supports this theory, as it fits a coordinated action on the vendor side. Against it is that the vendor would be more likely to recommend blocking outbound connections rather than shutting systems down completely.

### 5. A supporting measure for a law enforcement operation

Finally, it is conceivable that authorities are taking action against attacker infrastructure during the same period and want to prevent attackers from striking quickly in response. This would explain the short window and the role of law enforcement. The BKA's refusal to comment for investigative reasons points to ongoing investigations, but does not prove this scenario.

### Criticism of the communication

Skepticism predominates in the heise comments, and the objections are factually understandable: without information about the vulnerability, it is impossible to assess whether disconnecting from the internet by firewall would have been sufficient. A time window without an announced patch leaves unclear what applies after 10:00 a.m. And a warning sent only by email to customers does not reach all operators, for example those at partners, service providers, or after staffing changes. Regardless of which theory is correct, anyone operating Kiteworks should review logs after bringing systems back up and monitor the vendor's channels until an advisory is available.

## After the window: What operators can do now

Kiteworks lifted the shutdown recommendation on September 27 but has not published technical details. It is therefore impossible to assess whether or how the threat was eliminated. The following steps make sense when bringing systems back up and afterward:

1.  **Check the version:** Is version 9.5.1 running on all nodes? According to the vendor, all known vulnerabilities are fixed in it.

2.  **Check cluster status:** All nodes should be green in the Cluster Health Dashboard, and maintenance mode should be disabled.

3.  **Review logs:** Examine logins, administrator actions, and unusual file downloads around the shutdown window, especially on systems that were not shut down or were shut down late.

4.  **Restrict accessibility:** Where possible, block internet access to the administration interface and expose only required services.

5.  **Advanced Forms:** Anyone self-hosting the module should clarify the restart with Kiteworks Support beforehand.

6.  **Monitor channels:** Monitor Kiteworks' Security Updates, GitHub advisories, Newsroom, and customer emails until an advisory with technical details is available.

## Sources

1.  [heise online: Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Initial report with excerpts from the customer email and the time window; update from September 25, 5:41 p.m., with the BKA's response.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): English version with the CISO's original wording.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): Official vendor channel, with no entry on the warning as of September 28, 2026.

4.  [Kiteworks: Security Advisories on GitHub](https://github.com/kiteworks/security-advisories/security): Vendor advisory list, with the latest entry dated May 27, 2026, as of September 28, 2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): Official announcements, including the shutdown press release since September 25, 2026.

6.  [heise forum: Comments on the report](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Reader discussion with objections concerning the fixed time window and shutdown of internal systems, as well as the bait theory.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory on exploitation of the Accellion FTA in 2020/2021 followed by extortion; Accellion is Kiteworks' former name.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Vendor statement on its collaboration with Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO statement, time the warning was sent, reactions from the FBI and CISA (addendum), impact on one customer, and number of systems reachable from the internet.

10.  [Kiteworks: Precautionary Shutdown Advisory (press release)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Official announcement from September 25, 2026, with details on hosted instances, version 9.5.1, and unaffected subsidiaries; supplemented with the September 27, 2026 notice that the shutdown recommendation was lifted.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Time window by region and context on earlier attacks on file-sharing products.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowr's assessment of the unusual shutdown recommendation.

13.  [BornCity: Kiteworks: More than 1,000 organizations are asked to shut down servers](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): Number of notified organizations and industries in German-speaking countries.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): Context on the Clop attacks on Accellion in 2020/2021 and a quote from watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a “precautionary shutdown” in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): September 28, 2026 report on the lifting of the recommendation and operation of hosted instances.
