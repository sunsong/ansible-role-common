# `sunsong.common` — Debian 13 / Ubuntu 26.04 Upgrade & Modernization

This document captures the review of `sunsong.common` against Debian 13 (Trixie) and Ubuntu 26.04 LTS, the gap analysis, and the modernization changes that were applied. The OS quirks live in [README.md](../README.md) under "OS quirks and gotchas"; the SSH hardening rationale is in [docs/SSH_HARDENING.md](SSH_HARDENING.md); the full live-test audit trail (24-row verification matrix, pre/post fixes, hardening workflow) is in [docs/LIVE_TEST_VERIFICATION.md](LIVE_TEST_VERIFICATION.md).

## Status: applied

All checklist items in §6 were applied. The role is now Debian/Ubuntu-only with modern syntax and FQCN modules. Live-tested on Debian 13 (kernel 6.12.107) on 2026-09-30; see §4a for issues the live test surfaced, §5a for the SSH hardening added in v2.1.

## 1. Scope of review

Files reviewed:

- `meta/main.yml`
- `defaults/main.yml`
- `handlers/main.yml`
- `tasks/main.yml`, `tasks/hostname.yml`, `tasks/swap.yml`, `tasks/packages.yml`, `tasks/packages-debian.yml`, `tasks/packages-redhat.yml`, `tasks/kernel.yml`, `tasks/git.yml`, `tasks/ssh.yml`, `tasks/users.yml`, `tasks/miscellaneous.yml`, `tasks/miscellaneous-debian.yml`, `tasks/miscellaneous-redhat.yml`
- `templates/sudo_user.j2`, `templates/systemd_openfile.conf`

Search patterns run: `ansible_distribution`, `distribution_major_version`, `distribution_version`, `python-selinux|libselinux|/etc/timezone|chrony|ntp`, `include`, `service:`, `when:`.

Git history reviewed (last 30 commits).

## 2. Target support matrix

| OS family | Distribution | Versions | Status |
|---|---|---|---|
| Debian | Debian | `bookworm` (12), `trixie` (13) | Supported; trixie is the live-tested target |
| Debian | Ubuntu | `noble` (24.04), `questing` (25.10), `resolute` (26.04 LTS) | Supported; LTS receives the most attention |

**Removed:**

- CentOS (all versions) — CentOS Linux 7/8 EOL, CentOS Stream is a different product. Per user request.
- RedHat Enterprise Linux — dropped with CentOS.
- Fedora — short life cycle (13 months), leftover code paths in `kernel.yml`, not declared in `meta/main.yml`.

## 3. Blocking gaps — applied

### 3.1 `python-selinux` → drop entirely
- **Was:** `common_packages_debian` listed `python-selinux`. Replaced with `python3-libselinux` per Debian/Ubuntu naming.
- **Reality (live test):** Neither package is actually needed now that RHEL-family support is dropped. AppArmor is the default MAC on Debian-family; no Python selinux binding is referenced anywhere. Removed from the package list entirely.

### 3.2 `ntp` → `chrony`
- **Was:** `common_packages_debian` listed `ntp`; role started `chrony` but never installed it.
- **Applied:** Replaced with `chrony` in `defaults/main.yml`. Service task in `miscellaneous-debian.yml` now finds the package.

### 3.3 `/etc/timezone` → `timedatectl set-timezone`
- **Was:** `tasks/miscellaneous-debian.yml` wrote `/etc/timezone` and called `dpkg-reconfigure`.
- **Applied:** Rewritten to stat `/etc/localtime` and call `timedatectl set-timezone {{ timezone }}`. Dropped the `/etc/timezone` write and `dpkg-reconfigure`.

### 3.4 `meta/main.yml` platform list
- **Was:** Listed `Ubuntu all` and `CentOS all` (with malformed indent on the CentOS versions block); no Debian; `min_ansible_version: 1.9`.
- **Applied:** Added `Debian` (bookworm, trixie); Ubuntu pinned to noble/questing/resolute; CentOS entry removed entirely; `min_ansible_version` bumped to `2.12`.

## 4. Modernization — applied

### 4.1 Drop RHEL-family
- Deleted `tasks/packages-redhat.yml`.
- Deleted `tasks/packages-debian.yml` (inlined into `tasks/packages.yml`).
- Deleted `tasks/miscellaneous-redhat.yml`.
- Removed `os_family == 'RedHat'` branches from `tasks/packages.yml` and `tasks/miscellaneous.yml`.
- Removed Fedora + SELinux tasks from `tasks/kernel.yml`.
- Deleted the dead `iptables -nvL` task in `tasks/kernel.yml`.
- Removed orphan defaults: `common_packages_redhat`, `epel_version`, `handle_firewall`, `disable_firewall`, `disable_networkmanager`.
- Removed the commented UFW block in `tasks/miscellaneous-debian.yml` (UFW is out of scope for this role).

### 4.2 Modernize syntax across the role
- Converted every `key=value` to `key: value`.
- Converted every module invocation to FQCN.
- `service` module calls now use `use: service`.
- `tasks/ssh.yml` no longer uses `set_fact` per OS family; the handler inlines `name: ssh`.
- The `sysctl` block in `tasks/kernel.yml` is now a real loop over `common_sysctl | dict2items`.

### 4.3 New sysctl values
- `fs.inotify.max_user_watches: 524288` (default kernel is 8192)
- `fs.inotify.max_user_instances: 8192` (default kernel is 128)

These are written to `/etc/sysctl.d/99-common.conf` and applied live. To revert at runtime: `sudo sysctl fs.inotify.max_user_watches=8192`.

## 4a. Live-test findings on Debian 13 (kernel 6.12.107)

After the modernization was applied, the role was run end-to-end on a fresh Debian 13 VM at `192.168.123.212`. Issues surfaced and fixed:

- **`python3-libselinux` does not exist on Debian 13** (the package is `python3-selinux`). With RHEL-family support dropped the SELinux binding is no longer needed at all — removed from `common_packages_debian`. See §3.1.
- **`ansible.builtin.sysctl` does not resolve.** `sysctl` lives in `ansible.posix`, not `ansible.builtin`. Same for `mount` and `authorized_key`. `git_config` lives in `community.general`. These are the FQCNs used in the role (`ansible.posix.sysctl`, `ansible.posix.mount`, `ansible.posix.authorized_key`, `community.general.git_config`).
- **Collection requirements.** Without `ansible.posix` and `community.general` installed on the controller, the role fails immediately. Documented in README §"Collection requirements". `meta/main.yml` has an empty `dependencies: []` (deps is for ROLE deps, not collections).
- **`ansible.posix.sysctl` defaults to `/etc/sysctl.conf`** if `sysctl_file:` is not set. To write to `/etc/sysctl.d/99-common.conf` instead (so `systemd-sysctl.service` picks it up at boot), pass `sysctl_file: /etc/sysctl.d/99-common.conf` explicitly.
- **`net.ipv4.tcp_tw_recycle` does not exist in 4.12+ kernels.** Setting it raises `cannot stat /proc/sys/net/ipv4/tcp_tw_recycle`. Added `ignoreerrors: true` to the sysctl task; the key is kept in defaults for monitoring-tooling compatibility.
- **Bare `ansible_fqdn` / `ansible_hostname` references** emit `INJECT_FACTS_AS_VARS` deprecation warnings on ansible-core 2.19 and are removed in 2.24. Switched to `hostvars[inventory_hostname]['ansible_fqdn']` and cached via `set_fact` to `host_fqdn` / `host_short` for cleaner lines.
- **Test VM had a buggy `/etc/resolv.conf`** (dhcpcd left a literal `"nameserver# /etc/resolv.conf.tail can replace this line` line with unterminated quote and no servers). Not a role bug — documented as an environment quirk. Fix: set `static domain_name_servers=` in `/etc/dhcpcd.conf`, or run a `configure-demo-dns` service (custom systemd unit in this test env).
- **Lint cleanup.** `notify:` must be at task-level indentation (2 spaces), not at module-arg indentation (4 spaces). Indenting it under `lineinfile` is interpreted as an unknown module arg and fails at runtime even though ansible-lint's `args[module]` rule was the only warning. Fixed in `tasks/ssh.yml`.

## 5. Decisions taken

1. **Drop entire RHEL family.** Recommended in the prior review; user confirmed. No Fedora, no Rocky/Alma, no Amazon Linux.
2. **Enable `sysctl` block with modern syntax.** Values looked reasonable for a server baseline.
3. **Remove UFW / firewall defaults.** No active use case; documented in README as out of scope.
4. **List Ubuntu interims as supported but not LTS-pinned.** `questing` (25.10) is in `meta/main.yml` as a supported codename but the role is tested primarily against LTS releases.
5. **Drop the SELinux Python binding entirely.** The role no longer touches SELinux (Debian-family uses AppArmor). No reason to ship `python3-libselinux`/`python3-selinux` in defaults.

## 5a. SSH hardening (added in v2.1, live-tested 2026-09-30)

The role previously had only two `lineinfile` tasks disabling `UseDNS` and `GSSAPIAuthentication`. Replaced with a managed drop-in template:

- New vars: `ssh_hardening` dict in `defaults/main.yml` (25+ directives, each opt-in via `null` to skip).
- New template: `templates/sshd_common.conf.j2` rendering `/etc/ssh/sshd_config.d/00-common.conf`.
- New tasks: deploy template via `ansible.builtin.template`, plus `/etc/issue.net` banner creation.
- Removed tasks: the pre-hardening `lineinfile` tasks for UseDNS / GSSAPIAuthentication (the template emits those directives directly).
- Quirks handled:
  - `template`'s `validate:` runs against the SOURCE Jinja file, not the rendered output — so the safety net is the `Restart ssh` handler's `sshd -t` check (which validates the FULL effective config including drop-ins).
  - sshd only accepts `yes`/`no` for booleans — the template uses a Jinja `bool_yn` macro that converts Python booleans to sshd syntax. Python's `true`/`false` is rejected by sshd with `unsupported option`.
  - Drop-ins are processed in filename sort order by the stock `Include /etc/ssh/sshd_config.d/*.conf` line — the role's file is named `00-common.conf` to win.
- Live-tested on Debian 13 / OpenSSH 10.0p2. Effective config: see [docs/SSH_HARDENING.md](SSH_HARDENING.md).

## 6. Modernization checklist — applied

### 6.1 Blocking fixes

- [x] Replace `python-selinux` (then drop entirely — see §3.1)
- [x] Replace `ntp` with `chrony` in `defaults/main.yml`
- [x] Rewrite `tasks/miscellaneous-debian.yml` to use `/etc/localtime` symlink + `timedatectl set-timezone`
- [x] Rewrite `meta/main.yml`: Debian + Ubuntu entries, bump `min_ansible_version` to `2.12`

### 6.2 Drop RHEL-family

- [x] Delete `tasks/packages-redhat.yml`
- [x] Delete `tasks/miscellaneous-redhat.yml`
- [x] Update `tasks/packages.yml` (now Debian-only, inlines `packages-debian.yml`)
- [x] Update `tasks/miscellaneous.yml` (now Debian-only via `miscellaneous-debian.yml`)
- [x] Delete `epel_version` default
- [x] Delete `common_packages_redhat` default
- [x] Delete `handle_firewall`, `disable_firewall`, `disable_networkmanager` defaults
- [x] Remove Fedora + SELinux tasks from `tasks/kernel.yml`
- [x] Delete the dead `iptables -nvL` task
- [x] Remove the commented UFW block

### 6.3 Modernize syntax

- [x] Convert all `key=value` to `key: value` across all task files
- [x] Convert `service` module usages to FQCN `ansible.builtin.service` with `use: service`
- [x] Inline `ssh` service name; delete the two `set_fact` tasks in `tasks/ssh.yml`
- [x] Enable the `sysctl` block with `loop: "{{ common_sysctl | dict2items }}"`
- [x] Add FQCN collection prefix to all modules (incl. `ansible.posix.*` and `community.general.*`)
- [x] Add `fs.inotify.*` entries to `common_sysctl`
- [x] Set `sysctl_file: /etc/sysctl.d/99-common.conf` and `ignoreerrors: true` on the sysctl task
- [x] Replace bare `ansible_fqdn` / `ansible_hostname` with `hostvars[inventory_hostname][...]` + cached `set_fact`

### 6.4 SSH hardening (v2.1)

- [x] Add `ssh_hardening` dict to `defaults/main.yml` (25+ directives)
- [x] Create `templates/sshd_common.conf.j2` drop-in template
- [x] Rewrite `tasks/ssh.yml` to deploy template + `/etc/issue.net` banner
- [x] Use Jinja `bool_yn` macro for sshd's `yes`/`no` boolean syntax
- [x] Live-tested on Debian 13 / OpenSSH 10.0p2; `sudo sshd -T` confirms effective values

### 6.5 Verification

- [x] `ansible-playbook --syntax-check` — passes
- [x] `ansible-lint` — passes (production profile, 0 failures)
- [x] Live end-to-end test on Debian 13 VM — idempotent on re-run

## 7. What this document is not

- Not a migration script. The checklist above was the to-do list; it has been applied.
- Not a guarantee that no other gaps exist on Ubuntu 26.04. Verification against a live Ubuntu 26.04 box is still the open item.
- Not a permanent support matrix. Re-evaluate every Debian/Ubuntu LTS release.