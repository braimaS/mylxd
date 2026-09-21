Cobra is small once you see its core idea: **a CLI is a tree of `cobra.Command` structs, and cobra walks that tree to decide what to run.**

## The `Command` struct

Each command is just a struct with a few fields you fill in:

- **`Use`** is the command's name as typed (`"init"`, `"sql"`). For the root it's the program name.
- **`Short` / `Long`** are help text, shown by `--help`.
- **`Args`** is a validator for positional arguments, such as `cobra.ExactArgs(1)`, `cobra.NoArgs`, or `cobra.ArbitraryArgs`.
- **`RunE`** is the function that runs when this command is chosen. It returns an error (plain `Run` is the no-error version).
- **`PersistentPreRunE`** runs before `RunE` for this command *and all its children*. It's used for shared setup like logging.

## Flags

Every command owns a flag set, and there are two kinds:

- **`cmd.Flags()`** are local flags, valid only on that exact command.
- **`cmd.PersistentFlags()`** are inherited flags, valid on that command and every descendant. Global flags like `--debug` go here on the root.

Flags are usually bound directly to variables, so after parsing, the variables are already filled in.

## The tree and `Execute()`

You build the tree with `parent.AddCommand(child)`, then call `root.Execute()` once. Cobra then:

1. Reads `os.Args` and walks down the tree, matching words to child `Use` names (`lxd sql` goes root → `sql`).
2. Parses flags, including persistent ones inherited from ancestors.
3. Checks leftover positional args against that command's `Args` validator.
4. Runs `PersistentPreRunE` (from the nearest ancestor that defines one), then the chosen command's `RunE`.

## A tiny complete example

```go
package main

import (
    "fmt"
    "github.com/spf13/cobra"
)

func main() {
    var debug bool
    var name string

    root := &cobra.Command{
        Use:  "app",
        RunE: func(cmd *cobra.Command, args []string) error {
            fmt.Println("root running, debug =", debug)
            return nil
        },
    }
    root.PersistentFlags().BoolVar(&debug, "debug", false, "debug mode")

    greet := &cobra.Command{
        Use:  "greet",
        Args: cobra.NoArgs,
        RunE: func(cmd *cobra.Command, args []string) error {
            fmt.Println("hello", name, "debug =", debug)
            return nil
        },
    }
    greet.Flags().StringVar(&name, "name", "world", "who to greet")

    root.AddCommand(greet)
    root.Execute()
}
```

Here's what different invocations do:

- `app --debug` runs root's `RunE`. The root being runnable is exactly LXD's daemon trick.
- `app greet --name Mo` runs `greet`'s `RunE`.
- `app greet --debug` also works, because `--debug` is persistent and inherited from root.
- `app greet --name Mo extra` fails, because `greet` says `NoArgs`.

## The LXD pattern on top of cobra

LXD wraps each command in its own struct type, and you'll see this everywhere in `lxd/` and `lxc/`:

```go
type cmdInit struct {
    global  *cmdGlobal
    flagAuto bool         // flag values live as struct fields
}

func (c *cmdInit) command() *cobra.Command {
    cmd := &cobra.Command{}
    cmd.Use = "init"
    cmd.RunE = c.run
    cmd.Flags().BoolVar(&c.flagAuto, "auto", false, "...")
    return cmd
}

func (c *cmdInit) run(cmd *cobra.Command, args []string) error {
    // uses c.flagAuto, c.global.flagLogDebug, etc.
}
```

So each `cmdX` struct holds its flag values, `command()` builds the cobra command and binds flags to those fields, and `run()` does the work. The shared `cmdGlobal` struct holds the persistent flags (like `--debug`) so every subcommand can read them.

With that, `main.go` should read cleanly: build `cmdDaemon` as the root, attach persistent global flags, `AddCommand` each subcommand, `Execute()`. The only thing left to follow is `cmdDaemon.run`, which is where the daemon actually starts.