# mnemoverse

A mixin kit that gives a Claude Code sandbox persistent memory from
[Mnemoverse](https://mnemoverse.com/docs/api/mcp-server), a hosted memory
service for AI agents reached over MCP. The kit installs
[`@mnemoverse/mcp-memory-server`](https://github.com/mnemoverse/mcp-memory-server)
(MIT, pinned to 0.10.1) from the npm registry and registers it with Claude
Code at user scope. Memories written in the sandbox are stored under the
user's Mnemoverse account and are readable from every other tool connected to
it (Claude, ChatGPT, Cursor, VS Code, Gemini CLI or any MCP client); memories
written elsewhere are readable in the sandbox.

The Mnemoverse API key never enters the container. Inside the microVM
`MNEMOVERSE_API_KEY` holds the `proxy-managed` placeholder; on every request to
`core.mnemoverse.com` the sbx proxy rewrites the `X-Api-Key` header with the
real key sourced from the host, and denies egress to anything outside the
kit's allowlist.

This kit is **Claude Code-specific**: the registration step writes Claude
Code's own user-level MCP config, so the kit declares `requires: agent: claude`
and the engine rejects composing it onto another agent. The server itself
runs under any MCP client; a kit for another agent needs a registration step
in that agent's config format.

## Prerequisites

Create a Mnemoverse API key at
[console.mnemoverse.com](https://console.mnemoverse.com) (free tier, no card;
the key starts with `mk_live_`) and store it once in Docker Sandboxes'
host-side secret store:

```console
sbx secret set mnemoverse
```

## Usage

The primary form is the published OCI artifact on Docker Hub:

```console
sbx run --kit "docker.io/sbx/mnemoverse-kit:latest" claude
```

Or target this repo directly over git:

```console
sbx run --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=mnemoverse" claude
```

Or use a local clone:

```console
sbx run --kit ./mnemoverse/ claude
```

Inside the session, `/mcp` lists the `mnemoverse` server with its ten tools:
`memory_write`, `memory_read`, `memory_list_recent`, `memory_feedback`,
`memory_stats`, `memory_create_room`, `memory_invite_to_room`,
`memory_join_room`, `memory_list_rooms`, `vault_list`. A quick check: ask the
agent to remember something with `memory_write`, then open a new sandbox with
the same kit and ask for it back.

## How auth works

The kit declares one `mnemoverse` credential with one inject rule:
`core.mnemoverse.com`, header `X-Api-Key`, format `%s`. That is the header the
server sends (its API answers a missing one with "Missing API key. Send
X-Api-Key header."), so the proxy substitutes the value in place and the
server code is unchanged. `MNEMOVERSE_API_URL` is set to the server's default,
`https://core.mnemoverse.com/api/v1`, so the host the key travels to is visible
next to the inject rule; the server refuses to attach the key to a plain
`http://` address that is not loopback.

The credential is `required: true`. The server starts and lists its tools
without a key, but every tool call is refused, so a sandbox created without a
binding would carry a memory kit that cannot remember anything.

## Design notes

- **Pinned version (`@0.10.1`)**: the repo's style note asks for pinned
  installs where possible. Nothing outside this kit tracks the server's
  version, and the server freezes its tool list per released version, so a
  pin gives reproducible sandboxes at no cost. Bump the version in
  `spec.yaml` to move.
- **`npm install -g` at create time, not `npx` at session start**: the
  download happens once, and launching the server later needs no registry
  access. The install runs as root because the global prefix is root-owned
  (same as the `kernel` kit); nothing under `/home/agent` is touched by it.
- **Registration at install time, user scope, absolute path**: the entry is
  written before the interactive session reads its config (the same reasoning
  the `claude` kit gives for its own gateway registration), lands in the
  user-level config so it is visible from every workspace, and names the
  binary by absolute path so it does not depend on `PATH` at launch. The key
  is passed as the `proxy-managed` sentinel literal, the same value the
  engine sets in the container env and the same literal the `claude` kit
  writes into `settings.json`.
- **Allowlist**: `registry.npmjs.org` and `*.npmjs.org` for the install,
  `core.mnemoverse.com` for the server, plus the Ubuntu and Docker apt hosts
  because `requires.agent: claude` inherits the claude kit's background
  `apt-get update` (as in `claude-mem`). `mcp.mnemoverse.com` and
  `auth.mnemoverse.com` (Mnemoverse's remote MCP endpoint with a browser
  sign-in) are deliberately absent: this kit runs the local stdio server
  against the REST API instead, because a browser sign-in has nowhere to land
  inside a sandbox and the host-side secret store already covers the key.
- **No local state, no ports**: the server keeps nothing on disk; memories
  live under the Mnemoverse account. What each tool sends is listed in the
  server's README under "Privacy Policy".

## Debugging

```console
sbx exec <sandbox> -- claude mcp list
sbx exec <sandbox> -- claude mcp get mnemoverse
sbx exec <sandbox> -- npm ls -g @mnemoverse/mcp-memory-server
sbx policy log <sandbox>
```

## Cleanup

```console
sbx secret rm -g --service mnemoverse
```
