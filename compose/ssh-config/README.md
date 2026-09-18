# ssh-config

Drop the SSH certificate authority public key of your SSH Certificate Issuer
into this directory, named exactly `ca.pub`:

```
ssh-config/
└── ca.pub
```

The Compose file mounts this folder into the SSH bastion at
`/var/akeyless/creds/`, which is where the bastion looks for the trusted CA
public key.

You export the key from the DFC key behind your SSH Certificate Issuer in
[docs/04-ssh-certificate-issuer.md](../../docs/04-ssh-certificate-issuer.md).

The bastion will fail to start sessions without this file. See
[docs/11-troubleshooting.md](../../docs/11-troubleshooting.md) if it is missing.
