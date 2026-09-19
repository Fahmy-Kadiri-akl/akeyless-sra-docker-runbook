# 11. Troubleshooting, Podman Stream

Each section is one symptom, with the diagnosis path and the fix. Work top to
bottom within a symptom; the causes are ordered by how often they occur.

## Every podman command shows an empty stack

**Diagnosis:** `sudo podman ps` shows the four containers, but a command run
without `sudo` shows nothing, or a script reports the containers as missing.

**Fix:** the stack runs rootful, in the root Podman store, and an unprefixed
`podman` command reads the calling user's own store, which holds nothing.
Prefix every command with `sudo`, exactly as written in this stream. If you
have already run commands without `sudo`, check that nothing was
accidentally created in the user store:

```bash
podman ps -a
```

**Expected output:** an empty table. Any container listed there belongs to a
user-store accident; remove it with `podman rm` before it causes confusion.

## Gateway container never becomes healthy

**Diagnosis:**

```bash
sudo podman logs --tail 50 akeyless-gateway
```

Look at the last error lines before the health check failures.

**Fix, by what the log says:**

| Log message | Cause | Fix |
|---|---|---|
| Authentication failure against Akeyless | Wrong `GATEWAY_ACCESS_ID` or `GATEWAY_ACCESS_KEY` | Re-verify with the `akeyless auth` command from chapter 3, then correct `gateway.env` |
| Cache or Redis connection refused | `REDIS_PASS` mismatch, or cache container down | Confirm `cache.env` matches, `sudo podman ps` shows `akeyless-cache` Up, then recreate |
| Port already in use | Another service holds 8000, 8080, or 8889 on the host | Stop the conflicting service or change the host side of the mapping |

## Gateway runs but never appears in the console

**Diagnosis:** four containers Up, health OK, gateway absent from the
console's gateway list.

**Fix:** `CLUSTER_NAME` and `GATEWAY_ACCESS_ID` form the identity pair; a
wrong cluster name registers the gateway against nothing. Compare
`CLUSTER_NAME` in `gateway.env` against the cluster name shown in the
Akeyless console under **Configuration**.

## Bastion stays in Created or restarts repeatedly

**Diagnosis:**

```bash
sudo podman ps -a --filter name=akeyless-sra
sudo podman logs --tail 50 akeyless-sra-ssh
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

## Connect fails with `username session_... not part of allowed user list`

**Diagnosis:** the issuer rejects the certificate request before any SSH
traffic flows.

**Fix:** every session runs under a generated username of the form
`session_<id>`, so the issuer's Allowed Users list must contain
`session_*`. Rerun the `update-ssh-cert-issuer` command from chapter 8 with
the full comma-separated list, for example
`--allowed-users 'ubuntu,session_*'`. Repeating the `--allowed-users` flag
replaces the whole list, which is how the entry usually disappears.

## Connect fails with 401 Unauthorized

**Diagnosis:** the user token is valid, but the signing call is rejected.

**Fix:** if the role or its association changed less than a minute ago, wait
and retry first: permission changes take up to about a minute to propagate,
and the failure during that window looks identical to a missing rule. If the
role is older, the user's role needs an items rule with both `list` and
`read` on the issuer path. The `list` capability alone leaves the signing
call unauthorized; rerun the first `set-role-rule` from chapter 7 with
`--capability list --capability read`.

## Portal shows no targets after login

**Diagnosis:** portal login succeeds and the target list is empty, although
the issuer has SRA enabled from chapter 8.

**Fix:** the user's role has items rules but no SRA rule. Item and target
capabilities never grant sessions; a role with full `read` and `list` on the
issuer path still sees no targets. Only a rule of type `sra-rule` with a
capability such as `allow_access` makes targets appear. Add the second
`set-role-rule` from chapter 7 and allow a minute for propagation.

## Connect fails with `ERR! access-id is not in the allowed list`

**Diagnosis:** the connect is rejected before authentication.

**Fix:** `GATEWAY_AUTHORIZED_ACCESS_ID` is set in `gateway.env` and the user
auth method's Access ID is missing from the list. The variable takes a
comma-separated list of user auth method Access IDs. Add the missing one and
recreate the gateway, or unset the variable to remove the restriction
entirely. Chapter 7 explains what belongs in the list.

## Port refused although nothing seems to listen on it

**Diagnosis:** `sudo podman port akeyless-sra-ssh` shows the mapping and a
TCP probe to the port fails with connection refused, on a host that also
runs Kubernetes.

**Fix:** stale container-networking NAT rules can intercept a host port
after the service that created them is gone. Podman's own port handler
listening on the port is no proof it is reachable. Check both firewall
layers:

```bash
sudo iptables -t nat -S | grep 2222
sudo iptables-legacy -t nat -S | grep 2222
```

Any `CNI-DN-` or `KUBE-` rule mentioning the port is a leftover from a
Kubernetes hostPort or service. Delete the rule with
`iptables-legacy -t nat -D <chain> <rule spec>`, or reboot the host, then
restart the Compose stack. The free-port probe in chapter 2 catches this
before the first start.

## Bastion logs show rsyslog or CheckServicesStatus errors

**Diagnosis:** `sudo podman logs akeyless-sra-ssh` prints a crashing rsyslog
loop, or `CheckServicesStatus` entries with exit code 3 every few seconds,
while sessions work.

**Fix:** ignore them. Under the privileged container the bastion's service
supervisor cannot manage system services the way it expects, so its
watchdog reports failures that do not affect SSH proxying. The container is
healthy as long as `sudo podman ps` shows `Up` and sessions connect.

## Connect prints the target greeting, then the session dies

**Diagnosis:** `akeyless connect` reaches the target, the target's login
greeting appears, then the session closes with no command output. The
failing connect exits 254, or fails intermittently with 255; both numbers
are the same failure. This affects targets that are containers launched by
hand from an interactive SSH session, which is common in lab setups.

**Fix:** every process inside such a target container inherits the loginuid
of the session that launched it, and that loginuid is already set and
immutable. The stock `pam_loginuid` line in the target's `/etc/pam.d/sshd`
is `session required`, so PAM cannot write the loginuid, fails to open the
session, and sshd closes the connection after the greeting but before
running the command. On the target, relax the line to:

```
session optional pam_loginuid.so
```

That form is standard for sshd inside containers. Targets that are normal
hosts keep the stock line; there the kernel writes the loginuid at real
login time and the strict setting works.

## Connect prints the banner, then ends with exit status 255 and no output

**Diagnosis:** `akeyless connect` authenticates, the Akeyless banner
appears, then the session ends immediately with no command output and exit
status 255. There is no `Permission denied` and no timeout; the session
simply produces nothing.

Check the SSH bastion log:

```bash
sudo podman logs --tail 100 akeyless-sra-ssh
```

The signature is request forwarding that starts and dies, most visibly:

```
ERROR ... Failed to send request: EOF
```

repeated for the `env` and `shell` requests, sometimes followed by
`clientRequests channel closed or nil request received, stopping request
forwarding`. If those lines are absent and other symptoms fit better, this
is not the failure you have.

**Fix:** the bastion's in-memory session state has wedged. The condition was
observed on a bastion whose sessions had been interrupted repeatedly or
whose stack was recreated around other work; the account side, the issuer,
and the target all verified healthy at the same time. Restart the SSH
bastion, which drops only in-flight sessions:

```bash
sudo podman restart akeyless-sra-ssh
```

In the observed incident a restart restored one session and the failure then
returned. The verified complete recovery is a full stack recreate:

```bash
sudo podman compose --profile gateway --profile sra down
sudo podman compose --profile gateway --profile sra up -d
```

After that recreate, five consecutive connects each ran their command on the
target and closed cleanly. The cause inside the bastion was not identified,
so this entry documents the signature and the recovery, not a mechanism.

## Connection to the target times out

**Diagnosis:** connect hangs after the certificate is issued.

**Fix:** the target host firewall or security group does not allow SSH from
the container host. The bastion connects to the target from the container
host address, not from the user's address. Reread the network preparation in
chapter 8.

## Portal login rejects the API key

**Diagnosis:** the CLI works, the portal at `/sra/portal` refuses.

**Fix:** expected behavior, and it does not apply to the local console.
Distinguish the two web surfaces. The local console at port 8000 accepts
API-key sign-in, as verified in chapter 6, as long as the Access ID appears
in `ALLOWED_ACCESS_PERMISSIONS`. The SRA portal needs a browser login, which
requires SAML, OIDC, certificate, or LDAP authentication. For portal users,
create an OIDC or SAML auth method and add its Access ID to the role
association as in chapter 7.

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
2. The metrics profile is running:
   `sudo podman ps --filter name=prometheus`.
3. From inside the network:
   `sudo podman exec prometheus wget -qO- http://akeyless-gateway:8889`
   returns metrics text.

## Still stuck

Open an issue on the repository's GitHub issue tracker with: the chapter you
reached, the command you ran, the exact error text, and the tail of the
relevant container log from `sudo podman logs`. Remove any Access IDs, keys,
and internal hostnames before posting.
