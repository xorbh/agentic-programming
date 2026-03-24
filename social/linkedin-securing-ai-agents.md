If you're running AI agents on servers, the security question is really just "how do you let an untrusted process read system state without being able to modify it?" Unix answered this decades ago with users, groups, and permissions. We tried the modern approach first with command blocklists and pattern matching on dangerous strings, but the agent wrote a Python script to /tmp and bypassed all of it.

What actually worked was the stuff we've been layering over for years. Unix users and groups, sudo rules granting read-only access to specific commands, read-only Postgres roles, file permissions on secrets. No framework, no AI-specific tooling. Just the OS doing what it was designed to do.

We've spent a long time abstracting away OS and database-level permissions behind API layers and application-level access control. Those abstractions made sense for humans building web apps. But AI agents operate closer to the metal. They run shell commands and talk to databases directly, which means the old primitives become load-bearing again in a way they haven't been for most application developers in a while.

A sudoers rule and chmod 600 are harder to get around than any allowlist you'll write in application code, because the kernel doesn't care how creative the agent is.

Full writeup with configs and bypass testing: https://xorbh.github.io/agentic-programming/2026/03/23/securing-ai-agents-on-production-servers.html
