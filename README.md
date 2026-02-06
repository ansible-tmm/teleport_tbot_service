# Teleport tbot Service for Ansible Automation Platform

Ansible playbook to install and configure Teleport tbot (Machine ID) as a systemd service on RHEL9 execution nodes used by Ansible Automation Platform.

## Purpose

Deploy Teleport tbot as a systemd service on RHEL9 execution nodes for Ansible Automation Platform. tbot continuously maintains short-lived SSH certificates in an output directory for bind-mounting into Execution Environments.

## Prerequisites

### Teleport Administrator Must Complete (One-Time Setup)

Before running this playbook, a Teleport administrator must:

1. **Create an SSH access role** (e.g., `aap-ssh`) in Teleport with:
   - SSH login permissions for target hosts
   - Appropriate host labels and user mappings
   - Node access configuration

2. **Configure bot impersonation** - The bot role (auto-created as `bot-{bot_name}`) must allow impersonation of the access role:
   ```yaml
   kind: role
   version: v7
   metadata:
     name: bot-aap-bot
   spec:
     allow:
       impersonate:
         roles:
           - aap-ssh
   ```

**Why this matters:** Teleport uses a two-role model for bots:
- **Bot role** (`bot-aap-bot`): Bot's authentication identity
- **Access role** (`aap-ssh`): Permissions the bot impersonates for SSH access

The playbook creates the bot automatically but **cannot** create roles or configure impersonation permissions.

## Usage

```bash
ansible-playbook install_tbot.yml \
  -i inventory.ini \
  -e teleport_proxy_addr=sean-test.teleport.sh:443 \
  -e teleport_bot_name=aap-bot \
  -e teleport_access_role=aap-ssh \
  -e teleport_admin_user=your-username \
  -e teleport_admin_password=your-password \
  -e teleport_mfa_token=123456
```

## Required Extra Variables

- `teleport_admin_user`: Your Teleport admin username
- `teleport_admin_password`: Your Teleport admin password (handled securely with no_log)
- `teleport_mfa_token`: Your 6-digit MFA/TOTP code from authenticator app (handled securely with no_log)

## Optional Variables (with defaults)

- `teleport_proxy_addr`: Teleport proxy address (default: `teleport.example.com:443`)
- `teleport_bot_name`: Bot name for directory structure (default: `aap-bot`)
- `teleport_access_role`: Role that bot impersonates for SSH access (default: `aap-ssh`)
- `teleport_ca_pin`: CA pin for additional security (optional)
- `teleport_version`: Version to install (default: `17.1.4`)
- `teleport_arch`: Architecture (default: `amd64`)

## What It Does

1. Creates a dedicated `tbot` system user and group
2. Sets up directory structure:
   - `/var/lib/teleport-bot/` (base)
   - `/var/lib/teleport-bot/{bot_name}/data` (tbot state)
   - `/var/lib/teleport-bot/{bot_name}/out` (SSH certificates)
3. Installs Teleport binaries (tbot, tsh, tctl) from CDN tarball
4. **Performs admin login** with username/password/MFA
5. **Creates bot and generates join token automatically**
6. Generates tbot.yaml configured to:
   - Authenticate as `bot-{bot_name}`
   - Request impersonation of the access role
   - Write SSH identity artifacts to output directory
7. Creates and enables systemd service
8. Verifies service is running and certificates are generated
9. Cleans up admin session for security

## How It Works (Teleport RBAC Model)

1. **Bot authentication**: tbot authenticates as `bot-{bot_name}` (auto-created by Teleport)
2. **Role impersonation**: tbot requests to impersonate `{access_role}` (e.g., `aap-ssh`)
3. **Permission check**: Teleport verifies the bot role is allowed to impersonate the access role
4. **Certificate generation**: Teleport issues short-lived SSH certificates with `{access_role}` permissions
5. **File output**: Certificates are written to `/var/lib/teleport-bot/{bot_name}/out/`
6. **Continuous renewal**: tbot automatically refreshes certificates before expiration

**Key point:** Impersonation permissions are configured in Teleport RBAC, not in tbot.yaml or this playbook.

## Next Steps

After running the playbook:

1. Verify certificates exist:
   ```bash
   sudo ls -la /var/lib/teleport-bot/{bot_name}/out/
   ```

2. Check tbot service logs:
   ```bash
   sudo journalctl -u tbot -f
   ```

3. Test SSH access using the identity:
   ```bash
   sudo tsh \
     --identity=/var/lib/teleport-bot/{bot_name}/out/identity \
     --proxy={teleport_proxy_addr} \
     ssh user@target-host
   ```

4. Configure AAP to bind-mount `/var/lib/teleport-bot/{bot_name}/out` into Execution Environments

5. Use the SSH identity in Ansible playbooks for connections to Teleport-protected hosts