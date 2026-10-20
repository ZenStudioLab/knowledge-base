---
title: "Herdr: keep Pi sessions alive through out-of-memory kills"
created: "2026-10-20"
updated: "2026-10-20"
status: "active"
visibility: "private"
tags: [herdr, pi, systemd, oom, ubuntu, ghostty, runbook]
project: ""
publish_candidate: false
---

# Runbook: Herdr — Keep Pi Sessions Alive Through Out-of-Memory Kills

## Purpose

Two procedures for Ubuntu with Herdr running Pi coding-agent sessions in panes:

- **A. Move the Herdr server into its own systemd service**, so one out-of-memory (OOM) kill takes one process instead of every pane.
- **B. Bring Pi sessions back into their tabs** after Herdr lost them.

Status (2026-10-20): B was done once and verified. For A, the unit file and switch script are installed on the laptop and the service is enabled, so the switch happens at the next boot. Not verified until that boot.

### What happened on 2026-10-20

- 06:30: a `pi` process ran the laptop out of memory (`kernel: pi invoked oom-killer`). Ghostty's process group had peaked at 20 GB.
- Herdr's server had been started from Ghostty, so it lived in Ghostty's systemd group (`snap.ghostty.ghostty-*.scope`).
- systemd's default `DefaultOOMPolicy=stop` stops the **whole group** when the kernel kills one process in it. The Herdr server and all 20 Pi sessions died together.
- The panes exited before Herdr's last save, so `~/.config/herdr/session.json` kept the tabs but no Pi session records. The next start restored 27 plain shells, and nothing resumed.

```
Ghostty window                  ← one systemd group
 └ herdr (client)
    └ herdr server
       └ panes: Pi sessions, Claude, shells
```

## Preconditions

- Herdr with the Pi integration installed: `herdr integration status` shows `pi: current`. The integration reports each Pi session file to Herdr, which `herdr agent list` shows as `agent_session.value`.
- `jq` installed.
- For B: you know which pane held which session (see "Finding the sessions" below).

## Steps

### A. Run the Herdr server as its own service

1. Install the unit at `~/.config/systemd/user/herdr-server.service`:

   ```ini
   [Unit]
   Description=Herdr server in its own process group, outside the terminal

   [Service]
   ExecStart=%h/.local/bin/herdr server
   # systemd's default ("stop") ends the whole group when the kernel kills one
   # process for memory, which takes every pane and Pi session with it.
   OOMPolicy=continue
   Restart=on-failure

   [Install]
   WantedBy=default.target
   ```

   No extra `Environment=` is needed. Panes run interactive bash, which sources `~/.bashrc`, and that file sets `PI_*`, `NVM_*`, `JAVA_HOME`, Homebrew and the rest.

2. Install the switch script at `~/.local/bin/herdr-switch-to-service` and make it executable. It records pane → Pi session, stops the old server, starts the service, waits 20 s for Herdr's own resume, then restarts any Pi session that did not come back, stopping if free memory drops below 5 GB:

   ```bash
   #!/usr/bin/env bash
   set -u

   HERDR="$HOME/.local/bin/herdr"
   UNIT=herdr-server.service
   MIN_AVAIL_KB=$((5 * 1024 * 1024))
   STATE="$HOME/.local/state/herdr-switch"
   STAMP=$(date +%Y%m%dT%H%M%S)
   MAP="$STATE/pi-sessions-$STAMP.tsv"
   mkdir -p "$STATE"
   exec >"$STATE/report-$STAMP.txt" 2>&1

   if systemctl --user is-active --quiet "$UNIT"; then
     echo "Nothing to do: $UNIT is already running."
     exit 0
   fi

   "$HERDR" agent list | jq -r '.result.agents[]
     | select(.agent == "pi" and .agent_session.kind == "path")
     | [.pane_id, .agent_session.value] | @tsv' >"$MAP"
   echo "Recorded $(wc -l <"$MAP") Pi sessions in $MAP"

   "$HERDR" server stop
   for _ in $(seq 1 30); do
     pgrep -x -f "$HERDR server" >/dev/null || break
     sleep 1
   done
   if pgrep -x -f "$HERDR server" >/dev/null; then
     echo "ABORT: the old Herdr server is still running."
     exit 1
   fi

   systemctl --user daemon-reload
   systemctl --user enable --now "$UNIT"
   for _ in $(seq 1 60); do
     "$HERDR" agent list >/dev/null 2>&1 && break
     sleep 1
   done
   pid=$(systemctl --user show -p MainPID --value "$UNIT")
   if grep -q "$UNIT" "/proc/$pid/cgroup" 2>/dev/null; then
     echo "OK: Herdr server pid $pid runs in $UNIT"
   else
     echo "WARN: Herdr server is not running inside $UNIT"
   fi

   sleep 20
   while IFS=$'\t' read -r pane file; do
     id=$(basename "$file" .jsonl)
     short=${id##*_}
     short=${short:0:8}
     if "$HERDR" agent list | grep -q "$id"; then
       "$HERDR" agent rename "$pane" "pi-$short" >/dev/null 2>&1
       echo "auto-resumed  $pane pi-$short"
       continue
     fi
     avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
     if [ "$avail" -lt "$MIN_AVAIL_KB" ]; then
       echo "STOP: only $((avail / 1024)) MB free; remaining sessions are listed in $MAP"
       break
     fi
     if out=$("$HERDR" agent start "pi-$short" --kind pi --pane "$pane" --timeout 60000 -- --session "$file" 2>&1) \
       && ! grep -q '"error"' <<<"$out"; then
       echo "resumed       $pane pi-$short"
     else
       echo "FAILED        $pane pi-$short: ${out:0:200}"
     fi
   done <"$MAP"
   echo "Done. Free memory: $(($(awk '/MemAvailable/ {print $2}' /proc/meminfo) / 1024)) MB"
   ```

3. Choose when to switch:

   - **At the next boot (no disruption, used on 2026-10-20):**

     ```bash
     systemctl --user daemon-reload
     systemctl --user enable herdr-server.service   # no --now
     ```

     The running server is untouched. At the next login, systemd starts Herdr from the service before any terminal opens. Herdr restores the tabs and resumes the Pi sessions it has records for. One sign this works: after a clean reboot on 2026-10-20, Pi sessions were running again and receiving intercom messages within five minutes of Herdr's restore. After that boot, open `herdr` in the terminal and check the steps under Verification.

   - **Right now:** pick a quiet moment, because stopping the server closes **every** pane, including any agent mid-task and the agent running these steps. Run the switch from outside Herdr's process tree, so it survives the server stop:

     ```bash
     systemd-run --user --unit=herdr-switch --collect herdr-switch-to-service
     ```

     Then reopen `herdr` in the terminal. It should attach to the server the service started.

### B. Bring Pi sessions back into their tabs

Use this when Herdr restored the tabs as plain shells.

1. Check whether Herdr still has session records: `grep -c agent_session ~/.config/herdr/session.json`. If they're there, a Herdr restart resumes them on its own (`resume_agents_on_restore` defaults to true).
2. Otherwise, build the pane → session map (see "Finding the sessions").
3. Resume one session per pane, one at a time, and watch free memory:

   ```bash
   herdr agent start pi-<first 8 of session id> --kind pi --pane <pane id> -- --session <path to session .jsonl>
   ```

   The pane must be at a shell prompt. `agent start` waits until Pi is idle and ready. The name `pi-<id>` shows the session ID in Herdr and also works as a pi-intercom address (`pi-<id>@<machine>`).

#### Finding the sessions

- **Tab labels:** Herdr tab labels follow Pi session renames, and the `tab.rename` events in `~/.config/herdr/herdr-server.log` line up with Pi `session_info` name changes. So a labelled tab maps to the Pi session whose latest `session_info` name equals the label.
- **Unnamed tabs** (numeric labels): use an earlier `herdr agent list` snapshot (`pane_id` → `agent_session.value`). Pane IDs survive a Herdr restore.
- **Skip** any session that is already live in `herdr agent list`. Two Pi processes on one session file corrupt it.

## Verification

- `herdr agent list | jq -r '.result.agents[] | select(.agent=="pi") | [.pane_id,.name,.agent_status] | @tsv'` shows every resumed pane with its `pi-<id>` name.
- `grep -c agent_session ~/.config/herdr/session.json` matches the number of running agents. Herdr saved the records.
- After A: `cat /proc/$(systemctl --user show -p MainPID --value herdr-server.service)/cgroup` ends in `herdr-server.service`, not a Ghostty scope.

## Rollback

For A:

```bash
systemctl --user disable --now herdr-server.service
herdr   # from the terminal; the client starts its own server again
```

Then use B if any Pi session did not come back.

## Notes

- **What A doesn't cover:** Ubuntu's `systemd-oomd` watches `user@1000.service` (`ManagedOOMMemoryPressure=kill`, 50% pressure limit). Under sustained memory pressure it kills a whole group, and the Herdr service would be the biggest one. A blocks the kernel-OOM failure seen on 2026-10-20, not every memory crash.
- **Memory is the real limit:** about 19 idle Pi sessions use about 4.5 GB, and each grows once it starts working. Close finished sessions.
- **Resumed sessions can start working right away:** a session that was mid-task can pick its work back up on its own. On 2026-10-20 one resumed review-queue session restarted its browser bridge and resent a request without being prompted.
- **Names don't follow the session:** the `pi-<id>` name is fixed when the agent starts and is dropped when the agent exits. If the pane later switches to a new session (`/new`), the name goes stale. Fix it with `herdr agent rename <pane> pi-<new id>`.
- **Unverified as of 2026-10-20:**
  - that the Ghostty Herdr client attaches to a server it did not start;
  - that Herdr auto-resumes Pi sessions after a clean `herdr server stop`. That's why the script records the map first and resumes anything missing.
