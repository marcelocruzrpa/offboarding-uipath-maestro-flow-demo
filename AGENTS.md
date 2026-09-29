# AGENTS.md — Employee Offboarding (UiPath Maestro Flow)

Briefing for AI coding agents working in this repository. Humans: start with [README.md](README.md).

## What this is

A UiPath solution that automates employee offboarding with **Maestro Flow** (`.flow`). **It is not BPMN.**
A new Jira issue starts the flow. The flow looks up the leaver in Microsoft Entra ID, asks a manager to
approve, waits until the cutover time, revokes access, creates IT return tasks, reassigns the leaver's
open Jira issues, and writes an audit trail.

- **Real systems** (via Integration Service connectors): Microsoft Entra ID (`uipath-microsoft-azureactivedirectory`), Jira (`uipath-atlassian-jira`).
- **Mocked systems** (Data Fabric entities): `OffbEmployee` (HR record), `OffbAsset` (IT asset inventory), `OffbAuditLog` (audit trail).
- **Human step:** Action Center quick-form approval (`uipath.human-in-the-loop.quick-form`).
- **Not used:** AI agents, RPA processes.

The design rationale is in [docs/superpowers/specs/](docs/superpowers/specs/). Where the spec and the `.flow` disagree, the `.flow` is authoritative.

## Layout

```text
Offboarding_Solution.uipx            solution manifest (never hand-edit)
EmployeeOffboarding/
  EmployeeOffboarding.flow           the flow: main graph + 2 in-file subflows
  bindings_v2.json                   connection + trigger bindings
  project.uiproj / operate.json      project metadata
  evals/                             Studio Web evaluation scaffolding
resources/solution_folder/           solution resources (connections, package, process)
docs/superpowers/specs/              design spec
.local/, userProfile/                per-user Studio/debug state (git-ignored)
```

Flow shape at the time of writing: **45 top-level nodes**, subflow `revokeAccess` **24** nodes, subflow
`itTasksAndHandover` **18** nodes, **5** entries in `bindings[]`. Update these numbers whenever you
intentionally add or remove nodes.

## Skills to load

| Work | Skill |
|---|---|
| Anything touching `EmployeeOffboarding.flow` | `uipath-maestro-flow` (never `uipath-maestro-bpmn`) |
| `uip solution` pack / upload / publish / deploy, resources | `uipath-solution`, `uipath-platform` |
| Data Fabric entities and records (`uip df`) | `uipath-platform` |
| A run failed or misbehaves | `uipath-troubleshoot` |

## Hard rules

1. **Close the flow in UiPath Studio desktop before any CLI or file edit.** Studio auto-saves the open
   `.flow` and silently overwrites external edits. It has dropped CLI-added nodes and the whole
   `bindings` array before. If `UiPath.Studio` is running, ask the user to close the flow tab first.
2. **After every `node add` / `node configure` batch, re-check the node counts and the `bindings[]`
   length.** `uip maestro flow validate` does not detect lost nodes.
3. **Connector nodes that live inside a subflow must be added and configured at the top level first,
   then moved into the subflow.** The CLI cannot find nodes inside subflows.
4. **Respect node ownership.** Connector activities and triggers are CLI-owned (`node add` / `node configure`).
   Scripts, decisions, loops, HITL, Data Fabric, and end nodes are edited directly. Never full-file rewrite
   the `.flow`.
5. **Never hand-edit the `.uipx`.** After changing bindings, run `uip solution resources refresh`. Two
   resource files for the same connection key (for example, one name with spaces and one with
   underscores) make refresh fail with "Node already added to the graph". Delete the duplicate.
6. **Data Fabric reserved words.** The audit fields are `Stage` and `LoggedAt` (`Step` and `Timestamp`
   are reserved). All `Offb*` entity fields are STRING.
7. **`uip maestro flow debug` has real side effects.** It disables Entra accounts, edits Jira, and
   re-uploads the solution to Studio Web. Run it only with the user present, and complete the approval task
   promptly because debug runs have a caller timeout. Use `validate` for checks, not `debug`.
8. **Never redirect or drop stderr from `uip`.** Errors and confirmations go there.
9. **This repository is public.** Never commit personal or tenant-specific data: emails, org or tenant
   names, connection names, Jira site URLs, or tokens. The committed flow uses `example.com` placeholders.
   Keep real values in your tenant or in uncommitted local changes. `.local/` and `userProfile/` are
   git-ignored for this reason.

## Configuration points

| What | Where |
|---|---|
| Approver (UiPath user who gets the Action Center task) | `managerApproval` quick-form node → `inputs.recipient.assignee.value` (placeholder `approver@example.com`) |
| Default handover person | flow global `defaultHandoverEmail` (placeholder `handover@example.com`) |
| Demo timing | flow globals `demoMode` (default `true`), `demoDelayMinutes` (default `2`) |
| Real cutover timezone | flow global `cutoverUtcOffset` (ISO offset, used when `demoMode` is `false`) |
| Jira project | trigger `offboardingIssueCreated1` (project and issue type), `createReturnSubtask1` (project key and sub-task type), and the 3 status nodes `setStatusInProgress1`, `closeCancelledIssue1`, `setStatusDone1` (transition ids) |
| Connections | `bindings_v2.json` + `resources/solution_folder/connection/` |

## Current state and known gaps

- The flow validates and packs.
- The Jira trigger is temporarily bound to a scratch project, not the dedicated `OFFB` project from the
  spec. Switching projects means reconfiguring every node listed under "Jira project" above.
- Demo data (a test leaver in Entra ID, seed rows in `OffbEmployee` / `OffbAsset`) is not set up yet.
- The approver is a static email in the quick-form node. The spec's `approverUser` variable was not implemented.
- There is no approval timeout. An unopened approval task leaves the instance running.

## Common commands

Run these from the solution root. Add `--output json` when you parse results.

```bash
uip login status                                             # confirm tenant before anything remote
uip maestro flow validate EmployeeOffboarding                # after every edit; zero errors and warnings
uip maestro flow registry get <node-type>                    # exact input/output shape of a node type
uip maestro flow node configure ...                          # CLI-owned connector nodes only
uip solution resources refresh                               # after binding changes
uip solution pack . --dry-run                                # packaging gate, no artifact
uip solution upload .                                        # push to Studio Web (ask first)
uip maestro flow debug EmployeeOffboarding                   # real end-to-end run (ask first, see rule 7)
```

`uip <group> --help` is the source of truth when a command here is out of date. Fix this file in place when you find one.

## Git

- Commit and push only when the user asks.
- Never commit `.local/`, `userProfile/`, pack output (`*.nupkg`, `*.zip`, `*.uis`, `out/`), or real personal or tenant values.
- Before any commit, grep the staged tree for emails and tenant names.
