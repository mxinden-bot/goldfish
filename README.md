# goldfish

Persistent, cross-session memory for Claude Code.

Each Claude Code session starts with no memory of the last one — like a goldfish.
This repo fixes that. It's included in every Claude Code on the web session so the
agent reads accumulated context, preferences, and rules at startup.

## How it works

- **[`CLAUDE.md`](./CLAUDE.md)** is the memory. Claude Code loads `CLAUDE.md`
  automatically, so anything written there is in context from the first message
  of a new session.
- Future sessions append what they learn, keeping the file high-signal.

Using a different agent tool that reads `AGENTS.md` instead? Point it at the same
content with `ln -s CLAUDE.md AGENTS.md`.
