# Review Follow-Ups

Open items from the final role review on 2026-10-11. These are follow-up
questions and validation tasks, not known blockers for the currently tested
Debian 13 / Ubuntu 26.04 configurations.

## Hostname And `/etc/hosts`

- [ ] Preserve pre-existing aliases when adding the current short hostname to
  `127.0.0.1`; review whether replacing the complete IPv4 localhost line is
  necessary.
- [ ] Preserve existing `::1` aliases instead of replacing the complete IPv6
  localhost line. The machine hostname should remain off IPv6 loopback unless
  there is an explicit requirement for it.
- [x] Re-run the hostname integration check on Debian as well as Ubuntu. The
  last available bootstrap inventory contained only `ubuntu-test`.
- [ ] Confirm expected behavior on Ubuntu cloud images where cloud-init may
  rewrite `/etc/hosts` on boot.
- [ ] Restore `debian-test` in the kvm-vm-bootstrap generated inventory; the
  2026-10-11 two-OS run used a temporary copy with the Debian line added.

## Swap

- [ ] Validate an existing `swap_path` before assuming it is a valid swap
  file. Current creation and `mkswap` tasks skip whenever the path already
  exists.
- [ ] Decide whether an existing swap file whose size differs from
  `swap_size` should be reported, preserved, or reconciled. Avoid resizing or
  replacing an active swap file without an explicit safe procedure.
- [ ] Keep zram and swapfile verification based on live `/proc/swaps` entries
  and priorities, not only package/config-file presence.

## Sysctls And Integration Coverage

- [ ] Narrow the integration test's ignored `sysctl -n` failures. It currently
  skips every unavailable key; only known unsupported keys such as
  `net.ipv4.tcp_tw_recycle` should be exempt by default.
- [ ] Run a full enabled-feature integration pass on both supported test OSes
  with `enable_kernel` and `enable_packages` on. The 2026-10-11 run covered
  every other section on Debian 13 and Ubuntu 26.04; the whitespace-tolerant
  sysctl comparison was only checked locally.
- [ ] Decide whether the integration suite should be split into apply-and-
  verify and read-only verify modes. The current play applies the role before
  checking effective state; package management can perform a safe APT upgrade.

## Operational Defaults And Security

- [ ] Consider separating `apt upgrade: safe` from package installation behind
  a separate opt-in. `enable_packages` currently enables both operations.
- [x] Make the privilege level of `common_users[*].sudo: true` explicit in
  user-facing docs: the template grants `NOPASSWD: ALL`, i.e. unrestricted
  passwordless root command execution.
- [x] Verify controller dependencies for lint/runtime checks. `ansible-lint`
  previously warned that it could not load `community.general.git_config`
  for module option validation when that collection was unavailable.

## Supported Platforms

- [ ] Run the integration suite on Debian 12 and Ubuntu 24.04, not only Debian
  13 and Ubuntu 26.04, before treating all Galaxy-advertised platforms as
  verified.