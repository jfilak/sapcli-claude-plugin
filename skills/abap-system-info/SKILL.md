---
name: abap-system-info
description: Fetch and displays ABAP system information such as system ID, client, and user details.
---

## What I do

I connect to an ABAP system and retrieve essential information such as the
system ID, client, and user details. This information is then displayed to the
user in a clear and concise manner.

## When to use me

Use me when you need to quickly access and display key information about the
ABAP system you are connected to. This can be particularly useful for
troubleshooting, system monitoring, or when you need to confirm your connection
details.

## Displaying all system information

```bash
sapcli abap systeminfo
```

## Displaying all system particular information

```bash
sapcli abap systeminfo --key SID
```
