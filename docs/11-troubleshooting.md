# 11. Troubleshooting

Each section is one symptom, with the diagnosis path and the fix. Work top to
bottom within a symptom; the causes are ordered by how often they occur.

## Gateway container never becomes healthy

**Diagnosis:**

```bash
docker logs akeyless-gateway --tail 50
```

Look at the last error lines before the health check failures.

**Fix, by what the log says:**

| Log message | Cause | Fix |
|---|---|---|
| Authentication failure against Akeyless | Wrong `GATEWAY_ACCESS_ID` or `GATEWAY_ACCESS_KEY` | Re-verify with the `akeyless auth` command from chapter 3, then correct `gateway.env` |
| Cache or Redis connection refused | `REDIS_PASS` mismatch, or cache container down | Confirm `cache.env` matches, `docker ps` shows `akeyless-cache` Up, then recreate |
| Port already in use | Another service holds 8000, 8080, or 8889 on the host | Stop the conflicting service or change the host side of the mapping |

## Gateway runs but never appears in the console

**Diagnosis:** four containers Up, health OK, gateway absent from the
console's gateway list.

**Fix:** `CLUSTER_NAME` and `GATEWAY_ACCESS_ID` form the identity pair; a
wrong cluster name registers the gateway against nothing. Check the cluster
name with `akeyless describe-account-details` as admin and compare it to
`gateway.env`.

## Bastion stays in Created or restarts repeatedly

**Diagnosis:**

```bash
docker ps -a --filter name=akeyless-sra
docker logs akeyless-sra-ssh --tail 50
```

**Fix:** the bastions wait for the gateway health check, so this follows
from an unhealthy gateway; fix the gateway first. If the gateway is healthy,
the bastion log names its own problem, most often the cache.

## Permission denied when the session opens

**Diagnosis:** connect reaches the target and the target refuses.

**Fix, in order:**

1. The username must be in the issuer's Allowed Users list, from
   `--allowed-users` in chapter 4.
2. The username must exist as a real OS user on the target.
3. The target must trust the CA: rerun the chapter 4 verification on the
   target host.
4. On OpenSSH 8.2 and newer, the `PubkeyAcceptedKeyTypes` line from
   chapter 4 must be present.

## Connection to the target times out

**Diagnosis:** connect hangs after the certificate is issued.

**Fix:** the target host firewall or security group does not allow SSH from
the Docker host. The bastion connects to the target from the Docker host
address, not from the user's address. Reread the network preparation in
chapter 8.

## Portal login rejects the API key

**Diagnosis:** the CLI works, the portal refuses.

**Fix:** expected behavior. The portal needs a browser login, which requires
SAML, OIDC, certificate, or LDAP authentication. An API key works for CLI
sessions only. Create an OIDC or SAML auth method and add its Access ID to
the role association as in chapter 7.

## Web or RDP session closes right after login

**Diagnosis:** authentication succeeds, the window opens, then closes
without an error.

**Fix:** cross-check `UAM_ADDR` and the account region as the first step;
misalignment between them produces exactly this symptom. See chapter 10 for
the variable.

## Host key warning after every container recreate

**Diagnosis:** users see a changed host key for the bastion after upgrades.

**Fix:** persistent host keys, chapter 10: set `SSH_HOST_KEYS_PATH` and add
the volume, then the bastion keeps one identity across recreates.

## Idle sessions drop

**Fix:** the four keepalive variables from chapter 10, applied to `sra.env`
and recreated. If a load balancer fronts the host, keep the interval times
the count below its idle timeout.

## Metrics target down in Prometheus

**Diagnosis:** the `akeyless-gateway` target on Prometheus Targets page is
DOWN.

**Fix, in order:**

1. `ENABLE_METRICS="true"` is set in `gateway.env`, and the gateway was
   recreated after the change.
2. The metrics profile is running: `docker ps --filter name=prometheus`.
3. From inside the network:
   `docker exec prometheus wget -qO- http://akeyless-gateway:8889` returns
   metrics text.

## Still stuck

Open an issue on the repository's GitHub issue tracker with: the chapter you
reached, the command you ran, the exact error text, and the tail of the
relevant container log. Remove any Access IDs, keys, and internal hostnames
before posting.
