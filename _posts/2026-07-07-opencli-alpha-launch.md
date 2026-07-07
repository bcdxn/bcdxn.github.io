---
title: "OpenCLI Spec Is Now in Alpha"
date: 2026-07-07 09:00:00 -0700
categories: [launch, developer-tools]
tags: [opencli, cli, product-launch, alpha]
excerpt: "OpenCLI is now in alpha: a faster way to write, test, and share command-line workflows with an interactive editor."
image: /assets/images/opencli/opengraph-image.png
---

#### Design, Build, and Document CLIs with Speed and Consistency

![My Image](/assets/images/opencli/opengraph-image.png){: style="max-height: 400px; width: auto;" }

Today, after over a year of on-and-off development, I'm excited to announce the alpha release of the OpenCLI Spec ecosystem.

[OpenCLI Spec](htktps://opencli.dev/specification) is an open document specification and toolkit for designing, documenting, and building command-line interfaces with greater consistency and less friction.

If you're new to the project, visit the [OpenCLI website](https://opencli.dev/) to learn more about the vision, specification, and current capabilities. Visit the [OpenCLI GitHub repository](https://github.com/bcdxn/opencli) if you'd like to [propose new features](https://github.com/bcdxn/opencli/issues) or [contribute](https://github.com/bcdxn/opencli/blob/main/CONTRIBUTING.md).

## Why Use OpenCLI Spec?

- **Document your CLI** with a machine-readable specification, much like OpenAPI does for REST APIs.
- **Design CLIs contract-first**, defining the interface before implementation.
- **Generate beautiful and consistent documentation** from a single source of truth.
- **Generate boilerplate code deterministically** while remaining framework-agnostic.
- **Help AI tools understand your CLI** without spending precious tokens scanning the entire codebase.

OpenCLI's goal is to make command-line tools easier to design, implement, and document, by providing a reliable contract for both humans and AI.

## Example

OpenCLI Spec documents are simply YAML (or JSON) files that define the surface area of available commands and their parameters. Let's look at a sample document for a fictional `pleasantries` CLI that is OpenCLI Spec-compliant:

```yaml
# pleasantries.ocs.yaml
opencliVersion: 1.0.0-alpha.10

info:
  title: Pleasantries
  summary: A fun CLI to greet or bid farewell
  version: 1.0.0
  binary: pleasantries

commands:
  pleasantries {command} <arguments> [flags]:
    group: true

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
```

The `info` property defines metadata about the CLI. The `commands` property defines a map where each key is the command and the value defines command parameters. You can find the exact properties detailed in the [specification schema](https://opencli.dev/specification) and/or look at more in-depth examples in the [GitHub repository](https://github.com/bcdxn/opencli/tree/main/examples).

Although commands in a CLI are hierarchical, the document is intentionally designed to be flat to improve readability. The hierarchy can be (and is) maintained in the tooling ecosystem when [unmarshalling](https://pkg.go.dev/github.com/bcdxn/opencli@v1.0.0-alpha.9/codec) the document.

This document describes a CLI that has a single nested command `greet`:

```sh
$ pleasantries greet Bob --language english
# hello Bob!
$ pleasantries greet Alice --language spanish
# Hola Alice!
```

## OpenCLI Spec Editor

Check out the [OpenCLI live editor](https://opencli.dev/editor) and see how documentation is automatically generated from the specification document.

<a href="https://opencli.dev/editor" class="btn btn--primary">Try It Out</a>

## Open Source and Feedback-Friendly

The project is open source, and you can review code, issues, and progress in the [OpenCLI GitHub repository](https://github.com/bcdxn/opencli).

Alpha means the core ideas are live, and now community feedback is especially valuable. If you care about developer experience, CLI ergonomics, or workflow reliability, your input can shape what comes next.
