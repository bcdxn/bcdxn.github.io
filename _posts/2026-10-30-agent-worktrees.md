---
title: "Letting My Sandboxed Agents Commit, But Not Push"
date: 2026-10-30 07:30:00 -0400
published: false # OUTLINE ONLY - remove when the post is written
categories: [blog]
tags: [ai, agents, security, docker, vscode, git, localllm]
excerpt: "Git worktrees give each agent its own branch; the sandbox gives it no way to push. Every commit says which agent made it."
image:
  path: /assets/images/banners/2026-09-03.jpg # TODO: new banner
  alt: header
header:
  og_image: /assets/images/banners/2026-09-03.jpg # TODO: new banner
---

<!-- OUTLINE ONLY. Technical details live in scratch.md -->

## Intro

- Follow-up to [Sandboxing My Coding Agents](/blog/2026/09/03/agent-sandboxing.html) and the [Dev Container follow-up](/blog/2026/09/05/minimal-vscode-devcontainer.html)
- Where that left off: the agent is boxed in on files and network, but every change lands as one big uncommitted diff in my working copy
- What I want now:
  - Several agents working in parallel without stepping on my checkout or on each other
  - The agent can commit, so I get small reviewable steps and history
  - Commits say plainly that an agent wrote them (which agent, which model, which session)
  - The agent can't push. Getting anything to GitHub stays my call
- One-line thesis: committing is a local action and pushing is a credentialed one, so they can be split

## Why git worktrees

- Quick primer: one repo, many checked-out branches, shared object store
- Compared with the other options: branch switching in one checkout, separate clones, patch files
- The bare-repo + sibling-worktrees layout I settled on (directory tree)
- Branch naming convention: `agent/<agent-id>/<task>`

## The first thing that breaks: worktree paths in a container

- The worktree's `.git` is a file containing an absolute host path
- Mount it at `/workspace` and you get `fatal: not a git repository`
- Two fixes: mount at the same absolute path vs. `--relative-paths` (git 2.48+, and the old git in my base image)
- Which one I picked and why

## What the agent can see, and what it can only read

- The mount table: worktree rw, shared git dir rw, plus read-only overlays
- Why a writable `.git` gives the agent a way to run code on the host (`config`, `hooks`, the worktree's `gitdir:` pointer)
- Nested read-only bind mounts as the fix
- Short aside: these are the same file-system boundary ideas as the first post, applied to git

## Giving the agent its own identity

- `GIT_AUTHOR_*` / `GIT_COMMITTER_*` in the container environment
- Choosing a name and email (`.invalid` domain vs. a GitHub machine user)
- Provenance trailers via a baked-in `commit-msg` hook (agent, model, session ID)
- Example `git log` output showing my commits next to agent commits
- Open decision for the post: agent as author and committer, or agent as author and me as committer

## Taking away the ability to push

- Layered, not one switch:
  - No credentials mounted (`~/.ssh`, `~/.gitconfig`, credential helpers, `gh`)
  - No SSH agent socket, no `openssh-client` in the image
  - `protocol.allow=never` and `pushInsteadOf` set through `GIT_CONFIG_*` environment variables
  - Network: Squid has no route to GitHub (Pi) / port 22 never allowed
- Demo: the agent tries `git push` and gets a clear failure

## The VS Code catch: Dev Containers forward your credentials by default

- What VS Code does for convenience: copies `.gitconfig`, adds a credential helper back to the host, forwards the SSH agent
- Why that matters more here: Copilot needs `*.github.com` open in the proxy
- The settings that turn it off, plus verifying that they took effect
- `.devcontainer/` lives inside the worktree the agent can edit, so mount it read-only

## Putting it together: a launcher script

- One command: create worktree, work out mount paths, create session ID, start the container
- Running two agents side by side (one compose project per worktree)
- Short screencast or terminal recording?

## Reviewing and shipping agent work

- Review from my own worktree through refs only (`git log main..agent/...`, `git diff main...agent/...`)
- Why I don't run things inside the agent's working tree on the host
- Pushing the agent branch myself and opening a PR
- Cleaning up worktrees and branches

## Gaps that remain

- The agent can still move refs in the shared repo, including `main`
  - Reflog for detection, a host `pre-push` hook, GitHub branch protection as the backstop
- Identity is a convention, not proof: the agent can change its env and claim to be me
  - Signing my own commits on the host as the fix
- Hooks inside the sandbox are advisory (`--no-verify`)
- Alternatives with stronger isolation: separate clone per agent, patch-only outbox, a credential broker

## Tradeoffs and what's next

- Cost: more setup and moving parts. Benefit: parallel agents with history I can audit
- Ties back to the platform-engineering theme: split the capability (commit) from the authority (push)
- Next: a broker that pushes `agent/*` branches only after tests pass? Per-task egress policies?
