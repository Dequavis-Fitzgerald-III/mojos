# BakerOS — CLAUDE.md

## What this repo is

This is the live OS configuration for this machine. The scripts and manifests here define everything installed and configured on the running system. The goal is to keep the current setup working — no planned upgrades or new features.

GitHub: `github.com/Dequavis-Fitzgerald-III/baker`

---

## The Baker Fleet

All machines run Arch Linux. Tailnet name: **circus-tent**.

| Hostname | Role |
|---|---|
| nomadbaker | Laptop |
| pearlybaker | Desktop (full GPU) |

---

## Script Flow

```
install.sh  →  (reboot)  →  post-install.sh  →  (reboot)  →  post-reboot.sh
     ↓                                                               ↓
configure.sh                                                  [system running]
  (called from                                                       ↓
   chroot)                                                    baker-update  ← run anytime to sync
                                                                     ↓
                                                              configure.sh
                                                               (called by
                                                               upgrade.sh)
```

### `install.sh`
Run from the Arch live ISO as root. Interactive questions upfront (profile, hostname, CPU/GPU, timezone, disk, dual boot, LUKS, dotfiles repo), then fully unattended from the confirmation onwards.

Sections:
1. Interactive questions
2. Partitioning (GPT, EFI + root, LUKS optional)
3. LUKS setup (luks2, opens as `cryptroot`) — also computes `LUKS_UUID` via `blkid`
4. Format + mount
5. `pacstrap` — fetches `packages/base.txt` + `packages/profile/<profile>.txt` + `packages/hardware/gpu-<gpu>.txt` from the repo, adds bootstrap packages (kernel, bootloader, ucode) on top, installs everything
6. `fstab` generation (+ optional secondary HDD entry)
7. `arch-chroot` — bootstrap only: passwords, user creation, `grub-install`, services, then calls `configure.sh` via curl
8. Write `.baker-config`
9. Download `post-install.sh` + `post-reboot.sh`

Key design decisions:
- EFI mounted at `/boot` (single boot) or `/boot/efi` (dual boot) to avoid clobbering the Windows bootloader
- `kms` hook excluded from mkinitcpio when GPU is Nvidia (avoids black screen)
- `sshd` hardened from first boot via `/etc/ssh/sshd_config.d/99-baker.conf` (key-only auth, no passwords)

### `configure.sh`
Applies all idempotent system configuration. Called by `install.sh` (inside the chroot, vars passed via environment) and by `upgrade.sh` (on a live system, reads `.baker-config`). Safe to run at any time — checks current state before making changes.

Covers: timezone, locale, console keymap, hostname (`/etc/hostname` + `/etc/hosts`), sudo, hyprland-welcome removal, HDD mount point, mkinitcpio HOOKS + initramfs rebuild (only if HOOKS changed), GRUB cmdline + timeout + dual boot + `grub-mkconfig` (only if config changed), sshd hardening drop-in.

Everything configure.sh does is driven by a key in `.baker-config` — edit a value and run `baker-update` to apply it.

### `post-install.sh`
Run as the regular user after first boot. Reads `.install-config` written by `install.sh`.

Sections:
1. Network check (auto-wifi on laptop using saved credentials)
2. Install `yay` (AUR helper)
3. AUR packages — reads `[aur]` section from `packages/base.txt`
4. Chrome flags (disable keyring prompt)
5. Flatpak packages — reads `[flatpak]` section from `packages/base.txt` via curl
6. Home directory setup: clone `baker` + dotfiles repos over HTTPS, symlink dotfiles
7. NordVPN group + service setup
8. Locale/timezone confirmation via `localectl`/`timedatectl`
9. Services: NetworkManager, sddm, ufw, pipewire (user), laptop extras
10. SSH: generate ed25519 keypair, add to GitHub, configure git SSH rewrite, register key in `baker/keys/`, rebuild `authorized_keys` + `~/.ssh/config`, commit + push key to repo

Self-deletes on completion, then reboots.

### `post-reboot.sh`
Short final script run after the post-install reboot.

1. Tailscale login (`tailscale up`, browser flow) — must happen before NordVPN connects, otherwise NordLynx captures the default route and breaks the Tailscale auth browser flow.
2. NordVPN login (browser flow) + set autoconnect (us). No killswitch — NordLynx (WireGuard) + Tailscale (WireGuard) conflict at the routing level and the killswitch breaks Tailscale entirely.
3. `.bashrc` additions: `WORKON_HOME`, `baker-update` alias, todo checklist hook
4. Writes `~/.todo` with remaining manual steps

Self-deletes on completion.

---

## Key Registry — `keys/`

`keys/` is the_baker fleet key registry. Each install copies `~/.ssh/id_ed25519.pub` to `keys/<hostname>.pub` and commits + pushes it.

`authorized_keys` and `~/.ssh/config` are rebuilt from whatever `.pub` files exist in the directory — the machine list is derived from the repo, not hardcoded anywhere.

`~/.ssh/config` uses Tailscale MagicDNS short names (`Hostname nomadbaker`) so SSH across the_baker's fleet works once Tailscale is up, without any further config.

### `sync-baker-keys.sh`
Run on existing machines when a new machine joins. Pulls the repo and rebuilds `authorized_keys` + `~/.ssh/config`.

---

## Package Manifests — `packages/`

Single source of truth for what's installed on baker machines. Both `install.sh` and `upgrade.sh` read from these files — add a package once and it flows to fresh installs and existing machines alike.

| File | Contents |
|---|---|
| `packages/base.txt` | Always installed — `[pacman]`, `[aur]`, `[flatpak]` sections |
| `packages/profile/workstation.txt` | pearlybaker extras — `[pacman]` section |
| `packages/profile/laptop.txt` | nomadbaker extras — `[pacman]` section |
| `packages/hardware/gpu-nvidia.txt` | Nvidia GPU drivers — `[pacman]` section |
| `packages/hardware/gpu-amd.txt` | AMD GPU drivers — `[pacman]` section |
| `packages/hardware/gpu-intel.txt` | Intel GPU drivers — `[pacman]` section |

Bootstrap packages (kernel, bootloader, ucode) are hardcoded in `install.sh` only — they are install-time specific and don't belong in the manifests.

### `upgrade.sh`
Run on any live baker machine to converge it to the current desired state. Safe to run at any time — all steps are idempotent. Accessible via the `baker-update` alias added to `.bashrc` by `post-reboot.sh`.

1. Pull latest baker repo
2. `pacman -Syu` — full system upgrade
3. Install missing pacman packages from manifests (`--needed` skips already-installed)
4. Install missing AUR packages
5. Install missing Flatpak packages
6. Pull dotfiles + re-symlink
7. Ensure services enabled
8. Rebuild SSH config via `sync-baker-keys.sh`
9. Sync `.baker-config` — re-detects hardware (GPU, LUKS, HDD, dual boot) and rewrites file
10. Run `configure.sh` — converges system config to the freshly synced `.baker-config`

### `.baker-config`
Permanent machine config file written by `install.sh` to `~/.baker-config`. Persists across reboots and is never deleted. Has two sections:

**System Config (editable)** — change a value and run `baker-update` to apply:
`HOSTNAME`, `TIMEZONE`, `LOCALE`, `KEYMAP`, `DOTFILES_URL`

**Hardware (auto-detected, do not edit)** — reflects physical machine reality, reset on every update:
`USERNAME`, `PROFILE`, `GPU`, `LUKS`, `LUKS_UUID`, `HDD`, `HDD_MOUNT`, `DUAL_BOOT`

---

## Stack

| Tool | Purpose |
|---|---|
| Hyprland | Wayland compositor |
| sddm | Display manager |
| waybar | Status bar |
| kitty | Terminal |
| rofi-wayland | App launcher |
| pipewire | Audio |
| NetworkManager | Networking |
| Tailscale | VPN mesh / MagicDNS (circus-tent tailnet) |
| NordVPN | Privacy VPN (autoconnect us, no killswitch) |
| ufw | Firewall (deny incoming, allow outgoing) |
| ollama | Local LLM inference (workstation only) |
| yay | AUR helper |

---

## Packages Pending Addition to Manifests

Packages installed ad-hoc that should be added to `packages/base.txt`:

| Package | Source | Purpose |
|---|---|---|
| `grim` | pacman | Wayland screenshot tool |
| `slurp` | pacman | Interactive region/window selector for Wayland (used with grim) |
| `wl-clipboard` | pacman | Wayland clipboard (`wl-copy`/`wl-paste`) |
| `grimblast` | AUR | Wrapper around grim+slurp with Hyprland window detection; `grimblast copy area` → screenshot to clipboard |

---

## Known Issues

- NordVPN killswitch is disabled — NordLynx (WireGuard) and Tailscale (WireGuard) conflict at the routing level; enabling the killswitch breaks Tailscale.

---

## Network DNS Notes

Some networks (confirmed: Newcastle University) block Tailscale's domains (`login.tailscale.com`, `controlplane.tailscale.com`) at the DNS level. Symptoms:
- `tailscale up` browser auth page fails to load (`ERR_NAME_NOT_RESOLVED`)
- Tailscale health warning: "hasn't received a network map from the coordination server"
- Chicken-and-egg on machines already using Tailscale MagicDNS (`100.100.100.100`) as their DNS — Tailscale DNS goes down when Tailscale loses the control server

`post-reboot.sh` handles this automatically by testing DNS resolution before `tailscale up` and overriding to `8.8.8.8` if needed. On an existing machine, manual fix: `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf && sudo systemctl restart tailscaled`.

---

## Conventions

- Explain everything: before running any command or making any change, explain what it does in plain terms. Never assume prior knowledge of a tool, flag, or concept.
- One section at a time: explain the change, then write it, so each step can be reviewed before continuing.
- No co-author lines in git commits.
- Commit messages: conventional commits style (`feat:`, `fix:`, `refactor:` etc.).
- All scripts use `set -e` and the same colour/logging helpers (`info`, `success`, `warn`, `error`, `section`).
- HTTPS for all clones in `post-install.sh` (no SSH key needed yet); git URL rewrite configured at the end of Section 10 so everything switches to SSH after that.
- **Naming convention:** commands use kebab-case (`baker-update`, `sync-baker-keys`); scripts use kebab-case (`post-install.sh`, `post-reboot.sh`); config files use underscore (`.baker_config`, `the_baker.conf`).
- **the_baker/user split:** `the_baker` is the role for whoever operates the_baker's fleet. `user` is anyone running BakerOS without a the_baker token. the_baker-specific steps and packages are gated on the the_baker token.

## Fix Workflow

When making a fix, follow this sequence:

1. **Read the docs** — understand the relevant man pages, upstream docs, or Arch Wiki before touching anything.
2. **Explain the fix** — describe what's changing and why before writing it, so it can be reviewed and understood fully.
3. **Make the fix** — same rules as always: `set -e`, colour helpers, conventional commit style, one section at a time.
4. **Commit and push** — descriptive conventional commit message; no co-author lines.
5. **Update context** — if the fix changes how something works, update `CLAUDE.md` to reflect it.
