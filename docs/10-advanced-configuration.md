# 10. Advanced Configuration

Optional settings, each independent of the others. Apply them when a real
need appears; the deployment works without any of them.

## Session keepalives

Idle sessions dropping is usually the keepalive defaults, not a network
fault. Set the four variables in `sra.env` to make both legs of the session,
client to bastion and bastion to target, send keepalives:

| Variable | Meaning | Example |
|---|---|---|
| `SSH_CLIENT_ALIVE_INTERVAL` | Seconds between checks on the client side | `120` |
| `SSH_CLIENT_ALIVE_COUNT_MAX` | Failed checks before the client leg is dropped | `2` |
| `SSH_SERVER_ALIVE_INTERVAL` | Seconds between keepalives toward the target | `120` |
| `SSH_SERVER_ALIVE_COUNT_MAX` | Failed keepalives before the target leg is dropped | `2` |

Apply with the recreate sequence from chapter 9. If sessions still drop
behind a load balancer, compare the keepalive math against the balancer's
idle timeout: the interval times the count must stay below the balancer
timeout.

## Persistent bastion host keys

By default the SSH bastion regenerates its host keys on recreate, so clients
see a changed host key warning after every container rebuild. Persist them:

1. Set in `sra.env`: `SSH_HOST_KEYS_PATH="/sra/host-keys"`.
2. Add a volume for the path in the `akeyless-ssh` service of the Compose
   file, for example `- ./ssh-host-keys:/sra/host-keys`.
3. Recreate with the chapter 9 sequence.

## Gateway remote-access tuning

Three settings apply through the gateway CLI against your gateway URL:

```bash
akeyless gateway update remote-access --keyboard-layout de-de-qwertz \
  --gateway-url http://sra.example.internal:8000
```

| Setting | Flag | When you need it |
|---|---|---|
| Keyboard layout | `--keyboard-layout <layout>` | Windows sessions with a non-US keyboard; default `en-us-qwerty` |
| Key exchange algorithms | `--kexalgs <algorithm>` | Targets that restrict key exchange, for example `curve25519-sha256` |
| Legacy SSH algorithm | `--legacy-ssh-algorithm true` | Older targets that reject the modern signing algorithm and need `ssh-rsa-cert-v01@openssh.com` |

The legacy algorithm flag exists because both SSH and RDP access build on SSH
certificates. Prefer fixing the target with the `PubkeyAcceptedKeyTypes`
line from chapter 4 over enabling legacy signing.

## Restricting portal redirects

Docker SSH bastion deployments can set `ALLOWED_BASTION_URLS` as an
allowlist. The portal removes user-provided endpoint URLs that are not on
it. Set it in `gateway.env` when users should reach bastions only through
known addresses.

## Non-default account regions

If your account is in a region other than the default, the bastions need
`UAM_ADDR` in their environment, pointing at the authentication service
endpoint for your region. The symptom of a misaligned `UAM_ADDR` is specific
and quiet: RDP and web sessions close immediately after authentication
succeeds, with nothing useful in the logs. When that happens, cross-check the
`UAM_ADDR` value and the account region first.

## Session recordings

RDP sessions can be recorded. Configure it in the local console:
**Remote Access**, then **Session Recording**, then **RDP Recordings**,
enable it, and choose storage: local path on the gateway, or upload to Amazon
S3, S3-compatible storage, or Azure Blob. Quality, compression, and
encryption options live on the same page.

## TLS on the gateway

This deployment runs plain HTTP, which is why it uses the local console and
not `console.akeyless.io`: the public console cannot manage a plain-HTTP
gateway. To move to TLS:

1. Set `ENABLE_TLS="true"` and keep `ENABLE_TLS_CONFIGURE="true"` in
   `gateway.env`.
2. Mount your certificate and key into the gateway container; the Compose
   file shows the volume lines as a commented block with the expected
   container paths, `akeyless-api-cert.crt` and `akeyless-api-cert.key`.
3. Recreate, then serve the console and portal over HTTPS on port 8000 and
   update every `-g` URL your users use to `https://`.

## Next step

[Chapter 11: Troubleshooting](11-troubleshooting.md) when something does not
behave.
