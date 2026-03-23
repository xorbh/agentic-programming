---
layout: post
title: "Securing AI Agents on Production Servers"
date: 2026-03-23
tags: [ai-agents, security, linux, devops, claude-code]
---

<div class="tldr">
<strong>TL;DR:</strong> When deploying an autonomous AI agent on a production server, don't try to filter commands in application code. Instead, run the agent as a restricted Linux user with surgical sudo rules, a read-only database role, and hardened file permissions. The OS is your security boundary, not your code.
</div>

## The Setup

We run a manufacturing automation server with Telegram bots, NocoDB, n8n, PostgreSQL, and Redis. When something breaks at 2am, someone has to SSH in, check container logs, query the database, and trace the failure chain.

We wanted an AI agent that could do this. Talk to it in a Telegram group, and it investigates. "Why are the bots returning auth errors?" It checks NocoDB, finds Redis is in a restart loop, traces it to a corrupted AOF file from a disk-full incident. Real diagnosis, not canned checks.

The tool: Claude Code SDK. It gives the agent a bash shell and lets Claude decide what commands to run. The same way a human would debug. `docker ps`, `docker logs`, `psql` queries, `df -h`. The agent is autonomous. You ask a question, it runs commands, reads the output, runs more commands, and comes back with an answer.

The problem: a full bash shell on a production server. What could go wrong?

## Why Application-Level Filtering Doesn't Work

The obvious approach is to intercept commands before execution. Maintain a blocklist. Check for `rm`, `docker stop`, `kill`, `DROP TABLE`.

This fails for a few reasons.

First, there are too many ways to express the same command. `rm file`, `python3 -c "import os; os.remove('file')"`, `perl -e 'unlink("file")'`, `find . -delete`. You can't enumerate every possible bypass.

Second, the AI is specifically trained to be creative with shell commands. Pipe through `sh`, encode in base64, use `eval`. It's not adversarial. It's just good at bash. The same capability that makes it useful for debugging makes it dangerous if the only barrier is pattern matching.

Third, the Claude Code SDK spawns bash subprocesses. You don't control the shell invocation. The SDK gives Claude a bash tool, and Claude uses it however it sees fit.

## The Right Layer: The Operating System

Linux has been solving this exact problem for decades. Untrusted process needs scoped access to a shared system. The answer is users, groups, and permissions.

The principle: **let the agent run anything it wants, but limit what the user account can do.** The security boundary is the kernel, not your code.

### Layer 1: A User With Nothing

Create a `monitor` user that starts with no special access:

```bash
useradd -m -s /bin/bash monitor
```

Critically, this user is **not** in the `docker` group. On Linux, the Docker socket (`/var/run/docker.sock`) is owned by `root:docker`. If you're not in the group, you can't talk to Docker at all. Not through the CLI, not through curl, not through Python.

```
srw-rw---- 1 root docker 0 Mar 2 12:45 /var/run/docker.sock
```

The agent runs as `monitor`. It literally cannot access Docker. Which means it can't do anything useful yet.

### Layer 2: Surgical sudo

Here's where it gets interesting. We give the user sudo access to exactly the commands it needs, with no password:

```
Defaults:monitor !targetpw, !requiretty

monitor ALL=(root) NOPASSWD: /usr/bin/docker ps *
monitor ALL=(root) NOPASSWD: /usr/bin/docker logs *
monitor ALL=(root) NOPASSWD: /usr/bin/docker inspect *
monitor ALL=(root) NOPASSWD: /usr/bin/docker stats *
monitor ALL=(root) NOPASSWD: /usr/bin/docker info
monitor ALL=(root) NOPASSWD: /usr/bin/docker system df *

monitor ALL=(root) NOPASSWD: /usr/bin/docker exec aapl-postgres psql -U monitor_readonly *

monitor ALL=(root) NOPASSWD: /usr/bin/journalctl *
```

`sudo docker ps` works. `sudo docker stop` asks for a password the agent doesn't have.

The `docker exec` rule is the most interesting. It allows `docker exec aapl-postgres psql -U monitor_readonly ...` and nothing else. The agent can query the database, but only as the read-only user. It can't exec into a bash shell. It can't use a privileged database user. The sudo argument matching is exact.

### Layer 3: Read-Only Database Role

Even if the agent can run SELECT queries, PostgreSQL provides its own enforcement:

```sql
CREATE USER monitor_readonly WITH PASSWORD '...';
GRANT CONNECT ON DATABASE nocodb, n8n, aapl_db TO monitor_readonly;
GRANT USAGE ON SCHEMA public TO monitor_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO monitor_readonly;
```

The agent can explore the entire database. It can check table sizes, query failed workflow executions, examine NocoDB metadata. But `INSERT`, `UPDATE`, `DELETE`, `DROP` all fail at the database level. Two independent enforcement points for the same restriction.

### Layer 4: File Permissions

Sensitive files get locked to their owning user:

```bash
chmod 600 /home/aapl/automation/.env
chmod 600 /home/aapl/automation/secrets/postgres_password.txt
```

The agent can read docker-compose files (useful for debugging). It can't read API keys, database passwords, or bot tokens.

The agent's home directory is read-only (owned by root). It can write to `/tmp` because the Claude Code SDK needs it for temp files, but `/tmp` is on tmpfs (RAM-backed), so it can't fill the main disk.

## Testing It

We tested every bypass we could think of:

```
sudo docker stop redis           → "a password is required"
sudo docker exec postgres bash   → "a password is required"
sudo docker exec postgres psql -U aapl_user  → "a password is required"
cat /home/aapl/automation/.env   → "Permission denied"
docker ps (without sudo)         → "permission denied" (not in docker group)
curl --unix-socket /var/run/docker.sock ...  → "Permission denied"
```

The Python bypass? Write a script to `/tmp`, run it. It still runs as `monitor`. Same permissions apply. `subprocess.run(["sudo", "docker", "stop", "redis"])` still asks for a password. The Docker socket is still inaccessible.

## What the Agent Actually Does

With this setup, the agent is genuinely useful for debugging. Here's a real interaction:

> **User:** @agent why are the bots returning auth errors?
>
> **Agent:** *runs `sudo docker logs --tail 50 aapl-production-bot`*
> *sees "Failed to fetch auth data from NocoDB" and 502 errors*
> *runs `sudo docker logs --tail 20 aapl-nocodb-new`*
> *sees "MISCONF Redis is configured to save RDB snapshots, but can't persist to disk"*
> *runs `sudo docker logs --tail 20 aapl-redis`*
> *sees "No space left on device"*
> *runs `df -h /var`*
>
> "/var is 100% full. Redis can't save snapshots, which causes NocoDB to fail, which breaks bot authentication. The cascade is: disk full -> Redis fails -> NocoDB returns 502 -> bot auth lookup fails -> all users blocked."

That's the exact debugging chain a human would follow, done in 20 seconds.

## The Architecture

```
Telegram Group Chat
       |
       v
Bot Process (systemd, User=monitor)
       |
       +-- Claude Code SDK --> spawns bash as 'monitor'
               |
               |-- sudo docker ps/logs/inspect    (allowed by sudoers)
               |-- sudo docker stop/restart        (blocked, needs password)
               |-- sudo docker exec psql -U monitor_readonly  (allowed)
               |-- sudo docker exec bash           (blocked, needs password)
               |-- df, free, uptime, ps            (allowed, no sudo needed)
               |-- cat .env                        (blocked, file permissions)
               +-- docker ps (no sudo)             (blocked, not in docker group)
```

## When This Approach Applies

This works when:
- The agent needs host-level access (not just API calls)
- You want to give it real investigative capability, not just canned health checks
- The server runs multiple services that interact in complex ways
- You need the AI to trace failures across service boundaries

It doesn't apply when:
- The agent only needs API access (use API keys with scoped permissions instead)
- You can run the agent in a container (use a Docker socket proxy)
- The agent needs write access (this model is fundamentally read-only)

## The Broader Principle

The lesson isn't specific to AI agents. It's the same principle behind any privilege separation: **enforce at the lowest possible layer.**

Application-level checks are suggestions. OS-level permissions are facts. Database roles are facts. File ownership is a fact. When the enforcement layer is the kernel, the only bypass is a kernel exploit.

AI agents make this more important, not less. They're creative, autonomous, and will find paths you didn't anticipate. The path shouldn't matter if the destination is locked.
