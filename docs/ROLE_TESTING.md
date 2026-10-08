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
variables. This command tests the current checkout on both VMs, keeps the
bootstrap variables (including zram and swap enabled), and disables unrelated
role sections for a focused SSH/zram/swap run:

```sh
ansible-playbook \
  -i /home/sloppysun/workspace/kvm-vm-bootstrap/artifacts/inventory.ini \
  tests/integration.yml \
  --limit debian-test,ubuntu-test \
  -e @/home/sloppysun/workspace/kvm-vm-bootstrap/ansible/common.yml \
  -e "role_under_test=$PWD" \
  -e '{"enable_hostname":false,"enable_packages":false,"enable_users":false,"enable_kernel":false,"enable_git":false,"enable_misc":false}'
```

This is an **apply-and-verify integration test**, not a read-only test. It
applies the selected role tasks, flushes handlers (including the SSH restart),
then checks effective host state. The focused command above changes/checks SSH,
zram, and swap. It avoids the `packages.yml` task, which performs a safe APT
upgrade before installing packages. Remove only the final JSON `-e` argument
if you intend to test every section enabled in `common.yml` and accept those
changes; keep the vars-file and local `role_under_test` arguments.

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
cat /proc/swaps
cat /sys/block/zram0/comp_algorithm
cat /sys/block/zram0/disksize
sysctl vm.swappiness
```

Expect `permitrootlogin no`; with both swap layers enabled, `/proc/swaps`
should list `/dev/zram0` at a higher priority than `/swapfile`. zram is
activated by `zramswap.service`, not an `/etc/fstab` entry.

## What It Checks

- Target is Debian 12+ or Ubuntu 24.04+.
- Enabled hostname, package, and configured user state.
- Available configured sysctls match their runtime values (unsupported kernel
  keys skipped by the role are not treated as failures).
- `sshd -t` succeeds and each non-null SSH setting covered by the verifier
  matches `sshd -T` output.
- When enabled, zram service/runtime device, compression algorithm, and active
  swap priority; when both layers are enabled, the disk swapfile priority is
  also checked to be lower. The file must exist, be mode `0600`, be active,
  and have an `/etc/fstab` entry.
- Enabled global Git identity, timezone, chrony service, and profile settings.

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