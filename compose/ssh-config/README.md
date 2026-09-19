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
[chapter 4](../../docs/common/04-ssh-certificate-issuer.md).

The bastion will fail to start sessions without this file. See chapter 11 of
your stream, [Docker](../../docs/docker/11-troubleshooting.md) or
[Podman](../../docs/podman/11-troubleshooting.md), if it is missing.
