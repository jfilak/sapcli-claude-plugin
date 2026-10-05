---
name: badi-inspect
description: "Inspect and configure BAdI implementations on a connected SAP system via `sapcli badi` — list all implementations registered under an enhancement spot, check their active state, and activate or deactivate individual BAdIs. Load when the user needs to verify system configuration for a BAdI enhancement spot, check whether a specific BAdI implementation is active, or toggle activation state programmatically."
---

## What I do

I cover operational BAdI configuration using `sapcli badi`:

- **List** all BAdI implementations registered under a given Enhancement
  Implementation object (ENHO/ENHSPOT), with their active state, implementing
  class, and definition name.
- **Activate / deactivate** individual BAdI implementations (`set-active`).

I am **distinct from the `abap-badi-contract-test` skill** — that skill tests whether a
BAdI still triggers correctly and fulfills its behavioral contract. I focus on
the configuration plane: which implementations exist, what state they are in,
and how to toggle them.

I am also distinct from the `abap-enhancement-inventory` skill — that skill inventories
the enhancement *surface* an object or package offers (BAdIs, enhancement
spots, user exits). I work with a known enhancement spot and tell you what's
currently configured inside it.

---

## List BAdI implementations

```bash
sapcli badi -i ENHSPOT_NAME list
```

Output is space-separated rows, headerless by default. Columns (fixed order):
`name`, `active`, `implementing_class.name`, `badi_definition.name`,
`customizing_lock`, `default`, `example`, `short_text`.

No `--columns` or `--noheadings` flag exists in this sapcli build — the output is
always the full row set in the fixed order above. Filter with standard shell tools:

```bash
# Find active implementations
sapcli badi -i ENHSPOT_NAME list | awk '$2 == "true"'

# Check one implementation by name
sapcli badi -i ENHSPOT_NAME list | grep IMPL_NAME
```

**Example — Configuration check for an enhancement spot:**

```bash
sapcli badi -i Z_MY_ENHSPOT list
```

The ENHO name is the enhancement implementation object name as it appears in
transaction `SE19` / TADIR type `ENHO`. It is NOT the same as the BAdI
definition name.

---

## Activate or deactivate a BAdI

```bash
# Set state only (no ADT activation run)
sapcli badi -i ENHSPOT_NAME set-active -b BADI_IMPL_NAME false
sapcli badi -i ENHSPOT_NAME set-active -b BADI_IMPL_NAME true

# Set state + run ADT activation in one step (recommended)
sapcli badi -i ENHSPOT_NAME set-active -b BADI_IMPL_NAME true -a
```

`{true,false}` is a **positional** argument (the new active state).
`-a` / `--activate` is a **boolean flag** — it triggers the ADT activation of
the ENHO object after changing the active flag. Do not confuse it with the state
value; `-a` takes no value.

**Transport required for transported objects.** If the ENHO is under transport
control, `set-active` will fail with `ExceptionParameterNotFound: Parameter
corrNr could not be found` unless you pass `--corrnr`:

```bash
sapcli badi -i ENHSPOT_NAME set-active -b BADI_IMPL_NAME true --corrnr TRANSPORT_NUMBER
sapcli badi -i ENHSPOT_NAME set-active -b BADI_IMPL_NAME true -a --corrnr TRANSPORT_NUMBER
```

Use a transport management skill to find or create a suitable modifiable
transport for the ENHO's package before running `set-active`.

**BADI_IMPL_NAME** is the name of the individual BAdI implementation within the
enhancement implementation, not the ENHO object name itself. Use
`badi … list` first to get the correct implementation names.

---

## Typical configuration flow

1. **Identify the enhancement spot:** get the ENHO name from the project's
   BAdI configuration spec or from transaction `SE19`.

2. **List current state:**
   ```bash
   sapcli badi -i <ENHO> list
   # grep for the implementation of interest:
   sapcli badi -i <ENHO> list | grep <IMPL_NAME>
   ```

3. **Pick a transport** for the ENHO's package if it is under transport control.
   You need the transport number for the next step.

4. **Activate the required implementation:**
   ```bash
   # Without transport (local/non-transportable objects only):
   sapcli badi -i <ENHO> set-active -b <IMPL_NAME> true -a

   # With transport:
   sapcli badi -i <ENHO> set-active -b <IMPL_NAME> true -a --corrnr <TRANSPORT>
   ```

5. **Verify:**
   ```bash
   sapcli badi -i <ENHO> list | grep <IMPL_NAME>
   # column 2 should show "true"
   ```

6. **Run a contract test** to confirm the BAdI still triggers correctly:
   Consult the `abap-badi-contract-test` skill.

---

## What I don't do

- I don't create or delete ENHO objects — those are created via ADT or
  transaction `SE19`.
- I don't author the implementing class body — see the `abap-classic-developer` skill.
- I don't test whether the BAdI triggers correctly at runtime — see
  the `abap-badi-contract-test` skill.
