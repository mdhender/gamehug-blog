---
title: "Sprint 6: Pulling Out the Plumbing"
date: 2026-03-25T12:00:00
---

{{< callout type="info" >}}
   Sprint 6 is complete. The CLI framework and config plumbing have been replaced — cobra, godotenv, and four copies of hand-rolled resolver helpers are gone.
{{< /callout >}}

## What We Did

This sprint didn't add features. It replaced **infrastructure** — the kind of change that makes every future sprint easier.

The project was using [spf13/cobra](https://github.com/spf13/cobra) for command trees, [joho/godotenv](https://github.com/joho/godotenv) for `.env` loading, and hand-rolled `resolveString`/`resolveDuration` helpers for the flag → env → fallback chain. Those helpers were copy-pasted in four files (`cmd/api`, `cmd/cli`, `runtime/cli`, `delivery/cli`). Every time the resolution logic needed to change, it needed to change in four places.

The replacement is [peterbourgon/ff/v4](https://github.com/peterbourgon/ff), which handles all of that in one place:

- **`ff.Command`** replaces `cobra.Command` — declarative command trees with the same subcommand structure.
- **`ff.FlagSet`** replaces `pflag` — flag definitions with parent chaining.
- **`ff.Parse` with `WithEnvVarPrefix("EC")`** replaces all four `resolveString` copies — `--data-path` automatically maps to `EC_DATA_PATH`, no glue code needed.
- **`ffenv.Parse`** replaces `godotenv` — `.env` files are parsed directly by the flag framework.

The priority chain is now built in: **flag value → env var → .env file → default**. One mechanism, zero helpers.

---

## Bugs Fixed Along the Way

The migration shook out two bugs that had been hiding in the old code:

{{< cards cols="1" >}}
   {{< card
      title="CmdShowMagicLink's silent flag"
      subtitle="The command defined a --data-path flag but then called resolveString(\"path\") to read it. The flag was silently ignored — the value always came from the environment or the default. With ff, the flag just works."
   >}}

   {{< card
      title="The --info flag that nobody read"
      subtitle="Both entry points (cmd/api and cmd/cli) defined an --info flag and wired it into PersistentPreRunE. But the flag was never checked — the code only looked at --debug and --quiet. Removed entirely."
   >}}
{{< /cards >}}

Neither bug had caused visible failures because the env var fallback happened to paper over the first one, and the second was a no-op. But they were real bugs — dead code and broken bindings that would have confused anyone reading the source.

---

## What Changed, File by File

The diff is mostly deletion:

- **Deleted** `internal/dotfiles/` — the entire package. Its job (loading `.env`) is now one parse option: `ff.WithConfigFile(".env")`.
- **Deleted** `delivery/cli/resolver.go` — the `resolveString`/`resolveDuration` helpers. ff does this automatically.
- **Rewrote** `cmd/api/main.go` and `cmd/cli/main.go` — cobra root commands became ff root commands. Both use identical parse options.
- **Converted** `delivery/cli/cluster.go` and `delivery/cli/game_config.go` — every command-builder function now returns `*ff.Command` instead of `*cobra.Command`.
- **Updated** `runtime/cli/cli.go` — `AddCommands` became `BuildCommands`, returning a slice of `*ff.Command` instead of mutating a cobra root.

The SOUSA boundaries didn't move. `delivery/cli` still has no `infra` imports. `runtime/cli` still does the wiring. The layers are the same; the framework inside them changed.

---

## One Rename

The game config commands (`create game`, `create empire`, `remove empire`, `show magic-link`) previously used `--path` for the data directory. That was inconsistent with the env var `EC_DATA_PATH` and with the API server's `--data-path` flag.

Renamed to `--data-path` everywhere. The cluster commands still use `--path` — that's a file path to a cluster JSON file, not the data directory. Different things, different names.

---

## Logs Go to Stderr Now

Both entry points now write log output to `os.Stderr`. Command output (version strings, reports, magic links) goes to `os.Stdout`. This was always the intent but wasn't consistently enforced. Now it is — you can pipe command output without log noise mixed in.

---

## What's Not Here Yet

The missing-feature list hasn't changed from Sprint 5:

- **No empire placement** — empires are registered but don't have starting colonies.
- **No order parsing** — orders are stored but not interpreted.
- **No turn processing** — the engine pipeline hasn't started.
- **No database** — still file-backed.

---

## What's Next

This sprint was about paying down infrastructure debt before it compounded. With the config plumbing unified, the next sprints can add flags and env vars to new commands without copying boilerplate.

The roadmap is the same: empire placement, order parsing, turn execution, report generation. The core loop.

---

## Version

The project is now at **v0.5.0-alpha**. All tests pass, both entry points build, and `go.mod` is four dependencies lighter.
