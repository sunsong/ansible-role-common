# Changelog

All notable changes to `sunsong.common` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- **SSH server hardening** via managed drop-in at `/etc/ssh/sshd_config.d/00-common.conf`. See [docs/SSH_HARDENING.md](docs/SSH_HARDENING.md) for full rationale; see [docs/LIVE_TEST_VERIFICATION.md](docs/LIVE_TEST_VERIFICATION.md) for the verification matrix.
  - New vars: `ssh_hardening` dict in `defaults/main.yml` with 25+ directives, each opt-in via `null` to omit.
  - New template: `templates/sshd_common.conf.j2` using a `bool_yn` macro to convert Python booleans to sshd's `yes`/`no` syntax.
  - New task: deploy template + create `/etc/issue.net` banner.
- **Inotify sysctl limits** in `common_sysctl` — `fs.inotify.max_user_watches: 524288` and `fs.inotify.max_user_instances: 8192`. Addresses "too many open files" failures in dev workloads (Node, IDEs, Docker).

### Changed

- **Dropped RedHat-family support entirely.** CentOS, RHEL, Rocky/Alma, Fedora, Amazon Linux are out of scope. Removed `tasks/packages-redhat.yml`, `tasks/miscellaneous-redhat.yml`, `tasks/packages-debian.yml` (inlined into `packages.yml`), orphan defaults (`common_packages_redhat`, `epel_version`, `handle_firewall`, `disable_firewall`, `disable_networkmanager`), and all `os_family == 'RedHat'` branches.
- **Modernized syntax across the role:**
  - All `key=value` → `key: value`
  - All modules converted to FQCN (`ansible.builtin.*`, `ansible.posix.*`, `community.general.*`)
  - `service` module calls use `use: service`
  - Bare fact references (`ansible_fqdn`, `ansible_hostname`) replaced with `hostvars[inventory_hostname][...]` and cached via `set_fact` to `host_fqdn` / `host_short` (forward-compat with ansible-core 2.24)
  - `tasks/ssh.yml` simplified — service name inlined as `ssh` (no more per-OS-family `set_fact`)
- **Timezone task rewritten** to use `/etc/localtime` symlink + `timedatectl set-timezone`. Dropped `/etc/timezone` write and `dpkg-reconfigure` call (legacy; systemd ignores `/etc/timezone` on Debian 12+ / Ubuntu 24.04+).
- **`tasks/kernel.yml` cleaned up:**
  - Removed Fedora + SELinux tasks (dead after RHEL drop)
  - Removed dead `iptables -nvL` probe
  - Enabled the `sysctl` block with `loop: "{{ common_sysctl | dict2items }}"`, `sysctl_file: /etc/sysctl.d/99-common.conf`, `ignoreerrors: true`
- **`meta/main.yml` cleaned up:**
  - Fixed malformed CentOS indent
  - Added `Debian` (bookworm, trixie) and updated Ubuntu (noble, questing, resolute)
  - Removed CentOS entry
  - Bumped `min_ansible_version` to `2.12`
  - Renamed `name` → `role_name` (modern schema)
- **`tasks/swap.yml` switched from `dd if=/dev/zero` to `fallocate -l`** for swap file allocation. Fallocate is supported on ext4/xfs since kernel 3.5 and on btrfs since 5.8; Debian 13 (6.12 LTS) and Ubuntu 26.04 both support it on default root filesystems. Includes a rescue task that falls back to `dd` if fallocate fails (e.g. on a ZFS root).

### Removed

- CentOS / RedHat / Rocky / Alma / Fedora / Amazon Linux support (per user request).
- `python-selinux` from default packages — replaced with `chrony` (no SELinux binding needed; Debian/Ubuntu use AppArmor).
- `ntp` from default packages — replaced with `chrony` (chrony is the default time-sync daemon on Debian 12+ / Ubuntu 24.04+).
- SELinux-disable task (Debian/Ubuntu use AppArmor, not SELinux).
- Commented UFW block (UFW is out of scope for this role).

### Fixed

- `chrony` was being started but never installed → now installed via `common_packages_debian`.
- Module FQCNs corrected after live test: `ansible.posix.sysctl`, `ansible.posix.mount`, `ansible.posix.authorized_key`, `community.general.git_config`.
- `ansible.posix.sysctl` no longer writes to legacy `/etc/sysctl.conf` — explicitly targets `/etc/sysctl.d/99-common.conf`.
- Bare `ansible_fqdn` / `ansible_hostname` no longer emit `INJECT_FACTS_AS_VARS` deprecation (removed in ansible-core 2.24) — switched to `hostvars[inventory_hostname][...]`.
- `template.validate:` no longer fails on the sshd drop-in — was running `sshd -t` against the SOURCE .j2 (Jinja syntax), not the rendered output. Validation moved to the `Restart ssh` handler which checks the FULL effective config.
- sshd's `true`/`false` boolean rejection fixed via Jinja `bool_yn` macro in `templates/sshd_common.conf.j2`.
- `notify: Restart ssh` indentation fixed — was at module-arg level (4 spaces), causing "Unsupported parameters" runtime errors.
- `tasks/kernel.yml` sysctl task no longer fails on unknown keys — `ignoreerrors: true` covers `net.ipv4.tcp_tw_recycle` (removed in kernel 4.12+).

### Documentation

- `README.md` — rewritten for Debian/Ubuntu-only role with supported-OS matrix, OS quirks per distro, sysctl notes, collection requirements, idempotency notes, and SSH hardening summary.
- `docs/SSH_HARDENING.md` — full per-setting rationale, override examples, rollback steps.
- `docs/UPGRADE_CHECKLIST.md` — gap analysis + modernization record + live-test findings + SSH hardening entry.
- `docs/LIVE_TEST_VERIFICATION.md` — 30-row verification matrix (test → expected → actual → pass), pre/post fixes, full SSH hardening workflow.
- `CHANGELOG.md` — this file.

## [v1] — legacy

Pre-modernization. Supported Ubuntu + CentOS, used bare module names, legacy YAML syntax, used `service` instead of `ansible.builtin.service`, mixed `key=value` and `key: value`, no inotify fix, no SSH hardening beyond two `lineinfile` directives.

See git history: `git log --oneline` for full pre-modernization commits.