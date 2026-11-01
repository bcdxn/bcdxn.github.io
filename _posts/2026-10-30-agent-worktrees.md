---
title: "Protecting My Git Repos From My Coding Agents"
date: 2026-11-01 07:30:00 -0500
categories: [blog]
tags: [ai, agents, security, docker, vscode, git, localllm]
excerpt: "How to sandbox your agents to preserve git history and audit changes."
image:
  path: /assets/images/banners/2026-11-01.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-11-01.jpg
---

[![banner](/assets/images/banners/2026-11-01.jpg)](https://unsplash.com/photos/emerald-bay-state-park-aerial-view-UWQP2mh5YJI)

This is a follow-up to [Sandboxing My Coding Agents](/blog/2026/09/03/agent-sandboxing.html) and the [Dev Container follow-up](/blog/2026/09/05/minimal-vscode-devcontainer.html). Those posts boxed my agents in: they can only see the folder I mount and only reach the network destinations I allow-list. That setup has been working well, but it did not adequately protect my git repositories (both local and remote).

In fact, I didn't really isolate my git repository at all. I simply mounted the full project workspace as writable, giving the agent full access to my local repository. With the coarse network permissions required by GitHub Copilot and the [automatic injection of git credentials](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials) by VS Code Dev Containers, this gives the agent all of the permissions required to make some big mistakes, e.g. overwrite history or outright delete repository contents with force pushes.

I could alter the mount setup slightly or mount a copy of the working directory, but I can still only run one agent per checkout, and when it's done I'm reviewing a wall of changes with no record of how it got there. I want my agents to work more like a teammate on a branch: small commits I can step through, with several of them working at the same time.

That means giving the agent git, and git is where the sandbox gets leaky. To commit, the agent needs write access to the repository itself, and the repository holds every branch (mine included), the repo config, and the hooks that run on my machine.

Here's what I'm after:

- **Parallel agents, isolated from each other and from me.** Each agent gets its own branch and working directory. File system permissions, not instructions in a prompt, should stop it from touching another agent's branch or my own work, whether by accident or on purpose.
- **Commit histories.** The agent commits as it goes, so I get small, reviewable steps instead of one giant diff.
- **Clear attribution.** Every commit says an agent wrote it: which agent and which model.
- **No pushing.** The sandbox never has my git credentials. Nothing reaches GitHub unless I push it myself.

There are two concepts I'm going to use to get my preferred level of isolation (in addition to those covered in my previous post):

1. Git worktrees, which is a relatively popular approach to parallelizing git workflows (for humans and agents)
2. Committing and pushing are different kinds of actions. A commit is a local write to a folder on my machine, which I can control with mounts on my sandbox Docker container. A push is a credentialed call to GitHub, which I can control by never handing the agent those credentials. Split the two, and the agent can do useful work with git without being able to publish anything.

## Why Git Worktrees

A [git worktree](https://git-scm.com/docs/git-worktree) is an extra checkout of the same repository. Each worktree is its own folder with file system isolation so agents don't step on each other.

Agent tooling has started building this in (Claude Code, for example, has [built-in worktree support](https://code.claude.com/docs/en/worktrees) for running parallel sessions, and the GitHub Copilot app offers [similar functionality](https://github.blog/ai-and-ml/github-copilot/what-are-git-worktrees-and-why-should-i-use-them/)). The interesting part for my use case is that a worktree is just a folder, which makes it something I can mount into a sandboxed Docker container and/or VS Code Dev Container.

I could get similar isolation with a separate clone per agent, but worktrees are faster to create, share a single object store, and keep every branch visible from my own checkout for review.

### The Layout

I settled on a bare repository with every worktree as a sibling folder:

```
~/Repos/myproj/
  .bare/              # the repository itself (no files checked out)
  .git                # file containing "gitdir: ./.bare"
  me-main/            # my worktree on branch main
  ag-pi-feat-x/       # agent worktree on branch agent/pi/feat/x
  ag-copilot-bug-12/  # agent worktree on branch agent/copilot/bug/12
```

A bare clone is just the repository database with no working files, which is how git servers store repos. Since the top-level folder isn't a checkout itself, I can keep the worktree folders in the same parent directory and they never show up as untracked files. My `me-main/` checkout is no more special than the agents' checkouts; it's just a simple naming convention to help me organize in-flight work for human (and adversarial agent) reviews.

Setting it up:

```sh
mkdir -p ~/Repos/myproj && cd ~/Repos/myproj
git clone --bare git@github.com:bcdxn/myproj.git .bare
echo "gitdir: ./.bare" > .git

# bare clones don't set a fetch refspec, so origin/* refs won't exist without this
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin

# Adding my worktree to track main for my own work and orchestration
git worktree add me-main main
```

**<i class="fas fa-lightbulb"></i>** The `.git` file is a convenience that lets me run commands like `git fetch` and `git worktree list` from the top-level folder.
{: .notice--info}

### Creating an Agent Worktree

Agent branches follow an `agent/<agent-id>/<task>` naming convention, e.g. `agent/pi/feat/x` or `agent/copilot/bug/12`. I shorten the associated folder name just to try to minimize my file paths (this is personal preference).

```sh
cd ~/Repos/myproj
git fetch origin
git worktree add -b agent/pi/feat/x ag-pi-feat-x origin/main
```

And when the work is merged (or abandoned):

```sh
git worktree remove ag-pi-feat-x
git branch -D agent/pi/feat/x
```

## Mounting the Workspace in the Sandbox

Now the goal is to mount the worktree directory so that our agent can do its work and we have strong guarantees that it cannot alter any other files, including other worktrees and our overall git repository history.

One simple approach is to mount the git directory as read-only:

```sh
WT=~/Repos/myproj/ag-pi-feat-x
GCD=~/Repos/myproj/.bare

docker run --rm -ti \
  -v "$WT:$WT:rw" \
  -v "$GCD:$GCD:ro" \
  -w "$WT" \
  agent-sbx:latest
```

**<i class="fas fa-lightbulb"></i>** The file paths in the container now need to mimic the paths on the host because git uses internal pointer files with hardcoded absolute paths. We'll need those pointers to work inside the Docker container to enable git commands to view repository history, etc.
{: .notice--info}

With this approach, agents can read and write files and run read-only git commands to view repository history.

The problem with this approach is that the agent cannot commit to its worktree; it can only make changes within the working directory. Depending on your workflow this may be fine. But my goal was to explicitly allow the agent to commit incremental progress locally.

This admittedly makes our mounting scheme more complicated. I have to do a specific dance of adding and nesting directories with read/write and read-only permissions.

```sh
WT=~/Repos/myproj/ag-pi-feat-x
GCD=~/Repos/myproj/.bare
# the directory name of the worktree where the agent is working
NAME=ag-pi-feat-x

docker run --rm -ti \
  # mount the overall worktree dir as read/write
  -v "$WT:$WT:rw" \
  # prevent agent from pointing to a different git directory
  -v "$WT/.git:$WT/.git:ro" \
  # mount full git repository then progressively lock it down;
  # in part this is what allows the agent to make commits
  -v "$GCD:$GCD:rw" \
  # prevent alterations to the repository git config
  -v "$GCD/config:$GCD/config:ro" \
  # prevent alterations to the repository git hooks
  -v "$GCD/hooks:$GCD/hooks:ro" \
  # prevent alterations to other worktrees' state
  -v "$GCD/worktrees:$GCD/worktrees:ro" \
  # allow alterations to the agent-specific worktree history
  # the write permissions needed to make commits
  -v "$GCD/worktrees/$NAME:$GCD/worktrees/$NAME:rw" \
  # set the working directory to our worktree directory
  -w "$WT" \
  agent-sbx:latest
```

## Agent Visibility and Permissions

Here's the container's view of my machine with those mounts in place:

```
~/Repos/myproj/
  ag-pi-feat-x/         # read/write: the code it's working on
    .git                # read-only: pointer to its git directory
  .bare/                # read/write: objects and branches, so it can commit
    config              # read-only
    hooks/              # read-only
    worktrees/          # read-only: every worktree's git state...
      ag-pi-feat-x/     # ...except its own, which is read/write
```

My `me-main/` checkout, other agents' worktrees, and the rest of my home directory don't exist inside the container. I've also put some extra care into protecting the git directory. This protects my git history from mistakes by the agent and ensures nothing sensitive can be overwritten (e.g. config, hooks, etc. that can ultimately end up in arbitrary code execution).

Specifically, the nested mounts make this work. Docker applies mounts from the shortest path to the longest, so the read-only and read/write mounts layer correctly to enforce proper isolation for the agent.

## Agent Identity in Git Histories

Git reads identity from environment variables before it looks at any config, so the container just sets them:

```sh
GIT_AUTHOR_NAME="pi-agent (qwen3.6-27b)"
GIT_AUTHOR_EMAIL="pi-qwen3.6_27b@agents.invalid"
GIT_COMMITTER_NAME="pi-agent (qwen3.6-27b)"
GIT_COMMITTER_EMAIL="pi-qwen3.6_27b@agents.invalid"
```

The `.invalid` top-level domain is [reserved](https://www.rfc-editor.org/rfc/rfc2606), so the email can never match a real GitHub account and accidentally attribute agent work to a stranger.

Here's what an agent branch looks like from my `me-main/` worktree:

```
$ git log --format='%h %<(24)%an %s' main..agent/pi/feat/x
9c41e2a pi-agent (qwen3.6-27b)   Add tests for retry backoff
5f0a3b8 pi-agent (qwen3.6-27b)   Retry telemetry uploads with exponential backoff
e27d9c4 pi-agent (qwen3.6-27b)   Extract upload client into its own package
```

And a single commit in full:

```
$ git log -1 --format=fuller 5f0a3b8
commit 5f0a3b8d1e6c0a92f7b4e3c18d2a9f6e0b71c45d
Author:     pi-agent (qwen3.6-27b) <pi-qwen3.6_27b@agents.invalid>
AuthorDate: Sun Nov 1 09:41:27 2026 -0500
Commit:     pi-agent (qwen3.6-27b) <pi-qwen3.6_27b@agents.invalid>
CommitDate: Sun Nov 1 09:41:27 2026 -0500

    Retry telemetry uploads with exponential backoff
```

### Author vs. committer

Every commit records two identities: the **author** (who wrote the change) and the **committer** (who created that commit in the repo). Inside the sandbox the agent is both. Then, I can land the commits with a rebase making myself the committer and leave the agent as the author.

## Taking Away the Ability to Push

Committing only needs the local files I've already mounted. Pushing needs two more things: credentials and a route to GitHub. The sandbox gets neither.

**No credentials.** The container only sees the worktree and the git directory. My `~/.ssh`, `~/.gitconfig`, and keychain-backed credential helpers were never mounted, and there's no SSH agent socket.

**No route.** For a CLI harness like Pi, the [Squid allow-list from the first post](/blog/2026/09/03/agent-sandboxing.html#adding-a-controlled-egress-proxy) already does the work: it only allows my LAN model host, so `github.com` is denied, and port 22 was never in `Safe_ports`. I just make sure I don't add any GitHub domains to it.

As a final layer, git itself is told not to talk to remotes at all. Config set through environment variables overrides the repo config, and `protocol.allow=never` blocks every remote transport while leaving local commands like `commit`, `log`, and `diff` alone:

```sh
GIT_CONFIG_COUNT=1
GIT_CONFIG_KEY_0=protocol.allow
GIT_CONFIG_VALUE_0=never
```

Inside the container:

```
$ git push origin agent/pi/feat/x
fatal: transport 'ssh' not allowed
```

The agent doesn't need to fetch either, since I fetch on the host before creating the worktree.

VS Code is a different story. Copilot needs GitHub domains open in the proxy, and Dev Containers are helpful in ways that work against us here. That's next.

## Sandboxing the GitHub Copilot Harness

Ironically, protecting my git repository takes more steps when the harness is GitHub Copilot (whether via the CLI or the VS Code integration). Two things work against me:

1. **The proxy has to allow GitHub.** Copilot talks to `*.github.com` and friends, so the "no route" layer from the previous section is gone. If a usable credential ends up in the container, it can reach GitHub.
2. **Credentials tend to show up on their own.** Copilot itself needs a GitHub token, and VS Code Dev Containers [share your git credentials with the container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials) by default: they copy your `~/.gitconfig`, install a credential helper that forwards back to your host's keychain, and forward your SSH agent.

So for Copilot, the job is keeping push-capable credentials out, and leaning on `protocol.allow=never` (same environment variables as before) as the layer that still works when the network is open.

### Copilot CLI

The CLI needs a token, so give it one that can't push. A [fine-grained personal access token](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli) with only the **Copilot Requests** permission and no repository access, passed in as `GH_TOKEN`, lets Copilot work without being able to write to any repo.

### VS Code Dev Containers

Turn off the credential sharing in my _user_ settings (on the host, so it applies to every Dev Container):

```json
{
  "dev.containers.copyGitConfig": false,
  "dev.containers.gitCredentialHelperConfigLocation": "none"
}
```

Then, in `devcontainer.json`, blank the SSH agent socket and stop VS Code from offering its own GitHub sign-in for git:

```json
"remoteEnv": {
  "SSH_AUTH_SOCK": "",
  "GIT_TERMINAL_PROMPT": "0"
},
"customizations": {
  "vscode": {
    "settings": {
      "github.gitAuthentication": false,
      "git.terminalAuthentication": false
    }
  }
}
```

One more gotcha: `.devcontainer/` lives _inside_ the worktree, which the agent can edit. The agent could add a mount to its own `docker-compose.yaml` that takes effect the next time I rebuild. Mounting that folder read-only closes the loophole:

```yaml
- ${WT}/.devcontainer:${WT}/.devcontainer:ro
```

Finally, check from a terminal inside the container that nothing slipped through:

```sh
git config --show-origin --list | grep -i credential   # nothing
test -S "$SSH_AUTH_SOCK" && echo "ssh agent leaked"     # nothing
git push origin HEAD                                    # fatal: transport 'ssh' not allowed
```

**<i class="fas fa-lightbulb"></i>** If VS Code ever pops up a prompt asking to let an extension sign in to GitHub with repository access while I'm in one of these containers, the answer is no.
{: .notice--info}

## Scaling Beyond my Local Developer Environment

There's currently a lot of manual setup that relies on me doing the right thing. At scale I'd keep the same concepts and centralize enforcement:

- **Give agents their own identity.** Short-lived, narrowly scoped credentials per run (e.g. a GitHub App token for one repo) instead of borrowing a developer's. "Can't push" becomes "can only push to `agent/*` branches," enforced by branch rules on the server, and attribution becomes provable instead of a convention.
- **Run autonomous agents somewhere disposable.** Fresh, hardened, remote sandboxes that clone, work, open a PR, and disappear. Use ephemeral environments instead of my local Docker engine.
- **Centralize the chokepoints.** An egress gateway instead of a per-container Squid config, an AI gateway so model keys never enter the sandbox, and a package proxy for supply-chain policy.
- **Keep a human on the merge.** Required reviews and protected branches mean an agent can propose anything and land nothing on its own.
- **Make the safe path the easy path.** A supported Dev Container image and managed editor settings for the interactive, local work that won't go away.

Agents are going to keep getting more capable, and that's exactly why the boundary around them should be something we design into our development practices rather than something we hope for.
