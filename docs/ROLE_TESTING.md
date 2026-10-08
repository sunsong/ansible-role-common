# Role Integration Tests

`tests/integration.yml` applies the role to the selected hosts and then checks
effective host state. It is an integration test, not a dry-run: it requires
SSH access and privilege escalation, and it makes the same changes as a normal
role run. Use disposable VMs or hosts you are authorized to configure.

## Run Against a Host

Install the role's controller-side collections:

```sh
ansible-galaxy collection install ansible.posix community.general
```

Run the checkout under test by passing its absolute path as
`role_under_test`:

```sh
ansible-playbook \
  -i /path/to/inventory.ini \
  tests/integration.yml \
  -e "role_under_test=$PWD" \
  --limit debian,ubuntu
```

The inventory must configure connection details and any desired role-variable
overrides. For example, `enable_zram: true` and `enable_swapfile: true` test
both swap tiers; the defaults leave both disabled. The play applies the role
using those inventory variables, then validates the effective result.

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

For a read-only spot check after a previous deployment, run these commands on
each host (they are also the swap checks used by the integration play):

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