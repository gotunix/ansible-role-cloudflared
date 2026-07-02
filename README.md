# Ansible Role — Cloudflared (Multiple Tunnels)

Install cloudflared and manage multiple Cloudflare Tunnels, each with its own systemd service. Supports two authentication methods.

## Authentication Methods

### Method 1: Token (Recommended — no CLI needed)

Create the tunnel and configure ingress in the **Cloudflare Dashboard**:

1. Go to [Cloudflare Zero Trust → Networks → Tunnels](https://one.dash.cloudflare.com/)
2. Click **Create a tunnel** → name it → click **Save tunnel**
3. Copy the **tunnel token** (long base64 string)
4. Configure your public hostnames and ingress rules **in the dashboard**
5. Store the token in your Ansible vault

```yaml
# group_vars/all/vault.yaml (encrypted)
vault_web_token: "eyJhIjoiNjY..."

# group_vars/all/cloudflared.yaml
cloudflared_tunnels:
  - name: web
    token: "{{ vault_web_token }}"
```

> [!NOTE]
> With tokens, ingress rules are managed in the Cloudflare dashboard, not
> in Ansible. This is the simplest approach — no credentials JSON needed.

### Method 2: Credentials JSON (from CLI)

Create tunnels via the `cloudflared` CLI:

```bash
cloudflared tunnel login
cloudflared tunnel create web-tunnel
# Saves credentials to ~/.cloudflared/<tunnel-id>.json
```

Store the credentials JSON in vault:

```yaml
# group_vars/all/vault.yaml (encrypted)
vault_web_creds: |
  {"AccountTag":"...","TunnelSecret":"...","TunnelID":"..."}

# group_vars/all/cloudflared.yaml
cloudflared_tunnels:
  - name: web
    tunnel_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    credentials: "{{ vault_web_creds }}"
    ingress:
      - hostname: app.example.com
        service: http://localhost:8080
      - service: http_status:404
```

> [!NOTE]
> With credentials, ingress rules are managed in Ansible via the `ingress`
> list. You also need to configure DNS (CNAME) in Cloudflare separately.

## Quick Start

```bash
ansible-playbook site.yaml -i inventory.ini --ask-vault-pass
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `cloudflared_state` | `present` | `present` or `absent` |
| `cloudflared_config_dir` | `/etc/cloudflared` | Config directory |
| `cloudflared_user` | `cloudflared` | Service user |
| `cloudflared_group` | `cloudflared` | Service group |
| `cloudflared_tunnels` | `[]` | List of tunnel dicts |

### Tunnel Format

```yaml
cloudflared_tunnels:
  # Token-based (dashboard manages ingress)
  - name: web
    token: "{{ vault_web_token }}"

  # Credentials-based (Ansible manages ingress)
  - name: api
    tunnel_id: "xxxx-xxxx"
    credentials: "{{ vault_api_creds }}"
    ingress:
      - hostname: api.example.com
        service: http://localhost:3000
        originRequest:                  # optional per-route settings
          noTLSVerify: true
      - service: http_status:404        # catch-all (required)
```

## Multiple Tunnels

Each tunnel gets its own systemd service:

```yaml
cloudflared_tunnels:
  - name: web
    token: "{{ vault_web_token }}"

  - name: internal
    token: "{{ vault_internal_token }}"
```

```bash
systemctl status cloudflared-web
systemctl status cloudflared-internal
```

## Managing Tunnels

```bash
# Check status
systemctl status cloudflared-web

# View logs
journalctl -u cloudflared-web -f

# Restart a tunnel
systemctl restart cloudflared-web

# List all tunnel services
systemctl list-units 'cloudflared-*'
```

## Tags

| Tag | Controls |
|---|---|
| `cloudflared` | Entire role (use with `--skip-tags`) |
| `install` | Package installation only |
| `configure` | Tunnel config and services |
| `uninstall` | Removal |

## File Structure

```
cloudflared/
├── site.yaml
└── roles/
    └── cloudflared/
        ├── defaults/main.yaml
        ├── handlers/main.yaml
        ├── tasks/
        │   ├── main.yaml
        │   ├── install.yaml
        │   ├── configure.yaml
        │   └── uninstall.yaml
        └── templates/
            ├── tunnel-config.yaml.j2
            └── cloudflared.service.j2
```

## Security Notes

- Credentials/tokens are `no_log` during deployment
- Credentials files are `0600`, owned by the service user
- Config directory is `0750`
- Systemd hardening: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`
- **Always store tokens and credentials in Ansible vault**

## Requirements

- Ansible ≥ 2.9
- Target hosts running Debian/Ubuntu
- Cloudflare account with Zero Trust / Tunnel access
