---
title: "Facilitator in Microsoft Teams: What the meeting agent can and cannot do"
navTitle: "Teams Facilitator"
description: "Facilitator is a Copilot agent that writes shared notes in scheduled Teams meetings, manages the agenda, monitors time, and answers questions visibly to everyone. This article explains features, licensing and admin requirements, useful use cases, and limitations related to meeting types, language, and compliance."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min read"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-in-microsoft-teams-what-the-meeting-agent-can-and-cannot-do"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/en/blog/facilitator-in-microsoft-teams-what-the-meeting-agent-can-and-cannot-do
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:39:14.680Z
translationReview: required
---

Since Microsoft began rolling out Facilitator in Teams meetings, a new entry has appeared in the meeting options in many tenants: “Allow Copilot and Facilitator.” The name does not make clear what it does. Facilitator is an AI agent that joins the meeting as a participant and works visibly for everyone. This creates both its possible uses and its limitations.

## What Facilitator is

Facilitator is a Microsoft 365 Copilot agent that acts as a moderation assistant in a scheduled Teams meeting. During the meeting, it writes notes on a shared Loop page that all internal participants can read and edit. It reads the agenda from the invitation, displays an agenda timer on the meeting stage, and responds to @mentions in the meeting chat.

The difference from other AI features in Teams is visibility:

| Feature | Who sees the result | When |
|---|---|---|
| Copilot in Teams meetings | only the person asking | during and after the meeting |
| Intelligent Recap | participants with a Teams Premium or Copilot license | after the meeting |
| Facilitator | all internal participants | during the meeting; notes also afterward |

A question to Copilot remains private. A question to Facilitator and its answer appear in the meeting chat for everyone. Facilitator therefore works collectively for all participants.

## Overview for users

The following applies to all features: The meeting is a scheduled Teams meeting within your own organization, with no external participants; “Allow Copilot and Facilitator” is enabled in the meeting options; and the spoken language was selected when starting. Only the person who starts Facilitator or assigns it tasks needs a Copilot license. All other internal participants can see the results without their own Copilot license.

| Feature | What you can do with it | Who can use it | Additional requirements |
|---|---|---|---|
| Start Facilitator | Turn on the agent before or during the meeting | People with a Microsoft 365 Copilot license | Loop enabled in Teams; “Facilitator” app not blocked |
| Live notes | Decisions, open questions, and tasks are continuously captured in the “Notes” tab; you can correct and add to them directly | Read and edit: all internal participants | Transcription is running (starts automatically); only speech in the selected language is captured, not chat messages |
| Agenda and check-off | The agenda from the invitation appears in chat, and discussed items are checked off | View: all internal participants | Agenda is in the invitation or provided through @Facilitator in chat; invitation is not encrypted; at least one other person besides the organizer is in the meeting |
| Timer and reminders | Countdown for an item, notice at the halfway point, before the meeting ends, and when it runs over | Request timer: Copilot license; view: all internal participants | none |
| Questions about the meeting | Ask via @Facilitator in chat, for example, “What did we decide on item 2?”; question and answer are visible to everyone | Ask: Copilot license; read answers: all internal participants | Questions after the meeting ends only with the “During and after the meeting” option |
| Questions with web search | Brief research during the meeting, for example about a product version | Copilot license | Web search for Copilot not disabled in the tenant |
| Create a Word document | Generate a document from the notes, such as draft meeting minutes | Copilot license | Stored in the requester's OneDrive |
| Create a PowerPoint presentation | Generate a presentation from the meeting | Copilot license | Public Preview: participation in Teams Public Preview |
| Transfer tasks to Planner | Transfer captured tasks to Planner with “Accept to sync” | Copilot license | Public Preview: participation in Teams Public Preview |
| Notes in an in-person meeting (smartphone) | Use the Teams app on a smartphone to document an in-person conversation | Copilot license | Teams app on a smartphone |
| Notes in a Teams Room | Display chat, notes, and timers on the room screen | Copilot license only for the organizer | Teams Rooms on Windows; for unscheduled in-person meetings, Teams Rooms Pro (Public Preview) |

## Features and status

Microsoft lists features separately in the admin documentation as generally available and Public Preview (as of November 2026):

| Feature | Status |
|---|---|
| Live notes in the “Notes” tab (Loop page, collaboratively editable) | generally available |
| Answering questions using meeting content and web search | generally available |
| Agenda timer with notifications at the halfway point and before the meeting ends | generally available |
| Create a Word document from the meeting | generally available |
| @mention Facilitator in the meeting chat | generally available |
| Notes on a smartphone for in-person meetings | generally available |
| Teams Rooms: chat, notes, and timer in the room | generally available |
| Capture tasks and synchronize with Planner | Public Preview |
| Create a PowerPoint presentation from the meeting | Public Preview |
| Notes for unscheduled in-person meetings in Teams Rooms | Public Preview |

Regarding the agenda: If the invitation contains an agenda, Facilitator adopts it, posts it in chat, and checks off items once they have been discussed. Items are checked off at intervals, not immediately. If the meeting runs past the scheduled end time, it points this out. On request, it also sets a countdown for an individual item.

Regarding notes: Facilitator summarizes decisions, open questions, and tasks. It relies exclusively on spoken words through the transcript. Anything only written in the meeting chat is not included in the notes. The notes can be manually corrected and supplemented at any time; Facilitator continues writing on the same page.

## Requirements

Several conditions must be met at the same time for operation:

- **License:** Anyone who adds Facilitator to a meeting or turns it on during the meeting needs a Microsoft 365 Copilot license in addition to an appropriate Microsoft 365 and Teams license. Participants without a Copilot license cannot ask Facilitator questions, but they can see the notes, other participants’ questions, and its answers.
- **Loop:** Loop experiences in Teams must be enabled in the tenant. Without Loop, there are no notes.
- **Transcription:** Turning on Facilitator starts transcription automatically. If the “Allow Copilot and Facilitator” meeting option is set to “Only during the meeting,” Facilitator runs without a saved transcript; questions after the meeting ends are then not possible.
- **App approval:** The “Facilitator” app is allowed by default, unless all apps are blocked in the Teams Admin Center.

In the invitation, enable Facilitator under **Options** > **Copilot and other AI** using the “Allow Copilot and Facilitator” setting. Alternatively, it can be turned on during the meeting through the Copilot menu.

## Facilitator is grayed out

In the meeting, Teams shows three toggles in the “Meeting AI” menu: Copilot, Facilitator, and Intelligent Recap. If all three are grayed out and the message “You don’t have access to AI in this meeting” appears, the cause is the license or the meeting itself. If only Facilitator is grayed out, the cause is the app or Loop.

| Symptom | Possible cause | Check |
|---|---|---|
| All three toggles grayed out | The account has neither a Microsoft 365 Copilot license nor a Teams Premium license | Microsoft 365 Admin Center > Users > Licenses. A newly assigned license often only takes effect in Teams after a few hours or after signing in again |
| All three toggles grayed out | Instant meeting (“Meet now”), meeting started from chat, channel meeting, or call | Create the meeting through the calendar as a scheduled Teams meeting |
| All three toggles grayed out | Meeting from another organization, joining as an external participant or guest | Facilitator is not available in external meetings |
| All three toggles grayed out | The “Allow Copilot and Facilitator” meeting option is set to “Off,” by the organizer, a meeting template, or a sensitivity label | Meeting options > Copilot and other AI |
| All three toggles grayed out | Teams meeting policy blocks Copilot or transcription | Teams Admin Center > Meetings > Meeting policies |
| Only Facilitator grayed out | “Facilitator” app is blocked, for example because all apps are blocked | Teams Admin Center > Teams apps > Manage apps |
| Only Facilitator grayed out or no notes | Loop for meeting notes is disabled | `Get-SPOTenant` shows `IsCollabMeetingNotesFluidEnabled` |

The license is the most common cause. Without a Microsoft 365 Copilot license, the Facilitator toggle remains grayed out in every meeting, regardless of meeting options and policies.

## What Facilitator is suitable for

Facilitator provides the greatest benefit in meetings where someone would otherwise take minutes and therefore cannot fully participate in the discussion. Typical examples:

- **Project and status meetings with a fixed agenda:** Agenda timers and halfway notifications keep the schedule on track, while the notes capture decisions and open items without separate minutes.
- **Workshops with decisions:** The shared Loop page is visible during the meeting. Participants correct wording directly instead of reconciling minutes by email afterward.
- **Questions during the meeting:** Facilitator answers “What did we decide on item 2?” or performs a brief web search about a product version in chat, visible to everyone.
- **Later summary:** On request, Facilitator creates a Word document from the notes and stores it in the requester's OneDrive.
- **In-person meetings:** The Teams app on a smartphone or a Teams Room can also document meetings held on-site.

## What Facilitator cannot do

The limitations are clearly described in Microsoft documentation and primarily concern meeting types, language, and compliance.

**Unsupported meeting types.** Facilitator runs only in scheduled Teams meetings. Instant meetings (“Meet now”), meetings started from a chat, Teams calls, channel meetings, and 1:1 and group chats are not supported. The documentation does not mention support for webinars and town halls.

**No external participants.** AI-generated notes are not available in meetings and chats with external people. External participants generally cannot see the notes. This means Facilitator cannot be used as a minutes tool for meetings with customers or vendors.

**One language per meeting.** The spoken language is selected when turning it on. Notes are generated only for passages in that language. In a meeting that switches between German and French, the French portions will be missing. If the selected language differs from the spoken language, no notes are generated. Swiss German is not listed as a separate language among the supported Copilot languages.

**No recording.** Facilitator turns on transcription, but not recording. Anyone who needs video and audio must start recording separately.

**No agenda from protected invitations.** If the invitation is encrypted with a sensitivity label, Facilitator cannot read the agenda. The same applies while only the organizer is in the meeting, because it does not read the organizer’s copy of the invitation.

**Not a replacement for formal minutes.** The notes are an AI summary of the transcript. Board meetings, personnel discussions, or other meetings with legal implications still require reviewed and approved minutes.

## Data storage and compliance

The notes are stored as a `.loop` file in the “Meetings” folder in the OneDrive of the person who started Facilitator. Microsoft treats them like transcript data. Three limitations are relevant for compliance professionals:

- **eDiscovery:** AI-generated meeting notes are not automatically captured as cloud attachments in Microsoft Purview eDiscovery.
- **Retention:** Retention labels are not supported for cloud attachments in AI-generated notes.
- **Sensitivity labels:** The notes do not inherit the meeting's label. A label can be applied to the Loop component afterward. Settings such as “Prevent copy and paste” and watermarks do not apply to Facilitator answers and notes.

For organizations with retention obligations, this means that notes intended to serve as business records must be moved to storage with functioning retention. Microsoft describes the Purview aspects on a separate page (see Sources).

## Controlling and disabling Facilitator

In the Teams Admin Center, find the app under **Teams apps** > **Manage apps** by searching for “Facilitator.” There, it can be allowed or blocked for the entire tenant. For individual user groups, manage the app through app permission policies or app-centric management.

The Loop meeting notes on which Facilitator is based depend on a SharePoint tenant setting. This applies to all users and cannot be set per person:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Explains options</summary>

| Option | Effect |
|---|---|
| `-Url` | Address of the tenant’s SharePoint Admin Center (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` turns off shared meeting notes and therefore Facilitator’s AI notes for all users; `$true` turns them back on |

</details>

This setting also removes the note feature for meetings without Facilitator. If only the agent should be blocked, blocking the app in the Teams Admin Center is the more targeted approach. For meetings without a saved transcript, set the Copilot and Facilitator option in the meeting policy to “Only during the meeting.”

## Sources

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Admin documentation with licensing requirements, feature status, data storage, compliance limitations, and app management (as of November 18, 2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): End-user guide to meeting options, agenda, timer, supported meeting types, and language selection.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): In-person meetings in Teams Rooms on Windows, organizer licensing, and assignment of speakers in the room.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Microsoft 365 Insider blog from October 2025 on transcription, the basis for notes, and operation without a saved transcript.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Purview perspective on Facilitator data for retention and eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Tenant settings `IsLoopEnabled` and `IsCollabMeetingNotesFluidEnabled` for Loop components and meeting notes.
