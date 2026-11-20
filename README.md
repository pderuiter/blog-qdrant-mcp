# blog-qdrant-mcp

Companion code for the blog post
[Building Long-Term AI Memory with Qdrant and Claude's MCP Server](https://webuildyourcloud.com/building-long-term-ai-memory-with-qdrant-and-claudes-mcp-server/).

It builds the setup from the post. It creates an Ubuntu 24.04 VM and installs Docker CE. Then it runs Qdrant with the
post's `docker-compose.yml`, which secures it with an API key. Finally, on your own machine, it registers
`mcp-server-qdrant` with Claude Code through `claude mcp add`, so Claude gets the `qdrant-store` and `qdrant-find`
tools.

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | One Proxmox VE VM from the Ubuntu 24.04 cloud image (bpg/proxmox provider). Outputs an Ansible inventory. |
| `ansible/roles/docker_engine` | Docker CE, containerd and the Compose v2 plugin from download.docker.com. |
| `ansible/roles/qdrant` | The post's Compose file: a pinned Qdrant release, the `qdrant_data` volume, the API key, no published ports, and Traefik labels that route `qdrant_hostname` on `websecure` to port 6333. It checks that the Traefik network exists, waits until `/healthz` answers `healthz check passed` on the container's address, then checks that `/collections` returns 401 without the key. |
| `ansible/roles/mcp_client` | Runs on the `workstation` group (your machine, `ansible_connection: local`). It runs `claude mcp add qdrant-memory --transport stdio -e ... -- uvx mcp-server-qdrant` with the post's `QDRANT_URL`, `QDRANT_API_KEY`, `COLLECTION_NAME` and `EMBEDDING_MODEL`. The entry is only rewritten when one of those values changes. |
| `ansible/site.yml` | Qdrant on the `qdrant` group, then the MCP registration on `workstation`. |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM).
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`.
- SSH access with sudo to the machine. The Terraform cloud-init user is `ubuntu`.
- On the workstation: the Claude Code CLI (`claude`) and `uv` (for `uvx`).
- Traefik on the same Docker host, set up as in the post
  [Configuring Traefik with Let's Encrypt on Docker](https://webuildyourcloud.com/configuring-traefik-with-lets-encrypt-on-docker/)
  (companion repo `blog-traefik-letsencrypt-docker`): the `traefik-network` network, the `websecure` entry point and
  the `transip` certificate resolver. Qdrant itself runs without TLS and publishes no ports, so Traefik is the only way in.

## Quick start

```bash
# 1. Machine (optional; any Ubuntu 24.04 host works)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IP, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configuration
cd ../ansible
ansible-galaxy collection install -r requirements.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml   # vault_qdrant_api_key (openssl rand -hex 32)
$EDITOR group_vars/all/vars.yml                  # mcp_client_qdrant_url: how your workstation reaches Qdrant
ansible-playbook site.yml --ask-vault-pass
```

The Terraform inventory only contains the `qdrant` group. To register the MCP server as well, add the `workstation`
group from `inventory.example.yml` to `inventory.yml`. Without it, the second play has no hosts and is skipped.

Check the registration with `claude mcp get qdrant-memory`, then ask Claude to remember something.

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `qdrant_api_key` | `{{ vault_qdrant_api_key }}` | `QDRANT__SERVICE__API_KEY` |
| `qdrant_image` | `qdrant/qdrant:v1.20.2` | Pinned release, as in the post |
| `qdrant_hostname` | `qdrant.example.com` | Host name Traefik routes to Qdrant (the post's `qdrant.bsdserver.nl`) |
| `qdrant_traefik_network` | `traefik-network` | External Docker network shared with Traefik; must exist |
| `qdrant_traefik_entrypoint` / `qdrant_traefik_certresolver` | `websecure` / `transip` | Traefik entry point and ACME resolver from the Traefik post |
| `qdrant_dir` | `/opt/qdrant` | Project directory on the host |
| `mcp_client_name` | `qdrant-memory` | MCP server name in Claude Code |
| `mcp_client_scope` | `user` | `local`, `user` (`~/.claude.json`) or `project` (`.mcp.json` in `mcp_client_project_dir`) |
| `mcp_client_qdrant_url` | `https://{{ qdrant_hostname }}:443` | `QDRANT_URL` (through Traefik) |
| `mcp_client_collection_name` | `architectural-decisions` | `COLLECTION_NAME` |
| `mcp_client_embedding_model` | `sentence-transformers/all-MiniLM-L6-v2` | `EMBEDDING_MODEL`; must be a model FastEmbed supports |

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (Docker 29) and a local workstation with Claude Code
2.1, using a throw-away `HOME` (`mcp_client_home`). The second run reported `changed=0` on both. Results:
- The written `~/.claude.json` entry matched the JSON in the post.
- `mcp-server-qdrant` (via `uvx`), pointed at the deployed Qdrant, listed `qdrant-store` and `qdrant-find`.
- Two decisions were stored, and both queries from the post returned the right one first.
- The collection was created with 384-dimension Cosine vectors.

ansible-lint (production profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not been
applied against a real Proxmox cluster.

## License

MIT
