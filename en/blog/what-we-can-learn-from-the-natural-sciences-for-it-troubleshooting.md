---
title: "What We Can Learn from the Natural Sciences for IT Troubleshooting"
navTitle: "Controlled Experiments"
description: "Falsifiability, control groups, confounding variables, and sampling bias: the method natural sciences have used for centuries solves exactly the problems that routinely cause IT troubleshooting to fail, illustrated with mail flow examples."
date: "2026-08-11"
kategorie: "SMTP / Mail Flow"
timeToRead: "15 min read"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "exchange"
protokolle:
  - "smtp"
  - "troubleshooting"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - einliefernde-ip-adressen-aggregieren
  - typische-ursachen-fuer-mail-loops-und-deren-behebung
slug: "what-we-can-learn-from-the-natural-sciences-for-it-troubleshooting"
translationId: "article-098ed40e6d027b8b"
draft: false
translationOf: mailflow-fehlersuche-kontrollierte-experimente
translationSourceHash: 1759c709160723c161883fede66d48cd968bb6a587cb49c73781b3a39498cd66
translationModel: gpt-5.6-terra
translatedAt: 2026-10-30T03:28:53.741Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/en/blog/what-we-can-learn-from-the-natural-sciences-for-it-troubleshooting
---

# What We Can Learn from the Natural Sciences for IT Troubleshooting

A message does not arrive. The log provides an error message that immediately suggests an explanation. You investigate that explanation, find evidence, and two hours later it turns out that the explanation was wrong and the evidence was coincidental.

This is not a beginner's mistake; it is the rule. And it is remarkable that our industry rarely has a method for this problem, even though one has existed for centuries and works exceptionally well. The natural sciences have exactly the same task: inferring causes from observations in systems that cannot be fully understood at a glance.

This article applies five principles of the scientific method to mail flow troubleshooting. The examples come from practice, but the approach is not specific to email.

## Why IT troubleshooting is systematically vulnerable

Mail flow is a chain of systems, each with its own view of the same message: the gateway, the filtering layer, the local transport server, the cloud service, the destination mailbox. Every message is written from the perspective of just one layer.

On top of that, error messages are umbrella terms. The same wording often describes entirely different situations because the rejecting system only has a coarse classification scheme. Enhanced status codes are designed precisely for that purpose: to form classes, not identify individual cases.

An example: A cloud service rejected a message stating that the sender was not permitted for outbound delivery. The same wording occurred in the same environment in two fundamentally different situations. In one case, a system attempted to deliver through the service to an external recipient—an actual attempt to relay externally. In the other, the recipient was a regular mailbox in the service, and only the sender domain was being objected to.

Anyone who takes the text literally looks for the same thing in both cases. And because it contains the word “outbound,” they start looking at the wrong end.

## Principle 1: A hypothesis must rule something out

Karl Popper contributed an insight to the philosophy of science that is immediately practical for troubleshooting: **A statement is useful only if it can be disproven.** An explanation that fits every conceivable observation explains nothing.

Applied here, that means: formulate your assumption so that it includes a **prediction** that can be wrong. Not “something is wrong with the sender domain,” but “if I send the same message with a different sender domain through the same path, it will arrive.”

The second formulation is valuable because it can be disproven in five minutes. You can spend hours feeding the first one with evidence without ever becoming any wiser.

A good test: before running the experiment, ask yourself what result would **disprove** your hypothesis. If you cannot think of one, you do not have a hypothesis—you have a feeling.

## Principle 2: One variable; otherwise, keep everything the same

The core of an experiment is controlling confounding variables. In practice, the opposite happens regularly: people compare two cases that happen to be available. And they almost always differ in several characteristics at the same time.

From a real case: Messages from `example-test.com` were rejected, while messages from `partner.example` arrived. The two domains differed in at least four characteristics: whether they belonged to the organization, where email was hosted, whether a strict authentication policy was configured, and the submission path. You can infer precisely nothing from two data points with four differences. Any of the four explanations fits.

So build the comparison yourself. Same submission point, same recipient, same path, same time, and **exactly one** changed characteristic. If you suspect the sender domain, change only that.

## Principle 3: Without a control test, the result is worthless

This is the part people most want to skip, and it is the most important. In clinical research, the control group is a given; in IT, it is usually omitted, and then people wonder about contradictory results.

**Your test setup must reproduce the error first.** If you cannot produce the failure case with your own means, a successful counter-test says nothing. Perhaps your test message works only because you submit it somewhere other than the original system, or because a check does not apply on your path at all.

A useful test therefore consists of at least two messages:

| | Purpose | Expected result |
|---|---|---|
| Test 1 | Control, replicates the original case | **must fail** |
| Test 2 | Hypothesis, one variable changed | should succeed |

If test 1 does not fail, your setup is not representative. Then you have learned nothing about the original case, only about your test setup, and you need to submit closer to the original.

## A worked example

Back to the case above, anonymized. Messages from one system did not reach recipients in the cloud, while other messages to the same recipients arrived without issue. Three tests through the same path, to the same recipient, a few minutes apart:

| Test | Sender domain | Hypothesis it tests | Result |
|---|---|---|---|
| 1 (control) | `example-test.com` | Setup is representative | Rejected, identical to the original |
| 2 | `example.com`, destination's own domain | The issue is the sender domain | Delivered |
| 3 | `other-test.com`, external domain from the same organization | The issue is organizational affiliation | Delivered |

Test 1 reproduced the error, so the setup was valid. Test 2 showed that the issue was tied to the sender domain rather than the recipient, mailbox, routing, or permissions. Test 3 was the truly elegant one: it specifically tested the most obvious alternative explanation and **disproved it**, because `other-test.com` belonged to the same organization and still got through.

Three messages, ten minutes, and the cause was proven rather than assumed. Before that, several hours had gone into attempts to explain it, none of which ultimately held up.

## Principle 4: Disproving is real progress

A disproven hypothesis feels like a setback. In fact, it is the only thing you know for certain. Confirmations are weak because an observation can fit several explanations. A clean disproof permanently removes an entire branch from the search space.

This is exactly where confirmation bias has its strongest effect. Once you have an assumption, you can almost always find something that fits it. In the analysis described above, there was a correlation between rejection and where the sender domain hosted its email. It looked convincing, but it was based on two data points that differed in several characteristics. The third test refuted it.

Therefore, document disproven explanations together with the reason they were rejected. This is nothing more than a lab notebook. It has two effects: whoever takes over the case later will not go down the same dead ends. And you yourself notice when you are thinking in circles because an already rejected idea returns under a new name.

In the documentation, disproven points explicitly belong alongside the proven ones. A report that contains only the correct answer conceals half the work and invites others to repeat it.

## Principle 5: Know your sample

The most subtle source of error is sampling bias, and in IT it especially affects queries that return paginated results.

You query seven days of message tracking, filter locally for a characteristic, and get no result. The obvious conclusion is that this traffic did not occur. In reality, you filtered only the first page, which at high volume may cover only a few minutes.

The correct result is: not found in the sample. It is not: does not exist. The distinction is the same as between “no effect can be demonstrated in our study” and “there is no effect.”

Two ways out work. Narrow the time window until a single page covers it completely, as indicated by the absence of a notice about additional results. Or page through all results and then evaluate them.

And a third option, often overlooked: for the question of whether something **never** occurs, a configuration check is superior to any observation. If a system has no route to a destination, it cannot deliver there, regardless of any observation window. That is the difference between an empirical and a structural argument; wherever you can use the structural one, use it.

## The transfer: Tie the burden of proof to reversibility

This is where the analogy to science ends and the engineering perspective takes over. Research seeks truth; operations seeks a working system. This leads to a standard that science does not know: **The effort required for verification depends on the reversibility of the intervention.**

Disabling a connector is one command, and undoing it is another. Reasonable evidence is sufficient because a mistake can be fixed in a minute and becomes apparent immediately. Deleting the same connector is not reversible; that justifies additional verification through the configuration of the counterpart or a server-side usage report.

The same applies to rule changes. You may introduce a purely observational stage that logs and does not reroute anything on limited evidence. It has no consequences and obtains exactly the data missing for the decisive step. Only the change that can hold back messages requires solid evidence.

Those who do not apply this standard regularly make both mistakes at once: they demand weeks of proof for a change that could be undone in seconds, and they put something into production without safeguards even though it can stop mail traffic.

## When you may stop

There is a point at which further digging creates no value: when the fix is clear but the mechanism remains unclear.

In the example above, three tests established that the sender domain was the trigger, that everything else in the mail path worked, and that there was no broader issue. Why the cloud service made exactly that decision internally remained open. It did not matter for the correction because that lay with the sending application.

Therefore, deliberately separate two questions: What must I change to make it work? And why does the system behave this way? You must answer the first; you may hand the second to the vendor. A support case with three controlled tests, timestamps, message IDs, and a working counterexample is far more valuable than a description of the symptom anyway.

Incidentally, this is also the point at which science and operations can be cleanly separated. Science may not abandon the question of mechanism. Operations must prioritize it.

## The short version

Formulate hypotheses so they can fail, and ask yourself in advance what result would disprove them. Never compare two cases that happen to be available; instead, build the comparison with exactly one changed variable. Reproduce the error in the control test before believing the counter-test. Treat disproofs as progress and document them in writing. With every query, check whether you are seeing the entire population or a sample. And base the required depth of proof on how easily the planned intervention can be reversed.

The relevant queries are available in [Analyzing Exchange mail flow: message tracking, SMTP logs, and Receive connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). If you would rather click together the commands than type them, you can find them in the [Command Generator](https://rafaelpfister.ch/tools/command-builder).

## Change log

- **October 25, 2026:** Updated source links to the new addresses from Mohr Siebeck and Microsoft Learn.

## Sources

1.  [Karl Popper: The Logic of Scientific Discovery](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161484100/): Origin of the falsification principle, according to which a statement is scientific only if it remains refutable.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): Explains why enhanced status codes are deliberately broad classes and allow the same code for different causes.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): Event types and fields, the basis for determining the final processing step.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2): Message tracking pagination logic, which encourages sampling errors.
