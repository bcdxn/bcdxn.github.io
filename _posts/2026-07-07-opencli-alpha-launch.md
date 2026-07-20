---
title: "OpenCLI Spec Is Now in Alpha"
date: 2026-07-07 09:00:00 -0400
categories: [blog]
tags: [opencli, cli, developer-tools]
excerpt: "OpenCLI is now in alpha: a faster way to write, test, and share command-line workflows with an interactive editor."
image: /assets/images/opencli/opengraph-image.png
header:
  og_image: /assets/images/opencli/opengraph-image.png
---

#### Design, Build, and Document CLIs with Speed and Consistency

![My Image](/assets/images/opencli/opengraph-image.png){: style="max-height: 400px; width: auto;" }

Today, after over a year of on-and-off development, I'm excited to announce the alpha release of the OpenCLI Spec ecosystem.

[OpenCLI Spec](https://opencli.dev/specification) is an open document specification and toolkit for designing, documenting, and building command-line interfaces with greater consistency and less friction.

If you're new to the project, visit the [OpenCLI website](https://opencli.dev/) to learn more about the vision, specification, and current capabilities. Visit the [OpenCLI GitHub repository](https://github.com/bcdxn/opencli) if you'd like to [propose new features](https://github.com/bcdxn/opencli/issues) or [contribute](https://github.com/bcdxn/opencli/blob/main/CONTRIBUTING.md).

## Why Use OpenCLI Spec?

- **Document your CLI** with a machine-readable specification, much like OpenAPI does for REST APIs.
- **Design CLIs contract-first**, defining the interface before implementation.
- **Generate beautiful and consistent documentation** from a single source of truth.
- **Generate boilerplate code deterministically** while remaining framework-agnostic.
- **Help AI tools understand your CLI** without spending precious tokens scanning the entire codebase.

OpenCLI's goal is to make command-line tools easier to design, implement, and document by providing a reliable contract for both humans and AI.

## Example

OpenCLI Spec documents are simply YAML (or JSON) files that define the surface area of available commands and their parameters. Let's write a document for a fictional `pleasantries` CLI that is OpenCLI Spec-compliant.

We always start with the `opencliVersion` property at the root of the file. This tells the OpenCLI tooling what version of OpenCLI specification is used so that it can be properly parsed.

```yaml
# pleasantries.ocs.yaml
opencliVersion: 1.0.0-alpha.10
```

Next, we add the `info` property to the root of the document. `info` defines metadata about the CLI. See more acceptable properties in the [OpenCLI specification documentation](https://opencli.dev/specification#info-object).

```yaml
# pleasantries.ocs.yaml
opencliVersion: 1.0.0-alpha.10

info:
  title: Pleasantries
  summary: A fun CLI to greet or bid farewell
  version: 1.0.0
  binary: pleasantries
```

Finally, we can add the `commands` property which defines the interface of our command line interface. The `commands` property is a map where each key is the command and the value defines command parameters. You can find the exact properties detailed in the [specification schema](https://opencli.dev/specification) and/or look at more in-depth examples in the [GitHub repository](https://github.com/bcdxn/opencli/tree/main/examples).

```yaml
# pleasantries.ocs.yaml
opencliVersion: 1.0.0-alpha.10

info:
  title: Pleasantries
  summary: A fun CLI to greet or bid farewell
  version: 1.0.0
  binary: pleasantries

commands:
  pleasantries greet <name> [flags]:
    summary: "Say hello"
    args:
      - name: "name"
        summary: "A name to include in the greeting"
        required: true
        type: "string"
    flags:
      - name: "language"
        summary: "The language of the greeting"
        type: "string"
        choices:
          - value: "english"
          - value: "spanish"
        default: "english"
  pleasantries farewell <name> [flags]:
    summary: "Say goodbye"
    args:
      - name: "name"
        summary: "A name to include in the farewell"
        required: true
        type: "string"
    flags:
      - name: "language"
        summary: "The language of the farewell"
        type: "string"
        choices:
          - value: "english"
          - value: "spanish"
        default: "english"
```

This document describes a CLI that has two nested commands `greet` and `farewell`:

```sh
$ pleasantries greet Bob --language english
# hello Bob!
$ pleasantries greet Alice --language spanish
# Hola Alice!
$ pleasantries farewell Bob --language english
# good bye Bob!
$ pleasantries farewell Alice --language spanish
# adiós Alice!
```

## OpenCLI Spec Editor

Check out the [OpenCLI live editor](https://opencli.dev/editor) for a more in-depth answer and see how documentation is automatically generated from the specification document.

<a href="https://opencli.dev/editor" class="btn btn--primary">Try It Out</a>

## Open Source and Feedback-Friendly

The project is open source, and you can review code, issues, and progress in the [OpenCLI GitHub repository](https://github.com/bcdxn/opencli).

Alpha means the core ideas are live, and now community feedback is especially valuable. If you care about developer experience, CLI ergonomics, or workflow reliability, your input can shape what comes next.
