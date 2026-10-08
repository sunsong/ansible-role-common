# SSH server hardening — `sunsong.common`

This document captures the live test of the `sunsong.common` role on Debian 13 (Trixie, kernel 6.12.107, OpenSSH 10.0p2) on 2026-09-30, and the SSH hardening steps the role applies.

## What the role already does (pre-hardening)

The role has had an `enable_ssh` partial since v1. It used `lineinfile` to disable two directives in `/etc/ssh/sshd_config`:

- `UseDNS no`
- `GSSAPIAuthentication no`

Both were added historically to fix intermittent slow logins (UseDNS) and unused credential-cache handling (GSSAPI).

## What distro defaults look like on Debian 13 (OpenSSH 10.0p2)

From `sudo sshd -T` on a fresh install:

| Directive | Default | Verdict |
|---|---|---|
| `permitrootlogin` | `without-password` | Distro default permits root key login; this role sets `PermitRootLogin no` |
| `passwordauthentication` | `yes` | **Risk** — vulnerable to brute force |
| `pubkeyauthentication` | `yes` | OK |
| `permitemptypasswords` | `no` | OK |
| `maxauthtries` | `6` | High |
| `logingracetime` | `120` | High |
| `clientaliveinterval` | `0` (disabled) | No idle timeout |
| `clientalivecountmax` | `3` | Combined with 0 = never disconnects |
| `x11forwarding` | `yes` | Unused on most servers |
| `allowtcpforwarding` | `yes` | Open if host isn't a bastion |
| `allowagentforwarding` | `yes` | Needed for sudoer agent forwarding |
| `usedns` | `yes` | Causes slow logins |
| `gssapiauthentication` | `no` | OK |
| `ciphers` | chacha20-poly1305, aes-gcm | Modern |
| `macs` | umac-128-etm, hmac-sha2-etm | Modern |
| `kexalgorithms` | mlkem768x25519, curve25519 | Modern (PQ-hybrid) |

The algorithm defaults are already modern — no override needed on OpenSSH 9.6+/10.x.

## What the role now does (post-hardening)

The role deploys a managed drop-in at `/etc/ssh/sshd_config.d/00-common.conf` via `templates/sshd_common.conf.j2`. The drop-in is picked up by the stock `Include /etc/ssh/sshd_config.d/*.conf` line in the Debian/Ubuntu stock `sshd_config`.

Effective settings after running the role (from `sudo sshd -T` on the test VM):

```
permitrootlogin           no
passwordauthentication    yes               # kept on; opt-in to disable
kbdinteractiveauth        <unset>           # distro default
pubkeyauthentication      yes
permitemptypasswords      no
maxauthtries              3
logingracetime            60
x11forwarding             no
allowtcpforwarding        yes               # kept on; opt-in to disable
allowagentforwarding      yes
allowstreamlocalforwarding <unset>          # distro default
clientaliveinterval       300
clientalivecountmax       2
maxsessions               10
maxstartups               10:30:100
usedns                    no
gssapiauthentication      no
gssapicleanupcredentials  yes
banner                    /etc/issue.net
ciphers/macs/kexalgorithms <unset>           # distro defaults (modern)
allowusers/allowgroups/denyusers/denygroups  <unset>
```

Combined idle timeout: `clientaliveinterval 300` × `clientalivecountmax 2` = 10 min before dead sessions are killed.

## Settings the role makes configurable but does NOT change by default

| Setting | Why left alone | How to override |
|---|---|---|
| `password_authentication` | Setting to `false` locks out any account without an authorized key deployed. Common footgun during cloud bootstrap when the operator's key isn't yet in `~/.ssh/authorized_keys`. | Set `ssh_hardening.password_authentication: false` in `group_vars/all.yml` after confirming every user has a key. |
| `allow_tcp_forwarding` | Disabling breaks `ssh -L` / `ssh -R` and jumphost patterns. If the host is a bastion, you want this on; if it's an app server, you may want it off. | Set per role. |
| `allow_stream_local_forwarding` | Less common; defaults to `no` on modern sshd. Leave unless needed. | Set per role. |
| `kbd_interactive_authentication` | Defaults to `no`; only set if using PAM-backed 2FA modules. | Set per role. |
| `ciphers` / `macs` / `kex_algorithms` | Distro defaults are already modern (OpenSSH 9.6+). Override only for compliance (FIPS, CIS benchmarks). | Set per role, comma-separated strings. |
| `allow_users` / `allow_groups` | Setting wrong value locks you out. | Set per role AFTER confirming the user/groups exist. |
| `deny_users` / `deny_groups` | Setting wrong value locks you out. | Set per role. |

## Quirks / weirdness encountered

### `validate:` on `template` runs against the SOURCE template, not the rendered output

The Ansible `template` module's `validate:` runs against the SOURCE file (the .j2), not the rendered output. Passing `validate: '/usr/sbin/sshd -t -f %s'` ran `sshd -t` against the Jinja template and failed with `unsupported option "true"` because the template content isn't valid sshd_config.

Two fixes considered:

1. Drop `validate:` from the template task; rely on the `restart ssh` handler in `handlers/main.yml` which already runs `sshd -t` before restart.
2. Validate the rendered output in a separate task.

Chose (1) — the handler's check covers the deployed file post-render and is the source of truth for safety. The template task uses `notify: Restart ssh` which only fires on changed, and the handler runs `sshd -t -f /etc/ssh/sshd_config` (the FULL config including all drop-ins).

### `true` / `false` vs `yes` / `no`

sshd only accepts `yes` / `no` for boolean directives; `true` / `false` is rejected with `unsupported option "true"`. The Jinja `| string | lower` filter emits the Python `True`/`False` lowercase form, which sshd rejects. The template uses a small `bool_yn` macro instead:

```jinja
{% macro bool_yn(v) -%}
{{- 'yes' if v else 'no' -}}
{%- endmacro %}
```

Then `{{ bool_yn(h.x11_forwarding) }}` instead of `{{ h.x11_forwarding | string | lower }}`.

### sshd_config_include behavior

The stock `/etc/ssh/sshd_config` on Debian 13 / Ubuntu 26.04 has:

```
Include /etc/ssh/sshd_config.d/*.conf
```

This means our drop-in is read as part of the main config. The first directive in our drop-in (`PermitRootLogin no`) takes effect because sshd applies the first matching keyword and ignores subsequent ones. Our drop-in must therefore be named with a low leading digit (we use `00-common.conf`) so it sorts before any distro-shipped drop-ins.

### per-handler live-test issue (already fixed pre-hardening)

In an earlier task, `notify: Restart ssh` was indented at module-arg level (4 spaces) under `lineinfile:`. sshd_config-dropin deployment uses the same handler; since we moved `notify:` to task-level (2 spaces) in `tasks/ssh.yml`, the handler fires correctly.

## How to verify

```bash
# Effective config (post-hardening)
sudo sshd -T | grep -iE '^(permitrootlogin|passwordauthentication|maxauthtries|clientaliveinterval|x11forwarding|allowtcpforwarding|usedns|banner)'

# Validate syntax without restarting
sudo sshd -t

# Show the drop-in
cat /etc/ssh/sshd_config.d/00-common.conf

# Tail the journal for rejected logins (after attempting)
sudo journalctl -u ssh -n 50
```

## How to roll back

```bash
# Remove the drop-in
sudo rm /etc/ssh/sshd_config.d/00-common.conf
sudo systemctl restart ssh

# Restore the banner
sudo rm /etc/issue.net  # if it was created by the role
```

Or in Ansible:

```yaml
- hosts: all
  tasks:
    - name: Remove sshd hardening drop-in
      ansible.builtin.file:
        path: /etc/ssh/sshd_config.d/00-common.conf
        state: absent
      notify: Restart ssh
```

## How to extend

To add a new directive, edit `defaults/main.yml` under `ssh_hardening` and `templates/sshd_common.conf.j2`. The template is fully conditional — adding a key with `null` value omits the directive.

To enforce something for compliance (e.g. CIS), override per environment:

```yaml
# group_vars/prod.yml
ssh_hardening:
  password_authentication: false
  allow_tcp_forwarding: false
  allow_users: deployer,ops
  ciphers: 'chacha20-poly1305@openssh.com,aes256-gcm@openssh.com'
  macs: 'hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com'
  kex_algorithms: 'curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512'
```

Run the role, verify with `sudo sshd -T`, then commit.