# Teleport tbot Service for Ansible Automation Platform

Ansible playbook to install and configure Teleport tbot (Machine ID) as a systemd service on RHEL9 execution nodes used by Ansible Automation Platform.

## Purpose

Deploy Teleport tbot as a systemd service on RHEL9 execution nodes for Ansible Automation Platform. tbot will continuously maintain short-lived SSH certificates in an output directory for bind-mounting into Execution Environments.

## Usage

```bash
ansible-playbook install_tbot.yml \
  -i inventory.ini \
  -e teleport_proxy_addr=sean-test.teleport.sh:443 \
  -e teleport_bot_name=sean-test \
  -e teleport_admin_user=your-username \
  -e teleport_admin_password=your-password \
  -e teleport_mfa_token=123456 \
  [-e teleport_ca_pin=sha256:xxxxx]
```

## Required Extra Variables

- `teleport_admin_user`: Your Teleport admin username
- `teleport_admin_password`: Your Teleport admin password (handled securely with no_log)
- `teleport_mfa_token`: Your 6-digit MFA/TOTP code from authenticator app (handled securely with no_log)

## Optional Variables (with defaults)

- `teleport_proxy_addr`: Teleport proxy address (default: `teleport.example.com:443`)
- `teleport_bot_name`: Bot name for directory structure (default: `aap-bot`)
- `teleport_ca_pin`: CA pin for additional security (optional)
- `teleport_version`: Version to install (default: `17.1.4`)
- `teleport_arch`: Architecture (default: `amd64`)

## Features

- **Idempotent**: Safe to run multiple times across many nodes
- **Secure**: Sensitive data handled with `no_log: true`
- **SELinux**: Contexts set appropriately for RHEL9
- **Systemd**: Production-ready service with restart policies
- **Verification**: Comprehensive health checks and validation

## What It Does

1. Creates a dedicated `tbot` system user and group
2. Sets up directory structure:
   - `/var/lib/teleport-bot/` (base)
   - `/var/lib/teleport-bot/{bot_name}/data` (tbot state)
   - `/var/lib/teleport-bot/{bot_name}/out` (SSH certificates)
3. Installs Teleport binaries (tbot, tsh, tctl) from official repo (with tarball fallback)
4. **Performs admin login** with username/password/MFA
5. **Creates bot and generates join token automatically**
6. Generates tbot YAML configuration with the captured token
7. Creates and enables systemd service
8. Verifies service is running and certificates are generated
9. Cleans up admin session for security

## Next Steps

After running the playbook:

1. Verify certificates: `ls -la /var/lib/teleport-bot/{bot_name}/out`
2. Check logs: `journalctl -u tbot -f`
3. Configure AAP to bind-mount the output directory into Execution Environments
4. Use the SSH identity for connections to Teleport-protected hosts

Example SSH usage:
```bash
ssh -F /var/lib/teleport-bot/{bot_name}/out/ssh_config user@target-host
```