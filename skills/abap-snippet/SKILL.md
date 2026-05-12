---
name: abap-snippet
description: Run ABAP code snippet to very syntax, execute Function Modules or perform complicate queries.
---

## What I do

I create a temporary ABAP class implementing the interface `IF_OO_ADT_CLASSRUN`
and put your ABAP code snippet into implementation of the method `IF_OO_ADT_CLASSRUN~MAIN`.

I run syntax check for the temporary class and if there are no syntax errors, I execute
the method `IF_OO_ADT_CLASSRUN~MAIN` and print the output of the execution into
the standard output.


## Basic usage: one-liner

```bash
echo "out->write( |Hello, { sy-uname }!| )." | sapcli abap run -
```

## Multiple lines of ABAP code

```bash
sapcli abap run - <<EOF
  out->write( |Hello, { sy-uname }!| ).
  out->write( |Current date is { sy-datum }| ).
EOF
```

## Running ABAP code snippet from file

First create a file `snippet.abap` with the following content:

```abap
out->write( |Hello, { sy-uname }!| ).
out->write( |Current date is { sy-datum }| ).
```

Then run the code snippet from the file:

```bash
sapcli abap run snippet.abap
```

