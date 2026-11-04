# ArpScayl Systems Linux Provisioning with Ansible

Reusable Ansible toolkit for applying and verifying a conservative Linux server baseline over SSH. It is designed for Ubuntu Server 24.04 LTS and Debian 12. RHEL-compatible support is not implemented.

## Scope

The baseline validates OS support, can optionally set hostname/timezone/locale, refreshes package indexes, installs approved packages, and verifies selected services. User, SSH, UFW firewall, unattended security update, time-sync, journald-retention, and sysctl controls are opt-in. The monitoring role is a disabled-by-default interface and fails closed when enabled because no provider adapter has been selected or implemented. Full distribution upgrades and firewall resets are not implemented. Audit is read-only; package maintenance and reboot scheduling are disabled unless separately authorized. The unattended-upgrades role can schedule reboots only after separate explicit confirmation.

## Repository layout

- `playbooks/`: baseline entry points (`site.yml`, `provision.yml`) and read-only verification (`verify.yml`).
- `roles/`: modular common, package management, users, SSH, firewall, updates, time sync, logging, sysctl, monitoring interface, and verification roles.
- `inventories/`: non-routable documentation examples for development, staging, and production.
- `group_vars/`: safe shared defaults and a Vault example.
- `.github/workflows/`: validation-only CI; it never connects to infrastructure.

## Prerequisites and installation

Use Python 3.12 and a virtual environment. Dependency constraints are in `requirements.txt`; the repository was prepared against ansible-core 2.20.7. Install with:

```bash
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
ansible --version
```

The base roles use built-in modules. User public-key management and sysctl use the pinned `ansible.posix` collection; UFW rules and opt-in locale generation use the pinned `community.general` collection.

## Inventory and variables

Example hosts use the reserved TEST-NET range and cannot represent reachable client systems. Replace them with an inventory stored outside this repository, or provide a separate private inventory path. Never put real addresses, credentials, or client data in the shared template. SSH credentials should use an agent or an approved secrets manager; do not put passwords in inventory.

Set environment values in `inventories/<environment>/group_vars/all.yml` for private copies. Hostname, timezone, and locale management are disabled by default. Package removal defaults to an empty list. The example production inventory is inert documentation only.

Use Ansible Vault for secrets when an external manager is not available. Encrypt the file before storing it:

```bash
ansible-vault create inventories/private/group_vars/all/vault.yml
ansible-vault encrypt_string --stdin-name 'example_secret'
```

Do not commit Vault password files. Store Vault credentials outside the repository and protect them with restrictive permissions.

## Validation and tests

```bash
ansible --version
ansible-inventory -i inventories/development/hosts.yml --list
ansible-playbook -i inventories/development/hosts.yml playbooks/site.yml --syntax-check
ansible-playbook -i inventories/development/hosts.yml playbooks/verify.yml --syntax-check
ansible-playbook -i /secure/private/inventory.yml playbooks/audit.yml --check
yamllint .
ansible-lint
pytest -q
```

For a local static check, use the example inventories; syntax checks do not connect to hosts. Integration testing requires a disposable Ubuntu 24.04 or Debian 12 VM/container and is not performed against client servers. The suite includes repository and audit read-only structure checks, localhost-only access/policy/monitoring rejection checks, and a maintenance no-op run. These do not configure a host. UFW, systemd, journald, unattended upgrades, locale generation, and kernel behavior still need a disposable supported VM for integration verification.

CI secret scanning uses the Gitleaks CLI container pinned by version and image digest. CI has not yet been run for this repository; see [production readiness](docs/production-readiness.md) for verified checks and remaining gaps.

## Reviewing and running

Provisioning a new server means preparing SSH access and applying the baseline. Enforcing a baseline on an existing server applies the same settings and should follow a change review. `audit.yml` runs read-only checks against the declared baseline; `maintenance.yml` supports explicitly authorized ordinary package upgrades and a separately authorized reboot; `verify.yml` checks OS support and the configured operational state, including any enabled locale, SSH, firewall, update, time-sync, logging, and sysctl policies.

For deployments, require an explicit private inventory, approved credentials, reviewed variables, operator authorization, and a maintenance/recovery plan. First review the predicted changes:

```bash
ansible-playbook -i /secure/private/inventory.yml playbooks/provision.yml --check --diff
```

Then apply only after reviewing and authorizing the change:

```bash
ansible-playbook -i /secure/private/inventory.yml playbooks/provision.yml
ansible-playbook -i /secure/private/inventory.yml playbooks/verify.yml
```

`--check` does not guarantee safety or predict every module or external-state effect. The audit and verify playbooks are read-only assessment entry points. Audit runs in Ansible check mode and its state probes are explicitly read-only; `--check` alone is not a guarantee of safety for enforcement playbooks. `--diff` may expose sensitive configuration in terminal output and CI logs; avoid enabling it where output is retained broadly. For SSH and user changes, keep the current session open and review [rollback and recovery](docs/rollback-and-recovery.md).

## Troubleshooting and recovery

- Confirm OpenSSH connectivity and administrative privilege escalation before applying changes.
- Unsupported OS versions fail early with a clear message.
- If a package task fails, inspect APT repository and DNS state on the disposable target before retrying.
- Keep an out-of-band console path for SSH, account, or firewall changes.
- See `docs/installation.md`, `docs/architecture.md`, `docs/troubleshooting.md`, and `docs/network-and-os-baseline.md` for platform details.

## Phase 2 access management

Set `users_accounts` only for approved accounts. Public keys and sudo-group membership are optional per account; passwords are not accepted. Account disabling/removal needs `users_allow_account_removal: true`, and removal also requires a surviving operator-verified administrator. The initial provisioning account is protected from disabling/removal. Review [the users role guide](roles/users/README.md).

SSH management defaults off. To enable it, set `ssh_hardening_manage: true` in the private inventory. Password authentication stays enabled by default. Enabling any SSH changes requires an operator attestation that a configured sudo administrator can connect. Disabling password authentication additionally requires a public key tested in a separate session. A port change requires a verified external path and a matching, approved rule in the opt-in firewall role. Keep the current session open and follow [rollback and recovery](docs/rollback-and-recovery.md).

Run the no-host access guardrail checks with:

```bash
ansible-playbook -i localhost, tests/integration/access_guardrails.yml
```

Review and check-mode a hardening run against an approved private inventory before authorizing it:

```bash
ansible-playbook -i /secure/private/inventory.yml playbooks/harden.yml --check --diff
```


## Phase 4 audit, maintenance, and verification

Run an audit with an explicit private inventory:

```bash
ansible-playbook -i /secure/private/inventory.yml playbooks/audit.yml --check
ansible-playbook -i /secure/private/inventory.yml playbooks/verify.yml
```

The audit playbook gathers declared package, user, hostname/timezone/locale, service, SSH, firewall, update, time-sync, logging, sysctl, and listening-port state without applying changes. Set `verification_required_services` and `verification_required_listening_ports` for environment-specific checks. Output labels missing policy, failed checks, checks that could not run, and checks that do not apply.

Maintenance does nothing by default. A private, reviewed maintenance vars file must set `maintenance_updates_enabled` and `maintenance_updates_authorized` to true. The playbook only runs ordinary `apt upgrade`; full distribution upgrades are not supported. A reboot requires separate authorization, a confirmed window, and a reason. Review check mode first, then execute the approved run inside that window.

## Phase 3 network and OS controls

Phase 3 changes are opt-in. UFW requires a reviewed policy, an explicit source-restricted SSH allow rule, and an installed UFW package; it adds approved rules before default-deny inbound and never resets existing rules. Automatic security updates retain the operating system's vendor security-origin settings. Reboot scheduling requires separate confirmation. Time sync uses `systemd-timesyncd` and fails if a competing daemon is active. Logging only checks journald unless retention management is enabled. Sysctl accepts only the documented pointer/log disclosure settings and does not change forwarding or IPv6. See [network and OS baseline notes](docs/network-and-os-baseline.md).

To review policy guardrails locally without changing a server:

```bash
ansible-playbook -i localhost, tests/integration/phase3_guardrails.yml
```

## Client onboarding and readiness

Use the repeatable, client-data-safe process in [client onboarding](docs/client-onboarding.md) before preparing a client inventory. The current [production readiness assessment](docs/production-readiness.md) identifies which validations have actually run and what remains before an initial release. The monitoring role is only an interface; no agent is installed.
# arpscayl-ansible-template
