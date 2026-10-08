---
title: "Kiteworks: Vendor Recommends Shutdown on September 26 – What Is Known So Far"
navTitle: "Kiteworks Shutdown"
description: "Kiteworks asked its customers to shut down all systems on Saturday, September 26, 2026, from 4:00 a.m. to 10:00 a.m. Final report: During the shutdown, the vendor found and closed a critical flaw without a CVE; on September 30, 125 advisories followed, including CVE-2026-54154 (CVSS 10.0) in the Email Protection Gateway. Totemomail is not affected."
date: "2026-09-25"
kategorie: "Kiteworks / Totemomail"
timeToRead: "12 min read"
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
translationSourceHash: 15d32c4610fafdeff76397f77ace28ebf6f4aab220c10dc03f5dd4713fab9e48
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:50:33.488Z
translationReview: required
url: https://rafaelpfister.ch/en/blog/kiteworks-vendor-recommends-shutdown-on-september-26-what-is-known-so-far
---

# Kiteworks: Vendor Recommends Shutdown on September 26 – What Is Known So Far

Kiteworks emailed its customers on September 25, 2026, asking them to shut down all Kiteworks systems on Saturday, September 26, from 4:00 a.m. to 10:00 a.m. (Central European Time). According to the letter from CISO Frank Balonis, the vendor had received information from law enforcement authorities indicating that an attack on Kiteworks systems could be imminent that weekend. Customer support justified the shutdown as protection against potential zero-day attacks. heise online confirmed the authenticity of the message with support by phone.

<div class="update-hinweis">
<p class="update-hinweis__titel">Final report as of October 7, 2026</p>
<p>From the vendor’s perspective, the incident is closed. The shutdown recommendation has no longer applied since September 27, and no attack on Kiteworks or customer systems has become known to date. The key findings:</p>
<ul>
<li><strong>Critical flaw found during the shutdown:</strong> According to the September 28 press release, while analyzing the matter with federal authorities, Kiteworks came across a previously unknown critical vulnerability in a feature enabled by fewer than 1% of customers. The vendor developed and deployed a fix during the shutdown window and additionally activated a protection layer in all environments. The affected feature has not been disclosed; there is still no CVE number for it.</li>
<li><strong>125 advisories on September 30:</strong> Two days later, Kiteworks published 125 security advisories on GitHub for Kiteworks Core (66), Email Protection Gateway (28), Secure Data Forms (28), and MFT Server (3); 12 of them critical and 49 high. All are fixed in versions through 9.5.1, and most were reported through the bug bounty program on YesWeHack. Based on the information available today, they are unrelated to the flaw found during the shutdown window.</li>
<li><strong>CVE-2026-54154 (CVSS 10.0):</strong> The most severe vulnerability affects Email Protection Gateway versions before 9.4.1. An unauthenticated attacker can execute code with root privileges through publicly accessible endpoints. Ten of the 12 critical advisories affect the Email Protection Gateway.</li>
<li><strong>No known exploitation:</strong> There are no reports of attacks exploiting any of the vulnerabilities; as of October 7, the CISA KEV catalog contains no Kiteworks entry from 2026. According to BleepingComputer, Shadowserver counts just under 400 Internet-accessible Kiteworks instances.</li>
<li><strong>Totemomail:</strong> It does not appear in any of the advisories and, according to the vendor, was not affected by the shutdown.</li>
</ul>
<p><strong>Action required:</strong> Anyone operating Kiteworks themselves should update all components to version 9.5.1, with the Email Protection Gateway taking priority. Details are in the <a href="#abschlussbericht">Final report</a> section.</p>
</div>

<div class="update-hinweis">
<p class="update-hinweis__titel">Update from September 28, 2026: Kiteworks lifts shutdown recommendation</p>
<p>Kiteworks added a note to its press release: Since September 27, the shutdown recommendation no longer applies to any customers.</p>
<blockquote lang="en">
<p>As of September 27th, the shutdown recommendation is now lifted for all customers. If you have not already restarted, you may bring your Kiteworks system back online. Customers with self-hosted Advanced Forms should contact Customer Support for assistance. All systems Kiteworks hosts on customers’ behalf have been brought back up and are operating normally.</p>
</blockquote>
<p>Anyone who has not yet restarted their systems can now do so. Anyone self-hosting Advanced Forms should contact Kiteworks support before restarting. Instances hosted by Kiteworks are running again. There is still no CVE number, no new version beyond 9.5.1, no indicators of compromise, and no statement on whether an attack was attempted or what prompted the warning. The security updates page and GitHub advisories remain unchanged.</p>
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

All times are Central European Summer Time (CEST). Where no time is given, no reliable time information is available.

<ol class="timeline">
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Customer advisory</p>
<p>CISO Frank Balonis informs customers by email about indications from law enforcement authorities of a possible attack that weekend and recommends a six-hour shutdown. According to the advisory, all known vulnerabilities are fixed in version 9.5.1.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Initial media reports</p>
<p>heise online reports that Kiteworks support confirmed the message’s authenticity and justified the shutdown as protection against potential zero-day attacks. TechCrunch, BleepingComputer, Computer Weekly, and others follow shortly afterward.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25, 5:41 p.m.</p>
<p class="timeline__titel">BKA declines to comment</p>
<p>heise adds: The BKA declines to comment for investigative reasons. The BSI does not respond, and the FBI declines to comment to TechCrunch.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Fri, September 25</p>
<p class="timeline__titel">Statement and press release</p>
<p>Kiteworks describes the shutdown as a precautionary measure with no known compromise. The press release cites “federal intelligence authorities” as the source and lists the unaffected subsidiaries, including totemo. The vendor shuts down Kiteworks-hosted instances itself.</p>
</li>
<li class="timeline__item timeline__item--fenster">
<p class="timeline__zeit">Sat, September 26, 4:00 a.m. to 10:00 a.m.</p>
<p class="timeline__titel">Shutdown window</p>
<p>The window occurs simultaneously worldwide: 2:00 a.m. to 8:00 a.m. UTC, noon to 6:00 p.m. in Sydney, and Friday 10:00 p.m. to Saturday 4:00 a.m. in New York.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sat, September 26, 10:00 a.m.</p>
<p class="timeline__titel">End of the window</p>
<p>The window stated in the customer email ends. The formal lifting of the recommendation follows on September 27.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Sun, September 27</p>
<p class="timeline__titel">Recommendation lifted</p>
<p>Kiteworks adds to the press release: The shutdown recommendation is lifted for all customers, and systems may run again. Hosted instances are back in service. Customers self-hosting Advanced Forms should contact support.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Mon, September 28</p>
<p class="timeline__titel">Critical flaw found and closed</p>
<p>In another press release, Kiteworks reports that a previously unknown critical vulnerability was found while working with federal authorities during the shutdown. It affects a feature enabled by fewer than 1% of customers. The fix and additional protection layer have been deployed, and there are no indications of compromise. The vendor does not identify the feature or provide a CVE number.</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Wed, September 30, from 6:38 p.m.</p>
<p class="timeline__titel">125 security advisories on GitHub</p>
<p>Kiteworks publishes 125 advisories for Core, Email Protection Gateway, Secure Data Forms, and MFT Server, all fixed through version 9.5.1. The most severe flaw is CVE-2026-54154 in Email Protection Gateway before 9.4.1 (CVSS 10.0, root-level code execution without authentication).</p>
</li>
<li class="timeline__item">
<p class="timeline__zeit">Thu, October 1</p>
<p class="timeline__titel">Media reports and MS-ISAC advisory</p>
<p>BleepingComputer, SecurityOnline, and others report on the advisories; MS-ISAC (Center for Internet Security) issues its own advisory on CVE-2026-54154. No exploitation is known.</p>
</li>
<li class="timeline__item timeline__item--offen">
<p class="timeline__zeit">As of Wed, October 7</p>
<p class="timeline__titel">Conclusion</p>
<p>No reports of a successful or attempted attack, and no Kiteworks entry in the CISA KEV catalog. The affected feature, a CVE number for the flaw found during the shutdown window, and the background to the authorities’ warning remain unknown.</p>
</li>
</ol>

## What is known

The recommendation applies worldwide; the email specifies the time window for all time zones from AEST to PDT. Kiteworks advises shutting down systems before the window begins, including systems that are not Internet-accessible.

Until September 28, almost everything else was unknown: There was no public security advisory, no CVE number, no patch, and no indication of which products or versions were affected. The press release names “federal intelligence authorities” as the source, presumably U.S. federal authorities; which ones remains unknown. In Kiteworks’ GitHub advisories, the last entry until then was dated May 27, 2026; the September 30 advisories are summarized in the [Final report](#abschlussbericht) section.

Kiteworks CISO Frank Balonis gave TechCrunch the same statement verbatim. The BKA declined to comment to heise for investigative reasons, while the BSI did not respond. The FBI declined to comment to TechCrunch, and a CISA spokesperson did not wish to comment publicly. According to TechCrunch, a healthcare customer immediately took its server offline, causing noticeable operational restrictions: doctors could only reach their patients with delays for a time. According to a security researcher cited by TechCrunch, at least 1,000 Kiteworks systems are accessible from the Internet; BornCity says more than 1,000 organizations received the warning.

The press release differs from the customer email on one point: It refers to a nine-hour shutdown window, while the customer advisory refers to six hours. According to the press release, the recommendation affects only self-hosted installations (on-premises, AWS, Azure). According to the vendor, subsidiaries Zivver, DRACOON, totemo, ownCloud, WAMNET, Maytech, Bonfy.ai, and 123FormBuilder are not affected.

## The customer advisory

In addition to the warning, the customer email from September 25 contains a schedule for each time zone and instructions for clusters. Converting the times to UTC gives the same window for all regions: 2:00 a.m. to 8:00 a.m. UTC.

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

2.  **Create a backup:** take a snapshot of each node or back up the Kiteworks database (System Setup > Cluster Configuration > System Configuration). Only one database backup is retained; each new one replaces the previous one.

3.  **Record roles:** Under System Setup > Locations, the Assigned Roles column shows which nodes have the Application role; the primary Application node is marked with an asterisk. Note the nodes and their IP addresses, as they are needed for restarting.

4.  **Shut down in this order:** first all nodes without the Application role, then the remaining Application nodes, and finally the primary Application node. This can be done through the Shut Down tab of the respective node or through the hypervisor console (such as VMware or AWS) if the Kiteworks interface is no longer accessible.

5.  **Restart in reverse order** through the hypervisor, because the admin console is not accessible until enough nodes are running (Appendix E of the Administrator Guide): first the primary Application node, then the remaining Application nodes one at a time, and only once the previous one is fully running, so that the database servers can form a quorum. Then restart the storage servers, followed by the remaining roles (Repositories Gateway, Search, SFTP, Antivirus), and lastly the web servers.

6.  **Disable maintenance mode** as soon as all nodes are green in the Cluster Health Dashboard on the Admin Console status page.

In response to an inquiry, Kiteworks support also confirmed that none of Kiteworks’ subsidiaries are affected.

## Possible causes: Theories

This section was written before September 28; the assessment based on the current state is in the [Final report](#abschlussbericht) section. The following explanations are hypotheses that can be derived from the known facts; some are also discussed in comments on the heise report. None has been confirmed.

Three facts narrow the possibilities. First, the warning names a fixed time window rather than an indefinite shutdown until a patch is available. Second, even systems not accessible from the Internet are to be taken offline. Third, the window is at the same time worldwide (2:00 a.m. to 8:00 a.m. UTC), rather than during local nighttime. A conventional vulnerability exploitable over the Internet would not explain the first two points: Disconnecting the system from the Internet would help, and should remain in place until the patch is available.

### 1. Authorities know of a planned time

Law enforcement authorities occasionally learn of the timing of a planned campaign in advance, for example from monitored communications by a criminal group or seized infrastructure. Mass exploitation of file transfer products typically takes place in a brief, coordinated window, often on weekends or holidays, when fewer staff are on duty. Kiteworks is the successor to Accellion, whose File Transfer Appliance was attacked in exactly this manner in 2020 and 2021, at the time attributed to the Clop group: data was exfiltrated through several vulnerabilities, and the affected organizations were then extorted.

The narrowly defined weekend window supports this theory. Against it is the objection raised by several commenters on heise: The warning went to all customers, so attackers are likely to know about it and can simply postpone the attack. A delay, however, would give the vendor time to develop a patch.

### 2. The vendor does not yet know the vulnerability itself

It is also possible that Kiteworks has no technical details beyond the authorities’ information, meaning it knows neither the affected component nor a patch or configuration change it can recommend. In that case, shutdown is the only measure that works without knowledge of the vulnerability, and the fixed end time is a compromise customers are more likely to accept. The heise comments speculate that the vendor could leave individual systems online as bait during the window to observe the attack. There is no evidence for this.

Supporting this theory is that neither an advisory nor a mitigation was named. Against it is that Kiteworks says it works with Mandiant and that authorities generally provide at least indicators when issuing a warning.

### 3. A previously planted backdoor with a time trigger

The recommendation to also shut down internal systems fits a scenario in which the attack does not come from outside but has already been prepared on the appliances: for example, a backdoor from an earlier compromise that activates at a fixed time or contacts a command-and-control server. A powered-off system cannot execute anything at that time.

Supporting this theory is that Internet accessibility is irrelevant in this scenario. Against it is that a vendor would be more likely to recommend checking for compromise and reinstalling than restarting after six hours.

### 4. Compromise on the vendor side

Another route that reaches internal systems is connections established by the appliance to the vendor, for example for updates, license verification, or remote maintenance. If such a channel is compromised, a firewall does not protect against incoming traffic. In this scenario, the shutdown would give the vendor a window to clean up its own infrastructure, replace keys or certificates, and only then allow connections again.

The globally consistent time supports this theory, as it fits a coordinated action on the vendor side. Against it is that the vendor would be more likely to recommend blocking outbound connections than shutting down systems completely.

### 5. Accompanying measure for a law enforcement operation

Finally, it is conceivable that authorities take action against the attackers’ infrastructure during the same period and want to prevent them from striking quickly in response. That would explain the short window and the role of law enforcement. The BKA’s refusal to comment for investigative reasons points to ongoing investigations, but does not prove this scenario.

### Criticism of the communication

Skepticism predominates in the heise comments, and the objections are factually understandable: Without information about the vulnerability, it is impossible to assess whether disconnecting from the Internet with a firewall would have been sufficient. A time window without an announced patch leaves unclear what applies after 10:00 a.m. And a warning sent only by email to customers does not reach all operators, such as partners, service providers, or those affected by personnel changes. Regardless of which theory is correct: Anyone operating Kiteworks should review logs after restarting and monitor the vendor’s channels until an advisory is available.

## Final report

As of October 7, 2026, the incident is closed from the vendor’s perspective. The events after the shutdown window can be separated into two threads: the vulnerability found during the shutdown and the bulk publication of advisories two days later.

### The flaw found during the shutdown window

On September 28, Kiteworks published a second press release. According to it, the vendor worked with federal authorities throughout the weekend; in the process, a previously unknown critical vulnerability was discovered that is limited to a feature enabled by fewer than 1% of customers. Kiteworks developed and deployed a fix during the shutdown window and additionally activated a protection layer in all environments. Continuous monitoring showed no suspicious activity, and there is no indication of compromise of Kiteworks or customer systems. All other Kiteworks products were not affected.

The affected feature, a CVE number, the versions containing the fix, and whether self-hosted installations received the fix automatically have not been disclosed. The lifting of the recommendation on September 27 contained only one exception: Customers self-hosting Advanced Forms were to contact support before restarting. Kiteworks has not confirmed whether this was the affected feature.

Regarding the theories above: The press release describes a vulnerability that was only found during the window. This fits Theory 2 (the vendor did not know the vulnerability beforehand) in combination with Theory 1 (the authorities knew of a planned time). There is no confirmation for Theories 3 through 5. What the authorities specifically knew and whether an attack was attempted remain unknown.

### 125 security advisories from September 30

On September 30, starting at 6:38 p.m., Kiteworks published 125 security advisories on GitHub all at once. They are distributed as follows:

| Product | Advisories | Critical |
|---|---:|---:|
| Kiteworks Core | 66 | 2 |
| Email Protection Gateway (EPG) | 28 | 10 |
| Secure Data Forms (SDF) | 28 | 0 |
| MFT Server | 3 | 0 |
| **Total** | **125** | **12** |

By severity, there are 12 critical, 49 high, 52 medium, and 12 low ratings. All vulnerabilities are fixed in versions through 9.5.1; the oldest entries affect version 9.2.1. This is therefore a retrospective disclosure of fixes already delivered, not a new version. The advisories predominantly name participants in the bug bounty program on YesWeHack as reporters. Kiteworks does not establish a connection to the flaw found during the shutdown window; one week after the window, the status of the advisories still matches the September 25 statement that all known vulnerabilities are fixed in 9.5.1.

The critical advisories:

| CVE | Product | CVSS 3.1 | Fixed in | Impact |
|---|---|---|---|---|
| CVE-2026-54154 | EPG | 10.0 | 9.4.1 | Root-level code execution without authentication |
| CVE-2026-85065 | EPG | 9.8 | 9.5.0 | Account takeover |
| CVE-2026-85066 | EPG | 9.8 | 9.5.0 | Account takeover |
| CVE-2026-102115 | Core | 9.8 | 9.5.0 | Account takeover through password reset |
| CVE-2026-102149 | EPG | 9.4 | 9.5.1 | Account takeover |
| CVE-2026-102147 | Core | 9.3 | 9.5.1 | Account takeover |
| CVE-2026-102106 | EPG | 9.1 | 9.5.0 | Security feature bypass |
| CVE-2026-102095, CVE-2026-102102 through 102105 | EPG | 9.1 | 9.5.0 | Access to internal network resources (SSRF) |

CVE-2026-54154 is the most severe vulnerability: According to the advisory, a combination of input validation flaws in publicly accessible Email Protection Gateway endpoints allows an unauthenticated attacker to execute code and, through additional local weaknesses, gain root privileges on the appliance. MS-ISAC issued its own advisory on October 1. For mail administrators, the gateway is the relevant part of the publication: It typically sits directly in the mail flow and is accessible from the Internet.

### Exploitation and prevalence

There are no reports of exploitation or public exploits for any of the vulnerabilities. As of October 7, the CISA KEV catalog contains only the four Accellion FTA entries from 2021. According to BleepingComputer, Shadowserver counts just under 400 Internet-accessible Kiteworks instances; how many are already running 9.5.1 is unknown. Totemomail does not appear in any of the advisories.

## After the window: What operators can do now

The shutdown recommendation has been lifted, and the known vulnerabilities are fixed in version 9.5.1. The following steps are appropriate for self-hosted installations:

1.  **Check the version:** Are all nodes and all components (Core, Email Protection Gateway, Secure Data Forms, MFT Server) running version 9.5.1? An Email Protection Gateway before 9.4.1 is affected by CVE-2026-54154 and should be updated first.

2.  **Check cluster status:** All nodes should be green in the Cluster Health Dashboard, and maintenance mode should be disabled.

3.  **Review logs:** Review logins, administrator actions, and unusual file downloads around the shutdown window, especially on systems that were not shut down or were shut down late.

4.  **Restrict accessibility:** Where possible, block Internet access to the administrative interface and expose only required services.

5.  **Advanced Forms:** Anyone self-hosting the module who has not yet contacted support should clarify with Kiteworks whether the fix from the shutdown window has reached their installation.

6.  **Compare advisories:** GitHub advisories can be filtered by product (prefix `[Core]`, `[EPG]`, `[SDF]`, `[MFT]`). For every component in use, check whether the installed version is below the respective listed fixed version.

7.  **Monitor channels:** Monitor Kiteworks GitHub advisories, Newsroom, and customer emails in case the vendor does publish an advisory with a CVE number for the flaw from the shutdown window. The [CVE tracker](/cve) on this page also lists new CVEs for Kiteworks Email Protection Gateway and Totemomail; email alerts can be subscribed to there.

<div class="notfall-hinweis">
<p class="notfall-hinweis__titel">Support with the update</p>
<p>If you need help updating a Kiteworks or Totemomail gateway, such as redirecting mail flow during the maintenance window or reviewing logs, please use the <a href="https://adeptio.ch/">contact form on adeptio.ch</a>.</p>
</div>

## Sources

1.  [heise online: Imminent zero-day attack: KiteWorks urges customers to shut down servers](https://www.heise.de/news/Bevorstehender-Zero-Day-Angriff-KiteWorks-draengt-Kunden-zur-Serverabschaltung-11466114.html): Initial report with excerpts from the customer email and the time window; update from September 25, 5:41 p.m., with the BKA response.

2.  [heise online (EN): Imminent Zero-Day Attack: KiteWorks Urges Customers to Shut Down Servers](https://www.heise.de/en/news/Imminent-Zero-Day-Attack-KiteWorks-Urges-Customers-to-Shut-Down-Servers-11466375.html): English version with the CISO’s original wording.

3.  [Kiteworks: Security Updates](https://www.kiteworks.com/company/security-updates/): Older vendor update page, as of October 7, 2026, with no entry on the warning or the September 30, 2026 advisories.

4.  [Kiteworks: Security Advisories on GitHub](https://github.com/kiteworks/security-advisories/security): Vendor advisory list; through September 28, 2026, the last entry was from May 27, 2026; on September 30, 2026, 125 new advisories for Core, EPG, SDF, and MFT. Figures in this article counted through the GitHub API.

5.  [Kiteworks: Newsroom](https://www.kiteworks.com/newsroom/): Official announcements, including the press release on the shutdown since September 25, 2026.

6.  [heise forum: Comments on the report](https://www.heise.de/forum/heise-online/Kommentare/Server-am-Samstagmorgen-herunterfahren-Kiteworks-warnt-Admins-vor-Zero-Day/forum-591001/comment/): Reader discussion featuring objections to the fixed time window and shutdown of internal systems, as well as the bait theory.

7.  [CISA: Exploitation of Accellion File Transfer Appliance (AA21-055A)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-055a): Advisory on exploitation of Accellion FTA in 2020/2021 and subsequent extortion; Accellion is Kiteworks’ former name.

8.  [Kiteworks: Case Study Mandiant](https://www.kiteworks.com/case-study-mandiant/): Vendor statement regarding collaboration with Mandiant.

9.  [TechCrunch: Kiteworks urges customers to shut down their servers amid 'imminent' threat of cyberattack](https://techcrunch.com/2026/09/25/kiteworks-urges-customers-to-shut-down-their-servers-amid-imminent-threat-of-cyberattack/): CISO statement, warning dispatch time, reactions from the FBI and CISA (update), impact on a customer, and number of Internet-accessible systems.

10.  [Kiteworks: Precautionary Shutdown Advisory (press release)](https://www.kiteworks.com/company/press-releases/kiteworks-precautionary-shutdown-advisory/): Official announcement from September 25, 2026, with information on hosted instances, version 9.5.1, and unaffected subsidiaries; supplemented by the September 27, 2026 notice that the shutdown recommendation was lifted.

11.  [BleepingComputer: Kiteworks urges 6-hour server shutdown over potential zero-day attacks](https://www.bleepingcomputer.com/news/security/kiteworks-urges-6-hour-server-shutdown-over-potential-zero-day-attacks/): Time windows by region and context on earlier attacks on file transfer products.

12.  [Computer Weekly: Expecting cyber attack, Kiteworks tells users to turn off servers](https://www.computerweekly.com/news/366651301/Expecting-cyber-attack-Kiteworks-tells-users-to-turn-off-servers): watchTowr’s assessment of the unusual shutdown recommendation.

13.  [BornCity: Kiteworks: More than 1,000 organizations are told to shut down servers](https://borncity.com/news/kiteworks-mehr-als-1-000-organisationen-sollen-server-abschalten/): Number of notified organizations and industries in German-speaking regions.

14.  [The Record: Kiteworks urges customers to stop using platform after warning from federal intelligence agencies](https://therecord.media/kiteworks-urges-customers-to-stop-using-systems-incident): Context on the Clop attacks on Accellion in 2020/2021 and a quote from watchTowr.

15.  [Cyber Daily: Kiteworks warns customers to enact a „precautionary shutdown“ in wake of attack intelligence](https://www.cyberdaily.au/security/14241-kiteworks-warns-customers-to-enact-a-precautionary-shutdown-in-wake-of-attack-intelligence): September 28, 2026 report on lifting the recommendation and operation of hosted instances.

16.  [Kiteworks: Kiteworks Restores Systems After Credible Threat (press release)](https://www.kiteworks.com/company/press-releases/kiteworks-restores-systems-credible-threat/): September 28, 2026 announcement about the critical flaw found during the shutdown, the fix, and the additional protection layer.

17.  [The Hacker News: Kiteworks Fixes Critical Flaw Found During Nine-Hour Precautionary Shutdown](https://thehackernews.com/2026/09/kiteworks-fixes-critical-flaw-found.html): Summary of the second press release with quotes from the CISO.

18.  [GitHub Advisory GHSA-5xhq-9wq3-rvj6: CVE-2026-54154](https://github.com/kiteworks/security-advisories/security/advisories/GHSA-5xhq-9wq3-rvj6): Vendor information on code execution in Email Protection Gateway before 9.4.1, CVSS 10.0, reported through YesWeHack.

19.  [BleepingComputer: Kiteworks patches max severity code injection vulnerability](https://www.bleepingcomputer.com/news/security/kiteworks-patches-max-severity-email-protection-gateway-code-injection-vulnerability/): October 1, 2026 report on CVE-2026-54154 and the number of instances counted by Shadowserver.

20.  [MS-ISAC Advisory 2026-107: A Vulnerability in Kiteworks EPG Could Allow for Arbitrary Code Execution](https://www.cisecurity.org/advisory/a-vulnerability-in-kiteworks-epg-email-security-gateway-could-allow-for-arbitrary-code-execution_2026-107): Center for Internet Security advisory from October 1, 2026, with recommendations.

21.  [SecurityOnline: Kiteworks Patches 78 Vulnerabilities, Including Critical Account Takeover Flaw](https://securityonline.info/kiteworks-vulnerabilities/): Context on account takeover vulnerabilities in Core, including CVE-2026-102115; the count differs from the advisory list.

22.  [CISA: Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): As of October 7, 2026, only the four Accellion FTA entries from 2021, with no Kiteworks entry from 2026.
