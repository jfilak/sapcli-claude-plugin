---
name: abap-dump-analyse
description: Read and analyse ABAP short dumps (transaction ST22 runtime errors) on S/4HANA Public Cloud / ABAP Cloud without the SAP GUI — list recent dumps, fetch a single dump's full text, and map the runtime error to its cause and the relevant fix rule. Primary path is native `sapcli abap shortdumps` (requires sapcli pulled from jfilak/sapcli upstream, post-v1.0.0).
---

## What I do

I read ABAP short dumps — the runtime errors you would otherwise inspect in
transaction **ST22** — straight from the command line, and analyse them:

- **List** recent dumps (timestamp, runtime error, terminated program).
- **Show** one dump's full ST22 text (What happened, Error analysis, How to
  correct, source extract, call stack, selected variables).
- **Map** the runtime error to its cause and, where one exists, the rule that
  explains and prevents it (e.g. `BEHAVIOR_CONTRACT_VIOLATION CC/C:MISSING_CID`
  → `rap-eml-create-requires-cid`).

This closes the loop after a failed RAP/CDS build or a test run.

## Access path — native sapcli

> **Version note:** `sapcli abap shortdumps` and `sapcli abap feeds` were added
> after v1.0.0. If `sapcli abap --help` does not show them, pull the upstream:
> `cd <your-sapcli-clone> && git pull upstream master && pip install -e .`

```bash
# List recent short dumps (Author | Title | Updated | Id)
sapcli abap shortdumps list

# Show full ST22 text of a dump by its ID (from the list output)
sapcli abap shortdumps show DUMP_ID
```

## Usage

### Producing a dump deliberately (for testing)

`sapcli abap run` is the zero-artifact path — no transport, no permanent class:

```bash
# Write a snippet to a temp file and run it
cat > /tmp/snippet.abap <<'ABAP'
TRY.
    DATA(lt) = VALUE string_table( ( `x` ) ).
    DATA(lv) = lt[ 2 ].                        " ITAB_LINE_NOT_FOUND
  CATCH cx_sy_itab_line_not_found INTO DATA(lx).
    RAISE EXCEPTION TYPE cx_sy_itab_line_not_found  " concrete subclass of CX_NO_CHECK
      EXPORTING previous = lx.                  " → "Chain of Exception Objects" in dump
ENDTRY.
ABAP
sapcli abap run /tmp/snippet.abap    # exits non-zero (HTTP 500) — that IS the dump
```

sapcli wraps the snippet in a temporary `IF_OO_ADT_CLASSRUN` runner
(`ZCL_SAPCLI_RUN_DEVELOPER_*`), executes it over ADT HTTP, and cleans up the
runner automatically. The 500 response means the program dumped — find the dump
immediately with `sapcli abap shortdumps list`.

> **Important:** `CX_NO_CHECK` is **abstract** — it cannot be instantiated
> directly (`MESSAGE(G?5): Instances of the abstract class cannot be generated`).
> Use a concrete subclass such as `CX_SY_ITAB_LINE_NOT_FOUND` or a custom
> `ZCX_*` as the outer wrapper in a chained raise.

```bash
# List recent short dumps
sapcli abap shortdumps list

# Show full ST22 text of a dump by its ID (from the list output)
sapcli abap shortdumps show DUMP_ID
```

## Analysis workflow

1. **`list`** — scan recent dumps. Group by runtime error; the interesting ones
   are usually the `RAISE_*` / `*_CONTRACT_VIOLATION` / `RAISE_EXCEPTION` from
   your own `Z*`/`ZCL_SAPCLI_RUN_*` programs, not the periodic `SYSTEM_*` /
   `SUSAGE_*` housekeeping dumps.

   **Triage before deep-diving into a single dump:**
   - *Scope:* one user or many? one document/key or any input?
   - *Context:* dialog transaction, OData request, background job, update task,
     or ADT classrun (`sapcli abap run` / ABAP Unit over HTTP — the dominant
     context for Z* dev dumps)?
   - *Recurrence:* first occurrence or repeating pattern?

   These three questions decide the investigation approach: a single-user dialog
   dump often points to missing data or an auth config; a multi-user repeating
   dump usually points to code or a schema change.

2. **`show <DUMP_ID>`** — read the dump in this order:
   - **Short Text** — runtime error name + brief description.
   - **Error analysis** — the *sub-reason* (not the name alone). This is where
     `CC/C:MISSING_CID` vs. an auth violation are distinguished for the same
     `BEHAVIOR_CONTRACT_VIOLATION` name. **Exception:** for classic
     exception-condition RAISEs (e.g. `TYPE_NOT_FOUND`, `ITAB_LINE_NOT_FOUND`),
     the sub-reason often appears only in **Selected Variables** — Error-analysis
     may show generic boilerplate. Check both sections.
   - **Active Calls/Events** — the call stack. Identify the **detection point**
     (where ABAP stopped) and the **introduction point** (where invalid state
     first entered the flow — usually the call just above the first custom-code
     frame in the stack). These are often different frames.
   - **Source Code Extract** — the statement marked `>>>>>` is the detection
     point. Do not assume it is also where the fix belongs.
   - **Selected Variables** — the actual runtime values at termination. For
     classic exception-condition RAISEs (e.g. `TYPE_NOT_FOUND`,
     `ITAB_LINE_NOT_FOUND`) the sub-reason often lives here, not in
     Error-analysis.
   - **Chain of Exception Objects** (SAP releases ≥ 7.4x title it this; some older
     docs call it "Previous Exceptions") — check this section when the dump
     involves an exception chain. It is only present when a chain exists; absence
     means the top exception is the root cause. The chain narrative also appears
     in the formatted `show` output as *"The exception is related to the previous
     exception '…' that occurred in …"*. In the raw ADT XML this renders as a
     chapter entry titled `Chain of Exception Objects` — searching for the literal
     string "Previous Exceptions" in the XML will produce a false negative.
   - **Contents of System Fields** — SY-* field values captured at the exact
     moment of termination. Useful when the dump itself is generic: `SY-SUBRC`
     (last return code), `SY-TABIX` (last internal-table index), `SY-DBCNT`
     (rows affected by last DB operation) often explain a READ TABLE /
     SELECT failure that doesn't appear in Error analysis.

3. **Map to a cause + rule.** Translate the runtime error into the fix. Common
   ones on RAP/CDS work:

   | Runtime error (Short Text) | Cause | Rule |
   |---|---|---|
   | `BEHAVIOR_CONTRACT_VIOLATION` — `CC/C:MISSING_CID` | EML `CREATE` row without `%cid` under `strict(2)` | `rap-eml-create-requires-cid` |
   | `BEHAVIOR_CONTRACT_VIOLATION` — auth-related | forced auth handler left empty (must GRANT, not just exist) | `auth-master-global-and-instance` |
   | `BEHAVIOR_READONLY_FIELD` | EML `MODIFY … UPDATE` set a BDEF read-only field (managed numbering, `field ( readonly )`, or a key) — the field must not be in the `FIELDS ( … )` clause | remove the read-only field from the EML update |
   | `RAISE_EXCEPTION` / `CX_SADL_*` | CDS/SADL model error — bad projection, unreleased base, mapping mismatch | check CDS view authoring / RAP CDS extension rules |
   | `RAISE_SHORTDUMP` — `CX_SADL_DUMP_APPL_MODEL_ERROR` | application-modelling error surfaced through SADL (see the exception chain in the dump) | inspect the chained exception; usually a CDS/service-definition fault |
   | `RAISE_EXCEPTION` with `CX_SY_*` wrapping a lower error | may indicate a chained exception — but `CX_SY_*` classes also surface as `UNCAUGHT_EXCEPTION` without chaining; check the **"Chain of Exception Objects"** section in the dump (SAP releases ≥7.4x); if present it carries the root cause (DB error, conversion, auth); if absent, the `CX_SY_*` itself is the root | check the chain section only if present — do not assume chaining from the error name alone; the raw ADT XML titles it "Chain of Exception Objects", not "Previous Exceptions" |
   | `OBJECTS_OBJREF_NOT_ASSIGNED` / `ITAB_LINE_NOT_FOUND` | reference or table row assumed to exist; check caller contract and whether a lookup earlier in the stack failed silently | trace to introduction point in call stack — likely a missing initialization or a `READ TABLE` without error handling |
   | `TSV_TNEW_PAGE_ALLOC_FAILED` | memory exhaustion — internal table unbounded, result set too large | look for missing `WHERE` clause or implicit SELECT * — this is a resource, not a logic bug |

   For runtime errors not in the table, read the **Error analysis** + **How to
   correct** sections verbatim — they carry SAP's own diagnosis; do not guess
   from the error name alone.

4. **Report** the finding: runtime error, terminated program, the one-line cause,
   and the rule to apply. If the dump reveals a rule gap (a cause no rule covers
   yet), that is a candidate for a new rule — surface it.

## Read the dump, don't guess from the name

The runtime error **name** (`BEHAVIOR_CONTRACT_VIOLATION`) is not the diagnosis —
the **sub-reason** in Error analysis is (`CC/C:MISSING_CID` vs an auth violation
are the same runtime error, different fixes). Always `show` the dump and read the
Error-analysis / How-to-correct text before naming a cause. Do not answer from
the list line alone.

## Call stack: detection vs. introduction point

The highlighted source line is where ABAP *detected* the failure and stopped —
not necessarily where the fix belongs.

**Scan the stack top-down looking for:**
- The first custom-code frame (`Z*`, `ZCL_*`, BAdI implementation, enhancement
  implementation) — this is the most likely *introduction point*.
- SAP framework frames surrounding custom code — the framework correctly detected
  an invalid state that custom code created.
- Update-task or background-job frames — the actual error may have originated in
  the dialog step that queued the update; the dump fires later (asynchronously)
  in the update work process.

**Common misleading patterns:**
- A RAP framework or OData handler frame dumps while the actual cause is a BAdI
  or enhancement earlier in the stack.
- An exception is caught and re-raised with `PREVIOUS =` — the dump shows
  **"Chain of Exception Objects"** (raw ADT XML chapter title; some docs call it
  "Previous Exceptions"). Two distinct object IDs in Selected Variables
  (e.g. `{O:114*\CLASS=CX_SY_...}` and `{O:115*\CLASS=...}`) confirm the
  two-link chain. Note: `CX_NO_CHECK` is **abstract** — a concrete subclass
  must be used as the outer wrapper (`CX_SY_*` subclass or a custom `ZCX_*`).
- A defensive `ASSERT` or `MESSAGE TYPE X` in standard code fires because custom
  code supplied invalid input — the standard frame is the detection, not the cause.
- The dump fires in an *update task* (`/1BCDWB/SM_*`, `SAPLSMW*` frames visible
  in the stack) — the defect was in the dialog application that scheduled the
  update, not in the update module itself.

> **Fix the introduction point, not just the detection point.**

## Message looks truncated? Resolve it — don't improvise

When an error text arrives clipped (a `…`, a cut-off sentence, or a bare message
ID like `/DMO/CM_FLIGHT_LEGAC 016`), the full text is **retrievable
deterministically**. Do **not** "get creative" hunting for it — that is wasted
effort. There is always a direct read:

| What you have | Get the full text with |
|---|---|
| A dump whose console render looks clipped | `sapcli abap shortdumps show DUMP_ID` — prints the full ST22 text; the list line is a summary, never the diagnosis. |
| A message shown only as `MSGID NNN` / `MSGCLASS/NNN` | `sapcli messageclass message read <MSGCLASS> <NNN>` returns the full T100 text (the `&1`–`&4` template). See the `abap-message-class` skill (if installed). |
| A message text with `&1`–`&4` placeholders you can't fill | The dump's **selected-variables** section (in `sapcli abap shortdumps show`) holds the actual substitution values; read them there rather than inventing plausible ones. |

> **Anti-pattern: "the message is truncated, so I'll search for the complete
> one."** The message is not lost — it's one read away (`sapcli abap shortdumps show DUMP_ID`, or
> `messageclass message read` for an ID-only reference). Improvising a search, or
> reconstructing the text from the message name, burns time and risks a
> confidently-wrong diagnosis. Resolve the full text first, *then* analyse.

## Scope and safety

- **Read-only.** `sapcli abap shortdumps` issues only HTTP GETs. It cannot modify the system.
- **Credentials** come from `~/.sapcli/config.yml`. Nothing new is stored; the password is read at call time and never printed.
- **Public Cloud reality:** dump *content* is available through the ADT feed,
  but there is **no released ABAP API** to read dumps from within ABAP code — so
  this stays an operational/diagnostic tool at the CLI layer, not something a
  cloud app can call.

## SAP documentation

- [How to find a solution for short dumps from ST22](https://help.sap.com/docs/SUPPORT_CONTENT/wdabap/3362186674.html) — SAP's own methodology: start with Short Text + Error Analysis + Source Code Extract (`>>>>>` marker), then call stack + variables
- [ABAP Dump Analysis (ST22)](https://help.sap.com/docs/SAP_NETWEAVER_702/fe1a4b276c551014b24d80fe2b500e38/b134ab1cd8e44562b0fee9524c638cca.html) — official ST22 reference
- [Runtime Errors — ABAP Keyword Documentation](https://help.sap.com/doc/abapdocu_816_index_htm/8.16/en-US/ABENNONCAT_EXCEPTIONS.html)

## See also

- `abap-feeds-mcp` skill (if installed) — the sibling for *every other* ABAP feed (SAP Gateway error log, ABAP system messages, contract-check violations) over the ADT MCP transport. Dumps → this skill (via `sapcli abap shortdumps`); any non-dump feed → `abap-feeds-mcp`.
