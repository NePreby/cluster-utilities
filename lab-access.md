# Reusable SSH connections — clusters and the lab workstations

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

## The lab workstations — lkj003 … lkj010 -psy

Seven Tailscale hosts, **key + password auth**. The laptop's `id_ed25519` is in each one's
`~/.ssh/authorized_keys`, so login needs no password (verified 2026-10-24). Password login stays enabled
server-side on purpose, for `ssh-copy-id` and LAN bootstrap. The lab boxes can also ssh to each other
(`bin/fleet-mesh`). Inventory and specs: `fleet/hosts.yaml`; adding a machine: `fleet/README.md`.
Aliases follow the hostname, `lkj0NN-psy` → `labNN`; `lab8` (lkj008-psy, the main research box) also
answers to the old `lab`:

```sshconfig
Host lab8 lab lkj008 lkj008-psy
  HostName 100.86.65.72     # Tailscale IP (tailnet-private)
  User yorguin
  # inherits ControlMaster/ControlPersist from Host *
# … and lab3, lab4, lab6, lab7, lab9, lab10 the same way (IPs in fleet/hosts.yaml)
```

The key logs in with no prompt, and the first `ssh lab8` also opens the multiplexed master that later
`ssh -o BatchMode=yes lab8` calls (and rsync/scp, and agents) reuse. Because it's key auth, BatchMode
calls succeed even when that socket is down.

Then it behaves exactly like a cluster alias:

```bash
ssh lab8                      # interactive; opens the master on first use
ssh -o BatchMode=yes lab8 '<cmd>'   # agent-safe: fails fast instead of hanging if the master is down
rsync -avP <src> lab8:~/dest/ # reuses the same socket
cluster-login lab8            # (optional) the "refresh the master" helper also works here
```

Because there is no Duo on the lab box, `cluster-login lab8` isn't usually needed — a plain `ssh lab8`
opens the master fine. The `-o BatchMode=yes` form is still the right one for scripted/agent calls:
it returns non-zero on an auth problem rather than blocking on a prompt.

Spec: Xeon E3 (8 threads), 32 GB RAM each, Quadro K620/K2200/P2000 — good for light ML (LR) and analysis; use the Alliance
clusters for heavy fits or feature re-extraction.

## Adding any other reusable host

Same recipe: one `Host <alias>` block with `HostName` + `User`; the global `Host *` gives it socket
reuse automatically. For an Alliance cluster you'd also want the Duo-aware `cluster-login <alias>`
refresh flow (see `AGENT.md`); for a keyed host like the lab box, the plain alias is enough.
