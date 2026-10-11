# Role Integration Tests

`tests/integration.yml` applies the role to the selected hosts and then checks
effective host state. It is an integration test, not a dry-run: it requires
SSH access and privilege escalation, and it makes the same changes as a normal
role run. Use disposable VMs or hosts you are authorized to configure.

## Run Against the Debian and Ubuntu VMs

From the role checkout, install the controller-side collections if needed:

```sh
ansible-galaxy collection install ansible.posix community.general
```

The VM bootstrap repo provides the inventory, SSH credentials, and per-VM
variables. This command tests the current checkout on both VMs with every
section enabled in the bootstrap `common.yml`, except package management:

```sh
ansible-playbook \
  -i /home/sloppysun/workspace/kvm-vm-bootstrap/artifacts/inventory.ini \
  tests/integration.yml \
  --limit debian-test,ubuntu-test \
  -e @/home/sloppysun/workspace/kvm-vm-bootstrap/ansible/common.yml \
  -e "role_under_test=$PWD" \
  -e '{"enable_packages":false}'
```

The generated inventory lists only the hosts `scripts/run-ansible.sh` was last
run for. If a VM is missing (Ansible warns `Could not match supplied host
pattern`), add its line, e.g.
`debian-test ansible_host=192.168.123.201 ansible_user=debian`.

This is an **apply-and-verify integration test**, not a read-only test. It
applies the enabled role tasks, flushes handlers (including the SSH restart),
then checks effective host state. `enable_packages: false` skips
`packages.yml`, which performs a safe APT upgrade before installing packages;
remove that argument to include it. To narrow the run, add more `enable_*:
false` keys to the same JSON, e.g. `"enable_users":false`.

To use a different inventory, replace the inventory path and host limit. To
test another role source, change `role_under_test`; `$PWD` means this checkout.
Ansible exits nonzero if a connection or assertion fails. A passing run prints
one `PASS` message per host and ends with `failed=0` in the recap.

## Read-Only Checks on Already-Configured Hosts

For a verification run that must not apply the role, use these commands on
each host. They query effective runtime state directly:

```sh
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|maxauthtries|banner) '
systemctl is-active ssh
getent ahostsv4 "$(hostname)"
cat /proc/swaps
cat /sys/block/zram0/comp_algorithm
cat /sys/block/zram0/disksize
sysctl vm.swappiness
```

Expect `permitrootlogin no`, and the hostname to resolve to `127.0.0.1` (not
`127.0.1.1`); with both swap layers enabled, `/proc/swaps`
should list `/dev/zram0` at a higher priority than `/swapfile`. zram is
activated by `zramswap.service`, not an `/etc/fstab` entry.

## What It Checks

- Target is Debian 12+ or Ubuntu 24.04+.
- Hostname: the role did not rename the host, and `getent ahostsv4` resolves
  the short hostname to `127.0.0.1` and not `127.0.1.1`. Other addresses,
  such as an installer's static-IP entry, are allowed.
- Enabled package and configured user state (accounts exist; SSH keys and
  sudoers fragments are not asserted).
- Available configured sysctls match their runtime values, ignoring
  whitespace differences (the kernel prints `ip_local_port_range`
  tab-separated). Unsupported kernel keys skipped by the role are not
  treated as failures.
- `ssh.service` is running after the role's handlers, `sshd -t` succeeds, and
  each non-null setting in the merged `ssh_hardening_effective` matches
  `sshd -T` output. `permitrootlogin no` is required regardless of overrides.
- When enabled, zram service/runtime device, compression algorithm, active
  swap priority, and effective `vm.swappiness`; when both layers are enabled,
  the disk swapfile priority is also checked to be lower. The file must exist,
  be mode `0600`, be active, and have an `/etc/fstab` entry.
- Global Git identity and timezone when set (both default to `null`), plus
  chrony service and profile settings when `enable_misc` is on.

Ansible exits nonzero on the first failed assertion and reports the host and
check name. A successful run ends with a `PASS` message per host. The playbook
does not force memory pressure to prove that pages are actually compressed;
it verifies that zram is configured, active, and selected as the higher
priority swap device.

## Verify Existing Hosts Without Reapplying

For additional zram detail after the checks above:

```sh
sudo systemctl is-active zramswap
sudo zramctl
cat /proc/swaps
cat /sys/block/zram0/comp_algorithm
cat /sys/block/zram0/disksize
sysctl vm.swappiness
```

`/etc/fstab` contains the optional disk swapfile, not zram. zram is created
and activated by `zramswap.service` at boot.