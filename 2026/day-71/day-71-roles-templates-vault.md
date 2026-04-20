# Day 71 — Ansible Roles, Jinja2 Templates, Galaxy & Vault

## Overview

Today covered four pillars of production-grade Ansible automation:

- **Ansible Roles** — structured, reusable automation units
- **Jinja2 Templates** — dynamic config file generation
- **Ansible Galaxy** — community role marketplace
- **Ansible Vault** — AES256 secrets encryption

---

## Task 1: Jinja2 Templates

Templates use the `.j2` extension (Jinja2). They let you generate config files dynamically using variables and Ansible facts gathered at runtime.

### Key Jinja2 Syntax

| Syntax | Purpose |
|---|---|
| `{{ variable }}` | Render a variable value |
| `{% if condition %}` | Conditional block |
| `{% for item in list %}` | Loop block |
| `\| default(value)` | Fallback if variable is undefined |

### nginx-vhost.conf.j2

```jinja2
# Managed by Ansible -- do not edit manually
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

Variables like `ansible_hostname` and `ansible_default_ipv4.address` are **Ansible facts** — gathered automatically from the target host. No manual input needed.

---

## Task 2: Ansible Role Structure

Generated with:
```bash
ansible-galaxy init roles/webserver
```

### Directory Structure

```
roles/webserver/
├── README.md              # Auto-generated documentation template
├── defaults/
│   └── main.yml           # Default variables (LOWEST priority)
├── handlers/
│   └── main.yml           # Handlers (e.g. Restart Nginx)
├── meta/
│   └── main.yml           # Role metadata and Galaxy dependencies
├── tasks/
│   └── main.yml           # Main task list (loaded automatically)
├── templates/             # Jinja2 templates (created manually)
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml           # Role variables (HIGH priority)
```

### defaults/main.yml vs vars/main.yml

| | `defaults/main.yml` | `vars/main.yml` |
|---|---|---|
| Priority | Lowest in Ansible | High in Ansible |
| Purpose | Fallback values, easily overridden by callers | Internal constants, should not be overridden |
| Use case | `http_port: 80` — callers can change this | Package names, internal paths |
| Override? | Yes — playbook vars, inventory, CLI all win | Only very few things can override it |

**Rule of thumb:** Use `defaults/` for 90% of variables. Use `vars/` only for internal role constants that must not change.

---

## Task 3: Custom Webserver Role

### Role Files Created

**defaults/main.yml**
```yaml
http_port: 80
app_name: myapp
max_connections: 512
```

**tasks/main.yml** — 6 tasks:
1. Install Nginx
2. Deploy Nginx config from template
3. Deploy vhost config from template
4. Create web root directory
5. Deploy index page from template
6. Start and enable Nginx service

**handlers/main.yml**
```yaml
- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

**Templates created:**

- `nginx.conf.j2` — Main Nginx config using `{{ max_connections }}`
- `vhost.conf.j2` — Virtual host config using `{{ http_port }}`, `{{ app_name }}`, `{{ ansible_hostname }}`
- `index.html.j2` — Dynamic HTML using facts and variables

### Calling the Role (site.yml)

```yaml
- name: Configure web servers
  hosts: app
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80
```

### Rendered Output (curl http://localhost)

```html
<h1>terraweek</h1>
<p>Server: ip-10-0-1-242</p>
<p>IP: 10.0.1.242</p>
<p>Environment: development</p>
<p>Managed by Ansible</p>
```

Variables resolved at runtime:
- `app_name` → overridden from playbook (`terraweek`)
- `ansible_hostname` → gathered fact (`ip-10-0-1-242`)
- `ansible_default_ipv4.address` → gathered fact (`10.0.1.242`)
- `app_env` → used `| default('development')` fallback since not set

---

## Task 4: Ansible Galaxy

### Useful Commands

```bash
# Search for roles
ansible-galaxy search nginx --platforms EL   # 805 results
ansible-galaxy search mysql                  # 692 results

# Install a role
ansible-galaxy install geerlingguy.docker

# Install specific version
ansible-galaxy install "geerlingguy.docker,7.4.1" --force

# List installed roles
ansible-galaxy list

# Install from requirements file
ansible-galaxy install -r requirements.yml
```

### requirements.yml

```yaml
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
  - name: geerlingguy.ntp
```

### Why requirements.yml over manual installs?

- **Version-pinned** — no surprise breaking changes between team members
- **Single command** — `ansible-galaxy install -r requirements.yml` sets up everything
- **CI/CD friendly** — pipelines run this before every playbook execution
- **Self-documenting** — acts as a manifest of all role dependencies
- **Idempotent** — skips already-installed roles at correct versions

### geerlingguy.docker role highlights

- Auto-detected RedHat family → used `setup-RedHat.yml`
- Added Docker GPG key and repository
- Installed Docker packages
- Started and enabled Docker service
- Version installed: `7.4.1` (8.0.0 had a breaking change with older Ansible)

---

## Task 5: Ansible Vault

### Vault Workflow

```bash
# Create encrypted file
ansible-vault create group_vars/db/vault.yml

# View without editing
ansible-vault view group_vars/db/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/db/vault.yml

# Encrypt an existing plain file
ansible-vault encrypt group_vars/db/secrets.yml

# Decrypt a file permanently
ansible-vault decrypt group_vars/db/vault.yml

# Encrypt a single inline string
ansible-vault encrypt_string 'MySecret' --name 'vault_db_password'
```

### Vault file contents (before encryption)

```yaml
vault_db_password: SuperSecretP@ssw0rd
vault_db_root_password: R00tP@ssw0rd123
vault_api_key: sk-abc123xyz789
```

### After encryption (AES256)

```
$ANSIBLE_VAULT;1.1;AES256
31363130366237663164323164383266373565663938656539393038...
```

The encrypted blob changes every time the file is saved (fresh IV) — this is a security feature of AES256.

### Running playbooks with vault

```bash
# Interactive (not suitable for CI/CD)
ansible-playbook db-setup.yml --ask-vault-pass

# Password file (preferred for automation)
ansible-playbook db-setup.yml --vault-password-file .vault_pass
```

### Password file setup

```bash
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass          # Only owner can read
echo ".vault_pass" >> .gitignore  # NEVER commit to Git
```

### ansible.cfg integration

```ini
[defaults]
vault_password_file = .vault_pass
inventory = hosts.ini
```

No flags needed after this — Ansible reads vault automatically.

### Why --vault-password-file over --ask-vault-pass for CI/CD?

`--ask-vault-pass` requires a human to type the password interactively. Jenkins, GitHub Actions, and GitLab CI cannot do this. `--vault-password-file` reads from a file automatically, enabling fully unattended pipeline runs. The password file itself is protected by `chmod 600` and stored as a CI/CD secret — never committed to Git.

---

## Task 6: Combined site.yml

Final playbook combining all 4 concepts:

```yaml
---
- name: Configure web servers          # Uses custom webserver ROLE
  hosts: app
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80

- name: Configure app servers with Docker   # Uses GALAXY role
  hosts: app
  become: true
  roles:
    - geerlingguy.docker

- name: Configure database servers     # Uses TEMPLATE + VAULT secrets
  hosts: db
  become: true
  tasks:
    - name: Create DB config with secrets
      template:
        src: templates/db-config.j2
        dest: /etc/db-config.env
        owner: root
        mode: '0600'
```

### db-config.j2 rendered output on db server

```
# Database Configuration -- Managed by Ansible
DB_HOST=10.0.1.161
DB_PORT=3306
DB_PASSWORD=SuperSecretP@ssw0rd
DB_ROOT_PASSWORD=R00tP@ssw0rd123
```

File permission: `-rw------- root root` (600) — only root can read it ✅

---

## When to use what?

| Approach | Use when |
|---|---|
| **Ad-hoc commands** | Quick one-off tasks, checking facts, testing connectivity |
| **Playbooks** | Multi-step automation for a specific workflow |
| **Roles** | Reusable automation shared across multiple playbooks/projects |
| **Galaxy roles** | Standard software installs (Docker, Nginx, MySQL) — don't reinvent the wheel |

---

## Key Takeaways

- Roles enforce **standard structure** — every team member knows where to look
- `defaults/` makes roles **flexible**; `vars/` makes them **predictable**
- Jinja2 templates + Ansible facts = **zero hardcoded values** in config files
- Galaxy roles save hours — `geerlingguy.docker` installs Docker with 4 lines of YAML
- Vault + password file = **secrets never in plain text**, pipelines fully automated
- `requirements.yml` is your role dependency manifest — always version-pin in production
