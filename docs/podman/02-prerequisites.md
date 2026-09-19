# 2. Prerequisites, Podman Stream

Everything you need before chapter 3. Each requirement has a checkbox, and
the software ones have a verification command with the expected result. Do
not skip the host sizing table: an undersized host produces intermittent SRA
timeouts that look like network problems.

This chapter assumes three Podman-specific habits throughout, so they come
first.

## How this stream runs Podman

**Rootful, with `sudo`.** The stack runs in the root Podman store. A
`podman` command without `sudo` looks at the calling user's own store, sees
nothing there, and reports the containers as missing or the stack as empty.
That empty-looking output is the number one source of confusion on Podman
hosts, and chapter 11 lists it as its own symptom. Every command in this
stream carries `sudo`; keep the habit even when a command appears to work
without it, because a user store that quietly diverges from the root store is
worse than a clean failure.

**Flags before container names.** Podman rejects a flag placed after the
container name. `sudo podman logs akeyless-gateway --tail 50` fails with
`no container with name or ID "--tail" found`, while
`sudo podman logs --tail 50 akeyless-gateway` works. Every command in this
stream uses the flag-first form.

**No Docker Engine running alongside.** Podman and Docker Engine both write
host firewall rules for published ports, and either engine can break the
other's rules. If Docker is installed, stop it before bringing this stack up.
A Docker installed as a snap stops with `sudo snap stop docker`; a packaged
Engine stops with `sudo systemctl disable --now docker`.

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
| Podman | 4.9 | `podman --version` | A version number of 4.9 or higher |
| Docker Compose v2, standalone binary | v2 | `docker-compose version` | A version string, no error |
| Akeyless CLI | Current release | `akeyless --version` | A version string |
| curl | Any recent | `curl --version` | A version string |

Install Podman from your distribution. Install the standalone Docker
Compose v2 binary from its upstream release archive and place it on your
`PATH` as `docker-compose`; Podman finds it there and uses it as its Compose
provider, so `podman compose ...` runs the same Compose v2 implementation
Docker hosts use. The `podman-docker` package, which maps `docker` to
`podman`, is optional; install it only if you want to type `docker`. If you
do, the three habits above still apply unchanged to every `docker` command,
because the shim hands the arguments to `podman`.

### Verify

```bash
podman --version
sudo podman info --format '{{.Host.Arch}}'
docker-compose version
akeyless --version
```

**Expected output:** a Podman version of 4.9 or higher, the host
architecture `amd64`, a Compose version string, and a CLI version string.
The host must be x86-64: the Compose file pins every Akeyless image to
`linux/amd64`.

### Survive host reboots

Podman does not restart containers marked `restart: always` after a reboot
by itself. Enable the service that does:

```bash
sudo systemctl enable --now podman-restart.service
```

**Expected output:** the unit reports that it is enabled, with a symlink
line. Without it, the stack stays down after every reboot.

## Network requirements

### Outbound, from the container host

| Destination | Port | Protocol | Purpose |
|---|---|---|---|
| Akeyless SaaS endpoint for your account region | 443 | TCP | All gateway communication with the control plane |
| Container registry | 443 | TCP | Image pulls on first start and upgrades |
| DNS resolvers | 53 | TCP and UDP | Name resolution |
| NTP servers | 123 | UDP | Clock sync; the gateway validates certificates against the clock |

### Inbound, to the container host, from your management network only

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
| Inbound | Standard SSH on 22, reachable from the container host. |
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
      `Default SSH Username` on the issuer. Common values: `ubuntu`,
      `ec2-user`, `admin`, `root`.

## Client requirements, for each end user

- [ ] The Akeyless CLI installed.
- [ ] Network reachability to the container host on ports 8000 and 2222 for
      CLI sessions, and 8888 or 8000 for browser sessions.
- [ ] A browser for the portal and web client, if you use them.

## Information to gather before starting

Collect these once; chapters 3 to 8 consume them:

| Parameter | Description | Example |
|---|---|---|
| Cluster name | Your Akeyless cluster name from the console | `cl-xxxxxxxxx` or a custom name |
| Admin API key Access ID | The Access ID of the admin API key from chapter 3, used in the permissions JSON | `p-xxxxxxxxxxxx` |
| Gateway Access ID | The Access ID of the API key you create in chapter 3, starts with `p-` | `p-xxxxxxxxxxxx` |
| Gateway Access Key | The secret generated with that API key, shown once | a long random string |
| Container host address | DNS name or IP your users reach | `sra.example.internal` |
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

[Chapter 3: Akeyless Account Setup](../common/03-akeyless-account-setup.md)
creates the gateway identity.
