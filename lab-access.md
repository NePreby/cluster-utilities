# Reusable SSH connections — clusters and the lab workstation

The same **ControlMaster multiplexing** that makes the Alliance clusters reusable (authenticate
once, reuse one socket) works for any host, including a Tailscale lab machine. The trick is a global
`Host *` block in `~/.ssh/config` plus one short alias per host.

## The global block (already in `~/.ssh/config`)

```sshconfig
Host *
  ServerAliveInterval 300
  ControlMaster auto
  ControlPersist yes
  ControlPath ~/.ssh/cm-%r@%h:%p
```

Every alias below inherits this, so the first connection opens a background master socket and every
later `ssh`/`rsync`/`scp` to that host reuses it — no re-auth, instant reconnects.

## The lab workstation — LKJ008-psy

Tailscale host, **password auth**: the SSH key was removed so a second person can reach it via a
Tailscale share (keys were causing trouble there), so no `ssh-copy-id` / keyless login. Alias:

```sshconfig
Host lab lkj008 lkj008-psy
  HostName 100.86.65.72     # Tailscale IP (tailnet-private)
  User yorguin
  # inherits ControlMaster/ControlPersist from Host *
```

First `ssh lab` prompts for the password and opens the multiplexed master; after that,
`ssh -o BatchMode=yes lab` (and rsync/scp, and agents) reuse the socket with no re-prompt.

Then it behaves exactly like a cluster alias:

```bash
ssh lab                       # interactive; opens the master on first use
ssh -o BatchMode=yes lab '<cmd>'   # agent-safe: fails fast instead of hanging if the master is down
rsync -avP <src> lab:~/dest/  # reuses the same socket
cluster-login lab             # (optional) the "refresh the master" helper also works here
```

Because there is no Duo on the lab box, `cluster-login lab` isn't usually needed — a plain `ssh lab`
opens the master fine. The `-o BatchMode=yes` form is still the right one for scripted/agent calls:
it returns non-zero on an auth problem rather than blocking on a prompt.

Spec (as of migration): 8 cores, 31 GB RAM — good for light ML (LR) and analysis; use the Alliance
clusters for heavy fits or feature re-extraction.

## Adding any other reusable host

Same recipe: one `Host <alias>` block with `HostName` + `User`; the global `Host *` gives it socket
reuse automatically. For an Alliance cluster you'd also want the Duo-aware `cluster-login <alias>`
refresh flow (see `AGENT.md`); for a keyed host like the lab box, the plain alias is enough.
