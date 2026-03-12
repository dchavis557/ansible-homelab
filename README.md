# ansible-homelab

Ansible playbooks for automating my homelab infrastructure.

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
