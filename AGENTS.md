# Project Context: ansible-role-postgresql

**Type:** Ansible Role / Infrastructure Automation  
**Target:** Debian 12 (Bookworm), Debian 13 (Trixie), Ubuntu 22.04 (Jammy), Ubuntu 24.04 (Noble)  
**Service:** PostgreSQL 16 (configurable 14-17) using upstream PGDG packages, streaming replication, automated systemd backups, PgBouncer connection pooling, and Prometheus `postgres_exporter`.

---

## Layout

Standard standalone Ansible project structure wiring shared-inventory:
- `ansible.cfg` (wires `../shared-inventory/inventory/hosts.yml` and `inventory/hosts.yml`)
- `requirements.yml` (`community.postgresql`, `community.general`)
- `inventory/`
  - `hosts.yml` (Pre-configured groups: `postgresql_servers`, `postgresql_primary`, `postgresql_standby`)
  - `group_vars/postgresql_servers/main.yml`
- `playbooks/`
  - `site.yml` (Complete deployment: PGDG repo, packages, configuration, databases, users, replication, backups, pooler, exporter)
  - `deploy.yml` (Rapid configuration/database/user/extension deployment skipping package installation)
  - `backup.yml` (On-demand manual backup trigger)
- `roles/postgresql/` (Core role tasks, handlers, templates, defaults, and OS vars)

---

## Common Commands

```bash
# Syntax check
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml --syntax-check
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/backup.yml --syntax-check

# Dry-run / Check mode
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff

# Full deployment
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# Rapid configuration or schema update
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml

# Trigger manual backup
uv run --with ansible-core ansible-playbook -i inventory/hosts.yml playbooks/backup.yml
```

---

## Architecture & Conventions

- **Packaging**: Default upstream PGDG repository (`apt.postgresql.org`) with official GPG keyring at `/etc/apt/keyrings/postgresql.gpg`.
- **Drop-in Configuration**: Custom settings are decoupled in `/etc/postgresql/{{ version }}/main/conf.d/99-ansible-custom.conf`.
- **Security**: Default password hashing is `scram-sha-256`. Client authentication rules (`pg_hba.conf`) are fully templated.
- **Automated Backups**: Backups are run via systemd service (`postgresql-backup.service`) scheduled by timer (`postgresql-backup.timer`), creating gzipped dumps and retaining them for `postgresql_backup_retention_days`.
- **Streaming Replication**: Physical streaming replication with dedicated replication user and physical replication slots. Standbys bootstrapped via `pg_basebackup`.
- **Connection Pooling & Metrics**: Optional modular tasks for PgBouncer (`:6432`) and Prometheus `postgres_exporter` (`:9187`).
