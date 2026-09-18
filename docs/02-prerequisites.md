# 2. Prerequisites

Everything you need before chapter 3. Each requirement has a checkbox, and
the software ones have a verification command with the expected result. Do
not skip the host sizing table: an undersized host produces intermittent SRA
timeouts that look like network problems.

## Host sizing

Per-component minimums from the Akeyless SRA requirements page:

| Component | Minimum vCPU | Minimum memory | Basis |
|---|---|---|---|
| `akeyless-gateway` | 1 | 2 GB | Gateway requirement |
| `akeyless-sra-ssh` | 1 | 2 GiB | SRA component requirement |
| `akeyless-sra-web` | 1 | 2 GiB | SRA component requirement |
| `akeyless-cache` | shared | shared | Redis Alpine image, negligible |
| **Total for this deployment** | **3** | **6 GiB** | Sum of the above |

Practical additions beyond the documented minimums: give the host at least
4 vCPU and 8 GiB RAM if it also runs the metrics profile or serves more than
a handful of concurrent sessions, and keep at least 10 GB free disk for the
container images and logs.

## Software requirements

| Software | Minimum version | Check with | Expected result |
|---|---|---|---|
| Docker Engine | 20.10 | `docker version --format '{{.Server.Version}}'` | A version number of 20.10 or higher |
| Docker Compose | v2, or 1.29 | `docker compose version` | A version string, no error |
| Akeyless CLI | Current release | `akeyless --version` | A version string |
| curl | Any recent | `curl --version` | A version string |

### Verify

```bash
docker version --format '{{.Server.Version}}'
docker compose version
akeyless --version
```

**Expected output:** three version strings. If `docker` reports a permission
error, add your user to the `docker` group with
`sudo usermod -aG docker "$USER"` and log in again, or prefix the Docker
commands in this runbook with `sudo`.

The Docker host must be x86-64. The Compose file pins every Akeyless image to
`linux/amd64`.

### Running the stack on Podman instead

The chapters say `docker`, and Podman 4.9 or newer runs the same Compose kit
unchanged through the `podman-docker` compatibility shim. This path was
validated end to end in September 2026. Install `podman`, the
`podman-docker` package, and Docker Compose v2 as a standalone binary; Podman
picks up the standalone Compose as its external provider, so
`docker compose ...` keeps working through the shim. Check with
`podman --version` and `docker compose version`.

Three shim differences change how you type the commands:

1. Prefix every `docker` command with `sudo`. The shim maps `docker` to
   `podman` against the calling user's container store, and the stack runs
   rootful, so without `sudo` every command sees an empty stack and reports
   the containers as missing.
2. Put flags before container names. The shim hands the arguments to
   `podman`, which rejects a flag after the container name:
   `docker logs akeyless-gateway --tail 50` fails with
   `no container with name or ID "--tail" found`, while
   `docker logs --tail 50 akeyless-gateway` works. Every command in this
   runbook uses the flag-first form, which Docker Engine accepts as well.
3. Do not run Docker Engine and Podman containers on the same host at the
   same time. Both engines write host firewall rules, and either engine can
   break the other's published ports. If Docker is installed as a snap, stop
   it with `sudo snap stop docker` before starting this stack under Podman.

Podman also does not restart `restart: always` containers after a reboot by
itself. Enable the service that does:

```bash
sudo systemctl enable --now podman-restart.service
```

## Network requirements

### Outbound, from the Docker host

| Destination | Port | Protocol | Purpose |
|---|---|---|---|
| Akeyless SaaS endpoint for your account region | 443 | TCP | All gateway communication with the control plane |
| Container registry | 443 | TCP | Image pulls on first start and upgrades |
| DNS resolvers | 53 | TCP and UDP | Name resolution |
| NTP servers | 123 | UDP | Clock sync; the gateway validates certificates against the clock |

### Inbound, to the Docker host, from your management network only

| Host port | Protocol | Needed by | Chapter |
|---|---|---|---|
| 8000 | TCP | Users calling the API, opening the local console and the SRA portal | 6, 8 |
| 8080 | TCP | Health monitoring of the host | 6 |
| 2222 | TCP | Users connecting with `akeyless connect` or plain `ssh` | 8 |
| 8888 | TCP | Users opening the web bastion in a browser | 8 |
| 8889, 9090, 3000 | TCP | Metrics and dashboards, optional | 9 |

Do not expose port 8000 to the public internet. The gateway API and the SRA
portal ride on it, and in this plain-HTTP deployment nothing authenticates the
transport layer. Bind these ports to the management interface or restrict them
with the host firewall.

Confirm each inbound port is free before chapter 6:

```bash
for p in 8000 8080 2222 8888 8889; do
  timeout 1 bash -c "</dev/tcp/127.0.0.1/$p" 2>/dev/null && echo "$p IN USE" || echo "$p free"
done
```

**Expected output:** `free` on every line. A container or a Kubernetes
hostPort rule on the host can intercept a port even when nothing appears to
listen on it; chapter 11 has the diagnosis path if a port that shows free
still refuses connections later.

### On target hosts

| Requirement | Detail |
|---|---|
| Outbound | None. Targets never call Akeyless. |
| Inbound | Standard SSH on 22, reachable from the Docker host. |
| sshd | `TrustedUserCAKeys` support, present in OpenSSH 6.8 and newer. |

## Akeyless account requirements

- [ ] An Akeyless account where you can sign in to the console as an admin.
- [ ] Permission to create auth methods, roles, DFC keys, and SSH Certificate
      Issuers. The account admin has all of these by default.
- [ ] The account's cluster name, visible in the Akeyless console under
      **Configuration**. You set it as `CLUSTER_NAME` in chapter 5.

## Target host requirements

- [ ] At least one Linux host you can reach with SSH as a sudo-capable user.
- [ ] That user name becomes the `--allowed-users` value in chapter 4 and the
      `Default SSH Username` on the issuer. Common values: `ubuntu`, `ec2-user`,
      `admin`, `root`.

## Client requirements, for each end user

- [ ] The Akeyless CLI installed.
- [ ] Network reachability to the Docker host on ports 8000 and 2222 for CLI
      sessions, and 8888 or 8000 for browser sessions.
- [ ] A browser for the portal and web client, if you use them.

## Information to gather before starting

Collect these once; chapters 3 to 8 consume them:

| Parameter | Description | Example |
|---|---|---|
| Cluster name | Your Akeyless cluster name from the console | `cl-xxxxxxxxx` or a custom name |
| Admin API key Access ID | The Access ID of the admin API key from chapter 3, used in the permissions JSON | `p-xxxxxxxxxxxx` |
| Gateway Access ID | The Access ID of the API key you create in chapter 3, starts with `p-` | `p-xxxxxxxxxxxx` |
| Gateway Access Key | The secret generated with that API key, shown once | a long random string |
| Docker host address | DNS name or IP your users reach | `sra.example.internal` |
| Target SSH username | The sudo user on target hosts | `ubuntu` |
| Target host address | DNS name or IP of the first target | `10.0.1.23` |
| Redis password | A strong password you generate now, used in `cache.env` | any 24+ character string |

Generate the Redis password now if you have not:

```bash
openssl rand -base64 24
```

**Expected output:** one 32-character random string. Store it with the other
values from this table.

## Next step

[Chapter 3: Akeyless Account Setup](03-akeyless-account-setup.md) creates the
gateway identity.
