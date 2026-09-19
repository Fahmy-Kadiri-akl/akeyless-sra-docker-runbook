# 4. SSH Certificate Issuer

Short-lived SSH certificates need a certificate authority. In Akeyless the CA
is a DFC key, and the SSH Certificate Issuer defines how that key signs user
certificates: which logins, how long, with which options. This chapter
creates both, exports the CA public key, and makes your target hosts trust
it.

## Create the DFC key

```bash
akeyless create-dfc-key \
  --name /sra/SSHSignerKey \
  --alg RSA2048
```

**Expected output:** `A new RSA2048 DFC key named /sra/SSHSignerKey was
successfully created`. The private key never leaves Akeyless; only public
parts are exportable.

## Create the issuer

```bash
akeyless create-ssh-cert-issuer \
  --name /sra/SSHCertIssuer \
  --signer-key-name /sra/SSHSignerKey \
  --allowed-users 'ubuntu,session_*' \
  --ttl 300
```

Replace `ubuntu` with the target SSH username you chose in chapter 2. The
`session_*` entry is required by the SSH bastion itself: every connect
session is opened under a generated username of the form `session_<id>`,
signed by the same issuer, and the issuer rejects certificate requests for
names outside this list. If you have several target usernames, pass them all
comma-separated together with `session_*`, for example
`'ubuntu,ec2-user,session_*'`. Repeating the flag replaces the whole list;
add and remove names only through the comma syntax.

**What each flag controls:**

| Flag | Value in this runbook | Meaning |
|---|---|---|
| `--signer-key-name` | `/sra/SSHSignerKey` | The DFC key acting as the CA |
| `--allowed-users` | `ubuntu,session_*` | The only login names that certificates may carry; `session_*` covers the bastion's per-session usernames |
| `--ttl` | `300` | Certificate lifetime in seconds, five minutes |
| `--secure-access-enable` | not set yet | Enabled in chapter 8 when the gateway is running |

The five minute TTL is deliberate. A certificate this short cannot be reused
after the session for which it was issued. Users authenticate to Akeyless
again for the next session, and every authentication is visible in the
account's audit log.

## Export the CA public key

```bash
akeyless get-rsa-public \
  --name /sra/SSHSignerKey \
  --json \
  --jq-expression='.ssh' > compose/ssh-config/ca.pub
```

Run this from the repository root so the key lands in the `ssh-config`
folder that chapter 5 mounts into the SSH bastion.

**Expected output:** no console output, and the file
`compose/ssh-config/ca.pub` now contains one line starting with
`ssh-rsa AAAA...`.

### Verify

```bash
cat compose/ssh-config/ca.pub
```

**Expected output:** a single line beginning with `ssh-rsa`. If the file is
empty, the `--jq-expression` filter did not match; run the command without
`--jq-expression`, inspect the JSON, and check the `.ssh` field name.

## Make target hosts trust the CA

On each target host, as your sudo user, install the public key and tell sshd
to trust it:

```bash
sudo cp ca.pub /etc/ssh/ca.pub
echo 'TrustedUserCAKeys /etc/ssh/ca.pub' | sudo tee -a /etc/ssh/sshd_config
```

If the target runs OpenSSH 8.2 or newer, which current distributions do, the
RSA certificate type must also be re-enabled, because newer OpenSSH versions
disable it by default:

```bash
echo 'PubkeyAcceptedKeyTypes +ssh-rsa-cert-v01@openssh.com' | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart sshd
```

Copy `ca.pub` to the target first, with `scp` or by pasting the single line
into an editor. The restart drops existing SSH sessions only briefly; stay
logged in while you restart so you cannot lock yourself out.

### Verify

```bash
sudo sshd -T | grep -iE 'trustedusercakeys|pubkeyaccepted'
```

**Expected output:** `trustedusercakeys /etc/ssh/ca.pub`, and a
`pubkeyaccepted...` line containing `ssh-rsa-cert-v01@openssh.com`. OpenSSH
8.x prints that line as `pubkeyacceptedkeytypes`; OpenSSH 9.x prints it as
`pubkeyacceptedalgorithms`, the renamed form of the same directive. If sshd
prints a configuration error instead, check the lines you appended for
typos; sshd refuses to start a session with a bad directive.

## What you have at this point

| Object | State |
|---|---|
| `/sra/SSHSignerKey` | DFC key, RSA 2048, private part in Akeyless |
| `/sra/SSHCertIssuer` | Signs certificates for `ubuntu` and `session_*`, TTL 300 seconds, SRA disabled for now |
| `compose/ssh-config/ca.pub` | Exported CA public key, ready to mount |
| Target hosts | Trusting the CA, no Akeyless software installed |

## Next step

[Chapter 5: Compose Configuration](05-compose-configuration.md) wires the
gateway identity and these files into the environment files.
