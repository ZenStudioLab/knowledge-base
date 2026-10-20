---
title: "Pi intercom: send messages between Pi agents on two machines over SSH"
created: "2026-10-20"
updated: "2026-10-20"
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

Status (2026-10-20): planned, not set up. pi-intercom 0.17.1 and its local message broker already run on both machines, and the laptop reaches the workbox over SSH with a key. Every step under "Steps" is still to do.

## How it works

Each machine keeps its own broker, which only accepts local connections. To send to `worker@workbox`, a laptop Pi session:

1. finds `workbox` in Herdr's saved SSH machines (`herdr machine list`);
2. looks up an agent named `worker` on that machine, with a 5-second timeout;
3. runs `ssh <saved machine target> "<remoteCommand> relay --envelope-stdin --json"` and pipes the message in, with a 15-second timeout.

- The name before `@` is the Herdr agent or pane name, not the Pi session name (`cross-machine-discovery.ts:66`). A full session UUID also works.
- Version 0.17.1 relays `send` only. `ask`, `replyTo`, attachments, retries and lifecycle actions don't cross machines.
- The receiver shows `From <name>@<machine> · unverified cross-machine`. The sender is whoever the SSH caller says it is.

## What the upstream guide misses

- **`node` over SSH:** `cli.mjs` starts with `#!/usr/bin/env node`, and node comes from nvm, which a non-interactive SSH command doesn't load. Even the full path to `cli.mjs` fails with `/usr/bin/env: 'node': No such file or directory`. A wrapper that calls nvm's node directly fixes it (step 1).
- **Agent names:** a remote agent can only be addressed by name after it has been given one (step 4). Until then, only its full session UUID works.
- **Herdr versions:** `herdr machine add` sets up Herdr on the remote side, so match versions first. On 2026-10-20 the laptop had 0.9.1 and the workbox 0.9.3. It's not confirmed that the mismatch breaks anything.

## Steps (laptop → workbox, one way)

1. On the workbox, create `~/.local/bin/pi-intercom` and make it executable:

   ```sh
   #!/bin/sh
   exec "$HOME/.nvm/versions/node/v22.23.3/bin/node" "$HOME/.pi/agent/npm/node_modules/pi-intercom/cli.mjs" "$@"
   ```

   Use the node version the workbox broker runs on, and update the path when nvm's default changes.

2. On the laptop, upgrade Herdr to the workbox version, then:

   ```bash
   herdr machine add --label workbox workbox
   herdr --machine workbox agent list   # should list the workbox agents
   ```

3. On the laptop, create `~/.pi/agent/intercom/config.json`. `remoteCommand` is the wrapper's absolute path on the workbox:

   ```json
   { "crossMachine": { "machineName": "laptop", "remoteCommand": "/home/<user>/.local/bin/pi-intercom" } }
   ```

   Restart the laptop's Pi sessions so they load it.

4. On the workbox, name one agent: `herdr agent rename <pane> worker`. Herdr drops the name when that agent exits.

5. Test from a laptop Pi session:

   ```
   intercom({ action: "send", to: "worker@workbox", message: "ping" })
   ```

   The workbox session should show `From …@laptop · unverified cross-machine`.

6. Before relying on it, make sure the workbox accepts key-based SSH logins only. Anyone who can run the relay over SSH can make workbox agents act on their messages. Setting `inboundTrigger` to `"replies"` or `"never"` in the receiver's `config.json` limits which messages start a turn.

**The other direction** (replies, and `/handover` back to the laptop) needs the reverse route: the workbox's public key in the laptop's `authorized_keys`, an SSH host entry for the laptop on the workbox, and steps 1–5 mirrored.

## When the SSH route changes

Neither tool knows about routes. They only use the alias `workbox`, and `~/.ssh/config` picks the route with `Match exec` blocks each time a connection opens: home LAN, then the phone hotspot, then a Cloudflare Access tunnel.

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
- **Tunnel login expiry:** when the Cloudflare Access login runs out, `cloudflared` wants a browser login, which an agent can't do. Off the LAN, sends fail until someone runs `ssh workbox` once by hand.
- **Untested:** the tunnel route had not worked end to end as of 2026-10-20.
- **Keep `Match` lines stable:** reserve the workbox's IP on the router and the hotspot.

## Related

- [Herdr: keep Pi sessions alive through out-of-memory kills](../runbooks/herdr-pi-sessions-survive-oom.md). Its `pi-<id>` agent names also work as intercom addresses, e.g. `pi-<id>@laptop`.
