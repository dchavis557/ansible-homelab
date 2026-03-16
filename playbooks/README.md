# Ansible Playbooks
---
## install-tools.yml

Installs base tooling across lab targets. Written as my first independently-designed
playbook — I defined each task's purpose and logic, using AI assistance to nail down
syntax and module parameters.

### What It Does

Installs my personal "home plate" package set on any Debian/Ubuntu target:

- `curl` — because it's always needed
- `btop` — modern resource monitor
- `mc` — Midnight Commander, fast terminal file navigation

Then confirms every package was installed via `which`.

### Why It's Built This Way

**`cache_valid_time: 3600`** — tells apt to skip re-fetching package indexes if they're
less than an hour old. Keeps the playbook idempotent and fast when running repeatedly
without hammering the repos unnecessarily.

**`which` + loop for confirmation** — rather than assuming success, each installed binary
gets explicitly verified and its path printed. Noisy on purpose during early lab work.

**`changed_when: false`** on the `command` task — `which` is read-only. Without this,
Ansible would flag every run as changed, which breaks idempotency reporting.

### Environment

- **Inventory group:** `lab` — Ubuntu VMs provisioned specifically to test this playbook
- **Tested on:** Ubuntu 24.04

---

## distro-update.yml

Ansible playbook for automated Linux patching across my homelab.
Handles full upgrade cycles, intelligent reboot detection, and post-reboot
verification — running in production against real infrastructure.

AI assisted with syntax and implementation. Logic reviewed and validated
before production deployment.

> A rewrite is planned as my Ansible fluency has grown since initial deployment.

### What It Does

- Refreshes package metadata on both Debian and RedHat-family hosts
- Performs full dist-upgrade (Debian) or full package update (Fedora/RHEL)
- Runs autoremove to clean up orphaned packages
- Detects whether a reboot is required using distro-appropriate methods
- Reboots only the hosts that need it, with pre/post delays for safety
- Re-gathers facts post-reboot for any downstream plays or tooling
- Runs a second verification play to confirm rebooted hosts came back online
- Prints a summary of which hosts rebooted — useful in Semaphore logs

### Why It's Built This Way

**Multi-distro `when` conditionals** — the homelab runs a mix of Ubuntu and
Fedora targets. Rather than maintaining separate playbooks, distro-specific
tasks are gated with `ansible_os_family` and `ansible_distribution` checks,
keeping everything in one place.

**Reboot detection is distro-aware** — Debian/Ubuntu writes a flag file to
`/var/run/reboot-required` after kernel updates. Fedora doesn't — it uses
`dnf needs-restarting -r` instead. Both results are normalized into a single
`reboot_needed` fact so the rest of the playbook doesn't have to care which
distro it's on.

**`serial: 1` (commented)** — available to enable for rolling updates so
hosts aren't rebooted simultaneously. Left commented for homelab use where
that's usually acceptable, but ready for environments where it isn't.

**Second verification play** — a separate play runs after the first to ping
rebooted hosts and wait for services to settle. Hosts that didn't reboot are
skipped explicitly rather than silently.

**`failed_when: false` on needs-restarting** — `dnf needs-restarting -r`
returns exit code 1 when a reboot *is* needed, which Ansible would treat as
a failure. This overrides that behavior so the return code can be evaluated
intentionally.

### Environment

- **Tested on:** Ubuntu 22.04, Fedora 39
- **Orchestrated via:** Ansible + Semaphore
- **Inventory:** Mixed Debian and RedHat-family homelab hosts

### Planned Improvements

- Migrate `yes/no` boolean values to `true/false` throughout
- Enable and test `serial: 1` for proper rolling update validation
- Add pre-patching snapshot task for Proxmox-hosted VMs
