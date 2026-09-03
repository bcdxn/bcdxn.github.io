---
title: "Sandboxing My Coding Agents"
date: 2026-09-03 10:00:00 -0400
categories: [blog]
tags: [ai, agents, security, docker, vscode, llm, localllm]
excerpt: "How I started sandboxing my coding agents with Docker and VSCode Dev Containers after watching them wander outside the repo"
image:
  path: /assets/images/banners/2026-09-03.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-09-03.jpg
---

[![sandbox](/assets/images/banners/2026-09-03.jpg)](https://unsplash.com/photos/green-leafed-trees-HrJSHoY9NG4)

This is a learn-as-I-go post. I wanted to put some guardrails around my agents after seeing them get a little too curious.

I was running OpenAI's Luna model in VSCode's agentic harness and noticed the "thinking" steps it was doing while exploring context for a task. It started wandering into other codebases on my machine that had nothing to do with the repo I was working in. That behavior, plus some [recent cyber security news](https://news.ycombinator.com/item?id=49454314) about LLMs and agents making unexpected network calls, made me want to actually sandbox things properly.

My first thought was just run the harness in Docker. Simple enough.

## A basic Docker sandbox for a CLI agent

I started with the Pi coding agent as a test case. Here's the Dockerfile I ended up with:

```dockerfile
FROM node:22-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    bash \
    curl \
    ripgrep \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g --ignore-scripts @earendil-works/pi-coding-agent

RUN useradd -m -s /bin/bash agentuser
USER agentuser
WORKDIR /workspace

ENTRYPOINT ["pi"]
```

Build it:

```sh
docker build -t agent-sbx:latest .
```

First test: can it reach the internet?

```sh
docker run --rm -ti \
  --read-only \
  --security-opt="no-new-privileges:true" \
  --cap-drop=ALL \
  --entrypoint /bin/bash \
  agent-sbx:latest
```

Inside the container we can run:

```sh
curl -I https://google.com
# HTTP/2 301
# ...
```

Yep, it can phone home. That's exactly what I want to control. Let's add full network isolation:

```sh
docker run --rm -ti \
  --network none \
  --read-only \
  --security-opt="no-new-privileges:true" \
  --cap-drop=ALL \
  --entrypoint /bin/bash \
  agent-sbx:latest
```

```sh
curl -I https://google.com
# curl: (6) Could not resolve host: google.com
```

Good. Now let's run it for real with a few more hardening options:

- `--network none`: no interfaces at all. No outbound HTTP, no package installs, no exfiltration.
- `--read-only`: root filesystem is read-only. Can't drop binaries or tamper with node_modules.
- `--tmpfs /tmp:exec`: RAM-backed scratch space that disappears when the container stops. `:exec` lets tools write temp scripts.
- `--tmpfs /home/agentuser/.pi`: fresh in-memory state per run, no cross-run leakage.
- `-v "$(pwd)/workspace:workspace:rw"`: the one intentional write path back to my machine.

```sh
docker run --rm -ti \
  --network none \
  --read-only \
  --tmpfs /tmp:exec \
  --tmpfs /home/agentuser/.pi \
  -v "$(pwd)/workspace:workspace:rw" \
  --security-opt="no-new-privileges:true" \
  --cap-drop=ALL \
  agent-sbx:latest
```

Now I can confirm that the agent can only see the file system within the Docker container.

```sh
ls /Users/bdxn
# ls: /Users/bdxn: No such file or directory
```

This is great. We've locked down the file system and the network successfully. Maybe too successfully... now it can't call my local LLM (or any externally hosted models for that matter). I need an allow-list.

## Adding a controlled egress proxy

I wanted the agent to be able to reach my local model endpoint on my LAN and maybe perform a search via an API like [Brave](https://brave.com/search/api/) or [Tavily](https://www.tavily.com) if I decide to allow it. The simplest way is a containerized egress proxy with ACLs. I picked [Squid](https://www.squid-cache.org) because it has great documentation and it's battle tested.

I opted for the prebuilt image as I didn't need any additional customization beyond the standard configuration file:

```sh
docker pull ubuntu/squid:latest
```

Here's my first pass at a basic `squid.conf` that acts as an explicit firewall:

```conf
http_port 3128

acl allowed_lan dst 192.168.1.173
acl allowed_wan dstdomain .google.com

acl Safe_ports port 80
acl Safe_ports port 8080
acl Safe_ports port 443
acl CONNECT method CONNECT

http_access deny !Safe_ports

http_access allow allowed_lan
http_access allow allowed_wan

http_access deny all
```

> **<i class="fas fa-lightbulb"></i>** Note that my local LLM is exposed on my local area network at `192.168.1.173:8080` at the moment, hence the addition of the IP address and non standard port.
> {: .notice--info}

> **<i class="fas fa-warning"></i>** Now only specified IPs and domains are allowed. Everthing else is denied.
> {: .notice--info}

Then I can use docker-compose to tie the container in which the agent will run with the squid container that will act as the egress proxy enforcing the ACLs:

```yaml
version: "3.8"

networks:
  agent_sandbox:
    internal: true
  egress_net:
    driver: bridge

services:
  egress-proxy:
    image: ubuntu/squid:latest
    volumes:
      - ./proxy/squid.conf:/etc/squid/squid.conf:ro
    networks:
      - agent_sandbox
      - egress_net

  sandbox-agent:
    build: ./agent-sbx-workspace
    image: agent-sbx
    stdin_open: true
    tty: true
    read_only: true
    security_opt:
      - "no-new-privileges:true"
    cap_drop:
      - ALL
    tmpfs:
      - /tmp:exec
      - /home/agentuser/.pi
    volumes:
      - ./workspace:/workspace:rw
    networks:
      - agent_sandbox
    environment:
      - HTTP_PROXY=http://egress-proxy:3128
      - HTTPS_PROXY=http://egress-proxy:3128
      - http_proxy=http://egress-proxy:3128
      - https_proxy=http://egress-proxy:3128
```

Now:

```sh
docker compose build
docker compose run --rm sandbox-agent
```

Inside the container:

```sh
curl -I https://google.com
# allowed

curl -I http://192.168.1.173:8080
# allowed

curl -I https://nytimes.com
# blocked
```

I've confirmed the agent can only talk to what I explicitly allow, both on my local filesystem and network.

## Doing the same for VSCode Copilot

Ultimately I want to continue to use VSCode's GitHub Copilot integration; I really like using Copilot inside the editor as a _copilot_, not the main pilot. Sandboxing VSCode uses the same concept as the CLI-based harness approach shown above. Instead of starting with a Dockerfile and building my own image, Microsoft provides _Dev Container images_. In fact VSCode has supported this concept of Dev Containers long before agentic coding was a thing. When it was introduced it was promoted as a way to securely work on code you may not fully trust or to have ephemeral development environments. In my case, I can use it when I may not fully trust my own agents or I just want a bit of extra isolation.

VS Code Dev Containers run your project inside an isolated Docker container that VS Code treats as a remote development environment. The editor launches a lightweight server inside the container that is isolated from the host machine using the protections granted by containerization. The VSCode UI then connects to the server to providing a familiar experience.

The first step is to Install the Dev Containers extension:

[![extension page](/assets/images/2026-09-03-extension-sc.png)](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

then, within your workspace in VSCode create:

```
.devcontainer/
  devcontainer.json
  docker-compose.yaml
  squid.conf
```

The squid config for VSCode needs to be looser. Copilot seems to need access to `.github.com` broadly, not just a partitioned subdomain. This is the minimal allow-list that worked for me with a local LLM:

```conf
http_port 3128

acl allowed_lan dst 192.168.1.173
acl allowed_vscode dstdomain .github.com

acl Safe_ports port 80
acl Safe_ports port 8080
acl Safe_ports port 443
acl CONNECT method CONNECT

http_access deny !Safe_ports

http_access allow allowed_lan
http_access allow allowed_vscode

http_access deny all
```

The docker-compose.yaml file is mostly the same, except we swap out the custom image I created for Pi with Microsofts image and we need to give some additional in-memory file system locations that VSCode expects to be able to write to during normal operation:

```yaml
version: "3.8"

networks:
  # Internal network with NO direct internet gateway for agent
  agent_sandbox:
    internal: true
  # Egress network giving Squid proxy internet access
  egress_net:
    driver: bridge # default; adding clarity

services:
  # Egress Proxy Service (The Gatekeeper)
  egress-proxy:
    image: ubuntu/squid:latest
    volumes:
      - ./squid.conf:/etc/squid/squid.conf:ro
    networks:
      - agent_sandbox
      - egress_net
    restart: always

  # Sandboxed Agent
  agent-sandbox:
    image: mcr.microsoft.com/devcontainers/base:ubuntu # <---- UPDATED FOR VSCODE
    stdin_open: true
    tty: true
    read_only: true
    security_opt:
      - "no-new-privileges:true"
    cap_drop:
      - ALL
    tmpfs:
      - /tmp:exec
      - /var/tmp
      - /home/vscode:exec,uid=1000,gid=1000 # <---- UPDATED FOR VSCODE
    volumes:
      - ..:/workspace:rw
    networks:
      - agent_sandbox
    environment:
      # Force all HTTP/HTTPS traffic through the proxy
      - HTTP_PROXY=http://egress-proxy:3128
      - HTTPS_PROXY=http://egress-proxy:3128
      - http_proxy=http://egress-proxy:3128
      - https_proxy=http://egress-proxy:3128
    # keep container alive for vscode connection
    command: ["sleep", "infinity"]
```

Lastly, the `devcontainer.json` file tells VSCode how to start in Dev Container mode.

```json
{
  "name": "Sandboxed Dev Container (Copilot Allowed)",
  "dockerComposeFile": "docker-compose.yaml",
  "service": "agent-sandbox",
  "workspaceFolder": "/workspace",

  // Inject proxy environment variables into VS Code Server process
  "remoteEnv": {
    "HTTP_PROXY": "http://egress-proxy:3128",
    "HTTPS_PROXY": "http://egress-proxy:3128",
    "http_proxy": "http://egress-proxy:3128",
    "https_proxy": "http://egress-proxy:3128"
  },

  // Automatically pre-install GitHub Copilot inside the sandbox
  "customizations": {
    "vscode": {
      "settings": {
        "http.proxy": "http://proxy:3128",
        "http.proxyStrictSSL": true
      },
      "extensions": ["github.copilot", "github.copilot-chat"]
    }
  },

  // Keep root OS read-only and enforce default user
  "remoteUser": "vscode"
}
```

The Dev Container runs the editor backend inside a container with the same read-only + tmpfs + proxy setup, and mounts only the workspace folder I want it to see.

```
$ docker ps
CONTAINER ID   IMAGE                                         COMMAND                  ...   PORTS      NAMES
0642bedb5e47   mcr.microsoft.com/devcontainers/base:ubuntu   "/bin/sh -c 'echo Co…"   ...   agent-sandbox_devcontainer-agent-sandbox-1
ba3b7a1aec14   ubuntu/squid:latest                           "entrypoint.sh -f /e…"   ...   3128/tcp   agent-sandbox_devcontainer-egress-proxy-1
```

It works but it's far from perfect. Copilot still needs some "phone home" capabilities even when I'm using a local model, which means opening up GitHub domains. There are more advanced ways to filter that traffic but Squid ACLs are the easy first step.

## Tradeoffs

An agent restricted this much is less useful by definition. It can't just `npm install` or `go get` whatever it wants; I have to do it manually when needed. Package managers are a risky attack vector anyway, as recent [supply chain attacks](https://aws.amazon.com/blogs/security/amazon-identifies-north-korean-hacker-group-behind-open-source-supply-chain-attacks/) have shown.

For hobby projects this feels right. I get isolation for my local file system and I can limit blanket network access, with explicit allow-lists for the things I actually want. For higher risk environments you'd want more: vette tools, sophisticated AI gateways with full access control authorizing agents' actions.

Balancing restrictive governance and utility is messy in practice. This setup gives me enough peace of mind that my agent isn't wandering around my machine or making surprise network calls, while still being able to use my local LLM.

Next step for me is tightening the GitHub allow-list further and experimenting with per-task egress policies instead of a static list.
