# Handoff: put a lab machine on the tailnet (for an agent)

Goal: a lab workstation reachable from the laptop as `ssh <alias>` over Tailscale, with key login and the
SSH session menu (plain fish / tmux / herdr). Background and the full procedure: `fleet/README.md`.

Before the repo is cloned, read this file with
`gh api repos/yjmantilla/cluster-utilities/contents/fleet/ONBOARDING-AGENT.md -H 'Accept: application/vnd.github.raw'`.

## Starting state of each new machine (2026-10)
- CachyOS, upgraded recently. The owner's login shell is **fish**.
- KDE Plasma, which by default **suspends after 15 min idle**. A suspended machine drops off the tailnet
  and nobody can wake it remotely, so A1 and A2 turn that off (and the reboot check in B7 tests it).
- Tailscale is installed. It may not be logged in, or it may be **logged in to another tailnet** (lkj010-psy
  was). The owner is signed in to the browser on that machine, as the tailnet account `yjmantilla@`
  (tailnet `tail2e782e.ts.net`).
- Claude Code and `gh` are logged in. git has no identity set yet.
- Already on the tailnet: `zephyrus-m16` (laptop), `lkj008-psy` (ssh alias `lab`), `lkj010-psy` (`lab2`).
  Progress and the remaining machines are tracked in the "onboard the remaining lab workstations" issue.

## Hard rules
- **Never run `fleet-onboard` over SSH.** It restarts NetworkManager and tailscaled; it refuses anyway
  unless `--force` is passed. Don't pass `--force`.
- **sudo, the Tailscale login and the admin console are the owner's steps.** You can't type a password
  or click a browser link. Ask the owner to run those commands in a normal terminal on that machine.
- **Never put a Tailscale auth key on a command line** or in a file you create. `--authkey` prompts for it
  hidden, if the owner wants that route. The default (a browser login) needs no key.
- **Password ssh login stays enabled on purpose.** Don't run `bin/harden-sshd-keyonly`, and don't edit
  sshd_config.
- Remote shells are fish. Send scripts as `ssh -o BatchMode=yes <alias> bash -s <<'EOF' … EOF`.
- Do one machine end to end, including the reboot check, before starting the next. An agent on the new
  machine does side A only and queues side B as an issue (A5), but the next machine still waits until
  that issue's reboot check has passed.

## A. On the new machine (agent running there, or the owner by hand)

1. Get the repo over SSH (`gh` is logged in). An agent can't answer SSH's "trust this host key?" prompt
   (it fails with `ssh_askpass … Host key verification failed`), so first add GitHub's host keys from
   `gh api meta`, which goes over authenticated HTTPS:
   ```bash
   mkdir -p ~/.ssh && chmod 700 ~/.ssh
   gh api meta --jq '.ssh_keys[]' | sed 's/^/github.com /' > /tmp/gh_keys
   ssh-keygen -lf /tmp/gh_keys    # ED25519 must be SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU
   cat /tmp/gh_keys >> ~/.ssh/known_hosts && rm /tmp/gh_keys
   ssh -T -o BatchMode=yes git@github.com    # "Hi yjmantilla! You've successfully authenticated…"
   ```
   If that last command says `Permission denied (publickey)`, this machine has no key on GitHub yet:
   `ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519 && gh ssh-key add ~/.ssh/id_ed25519.pub -t <name>`.
   If `gh` lacks the scope, the **owner** runs `gh auth refresh -s admin:public_key` (browser step) first.
   Then make SSH the default for `gh`, set the git identity (otherwise `git commit` fails with
   "Author identity unknown") and clone both infra repos:
   ```bash
   gh config set git_protocol ssh
   git config --global user.name yorguin
   git config --global user.email yjmantilla@gmail.com
   gh repo clone yjmantilla/cluster-utilities ~/Projects/cluster-utilities
   gh repo clone yjmantilla/agent-utilities ~/Projects/agent-utilities
   ```
   **Keep Claude Code transcripts.** By default Claude Code deletes chat transcripts older than 30 days
   (`~/.claude/projects/`); we keep them. Merge `"cleanupPeriodDays": 36500` into `~/.claude/settings.json`
   without dropping existing keys. Don't use `0`: it isn't a "never delete" value.
   ```bash
   python3 -c 'import json,os;p=os.path.expanduser("~/.claude/settings.json");d=json.load(open(p)) if os.path.exists(p) else {};d["cleanupPeriodDays"]=36500;os.makedirs(os.path.dirname(p),exist_ok=True);open(p,"w").write(json.dumps(d,indent=2)+"\n")'
   ```
   **Never sleep on idle (user side, no sudo).** Plasma's power management (powerdevil) runs the 15-min
   idle action per user. Set it to "do nothing" in every power profile (`0` = nothing; `1` sleep,
   `2` hibernate, `8` shut down are what we don't want). It's what System Settings → Power Management →
   "When inactive: Do nothing" writes. The sudo half, which also covers the login screen and other
   users, is in A2.
   ```bash
   for p in AC Battery LowBattery; do
     kwriteconfig6 --file powerdevilrc --group $p --group SuspendAndShutdown --key AutoSuspendAction 0
   done
   qdbus6 org.kde.Solid.PowerManagement /org/kde/Solid/PowerManagement \
     org.kde.Solid.PowerManagement.refreshStatus 2>/dev/null || echo "takes effect at next login"
   ```
2. **Check which tailnet it's on first.** `fleet-onboard` sees any running login as "already logged in"
   and skips `tailscale up`, even when it's another tailnet's login:
   ```bash
   tailscale status --json | python3 -c 'import json,sys;d=json.load(sys.stdin);print(d["BackendState"],(d.get("CurrentTailnet") or {}).get("Name"),(d.get("CurrentTailnet") or {}).get("MagicDNSSuffix"))'
   ```
   `NeedsLogin`, or `Running … tail2e782e.ts.net`, is fine. Anything else: stop and tell the owner which
   tailnet and account it's on and which other devices are on it (`tailscale status`). Leaving a tailnet
   cuts off whoever uses this machine through it. Only after the owner confirms, add
   `sudo tailscale logout` to the commands below.

   **Owner, in a normal terminal on that machine** (needs sudo, and a browser click unless an auth key is
   used). Give the owner the exact commands as one block, with the logout line if needed and the lowercase
   name. Ask whether they have an auth key; if so, use the `--authkey` line:
   ```bash
   bash ~/Projects/cluster-utilities/bin/fleet-onboard              # keep the current hostname
   bash ~/Projects/cluster-utilities/bin/fleet-onboard lkj0XX-psy   # or rename first (= Tailscale name)
   bash ~/Projects/cluster-utilities/bin/fleet-onboard --authkey lkj0XX-psy   # no browser: prompts (hidden) for the key
   ```
   Always add this line to the owner's block. It masks the sleep targets, so nothing (powerdevil, the
   login screen after a reboot, another user's session, logind) can suspend or hibernate the machine:
   ```bash
   sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
   ```
   Masking doesn't block poweroff, so the A1 powerdevil setting is still needed to keep idle from
   choosing "shut down". Also check `IdleAction` is `ignore` (the default):
   `busctl get-property org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager IdleAction`.
   **Auth key (optional, the owner makes it, saves the browser login on each machine).** In the admin
   console, [Settings → Keys](https://login.tailscale.com/admin/settings/keys) → **Generate auth key**:
   - **Reusable: on** when doing several machines in one sitting (off for a single one).
   - **Expiration: 1 day**, the shortest that covers the sitting. Revoke it on the same page when done.
   - **Ephemeral: OFF.** Ephemeral devices are deleted from the tailnet when they go offline, so a lab
     machine would vanish at its first reboot.
   - **Pre-approved: on.** It only matters if device approval is enabled, and skips the
     "waiting for admin approval" step.
   - **Tags: none.** A tagged device belongs to the tag instead of `yjmantilla@`, which changes ACLs and
     breaks the `yjmantilla@` checks here.

   The key (`tskey-auth-…`) is shown once. The owner keeps it in a password manager and pastes it at
   `fleet-onboard`'s hidden prompt. **Never ask for it, and never put it in a command, file, issue or
   chat.** The key's expiry is separate from the device's key expiry: A4 is still needed.
   It runs `pacman -Syu`, installs openssh/fish/bind/…, enables sshd and tailscaled, installs the SSH menu,
   fixes MagicDNS only if needed (and restores the file if DNS would break), then runs `tailscale up`, which
   **prints a login URL**: the owner opens it in that machine's browser and approves. If it says
   "reboot, then rerun", reboot and run the same command again.
3. Verify, and report these values back to the laptop side:
   ```bash
   hostnamectl --static                    # -> <name>
   tailscale ip -4                         # -> 100.x.y.z
   tailscale status | head -3              # this machine listed, user yjmantilla@
   systemctl is-active sshd tailscaled     # active / active
   readlink -f /etc/resolv.conf            # /run/systemd/resolve/stub-resolv.conf (or the script said why not)
   ls ~/.config/fish/conf.d/ssh-session-menu.fish
   whoami                                  # -> ssh user
   ls /usr/lib/modules | grep -x "$(uname -r)"   # running kernel still installed (else: reboot pending)
   systemctl is-enabled sleep.target suspend.target hibernate.target   # masked / masked / masked
   for p in AC Battery LowBattery; do kreadconfig6 --file powerdevilrc --group $p --group SuspendAndShutdown --key AutoSuspendAction; done   # 0 0 0
   lscpu | grep 'Model name'; free -g | awk '/Mem/{print $2" GB"}'; nvidia-smi -L; df -h /home   # spec line
   ```
4. **Owner, in the [Tailscale admin console](https://login.tailscale.com/admin/machines):** on the new
   machine, `…` → **Disable key expiry**. Otherwise it drops off the tailnet after about 180 days.
5. **Queue side B as an issue** instead of doing it from here. Copy the issue "fleet: finish onboarding
   lkj010-psy (lab2) from the laptop": the A3 results, the `~/.ssh/config` block (next free alias:
   `lab3`, …), a ready-to-paste `hosts.yaml` entry with the spec line, the B checks, the reboot check and
   the key-expiry reminder if A4 isn't done yet. Then update the machine table in the "onboard the
   remaining lab workstations" issue.

## B. On the laptop (agent running in `~/code/cluster-utilities` or `~/Projects/cluster-utilities`)

1. Check it's visible: `tailscale status | grep <name>` → online.
2. Add an alias to `~/.ssh/config`. Pick a short alias: `lab2`, `lab3`, … in onboarding order.
   ```sshconfig
   Host lab2 <name>
     HostName 100.x.y.z
     User <user>
   ```
   The global `Host *` ControlMaster block already applies.
3. **Owner** (asks for that machine's password once): `! ssh-copy-id lab2`
4. Verify key login with no prompt, which is what agents use:
   ```bash
   ssh -o BatchMode=yes -o ControlPath=none lab2 'hostname; echo KEY_OK'
   ```
5. The menu only shows in an interactive session, so ask the owner to run `ssh lab2` and confirm the
   `Session: 1) none 2) tmux 3) herdr` prompt appears and falls back to fish after 5 s.
   (`ssh -o BatchMode=yes lab2 cmd` and scp/rsync must NOT show it.)
6. Register it in `fleet/hosts.yaml`. Copy the commented `lab2` template. Keep the format exact: 2-space
   host id, 4-space keys, no tabs, roots must start with `/` or `~`.
   ```yaml
     lab2:
       ssh: lab2
       tailscale: <name>
       ip: 100.x.y.z
       role: research
       spec: <cpu / RAM / GPU from `lscpu`, `free -g`, `nvidia-smi -L`>
       auth: key+password
       roots: ~/Projects
       status: active
   ```
   Then `bin/fleet-audit lab2` must end with `== lab2: audited (…)` and exit 0. A `~/Projects` that doesn't
   exist yet shows `ROOT-MISSING`, which is fine.
7. **Reboot check (owner reboots the new machine).** Afterwards, from the laptop:
   `ssh -o BatchMode=yes lab2 hostname` works, and `ssh <name> hostname` (MagicDNS) works.
   Leave it untouched at the login screen for 20+ minutes and check again: it must still answer
   (that's the no-suspend test).
8. Commit the `hosts.yaml` change (and nothing else), and ask before pushing.

## If something goes wrong
| Symptom | What to do |
|---|---|
| Clone fails: `ssh_askpass` / "Host key verification failed" | github.com isn't in known_hosts. Do the host-key part of A1 |
| `fleet-onboard` says "already logged in" but the IP/tailnet isn't ours | Logged in to another tailnet. See A2: confirm with the owner, `sudo tailscale logout`, rerun |
| `git commit`: "Author identity unknown" | Git identity not set: the `git config --global` lines in A1 |
| "Refusing over SSH" | It's being run over ssh. Run it at the machine's keyboard |
| "reboot, then rerun" | A kernel upgrade means tun can't load until reboot. Reboot, run the same command |
| "DNS broke after the switch: restoring …" | The script already put the old resolv.conf back and exited 1. Report the output; don't retry blindly |
| "stub listener isn't answering" | resolv.conf was left alone; MagicDNS may not work. `ssh <ip>` still works. Report `resolvectl status` |
| "waiting for admin approval" / IP placeholder | Approve the machine in the admin console, then `tailscale ip -4` |
| Machine missing from `tailscale status` on the laptop | On the machine, `tailscale status`: it must show `yjmantilla@`. If it shows another account, the owner runs `sudo tailscale logout && sudo tailscale up` and logs in as yjmantilla@ |
| `ssh -o BatchMode=yes` → Permission denied | Key not installed: redo B3. Don't change sshd settings |
| `fleet-audit` exits 2 | hosts.yaml format; the message names the line |
| Machine goes offline after ~15 min idle, comes back when someone touches it | It's suspending. Redo the powerdevil part of A1 and the `systemctl mask` line in A2; check A3's sleep lines |
