# Fleet: my own machines over Tailscale

The Alliance clusters are covered in `AGENT.md`. This folder covers the machines I own or
command myself: the laptop and the lab workstations, all on the tailnet `tail2e782e.ts.net`.

| File | What it is |
|---|---|
| `hosts.yaml` | Inventory: every machine, its ssh alias, Tailscale name/IP, spec, role |
| `projects.yaml` | Placement: which host owns each project (one host, except shared infra) |
| `../bin/fleet-onboard` | Run **on a new machine** to put it on the tailnet with sshd |
| `../bin/fleet-audit` | Run **from the laptop**: every repo on every host, with anything unsaved or misplaced |

## The rules

1. **One project, one host.** Research goes on the lab machines by default. The laptop keeps
   desktop/personal things. Only infra (`cluster-utilities`, `agent-utilities`,
   `system-utilities`) lives on every host.
2. **Repos live in `~/Projects`.** `~/code` is legacy and being emptied (fleet-audit flags it as LEGACY).
3. **Before deleting a copy**, `fleet-audit` must show it with no NOGIT / NOREMOTE / UNPUSHED /
   DIRTY / UNTRACKED / STASH flags. Ignored files (data, caches, results) are **not** checked, so
   copy those to the owning host first (`rsync -aP --ignore-existing`) and compare sizes.

## Adding a lab machine (CachyOS already installed)

**0. Optional: make an auth key** so the box needs no browser login. In
<https://login.tailscale.com/admin/settings/keys>, generate an auth key (reusable if you're doing
several machines, short expiry). Treat it like a password and don't commit it.

**1. On the new machine, at its keyboard.** Get the script, either by copying it from the
existing lab box over the LAN (sshd + password are on there):
```bash
scp yorguin@<lkj008-lan-ip>:Projects/cluster-utilities/bin/fleet-onboard .
```
or by cloning the repo if the machine has GitHub access. Then:
```bash
./fleet-onboard lkj0XX-psy                         # interactive: open the printed URL anywhere
TS_AUTHKEY=tskey-auth-… ./fleet-onboard lkj0XX-psy # or with the key from step 0
```
It sets the hostname, installs and enables `sshd` + `tailscaled`, applies the CachyOS MagicDNS fix
(the resolv.conf symlink), and joins the tailnet. It leaves password login on.

**2. In the admin console**, disable key expiry for the new machine. Otherwise it silently
drops off the tailnet after ~180 days, which you'd only notice once you're remote.

**3. On the laptop**, using the snippet `fleet-onboard` printed:
```sshconfig
Host lab2 lkj0XX-psy
  HostName 100.x.y.z        # Tailscale IP
  User yorguin
```
```bash
ssh-copy-id lab2                          # password once
ssh -o BatchMode=yes lab2 hostname        # must succeed with no prompt; that's what agents use
```
The global `Host *` ControlMaster block (`../lab-access.md`) applies automatically.

**4. Register it.** Add a stanza to `hosts.yaml` with `status: active`, then run `fleet-audit` to
confirm it's reachable.

## Auditing

```bash
fleet-audit            # problems only, every active host (~5 s)
fleet-audit --fetch    # git fetch first so BEHIND is current
fleet-audit --size     # add du sizes (slow on the big data dirs)
fleet-audit --all      # include clean, correctly placed dirs
fleet-audit lab        # one host
```
`UNPUSHED` counts commits on local branches that aren't on any remote. A branch that was
squash-merged and then deleted on GitHub shows up here too, so check `git log --branches --not --remotes`
before assuming work is at risk.

## Auth notes

- **Keys + password both work** on the lab boxes. Password login was kept on purpose
  (`ssh-copy-id`, LAN bootstrap). If a box should ever be key-only, `../bin/harden-sshd-keyonly`
  does that safely.
- Agents always use `ssh -o BatchMode=yes <alias>`. With a key, that works even if the
  ControlMaster socket is down.
