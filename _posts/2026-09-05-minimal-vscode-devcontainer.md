---
title: "VS Code Copilot Sandbox Dev Containers"
date: 2026-09-05 06:30:00 -0400
categories: [blog]
tags: [ai, agents, security, docker, vscode, llm, localllm]
excerpt: "A quick follow-up on sandboxing coding agents"
image:
  path: /assets/images/banners/2026-09-03.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-09-03.jpg
---

This is a quick follow-up to [my previous post](/blog/2026/09/03/agent-sandboxing.html) on sandboxing coding agents. After playing around with the concept for a couple of days, I've landed on a set of configurations that let me stay productive — and I thought it was worth sharing.

I had to customize the base image to make it more useful for VS Code Copilot's agentic harness, and open up additional paths through my Squid proxy (specific to my project) so the agent could install packages. In an enterprise setting, allowing a single domain — such as a custom Artifactory instance that proxies requests to other package registries — would significantly reduce the blast radius (though that approach has had [its own share of problems](https://jfrog.com/blog/jfrog-and-openai-collaboration-on-zero-day-security-findings/)).

### A Minimal Dockerfile for a Go-Focused Agent Container

```dockerfile
FROM mcr.microsoft.com/devcontainers/go:latest


RUN apt-get update && apt-get install -y --no-install-recommends \
    ripgrep \
    fd-find \
    fq \
    yq \
    fzf \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN go install golang.org/x/tools/cmd/goimports@latest && \
    go install honnef.co/go/tools/cmd/staticcheck@latest

WORKDIR /workspace
```

### Minimal Squid Proxy Configuration

The following configuration is tailored to my project, as it includes additional allow-listed domains for its Go dependencies:

```conf
# Network interface ports
http_port 3128

# Add LAN IP hosting model to the allow list
acl allowed_lan dst 192.168.1.173
# copilot dependencies
acl allowed_vscode dstdomain .github.com
acl allowed_vscode dstdomain .githubusercontent.com
acl allowed_vscode dstdomain .githubcopilot.com
acl allowed_vscode dstdomain .githubassets.com
acl allowed_vscode dstdomain .default.exp-tas.com
# go packages
acl allowed_go_pkg dstdomain .proxy.golang.org
acl allowed_go_pkg dstdomain .sum.golang.org
acl allowed_go_pkg dstdomain .pkg.go.dev
acl allowed_go_pkg dstdomain .go.googlesource.com
## sqlite go packages
acl allowed_go_pkg dstdomain .modernc.org
acl allowed_go_pkg dstdomain .storage.googleapis.com
acl allowed_go_pkg dstdomain .gitlab.com
acl allowed_go_pkg dstdomain .bitbucket.org

# Standard default rules for system safety
acl Safe_ports port 80          # http
acl Safe_ports port 8080        # non-root http (llama.cpp)
acl Safe_ports port 443         # https
acl CONNECT method CONNECT

# Deny access to unintended ports
http_access deny !Safe_ports

# Explicitly allow access to allow-listed domains/IPs
http_access allow allowed_lan
http_access allow allowed_vscode
http_access allow allowed_go_pkg

# Block everything else
http_access deny all
```

### Tying It All Together: the `.devcontainer/docker-compose.yaml`

```yaml
networks:
  # Internal network with NO direct internet gateway for agent
  agent_sandbox:
    internal: true
  # Egress network giving Squid proxy internet access
  egress_net:
    driver: bridge

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
    # build dev container image using custom Dockerfile
    build: .
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
      - /home/vscode:exec,uid=1000,gid=1000
      - /go:exec
      - /root/.cache/go-build
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

> **<i class="fas fa-lightbulb"></i>** Note the extra `tmpfs` mounts I've added for Go's build cache and module directories.
> {: .notice--info}

Good luck containing your agents!
