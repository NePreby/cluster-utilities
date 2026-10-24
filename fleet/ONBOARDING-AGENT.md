# Handoff: put a lab machine on the tailnet (for an agent)

Goal: a lab workstation reachable from the laptop as `ssh <alias>` over Tailscale, with key login and the
SSH session menu (plain fish / tmux / herdr). Background and the full procedure: `fleet/README.md`.

## Starting state of each new machine (2026-10)
- CachyOS, upgraded recently. The owner's login shell is **fish**.
- Tailscale **installed but not logged in**. The owner is signed in to the browser on that machine, as the
  tailnet account `yjmantilla@` (tailnet `tail2e782e.ts.net`).
- Claude Code and `gh` are logged in.
- Already on the tailnet: `zephyrus-m16` (laptop), `lkj008-psy` (ssh alias `lab`).

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
- Do one machine end to end, including the reboot check, before starting the next.

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
   Then make SSH the default for `gh` and clone:
   ```bash
   gh config set git_protocol ssh
   gh repo clone yjmantilla/cluster-utilities ~/Projects/cluster-utilities
   ```
2. **Owner, in a normal terminal on that machine** (needs sudo and a browser click):
   ```bash
   bash ~/Projects/cluster-utilities/bin/fleet-onboard              # keep the current hostname
   bash ~/Projects/cluster-utilities/bin/fleet-onboard lkj0XX-psy   # or rename first (= Tailscale name)
   ```
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
   ```
4. **Owner, in the [Tailscale admin console](https://login.tailscale.com/admin/machines):** on the new
   machine, `…` → **Disable key expiry**. Otherwise it drops off the tailnet after about 180 days.

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
8. Commit the `hosts.yaml` change (and nothing else), and ask before pushing.

## If something goes wrong
| Symptom | What to do |
|---|---|
| Clone fails: `ssh_askpass` / "Host key verification failed" | github.com isn't in known_hosts. Do the host-key part of A1 |
| "Refusing over SSH" | It's being run over ssh. Run it at the machine's keyboard |
| "reboot, then rerun" | A kernel upgrade means tun can't load until reboot. Reboot, run the same command |
| "DNS broke after the switch: restoring …" | The script already put the old resolv.conf back and exited 1. Report the output; don't retry blindly |
| "stub listener isn't answering" | resolv.conf was left alone; MagicDNS may not work. `ssh <ip>` still works. Report `resolvectl status` |
| "waiting for admin approval" / IP placeholder | Approve the machine in the admin console, then `tailscale ip -4` |
| Machine missing from `tailscale status` on the laptop | On the machine, `tailscale status`: it must show `yjmantilla@`. If it shows another account, the owner runs `sudo tailscale logout && sudo tailscale up` and logs in as yjmantilla@ |
| `ssh -o BatchMode=yes` → Permission denied | Key not installed: redo B3. Don't change sshd settings |
| `fleet-audit` exits 2 | hosts.yaml format; the message names the line |
