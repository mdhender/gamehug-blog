---
name: using-ff-v4
description: "Write CLI commands and flag parsing with peterbourgon/ff/v4. Use when creating new CLI commands, adding flags, wiring subcommands, or working with env var / .env config file integration."
---

# Using ff/v4 in This Project

This skill covers the ff/v4 CLI framework as used in this project. The project
uses **ff v4.0.0-beta.1** (`github.com/peterbourgon/ff/v4`).

## Key Concepts

ff/v4 provides three things that replace cobra + pflag + godotenv:

1. **`ff.FlagSet`** — flag definitions with short (`-f`) and long (`--foo`) names
2. **`ff.Command`** — declarative command trees with `Exec` functions
3. **`ff.Parse`** — unified parsing: flag → env var → .env file → default

## Project Conventions

### Env Var Prefix

All entry points use `ff.WithEnvVarPrefix("EC")`. This auto-maps flag names to
env vars: `--data-path` → `EC_DATA_PATH`, `--jwt-secret` → `EC_JWT_SECRET`.

### Parse Options (standard set)

Both `cmd/api/main.go` and `cmd/cli/main.go` use these options:

```go
err := rootCmd.ParseAndRun(ctx, os.Args[1:],
    ff.WithEnvVarPrefix("EC"),
    ff.WithConfigFile(".env"),
    ff.WithConfigFileParser(ffenv.Parse),
    ff.WithConfigAllowMissingFile(),
    ff.WithConfigIgnoreFlagNames(),
    ff.WithConfigIgnoreUndefinedFlags(),
)
```

- `WithEnvVarPrefix("EC")` — flags are set from `EC_*` env vars
- `WithConfigFile(".env")` — loads `.env` from the current working directory
- `WithConfigFileParser(ffenv.Parse)` — parses `KEY=VALUE` format
- `WithConfigAllowMissingFile()` — no error if `.env` doesn't exist
- `WithConfigIgnoreFlagNames()` — .env keys match env var names only (not flag names)
- `WithConfigIgnoreUndefinedFlags()` — .env can contain keys for other commands

**Only `.env` is loaded.** The old multi-file dotenv chain (`.env.local`,
`.env.development.local`, etc.) was removed in Sprint 6.

### Priority Chain

Values resolve in this order (highest priority first):

1. CLI flag (`--data-path /foo`)
2. Environment variable (`EC_DATA_PATH=/foo`)
3. `.env` file (`EC_DATA_PATH=/foo`)
4. Default value from flag definition

### Config File Format

The `.env` file uses env var names (with the `EC_` prefix), not flag names:

```env
EC_DATA_PATH=/var/data/ec
EC_JWT_SECRET=supersecret
EC_HOST=0.0.0.0
```

### Logging

- Logs go to `os.Stderr`, command output goes to `os.Stdout`.
- Root flags: `--log-level`, `--log-source`, `--debug`, `--quiet`.
- `--debug` and `--quiet` are mutually exclusive overrides of `--log-level`.

### Error Handling

```go
if err != nil {
    if errors.Is(err, ff.ErrHelp) {
        fmt.Fprintf(os.Stderr, "%s\n", ffhelp.Command(rootCmd))
        os.Exit(0)
    }
    if errors.Is(err, ff.ErrNoExec) {
        fmt.Fprintf(os.Stderr, "%s\n", ffhelp.Command(rootCmd))
        os.Exit(1)
    }
    fmt.Fprintf(os.Stderr, "error: %v\n", err)
    os.Exit(1)
}
```

---

## FlagSet API

### Creating a FlagSet

```go
fs := ff.NewFlagSet("command-name")
```

### Parent Chaining

Subcommand flag sets chain to their parent so parent flags are visible:

```go
rootFlags := ff.NewFlagSet("cli")
serveFlags := ff.NewFlagSet("serve").SetParent(rootFlags)
```

### Flag Definition Methods

All methods return a pointer to the value. All panic on error (e.g., duplicate name).

#### String

```go
host := fs.StringLong("host", "localhost", "listen host")
name := fs.String('n', "name", "default", "usage")   // short + long
abbr := fs.StringShort('n', "default", "usage")       // short only
```

#### Bool

```go
verbose := fs.BoolLong("verbose", "enable verbose output")
debug := fs.Bool('d', "debug", "enable debug mode")   // short + long
```

Bool flags default to `false`. On the command line, `--verbose` sets to `true`.
`--verbose=false` explicitly sets `false`.

#### Int / Int64

```go
count := fs.IntLong("count", 10, "number of items")
big := fs.Int64Long("big-number", 0, "a large number")
```

#### Uint / Uint64

```go
port := fs.UintLong("port", 8080, "listen port")
seed := fs.Uint64Long("seed", 0, "random seed")
```

**Note:** ff/v4 beta.1 does have native `Uint64Long`. The sprint plan said it
didn't, but the actual API includes it. Existing code uses `StringLong` +
`strconv.ParseUint` for seed flags — either approach works.

#### Duration

```go
timeout := fs.DurationLong("timeout", 0, "auto-shutdown timeout")
```

#### Float64

```go
rate := fs.Float64Long("rate", 1.0, "processing rate")
```

#### StringEnum

```go
format := fs.StringEnumLong("format", "choose output format", "json", "text", "csv")
```

Restricts the value to the listed options.

#### StringList

```go
tags := fs.StringListLong("tag", "add a tag (repeatable)")
```

Supports repeated flags: `--tag foo --tag bar`.

#### Func

For custom validation that runs at parse time:

```go
fs.FuncLong("validate", func(val string) error {
    if len(val) < 3 {
        return fmt.Errorf("must be at least 3 characters")
    }
    return nil
}, "value to validate")
```

#### AddFlag (low-level)

For full control over flag configuration:

```go
fs.AddFlag(ff.FlagConfig{
    ShortName:   'f',
    LongName:    "file",
    Usage:       "path to `file`",    // backtick = placeholder
    Placeholder: "PATH",
    Value:       ffval.NewValueDefault(&myVar, "default"),
})
```

#### AddStruct

Define flags from a struct with `ff:` tags:

```go
type Config struct {
    Host string `ff:"long: host, short: h, default: localhost, usage: 'listen host'"`
    Port int    `ff:"long: port, default: 8080, usage: 'listen port'"`
}
var cfg Config
fs.AddStruct(&cfg)
```

### Querying Flags After Parse

```go
flag, ok := fs.GetFlag("host")    // by long name
flag.IsSet()                       // true if explicitly provided
flag.GetValue()                    // current value as string
flag.GetDefault()                  // default value as string
fs.GetArgs()                       // leftover args after parse
fs.IsParsed()                      // true after successful parse
```

### Walking Flags

```go
fs.WalkFlags(func(f ff.Flag) error {
    long, _ := f.GetLongName()
    fmt.Printf("%s = %s\n", long, f.GetValue())
    return nil
})
```

### Reset

To parse the same flag set again (e.g., in tests):

```go
cmd.Reset()  // resets command tree + all flag sets
```

---

## Command API

### Defining a Command

```go
cmd := &ff.Command{
    Name:      "serve",
    Usage:     "api serve [FLAGS]",
    ShortHelp: "start the API server",
    LongHelp:  "Detailed multi-line description...",
    Flags:     serveFlags,
    Exec: func(ctx context.Context, args []string) error {
        // args = leftover args after flag parsing
        return nil
    },
}
```

### Command Tree

```go
rootCmd := &ff.Command{
    Name:  "cli",
    Flags: rootFlags,
    Subcommands: []*ff.Command{
        createCmd,
        showCmd,
        versionCmd,
    },
}

createCmd := &ff.Command{
    Name:  "create",
    Flags: ff.NewFlagSet("create").SetParent(rootFlags),
    Subcommands: []*ff.Command{
        clusterCmd,
        gameCmd,
    },
}
```

Subcommand selection is case-insensitive and based on the first post-parse arg.

### Parse, Run, ParseAndRun

```go
// Separate parse and run:
if err := cmd.Parse(os.Args[1:], options...); err != nil { ... }
if err := cmd.Run(ctx); err != nil { ... }

// Combined:
if err := cmd.ParseAndRun(ctx, os.Args[1:], options...); err != nil { ... }
```

### Command Navigation

```go
cmd.GetSelected()  // terminal command after parse
cmd.GetParent()    // parent command (set during parse)
```

---

## Subpackages

### ffenv — .env file parser

```go
import "github.com/peterbourgon/ff/v4/ffenv"
```

Parses `KEY=VALUE` format. Supports:
- `# comments` (full-line only, no end-of-line comments)
- `"double quoted"` values (with `\n` expansion)
- Blank lines are skipped

Used via `ff.WithConfigFileParser(ffenv.Parse)`.

### ffhelp — help text generation

```go
import "github.com/peterbourgon/ff/v4/ffhelp"
```

```go
// For a command tree:
fmt.Fprintf(os.Stderr, "%s\n", ffhelp.Command(rootCmd))

// For a standalone flag set:
ffhelp.Flags(fs, "myapp [FLAGS]").WriteTo(os.Stderr)
```

`ffhelp.Command` automatically follows `GetSelected()` to show help for the
selected subcommand.

### ffval — custom value types

```go
import "github.com/peterbourgon/ff/v4/ffval"
```

- `ffval.NewValueDefault[T](&var, defaultVal)` — typed value with default
- `ffval.Func(fn)` — function-based value for custom parsing
- Implements `flag.Value` + `ff.Resetter`

### fftoml / ffjson / ffyaml — config file parsers

Available but not used in this project. The project uses `ffenv.Parse` for
`.env` files.

---

## Errors

| Error | Meaning |
|---|---|
| `ff.ErrHelp` | User requested help (`-h`, `--help`) |
| `ff.ErrNoExec` | Command has no `Exec` function |
| `ff.ErrNotParsed` | `Run` called before `Parse` |
| `ff.ErrAlreadyParsed` | `Parse` called twice without `Reset` |
| `ff.ErrDuplicateFlag` | Two flags with the same name |
| `ff.ErrUnknownFlag` | Flag not defined in the flag set |
| `ff.ErrAmbiguousFlag` | Flag name matches multiple definitions |

Use `errors.Is()` for matching. `ErrUnknownFlag` wraps as `*ff.UnknownFlagError`
with `GetFlagName()` / `GetName()` methods.

---

## Project-Specific Patterns

### Delivery Layer Commands

Commands in `delivery/cli/` are thin — parse flags, validate required fields,
call a service, format output. No business logic, no file I/O beyond what the
service provides.

```go
func CmdCreateThing(svc *app.ThingService) *ff.Command {
    fs := ff.NewFlagSet("thing")
    dataPath := fs.StringLong("data-path", "", "path to data directory")
    name := fs.StringLong("name", "", "thing name")

    return &ff.Command{
        Name:      "thing",
        Usage:     "cli create thing [FLAGS]",
        ShortHelp: "create a new thing",
        Flags:     fs,
        Exec: func(ctx context.Context, args []string) error {
            if *dataPath == "" {
                return fmt.Errorf("--data-path is required")
            }
            if *name == "" {
                return fmt.Errorf("--name is required")
            }
            return svc.CreateThing(*dataPath, *name)
        },
    }
}
```

### Runtime Wiring

`runtime/cli/cli.go` builds the command tree by instantiating services and
calling delivery command builders:

```go
func BuildCommands() []*ff.Command {
    store := filestore.NewStore("")
    clusterSvc := app.NewClusterService(store)
    gameConfigSvc := app.NewGameConfigService(store)

    createCmd := &ff.Command{
        Name: "create",
        Subcommands: []*ff.Command{
            cli.CmdCreateCluster(clusterSvc),
            cli.CmdCreateGameState(clusterSvc),
            cli.CmdCreateGame(gameConfigSvc),
            cli.CmdAddEmpire(gameConfigSvc),
        },
    }
    // ...
    return []*ff.Command{createCmd, removeCmd, showCmd, testCmd}
}
```

### Required Flag Validation

ff/v4 has no "required flag" mechanism. Validate after parse:

```go
if *dataPath == "" {
    return fmt.Errorf("--data-path is required (or set EC_DATA_PATH)")
}
```

This is intentional — it lets env vars and .env files satisfy "required" fields.

### Uint64 Flags (seed pattern)

Existing code uses `StringLong` + `strconv.ParseUint` for uint64 flags, but
ff/v4 beta.1 does provide `Uint64Long` natively. Either works:

```go
// Option A: native (preferred for new code)
seed := fs.Uint64Long("seed", 0, "random seed")

// Option B: string + parse (used in existing cluster commands)
seedStr := fs.StringLong("seed", "0", "random seed")
// in Exec:
seed, err := strconv.ParseUint(*seedStr, 10, 64)
```

### FlagSet Without Parent

Commands in `delivery/cli/` create flag sets without parents. The parent is set
by the runtime layer when it wires the command tree — ff.Command handles parent
flag visibility through the command hierarchy automatically.

### No Imports of cobra, pflag, godotenv, or dotfiles

These packages were removed in Sprint 6. Never add them back.
