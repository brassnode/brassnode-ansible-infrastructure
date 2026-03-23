# brassnode-ansible-infrastructure

Ansible playbooks for provisioning and configuring WireGuard VPN servers on Hetzner Cloud.

## Overview

This repo automates:

- Provisioning a Hetzner Cloud server
- Attaching floating IPs
- Installing and configuring WireGuard VPN

## Requirements

- Ansible
- Python 3
- `hetzner.hcloud` Ansible collection
- A Hetzner Cloud account and API token
- An SSH key added to your Hetzner project

Install the Hetzner collection:

```bash
ansible-galaxy collection install hetzner.hcloud
```

## Setup

Copy `.env.example` and fill in your values:

```bash
cp .env.example .env
```

Export all variables before running any playbook:

```bash
set -a && source .env && set +a
```

Or export manually:

```bash
export HCLOUD_TOKEN=your_token_here
export HCLOUD_SSH_KEY_NAME=dipm-deploy-ssh-key
export HCLOUD_DATACENTER=nbg1           # fsn1 | nbg1 | hel1
export DIPM_SERVICE_KEY=
export DIPM_SERVER_NAME=vpn-nbg1-1      # convention: vpn-{datacenter}-{n}
export DIPM_PRIVATE_IP=10.1.1.3         # see Networking below

```

## Usage

Run the full provisioning and configuration pipeline:

```bash
ansible-playbook playbooks/site.yml
```

Or run individual playbooks:

```bash
# Provision server only
ansible-playbook playbooks/01_server_provisioning.yml

# Install WireGuard only (server must already exist in inventory)
ansible-playbook playbooks/02_wireguard_installation.yml
```

## Networking

Servers are attached to a private Hetzner network. Assign private IPs according to datacenter:

| Datacenter | Private IP Range |
| ---------- | ---------------- |
| fsn1       | 10.0.1.x         |
| nbg1       | 10.1.1.x         |
| hel1       | 10.2.1.x         |

`VPN_IPS_PER_SERVER` controls how many floating IPs are created and attached to the server. Each IP is named `{server_name}-ip-{n}`.

## WireGuard Peers

Peers are the client devices you want to connect to the VPN. Each peer needs a WireGuard public key.

**Windows:** Install [WireGuard for Windows](https://www.wireguard.com/install/), click **Add Tunnel → Add empty tunnel**, and copy the generated public key.

**Linux/Mac:**

```bash
wg genkey | tee privatekey | wg pubkey > publickey
cat publickey
```

**Android/iOS:** Install the WireGuard app, create a new tunnel, and copy the public key shown.

Add peers to `playbooks/02_wireguard_installation.yml`:

```yaml
wireguard_peers:
  - name: laptop
    public_key: "your-public-key-here"
    allowed_ips:
      - "10.0.0.2/32"
    persistent_keepalive: 25
  - name: phone
    public_key: "your-phone-public-key"
    allowed_ips:
      - "10.0.0.3/32"
    persistent_keepalive: 25
```

Each peer gets a unique IP in the `10.0.0.0/24` range. The server itself is `10.0.0.1`.

## Project Structure

```
.
├── inventory/
│   └── group_vars/
│       └── all.yml
├── playbooks/
│   ├── 01_server_provisioning.yml
│   ├── 02_wireguard_installation.yml
│   └── site.yml
├── roles/
│   ├── server_provisioning/
│   │   ├── defaults/
│   │   ├── meta/
│   │   └── tasks/
│   │       ├── main.yml
│   │       ├── create_server.yml
│   │       └── attach_ips.yml
│   └── wireguard_installation/
│       ├── defaults/
│       ├── handlers/
│       ├── meta/
│       ├── tasks/
│       ├── templates/
│       └── vars/
├── vars/
│   ├── common.yml
│   ├── fsn1.yml
│   ├── nbg1.yml
│   └── hel1.yml
├── .env
├── .env.example
├── .gitignore
└── ansible.cfg
```

## Security

- Never commit `.env` or any file containing `HCLOUD_TOKEN` — it is listed in `.gitignore`
- SSH private key path is set via `ANSIBLE_SSH_PRIVATE_KEY_FILE` and never hardcoded
