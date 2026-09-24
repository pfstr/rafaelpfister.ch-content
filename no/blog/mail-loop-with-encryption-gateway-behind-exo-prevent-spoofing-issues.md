---
title: "Mail loop with encryption gateway behind EXO - prevent spoofing issues"
navTitle: "Gateway loop"
description: "If the encryption gateway (HIN in this example) is behind Exchange Online, every incoming message reaches EOP a second time: with an external sender domain, an invalid DKIM signature and the gateway IP as the source. The result is a spoof verdict and Junk. Four settings prevent this without disabling filtering via SCL -1: PTR record, Enhanced Filtering off, a spoof exception for the gateway infrastructure and CloudServicesMailEnabled on both connectors."
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
url: https://rafaelpfister.ch/no/blog/mail-loop-with-encryption-gateway-behind-exo-prevent-spoofing-issues
translationSourceHash: ac3ae7b36a2682c2913479a6d9d0356d1abd96c68459bcfb4912362884e0a708
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:36:54.878Z
translationReview: automatic
---

# Mail loop with encryption gateway behind EXO - prevent spoofing issues

Many organisations in the Swiss healthcare sector operate an HIN gateway, while others use SEPPmail or totemomail for S/MIME and PGP. If the MX record points to Microsoft and the gateway is behind Exchange Online, every incoming message passes through filtering twice. On the second pass, EOP sees a message with an external sender domain, an invalid DKIM signature and the gateway IP as the source. From the filter's perspective, this is the pattern of sender spoofing, and legitimate mail ends up in the Junk folder. I have configured this setup several times and describe here the four settings that make the loop work without a spoof verdict, and why each one is necessary. The HIN gateway is used as an example; the mechanism is identical for any other gateway in the same position.

## The setup

```text
Absender > MX > Exchange Online (1. Durchlauf)
                    > HIN-Gateway: Entschlüsselung, Signaturprüfung
                            > Exchange Online (2. Durchlauf) > Postfach
```

A transport rule routes incoming messages to the gateway via an outbound connector. The gateway decrypts, checks signatures and delivers the message back to Exchange Online through an inbound connector. A header field set by the gateway prevents the rule from being applied again.

There are good reasons for this setup: Microsoft filters first, the gateway receives only pre-checked mail, and operations do not have to expose their own MX to the internet. The price is the second pass, which cannot be disabled, only configured correctly.

## Why SCL -1 is the wrong answer

The common workaround is a transport rule on the return path that sets the Spam Confidence Level to `-1`. This causes Exchange Online to skip content inspection entirely on the second pass. It works immediately and has two drawbacks: the second pass no longer achieves anything, and the rule depends on a condition (connector or IP) that shifts with every change. Microsoft also describes SCL -1 as input for filtering, not as a final decision; the stamped value may differ. The four settings below solve the problem where it arises: in the assessment of the delivering infrastructure.

## The four settings

1. Publish a PTR record for the public IP of the HIN gateway in public DNS. Without this record, the entire path does not work.
2. Completely disable Enhanced Filtering on the inbound connector through which HIN delivers to Exchange Online.
3. Create a spoof exception for the delivering HIN infrastructure in the Tenant Allow/Block List.
4. Enable `CloudServicesMailEnabled` on the outbound connector to HIN and on the inbound connector from HIN.

For point 2, first check which connectors are affected:

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
<summary>Explanation of options</summary>

| Parameter | Effect |
|---|---|
| `EFSkipLastIP = $false` | The last hop is no longer automatically skipped. If no IP is simultaneously specified in `EFSkipIPs`, Enhanced Filtering is disabled on the connector. |
| `EFSkipIPs = $null` | Clears the list of IP addresses to skip. |
| `EFUsers = $null` | Removes the restriction to individual recipients; without active Enhanced Filtering, the value has no significance, but remains unambiguous this way. |
| `EFTestMode` | Query only: shows whether the connector is in test mode. Microsoft lists the parameter as internal, but it is readable. |

</details>

Point 3, the spoof exception:

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
<summary>Explanation of options</summary>

| Parameter | Effect |
|---|---|
| `Identity = "Default"` | The list itself; there is only one. |
| `Action = "Allow"` | Allows the combination. `Block` classifies it as phishing instead. |
| `SpoofedUser = "*"` | The address visible in the From field. The wildcard represents any sender. A wildcard is permitted on one side of the pair, not on both. |
| `SendingInfrastructure` | The source: the domain from the PTR record of the delivering IP (point 1). Without a PTR record, the list accepts only `<IP>/24`. |
| `SpoofType = "External"` | Applies to external sender domains. `Internal` covers your own accepted domains and requires a second entry. |

</details>

Point 4, the cross-premises headers:

```powershell
Set-OutboundConnector -Identity "<Outbound-Connector zu HIN>" -CloudServicesMailEnabled $true
$crossPremises = @{
    Identity                 = "<Inbound-Connector HIN>"
    TreatMessagesAsInternal  = $false
    CloudServicesMailEnabled = $true
}
Set-InboundConnector @crossPremises
```

I explain below under point 4 why `TreatMessagesAsInternal` is included in the same command for the inbound connector.

## On 1: PTR record

EOP identifies the delivering infrastructure through the reverse lookup of the source IP. The PTR value appears in the `Authentication-Results` header as the sending infrastructure, and this is precisely the value referenced by the spoof exception in point 3. If the PTR record is missing, Exchange Online assesses every message on the return path with `PTR:InfoDomainNonexistent`, and the exception must apply to an entire `/24` rather than to a name. A missing reverse DNS entry has also long been an established negative signal in every spam filter. The record should resolve forward to the same IP.

With HIN, the delivering IP is that of the HIN mail gateway through which messages return to Exchange Online. Who manages the PTR record for this IP depends on the operating model; responsibility should be clarified before the changeover.

## On 2: disable Enhanced Filtering

Enhanced Filtering for Connectors (Skip Listing) is designed for the reverse setup: gateway before Microsoft 365, with the MX pointing to the gateway. There, EOP skips the last hop and assesses the true origin of the message.

This does not work in the loop setup. Microsoft states in its documentation that Enhanced Filtering is not intended for services that process mail after Microsoft 365, and lists non-linear routing (Internet, Microsoft 365, external system, Microsoft 365) as unsupported. The documentation names precisely the symptom seen in practice as a consequence: Microsoft 365 checks returning mail again, assigns a `compauth` value, and the message may be classified as spam.

If Enhanced Filtering remains active, EOP skips the HIN IP and assesses the IP before it. In the loop, this is either a Microsoft-owned IP from the first pass or the original external sender. Both are wrong:

* The assessment is applied to an IP that no longer provides any information about the message on the second pass.
* The spoof exception from point 3 never applies because it is tied to the HIN infrastructure that EOP has just skipped.

Therefore, Enhanced Filtering must be off on this connector: only then is HIN visible and addressable as the delivering infrastructure.

## On 3: spoof exception

A spoof entry is always a pair consisting of the spoofed user (the From address or its domain) and the sending infrastructure (the source). Only this combination is allowed.

`SpoofedUser = "*"` together with the HIN infrastructure therefore means: any From address may deliver through HIN without triggering the spoof verdict. Any other source that uses the same senders continues to be checked. The exception is not a spam filter bypass: spam, content and threat scanning continue unchanged on the second pass. A message may therefore still be filtered because of its content; it is simply no longer considered forged.

The command above deliberately contains only `SpoofType = "External"`. Your own mail, meaning messages with senders from your own accepted domains, should not return through the gateway. They go from internal systems or on-premises Exchange directly to Exchange Online without touching HIN. Whether this is actually the case in your environment is shown by the Spoof Intelligence evaluation:

```powershell
Get-SpoofIntelligenceInsight |
    Select-Object SpoofedUser, SendingInfrastructure, SpoofType, MessageCount, Action |
    Sort-Object MessageCount -Descending |
    Format-Table -AutoSize |
    Out-String -Width 200
```

If your own domains appear there with the HIN infrastructure and `SpoofType Internal`, a second entry with `SpoofType = "Internal"` is required, or preferably: fix the reason why internal mail is passing through the gateway.

Spoof entries do not expire on their own. If the gateway is removed or its infrastructure changes, the entry should be removed.

## On 4: CloudServicesMailEnabled

This is the only point that Microsoft does not explicitly document for this scenario. It is my deduction from the documented behaviour of the parameter: it preserves cross-premises headers across the loop.

The parameter controls the handling of internal `X-MS-Exchange-Organization-*` headers. On the outbound connector, they are converted to `X-MS-Exchange-CrossPremises-*` and therefore survive the path through HIN. On the inbound connector, they are converted back to `X-MS-Exchange-Organization-*` and replace headers with the same name that are already present in the message. If the parameter is set to `$false`, the connector removes these headers.

In practical terms: what Exchange Online determined on the first pass, including the authentication status and the internal designation, survives the loop instead of being stripped upon re-entry. The headers of the delivered message then contain `X-CrossPremisesHeadersPromoted`, and the original check results remain available under `Authentication-Results-Original`. This alone does not prevent a spoof verdict (points 1 to 3 are responsible for that), but it preserves the verdict from the first pass.

Three things must be considered:

* If the connectors were created by the Hybrid Configuration Wizard (`ConnectorSource: HybridWizard`), a subsequent HCW run overwrites the setting. The HIN connectors should therefore be separate connectors created manually.
* On the inbound connector, `CloudServicesMailEnabled $true` and `TreatMessagesAsInternal $true` are mutually exclusive. If `TreatMessagesAsInternal` is already set to `$true`, Exchange Online rejects the command. Therefore, both parameters belong in the same `Set-InboundConnector` call, as shown above.
* Microsoft recommends setting the parameter only on the instruction of support or product documentation. There is no such documentation for the loop; the decision is yours and should be documented.

## Secure delivery

Because Exchange Online adopts headers from the return path after point 4 and accepts every From address from the HIN infrastructure after point 3, the inbound connector must be restricted so that only HIN can deliver through it. For a connector of type `Partner`, these are `RestrictDomainsToCertificate` with `TlsSenderCertificateName`, or `RestrictDomainsToIPAddresses` with `SenderIPAddresses`, in each case together with `RequireTls`. Without this restriction, any source that reaches the connector could deliver with arbitrary senders and adopted organisation headers.

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

After the changeover, send a test message from an external domain with a DMARC policy through the gateway and check the headers of the delivered message. In the `Authentication-Results` header of the second pass, the HIN domain should appear as the sending infrastructure and `compauth` should be set to `pass` with a reason from the Tenant Allow entries, rather than to `fail reason=001`. `X-CrossPremisesHeadersPromoted` and `Authentication-Results-Original` show that point 4 is working. The [Mail Header Analyzer](/tools/header-analyzer) on this site displays both passes as a flow diagram and highlights the transitions between Exchange Online and the gateway.

## Sources

1.  [Enhanced filtering for connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): distinction between linear and non-linear routing, note on services behind Microsoft 365, consequence `compauth` and spam classification, SCL -1 as input rather than decision, PowerShell parameters `EFSkipLastIP`, `EFSkipIPs`, `EFUsers`.

2.  [Set-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-inboundconnector): behaviour of `CloudServicesMailEnabled` (conversion and promotion of cross-premises headers, removal when `$false`), exclusion with `TreatMessagesAsInternal`, `RestrictDomainsToCertificate` and `RestrictDomainsToIPAddresses` for partner connectors.

3.  [Set-OutboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-outboundconnector): `CloudServicesMailEnabled` on the outbound connector.

4.  [Allow or block email using the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-email-spoof-configure): syntax of spoof entries, wildcard rules, determination of the sending infrastructure via PTR record or `/24`, coverage of DMARC-related spoof verdicts.

5.  [New-TenantAllowBlockListSpoofItems](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-tenantallowblocklistspoofitems): parameters `SpoofedUser`, `SendingInfrastructure`, `SpoofType`, `Action`.

6.  [Get-SpoofIntelligenceInsight](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-spoofintelligenceinsight): evaluation of detected spoof pairs from the past seven days.
