---
name: gcts-workflow
description: "Full gCTS (git-enabled Change and Transport System) repository lifecycle via sapcli — clone a repo onto the system, push/pull/commit/checkout branches, inspect objects and activity, manage tasks, and configure system-level gCTS settings. Load when the user needs to deploy ABAP code via gCTS push, switch branches, set up a new gCTS repository, or investigate a gCTS sync or lock problem on a gCTS-enabled S/4HANA or BTP ABAP Cloud system."
---

## What I do

I cover the full gCTS workflow from the command line using `sapcli gcts`.
gCTS-managed systems — BTP ABAP Cloud (Steampunk), SAP S/4HANA on-premise/PCE
with gCTS enabled — use git as the version-control backbone for ABAP packages.
Instead of releasing CTS transports to deploy code, you `push` the package;
instead of importing transports, you `pull`. The CTS transport still exists for
release management and audit, but gCTS handles the actual code movement.

I do **not** cover classic-transport-only systems (on-premise ERP without gCTS
enabled). For classic transport routing and lock management use classic-transport
skills.

---

## Scope check first

Before running any gCTS command, confirm the connected system uses gCTS:

```bash
sapcli gcts repolist           # if this returns an empty list or raises an
                               # HTTP 404, the system is not gCTS-managed
```

On a non-gCTS system `repolist` will fail with a 404 — do not proceed with
gCTS commands.

---

## Common workflows

### Deploy: push code to the system

The most common operation. After activating objects locally, push the package
to the remote gCTS repository so other systems / branches can pull it:

```bash
sapcli gcts push PACKAGE_NAME
# optional: --heartbeat N  (print a dot every N seconds while waiting)
```

**Important gotcha — `--corrnr` silent override**: if you also pass
`--corrnr TRANSPORT` and that transport does **not** currently hold the lock on
the package, sapcli exits 0 but the change silently routes to whichever
transport does hold the lock. Always verify with `repo objects` after a push
that uses `--corrnr`.

### Clone a repository onto the system

To seed a new gCTS repository from a remote git URL:

```bash
sapcli gcts clone \
  [--wait-for-ready SECONDS] \
  [--vsid VSID] \
  [--role SOURCE|TARGET] \
  [--type GITHUB|GITLAB|...] \
  [--vcs-token TOKEN] \
  [--starting-folder FOLDER] \
  [--sync-clone] \
  [--poll-period SECONDS] \
  [--no-import] \
  REMOTE_URL \
  [PACKAGE_NAME]
```

`--no-import` clones without importing objects (useful for inspection first).
`--sync-clone` waits for the clone activity to complete before returning.

### Commit: create a git commit from open CTS changes

After editing objects in a transport task, commit them into the gCTS git history:

```bash
sapcli gcts commit [-m "Message"] [--description "Body"] [--devc DEVC] [--heartbeat N] PACKAGE [CORRNR]
```

`CORRNR` is optional — omit to commit all open changes in the package. Passing
a specific `CORRNR` commits only the objects locked by that transport task.
`--devc DEVC` limits the commit to objects in a specific development class.

### Pull: import remote changes into the system

```bash
sapcli gcts pull PACKAGE
```

Fetches the latest commits from the remote and imports the changed objects into
the system.

### Checkout: switch branch

```bash
sapcli gcts checkout \
  [--wait-for-ready SECONDS] \
  [--no-import] \
  PACKAGE BRANCH_NAME
```

`--no-import` switches the branch pointer without importing objects — useful
for inspection. Without it, sapcli imports the branch's objects after the
switch.

### View history

```bash
sapcli gcts log PACKAGE
```

---

## Repository inspection

```bash
# Objects configured in the gCTS repository (what it knows about)
sapcli gcts repo objects PACKAGE [--format JSON|TRANSPORT] [--noheadings] [--columns COLUMNS]

# Repository layout (folder structure / root folder)
sapcli gcts repo layout PACKAGE

# Pending tasks (uncommitted changes)
# IMPORTANT: always pass --format JSON — the default table renderer crashes with
# TypeError when the task set is empty (sapcli v1.0.0 defect in TableWriter)
sapcli gcts repo tasks PACKAGE [--tid TASK_ID] --format JSON

# Activity log (commits, pulls, branch switches)
sapcli gcts repo activities PACKAGE \
  [--operation COMMIT|PULL|CLONE|BRANCH_SW] \
  [--limit N] [--offset N] \
  [--fromcommit SHA] [--tocommit SHA]

# Messages from the last activity
sapcli gcts repo messages PACKAGE [--process PROCESS_ID]
```

---

## Branch management

```bash
# List branches
sapcli gcts repo branch list PACKAGE [-a|--all] [-r|--remote] [--format JSON]

# Create a branch
sapcli gcts repo branch create PACKAGE BRANCH_NAME [--local-only]

# Delete a branch
sapcli gcts repo branch delete PACKAGE BRANCH_NAME
```

---

## Configuration

```bash
# Read all config keys for a package
sapcli gcts config --list PACKAGE

# Read one key
sapcli gcts config PACKAGE KEY_NAME

# Set a key
sapcli gcts config PACKAGE KEY_NAME VALUE

# Unset a key
sapcli gcts config --unset PACKAGE KEY_NAME

# Change the remote URL
sapcli gcts repo set-url PACKAGE NEW_URL

# Read arbitrary repo properties (JSON object with all fields)
sapcli gcts repo property get PACKAGE
sapcli gcts repo property set PACKAGE PROPERTY_NAME VALUE
```

---

## Task management

gCTS "tasks" represent uncommitted changes (roughly equivalent to CTS transport
tasks). To inspect and clean them up:

```bash
# Note: sapcli gcts repo tasks uses --format JSON to avoid a TableWriter
# crash on empty task sets. The sapcli gcts task subcommand is separate
# and only meaningful when you have a task ID from repo tasks output.
sapcli gcts task list PACKAGE
sapcli gcts task info PACKAGE --tid TASK_ID
sapcli gcts task delete PACKAGE --tid TASK_ID
```

---

## System-level gCTS config

These operate on the system's global gCTS settings, not on a specific package:

```bash
sapcli gcts system config list
sapcli gcts system config get KEY
sapcli gcts system config set KEY VALUE
sapcli gcts system config unset KEY
```

---

## User credentials

```bash
sapcli gcts user get-credentials [--endpoint URL]
sapcli gcts user set-credentials --api-url URL --token TOKEN
sapcli gcts user delete-credentials --api-url URL
```

---

## Delete a repository

```bash
sapcli gcts delete PACKAGE
```

Removes the gCTS repository configuration from the system. Does not delete
remote git content.

---

## What I don't do

- Release CTS transports — that is still a human step via SAP GUI or `sapcli cts release`.
- Cover classic-CTS scenarios — use classic-transport skills for those.
- Manage the git remote itself (branch protection, PR creation) — use the git server's own UI/API for that.
