---
title: "Downdetector Under Fire: What the U.S. Outage Week Reveals About Incident Reports"
navTitle: "Downdetector criticism"
description: "Between November 13 and 17, 2026, Downdetector received more outage reports in the United States than at any point in the past year. The outages at the FAA, AWS, Chase, United, and Apple were real, but had separate causes and were mostly resolved within hours. Downdetector's figures reinforced the impression of a connected crisis. What the platform measures, where the criticism lies, and how administrators can better assess incidents."
date: "2026-11-18"
kategorie: "Operations"
timeToRead: "7 min read"
themen: []
produkte: []
protokolle:
  - "troubleshooting"
slug: "downdetector-under-fire-what-the-u-s-outage-week-reveals-about-incident-reports"
translationId: article-0761d100437e6b25
translationOf: downdetector-kritik-ausfallwoche
url: https://rafaelpfister.ch/en/blog/downdetector-under-fire-what-the-u-s-outage-week-reveals-about-incident-reports
translationSourceHash: 3f7c9f1263f57a9ba194011c81d634786e31250e328f8cc02bb45a26564c46d8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T08:44:10.486Z
translationReview: automatic
---

# Downdetector Under Fire: What the U.S. Outage Week Reveals About Incident Reports

From November 13 to 17, 2026, several major systems went down in the United States: the FAA's air traffic control system, with a nationwide ground stop; parts of AWS in the us-east-1 region; JPMorgan Chase; United Airlines' ground operations; and, on Tuesday, iCloud sign-in and other Apple services. I followed the series in a [live ticker](https://rafaelpfister.ch/liveticker/usa-ausfallserie-2026-11). After the week ended, one thing was clear: the outages had separate causes, there is no indication of a cyberattack, and Europe was affected only marginally. Yet the public perception was that the internet was failing everywhere at once. A large part of that impression came from Downdetector charts, which were cited in almost every media report. That has drawn widespread criticism since Monday.

## What Downdetector measures

Downdetector belongs to Ookla and, since June 17, 2026, to Accenture. The platform collects problem reports from users who report an issue on a service's page and, according to its own description, supplements them with signals from social media and other online sources. For each service, Downdetector calculates a baseline: the average number of reports at the same time of day over the past year. If the current number is significantly higher, it is considered an outage. Downdetector does not publish how it weights its sources or the multiplier at which a spike counts as an outage.

This has three implications that are often overlooked in reporting:

1.  **Reports are not affected users.** A figure such as “20,000 reports” says nothing about how many customers actually lost access. It counts people who noticed a problem and visited Downdetector.

2.  **Users report the service they see.** If a payment network, cloud vendor, or identity service goes down, reports appear under the banks, apps, and stores that depend on it. The cause cannot be identified in the chart.

3.  **The baseline does not account for exceptional weeks.** If media outlets continuously report outages, more people check whether a service is disrupted whenever there is a delay. Every new headline increases the number of reports, including for services operating normally.

## The week's figures

According to Axios, Downdetector received nearly 6 million problem reports from the United States in the seven days through November 16, the highest weekly total in a year. However, the same analysis also shows that there were three major outages over 30 days, compared with the usual five. The record number of reports therefore coincided with a somewhat below-average number of actual major disruptions.

Several cases from the week show how far apart reports and the actual situation can be:

- **Banks on November 16:** Citing Downdetector, media outlets wrote that users were reporting problems at “virtually every major U.S. bank,” including Chase, Wells Fargo, and Capital One. Chase itself said transactions were taking longer; customers still had access to accounts and balances. According to the New York Times, the cause was a software error in JPMorgan Chase's own hardware, which may have affected other banks through shared transactions. The other banks did not confirm outages of their own.

- **Gmail on November 16 and 17:** Downdetector showed more than 2,800 reports, mainly concerning undelivered emails. Google said there was no indication of a widespread outage, and the Workspace Status Dashboard remained without an entry.

- **Charles Schwab on November 17:** According to the Wall Street Journal, reports rose around 9:50 a.m. local time and fell again half an hour later. There is no outage notice from Schwab.

- **Exchange Online on November 12:** For the same outage, Reuters cited more than 97,000 reports, while a local TV station cited nearly 119,000, both at different times. According to Downdetector, reports did not start rising until about 68 minutes after the start time Microsoft lists in the Admin Center.

The outages at the FAA, AWS, Chase, United, and Apple were real and confirmed by the operators. What was exaggerated was the connection drawn between them: five independent incidents and a general increase in reports became the narrative of a coordinated attack. The White House spokesperson said on November 17 that there was no crisis Americans needed to worry about. Liz Fong-Jones of Honeycomb told Axios the incidents had very little in common.

## Where the criticism begins

The criticism comes from several directions. Operators such as Google publicly object when Downdetector shows an outage that they do not see in their own data. Trade media point out that Downdetector counts reports rather than affected users, displays outages with a delay, and does not capture outages in components that end customers cannot see at all. Discussions among developers, such as on Hacker News, have also raised the theory that simply visiting a page such as “Is Chase down?” counts as an implicit report, allowing spikes to spread from one service to similar services. Downdetector has not confirmed this theory; the only established fact is that the platform evaluates other online sources in addition to active reports and does not disclose their weighting.

Defenders of the platform counter that Downdetector often shows outages earlier than vendors' status pages. During the Apple outage on November 17, Apple's status page still showed normal availability when reports were already rising sharply. The service is useful as an early warning system. It is not a measure of an outage's scope, yet that is exactly how it was used that week: headlines such as “Thousands report problems” appeared alongside confirmed outages without context.

## Better assessing incidents

For administrators who need to decide whether to inform users or initiate action when an outage is reported, the sequence is straightforward:

1.  **Check the vendor's status page and admin portal.** For Microsoft 365, that means Service Health in the Admin Center, not the public status page. AWS, Google Cloud, Cloudflare, and Apple maintain their own timestamped status pages.

2.  **Use your own measurements.** A simple synthetic test—sign-in, sending email, retrieving a page—from your own network shows whether your organization is affected. My [M365 connectivity monitor](https://rafaelpfister.ch/liveticker) checks the reachability of Microsoft 365 endpoints from Europe every five minutes.

3.  **Read Downdetector only as an indication.** A spike is a reason to investigate. The absolute number and spikes across several similar services at the same time say little about the cause or scope.

4.  **Consider regional origin.** Most figures in media reports refer to Downdetector's U.S. site. For users in Switzerland, they are relevant only if the vendor also confirms the outage for European regions.

## The live ticker

I concluded the live ticker on the outage series with this article. It remains online as a chronicle of the week, including a final assessment of the individual causes and the note that Europe was not affected except for transatlantic flights and the worldwide iCloud outage.

## Sources

1.  [Axios: Tech outages spark fears over AI and cybersecurity](https://www.axios.com/2026/11/17/tech-outages-ai-fears-internet-breaking): nearly 6 million Downdetector reports in seven days, three major outages rather than the usual five over 30 days, and Liz Fong-Jones's assessment.

2.  [PPC Land: Downdetector](https://ppc.land/downdetector/): how it works, baseline by time of day, acquisition by Accenture, and differing figures for the Exchange Online outage on November 12.

3.  [Futurism: Wait, Why Did Every Major Bank Just Go Down?](https://futurism.com/future-society/banks-go-down-outage): Downdetector reports at several U.S. banks and Chase's statement.

4.  [Hot Air: More Outages Today, Still No Evidence of Cyberattacks](https://hotair.com/john-s-2/2026/11/17/more-outages-today-still-no-evidence-of-cyberattacks-n3821101): cites the New York Times and Wall Street Journal on Apple, Charles Schwab, and the cause at JPMorgan Chase.

5.  [Pasquale Pillitteri: Gmail down, Google denies a broad outage after 2,800 reports](https://pasqualepillitteri.it/en/news/34509/gmail-down-google-denies-broad-outage-2800-reports): Downdetector reports concerning Gmail and Google's denial.

6.  [Forbes: Why Are So Many Outages Happening? Companies Deny Cyberattacks](https://www.forbes.com/sites/conormurray/2026/11/17/outages-plague-america-these-are-the-explanations/): overview of the operators' explanations.

7.  [Time: What to Know About the Apple Outage](https://time.com/article/2026/11/17/apple-outage-down-update-issue/): timeline of the Apple outage, with the status page lagging behind reports.

8.  [Hacker News: Discussion of Downdetector's methodology](https://news.ycombinator.com/item?id=50521819): theories about implicit reports and spillover between similar services, not confirmed by Downdetector.

9.  [WMTV: Apple, banks, FAA and Microsoft outages raise questions](https://www.wmtv15news.com/2026/11/18/apple-banks-faa-microsoft-outages-raise-questions-cybersecurity-expert-weighs/): assessment by a security expert, with no indication of a connection.
