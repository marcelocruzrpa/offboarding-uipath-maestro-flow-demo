# Employee Offboarding with UiPath Maestro Flow

An end-to-end employee offboarding process built with **UiPath Maestro Flow**, connecting real SaaS
systems, a manager approval, and a timed last-day cutover.

When HR opens a Jira issue, the flow:

1. looks up the leaver in **Microsoft Entra ID**,
2. asks a manager to approve in **Action Center**,
3. waits until the cutover time,
4. disables the account and removes groups, directory roles, and direct licenses,
5. creates Jira sub-tasks to return IT assets and reassigns the leaver's open issues,
6. records every stage in an audit log and closes the Jira issue.

It was built as a showcase. One run can be recorded in about 2–3 minutes and is visible live in the
Maestro run view, the Jira issue, and the Entra admin center.

## How it works

| Stage | What happens | Systems |
|---|---|---|
| Trigger | A new Jira issue with summary `Offboard: <UPN>` and a due date (the last working day) starts the flow | Jira |
| Parse and validate | Extracts UPN, due date, and issue key. Bad input comments "Cannot start: …" and ends | Jira |
| Load leaver | Entra user, manager, groups, and directory roles; HR record and assigned assets | Entra ID, Data Fabric |
| Plan cutover | `now + demoDelayMinutes` in demo mode, otherwise 18:00 on the due date | — |
| Manager approval | Quick-form showing the leaver, access to be removed, and assets. The approver can change the handover person | Action Center + email |
| Rejected | Comment, audit row, issue closed | Jira, Data Fabric |
| Wait for cutover | Long-running timer until the cutover time | — |
| Revoke access (subflow) | Disable account → remove groups (skipping dynamic groups) → remove directory roles → remove direct licenses. Failures after the disable are collected, not fatal | Entra ID |
| IT tasks and handover (subflow) | One return sub-task per asset, asset marked "Return requested", leaver's open issues reassigned to the handover person | Jira, Data Fabric |
| Close-out | Employee marked `Offboarded`, summary comment with a "Needs attention" list when anything failed. The issue goes to **Done**, or stays **In Progress** when something needs attention | Jira, Data Fabric |

Every failure path comments on the Jira issue and writes an audit row, so the Jira issue is the case record.

### Data Fabric entities (mock systems)

| Entity | Purpose | Fields |
|---|---|---|
| `OffbEmployee` | HR system of record | EmployeeId, FullName, UPN, Department, EmploymentStatus |
| `OffbAsset` | IT asset inventory | AssetTag, Type, Model, AssignedToUPN, Status |
| `OffbAuditLog` | Audit trail, one row per stage | JiraKey, UPN, Stage, Outcome, Details, LoggedAt |

All fields are text.

### Flow variables

| Variable | Default | Purpose |
|---|---|---|
| `demoMode` | `true` | Shorten the cutover wait for demos |
| `demoDelayMinutes` | `2` | Minutes between approval and cutover in demo mode |
| `cutoverUtcOffset` | `+01:00` | Timezone for the 18:00 cutover when `demoMode` is `false` |
| `defaultHandoverEmail` | `handover@example.com` | Prefilled person who takes over the leaver's Jira issues |

The approver is set on the **Manager approval** node (placeholder `approver@example.com`).

## Repository layout

```text
Offboarding_Solution.uipx          UiPath solution manifest
EmployeeOffboarding/               Maestro Flow project (EmployeeOffboarding.flow, bindings, evals)
resources/solution_folder/         solution resources (connections, package, process)
docs/superpowers/specs/            design spec
AGENTS.md                          briefing for AI coding agents
```

## Use it in your own tenant

**Prerequisites**

- A UiPath Automation Cloud tenant with Maestro, Action Center, Integration Service, and Data Fabric
- The [`uip` CLI](https://docs.uipath.com/) (`uip login`), or UiPath Studio / Studio Web
- A Microsoft Entra ID tenant where you can create a test user
- A Jira Cloud site with a project that has sub-tasks enabled

**Setup**

1. Create Integration Service connections for **Microsoft Entra ID** and **Jira**, and bind them to the
   solution's two connection resources.
2. Create the three Data Fabric entities above.
3. Point the Jira trigger, the return sub-task node, and the three status nodes at your Jira project. The
   status nodes need your project's transition ids. See the Configuration points table in [AGENTS.md](AGENTS.md).
4. Set the approver on the **Manager approval** node and the `defaultHandoverEmail` variable to real
   users in your tenant.
5. Validate and pack:

   ```bash
   uip maestro flow validate EmployeeOffboarding
   uip solution pack . --dry-run
   ```

6. Upload to Studio Web (`uip solution upload .`) to debug, or pack, publish, and deploy to run it for real.

**Demo data (one-time)**

- **Entra ID:** a test leaver with a manager, 3 group memberships, 1 directory role, and 1 directly assigned license.
- **Jira:** the leaver invited to the site with 2–3 open issues assigned. The handover person must also have an account.
- **Data Fabric:** an `OffbEmployee` row for the leaver (plus a couple of others) and 2 `OffbAsset` rows assigned to them.

**Reset between takes**

1. Re-enable the leaver's Entra account and re-add the groups, directory role, and license.
2. Reassign the leaver's issues back to them.
3. Set `OffbEmployee.EmploymentStatus = Active` and each of the leaver's `OffbAsset.Status = Assigned`.
4. Create a new offboarding issue.

## Known limitations

- There is no approval timeout or escalation. An unopened approval task keeps the instance waiting.
- Reset between demo runs is manual.
- Out of scope: password reset, session revocation, mailbox or OneDrive handover, and chat notifications.
- The Jira trigger is currently bound to a scratch project. Rebind it to your own project as described above.

## Design

The full design is in [the spec](docs/superpowers/specs/2026-09-29-employee-offboarding-maestro-flow-design.md): components, error handling, and test scenarios.
