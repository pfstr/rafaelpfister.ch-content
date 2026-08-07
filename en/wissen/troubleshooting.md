---
title: "Troubleshooting and Diagnosis: Systematic Fault Finding in Infrastructure"
blatt: "troubleshooting"
description: "The methodology behind successful fault finding: a layered model from the connection to the application, taking error messages literally, reproducing minimally, correlating changes, and documenting findings."
fakten:
  - { label: "Basic principle", wert: "Narrow down layer by layer instead of guessing" }
  - { label: "Diagnostic order", wert: "Reachability, connection, TLS, protocol, application" }
  - { label: "Most important source", wert: "the literal error message including its code" }
  - { label: "Classic triggers", wert: "Changes: updates, certificates, firewall rules, DNS" }
  - { label: "Network tools", wert: "nc, dig, openssl s_client, tcpdump", href: "/en/kb/tcp" }
  - { label: "Mail tools", wert: "SMTP dialog, header analysis, message trace", href: "/en/kb/smtp" }
werbung: ["tools", "newsletter"]
ctaThemen: ["smtp-mailflow"]
---

# Troubleshooting and Diagnosis: Systematic Fault Finding in Infrastructure

Good fault finding is not a talent but a procedure. It differs from frantic trial and error in a single discipline: every check narrows the problem space measurably, and every finding is recorded before the next hypothesis is taken up.

## The layer model as a search path

Almost every fault between two systems can be narrowed down along the same staircase: **name resolution** (does DNS return the expected address?), **reachability** (does a TCP connection come up?), **transport security** (does the TLS handshake succeed, is the certificate correct?), **protocol** (does the service answer correctly, and with which error code?), **application** (are permissions, configuration, and data correct?). The value of the staircase lies in the assignment: a timeout at step two is a network or firewall matter, a certificate error at step three is a PKI matter, and a 5xx at step four is a service matter. Knowing the step usually means knowing the responsible team as well.

## Taking error messages literally

Error codes are diagnostic currency, not decoration: an SMTP `550` differs fundamentally from a `451`, an LDAP `err=49` with subcode `52e` names exactly "wrong password", and an HTTP `403` refutes the hypothesis that "login is broken". The productive attitude: the message describes the state more precisely than any assumption. Preserve the original wording, look up the code, and only then interpret.

## Reproducing and minimizing

A fault that shows itself on command is half solved. Therefore: reproduce the problem with the smallest possible tool, one level below the affected application. If the mail client fails, conduct the SMTP dialog by hand; if the appliance integration fails, check the LDAP bind with a one-liner. The minimal example separates an environment problem from an application problem and incidentally supplies the evidence for support cases.

## The question of change

Infrastructure that worked yesterday and does not work today has almost always changed, and rarely on its own. The most productive candidates: installed updates, expired or renewed certificates, adjusted firewall rules, DNS changes, and expired tokens or passwords. A look at the change log, at certificate expiry dates, and the question "what happened exactly x days ago?" (expiry periods!) regularly beats hours of poking around by a wide margin.

## Recording what happened

The last step separates professionals from repeat offenders: document cause, evidence, and remedy briefly, ideally where the next person will look. Faults thus turn into check items, check items into checklists, and recurring patterns into monitoring rules.
