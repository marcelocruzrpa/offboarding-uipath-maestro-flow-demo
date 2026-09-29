# Employee Offboarding — UiPath Maestro Flow Showcase (Design)

- **Date:** 2026-09-29
- **Status:** Draft — awaiting review
- **Solution:** `Offboarding_Solution.uipx`

## 1. Purpose

A showcase of an enterprise employee offboarding process for a social/content audience (LinkedIn posts,
short videos, portfolio). The subject of the showcase is **UiPath Maestro Flow** (`.flow`) orchestrating
real SaaS systems, a manager approval, and a timed last-day cutover.

**Success criteria**

- One end-to-end run can be recorded in about 2–3 minutes.
- The run is visible live in three places: the Maestro Flow run view, the Jira issue (comments and
  sub-tasks appearing), and the Entra admin center (account disabled, memberships gone).
- The run is repeatable across takes using the manual reset checklist (section 9).

## 2. Scope

| In scope (v1) | Out of scope (v1) |
|---|---|
| Jira-triggered start, one Flow project | AI agents |
| Entra ID: disable account, remove groups, directory roles, direct licenses | RPA / legacy UI automation |
| Manager approval (Action Center quick-form + email) | OneDrive / mailbox handover |
| Timed cutover with a demo-mode shortcut | Slack / Teams notifications |
| Jira sub-tasks for asset return, reassignment of the leaver's open issues | Password reset, sign-in session revocation |
| Mock HR, asset inventory, and audit log in Data Fabric | Approval timeout / escalation |
| | Automated demo reset |

## 3. Environment

| Item | Value |
|---|---|
| UiPath environment | `staging.uipath.com`, org `<your-org>`, tenant `<your-tenant>` |
| CLI | `uip` 1.202.1 |
| Real systems | Microsoft Entra ID (connector `uipath-microsoft-azureactivedirectory`), Jira (connector `uipath-atlassian-jira`) |
| Mocked systems | Data Fabric (`uipath-uipath-dataservice` / native `core.datafabric.*` nodes) |

All three Integration Service connections exist and are enabled on the tenant. Jira has three enabled
connections; the Flow binds to the one on the Jira site that hosts the `OFFB` project.

## 4. Components

### 4.1 Solution layout

`Offboarding_Solution.uipx` contains one Maestro Flow project, **`EmployeeOffboarding`**, with a single
`.flow` file holding:

- the main Flow
- subflow `RevokeAccess`
- subflow `ITTasksAndHandover`

Subflows are in-file `core.subflow` nodes with isolated variable scope, explicit in/out variables, and an
`error` port.

### 4.2 Data Fabric entities (native, new)

| Entity | Fields | Role |
|---|---|---|
| `OffbEmployee` | EmployeeId, FullName, UPN, Department, EmploymentStatus (`Active` / `Offboarded`) | Mock HR system of record |
| `OffbAsset` | AssetTag, Type, Model, AssignedToUPN, Status (`Assigned` / `Return requested`) | Mock IT asset inventory |
| `OffbAuditLog` | JiraKey, UPN, Step, Outcome (`Success` / `Warning` / `Failed` / `Rejected`), Details, Timestamp | Audit trail, one row per stage |

The `Offb` prefix avoids collisions with existing entities (`AddressBook`, `PoProcessing`, `Vendor`,
`SystemUser`).

### 4.3 Flow configuration variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `demoMode` | bool | `true` | Compress the cutover wait for recording |
| `demoDelayMinutes` | int | `2` | Cutover delay after approval when `demoMode` is true |
| `cutoverUtcOffset` | string | set during demo setup (ISO offset, e.g. `+01:00`) | Timezone for the 18:00 cutover when `demoMode` is false |
| `approverUser` | string | `<approver-email>` | UiPath user who receives the approval task |
| `defaultHandoverEmail` | string | the presenter's Jira account email | Prefill for the handover person on the form |

### 4.4 Jira conventions

- Dedicated project `OFFB` with statuses To Do / In Progress / Done and sub-tasks enabled.
- HR creates an issue with summary `Offboard: <UPN>` and sets the standard **Due date** field to the
  leaver's last working day.
- The Jira issue is the case record: every stage comments on it.

### 4.5 Approver assignment

Action Center tasks can only be assigned to UiPath users, and the leaver's manager in Entra is not one.
The approval form therefore **displays** the Entra manager's name, and the task is **assigned** to
`approverUser`. In the demo, the presenter plays the manager.

## 5. Main Flow

| # | Stage | Nodes | Result |
|---|---|---|---|
| 1 | Trigger | `uipath.connector.trigger.uipath-atlassian-jira.issue-created`, filtered to project `OFFB` | Flow starts on a new offboarding issue |
| 2 | Parse & validate | script | Extracts `upn` (from summary `Offboard: <UPN>`), `dueDate`, `jiraKey`. Invalid → bad-input path (6.1). Valid → Jira comment "Offboarding started", status **In Progress** |
| 3 | Load leaver | Entra `get-user`, `get-manager`, `list-all-groups-of-a-user`, `get-user-roles`; `core.datafabric.read` on `OffbEmployee` and `OffbAsset` by UPN | Leaver profile, manager, current access, assets. User or employee record not found → bad-input path |
| 4 | Compute cutover | script | `cutoverAt` = now + `demoDelayMinutes` if `demoMode`, else `dueDate` at 18:00 in `cutoverUtcOffset` |
| 5 | Manager approval | `uipath.human-in-the-loop.quick-form` (see 5.1) | Approver decision, handover email, optional comment |
| 6 | Decision | decision on chosen outcome | Reject → Jira comment "Cancelled by approver: <comment>", audit `Rejected`, status **Done**, End |
| 7 | Approved | Jira `add-comment`, `core.datafabric.create` | Audit `Success` (approval); comment "Approved — access cutover scheduled for <cutoverAt>" |
| 8 | Wait | `core.logic.delay`, `timerType: timeDate`, `timerDate: cutoverAt` | Long-running wait until cutover |
| 9 | Revoke access | subflow `RevokeAccess` (section 5.2) | Audit row `Success` or `Warning` |
| 10 | IT tasks & handover | subflow `ITTasksAndHandover` (section 5.3) | Audit row `Success` or `Warning` |
| 11 | Close-out | `core.datafabric.update`, `core.datafabric.create`, Jira `add-comment`, `update-issue-status` | `OffbEmployee.EmploymentStatus = Offboarded`; final audit row; summary comment (table of actions, plus a "Needs attention" list when any failures exist); status **Done** if no failures, else stays **In Progress**; End |

### 5.1 Approval form (`quick-form`)

| Part | Content |
|---|---|
| Recipient | Assignee `approverUser`; channels Action Center + Email |
| Inputs (read-only) | Full name, department, Entra manager name, last day, `cutoverAt`, groups to remove, directory roles to remove, assets to return |
| InOuts (editable) | `handoverEmail`, prefilled with `defaultHandoverEmail` |
| Outputs | `comment` (optional) |
| Outcomes | **Approve** (primary), **Reject** — both use action `Continue` so the Flow can comment on Jira before ending; the next decision node branches on the chosen outcome |

### 5.2 Subflow `RevokeAccess`

- **In:** `upn`, `userId`
- **Out:** `disabled`, `removedGroups[]`, `removedRoles[]`, `removedLicenses[]`, `failures[]`

Steps, in security order:

1. `update-user` with `accountEnabled = false`. This step is critical: a failure exits through the
   subflow `error` port (section 6.2).
2. Re-read groups with `list-all-groups-of-a-user` (membership may have changed since approval). Loop
   `remove-member-from-group`, skipping dynamic-membership groups.
3. Loop `remove-member-from-role` over the directory roles.
4. Re-read licenses **after** group removal, since group-based licenses drop automatically. Loop
   `remove-license` over the remaining directly assigned licenses.

A failure in steps 2–4 appends to `failures[]` and the loop continues.

### 5.3 Subflow `ITTasksAndHandover`

- **In:** `jiraKey`, `upn`, `assets[]`, `handoverEmail`
- **Out:** `subtasks[]`, `reassigned[]`, `failures[]`

1. Loop over `assets[]`: `create-issue` as a sub-task of `jiraKey` with summary
   "Return <Type> <AssetTag> (<Model>)", then `core.datafabric.update` `OffbAsset.Status = Return requested`.
2. `find-user-by-email-address-or-display-name` for the handover person and for the leaver.
3. `search-issues-by-jql`: `assignee = <leaverAccountId> AND statusCategory != Done`.
4. Loop over the results: `update-issue-assignee` to the handover person, then `add-comment`
   "Reassigned from <leaver> due to offboarding (<jiraKey>)".

If the leaver or the handover person has no Jira account, reassignment is skipped and recorded in
`failures[]`. Each failed sub-task creation or reassignment is also recorded in `failures[]`.

## 6. Error handling

Every failure path comments on Jira, writes an audit row, and ends normally. There is no Terminate or
faulted path in v1.

### 6.1 Bad input

Triggers: summary does not match `Offboard: <UPN>`, no due date, UPN not found in Entra, or no
`OffbEmployee` record.

Handling: Jira comment "Cannot start: <reason>" → audit `Failed` → End. HR corrects the data and creates
a new issue.

### 6.2 Critical failure

Trigger: a subflow exits through its `error` port (for example, the account cannot be disabled).

Handling: Jira comment "Access revocation failed — manual action required" → audit `Failed` → End. The
issue stays **In Progress**.

### 6.3 Per-item failures

Triggers: entries in either subflow's `failures[]`.

Handling: listed under "Needs attention" in the close-out summary comment; the issue stays
**In Progress**.

## 7. Registry nodes (verified on the tenant, 2026-09-29)

- **Jira:** `uipath.connector.trigger.uipath-atlassian-jira.issue-created`, and actions `create-issue`,
  `add-comment`, `update-issue-status`, `search-issues-by-jql`, `update-issue-assignee`,
  `find-user-by-email-address-or-display-name`.
- **Entra ID:** `get-user`, `get-manager`, `list-all-groups-of-a-user`, `remove-member-from-group`,
  `get-user-roles`, `remove-member-from-role`, `remove-license`, `update-user`.
- **Core:** `uipath.human-in-the-loop.quick-form`, `core.logic.delay`, `core.subflow`, loop, decision,
  script / transform, `core.datafabric.read`, `core.datafabric.create`, `core.datafabric.update`.

Exact input shapes (license SKU IDs, the sub-task parent field, the HITL outcome variable) are resolved at
build time with `uip maestro flow registry get`.

## 8. Demo setup (one-time, manual)

- **Entra ID:** create user `alex.leaver@<your-tenant-domain>` with a manager set, membership in 3 groups,
  1 directory role, and 1 directly assigned license.
- **Jira:** create project `OFFB` (To Do / In Progress / Done, sub-tasks enabled, Due date on the create
  screen). Invite the leaver's email and assign them 2–3 open issues. Make sure the presenter's account
  (the `defaultHandoverEmail`) exists in the same Jira site.
- **Data Fabric:** seed `OffbEmployee` with the leaver plus 2 other employees, and `OffbAsset` with 2
  assets assigned to the leaver.
- **Flow config:** set `cutoverUtcOffset` to the demo timezone and confirm `approverUser` and
  `defaultHandoverEmail`.

## 9. Reset between takes (manual)

1. Re-enable the leaver's Entra account.
2. Re-add the 3 groups, the directory role, and the license.
3. Reassign the leaver's issues back to the leaver.
4. Set `OffbEmployee.EmploymentStatus = Active` and each of the leaver's `OffbAsset.Status = Assigned`.
5. Create a new `OFFB` issue for the next take.

## 10. Testing

- Run `uip maestro flow validate` after each authoring step; zero errors before upload.
- Run `flow debug` scenarios with the presenter present. They change the demo Entra tenant and Jira, and
  the approval task must be completed promptly because debug runs have a caller timeout.

| # | Scenario | Expected result |
|---|---|---|
| 1 | Happy path, `demoMode = true` | Account disabled; groups, role, license removed; 2 asset sub-tasks; leaver's issues reassigned; 4+ audit rows; issue **Done** |
| 2 | Reject | Jira comment and `Rejected` audit row; Entra untouched; issue **Done** |
| 3 | Bad summary | "Cannot start" comment; `Failed` audit row |
| 4 | Unknown UPN | "Cannot start" comment; `Failed` audit row |
| 5 | Partial failure (group-assigned license or dynamic group) | "Needs attention" list in summary; issue **In Progress** |

Evidence per run: Maestro instance trace, Jira comments and sub-tasks, Entra user state, Data Fabric rows.

**Final acceptance:** pack and publish the solution, activate the trigger, create a real `OFFB` issue,
and confirm the full run end-to-end.

## 11. Risks

- The staging environment may be less stable during recording.
- Three enabled Jira connections: binding to the wrong site would make the trigger miss `OFFB` issues.
- A HITL task that nobody opens leaves the instance running indefinitely (no approval timeout in v1).
