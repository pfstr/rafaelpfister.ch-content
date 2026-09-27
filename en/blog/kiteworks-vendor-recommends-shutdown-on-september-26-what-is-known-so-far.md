---
title: "Kiteworks: Vendor Recommends Shutdown on September 26 – What Is Known So Far"
navTitle: "Kiteworks Shutdown"
description: "Kiteworks has emailed its customers asking them to shut down all systems on Saturday, September 26, 2026, from 4:00 a.m. to 10:00 a.m. The reason is a warning from law enforcement agencies about a possible attack. Totemomail is not affected."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "9 min read"
themen:
  - totemomail
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
url: https://rafaelpfister.ch/en/blog/kiteworks-vendor-recommends-shutdown-on-september-26-what-is-known-so-far
translationSourceHash: c7274a068cc60b422ffcdf30dbaef2d90eac1fe72cf3f768b454a71c676aa046
translationModel: gpt-5.6-terra
translatedAt: 2026-09-26T08:42:54.360Z
translationReview: required
---

# Kiteworks: Vendor Recommends Shutdown on September 26 – What Is Known So Far

On September 25, 2026, Kiteworks emailed its customers asking them to shut down all Kiteworks systems on Saturday, September 26, from 4:00 a.m. to 10:00 a.m. (Central European Time). According to the letter from CISO Frank Balonis, the vendor has received information from law enforcement agencies suggesting that an attack on Kiteworks systems may be imminent this weekend. Customer support cites protection against potential zero-day attacks as the reason for the shutdown. heise online confirmed the authenticity of the message by phone with support.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Emergency support for switching mail flow</p>
<p>If you need help rerouting mail flow before the shutdown and switching it back afterward, please use the <a href="https://adeptio.ch/">contact form on adeptio.ch</a>. I can also respond on short notice.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update from September 25, 2026: Statement from Kiteworks</p>
<blockquote lang="en">
<p>Kiteworks received credible threat intelligence from law enforcement indicating that a threat actor may attempt to target some Kiteworks systems for customers. Out of an abundance of caution, we notified customers directly and recommended a precautionary shutdown window while we and our law enforcement partners work through the matter. We are not aware of any compromise of Kiteworks systems, and this advisory is preventative rather than a response to a confirmed breach. All known vulnerabilities are addressed in our current release, 9.5.1, and we continue to recommend customers run the latest version.</p>
<p>totemomail is not affected by this.</p>
</blockquote>
<p><strong>Totemomail is not affected.</strong> It remains unclear whether Kiteworks EPG (Email Protection Gateway) is affected.</p>
</div>

## Timeline

All times are in Central European Summer Time (CEST). Where no time is given, no reliable time information is available.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Customer advisory</p>
<p>CISO Frank Balonis informs customers by email of indications from law enforcement agencies of a possible attack this weekend and recommends a six-hour shutdown. According to the advisory, all known vulnerabilities are fixed in version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Initial media reports</p>
<p>heise online reports that Kiteworks support confirmed the authenticity of the message and cited protection against potential zero-day attacks as the reason for the shutdown. TechCrunch, BleepingComputer, Computer Weekly, and others follow shortly afterward.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25, 5:41 p.m.</p>
<p class="timeline__titel">BKA does not comment</p>
<p>heise adds: The BKA declined to comment for investigative reasons. The BSI did not respond, and the FBI declined to comment to TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Statement and press release</p>
<p>Kiteworks describes the shutdown as a precautionary measure, with no known compromise. The press release names “federal intelligence authorities” as the source and lists the unaffected subsidiaries, including totemo. The vendor is shutting down Kiteworks-hosted instances itself.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sat, September 26, 4:00 a.m. to 10:00 a.m.</p>
<p class="timeline__titel">Shutdown window</p>
<p>The window is simultaneous worldwide: 2:00 a.m. to 8:00 a.m. UTC, 12:00 p.m. to 6:00 p.m. in Sydney, and Friday 10:00 p.m. to Saturday 4:00 a.m. in New York.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">As of Sat, September 26</p>
<p class="timeline__titel">Still unresolved</p>
<p>No public advisory, no CVE number, no details about the vulnerability, and no reports of a successful attack.</p>
</li>
</ol>

## What is known

The recommendation applies worldwide; the email gives the time window for all time zones from AEST to PDT. Kiteworks advises shutting down systems before the window begins, including systems that are not accessible from the internet.

Almost everything else remains unclear: There is no public security advisory, no CVE number, no patch, and no information on which products or versions are affected. The press release names “federal intelligence authorities” as the source, presumably U.S. federal agencies; it is not known which ones. As of September 26, there is no entry under Security Updates or in Kiteworks’ GitHub advisories. The statement quoted above and the press release of September 25 are public.

Kiteworks CISO Frank Balonis gave TechCrunch the same statement verbatim. The BKA declined to comment to heise for investigative reasons, while the BSI did not respond. The FBI declined to comment to TechCrunch, and there was no response from CISA. According to TechCrunch, a customer in the healthcare sector immediately took its server offline, causing noticeable operational restrictions.

The press release differs from the customer email on one point: It refers to a nine-hour shutdown window, while the customer advisory refers to six hours. According to the press release, the recommendation applies only to self-operated installations (on-premises, AWS, Azure). According to the vendor, its subsidiaries Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai, and 123FormBuilder are not affected.

## The customer advisory

In addition to the warning, the September 25 customer email contains a schedule for each time zone and instructions for clusters. Converting the times to UTC results in the same window for all regions: 2:00 a.m. to 8:00 a.m. UTC.

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

1.  **Enable maintenance mode** under System Setup > Maintenance Mode so that users can no longer access the system.

2.  **Create a backup:** Take a snapshot of each node or back up the Kiteworks database (System Setup > Cluster Configuration > System Configuration). Only one database backup is retained; each new one replaces the previous one.

3.  **Record roles:** Under System Setup > Locations, the Assigned Roles column shows which nodes have the Application role; the primary Application node is marked with an asterisk. Note the nodes and their IP addresses, as they are needed for the restart.

4.  **Shut down in this order:** First all nodes without the Application role, then the remaining Application nodes, and finally the primary Application node. This can be done through the Shut Down tab of the respective node or through the hypervisor console (such as VMware or AWS) if the Kiteworks interface is no longer accessible.

5.  **Restart in reverse order** through the hypervisor, since the admin console is only accessible once enough nodes are running (Appendix E of the Administrator Guide): first the primary Application node, then the remaining Application nodes one at a time and only once the previous one is fully running, so that the database servers can form a quorum. Then the storage servers, followed by the remaining roles (Repositories Gateway, Search, SFTP, Antivirus), and finally the web servers.

6.  **Disable maintenance mode** once all nodes are green in the Cluster Health Dashboard on the admin console status page.

When asked, Kiteworks support also confirmed that none of Kiteworks’ subsidiaries are affected.

## Possible causes: Theories

As long as Kiteworks does not publish details, the cause remains unclear. The following explanations are hypotheses derived from the known facts; some are also discussed in comments on the heise article. None has been confirmed.

Three facts narrow down the possibilities. First, the warning names a fixed time window rather than an indefinite shutdown until a patch is available. Second, even systems that are not accessible from the internet are supposed to be taken offline. Third, the window falls at the same time worldwide (2:00 a.m. to 8:00 a.m. UTC) rather than during local nighttime. A conventional vulnerability exploitable over the internet would not explain the first two points: Disconnecting the system from the internet would protect against it, and should do so until the patch is available.

### 1. Authorities know of a planned time

Law enforcement agencies occasionally learn the timing of a planned campaign in advance, for example from monitored communications of an attacker group or seized infrastructure. Mass exploitation of file-transfer products typically takes place within a brief, coordinated window, often on weekends or holidays when fewer staff are on duty. Kiteworks is the successor to Accellion, whose File Transfer Appliance was attacked in exactly this way in 2020 and 2021: Data was exfiltrated through several vulnerabilities, after which the affected organizations were extorted.

The narrowly defined weekend window supports this theory. Countering it is an objection raised by several commenters on heise: The warning was sent to all customers, so the attackers are likely to know about it and can simply postpone the attack. However, postponement would give the vendor time to prepare a patch.

### 2. The vendor does not yet know the vulnerability itself

It is also possible that Kiteworks has no technical details beyond the authorities’ tip, meaning it knows neither the affected component nor a patch or configuration change it can recommend. In that case, shutting down is the only measure that works without knowledge of the vulnerability, and the fixed end time is a compromise that customers are more likely to accept. In the heise comments, there is speculation that the vendor might leave individual systems online as decoys during the window to observe the attack. There is no evidence for this.

This is supported by the fact that neither an advisory nor a mitigation is named. Countering it is that Kiteworks says it is working with Mandiant, and when warned by authorities, indicators are usually available at least.

### 3. A previously planted backdoor with a time trigger

The recommendation to shut down internal systems as well fits a scenario in which the attack does not come from outside but has already been prepared on the appliances: for example, a backdoor from an earlier compromise that activates at a fixed time or contacts a command-and-control server. A powered-off system cannot execute anything at that time.

This is supported by the fact that internet accessibility plays no role in this scenario. Countering it is that, in this case, a vendor would more likely recommend checking for compromise and reinstalling rather than restarting after six hours.

### 4. Compromise on the vendor side

Another path that reaches internal systems is connections established by the appliance to the vendor, for example for updates, license verification, or remote maintenance. If such a channel is compromised, a firewall does not protect against incoming traffic. In this scenario, the shutdown would give the vendor a window to clean up its own infrastructure, replace keys or certificates, and allow connections again only afterward.

The globally uniform timing supports this theory, as it fits a coordinated action on the vendor side. Countering it is that the vendor would more likely recommend blocking outbound connections rather than shutting down the systems entirely.

### 5. An accompanying measure for a law enforcement operation

Finally, it is conceivable that authorities are taking action against the attackers’ infrastructure during the same period and want to prevent them from striking quickly in response. That would explain the short window and the role of law enforcement. The BKA’s refusal to comment for investigative reasons points to an ongoing investigation, but does not prove this scenario.

### Criticism of the communication

Skepticism prevails in the heise comments, and the objections are factually understandable: Without details about the vulnerability, it is impossible to assess whether disconnecting from the internet using a firewall would have been sufficient. A time window without an announced patch leaves open what applies after 10:00 a.m. And a warning sent only by email to customers does not reach all operators, such as those at partners, service providers, or after personnel changes. Regardless of which theory is correct, anyone operating Kiteworks should review the logs after bringing systems back online and monitor the vendor’s channels until an advisory is available.

## Sources

1.  [heise online: Imminent zero-day attack: KiteWorks urges customers to shut down servers](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Initial report with excerpts from the customer email and the time window; update from September 25, 5:41 p.m., with the BKA’s response.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): English version with the CISO’s original wording.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): Official vendor channel, with no entry on the warning as of September 26, 2026.

4.  [Kiteworks: Security Advisories on GitHub](https://github.com/kiteworks/security-advisories/security): Vendor’s advisory list, with the latest entry dated May 27, 2026.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): Official announcements, including the shutdown press release since September 25, 2026.

6.  [heise Forum: Comments on the report](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Reader discussion with objections concerning the fixed time window and shutdown of internal systems, as well as the decoy theory.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory on exploitation of the Accellion FTA in 2020/2021 followed by extortion; Accellion is the former name of Kiteworks.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Vendor statement on its cooperation with Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO statement, warning delivery time, responses from the FBI and CISA, and impact on one customer.

10.  [Kiteworks: Precautionary Shutdown Advisory (press release)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Official statement from September 25, 2026, with information on hosted instances, version 9.5.1, and unaffected subsidiaries.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Time windows by region and context on earlier attacks on file-transfer products.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowr’s assessment of the unusual shutdown recommendation.
