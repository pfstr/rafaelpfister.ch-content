---
title: "Mail loop with encryption gateway behind EXO - Prevent spoofing issues"
navTitle: "Gateway loop"
description: "If the encryption gateway (HIN in this example) is behind Exchange Online, every incoming message reaches EOP a second time: with a foreign sender domain, an invalid DKIM signature, and the gateway IP as the source. The result is a spoof verdict and junk. Four settings prevent this without disabling filtering using SCL -1: a PTR record, Enhanced Filtering disabled, a spoof exception for the gateway infrastructure, and CloudServicesMailEnabled on both connectors."
date: "2026-09-23"
kategorie: "Mail flow and SMTP"
timeToRead: "9 min read"
themen:
  - smtp-mailflow
  - microsoft-365-exchange
  - hin-gateway
  - e-mail-verschluesselung
produkte:
  - "exchange-online"
  - "hybrid-mailfluss"
  - "hin"
protokolle:
  - "mail-auth"
  - "smtp"
  - "powershell"
slug: "mail-loop-with-encryption-gateway-behind-exo-prevent-spoofing-issues"
translationId: "article-b773c8d303aee87a"
aiPrompt: |
  Du bist mein Exchange-Online-Assistent. Ich betreibe ein Verschlüsselungsgateway hinter Exchange Online (Mail-Schlaufe: EXO, Gateway, EXO). Prüfe mit mir die vier Einstellungen: PTR-Record der Gateway-IP, Enhanced Filtering auf dem Inbound-Connector, Spoof-Ausnahme in der Tenant Allow/Block List für die Gateway-Infrastruktur und CloudServicesMailEnabled auf beiden Connectoren. Erkläre mir zu jeder Einstellung, warum sie nötig ist, und hilf mir, das Ergebnis anhand der Authentication-Results-Header einer Testnachricht zu verifizieren.
translationOf: verschluesselungsgateway-hinter-exchange-online
url: https://rafaelpfister.ch/en/blog/mail-loop-with-encryption-gateway-behind-exo-prevent-spoofing-issues
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:34:09.436Z
translationReview: automatic
---

# Mail loop with encryption gateway behind EXO - Prevent spoofing issues

Many organizations in the Swiss healthcare sector operate a HIN gateway, while others use SEPPmail or TotemoMail for S/MIME and PGP. If the MX record points to Microsoft and the gateway is behind Exchange Online, every incoming message passes through filtering twice. On the second pass, EOP sees a message with a foreign sender domain, an invalid DKIM signature, and the gateway IP as the source. From the filter's perspective, this is the pattern of sender spoofing, and legitimate mail ends up in the Junk Email folder. I have configured this setup several times and describe here the four settings that make the loop work without a spoof verdict, and why each of them is necessary. The HIN gateway is used as an example; the mechanism is identical for any other gateway in the same position.

## The setup

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

A transport rule routes incoming messages to the gateway through an outbound connector. The gateway decrypts, verifies signatures, and delivers the message back to Exchange Online through an inbound connector. A header set by the gateway prevents the rule from applying again.

There are good reasons for this setup: Microsoft filters first, the gateway receives only pre-screened mail, and the organization does not need to expose its own MX record to the internet. The cost is the second pass, which cannot be disabled, only configured correctly.

## Why SCL -1 is the wrong answer

The common workaround is a transport rule on the return path that sets the Spam Confidence Level to `-1`. This causes Exchange Online to skip content filtering entirely on the second pass. It works immediately and has two drawbacks: the second pass no longer accomplishes anything, and the rule depends on a condition (connector or IP) that changes whenever the setup changes. Microsoft also describes SCL -1 as input for filtering, not as a final decision; the stamped value can differ. The four settings below address the problem where it originates: in the assessment of the delivering infrastructure.

## The four settings

1. Publish a PTR record for the public IP address of the HIN gateway in public DNS. Without this entry, the entire path does not work.
2. Fully disable Enhanced Filtering on the inbound connector through which HIN delivers to Exchange Online.
3. Create a spoof exception for the delivering HIN infrastructure in the Tenant Allow/Block List.
4. Enable `CloudServicesMailEnabled` on the outbound connector to HIN and on the inbound connector from HIN.

For item 2, first check which connectors are affected:

```powershell
Get-InboundConnector |
    Select-Object Name, Enabled, ConnectorType, EFSkipLastIP, EFSkipIPs, EFUsers, EFTestMode |
    Format-List
```

Then disable it on the HIN connector:

```powershell
$efAus = @{
    Identity     = "<Inbound-Connector HIN>"
    EFSkipLastIP = $false
    EFSkipIPs    = $null
    EFUsers      = $null
}
Set-InboundConnector @efAus
```

<details class="options-details">
<summary>Options explained</summary>

| Parameter | Effect |
|---|---|
| `EFSkipLastIP = $false` | The last hop is no longer skipped automatically. If no IP is listed in `EFSkipIPs` at the same time, Enhanced Filtering is disabled on the connector. |
| `EFSkipIPs = $null` | Clears the list of IP addresses to skip. |
| `EFUsers = $null` | Removes the restriction to individual recipients; without active Enhanced Filtering, the value has no effect, but remains unambiguous this way. |
| `EFTestMode` | Query only: shows whether the connector is in test mode. Microsoft lists the parameter as internal, but it can be read. |

</details>

Item 3, the spoof exception:

```powershell
$spoof = @{
    Identity              = "Default"
    Action                = "Allow"
    SpoofedUser           = "*"
    SendingInfrastructure = "gateway.example.com"
    SpoofType             = "External"
}
New-TenantAllowBlockListSpoofItems @spoof
```

<details class="options-details">
<summary>Options explained</summary>

| Parameter | Effect |
|---|---|
| `Identity = "Default"` | The list itself; there is only one. |
| `Action = "Allow"` | Allows the combination. `Block` classifies it as phishing instead. |
| `SpoofedUser = "*"` | The address visible in the From field. The wildcard represents any sender. A wildcard is permitted on one side of the pair, not both. |
| `SendingInfrastructure` | The source: the domain from the PTR record of the delivering IP (item 1). Without a PTR record, the list accepts only `<IP>/24`. |
| `SpoofType = "External"` | Applies to foreign sender domains. `Internal` covers your own accepted domains and requires a second entry. |

</details>

Item 4, the Cross-Premises headers:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

I explain below under item 4 why `TreatMessagesAsInternal` is included in the same command for the inbound connector.

## 1: PTR record

EOP identifies the delivering infrastructure through a reverse lookup of the source IP. The PTR value appears in the `Authentication-Results` header as sending infrastructure, and the spoof exception from item 3 refers to exactly this value. If the PTR record is missing, Exchange Online evaluates every message on the return path with `PTR:InfoDomainNonexistent`, and the exception must apply to an entire `/24` rather than a name. A missing reverse DNS entry has also long been an established negative signal in every spam filter. The entry should resolve forward to the same IP.

With HIN, the delivering IP is that of the HIN mail gateway through which messages return to Exchange Online. Who manages the PTR record for this IP depends on the operating model; responsibility should be clarified before the change.

## 2: Disable Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) is designed for the reverse setup: gateway in front of Microsoft 365, with the MX pointing to the gateway. In that setup, EOP skips the last hop and evaluates the true origin of the message.

It does not work in the loop setup. Microsoft states in its documentation that Enhanced Filtering is not intended for services that process mail after Microsoft 365, and lists non-linear routing (internet, Microsoft 365, external system, Microsoft 365) as unsupported. The documentation names precisely the real-world symptom as a consequence: Microsoft 365 checks returning mail again, assigns a `compauth` value, and the message can be classified as spam.

If Enhanced Filtering remains enabled, EOP skips the HIN IP and evaluates the IP before it. In the loop, that is either a Microsoft-owned IP from the first pass or the original external sender. Both are wrong:

* The evaluation is based on an IP that no longer provides meaningful information about the message on the second pass.
* The spoof exception from item 3 never applies because it is tied to the HIN infrastructure that EOP has just skipped.

That is why Enhanced Filtering must be off on this connector: only then is HIN visible and addressable as the delivering infrastructure.

## 3: Spoof exception

A spoof entry is always a pair consisting of the spoofed user (the From address or its domain) and the sending infrastructure (the source). Only that exact combination is allowed.

`SpoofedUser = "*"` together with the HIN infrastructure therefore means: any From address may deliver through HIN without triggering the spoof verdict. Any other source using the same senders continues to be checked. The exception is not a spam-filter bypass: spam, content, and threat checks continue unchanged on the second pass. A message can therefore still be filtered because of its content; it is simply no longer considered a forgery.

The command above deliberately contains only `SpoofType = "External"`. Your own mail, meaning messages with senders from your own accepted domains, should not return through the gateway. It goes from internal systems or on-premises Exchange directly to Exchange Online without touching HIN. The Spoof Intelligence evaluation shows whether that is actually the case in your environment:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

If your own domains appear there with the HIN infrastructure and `SpoofType Internal`, a second entry with `SpoofType = "Internal"` is needed—or better yet, identify the reason internal mail is passing through the gateway.

Spoof entries do not expire on their own. If the gateway is decommissioned or its infrastructure changes, the entry should be removed.

## 4: CloudServicesMailEnabled

This is the only point Microsoft does not explicitly document for this scenario. It is my conclusion based on the documented behavior of the parameter: it preserves the Cross-Premises headers across the loop.

The parameter controls the handling of internal `X-MS-Exchange-Organization-*` headers. On the outbound connector, they are converted into `X-MS-Exchange-CrossPremises-*` and thus survive the route through HIN. On the inbound connector, they are converted back to `X-MS-Exchange-Organization-*` and replace headers with the same names already present in the message. If the parameter is set to `$false`, the connector removes these headers.

In practice, this means that what Exchange Online determined on the first pass, including the authentication status and internal designation, survives the loop instead of being stripped upon reentry. The headers of the delivered message then contain `X-CrossPremisesHeadersPromoted`, and the original verification results remain under `Authentication-Results-Original`. This alone does not prevent a spoof verdict (items 1 through 3 are responsible for that), but it preserves the verdict from the first pass.

Three things must be considered:

* If the connectors were created by the Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), a subsequent HCW run overwrites the setting. The HIN connectors should therefore be separate, manually created connectors.
* On the inbound connector, `CloudServicesMailEnabled $true` and `TreatMessagesAsInternal $true` are mutually exclusive. If `TreatMessagesAsInternal` is already set to `$true`, Exchange Online rejects the command. That is why both parameters must be included in the same `Set-InboundConnector` invocation, as shown above.
* Microsoft recommends setting the parameter only at the instruction of support or product documentation. There is no such documentation for the loop; the decision is yours and should be documented.

## Secure delivery

Because Exchange Online adopts headers from the return path after item 4 and accepts every From address from the HIN infrastructure after item 3, the inbound connector must be restricted so that only HIN can deliver through it. For a connector of type `Partner`, this means `RestrictDomainsToCertificate` with `TlsSenderCertificateName` or `RestrictDomainsToIPAddresses` with `SenderIPAddresses`, in each case together with `RequireTls`. Without this restriction, any source that reaches the connector could deliver with arbitrary senders and adopted organization headers.

```powershell
$absicherung = @{
    Identity                     = "<Inbound-Connector HIN>"
    RequireTls                   = $true
    RestrictDomainsToCertificate = $true
    TlsSenderCertificateName     = "gateway.example.com"
}
Set-InboundConnector @absicherung
```

## Verification

After the change, send a test message from an external domain with a DMARC policy through the gateway and check the headers of the delivered message. In the `Authentication-Results` header of the second pass, the HIN domain should appear as the sending infrastructure and `compauth` should be `pass` with a reason from the Tenant Allow entries, rather than `fail reason=001`. `X-CrossPremisesHeadersPromoted` and `Authentication-Results-Original` show that item 4 is taking effect. The [Mail Header Analyzer](/tools/header-analyzer) on this site displays both passes as a flowchart and highlights the transitions between Exchange Online and the gateway.

## Sources

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): distinction between linear and non-linear routing, guidance on services behind Microsoft 365, consequence `compauth` and spam classification, SCL -1 as input rather than a decision, PowerShell parameters `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): behavior of `CloudServicesMailEnabled` (conversion and promotion of Cross-Premises headers, removal when `$false`), mutual exclusion with `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` and `RestrictDomainsToIPAddresses` for partner connectors.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` on the outbound connector.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): syntax for spoof entries, wildcard rules, determining the sending infrastructure through a PTR record or `/24`, coverage of DMARC-related spoof verdicts.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): parameters `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): evaluation of detected spoof pairs from the last seven days.
