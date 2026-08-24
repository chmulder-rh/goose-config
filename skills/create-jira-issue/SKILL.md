---
name: jira-issue-creator
description: Create RHCLOUD Jira issues with validated fields and enriched descriptions via the Atlassian Rovo MCP
---

# Jira Issue Creator

Use this skill to create RHCLOUD Jira issues. Invoke it when the user asks to create a
Jira ticket, story, bug, spike, epic, or any RHCLOUD work item.

## Phase 1: Gather Fields

Parse the invocation text for:
- **summary**: the main phrase or first sentence
- **prefix hint**: `[text]` in brackets → extract as prefix (strip brackets)
- **assignee hint**: "assign to <identifier>", "assign to me", "assign to bot"

### Step 1: Resolve Assignee to Account ID

All assignees must be resolved to a **Jira account ID** before delegation.

- `"me"`           → run `git config user.email`, then call `lookupJiraAccountId` with
                     `cloudId: "redhat.atlassian.net"` and `searchString: <email>`
                     → use the returned account ID; assignee_type = "user"
- `"bot"`          → account ID is `712020:c6b31fa1-eaf5-4921-af5b-cb625f24bb1a` (no lookup needed);
                     assignee_type = "bot"
- contains `"@"`   → call `lookupJiraAccountId` with the email as searchString
                     → use returned account ID; assignee_type = "user"
- other identifier → call `lookupJiraAccountId` with the identifier as searchString
                     → use returned account ID; assignee_type = "user"
- not provided     → assignee_account_id = "unassigned"; assignee_type = "unassigned"

### Step 2: Ask for Team

⚠️ **Ask this question alone. Do not ask any other questions in this message. Wait for the answer.**

Infer the best option from context and present it as the recommendation.

Options:
1. Console - Framework
2. Console - UI

After the user responds, **echo back the full name of the option they selected** (e.g. "✅ Team: Console - Framework") before continuing to the next step.

### Step 3: Ask for Activity Type

⚠️ **Ask this question alone. Do not ask any other questions in this message. Wait for the answer.**

Infer from summary keywords:
- CVE, security, vulnerability, compliance → **Security & Compliance**
- bug, fix, crash, flaky, CI, refactor, lint, test → **Quality / Stability / Reliability**
- incident, hotfix, escalation, production → **Incidents & Support**
- upgrade, migration, architecture, DX, docs → **Future Sustainability**
- training, workshop, learning → **Associate Wellness & Development**
- feature, dashboard, new capability → **Product / Portfolio Work**

Options:
1. Quality / Stability / Reliability
2. Security & Compliance
3. Incidents & Support
4. Future Sustainability
5. Associate Wellness & Development
6. Product / Portfolio Work

After the user responds, **echo back the full name of the option they selected** (e.g. "✅ Activity Type: Future Sustainability") before continuing to the next step.

### Step 4: Ask for Issue Type

⚠️ **Ask this question alone. Do not ask any other questions in this message. Wait for the answer.**

Infer from summary keywords:
- bug, fix, broken, crash → **Bug**
- CVE, vulnerability, exploit → **Vulnerability**
- risk, threat, exposure → **Risk**
- weakness, CWE → **Weakness**
- spike, research, investigate, explore → **Spike**
- epic, initiative, quarter → **Epic**
- default → **Story**

Options:
1. Story
2. Bug
3. Spike
4. Epic
5. Risk
6. Weakness
7. Vulnerability

After the user responds, **echo back the full name of the option they selected** (e.g. "✅ Issue Type: Story") before continuing to the next step.

### Step 5: Ask for Prefix

⚠️ **Ask this question alone. Do not ask any other questions in this message. Wait for the answer.**

Suggest the current working directory basename as the prefix. Present options:
1. `[<cwd basename>]` ← suggested
2. No prefix
3. Other

After the user responds, **echo back the prefix that will be applied** (e.g. "✅ Prefix: [frontend-components]") before continuing to the next step.

### Step 6: Ask for Assignee (if not already resolved in Step 1)

⚠️ **Ask this question alone. Do not ask any other questions in this message. Wait for the answer.**

Options:
1. Unassigned
2. Assign to me
3. Assign to bot
4. Other (enter email or username)

Resolve the answer using `lookupJiraAccountId` as described in Step 1.

After resolving, **echo back the display name** (e.g. "✅ Assignee: Charles Mulder") before continuing.

### Step 7: Bot Label
If assignee_type is "bot", note that the ticket needs the label "hcc-ai-bot".

---

## Phase 2: Create Ticket via Recipe

### Step 8: Delegate to the create-jira-issue Recipe

Once all fields are confirmed, delegate to the **create-jira-issue** recipe with:

```
summary:              <confirmed summary text, WITHOUT prefix>
prefix:               <confirmed prefix, or "" if none>
team:                 <confirmed team name>
issue_type:           <confirmed issue type>
activity_type:        <confirmed activity type>
assignee_account_id:  <resolved Jira account ID, or "unassigned">
```

The recipe validates, maps all fields, calls `createJiraIssue`, and reports the
outcome as plain text.

### Step 9: Handle the Recipe Result

Read the recipe's text output:

- If output contains a ticket key (e.g. `RHCLOUD-1234`) → ticket was created successfully.
  Note the ticket key. Show immediately:
  ```
  Created RHCLOUD-XXXX
  View: https://redhat.atlassian.net/browse/RHCLOUD-XXXX
  ```

- If output contains validation errors → show them to the user, return to Step 2
  to correct the invalid fields, then retry the delegation.

---

## Phase 3: Description and Approval

### Step 10: Generate an Enriched Description

Write a comprehensive description using the summary and any codebase context available:
- **Background**: Technical context and motivation
- **Scope**: Affected files, components, or systems (if determinable)
- **Acceptance Criteria**: Functional requirements as checkboxes, with test coverage
  requirements tailored to the issue type:
  - Bug: unit test for fix, regression test, E2E validation
  - Story/Feature: unit, component, integration, and E2E tests
  - Spike: research questions, deliverables, time-box
  - Vulnerability/CVE: remediation steps, security verification tests
  - Epic/Risk: success criteria and measurable outcomes
- **Additional requirements**: documentation, accessibility, performance (if relevant)

### Step 11: Show Description and Ask for Approval

Display the full proposed description as plain text, then ask:

> Approve this description for RHCLOUD-XXXX?
> - **Approve and update ticket**
> - **Request changes** — describe what to change
> - **Cancel** — skip description update

- Approved → Step 8
- Changes needed → incorporate feedback, repeat Step 7
- Cancelled → stop, show ticket URL only

### Step 12: Update the Ticket Description

Call `editJiraIssue`:
```
cloudId:       "redhat.atlassian.net"
issueIdOrKey:  <ticket key from Step 5>
contentFormat: "markdown"
fields:        { "description": <approved description as markdown> }
```

---

## Phase 4: Confirm

### Step 13: Final Confirmation

```
Created RHCLOUD-XXXX
├── Type:          <issue type>
├── Summary:       <final summary>
├── Team:          <team>
├── Assignee:      <display name or "Unassigned">
├── Activity Type: <activity type>
└── View:          https://redhat.atlassian.net/browse/RHCLOUD-XXXX
```
