---
title: "What We Can Learn from Science for IT Troubleshooting"
navTitle: "Controlled Experiments"
description: "Falsifiability, control groups, confounding variables, and sampling bias: the method natural sciences have used for centuries solves exactly the problems where IT troubleshooting regularly fails, illustrated with examples from mail flow."
date: "2026-08-11"
kategorie: "SMTP / Mail Flow"
timeToRead: "15 min read"
themen:
  - smtp-mailflow
  - testing
  - exchange-onprem-hybrid
hauptthema: "smtp-mailflow"
produkte:
  - "uebergreifend"
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
translationSourceHash: e3fff70bc1386c28d78713ec89a35b4d6c29b7f16e809e8a84bd9850a40a261c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:15:13.834Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/what-we-can-learn-from-the-natural-sciences-for-it-troubleshooting
---

# What We Can Learn from Science for IT Troubleshooting

A message does not arrive. The protocol provides an error message that immediately suggests an explanation. You investigate that explanation, find evidence, and after two hours it turns out the explanation was wrong and the evidence was coincidental.

This is not a beginner's mistake; it is the rule. And it is remarkable that our industry rarely has a method for this problem, even though one has existed for centuries and works exceptionally well. The natural sciences have exactly the same task: inferring causes from observations in systems that cannot be fully understood at a glance.

This article applies five principles of the scientific method to mail flow troubleshooting. The examples come from real-world practice, but the approach is not specific to email.

## Why IT Troubleshooting Is Systematically Vulnerable

Mail flow is a chain of systems, each with its own view of the same message: the gateway, the filtering layer, the local transport server, the cloud service, the target mailbox. Every message is written from the perspective of exactly one layer.

In addition, error messages are catch-all terms. The same wording often describes entirely different situations because the rejecting system only has a coarse classification scheme. Enhanced status codes are designed precisely for this purpose: to form classes, not identify individual cases.

An example: A cloud service rejected a message stating that the sender was not allowed for outbound delivery. The same wording appeared in the same environment in two fundamentally different scenarios. In one case, a system was attempting to deliver through the service to an external recipient—a genuine attempt to relay externally. In the other, the recipient was a regular mailbox in the service, and only the sender domain was being flagged.

Anyone taking the text literally will look for the same thing in both cases. And because it contains the word “outbound,” they will first look in the wrong place.

## Principle 1: A Hypothesis Must Rule Something Out

Karl Popper contributed an insight to the philosophy of science that is immediately practical for troubleshooting: **A statement is useful only if it can be disproven.** An explanation that fits every conceivable observation explains nothing.

Applied here, that means: formulate your assumption so that it contains a **prediction** that can be wrong. Not “something is wrong with the sender domain,” but “if I send the same message with a different sender domain by the same route, it will arrive.”

The second formulation is valuable because it can be disproven in five minutes. You can feed the first one with evidence for hours without becoming any wiser.

A good test: before attempting anything, ask yourself what result would **disprove** your hypothesis. If you cannot think of one, you do not have a hypothesis—you have a feeling.

## Principle 2: One Variable, Everything Else the Same

The core of an experiment is controlling confounding variables. In practice, the opposite regularly happens: two cases that happen to be available are compared. And they almost always differ in several characteristics at the same time.

From a real case: Messages from `example-test.com` were rejected, while messages from `partner.example` arrived. The two domains differed in at least four characteristics: whether they belonged to the organization, where their email was hosted, whether a strict authentication policy was configured, and the submission path. Nothing at all can be inferred from two data points with four differences. Any of the four explanations fits.

Therefore, build the comparison yourself. Same submission point, same recipient, same route, same time, and **exactly one** changed characteristic. If you suspect the sender domain, change only that.

## Principle 3: Without a Control Experiment, the Result Is Worthless

This is the part people most like to skip, and it is the most important. In clinical research, the control group is a given; in IT, it is usually omitted, and people then wonder about contradictory results.

**Your test setup must first reproduce the error.** If you cannot generate the failure case using your own means, a successful comparison test tells you nothing. Perhaps your test message works only because you submit it somewhere different from the original system, or because a check does not apply on your route at all.

A useful test therefore consists of at least two messages:

| | Purpose | Expectation |
|---|---|---|
| Test 1 | Control, replicates the original case | **must fail** |
| Test 2 | Hypothesis, one variable changed | should succeed |

If Test 1 does not fail, your setup is not representative. Then you have learned nothing about the original case, only about your test setup, and you must submit closer to the original.

## A Worked Example

Back to the case above, anonymized. Messages from one system did not reach recipients in the cloud, while other messages to the same recipients arrived without issue. Three tests via the same route, to the same recipient, a few minutes apart:

| Test | Sender domain | Hypothesis it tests | Result |
|---|---|---|---|
| 1 (control) | `example-test.com` | Setup is representative | Rejected, identical to the original |
| 2 | `example.com`, target's own domain | It is caused by the sender domain | Delivered |
| 3 | `other-test.com`, external domain from the same organization | It is caused by organizational membership | Delivered |

Test 1 reproduced the error, so the setup was valid. Test 2 showed that the issue depended on the sender domain, not the recipient, mailbox, routing, or permissions. Test 3 was the truly elegant one: it specifically tested the most obvious alternative explanation and **disproved it**, because `other-test.com` belonged to the same organization and still got through.

Three messages, ten minutes, and the cause was established rather than assumed. Before that, several hours had gone into attempts at explanation, none of which ultimately held up.

## Principle 4: Disproving Is the Real Progress

A disproven hypothesis feels like a setback. In fact, it is the only thing you know for certain. Confirmations are weak because one observation can fit several explanations. A clean disproof removes an entire branch from the search space, permanently.

This is exactly where confirmation bias has the strongest effect. Once you have an assumption, you can almost always find something that fits it. In the analysis described above, there was a correlation between rejection and where the sender domain hosted its email. It looked convincing, but it was based on two data points that differed in several characteristics. The third test refuted it.

Therefore, document the disproven explanations along with the reason they were rejected. This is no different from a lab notebook. It has two effects: anyone who takes over the case later will not run into the same dead ends. And you will notice when you are thinking in circles because an idea that was already rejected returns under a new name.

In documentation, rejected points explicitly belong alongside substantiated ones. A report containing only the correct answer conceals half the work and invites others to repeat it.

## Principle 5: Know Your Sample

The most subtle source of error is sampling bias, and in IT it primarily affects queries that return results page by page.

You query seven days of message tracking, filter locally by a characteristic, and get no result. The obvious conclusion is that this traffic did not occur. In reality, you filtered only the first page, which may cover just a few minutes when volume is high.

The correct conclusion is: not found in the sample. It is not: does not exist. The difference is the same as between “no effect could be demonstrated in our study” and “there is no effect.”

Two ways out work. Reduce the time window until one page covers it completely, as indicated by the absence of a notice about additional results. Or page through all results and then evaluate them.

And a third, often overlooked one: for the question of whether something **never** occurs, a configuration check is superior to any observation. If a system has no route to a destination, it cannot deliver there, regardless of any observation window. That is the difference between an empirical and a structural argument, and where you can have the structural one, use it.

## The Transfer: Tie the Burden of Proof to Reversibility

This is where the analogy to science ends and the engineering perspective takes over. Research seeks truth; operations seeks a functioning system. This leads to a standard science does not have: **The effort required for proof depends on the reversibility of the intervention.**

Disabling a connector is one command, and undoing it is another. Well-founded indications are sufficient for that because a mistake can be fixed in one minute and becomes apparent immediately. Deleting the same connector is irreversible; in that case, the additional proof provided by the configuration of the counterpart or a server-side usage report is worthwhile.

The same applies to rule changes. You may introduce a purely observational stage that logs and does not redirect with limited evidence. It has no consequences and obtains exactly the data missing for the decisive step. Only a change that can hold back messages requires solid evidence.

Those who do not apply this standard regularly make both mistakes at once: they demand weeks of proof for a change that could be undone in seconds, and they enable something without safeguards that can stop mail traffic.

## When You May Stop

There is a point at which further digging no longer creates value: when the fix is clear but the mechanism remains unclear.

In the example above, after three tests it was established that the sender domain was the trigger, that everything else in the mail path worked, and that there was no broader problem. Why the cloud service made that exact internal decision remained open. That did not matter for the correction, because it belonged in the sending application.

Therefore, consciously separate two questions. What do I need to change to make it work? And why does the system behave this way? You must answer the first; you may hand the second to the manufacturer. A support case with three controlled tests, timestamps, message IDs, and a working counterexample is far more valuable than a description of the symptom anyway.

Incidentally, this is also the point at which science and operations can be cleanly separated. Science may not abandon the question of the mechanism. Operations must prioritize it.

## The Short Version

Formulate hypotheses so they can fail, and ask yourself beforehand which result would disprove them. Never compare two cases that happen to be available; instead, build the comparison with exactly one changed variable. Reproduce the error in the control experiment before believing the comparison test. Treat disproofs as progress and document them in writing. For every query, check whether you are seeing the full set or a sample. And base the required depth of proof on how easily the planned intervention can be reversed.

The specific queries are available in [Analyzing Exchange Mail Flow: Message Tracking, SMTP Logs, and Receive Connectors](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren). If you prefer clicking the commands together rather than typing them, you can find them in the [Command Generator](https://rafaelpfister.ch/tools/command-builder).

## Sources

1.  [Karl Popper: The Logic of Scientific Discovery](https://www.mohrsiebeck.com/buch/logik-der-forschung-9783161584350): Origin of the falsification principle, according to which a statement is scientific only if it remains open to disproof.

2.  [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463): explains why enhanced status codes are deliberately broad classes and allow the same code for different causes.

3.  [Message tracking in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking): event types and fields, the basis for determining the last processing step.

4.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): pagination logic in message tracking, which encourages sampling errors.
