# Automate Linux Configuration with Ansible

[![CI](https://github.com/kioji94/ansible-linux-automation/actions/workflows/ci.yaml/badge.svg)](https://github.com/kioji94/ansible-linux-automation/actions/workflows/ci.yaml)

Infrastructure as Code (IaC) project that automates common Linux administration tasks with Ansible: **package installation, user creation, service management and system updates**. Every run is idempotent, so running the playbook twice produces the same result and the second run changes nothing.

**Skills covered:** Ansible · Infrastructure Automation · YAML · Linux · GitHub Actions

---

## Project structure

```
ansible-linux-automation/
├── .github/workflows/ci.yaml     # CI: lint → apply → idempotence → verify
├── ansible.cfg                   # Ansible defaults (inventory, sudo, roles path)
├── inventory/
│   ├── hosts.ini                 # localhost + placeholder remote hosts
│   └── group_vars/all.yml        # ALL configurable variables live here
├── site.yml                      # Master playbook (runs every role)
├── roles/
│   ├── updates/                  # apt cache refresh, optional dist-upgrade, reboot check
│   ├── packages/                 # install / remove packages
│   ├── users/                    # groups, users, SSH keys, sudoers
│   └── services/                 # install + enable + start services
├── tests/verify.yml              # Post-run assertions (packages, users, services)
├── requirements.yml              # Ansible collections (ansible.posix)
├── requirements-dev.txt          # ansible-core, ansible-lint, yamllint
├── .yamllint / .ansible-lint     # Lint configuration
└── README.md
```

## What each role does

| Role | Tasks | Key modules |
|------|-------|-------------|
| `updates` | Refresh apt cache, optional `dist-upgrade`, autoremove, detect and optionally perform reboot | `apt`, `stat`, `reboot` |
| `packages` | Install a common toolset (`curl`, `git`, `vim`, `htop`, `jq`...), remove unwanted packages | `apt` |
| `users` | Create groups and users, authorize SSH keys, grant sudo through a `visudo`-validated drop-in | `group`, `user`, `authorized_key`, `copy` |
| `services` | Install packages, enable at boot, ensure running (nginx, cron by default) | `apt`, `service` |

All behavior is driven by `inventory/group_vars/all.yml`. Edit the lists there, not the tasks.

## Prerequisites

- Python 3.10+ and `ansible-core` 2.17+
- A Debian/Ubuntu target (local machine, VM or remote server) with `sudo`
- SSH access to remote targets

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

## Usage

```bash
# Run everything on localhost
ansible-playbook site.yml -l local --ask-become-pass

# Run one area only (tags: updates, packages, users, services)
ansible-playbook site.yml --tags users
ansible-playbook site.yml --tags "packages,services"

# Dry run: show what would change (needs packages installed first)
ansible-playbook site.yml --tags users --check --diff

# Full system upgrade (off by default)
ansible-playbook site.yml --tags updates -e system_upgrade=true

# Allow automatic reboot when /var/run/reboot-required exists
ansible-playbook site.yml --tags updates -e "system_upgrade=true updates_allow_reboot=true"

# Target remote servers (define them in inventory/hosts.ini first)
ansible-playbook site.yml -l webservers

# Verify the final state
ansible-playbook tests/verify.yml
```

Useful ad-hoc commands:

```bash
ansible all -m ping
ansible all -m setup -a "filter=ansible_distribution*"
ansible-inventory --graph
ansible-playbook site.yml --list-tasks
```

## Configuration example

`inventory/group_vars/all.yml`:

```yaml
common_packages: [curl, git, vim, htop, jq]

managed_users:
  - name: deploy
    groups: [devops]
    sudo: true
    ssh_key: "ssh-ed25519 AAAA... you@laptop"

managed_services:
  - { name: nginx, package: nginx, state: started, enabled: true }
```

## CI pipeline (`.github/workflows/ci.yaml`)

Runs on every push and pull request to `main`, and can be started manually (`workflow_dispatch`).

| Job | Step | Purpose |
|-----|------|---------|
| `lint` | `yamllint` | YAML style and syntax |
| `lint` | `ansible-lint` (production profile) | Ansible best practices |
| `test` | `--syntax-check` | Playbook parses correctly |
| `test` | Apply playbook | Real run against the GitHub runner (`localhost`) |
| `test` | Second run, require `changed=0` | **Idempotence** proof |
| `test` | `tests/verify.yml` | Asserts packages, users, sudoers and services are in the desired state |

Why no `--check` step in CI: on a fresh runner, check mode skips the nginx install, so the following `service` task would fail for a package that was never installed. The apply + idempotence + verify sequence is a stronger test anyway.

## Design notes and lessons learned

- **Idempotence:** modules like `apt`, `user` and `service` describe a desired state rather than run commands, which is the core of IaC. The CI second-run check enforces this.
- **Python interpreter:** the `apt` module needs `python3-apt`, which only exists in the system Python. `inventory/hosts.ini` pins `/usr/bin/python3` for localhost so CI works even when `actions/setup-python` puts another Python first in `PATH`.
- **Safe sudo:** sudoers drop-ins are written with `validate: /usr/sbin/visudo -cf %s` so a typo can never lock you out.
- **Safe defaults:** full upgrades and reboots are opt-in flags (`system_upgrade`, `updates_allow_reboot`).
- **Tags** let you apply one concern at a time, useful when debugging.
- **Role-prefixed variables** (`updates_*`) satisfy `ansible-lint` and avoid collisions.


## Ideas to extend

- Add a `ssh_hardening` role (disable root login and password auth) using `lineinfile` + a handler to restart `sshd`
- Add UFW firewall rules and fail2ban
- Use Jinja2 templates (`ansible.builtin.template`) for an nginx virtual host
- Encrypt secrets with `ansible-vault`
- Test across Ubuntu 22.04 / 24.04 / Debian 12 with Molecule + Docker
- Support RHEL-family hosts with `dnf` and `when: ansible_facts['os_family']` branching

## License

MIT
