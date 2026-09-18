# 7. Access Control

Four separate questions decide whether an SSH session happens. Confusing
them is the most common access-control mistake, so this chapter first
separates the layers, then builds the first end-user role.

## The four layers

| Layer | Question it answers | Where it lives |
|---|---|---|
| Privileged identity | Who can configure the gateway itself | `ALLOWED_ACCESS_PERMISSIONS` in `gateway.env` |
| Console admins | Who can sign in to the local console and with which permissions | Same JSON, the `permissions` array |
| Transport allowlist | Which auth methods may route API traffic through this gateway | `GATEWAY_AUTHORIZED_ACCESS_ID`, optional |
| SRA rules | Which user may open a session, and with which capability | Akeyless roles with SRA capabilities on issuer or target paths |

### The two env variables people mix up

`ALLOWED_ACCESS_PERMISSIONS` grants gateway administration. It answers who
manages the gateway. `GATEWAY_AUTHORIZED_ACCESS_ID` optionally restricts
which Access IDs may use the gateway as a transport for API traffic. It
answers which auth methods may pass through. Users connecting through SRA are
controlled by SRA rules, which are ordinary Akeyless role rules set in the
account, not in the environment file.

## SRA capabilities

The SRA permission model defines six capabilities. They are set on role rules
against the issuer or target path:

| Capability | CLI value | Effect |
|---|---|---|
| Allow Access | `allow_access` | Connect immediately, no reason and no approval needed |
| Request Access | `request_access` | Submit a reason and wait for an approver to grant a time-bounded window |
| Justify Access Only | `justify_access_only` | Connect immediately after entering a reason, no approval step |
| Approval Authority | `approval_authority` | Act as approver for requests on the path |
| Upload Files | `upload_files` | Upload files into the target over SFTP, granted on an issuer |
| Download Files | `download_files` | Download files from the target over SFTP, granted on an issuer |

Two rules of the model are worth internalizing early. A user is never allowed
to approve their own request: if one person holds both Approval Authority and
Request Access on a path, only a different approver can approve them. And
approving and connecting are separate authorizations; an approver with no
connect capability on a path can still gatekeep it, which is the intended
design.

## Create the first end-user auth method

As the admin CLI profile from chapter 3:

```bash
akeyless auth-method create api-key --name SraUsersKey
```

**Expected output:**

```
Auth method SraUsersKey successfully created
- Access ID: p-xxxxxxxxxxxxxx
- Access Key: <a long secret, shown once>
```

Give these to your first user; they are ordinary account credentials,
unrelated to the gateway identity of chapter 3.

For production, replace this API key with your corporate SAML or OIDC auth
method. The portal requires SAML, OIDC, certificate, or LDAP authentication,
because it needs a browser login; LDAP works only on your own gateway portal.
An API key works only for CLI sessions.

## Create the user role

Three commands build the role: create it, then two rules, then the binding.
The rules must be split, because `allow_access` is an SRA capability and
lives under a different rule type than `list`:

```bash
akeyless create-role --name SraUsers

akeyless set-role-rule \
  --role-name SraUsers \
  --path /sra/SSHCertIssuer \
  --capability list \
  --capability read

akeyless set-role-rule \
  --role-name SraUsers \
  --path /sra/SSHCertIssuer \
  --rule-type sra-rule \
  --capability allow_access

akeyless assoc-role-am \
  --role-name SraUsers \
  --am-name SraUsersKey
```

**Expected output:** `A new role named SraUsers was successfully created`,
then two rule confirmations, then
`Association ass-xxxxxxxxxxxxxx was successfully created`.

The first rule is an items rule. The `list` capability alone looks sufficient
for visibility, but certificate signing also reads the issuer; a user with
`list` and no `read` reaches the signing call and receives `401
Unauthorized`. The second rule carries the SRA capability under
`--rule-type sra-rule`; leaving the rule type off puts `allow_access` on an
items rule, where the CLI rejects it as an invalid capability.

Granting `allow_access` on the issuer path covers every host the issuer
serves. To narrow a user to one host, set
`--secure-access-enforce-hosts-restriction` on the issuer as shown in
chapter 8, or grant capabilities on a linked target path instead.

### Verify

```bash
akeyless list-roles --profile admin
```

**Expected output:** `SraGatewayRole` from chapter 3 and `SraUsers`, both
present.

## Choosing a stricter capability later

When your policy needs a reason or a second pair of eyes, swap the capability
on the rule: `request_access` instead of `allow_access` adds an approval step
in which an approver holding `approval_authority` grants a time-bounded
window from the Event Center; `justify_access_only` asks the user for a
reason at connect time without an approval wait. The flow is documented in
the Akeyless Request Access and Approval Flow guide, linked from this
repository's README. This runbook uses `allow_access` so the first session in
chapter 8 works with no other moving parts.

## What you have at this point

| Object | State |
|---|---|
| `SraUsersKey` | API key auth method for end users |
| `SraUsers` | Role with items rules `list` and `read`, plus an SRA rule with `allow_access`, on the issuer |
| Approval flow | Not used; the capability to enable it is documented above |

## Next step

[Chapter 8: First Session](08-first-session.md) turns SRA on for the issuer
and connects.
