---
layout: post
title: "Securing AI Agents on Production Servers"
date: 2026-03-23
tags: [ai-agents, security, linux, devops, claude-code]
---

<div class="tldr">
<strong>TL;DR:</strong> We deployed an autonomous AI agent on a production server and needed to give it real debugging power without the ability to break things. The answer wasn't a new framework or an AI-specific sandbox. It was Unix users, sudo rules, and file permissions. Primitives from the 1970s, still the best tool for the job.
</div>

## What Happened

A production server went down on a Sunday morning. A database table had silently grown to 32GB, filled the disk, and triggered a cascade across half a dozen Docker containers. Redis couldn't persist, the app layer started returning 502s, Postgres entered a crash-recovery loop.

We fixed it by pointing Claude at the server over SSH and letting it run commands directly. It traced the cascade across six containers, found the root cause, and walked us through the fix. Effective. Also terrifying. The account it was using had full privileges on a production server running Docker, PostgreSQL, Redis, and a handful of application services.

That experience was the motivation. We wanted to keep the power of an AI agent that can SSH in and actually debug things, but without the risk of it having the keys to everything.

So we built a proper setup. A [Claude Code SDK](https://docs.anthropic.com/en/docs/claude-code/sdk) bot sitting in a Telegram group chat. Ask it "why is the app returning errors?" and it actually investigates. Runs `docker logs` across containers, queries the database, checks disk usage, traces the cascade, and tells you the root cause. The same debugging workflow, done in 20 seconds, with guardrails this time.

The SDK gives Claude a bash shell. It decides what commands to run. `docker ps`, `docker logs`, `psql` queries, `df -h`. That's what makes it powerful. The question was how to keep that power without giving it the ability to break things.

## Our First Attempt: Filtering Commands

We started with the obvious approach. The Claude Code SDK has a `can_use_tool` callback that lets you inspect commands before execution. We built a blocklist:

```python
DENIED_PATTERNS = [
    "rm ", "docker stop", "docker restart", "docker kill",
    "kill ", "shutdown", "reboot",
    "INSERT", "UPDATE", "DELETE", "DROP", "TRUNCATE",
]
```

It took about five minutes to realize the problems.

The agent could write a Python script to `/tmp` and execute it. The script would run the same blocked commands. Or it could use `curl` against the Docker socket. Or pipe through `sh`. Or use `eval`. The AI is trained to be creative with shell commands. That's the whole point. You can't out-enumerate an LLM's knowledge of bash.

We also hit a practical issue: the SDK's `can_use_tool` callback required streaming mode, which changed the API contract. The approach was fighting the tool instead of working with it.

## The Realization

<div class="callout">
<strong>The key insight:</strong> We were trying to solve a privilege problem in application code. Linux solved this decades ago. The question "how do I let an untrusted process read system state without being able to modify it?" is as old as multi-user Unix. The answer is the same as it's always been: users, groups, permissions.
</div>

## How We Set It Up

Here's exactly what we did. If you're deploying something similar, this is a step-by-step guide.

### Step 1: Create a User With Nothing

```bash
useradd -m -s /bin/bash monitor
```

The key decision: do **not** add this user to the `docker` group. On Linux, the Docker socket is owned by `root:docker`:

```
srw-rw---- 1 root docker 0 Mar 2 12:45 /var/run/docker.sock
```

If you're not in the group, you can't access it. Not through the CLI, not through `curl --unix-socket`, not through Python's `socket` module. The kernel enforces this. There's no workaround short of a privilege escalation exploit.

At this point the agent can run `df` and `free` and not much else.

### Step 2: Grant Read-Only Docker Access via sudo

We wrote a sudoers file at `/etc/sudoers.d/monitor`:

```
Defaults:monitor !targetpw, !requiretty

# Read-only Docker commands
monitor ALL=(root) NOPASSWD: /usr/bin/docker ps *
monitor ALL=(root) NOPASSWD: /usr/bin/docker ps
monitor ALL=(root) NOPASSWD: /usr/bin/docker logs *
monitor ALL=(root) NOPASSWD: /usr/bin/docker inspect *
monitor ALL=(root) NOPASSWD: /usr/bin/docker stats *
monitor ALL=(root) NOPASSWD: /usr/bin/docker stats
monitor ALL=(root) NOPASSWD: /usr/bin/docker info
monitor ALL=(root) NOPASSWD: /usr/bin/docker system df *
monitor ALL=(root) NOPASSWD: /usr/bin/docker system df

# Database access: only psql, only as a read-only user, only on the db container
monitor ALL=(root) NOPASSWD: /usr/bin/docker exec my-postgres psql -U monitor_readonly *

# System logs
monitor ALL=(root) NOPASSWD: /usr/bin/journalctl *
```

`sudo docker ps` works. `sudo docker stop` asks for a password the agent doesn't have.

The `docker exec` rule is the most interesting. It allows exactly one pattern: `psql` with a specific read-only user on a specific container. The agent can't exec into a bash shell. It can't use a privileged database user. Sudo matches arguments exactly.

Two practical notes. `!targetpw` may be needed if your distro defaults to requiring the target user's password for any sudo. Without it, even whitelisted commands get blocked. `!requiretty` lets the agent run sudo from a non-interactive systemd process.

### Step 3: Create a Read-Only Database Role

```sql
CREATE USER monitor_readonly WITH PASSWORD '...';
GRANT CONNECT ON DATABASE nocodb, n8n, aapl_db TO monitor_readonly;

-- For each database and schema:
GRANT USAGE ON SCHEMA public TO monitor_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO monitor_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO monitor_readonly;
```

This is defense in depth. The sudoers rules already restrict which database user the agent can connect as. But even if that somehow got bypassed, PostgreSQL itself would reject any write operation. Two independent layers enforcing the same constraint.

### Step 4: Lock Down Sensitive Files

```bash
chmod 600 /home/deploy/app/.env
chmod 600 /home/deploy/secrets/db_password.txt
chmod 600 /home/deploy/docker-compose.override.yml  # if it contains secrets
```

The agent can read `docker-compose.yml` and config files that help with debugging. It can't read `.env` files with API keys, tokens, or database passwords. The rule is simple: if a file contains secrets, it should be `chmod 600` and owned by the service user.

We also made the agent's home directory read-only:

```bash
chown -R root:root /home/monitor
chmod 555 /home/monitor
```

The agent can write to `/tmp` (the Claude Code SDK needs it for session files), but `/tmp` is on tmpfs. It can't fill the main disk.

## Testing the Boundaries

We ran every bypass we could think of as the `monitor` user:

| What we tried | What happened |
|---|---|
| `sudo docker stop redis` | "a password is required" |
| `sudo docker restart redis` | "a password is required" |
| `sudo docker exec postgres bash` | "a password is required" |
| `sudo docker exec postgres psql -U admin` | "a password is required" |
| `docker ps` (no sudo) | "permission denied" (not in docker group) |
| `curl --unix-socket /var/run/docker.sock` | "Permission denied" |
| `cat /home/deploy/.env` | "Permission denied" |
| Write a Python script, run it | Runs as `monitor`. Same restrictions apply. |

The Python bypass question was the most interesting one to think through. Yes, the agent can write a Python script to `/tmp` and execute it. The script runs as the `monitor` user. `subprocess.run(["sudo", "docker", "stop", "redis"])` still gets blocked by sudoers. The Docker socket is still inaccessible. File permissions still apply. The script has the same walls around it that bash does, because the walls are at the OS level, not the language level.

## What the Agent Can Actually Do

After all this restriction, the agent is still genuinely useful. Here's a real interaction:

> **User:** @agent why is the app returning errors?
>
> The agent runs `sudo docker logs --tail 50 app-container`. Sees 502 errors pointing to an upstream service. Checks that service's logs. Finds Redis connection failures. Checks Redis logs. Sees "No space left on device." Runs `df -h /var`.
>
> **Agent:** "/var is 100% full. Redis can't save snapshots, which causes the cache layer to fail, which breaks the app. The cascade is: disk full, Redis fails, upstream returns 502, app can't serve requests."

That's a multi-container debugging chain traced in 20 seconds. The agent followed the same path a human would: start at the symptom, follow the errors upstream, find the root cause. It just did it faster because it doesn't need to think about which `docker logs` command to type next.

## The Architecture

```
Chat Interface (Telegram, Slack, etc.)
       |
       v
Agent Process (systemd, User=monitor)
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

The systemd unit runs the process as the `monitor` user. The Claude Code SDK spawns bash subprocesses that inherit the same UID. Every command the agent runs, regardless of how creatively it constructs it, hits the same OS-level restrictions.

## What We'd Do Differently

If the agent needed to run inside a Docker container instead of directly on the host, we'd use a [Docker socket proxy](https://github.com/Tecnativa/docker-socket-proxy) instead of sudoers. Same principle, different mechanism. The proxy filters Docker API calls at the HTTP level, allowing GET requests (read) and blocking POST/DELETE (write).

We chose the host approach because the agent needs to read real disk usage, check system memory, and access `journalctl`. A container sees its own isolated view of most of these. You can work around it with volume mounts, but at that point you're fighting containers instead of using them.

## The Broader Point

<p class="lead">None of this is new technology. Users, groups, permissions, sudo rules, database roles. These primitives are from the 1970s.</p>

That's the point. When everyone is reaching for new AI-specific sandboxing frameworks, the boring Unix security model is sitting right there, battle-tested for fifty years, enforced by the kernel, and understood by every sysadmin who's ever locked down a shared server.

The instinct with AI agents is to solve security problems in AI-specific ways: prompt engineering ("please don't delete anything"), application-level command filtering, custom sandboxing layers. These all operate at the wrong level. The agent can reason around prompts. It can find creative bypasses for filters. But it can't reason its way past `chmod 600` or a sudoers rule. The kernel doesn't care how clever you are.

<div class="callout">
<strong>The litmus test:</strong> If the agent found a creative new way to invoke a command, would your security still hold? If it depends on pattern matching, no. If it depends on the kernel, yes.
</div>

The recipe is simple. Start with a user that has nothing. Grant access upward, one command at a time, through sudo rules and database roles. Lock secrets with file permissions. Let the agent be as creative as it wants within those walls. The walls are made of the same material that's been keeping untrusted processes in check since the PDP-11.
