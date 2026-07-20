---
title: "Generating Cobra CLI Boilerplate with OpenCLI Specs"
date: 2026-07-15 07:30:00 -0400
categories: [blog]
tags: [cli, opencli]
excerpt: "Generate boilerplate CLI code deterministically using OpenCLI specifications - contract-first and framework-agnostic"
image: /assets/images/opencli/opengraph-image.png
header:
  og_image: /assets/images/opencli/opengraph-image.png
---

![OpenCLI Code Generation](https://images.unsplash.com/photo-1488972685288-c3fd157d7c7a?&auto=format&fit=crop)

Building a CLI means writing the same boilerplate over and over: parse flags, validate arguments, wire up help text, handle errors, repeat for every command. It's tedious work even when frameworks help automate things that distracts you from writing the actual logic behind each action.

What if you could define your CLI interface once in a declarative spec and generate all that scaffolding deterministically? That's exactly what [OpenCLI Spec](https://opencli.dev) enables.

An OpenCLI Spec (OCS) document defines a CLI's capabilities the same way an OpenAPI Spec defines an API, as a machine-readable contract. From that single source of truth, you can generate documentation, validate your design, and produce ready-to-use boilerplate code for popular frameworks like [Cobra](https://cobra.dev), [urfave/cli](https://cli.urfave.org), and [Yargs](https://yargs.js.org).

In this post, we'll walk through generating a complete Cobra-based CLI from an OpenCLI document and end up with a working, framework-agnostic application where your business logic is cleanly separated from generated scaffolding.

## The OpenCLI Document

Every OpenCLI-powered project starts with a spec-compliant YAML (or JSON) file. For this walkthrough, we'll use the [petstore-cli.ocs.yaml](https://github.com/bcdxn/opencli/blob/main/examples/petstore-cli.ocs.yaml) example from the [OpenCLI GitHub repository](https://github.com/bcdxn/opencli), modeled after the classic Swagger petstore API so the concepts feel familiar.

The document describes a CLI for managing pets, orders, and users. Here's a portion of what it looks like:

```yaml
opencliVersion: 1.0.0-alpha.12

info:
  title: PetStore CLI
  summary: An example CLI Document describing operations a petstore CLI may provide.
  # ...

commands:
  petstore pet add [flags]:
    summary: Add a new pet to the store
    args:
      - name: path-to-req-body
        type: string
        summary: The path to a JSON file containing the new pet payload
        required: false
    flags:
      - name: name
        aliases:
          - n
        type: string
        summary: The name of the pet
      - name: photo-urls
        aliases:
          - p
        type: string
        summary: A list of photo URLs to display for the pet
        description: |
          Provide this flag multiple times to set multiple photo URLs.
        variadic: true
      - name: status
        type: string
        summary: The pet status in the store
        choices:
          - value: available
          - value: pending
          - value: sold
      - name: tag
        type: string
        summary: Tag to assign to the pet for grouping/sorting
        description: |
          Provide this flag multiple times to add multiple tags.
        variadic: true

  petstore pet find-by-status [flags]:
    summary: Find pets by status
    flags:
      - name: status
        type: string
        summary: The status to filter pets by
        choices:
          - value: available
          - value: pending
          - value: sold
        required: true

  # ...
```

You can find the full example document [here](https://github.com/bcdxn/opencli/blob/main/examples/petstore-cli.ocs.yaml) and explore the complete specification schema at [opencli.dev/specification](https://opencli.dev/specification).

Notice how declarative this is; every command, argument, flag, and choice lives in one file. This document becomes your single source of truth for the CLI interface.

## Generating the Code

Now comes the powerful part: turning this declarative spec into working code. We'll generate a [Cobra](https://cobra.dev)-based CLI, but the same process works for [urfave/cli](https://cli.urfave.org) and [Yargs](https://yargs.js.org) as well.

The entire workflow is powered by `ocli` - the OpenCLI command-line tool that validates specs, generates documentation, and produces framework-specific boilerplate.

### Step 1: Install the `ocli` CLI

Install it using go or download the binary from [GitHub](https://github.com/bcdxn/opencli/releases).

```sh
go install github.com/bcdxn/opencli/cmd/ocli@latest
```

Verify the installation:

```sh
$ ocli --help

# A CLI for working with OpenCLI Specs. ocli is designed to make working with
# OpenCLI Spec documents (https://github.com/bcdxn/opencli) easier, providing:
#   • validation of OpenCLI Spec documents
#   • documentation generation from specs
#   • framework-specific code generation
#
# USAGE:
#   ocli {command} <arguments> [flags]
#
# AVAILABLE COMMANDS:
#   check  Check an OpenCLI Spec document for errors
#   gen    Generate code/docs from an OpenCLI Spec document
```

Want to add support for your favorite CLI framework? Open an [issue](https://github.com/bcdxn/opencli/issues) or submit a [pull request](https://github.com/bcdxn/opencli/blob/main/CONTRIBUTING.md).

### Step 2: Initialize the Project

Set up a fresh Go module and grab the petstore spec:

```sh
mkdir petstore && cd petstore
go mod init petstore
```

Pull in the example OCS document:

```sh
curl -O https://raw.githubusercontent.com/bcdxn/opencli/refs/heads/main/examples/petstore-cli.ocs.yaml
```

That's it for setup. One spec file, one module; now we're ready to generate code.

### Step 3: Generate Boilerplate Code

With the spec in hand, a single `ocli gen cli` command produces all the scaffolding:

```sh
ocli gen cli \
  --framework cobra \
  --out ./internal \
  ./petstore-cli.ocs.yaml

# → Reading spec:       ./petstore-cli.ocs.yaml
# → Generating CLI code:    framework=cobra, output=./internal
# ✓ CLI Code written to: ./internal
```

Then resolve dependencies:

```sh
go mod tidy
```

Now let's look at what `ocli` created. Everything is encapsulated in the `gencli` package. Each command gets its own file, plus supporting files for bootstrapping, error handling, and I/O management:

```
go.mod
petstore-cli.ocs.yaml
internal/
└── gencli/
    ├── actions.gen.go    Actions interface & command signatures
    ├── errors.gen.go     CLI error types & exit codes
    ├── help.gen.go       Default help/usage messaging
    ├── iostreams.gen.go  Standard I/O streams abstraction
    ├── params.gen.go     Command flags & parameter types
    ├── run.go            CLI entry point (Run function)
    └── cmd_...           Generated Cobra command definitions
```

> **<i class="fas fa-lightbulb"></i>key insight:**  
> the generated code defines an `ActionsInterface`. The interface creates a contract that maps one-to-one with every command in your spec. Your job is simply to implement it. Let's see how this works.
> {: .notice--info}

Each generated command file exposes a function that returns a Cobra command, receiving the `ActionsInterface` as a dependency:

```go
// cmd_petstore_pet_add.gen.go
func NewCmdPetstorePetAdd(a ActionsInterface) *cobra.Command {
  // ...
}
```

Inside, each command's `RunE` handler delegates to the corresponding method on `ActionsInterface`:

```go
// cmd_petstore_pet_add.gen.go
command := &cobra.Command{
  Use:   "add",
  // ...
  RunE: func(c *cobra.Command, args []string) error {
    // ... parse flags, validate args ...
    return a.PetstorePetAdd(c.Context(), cmdArgs, cmdFlags)
  },
}
```

This means the generated code handles all flag parsing, argument validation, and help text, and then calls your implementation. The `ActionsInterface` itself looks like this:

```go
// actions.gen.go
// Code generated by github.com/bcdxn/opencli@v1.0.0-alpha.14 DO NOT EDIT.
package gencli

import (
  "context"

  "github.com/bcdxn/opencli/spec"
)

// ActionsInterface defines all actions the petstore CLI supports.
type ActionsInterface interface {
  PetstoreList(ctx context.Context) error
  PetstorePetAdd(ctx context.Context, args PetstorePetAddArgs, flags PetstorePetAddFlags) error
  PetstorePetUpdate(ctx context.Context, args PetstorePetUpdateArgs, flags PetstorePetUpdateFlags) error
  PetstorePetFindByStatus(ctx context.Context, flags PetstorePetFindByStatusFlags) error
  PetstorePetFindByTags(ctx context.Context, flags PetstorePetFindByTagsFlags) error
  PetstorePetGet(ctx context.Context, flags PetstorePetGetFlags) error
  PetstorePetUpdateForm(ctx context.Context, flags PetstorePetUpdateFormFlags) error
  PetstorePetDelete(ctx context.Context, flags PetstorePetDeleteFlags) error
  PetstorePetUploadImage(ctx context.Context, args PetstorePetUploadImageArgs, flags PetstorePetUploadImageFlags) error
  PetstoreStoreInventory(ctx context.Context) error
  PetstoreStoreOrderPlace(ctx context.Context, args PetstoreStoreOrderPlaceArgs, flags PetstoreStoreOrderPlaceFlags) error
  PetstoreStoreOrderGet(ctx context.Context, flags PetstoreStoreOrderGetFlags) error
  PetstoreStoreOrderDelete(ctx context.Context, args PetstoreStoreOrderDeleteArgs) error
  PetstoreUserCreate(ctx context.Context, args PetstoreUserCreateArgs, flags PetstoreUserCreateFlags) error
  PetstoreUserCreateWithList(ctx context.Context, args PetstoreUserCreateWithListArgs) error
  PetstoreUserLogin(ctx context.Context, flags PetstoreUserLoginFlags) error
  PetstoreUserLogout(ctx context.Context) error
  PetstoreUserGet(ctx context.Context, flags PetstoreUserGetFlags) error
  PetstoreUserUpdate(ctx context.Context, flags PetstoreUserUpdateFlags) error
  PetstoreUserDelete(ctx context.Context, flags PetstoreUserDeleteFlags) error
  HelpFunc(cmd *spec.CommandItem)
  UsageFunc(cmd *spec.CommandItem) error
  IOStreams() IOStreams
  Version() string
}
```

Every method corresponds to a command defined in the spec. The generated types for `args` and `flags` are strongly typed, so you get compile-time safety — no more typos in flag names or mismatched types. A few helper methods (`HelpFunc`, `UsageFunc`, `IOStreams`, `Version`) round out the interface with sensible defaults.

### Step 4: Implement the Actions Interface

This is where you write your actual business logic. Create a type that satisfies `ActionsInterface` — and because it's just a Go interface, the pattern feels familiar if you've used [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) with OpenAPI specs.

The benefits of this contract-first approach are significant:

- **Your spec is your contract** — define the CLI surface first, then implement against a generated interface
- **Framework agnostic business logic** — your `Actions` type has zero dependencies on Cobra or any CLI framework
- **Always in sync** — documentation, and implementation all stay in sync with the OpenCLI Spec document as the source of truth

Start by creating a new package for your implementation:

```sh
mkdir -p ./internal/cliapp
touch ./internal/cliapp/actions.go
```

Define your `Actions` type:

```go
package cliapp

import (
  "context"
  "fmt"

  "petstore/internal/gencli"
  "github.com/bcdxn/opencli/spec"
)

func NewActions(version string) Actions {
  return Actions{version: version}
}

type Actions struct {
  version string
}
```

Now implement each method to fulfill the interface. For demonstration, we'll keep the bodies simple — in a real project, this is where you'd call your API, hit a database, or orchestrate whatever your CLI is designed to do:

```go
func (a Actions) PetstoreList(ctx context.Context) error {
  fmt.Println("listing all resources...")
  return nil
}

func (a Actions) PetstorePetAdd(
  ctx context.Context,
  args gencli.PetstorePetAddArgs,
  flags gencli.PetstorePetAddFlags,
) error {
  fmt.Printf("adding pet: name=%s, status=%s, tags=%v\n",
    flags.Name, flags.Status, flags.Tag)
  return nil
}

func (a Actions) PetstorePetUpdate(
  ctx context.Context,
  args gencli.PetstorePetUpdateArgs,
  flags gencli.PetstorePetUpdateFlags,
) error {
  fmt.Printf("updating pet with data from: %s\n", args.PathToReqBody)
  return nil
}

// ... implement remaining methods to satisfy ActionsInterface ...
```

Finally, wire up the helper methods using sensible defaults provided by the generated code (or replace them with custom implementations if you need tailored behavior):

```go
func (a Actions) HelpFunc(cmd *spec.CommandItem) {
  gencli.DefaultHelpFunc(a, cmd)
}

func (a Actions) UsageFunc(cmd *spec.CommandItem) error {
  gencli.DefaultUsageFunc(a, cmd)
  return nil
}

func (a Actions) IOStreams() gencli.IOStreams {
  return gencli.DefaultIOS()
}

func (a Actions) Version() string {
  return a.version
}
```

### Step 5: Wire Up the Entry Point

The final piece is a minimal `main.go` that bootstraps everything:

```sh
mkdir -p cmd/petstore
touch cmd/petstore/main.go
```

And we add the code:

```go
package main

import (
  "context"
  "os"

  "petstore/internal/cliapp"
  "petstore/internal/gencli"
)

var version = "DEV"

func main() {
  actions := cliapp.NewActions(version)
  code := gencli.Run(context.Background(), actions)
  os.Exit(code)
}
```

We need just three lines of substance, and critically, **no dependency on Cobra** in your 'user-land' code:

### Try it Out

That's the entire application. Let's run it:

```sh
$ go run cmd/petstore/main.go --help

# An example CLI Document describing operations a petstore CLI may provide.
#
# USAGE:
#   petstore {command} <arguments> [flags]
#
# AVAILABLE COMMANDS
#   list  List all endpoints available
#   pet   A collection of commands for managing pets
#   store A collection of commands for store operations
#   user  A collection of commands for user management
```

```sh
$ go run cmd/petstore/main.go pet add --name fluffy --status available --tag dog

# adding pet: name=fluffy, status=available, tags=[dog]
```

A fully functional CLI with zero framework coupling in your business logic. The spec defined the interface, `ocli` generated the scaffolding, and you implemented the business logic.

## Benefits of OpenCLI

Building CLIs traditionally means writing repetitive boilerplate: flag definitions, argument parsing, help text, error handling, version commands; multiply that by every subcommand and it gets tedious, fast. OpenCLI eliminates that tedium while giving you something better than manual authoring:

1. **Single source of truth** - your spec drives the code, the docs, and the validation. Change one file and regenerate.
2. **Contract-first development** - design your CLI interface before writing implementation logic, the same way OpenAPI enables contract-first API design.
3. **Framework-agnostic architecture** - swap Cobra for urfave/cli by changing a flag in `ocli gen cli`. Your business logic doesn't change. (Other CLI frameworks/languages are also supported)
4. **AI-friendly** - an OpenCLI document lets AI tools understand your CLI surface without scanning hundreds of lines of code, saving tokens and improving accuracy.
5. **Deterministic generation** - run `ocli gen cli` today or next month and get the same result. (oh, and it won't burn your precious tokens generating the scaffolding)

## Get Started

Try OpenCLI spec and `ocli` on your next CLI project:

- Read the [OpenCLI Specification](https://opencli.dev/specification) to understand the schema
- Experiment with the [live editor](https://opencli.dev/editor) to see docs generated in real time
- Install `ocli` and generate your first codebase: `go install github.com/bcdxn/opencli/cmd/ocli@latest`
- Explore examples and contribute at [github.com/bcdxn/opencli](https://github.com/bcdxn/opencli)

OpenCLI is in alpha — the core patterns are proven, and community feedback now shapes what comes next.
