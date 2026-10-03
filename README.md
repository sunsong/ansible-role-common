# `sunsong.common` — Ansible role

Common server bootstrap for Debian and Ubuntu hosts. Sets hostname, swap, packages, users, kernel tunables, sshd hardening, and miscellaneous shell defaults.

## Supported operating systems

| Distribution | Versions | Notes |
|---|---|---|
| Debian | 12 (bookworm), 13 (trixie) | Tested target: trixie. bookworm works. |
| Ubuntu | 24.04 LTS (noble), 25.10 (questing), 26.04 LTS (resolute) | LTS releases receive the most attention; interims work but aren't pinned in CI. |

**Not supported:** anything older than Debian 12 / Ubuntu 24.04, anything RedHat-family (CentOS, RHEL, Rocky, Alma, Fedora, Amazon Linux). See [docs/UPGRADE_CHECKLIST.md](docs/UPGRADE_CHECKLIST.md) for the rationale; see [CHANGELOG.md](CHANGELOG.md) for what changed.

Requires **Ansible 2.12+** for `include_tasks`, FQCN modules, and modern `key: value` syntax.

## Role variables

See `defaults/main.yml` for the full list. The most useful knobs:

| Variable | Default | Purpose |
|---|---|---|
| `enable_*` | `true` (swap: `false`) | Toggle each partial independently |
| `timezone` | `Asia/Shanghai` | Set via `timedatectl set-timezone` |
| `common_users` | `[]` | List of user dicts (see example in defaults) |
| `swap_size` | `2048` (MiB) | Swap file size in MiB |
| `swap_path` | `/swapfile` | Swap file location |
| `enable_ipv6` | `true` | Toggle `/etc/hosts` IPv6 entries |
| `ssh_hardening` | see file | Drop-in at `/etc/ssh/sshd_config.d/00-common.conf` — see [docs/SSH_HARDENING.md](docs/SSH_HARDENING.md) |

## Sysctl tunables of note

`common_sysctl` applies the following — drop the role's defaults in your inventory or `group_vars` to override.

- `vm.swappiness: 1` — keep working sets in RAM; raise on swap-heavy workloads.
- `net.ipv4.tcp_tw_reuse: 1` — allow reuse of TIME-WAIT sockets (safe for outbound clients).
- `fs.inotify.max_user_watches: 524288` and `fs.inotify.max_user_instances: 8192` — raises the inotify limits so dev workloads (Node.js, IDEs, file-sync tools, Docker) stop hitting "too many open files" errors.
- `net.ipv4.tcp_tw_recycle: 0` — kept at 0 for compatibility with monitoring tooling; the kernel removed this knob in 4.12+ so it's a no-op on Debian 13 / Ubuntu 26.04.

## OS quirks and gotchas

These are real, observed behaviors on the supported releases. Keep them in mind when adding features.

### Debian 13 (Trixie)

- **`/etc/timezone` is gone.** The timezone source is `/etc/localtime`. `dpkg-reconfigure tzdata` updates a file that systemd no longer reads. Use `timedatectl set-timezone`.
- **`ntp` package does not exist.** Use `chrony`. The role installs chrony and starts it as the time-sync daemon.
- **`python-selinux` is not a valid package.** It was renamed to `python3-libselinux` long ago, but since the role dropped RHEL/Fedora support the SELinux binding is no longer needed and is no longer in the default package list.
- **`apt-key` is removed.** Third-party repos must use `signed-by=` in sources.list. We don't add any third-party repos, so this is moot.
- **Default kernel is 6.12 LTS.** `tcp_tw_recycle` is a no-op here.
- **AppArmor is the default MAC.** No SELinux tasks are applied.
- **`needrestart` is enabled by default.** It will prompt for service restarts after `apt upgrade`. The role's `upgrade: safe` won't remove packages, so this is mostly harmless.

### Ubuntu 24.04 / 25.10 / 26.04

- **`/etc/timezone` is gone** since 24.04. Same `timedatectl` flow as Debian.
- **`chrony` is the default NTP daemon** since 24.04. We install it via apt.
- **`netplan` is the default network config tool** (since 18.04). The role does not touch networking; be aware that netplan-managed interfaces may be reconfigured on next boot by `cloud-init` if you haven't disabled it.
- **snap is the default for many daemons** (LXD, Firefox, some snaps ship `core22`/`core24`). Snaps run inside AppArmor confinement and use loop devices — they do NOT honor `/etc/security/limits.conf` or the `fs.inotify.*` sysctls for snap-managed processes. If a snap needs higher limits, configure them inside the snap's own config.
- **cloud-init on cloud images owns `/etc/hosts`.** The role's `lineinfile` writes will be overwritten on next boot by `cloud-init`'s `cc_set_hostname` module unless you disable that module or run `cloud-init clean` after.
- **AppArmor is the default MAC.** Same as Debian.
- **Ubuntu Pro (free for personal use, paid for enterprise) extends security support to 2034** for noble (24.04). The role does not enroll hosts in Pro.

### Cross-cutting

- **systemd-resolved handles DNS** on Ubuntu 24.04+ and (usually) Debian 12+. `/etc/resolv.conf` is typically a symlink to `/run/systemd/resolve/stub-resolv.conf` or `/run/systemd/resolve/resolv.conf`. The role does not touch DNS. On Debian 13 netinst installs that use `dhcpcd` instead of systemd-networkd, `/etc/resolv.conf` is rewritten by dhcpcd on every lease renewal and may end up empty if the DHCP server doesn't advertise DNS — set `static domain_name_servers=1.1.1.1 8.8.8.8` in `/etc/dhcpcd.conf` if this happens. Live test on Debian 13 hit this exact issue: `/etc/resolv.conf` was left with a literal `"nameserver# /etc/resolv.conf.tail can replace this line` (single trailing quote, no servers) until fixed.
- **`/etc/security/limits.conf` is parsed by PAM, not systemd.** Most server services on Debian 13 / Ubuntu 26.04 are pure systemd units (chrony, ssh, nginx, …) and bypass PAM. To raise the open-file limit for a systemd service, drop a unit drop-in like `templates/systemd_openfile.conf` into `/etc/systemd/system/<service>.service.d/`. The template is shipped but not auto-applied — wire it up per-service in your playbook.
- **cloud-init is preinstalled on cloud images.** It may rewrite `/etc/hosts`, `/etc/hostname`, and run apt upgrades on first boot. Either disable cloud-init after first boot (`touch /etc/cloud/cloud-init.disabled`) or accept that the first `ansible-playbook` may race with cloud-init.

## Collection requirements

The role uses these modules from outside `ansible.builtin`:

- `ansible.posix.sysctl` (in `tasks/kernel.yml`)
- `ansible.posix.mount` (in `tasks/swap.yml`)
- `ansible.posix.authorized_key` (in `tasks/users.yml`)
- `community.general.git_config` (in `tasks/git.yml`)

Install once on the controller:

```bash
ansible-galaxy collection install ansible.posix community.general
```

Or declare at playbook level:

```yaml
- hosts: all
  collections:
    - ansible.posix
    - community.general
```

## SSH hardening (live-tested)

The role deploys a managed drop-in at `/etc/ssh/sshd_config.d/00-common.conf`. Effective settings after running on Debian 13 (OpenSSH 10.0p2):

- `PermitRootLogin prohibit-password` (explicit; matches Debian default)
- `MaxAuthTries 3` (was 6)
- `LoginGraceTime 60` (was 120)
- `ClientAliveInterval 300` × `ClientAliveCountMax 2` = 10 min idle timeout (was: never)
- `X11Forwarding no` (was yes)
- `UseDNS no`, `GSSAPIAuthentication no` (kept from v1)
- `Banner /etc/issue.net` (created by the role if missing)
- `password_authentication`, `allow_tcp_forwarding`, `ciphers`, `macs`, `kex_algorithms`, `allow_users` / `allow_groups` — left as distro defaults (`null` in vars) to avoid breaking workflows; override per environment. See [docs/SSH_HARDENING.md](docs/SSH_HARDENING.md) for the full rationale and per-setting tradeoffs.

Quirks discovered while implementing: `template`'s `validate:` runs against the SOURCE Jinja file, not the rendered output; sshd rejects `true`/`false` and only accepts `yes`/`no`; the `Include /etc/ssh/sshd_config.d/*.conf` line in the stock sshd_config means drop-ins are processed in filename order — name yours `00-*` to win.

## Known module quirks

- **`ansible.builtin.sysctl` does not exist.** sysctl lives in `ansible.posix`. Same for `mount` and `authorized_key`. These were migrated out of core years ago but the old names still trip people up. (Confirmed on this repo's ansible-lint run — `args[module]` flagged `notify` on `lineinfile`, but `ansible.builtin.sysctl` simply doesn't resolve at runtime.)
- **`ansible.posix.sysctl` defaults to `/etc/sysctl.conf` if `sysctl_file:` is not set.** That's the legacy location and is read once at boot. The role writes to `/etc/sysctl.d/99-common.conf` for modern systemd-sysctl.service pickup. Without the explicit `sysctl_file:`, sysctl.d/ drop-ins would never run.
- **`ansible.posix.sysctl` fails on unknown keys by default.** `net.ipv4.tcp_tw_recycle` was removed from the Linux kernel in 4.12, so on Debian 13 / Ubuntu 26.04 setting it raises `cannot stat /proc/sys/net/ipv4/tcp_tw_recycle`. The role uses `ignoreerrors: true` and keeps the key in defaults for monitoring-tooling compatibility.
- **Bare fact references (`ansible_fqdn`, `ansible_hostname`) emit `INJECT_FACTS_AS_VARS` deprecation warnings** on ansible-core 2.19 and are removed in 2.24. The role uses `hostvars[inventory_hostname]['ansible_fqdn']` for forward compatibility, then caches to `host_fqdn` / `host_short` via `set_fact` to keep line lengths sane.
- **sshd rejects `true`/`false` for boolean directives.** Only `yes`/`no` are valid. The sshd drop-in template uses a Jinja `bool_yn` macro to convert Python booleans to sshd syntax.
- **`template`'s `validate:` runs against the SOURCE .j2 file, not the rendered output.** The sshd drop-in deploy task does not use `validate:`; safety is delegated to the `Restart ssh` handler in `handlers/main.yml` which runs `sshd -t` against the FULL effective config before restart.
## A note on `.ansible/`

If you've cloned this repo and run `ansible-galaxy role install` from inside it, you may see a `.ansible/` directory appear. **That directory is ansible-galaxy's local role/collection cache, not source code** — it contains symlinks pointing at other roles installed elsewhere on your disk. The repo's `.gitignore` excludes it.

How it ended up in commits: at some point during this role's development, `ansible-galaxy` was invoked with a `roles_path` or `collections_path` pointing inside the project. ansible-galaxy created `~/.ansible/roles/<namespace>.<role>/` symlinks targeting the role itself, then a `git add .` from inside the project picked them up. The role still works fine without that directory — it's an installation artifact, not a runtime dependency. If it ever shows up in `git status`, just `git rm -r --cached .ansible/` and `.gitignore` will keep it out.

The role's own runtime artifacts (drop-in files, sysctl values, banner) use the basename `common`, not `sunsong`:

- `/etc/ssh/sshd_config.d/00-common.conf` (sshd drop-in)
- `/etc/sysctl.d/99-common.conf` (sysctl drop-in)
- `/etc/issue.net` (banner)

This is deliberate — on-disk filenames should be short and stable; the namespace/author lives in the role's metadata (`meta/main.yml` → `namespace: sunsong, role_name: common`) and isn't repeated on the host.

## Limitations

- `tasks/swap.yml` — uses `fallocate -l` (instant allocation) instead of `dd` (slow synchronous write). For a 4 GiB swap this drops from ~30s to <1s. Auto-falls-back to `dd` if the filesystem doesn't support `fallocate` (e.g. ZFS root).
- No SELinux support; AppArmor is out of scope (Debian-family default, but not configured here).
- No swap over zram; uses a file-based swap only.
- No firewall management (ufw, nftables). If you need it, layer it on top with a separate role.
- No Docker / container runtime.
- No users beyond the configured `common_users` list — system accounts are not managed.

## License

MIT. See header in `meta/main.yml`.
