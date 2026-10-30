---
title: "Pi intercom: send messages between Pi agents on two machines over SSH"
created: "2026-10-20"
updated: "2026-10-30"
status: "active"
visibility: "private"
tags: [pi, pi-intercom, herdr, ssh, cross-machine, setup]
source: "pi-intercom 0.17.1 README and source (cross-machine-transport.ts, cross-machine-discovery.ts); Herdr 0.9.1 built-in docs"
project: ""
publish_candidate: false
---

# Pi Intercom: Send Messages Between Pi Agents on Two Machines Over SSH

## Summary

Goal: Pi agents on the laptop and on the workbox send each other messages with pi-intercom. Here "laptop" and "workbox" are both the SSH host aliases and the Herdr machine labels.

Status (2026-10-30): laptop → workbox works. A ping from a laptop Pi session reached the workbox agent `worker` and showed `From …@laptop · unverified cross-machine`. The reverse direction (replies and `/handover` back to the laptop) is not set up.

## How it works

Each machine keeps its own broker, which only accepts local connections. To send to `worker@workbox`, a laptop Pi session:

1. finds `workbox` in Herdr's saved SSH machines (`herdr machine list`);
2. looks up an agent named `worker` on that machine, with a 5-second timeout;
3. runs `ssh <saved machine target> "<remoteCommand> relay --envelope-stdin --json"` and pipes the message in, with a 15-second timeout.

- The name before `@` is the Herdr agent or pane name, not the Pi session name (`cross-machine-discovery.ts:66`). A full session UUID also works.
- Version 0.17.1 relays `send` only. `ask`, `replyTo`, attachments, retries and lifecycle actions don't cross machines.
- The receiver shows `From <name>@<machine> · unverified cross-machine`. The sender is whoever the SSH caller says it is.

## What the upstream guide misses

- **`node` over SSH:** `cli.mjs` starts with `#!/usr/bin/env node`, and a non-interactive SSH command has no node on its `PATH`. Even the full path to `cli.mjs` fails with `/usr/bin/env: 'node': No such file or directory`. A wrapper that calls node by absolute path fixes it (step 1).
- **Agent names:** a remote agent can only be addressed by name after it has been given one (step 4). Until then, only its full session UUID works.
- **Saved machine in the sidebar:** `herdr machine add` makes every open Herdr client connect to the workbox and list it in its sidebar. Intercom needs the saved entry to exist and be enabled, but not an open pane. A separate terminal running `herdr --remote workbox` works alongside it.
- **Herdr versions:** `herdr machine add` prepares Herdr on the remote side, so match versions first. On 2026-10-29 the laptop had 0.9.1 and the workbox 0.9.5 (the current stable release). `herdr update` refuses to run inside a Herdr session, so detach first. It checks the download against the SHA-256 in the official `herdr.dev` release manifest.

## Steps (laptop → workbox, one way)

Done: steps 1 to 6 (step 6 already held). Verified 2026-10-30.

1. On the workbox, create `~/.local/bin/pi-intercom` and make it executable:

   ```sh
   #!/bin/sh
   exec "$HOME/.nvm/versions/node/v22.23.3/bin/node" "$HOME/.pi/agent/npm/node_modules/pi-intercom/cli.mjs" "$@"
   ```

   Done. The workbox has no nvm. Pi ships its own node, and `~/.local/share/pi-node/current` is a link to the current version, so the path above survives node upgrades:

   ```sh
   #!/bin/sh
   exec "$HOME/.local/share/pi-node/current/bin/node" "$HOME/.pi/agent/npm/node_modules/pi-intercom/cli.mjs" "$@"
   ```

2. Done. On the laptop, detach from Herdr and run `herdr update` in a plain terminal so it matches the workbox version (0.9.5 on 2026-10-29), then:

   ```bash
   herdr machine add --label workbox workbox
   herdr --machine workbox agent list   # should list the workbox agents
   ```

3. Done. On the laptop, create `~/.pi/agent/intercom/config.json`. `remoteCommand` is the wrapper's absolute path on the workbox:

   ```json
   { "crossMachine": { "machineName": "laptop", "remoteCommand": "/home/<user>/.local/bin/pi-intercom" } }
   ```

   Restart the laptop's Pi sessions so they load it.

4. Done (from the laptop: `herdr --machine workbox agent rename <pane> worker`). On the workbox, name one agent: `herdr agent rename <pane> worker`. Herdr drops the name when that agent exits.

5. Done. Test from a laptop Pi session that was started after step 3:

   ```
   intercom({ action: "send", to: "worker@workbox", message: "ping" })
   ```

   The workbox session should show `From …@laptop · unverified cross-machine`.

6. Before relying on it, make sure the workbox accepts key-based SSH logins only. Anyone who can run the relay over SSH can make workbox agents act on their messages. Setting `inboundTrigger` to `"replies"` or `"never"` in the receiver's `config.json` limits which messages start a turn.

**The other direction** (replies, and `/handover` back to the laptop) needs the reverse route: the workbox's public key in the laptop's `authorized_keys`, an SSH host entry for the laptop on the workbox, and steps 1–5 mirrored.

## When the SSH route changes

Neither tool knows about routes. They only use the alias `workbox`, and `~/.ssh/config` picks the route each time a connection opens: the workbox is looked up by its SSH host key on whatever local network the laptop is on (home Wi-Fi or the phone hotspot), and otherwise the alias falls through to a private address that only exists inside Cloudflare Zero Trust, reached through WARP. The off-network route and its first-error diagnosis are in the workspace guide `docs/guides/workbox-remote-access-over-warp.md`.

- **Herdr:** its generated SSH config includes `~/.ssh/config` first, so the route rules apply.
- **pi-intercom:** the relay is plain `ssh <target>` (`cross-machine-transport.ts`).
- **Host key:** `HostKeyAlias workbox` gives every route one `known_hosts` entry, so Herdr's `StrictHostKeyChecking=yes` doesn't fail when the route switches.

| Change | What to redo |
|---|---|
| Laptop moves between home Wi-Fi, hotspot and outside | Nothing |
| Workbox gets a new IP on a known network | Edit one `Match` line in `~/.ssh/config` |
| A new network | Add one `Match` block, or let it fall through to the tunnel |
| Herdr or intercom pointed at a different alias | Re-add the Herdr machine, and every `name@machine` address changes. Avoid this: use one alias everywhere |

Things that can still go wrong:

- **Right after a network switch:** Herdr reuses one open SSH connection (`ControlMaster`, `ControlPersist`). When the network changes, that connection dies, and SSH can take about a minute to notice (`ServerAliveInterval 15`, `ServerAliveCountMax 4`). Sends fail until then. Setting `ServerAliveInterval 5` and `ServerAliveCountMax 2` on the alias in `~/.ssh/config` shortens it, because Herdr reads that file first.
- **WARP off or not enrolled:** off the home network, sends time out until `warp-cli status` shows Connected. Enrolling needs a one-time browser login, which an agent can't do alone; see the workspace guide above.
- **Verified:** on 2026-10-29, `ssh workbox` worked through WARP from outside the workbox's network.
- **Keep `Match` lines stable:** reserve the workbox's IP on the router and the hotspot.

## Related

- [Herdr: keep Pi sessions alive through out-of-memory kills](../runbooks/herdr-pi-sessions-survive-oom.md). Its `pi-<id>` agent names also work as intercom addresses, e.g. `pi-<id>@laptop`.
