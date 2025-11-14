# Ubuntu Server Autoinstall Templates

A collection of cloud-init/autoinstall templates for automated Ubuntu Server deployments across multiple cloud providers.

## Overview

This repository contains reusable cloud-init templates for deploying Ubuntu Server instances with pre-configured software, services, and settings. Templates use placeholders that can be replaced at deployment time via Terraform, allowing for secure secrets management and dynamic configuration.

## What is Cloud-Init / Autoinstall?

[Cloud-init](https://cloudinit.readthedocs.io/) is the industry-standard method for cloud instance initialization. It enables automated configuration of Ubuntu Server instances during first boot, including:

- User creation and SSH key management
- Package installation
- Service configuration
- Network setup
- Custom scripts and commands

## Templates

### Available Templates

| Template | Description | Use Case |
|----------|-------------|----------|
| `dagu-worker.yaml` | Dagu distributed workflow worker | Workflow orchestration |

### Template Structure

Each template is a cloud-init configuration file with placeholders for dynamic values:

```yaml
#cloud-config
hostname: HOSTNAME_PLACEHOLDER

users:
  - name: ubuntu
    ssh_authorized_keys:
      - SSH_PUBLIC_KEY_PLACEHOLDER
```

## Placeholder System

Templates use `PLACEHOLDER_NAME` format for values that should be replaced at deployment time:

| Placeholder Pattern | Purpose | Example |
|---------------------|---------|---------|
| `WORKER_NAME_PLACEHOLDER` | Instance/hostname | `web-01`, `db-master` |
| `SSH_PUBLIC_KEY_PLACEHOLDER` | SSH public key | `ssh-ed25519 AAAA...` |
| `TAILSCALE_AUTHKEY_PLACEHOLDER` | Tailscale auth key | `tskey-auth-xxx...` |
| `API_TOKEN_PLACEHOLDER` | API tokens | `sk_live_xxx...` |

### Adding Custom Placeholders

1. Use descriptive, uppercase names with `_PLACEHOLDER` suffix
2. Document all placeholders in template comments
3. Never commit actual secrets - only placeholders

## Usage

### Method 1: Terraform (Recommended)

Use Terraform's `data "http"` and `replace()` functions to fetch and process templates:

```hcl
# Fetch template from GitHub
data "http" "template" {
  url = "https://raw.githubusercontent.com/webmarcus/autoinstall/main/your-template.yaml"
}

# Process placeholders
data "cloudinit_config" "instance" {
  gzip          = false
  base64_encode = false

  part {
    content_type = "text/cloud-config"
    content = replace(
      replace(
        data.http.template.response_body,
        "HOSTNAME_PLACEHOLDER",
        "web-01"
      ),
      "SSH_PUBLIC_KEY_PLACEHOLDER",
      var.ssh_public_key
    )
  }
}

# Deploy instance
resource "hcloud_server" "instance" {
  name      = "web-01"
  user_data = data.cloudinit_config.instance.rendered
  # ...
}
```

### Method 2: Manual Deployment

For one-off deployments without Terraform:

```bash
# 1. Download template
curl -o template.yaml https://raw.githubusercontent.com/webmarcus/autoinstall/main/your-template.yaml

# 2. Replace placeholders
sed -i "s/HOSTNAME_PLACEHOLDER/web-01/g" template.yaml
sed -i "s|SSH_PUBLIC_KEY_PLACEHOLDER|$(cat ~/.ssh/id_ed25519.pub)|g" template.yaml

# 3. Deploy via cloud provider
hcloud server create \
  --name web-01 \
  --type cx21 \
  --image ubuntu-24.04 \
  --user-data-from-file template.yaml
```

## Secrets Management

**Never commit secrets to this repository!** Use placeholders and inject secrets at deployment time.

### Recommended Approach: Bitwarden Secrets Manager

```hcl
# Terraform example with Bitwarden
provider "bws" {
  access_token = var.bws_access_token
}

module "secrets" {
  source = "../modules/bitwarden-secrets"

  secrets = {
    tailscale_authkey = var.bitwarden_secret_id_tailscale
    ssh_public_key    = var.bitwarden_secret_id_ssh
  }
}

# Use secrets in template processing
content = replace(
  data.http.template.response_body,
  "TAILSCALE_AUTHKEY_PLACEHOLDER",
  module.secrets.values["tailscale_authkey"]
)
```

### Alternative: Environment Variables

```bash
# Load secrets from environment
export SSH_KEY="$(cat ~/.ssh/id_ed25519.pub)"
export API_TOKEN="sk_live_xxx"

# Replace in template
envsubst < template.yaml > configured.yaml
```

## Cloud Provider Support

Templates are designed to work with any cloud provider that supports cloud-init:

- **Hetzner Cloud** - Native cloud-init support via `user_data`
- **AWS EC2** - Via `user_data` parameter
- **Google Cloud** - Via `metadata.user-data`
- **Azure** - Via `custom_data` parameter
- **DigitalOcean** - Via `user_data`
- **Linode** - Via `stackscripts` or `metadata`

## Creating New Templates

### Template Guidelines

1. **Start minimal** - Include only essential configuration
2. **Use placeholders** - For any dynamic or sensitive values
3. **Document thoroughly** - Add comments explaining each section
4. **Test locally** - Use `cloud-init schema --config-file` to validate
5. **Keep generic** - Avoid hardcoded values specific to one deployment

### Template Template

```yaml
#cloud-config
# =============================================================================
# Template Name: Your Template Name
# Description: Brief description of what this configures
# =============================================================================
#
# Placeholders:
#   - HOSTNAME_PLACEHOLDER: Instance hostname
#   - SSH_PUBLIC_KEY_PLACEHOLDER: SSH public key for authentication
#   - YOUR_CUSTOM_PLACEHOLDER: Description of your placeholder
#
# =============================================================================

hostname: HOSTNAME_PLACEHOLDER

# User configuration
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - SSH_PUBLIC_KEY_PLACEHOLDER

# System configuration
timezone: Europe/Stockholm
locale: en_US.UTF-8

# Package installation
packages:
  - curl
  - wget
  - git

# Custom commands
runcmd:
  - echo "Instance configured successfully"
```

### Validation

```bash
# Install cloud-init
sudo apt install cloud-init

# Validate template syntax
cloud-init schema --config-file your-template.yaml

# Test locally (dry-run)
cloud-init analyze show
```

## Deployment Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Template Repository (GitHub)                             │
│    - Store templates with placeholders                      │
│    - Version control for all templates                      │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ HTTP GET
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Terraform / Deployment Tool                              │
│    - Fetch template from GitHub                             │
│    - Retrieve secrets from secure storage                   │
│    - Replace placeholders with actual values                │
│    - Generate cloud-init config                             │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ Cloud Provider API
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Cloud Provider                                           │
│    - Create VM with cloud-init as user-data                 │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ VM Boot
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Cloud-Init (First Boot)                                  │
│    - Execute cloud-init configuration                       │
│    - Install packages                                       │
│    - Configure services                                     │
│    - Run custom commands                                    │
│    - Mark deployment complete                               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ 2-10 minutes
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Ready for Use / Post-Configuration                       │
│    - Instance fully configured and operational              │
│    - Ready for Ansible/management tools                     │
└─────────────────────────────────────────────────────────────┘
```

## Post-Deployment Verification

### Check Cloud-Init Status

```bash
# SSH to instance
ssh ubuntu@instance-hostname

# Check cloud-init status
cloud-init status

# Expected output: "status: done"

# View detailed logs
sudo journalctl -u cloud-init-local
sudo journalctl -u cloud-init
sudo journalctl -u cloud-init-config
sudo journalctl -u cloud-init-final

# Check for errors
cloud-init analyze show
```

### Common Issues

**Cloud-init still running**
```bash
# Wait for completion
cloud-init status --wait

# Force completion (not recommended)
sudo cloud-init clean --reboot
```

**Package installation failed**
```bash
# Check cloud-init logs
sudo cat /var/log/cloud-init.log

# Manually retry
sudo cloud-init modules --mode final
```

## Security Best Practices

### DO ✅

- Store secrets in Bitwarden Secrets Manager or similar
- Use placeholders for all sensitive values
- Inject secrets at deployment time via Terraform
- Disable password authentication
- Use SSH key-based authentication only
- Review templates before deploying to production
- Validate cloud-init syntax before deployment

### DON'T ❌

- Commit secrets to version control
- Hardcode API tokens or passwords
- Leave default passwords in templates
- Store unencrypted secrets in cloud provider metadata
- Skip validation steps
- Use root login with password authentication

## Integration Examples

### With Terraform Modules

```hcl
module "web_servers" {
  source = "../modules/compute-hcloud"

  servers = {
    for i in range(3) : "web-${i + 1}" => {
      server_type = "cx21"
      location    = "hel1"
      image       = "ubuntu-24.04"
      user_data   = replace(
        data.http.web_template.response_body,
        "HOSTNAME_PLACEHOLDER",
        "web-${i + 1}"
      )
    }
  }
}
```

### With Ansible Post-Provisioning

```yaml
# After cloud-init completes, run Ansible for advanced configuration
---
- name: Post-provision web servers
  hosts: web_servers
  tasks:
    - name: Wait for cloud-init to complete
      command: cloud-init status --wait
      changed_when: false

    - name: Apply advanced configuration
      # ... your Ansible tasks
```

## Contributing

### Adding a New Template

1. Create template file with descriptive name (e.g., `postgres-primary.yaml`)
2. Use placeholders for all dynamic values
3. Document placeholders in template comments
4. Update this README with template description
5. Test template with cloud-init validation
6. Submit pull request

### Template Naming Convention

- Use lowercase with hyphens
- Be descriptive and specific
- Include role/purpose in name
- Examples: `web-server.yaml`, `redis-cluster.yaml`, `k8s-worker.yaml`

## Resources

- [Cloud-Init Documentation](https://cloudinit.readthedocs.io/)
- [Cloud-Init Examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [Terraform Cloud-Init Provider](https://registry.terraform.io/providers/hashicorp/cloudinit/latest/docs)

## License

MIT

---

**Maintained by**: webmarcus
**Repository**: https://github.com/webmarcus/autoinstall
