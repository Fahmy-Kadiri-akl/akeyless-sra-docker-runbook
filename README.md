# Akeyless Secure Remote Access on Docker Compose: Runbook

This repository is a self-contained, end-to-end runbook for deploying Akeyless
Secure Remote Access (SRA) with the Akeyless Gateway on a single Docker Compose
host. It ships a ready-to-use Compose kit plus eleven chapters that carry you
from an empty Linux host to your first SSH session through the bastion, with
verification at every step. You do not need any other document to complete the
deployment: every prerequisite, command, value, and fix lives in this
repository.

## What problem this solves

SSH access to infrastructure usually means long-lived keys copied onto laptops
and bastion hosts, with no per-session control and no audit trail. Akeyless SRA
replaces that with short-lived SSH certificates. Users authenticate to Akeyless
once, receive a certificate valid for minutes, and connect through the SRA
bastion. Sessions are logged, approvable, and revocable.

| Capability | How this deployment provides it |
|---|---|
| Short-lived SSH credentials | SSH Certificate Issuer with a 5 minute default TTL, chapter 4 |
| Brokered SSH access | SSH bastion container on host port 2222, chapter 6 |
| Browser-based SSH and RDP | Web bastion container on host port 8888, chapter 8 |
| Per-user permissions | Roles and SRA rules in the Akeyless account, chapter 7 |
| Session audit and approval | Gateway logs and approval flows, chapters 7 and 9 |

## Documents

| # | Document | Description |
|---|---|---|
| 1 | [Architecture Overview](docs/01-architecture-overview.md) | What gets deployed, how a session flows, full port inventory |
| 2 | [Prerequisites](docs/02-prerequisites.md) | Host sizing, software versions, network requirements, checklists |
| 3 | [Akeyless Account Setup](docs/03-akeyless-account-setup.md) | Gateway identity: auth method, role, permissions JSON |
| 4 | [SSH Certificate Issuer](docs/04-ssh-certificate-issuer.md) | DFC key, issuer, CA public key distribution to targets |
| 5 | [Compose Configuration](docs/05-compose-configuration.md) | Every environment variable in the shipped kit, explained |
| 6 | [Start and Verify](docs/06-start-and-verify.md) | First start, health checks, URLs |
| 7 | [Access Control](docs/07-access-control.md) | The four permission layers, first end-user role |
| 8 | [First Session](docs/08-first-session.md) | CLI and portal session, web access |
| 9 | [Day-2 Operations](docs/09-day-2-operations.md) | Upgrades, logs, metrics |
| 10 | [Advanced Configuration](docs/10-advanced-configuration.md) | TLS, keepalives, host keys, regional endpoints |
| 11 | [Troubleshooting](docs/11-troubleshooting.md) | Symptom-by-symptom diagnosis and fixes |

## Repository contents

| Path | Purpose |
|---|---|
| `compose/docker-compose.yaml` | Compose file with gateway, SSH bastion, web bastion, Redis cache, optional metrics stack |
| `compose/gateway.env.example` | Gateway environment file template; copy to `gateway.env` |
| `compose/sra.env.example` | SRA bastion environment file template; copy to `sra.env` |
| `compose/cache.env.example` | Redis password template; copy to `cache.env` |
| `compose/ssh-config/` | Drop the issuer CA public key here as `ca.pub` |
| `compose/metrics/prometheus/prometheus.yml` | Prometheus scrape config for the gateway metrics port |
| `docs/` | The eleven chapters listed above |

## Architecture at a glance

```mermaid
flowchart LR
    subgraph users["Users"]
        CLI["Engineer with akeyless CLI"]
        BR["Browser user"]
    end

    subgraph host["Docker host: single Compose project"]
        GW["akeyless-gateway<br/>API 8000, health 8080, metrics 8889"]
        SSHB["akeyless-sra-ssh<br/>host port 2222"]
        WEBB["akeyless-sra-web<br/>host port 8888"]
        CACHE["akeyless-cache<br/>Redis, loopback only"]
    end

    subgraph saas["Akeyless SaaS"]
        CONSOLE["Console and control plane"]
    end

    subgraph targets["Target hosts"]
        SRV["SSH server trusting the CA"]
    end

    CLI -->|"SSH via bastion, port 2222"| SSHB
    BR -->|"web client and portal, port 8000 or 8888"| WEBB
    CLI -->|"auth and API"| GW
    GW <-->|"HTTPS outbound 443"| CONSOLE
    GW --- CACHE
    GW --- SSHB
    GW --- WEBB
    SSHB -->|"short-lived certificate"| SRV
```

The gateway needs only outbound HTTPS to the Akeyless SaaS. Target hosts need
no Akeyless software: they trust one CA public key in `sshd_config`.

## Quick start

Experienced with Docker and Akeyless? The short path, about 45 minutes
end to end:

1. Work through [Prerequisites](docs/02-prerequisites.md).
2. Create the gateway identity in the Akeyless account: [chapter 3](docs/03-akeyless-account-setup.md).
3. Create the SSH Certificate Issuer and distribute the CA key: [chapter 4](docs/04-ssh-certificate-issuer.md).
4. Copy and fill the environment files in `compose/`: [chapter 5](docs/05-compose-configuration.md).
5. Start the stack and verify health: [chapter 6](docs/06-start-and-verify.md).
6. Create the first end-user role: [chapter 7](docs/07-access-control.md).
7. Open the first session: [chapter 8](docs/08-first-session.md).

## Compatibility

| Component | Requirement | Notes |
|---|---|---|
| Docker Engine | 20.10 or newer | Any Linux distribution that runs Docker |
| Docker Compose | v2.x, or 1.29 or newer | v2 recommended; commands in this runbook use `docker compose` |
| Akeyless CLI | Current release | Install before chapter 3 |
| Host architecture | x86-64 | Images are pinned to `linux/amd64` |
| Akeyless account | Admin access | You create auth methods, roles, and an SSH Certificate Issuer |

Verified against the Akeyless documentation and the upstream
akeylesslabs/docker-compose repository in September 2026.

## Additional resources

- [Akeyless SRA on Docker documentation](https://docs.akeyless.io/docs/remote-access-docker)
- [Gateway deployment with Docker Compose](https://docs.akeyless.io/docs/gateway-deploy-docker-compose)
- [SRA requirements](https://docs.akeyless.io/docs/sra-requirements)
- [Connecting your first SRA resource](https://docs.akeyless.io/docs/connecting-your-first-sra-resource)
- [Akeyless Labs docker-compose repository](https://github.com/akeylesslabs/docker-compose)

## License

[MIT](LICENSE). The Compose kit in `compose/` is adapted from the upstream
Akeyless Labs repository linked above.
