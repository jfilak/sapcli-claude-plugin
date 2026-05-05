---
name: abap-sql-query
description: Execute ABAP SQL (OpenSQL) queries against a connected SAP system using the sapcli datapreview osql command.
---

## What I do

I run ABAP SQL (OpenSQL) SELECT queries against a connected SAP system and
display the results. I use the `sapcli datapreview osql` command under the hood.

## When to use me

Use me when you need to query SAP database tables directly — for example to
inspect table contents, check configuration entries, or retrieve data for
analysis or debugging.

## Basic usage

```bash
sapcli datapreview osql "SELECT * FROM <TABLE>"
```

## Limiting rows returned

Use `--rows` to cap the number of result rows (default is determined by the
system):

```bash
sapcli datapreview osql --rows 20 "SELECT MANDT, CARRID, CONNID FROM SPFLI"
```

## Output formats

Results can be displayed in human-readable form (default) or as JSON:

```bash
# Human-readable table (default)
sapcli datapreview osql "SELECT * FROM T000"

# JSON output — useful for scripting or piping to jq
sapcli datapreview osql -o json "SELECT * FROM T000"
```

## Suppress headings

Use `-n` / `--noheadings` to omit column headers from the output:

```bash
sapcli datapreview osql -n "SELECT MANDT FROM T000"
```

## Disable aging check

Use `--noaging` to skip the data aging check:

```bash
sapcli datapreview osql --noaging "SELECT * FROM SFLIGHT UP TO 5 ROWS"
```

## Notes

- The `statement` argument must be valid ABAP SQL syntax **without** a trailing period.
- Only SELECT statements (no JOINS) are supported via `datapreview osql`.
