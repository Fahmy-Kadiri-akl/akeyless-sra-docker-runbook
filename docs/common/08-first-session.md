# 8. First Session

Everything built so far converges here: the issuer learns which host it may
serve, SRA is switched on, and your first user opens a session through the
bastion. At the end of this chapter the banner from the Akeyless guide
appears on your target host.

## Enable SRA on the issuer

The issuer currently signs certificates but offers no sessions. Enable Secure
Remote Access on it, from the console or the CLI, as the admin.

**Console path:** open **Items**, select `/sra/SSHCertIssuer`, open the
**Secure Remote Access** tab, click the pencil icon, enable **Secure Remote
Access**, and fill in the fields from the table below. Save with the tick
mark icon.

**CLI equivalent:**

```bash
akeyless update-ssh-cert-issuer \
  --name /sra/SSHCertIssuer \
  --signer-key-name /sra/SSHSignerKey \
  --allowed-users 'ubuntu,session_*' \
  --ttl 300 \
  --secure-access-enable true \
  --secure-access-api http://sra.example.internal:9900 \
  --secure-access-ssh sra.example.internal:2222 \
  --secure-access-ssh-creds-user ubuntu \
  --host-provider=explicit \
  --secure-access-host 10.0.1.23
```

Replace `sra.example.internal`, `ubuntu`, and `10.0.1.23` with your container
host, target username, and target host from chapter 2.

The three flags repeated from chapter 4, `--signer-key-name`,
`--allowed-users`, and `--ttl`, are required again here. An update call
without them fails with `required parameter missing`, because the update
rebuilds the issuer definition rather than patching single fields.

**What each flag sets:**

| Flag | Value here | Meaning |
|---|---|---|
| `--secure-access-enable` | `true` | Switches SRA on for the issuer |
| `--secure-access-api` | `http://<host>:9900` | The SSH bastion control API the portal and gateway use |
| `--secure-access-ssh` | `<host>:2222` | The SSH bastion endpoint users land on |
| `--secure-access-ssh-creds-user` | `ubuntu` | The default SSH username, must come from the issuer's Allowed Users list |
| `--host-provider` | `explicit` | Hosts are listed on the issuer; `target` switches to linked targets |
| `--secure-access-host` | `10.0.1.23` | A host users may reach; repeat the flag for more, CIDR notation supported |

Repeat `--secure-access-host` for every target host. To restrict connections
to exactly the listed hosts, add
`--secure-access-enforce-hosts-restriction true`; without it, users holding
`allow_access` may connect to hosts the issuer can reach even when unlisted.

The `--allowed-users` value repeats the list from chapter 4, including
`session_*`. The bastion opens each session under a generated username of the
form `session_<id>` and signs a certificate for it with this issuer. If the
pattern is missing, connect fails with
`username session_... not part of allowed user list`. The comma-separated
form matters: repeating `--allowed-users` replaces the whole list instead of
extending it, so an update intended to add a name can silently drop
`session_*`.

### Verify

```bash
akeyless describe-item --name /sra/SSHCertIssuer --profile admin
```

**Expected output:** a JSON blob whose `item_general_info` object contains a
`secure_remote_access_details` section like this:

```json
"secure_remote_access_details": {
  "enable": true,
  "bastion_api": "http://sra.example.internal:9900",
  "bastion_ssh": "sra.example.internal:2222",
  "ssh_user": "ubuntu",
  "host": ["10.0.1.23"],
  "is_cli": true,
  "host_provider_type": "explicit"
}
```

If the section is absent, the update did not apply; rerun the command and
check for an error message.

## One-time network preparation for targets

The SSH bastion connects to targets from the container host, not from the
user's machine. On each target host, allow SSH from the container host
address in the host firewall or security group. The Akeyless guide's most
common failure is skipping this step: the session then times out at connect.

## Connect from the CLI as the first user

On the user's machine, authenticate with the credentials from chapter 7:

```bash
akeyless auth \
  --access-id p-xxxxxxxxxxxx \
  --access-type access_key \
  --access-key xxxxxxxxxx
```

**Expected output:**

```
Authentication succeeded.
Token: t-xxxxxxxxxxxxxxxx
```

Copy the token, then:

```bash
akeyless connect \
  -t "ubuntu@10.0.1.23:22" \
  -c /sra/SSHCertIssuer \
  -v sra.example.internal:2222 \
  -g http://sra.example.internal:8000 \
  --token t-xxxxxxxxxxxx
```

**What each flag does:**

| Flag | Meaning |
|---|---|
| `-t` | OS user and target host with port; must match the issuer's username and an allowed host |
| `-c` | The SSH Certificate Issuer, with SRA enabled |
| `-v` | The SSH bastion endpoint, host port 2222 on the container host |
| `-g` | The gateway base URL the CLI authenticates and signs against |
| `--token` | The user token from `akeyless auth` |

Two prompts appear on the first connect run and never again. The CLI asks
`Can't find SSH keypair, would you like to create one? (Y/n)`; answer `Y`,
because connect needs a local keypair to request certificates with. Then
OpenSSH asks to confirm the bastion's host key; answer `yes`, or add
`StrictHostKeyChecking accept-new` for the bastion host in `~/.ssh/config`
when you script the connection.

### What success looks like

The terminal shows the Akeyless banner:

```
You are connecting to your remote server via Akeyless Bastion
```

followed by an ordinary shell prompt on the target host. The certificate used
dies after the issuer's five minute TTL; the session continues but no new
session can ride on it.

If you see `Permission denied`, the username is missing from the issuer's
Allowed Users or does not exist on the target. If connect fails with
`username session_... not part of allowed user list`, the issuer's Allowed
Users list lost `session_*` in an update; rerun the update command above
with the full comma-separated list. If the connection times out, return to
the network preparation above.

## Connect from the browser

The web bastion serves browser SSH sessions.

1. Open `http://sra.example.internal:8888` or the portal at
   `http://sra.example.internal:8000/sra/portal`.
2. Sign in. The portal requires SAML, OIDC, certificate, or LDAP
   authentication; the API key from chapter 7 works only with the CLI and the
   web client's token flow.
3. Pick the host from the portal list, or enter it, and the session opens in
   the browser.

Uploads and downloads over SFTP work in browser sessions when the target
supports SFTP, provided the user's role carries `upload_files` or
`download_files` from chapter 7.

## What you have at this point

A complete, working deployment: gateway, cache, both bastions, a signing CA
trusted by targets, a user role, and a verified end-to-end session.

## Next step

Chapter 9 is where the streams split: its commands differ per runtime.
Continue with [chapter 9 of the Docker stream](../docker/09-day-2-operations.md)
or [chapter 9 of the Podman stream](../podman/09-day-2-operations.md), which
keeps it running: upgrades, logs, and metrics.
