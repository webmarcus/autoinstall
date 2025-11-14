# Ubuntu Server Autoinstall - Dagu Worker (Terraform Template)

Autoinstall-konfiguration för Ubuntu Server som används som Terraform-template för att deploya Dagu Worker-noder.

## Översikt

Detta projekt tillhandahåller en autoinstall-konfiguration för Ubuntu Server som används tillsammans med Terraform för automatisk deployment av Dagu Worker-noder. Alla känsliga credentials hanteras via Bitwarden Secrets Manager och injiceras av Terraform vid deployment-tillfället.

## Deployment-flöde

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Terraform (Lokal maskin)                                │
│    - Hämtar secrets från Bitwarden                         │
│    - Läser worker.yaml template                            │
│    - Ersätter placeholders med secrets                     │
│    - Deployer VM med modifierad config                     │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ Cloud Provider API (Hetzner/AWS/GCP)
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Cloud Provider                                           │
│    - Skapar VM med worker.yaml som user-data              │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ VM Boot
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Autoinstall (First Boot)                                │
│    - Installerar Ubuntu Server (minimal)                   │
│    - Konfigurerar timezone, locale, keyboard               │
│    - Skapar användare (marcus)                             │
│    - Installerar Tailscale och registrerar i mesh          │
│    - Installerar Docker och Docker Compose                 │
│    - Installerar Dagu worker (headless mode)               │
│    - Konfigurerar systemd services                         │
│    - Startar alla services                                 │
│    - Skapar status marker: /var/lib/dagu-worker/status    │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ 5-10 minuter
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Redo för Ansible!                                       │
│    - Worker synlig i Tailscale mesh (tag:dagu-worker)     │
│    - Auto-discoverable via Ansible dynamic inventory       │
│    - Klar för NFS mount och final konfiguration           │
└─────────────────────────────────────────────────────────────┘
```

## Förutsättningar

### 1. Cloud Provider Account

- **Hetzner Cloud** (primär): [console.hetzner.cloud](https://console.hetzner.cloud/)
- **AWS EC2** (sekundär): AWS account med EC2 access
- **GCP Compute** (sekundär): GCP project med Compute Engine API

### 2. Terraform

```bash
# macOS
brew install terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

### 3. Bitwarden Secrets Manager

- Aktivt Bitwarden-konto med Secrets Manager
- Machine Account skapad
- Access Token genererat
- Bitwarden SDK installerat: `pip3 install bitwarden-sdk`

### 4. Secrets i Bitwarden

Följande secrets måste finnas i Bitwarden:

| Secret Name | Beskrivning | Exempel |
|-------------|-------------|---------|
| `tailscale_authkey_workers` | Tailscale ephemeral auth key med tag:dagu-worker | `tskey-auth-xxx...` |
| `SSH Public Key` (optional) | Din SSH public key | `ssh-ed25519 AAAA...` |

## Placeholders i worker.yaml

Terraform ersätter följande placeholders med faktiska värden:

| Placeholder | Ersätts med | Källa |
|-------------|-------------|-------|
| `WORKER_NAME_PLACEHOLDER` | Unique worker namn | Terraform variable |
| `TAILSCALE_AUTHKEY_PLACEHOLDER` | Tailscale auth key | Bitwarden secret |
| `SSH_PUBLIC_KEY_PLACEHOLDER` | SSH public key | Bitwarden secret eller Terraform variable |

## Deployment

### Metod 1: Via Terraform (Rekommenderat)

**Steg 1**: Konfigurera Terraform

```bash
cd terraform/dagu-worker

# Kopiera example config
cp terraform.tfvars.example terraform.tfvars

# Uppdatera med dina Bitwarden Secret IDs
vim terraform.tfvars
```

**Steg 2**: Exportera Environment Variables

```bash
# Hetzner Cloud token
export HCLOUD_TOKEN="your-hetzner-token"

# Bitwarden access token
export BWS_ACCESS_TOKEN="your-bitwarden-token"
```

**Steg 3**: Deploy Worker

```bash
# Använd deployment script
./deploy-worker.sh dagu-worker-01

# Eller direkt med Terraform
terraform init
terraform plan -var="worker_name=dagu-worker-01"
terraform apply -var="worker_name=dagu-worker-01"
```

Se [Terraform README](../terraform/dagu-worker/README.md) för fullständig dokumentation.

### Metod 2: Via Cloud Provider UI (Manuell)

Om du vill deploya manuellt utan Terraform:

**Steg 1**: Hämta secrets från Bitwarden

```bash
# Installera bws CLI
pip3 install bitwarden-sdk

# Exportera access token
export BWS_ACCESS_TOKEN="your-token"

# Hämta Tailscale key
TAILSCALE_KEY=$(bws secret get a71eebe6-fc6f-4bfb-969a-b38c016e0795 --output json | jq -r '.value')

# Hämta SSH key (optional)
SSH_KEY=$(bws secret get your-ssh-key-secret-id --output json | jq -r '.value')
```

**Steg 2**: Ersätt placeholders i worker.yaml

```bash
# Skapa working copy
cp autoinstall/worker.yaml /tmp/worker-configured.yaml

# Ersätt placeholders
sed -i "s/WORKER_NAME_PLACEHOLDER/dagu-worker-01/g" /tmp/worker-configured.yaml
sed -i "s|TAILSCALE_AUTHKEY_PLACEHOLDER|${TAILSCALE_KEY}|g" /tmp/worker-configured.yaml
sed -i "s|SSH_PUBLIC_KEY_PLACEHOLDER|${SSH_KEY}|g" /tmp/worker-configured.yaml
```

**Steg 3**: Deploy via cloud provider

```bash
# Hetzner Cloud
hcloud server create \
  --name dagu-worker-01 \
  --type cx31 \
  --location hel1 \
  --image ubuntu-24.04 \
  --user-data-from-file /tmp/worker-configured.yaml

# AWS EC2
aws ec2 run-instances \
  --image-id ami-xxx \
  --instance-type t3.medium \
  --user-data file:///tmp/worker-configured.yaml

# GCP Compute
gcloud compute instances create dagu-worker-01 \
  --machine-type n1-standard-2 \
  --image-family ubuntu-2404-lts \
  --image-project ubuntu-os-cloud \
  --metadata-from-file user-data=/tmp/worker-configured.yaml
```

## Post-Installation

### 1. Vänta på Autoinstall (5-10 minuter)

Workern bootar och kör autoinstall. Detta tar cirka 5-10 minuter.

```bash
# Följ via Tailscale
tailscale status | grep dagu-worker-01

# Efter 2-3 minuter ska workern dyka upp
```

### 2. Verifiera Installation

```bash
# SSH till workern (via Tailscale)
ssh dagu-worker-01

# Kontrollera deployment status
cat /var/lib/dagu-worker/status

# Expected output:
# DEPLOYED_AT=2025-11-12T10:30:00Z
# WORKER_NAME=dagu-worker-01
# STATUS=ready
```

### 3. Verifiera Services

```bash
# Tailscale
tailscale status

# Docker
sudo systemctl status docker
docker ps

# Dagu
sudo systemctl status dagu
sudo journalctl -u dagu -f

# Dagu config
cat /etc/dagu/config.yaml
```

### 4. Kör Ansible Provisioning

```bash
cd ansible/ansible-playbooks

# Verifiera att workern är discoverable
ansible-inventory --graph | grep dagu-worker-01

# Kör provisioning
ansible-playbook dagu-workers.yml --limit dagu-worker-01
```

Ansible konfigurerar:
- NFS client och mount coordinator shares
- Fine-tuning av Dagu konfiguration
- Monitoring och logging

## Vad Installeras

### Packages
- `python3`, `python3-pip`, `git`, `curl`, `wget`
- `docker.io`, `docker-compose`
- `nfs-common` (för coordinator mounts)
- `tailscale` (via install script)
- `dagu` binary (via GitHub release)

### System Configuration
- Timezone: Europe/Stockholm
- Locale: en_US.UTF-8
- Keyboard: Swedish (se)
- User: marcus (sudo without password)
- SSH: Key-based only, password auth disabled

### Directories
- `/opt/dagu/dags` - DAG definitions (NFS mount från coordinator)
- `/opt/dagu/logs` - Execution logs (NFS mount från coordinator)
- `/opt/dagu/data` - Worker data
- `/etc/dagu/config.yaml` - Dagu configuration
- `/var/lib/dagu-worker/status` - Deployment status

### Services
- `tailscaled.service` - Tailscale daemon
- `docker.service` - Docker engine
- `dagu.service` - Dagu worker

### Network
- Tailscale mesh networking (tag:dagu-worker)
- DHCP för public interface
- Optional: Private networking via cloud provider

## Dagu Worker Configuration

Worker körs i **headless mode** och ansluter till coordinator via gRPC:

```yaml
# /etc/dagu/config.yaml
host: 0.0.0.0
port: 8080
headless: true

distributed:
  mode: worker
  coordinatorHost: dagu-coordinator
  coordinatorPort: 50051

worker:
  id: WORKER_NAME_PLACEHOLDER
  maxActiveRuns: 100
  labels:
    region: eu-north-1
    type: general
    cloud: hetzner

dags: /opt/dagu/dags
logs: /opt/dagu/logs
data: /opt/dagu/data
```

## Troubleshooting

### Worker dyker inte upp i Tailscale

**Symptom**: Worker syns inte i `tailscale status` efter 5-10 minuter

**Lösning**:
1. Kontrollera via cloud provider console att VM:en är running
2. Kolla cloud-init logs via console/serial port
3. SSH med public IP (temporary):
   ```bash
   ssh root@<public-ip>
   sudo journalctl -u cloud-init -f
   ```

### Tailscale auth key expired

**Symptom**: `tailscale up` failar med "invalid auth key"

**Lösning**:
1. Skapa ny ephemeral key i [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Uppdatera secret i Bitwarden
3. Re-deploy worker

### Dagu inte running

**Symptom**: `systemctl status dagu` visar "failed"

**Lösning**:
```bash
# Kolla logs
sudo journalctl -u dagu -n 50

# Verifiera binary
which dagu
dagu version

# Testa manuellt
sudo -u marcus dagu start-all

# Restart service
sudo systemctl restart dagu
```

### NFS mount saknas

**Symptom**: `/opt/dagu/dags` är tom

**Lösning**:
Detta är förväntat! NFS mounts konfigureras av Ansible, inte autoinstall.

```bash
# Kör Ansible provisioning
cd ansible/ansible-playbooks
ansible-playbook dagu-workers.yml --limit <worker-name>
```

## Säkerhet

### Secrets Management

- ✅ Alla secrets i Bitwarden Secrets Manager
- ✅ Secrets injiceras av Terraform vid deployment
- ✅ Inga secrets committas i version control
- ✅ Inga secrets lagras i cloud provider metadata
- ✅ worker.yaml template innehåller endast placeholders

### SSH Access

- ✅ Primary: Tailscale SSH (MagicDNS + ACLs)
- ✅ Secondary: SSH keys (emergency access)
- ✅ Password authentication disabled
- ✅ Only key-based authentication

### Network Security

- ✅ Tailscale mesh networking (end-to-end encrypted)
- ✅ Workers endast tillgängliga via Tailscale
- ✅ Optional: Cloud firewall för public interface
- ✅ Optional: Private networking för intern trafik

## Struktur

```
autoinstall/
├── README.md              # Denna fil
└── worker.yaml            # Autoinstall template (med placeholders)

terraform/
└── dagu-worker/
    ├── main.tf            # Terraform main configuration
    ├── variables.tf       # Input variables
    ├── outputs.tf         # Outputs
    ├── versions.tf        # Provider versions
    ├── deploy-worker.sh   # Deployment script
    ├── terraform.tfvars.example
    └── README.md          # Terraform documentation

ansible/
└── ansible-playbooks/
    ├── dagu-workers.yml   # Ansible provisioning playbook
    └── group_vars/
        └── dagu_workers.yml
```

## Relaterade Projekt

- **Terraform Configuration**: `../terraform/dagu-worker/`
  - Deployment automation med Bitwarden integration
  - Support för Hetzner, AWS, GCP

- **Ansible Playbooks**: `../ansible/ansible-playbooks/`
  - `dagu-workers.yml` - Post-deployment provisioning
  - `group_vars/dagu_workers.yml` - Worker configuration
  - `group_vars/all/vault.yml` - Bitwarden secrets

- **Ansible Collections**:
  - `webmarcus.dagu` - Dagu installation och konfiguration
  - `webmarcus.nfs` - NFS client konfiguration
  - `freeformz.ansible` - Tailscale dynamic inventory

## Support

### Logs

```bash
# Cloud-init (autoinstall)
sudo journalctl -u cloud-init -f

# Tailscale
sudo journalctl -u tailscaled -f

# Docker
sudo journalctl -u docker -f

# Dagu
sudo journalctl -u dagu -f
```

### Documentation

- **Post-Install README**: `/root/POST-INSTALL-README.md` (på servern)
- **Terraform README**: `../terraform/dagu-worker/README.md`
- **Ansible README**: `../ansible/ansible-playbooks/README.md`

### Configuration Files

- **Dagu Config**: `/etc/dagu/config.yaml`
- **Dagu Service**: `/etc/systemd/system/dagu.service`
- **Deployment Status**: `/var/lib/dagu-worker/status`

## License

MIT

---

**Version**: 2.0.0 (Terraform Template Approach)
**Last Updated**: 2025-11-12
**Deployment Method**: Terraform + Autoinstall
**Secrets Management**: Bitwarden Secrets Manager
**Provisioning**: Cloud-Init + Ansible
