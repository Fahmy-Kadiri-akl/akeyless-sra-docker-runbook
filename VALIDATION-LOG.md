# Validation Log

This runbook was executed end to end on a clean host, chapters 1 through 8,
in September 2026. Every command ran as written, from a fresh clone with no
carryover state. Each deviation this pass found is recorded here with its
root cause, and the fix for every one of them is in this revision of the
runbook and the Compose kit.

## Environment

| Aspect | Value |
|---|---|
| Host | Fresh x86-64 virtual machine, current Docker Engine and Compose v2 |
| Akeyless CLI | Current release at validation time |
| Images | `akeyless/base` and `akeyless/zero-trust-bastion` current tags, Redis 8.2 Alpine per the Compose file |
| Account | Disposable lab Akeyless account |
| Target | One containerized Linux host running OpenSSH 9.x, reached over the Docker network |
| User flow | One admin API key, one end-user API key, roles and issuer under `/sra/` |

## Scope

Chapters 1 through 8: prerequisites, account setup, issuer, Compose
configuration, start and verify, access control, first session. Chapter 9
upgrade paths and chapter 10 optional features were outside this pass.

## How the runbook is structured

Chapters live in three places. `docs/common/` holds the seven chapters that
apply to both runtimes. `docs/docker/` and `docs/podman/` hold chapters 2,
6, 9, and 11, one native copy per runtime: the Docker stream is written
entirely with `docker compose`, the Podman stream entirely with rootful
`sudo podman compose`. Findings below name chapters by number; a runtime
chapter cited here exists in both streams.

## Findings and fixes

| # | Symptom during the pass | Root cause | Fix landed |
|---|---|---|---|
| 1 | Cache container exited immediately, `exec operation not permitted` | `no-new-privileges` blocked the Redis entrypoint from execing under a different uid | Flag removed from the Compose cache service, rationale in chapter 5 |
| 2 | SSH bastion exited during entrypoint | The image's default uid 1001 cannot write to `/etc/ssh` or run `usermod` | `user: "0:0"` added to the Compose SSH bastion service |
| 3 | Expected outputs described JSON blobs | The Akeyless CLI prints plain text confirmations for these commands | Chapters 3, 4, 7, and 8 expected outputs rewritten to the exact strings |
| 4 | Chapter 2 and 11 cited `akeyless describe-account-details` | No such CLI command exists | Chapter 2 points at the console Configuration page; chapter 11 compares `gateway.env` against the console |
| 5 | Health check documented as body `OK` | The endpoint returns `Health Check Ok` | Chapter 6 corrected |
| 6 | Connect failed with `username session_... not part of allowed user list` | Every session runs under a generated `session_<id>` username the issuer must be allowed to sign | `session_*` added to `--allowed-users` in chapters 4 and 8, comma syntax documented, symptom added to chapter 11 |
| 7 | `update-ssh-cert-issuer` failed with `required parameter missing` | Updates rebuild the issuer definition; signer key, allowed users, and TTL must be repeated | Chapter 8 command carries all three flags |
| 8 | Verify block named a nonexistent `secure_access_details` top-level field | SRA settings live under `item_general_info.secure_remote_access_details`, username field `ssh_user` | Chapter 8 verify block shows the exact section |
| 9 | `set-role-rule` rejected `allow_access`; signing returned 401 with `list` alone | SRA capabilities live under `--rule-type sra-rule`; signing also needs the items-rule `read` capability | Chapter 7 builds two rules; 401 symptom added to chapter 11 |
| 10 | Console sign-in guidance relied on an email `sub_claims` entry | `sub_claims` is meaningful only for SAML or OIDC; the local console accepts API-key sign-in when the Access ID is listed | Chapters 2, 3, and 6 and `gateway.env.example` corrected; chapter 11 distinguishes console from portal |
| 11 | First connect run stalled on two undocumented prompts | Key pair creation prompt and OpenSSH host-key confirmation appear once | Both documented in chapter 8 |
| 12 | Chapter 4 verification grepped for `pubkeyacceptedkeytypes` | OpenSSH 9.x prints the renamed `pubkeyacceptedalgorithms` | Chapter 4 verification accepts both forms |
| 13 | Port 2222 refused although Docker showed the mapping | Stale Kubernetes NAT rules on the host intercepted the port; a listening docker-proxy is no proof of reachability | Free-port probe added to chapter 2, diagnosis symptom added to chapter 11 |
| 14 | Bastion logs showed a crashing rsyslog loop and `CheckServicesStatus` exit code 3 | The service supervisor cannot manage system services inside the privileged container; SSH proxying is unaffected | Documented as benign in chapter 11 |
| 15 | Setting `GATEWAY_AUTHORIZED_ACCESS_ID` blocked user connects with `ERR! access-id is not in the allowed list` | The allowlist gates all traffic routed through the gateway, SRA sessions included, so user auth method Access IDs must be in it | Chapter 7 documents the correct list, chapter 5 and `gateway.env.example` carry the variable, symptom added to chapter 11 |

## Follow-up: transport allowlist

After the main pass, `GATEWAY_AUTHORIZED_ACCESS_ID` was exercised on the same
deployment in three states: unset, set without the user auth method's Access
ID, and set with it. The unset and correctly set states both connected; the
middle state failed before authentication with `ERR! access-id is not in the
allowed list`. Finding 15 and the chapter 7 guidance come from that sequence.

## Result

The pass ended with a complete working deployment: four containers healthy,
the gateway registered in the account console, API-key sign-in to the local
console verified, and an end-to-end SSH session through the bastion on host
port 2222, confirmed by the Akeyless banner and command execution on the
target as the expected OS user.

## Podman pass

A second pass, September 2026, ran the same Compose kit on Podman 4.9
rootful with a standalone Docker Compose v2 as the external provider,
invoked directly as `podman compose` and through the `podman-docker` shim.
The Docker findings above still stand unchanged. The differences this pass
found are folded into the Podman stream:

| # | Symptom during the pass | Root cause | Fix landed |
|---|---|---|---|
| P1 | Every command without `sudo` saw an empty stack | An unprefixed `podman` command reads the calling user's store, and the stack runs rootful | Podman stream chapter 2: prefix every command with `sudo`; chapter 11 lists the empty stack as its own symptom |
| P2 | `podman logs akeyless-gateway --tail 50` failed with `no container with name or ID "--tail" found` | Podman rejects a flag placed after the container name | Both streams write flags before container names, a form Docker Engine also accepts |
| P3 | Connections to the gateway through its second network were dropped | The gateway sits on two bridge networks, so it holds two routes, and strict reverse-path filtering discards traffic arriving on the secondary interface | `sysctls` on the gateway service set `rp_filter` to loose mode, validated on Podman and Docker Engine |
| P4 | Connect printed the containerized target's greeting, then died with exit 254, or failed intermittently with 255 | The target container inherited the launching session's loginuid, the stock `pam_loginuid required` line in its `/etc/pam.d/sshd` cannot rewrite it, PAM opens no session, and sshd exits after the greeting | Chapter 11 of both streams; containerized targets need `session optional pam_loginuid.so` |
| P5 | After a host reboot the stack would stay down | Podman does not honor `restart: always` on boot by itself | Podman stream chapters 2 and 9: enable `podman-restart.service` |

Lab-host specifics that are not general requirements: the Docker snap was
stopped because two container engines on one host overwrite each other's
firewall rules, host firewall forwarding was opened for the Podman bridge,
and the test client ran on the host itself, so reaching the published ports
through the host's external address needed NAT hairpin. The `:Z` mount
labels are no-ops on the non-SELinux lab host and stay untested on SELinux.

Verified after the fixes: ten consecutive connects plus a PTY session, three
connects after a full `down`/`up` recreate with the sysctls intact, one
connect after tearing the stack down and restoring it around a Docker Engine
regression run, the web bastion answering HTTP 400 on a bare GET as
documented, and the Compose file with the sysctls block still bringing up a
healthy gateway under Docker Engine 29.6. The reboot path was verified as
configuration, `podman-restart.service` enabled and active with the restart
policies set; an actual reboot was not exercised.

## Follow-up: sessions failing silently after abnormal session history

A later exercise on the Podman host produced a failure the passes above had
not seen: `akeyless connect` authenticated, printed the banner, then ended
with exit status 255 and no command output. The SSH bastion log showed the
session reaching request forwarding and then `Failed to send request: EOF`
for the `env` and `shell` requests. Everything else verified healthy at the
same time: the Compose kit, the environment files, the CA mount, the issuer,
the roles, the per-session chroot the bastion builds, and the target.

The failure appeared on a bastion that had been through a stack recreate and
repeated interrupted and hand-driven sessions during deep diagnosis. A full
stack down and up restored service completely: five consecutive connects each
ran their command on the target and closed with a clean exit-status
forwarding sequence in the bastion log. The internal cause inside the
bastion was not identified, so chapter 11 of both streams documents the
observed signature and the verified recovery, not a root cause.
