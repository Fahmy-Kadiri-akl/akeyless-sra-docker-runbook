# 5. Compose Configuration

This repository ships a complete Compose kit in `compose/`, adapted from the
official Akeyless Labs docker-compose repository with two changes: the SSH
bastion CA mount is enabled by default, and a Prometheus scrape config is
provided. You do not clone anything else.

## Copy the environment files

From the repository root:

```bash
cd compose
cp gateway.env.example gateway.env
cp sra.env.example sra.env
cp cache.env.example cache.env
```

Then edit `gateway.env` and `cache.env`. The file `sra.env` needs no changes;
its values match the Compose file.

## gateway.env, variable by variable

| Variable | Set to | Meaning |
|---|---|---|
| `CLUSTER_NAME` | Your cluster name from chapter 2 | Pairs with the Access ID to identify this gateway in the console |
| `UNIFIED_GATEWAY` | `true` | One container serves gateway API, console, portal, and SRA routing |
| `VERSION` | Empty | Pin a gateway version tag to upgrade deliberately, chapter 9 |
| `ENABLE_METRICS` | `false` now | Flip to `true` with the metrics profile in chapter 9 |
| `GATEWAY_ACCESS_ID` | Gateway Access ID from chapter 3 | The identity the gateway authenticates with |
| `GATEWAY_ACCESS_TYPE` | `access_key` | Must match the auth method type created in chapter 3. Other supported values: `password`, `saml`, `ldap`, `k8s`, `azure_ad`, `oidc`, `aws_iam`, `universal_identity`, `jwt`, `gcp`, `cert`, `oci`, `kerberos` |
| `GATEWAY_ACCESS_KEY` | Gateway Access Key from chapter 3 | The secret matching the Access ID |
| `ALLOWED_ACCESS_PERMISSIONS` | The JSON from chapter 3 | Who may administer the gateway console, with which permissions |
| `GATEWAY_AUTHORIZED_ACCESS_ID` | Unset | Optional transport allowlist. When set, every auth method routing through this gateway, SRA users included, must have its Access ID in this comma-separated list. Read chapter 7 before enabling it |
| `ENABLE_TLS_CONFIGURE` | `true` | Lets the gateway manage its own TLS configuration; chapter 10 |
| `GATEWAY_CLUSTER_CACHE` | `enable` | Gateway caching toggle; SRA requires the cache service |
| `USE_CLUSTER_CACHE` | `true` | Use the Redis-backed cache |
| `PREFER_CLUSTER_CACHE_FIRST` | `true` | Read the cache before the SaaS where possible |
| `PROACTIVE_CACHE_ENABLE`, `NEW_PROACTIVE_CACHE_ENABLE`, `PRO_ACTIVE_CACHE_ENABLE` | As shipped | Version-overlapping flags; leave them alone unless support instructs otherwise |
| `REDIS_ADDR` | `akeyless-cache:6379` | Cache address inside the Compose network |
| `REMOTE_ACCESS_WEB_SERVICE_INTERNAL_URL` | `http://akeyless-sra-web:8888` | Where the gateway finds the web bastion |
| `REMOTE_ACCESS_SSH_SERVICE_INTERNAL_URL` | `http://akeyless-sra-ssh:9900` | Where the gateway finds the SSH bastion control channel |

`GATEWAY_ACCESS_TYPE` is mandatory for SRA. A gateway without an access type
starts, but SRA sessions fail to authenticate.

## sra.env, what it holds

You keep the shipped values. Two entries deserve explanation:

| Variable | Shipped value | Meaning |
|---|---|---|
| `REMOTE_ACCESS_TYPE` | `ssh-proxy` | This bastion proxies SSH sessions; the Compose file overrides it to `web` for the web bastion container |
| `REMOTE_ACCESS_SSH_ENDPOINT` | `akeyless-ssh:22` | Where the SSH bastion listens inside the Compose network |

The endpoint value is `akeyless-ssh:22`, with port 22, because that is the
port the bastion listens on inside the Compose network. The host mapping
`2222:22` in the Compose file exists for clients connecting from outside the
host. The upstream Akeyless Labs `sra.env` ships `akeyless-ssh:2222`, which
does not match the container listener; this kit uses port 22 deliberately.

## cache.env

One variable:

| Variable | Set to | Meaning |
|---|---|---|
| `REDIS_PASS` | The password generated in chapter 2 | Required by both the Redis server command and the Akeyless services |

Redis is bound to `127.0.0.1:6379` on the host and runs as user 65534 with
all capabilities dropped. The password is defense in depth, not the only
barrier. One hardening flag is deliberately absent: the Compose file sets no
`no-new-privileges` on this service, because the Redis entrypoint execs the
server process under a different uid and current Docker releases kill it
with `exec operation not permitted` when the flag is present.

## The ssh-config mount

The Compose file mounts:

```yaml
- ./ssh-config/:/var/akeyless/creds/
```

into the SSH bastion. Chapter 4 placed `ca.pub` there. The bastion reads the
mounted folder, not a single file, so never put anything else in
`ssh-config/` unless a chapter tells you to.

### Verify the kit is consistent

From the `compose/` directory:

```bash
docker compose --profile gateway --profile sra config --quiet
```

**Expected output:** nothing. The command validates the Compose file, the
env files, and their references, and prints only when something is wrong. A
missing `cache.env` or `sra.env` produces an error naming the file; the fix
is the `cp` from the top of this chapter.

```bash
ls ssh-config/
```

**Expected output:** `ca.pub` and `README.md`. If `ca.pub` is missing, return
to chapter 4.

## What you have at this point

A complete, validated Compose kit: four environment files, a CA public key
in place, and a Compose definition whose references all resolve.

## Next step

[Chapter 6: Start and Verify](06-start-and-verify.md) brings the stack up.
