---
title: "Sprint 12 Cleanup: Parser Tests, CLI, and Doc Fixes"
date: 2026-03-27T21:00:00
---

{{< callout type="info" >}}
   Post-sprint cleanup for Sprint 12. Grammar docs corrected, file-driven parser tests added, a new CLI parse command shipped, and a referee how-to published.
{{< /callout >}}

## What Happened

Sprint 12 shipped the order parser and the `POST /orders/parse` API endpoint. The post-sprint review — [documented in the previous blog entry](/blog/sprint-12-mvp-order-language/#what-we-got-wrong) — found eight problems and fixed them all. This cleanup session tackles the next layer of housekeeping: making sure the grammar docs say what the parser actually does, adding file-driven test coverage, and giving referees a CLI tool they can use without touching the API.

---

## Grammar and Doc Fixes

Two player-facing reference pages had small mismatches with the parser:

- **`pay`** — the reference said the rate field was an integer. The parser accepts a decimal (e.g., `0.125`). The reference now matches.
- **`ration`** — the reference showed a bare integer for the percentage. The parser expects a `%` suffix (e.g., `50%`). The reference now matches.

These are the kind of drift that Sprint 12's review was designed to catch. The grammar document is the authority; when the parser and the docs disagree, we fix whichever one is wrong.

---

## File-Driven Parser Tests

Until now, every parser test case was an inline string literal in `parser_test.go`. That works for individual commands but doesn't exercise the parser the way a real order file does — multiple lines, mixed valid and invalid orders, comments, blank lines, quoted strings with embedded `//`.

Two testdata files now live alongside the test:

**`testdata/valid_orders.txt`** — 14 clean orders covering every MVP command, including edge cases like comma-separated quantities, decimal pay rates, percentage rations, and quoted names with embedded `//`. The test asserts 0 diagnostics and at least one parsed order.

**`testdata/errors_mixed.txt`** — 3 valid orders mixed with 8 intentionally bad lines hitting every diagnostic code: `unknown_command`, `not_implemented`, `syntax`, `invalid_value`, `unterminated_quote`, and `unexpected_end`. The test asserts exact counts for both orders and diagnostics.

Both tests log every parsed order (kind and phase) and every diagnostic (line, code, message) so failures produce useful output without needing a debugger.

---

## CLI Parse Command

The API endpoint is useful for integrations, but a referee running a game locally wants a command-line tool. The new `cli parse orders` subcommand reads one or more order files from disk, runs each through the parser, and prints a one-line summary per file:

```text
orders-empire1.txt: 14 orders, 0 diagnostics
```

When a file has errors, each diagnostic prints indented below the summary with line number, code, and message:

```text
orders-empire4.txt: 3 orders, 4 diagnostics
  line 5 [unknown_command]: unrecognized command "frobnicator"
  line 7 [syntax]: build change requires <colonyID> <groupNo> <unitKind>
  line 10 [unterminated_quote]: unterminated quoted string
  line 12 [invalid_value]: assemble: invalid unit kind "hyper-engine": …
```

The command exits with status 1 if any file had read errors or diagnostics. That makes it scriptable:

```sh
if cli parse orders orders-*.txt; then
    echo "all orders clean"
else
    echo "fix errors before running the turn"
fi
```

The implementation follows SOUSA. `delivery/cli/parse.go` defines the `ff.Command` and calls `ParseOrdersService.Parse()` — the same service the HTTP handler uses. `runtime/cli/cli.go` wires it with `ordertext.NewParser()`. The CLI never touches infra directly.

A full referee how-to is published at [Validate Player Order Files](/docs/referees/how-to/validate-order-files/).

---

## What's Next

The cleanup is done. Sprint 13 starts the turn engine — the scaffolding that will eventually execute the orders this parser accepts.

---

## Version

The project is now at **v0.12.1-alpha**. The build is green, all backend tests pass, and `go vet` is clean.
