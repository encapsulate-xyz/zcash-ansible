# zcash-ansible

The Ansible playbooks Encapsulate uses to run Zcash Crosslink nodes and finalizers on the Crosslink v14 Feature Net (`crosslink_featurenet_20260929`), and a service that converts a finalizer's commission into bonds.

Our finalizer, Encapsulate, runs on these playbooks. You can see it on Holdfast, our explorer: https://holdfast.crosslink.encapsulate.xyz

## What it sets up

`setup_node.yml` (role `node`), on every host with `type: validator`:

- **Firewall (ufw):** denies incoming traffic by default, opens the P2P port to everyone and the metrics port to the monitoring server only.
- **User and directories:** a system user with no login shell, and `bin`, `config`, `data`, `lib` and `log` under `/opt/<service>`.
- **Config:** `config.toml` from `roles/node/templates/testnet/config.toml.j2`.
- **Binary:** `zebrad` from the [ShieldedLabs/crosslink_monolith releases](https://github.com/ShieldedLabs/crosslink_monolith/releases), at the version in `group_vars/testnet.yml`.
- **glibc:** the release binary needs glibc 2.43. On a host with an older one, the playbook downloads Ubuntu's `libc6` package (checksum pinned), unpacks it into `lib/` and starts `zebrad` through the loader that ships with it. The host's own libc is left alone.
- **Seed:** with `fetch_vault_secrets=true`, restores `secret.seed` from HashiCorp Vault before the first start.
- **Service:** a systemd unit with a CPU quota, a 64 GB memory limit and a log file.

`setup_autoconvert.yml` (role `autoconvert`), only on hosts with `autoconvert: true`:

- A systemd service, `<service>-autoconvert`. Whenever the finalizer's reward bank holds enough, it converts the commission into bonds on the same finalizer, 1 cTAZ at a time, and waits for each conversion to be mined before sending the next.
- It runs as its own user, with its own copy of the seed, so nobody needs sudo on the server for it.
- The node needs the seed to sign each conversion, so the script talks only to the node's RPC on localhost, and systemd blocks the service from reaching anything else. The seed is only ever piped, never put on a command line or printed. The node's replies repeat the seed, so the script logs only a result code.
- It refuses to start if the node sets `explicit_bft_key_seed`, since the finalizer key then doesn't come from `secret.seed`.
- `-e autoconvert_dry_run=true` checks everything and sends nothing.

## Layout

```
.
├── ansible.cfg                 # SSH user and key, inventory path
├── collections/requirements.yml
├── group_vars/
│   ├── all.yml                 # ports, monitoring server, SSH port
│   ├── testnet.yml             # network, release version, glibc, auto-convert amounts
│   └── vault.yml               # Ansible Vault: the HashiCorp Vault URL
├── inventory/testnet.yml       # hosts and per-host settings
├── roles/
│   ├── node/                   # zebrad, config, glibc, systemd
│   ├── autoconvert/            # commission into bonds
│   └── common/                 # systemd handlers
├── tasks/                      # ufw, users and directories, Vault secrets
├── setup_node.yml
└── setup_autoconvert.yml
```

## Requirements

- Ansible, with `community.general` for ufw (it comes with the `ansible` package) and `community.hashi_vault`:
  `ansible-galaxy collection install -r collections/requirements.yml`
- Ubuntu hosts (the playbooks use `apt`, `ufw` and `dpkg-deb`).
- SSH as `ansible` with the key `~/.ssh/ansible` on port 8192 (`ansible.cfg`, `group_vars/all.yml`). Change these to match your servers.
- `curl` and `jq` on a host that runs the auto-convert.

## Configuration

Per host, in `inventory/testnet.yml`:

| Variable | Meaning |
|---|---|
| `type` | `validator` |
| `cpu_quota` | systemd `CPUQuota`, e.g. `"800%"` for eight cores |
| `internal_miner_threads` | threads for the built-in miner |
| `parallel_cpu_threads` | threads for block verification |
| `rpc_tailnet` | `true` binds the RPC on every interface so other machines can reach it over Tailscale. ufw's default deny still keeps it closed on the public interface. Otherwise the RPC listens on 127.0.0.1 only |
| `autoconvert` | `true` to run the auto-convert on this host |
| `autoconvert_finalizer` | the finalizer's public key, 64 hex characters, as `get_tfl_roster_zec` shows it |

Shared settings:

- `group_vars/all.yml`: ports (P2P 8788, metrics 8789, RPC 8790, and 12002 for Crosslink's own peer network), the monitoring server allowed to scrape metrics, and the SSH port.
- `group_vars/testnet.yml`: the network name, the release version, the glibc package, and the auto-convert amount, fee margin and timings.
- `roles/node/templates/testnet/config.toml.j2`: the node config. It mines to Encapsulate's address (`miner_address`), so change that to yours. The BFT and P2P peers are Shielded Labs' seed nodes for the Feature Net.

## Secrets

- `secret.seed` is the node's wallet and finalizer key. The playbooks read it from HashiCorp Vault (KV v2), mount `<env>`, path `<project>/<org>/<type>/secret.seed`, stored base64-encoded. With our inventory that's `testnet` and `zcash/encapsulate/validator/secret.seed`. The token comes from the `VAULT_TOKEN` environment variable.
- `group_vars/vault.yml` holds the Vault URL, `vault.default.hcp.vault.url`, encrypted with Ansible Vault. To use the playbooks with your own Vault, replace it with your own file, encrypted with your own password.
- Run each seed on one node only. Two finalizers with the same key vote twice.
- Without `fetch_vault_secrets=true`, nothing is fetched. A new node then creates its own seed on first start.

## Usage

```bash
ansible-galaxy collection install -r collections/requirements.yml

# a node, restoring its seed from Vault
VAULT_TOKEN=... ansible-playbook setup_node.yml --limit <host> -e fetch_vault_secrets=true --ask-vault-pass

# a config or version change on a node that already has its seed
ansible-playbook setup_node.yml --limit <host> --ask-vault-pass

# the auto-convert on the finalizer
VAULT_TOKEN=... ansible-playbook setup_autoconvert.yml --limit <host> -e fetch_vault_secrets=true --ask-vault-pass
```

`setup_node.yml` shows the host, environment and node type, then pauses for 40 s and again for 20 s, so you can stop it with Ctrl-C if it's the wrong host. A changed config, binary or library restarts the node.

Logs: `/opt/zcash/log/zcash.log` for the node and `/opt/zcash-autoconvert/log/zcash-autoconvert.log` for the auto-convert.

## Monitoring

The Grafana dashboard and Prometheus alert rules for these nodes are in [encapsulate-xyz/zcash-crosslink-monitoring](https://github.com/encapsulate-xyz/zcash-crosslink-monitoring).
