# Docker Stream

This stream runs the Compose kit on Docker Engine 20.10 or newer with Docker
Compose v2. Every command in chapters 2, 6, 9, and 11 below is written for
that setup and runs as shown, with `docker compose`.

The remaining chapters are shared with the Podman stream and live in
`docs/common/`. They describe the Akeyless account, the issuer, the Compose
kit, access control, the first session, and advanced settings; none of that
differs between the runtimes.

Reading order, chapter by chapter:

| # | Chapter | Lives in |
|---|---|---|
| 1 | [Architecture Overview](../common/01-architecture-overview.md) | shared |
| 2 | [Prerequisites](02-prerequisites.md) | this stream |
| 3 | [Akeyless Account Setup](../common/03-akeyless-account-setup.md) | shared |
| 4 | [SSH Certificate Issuer](../common/04-ssh-certificate-issuer.md) | shared |
| 5 | [Compose Configuration](../common/05-compose-configuration.md) | shared |
| 6 | [Start and Verify](06-start-and-verify.md) | this stream |
| 7 | [Access Control](../common/07-access-control.md) | shared |
| 8 | [First Session](../common/08-first-session.md) | shared |
| 9 | [Day-2 Operations](09-day-2-operations.md) | this stream |
| 10 | [Advanced Configuration](../common/10-advanced-configuration.md) | shared |
| 11 | [Troubleshooting](11-troubleshooting.md) | this stream |

Running Podman instead? Use the [Podman stream](../podman/README.md). The
Compose kit in `compose/` is identical for both.
