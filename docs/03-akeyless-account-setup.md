# 3. Akeyless Account Setup

The gateway is a client of your Akeyless account, and it needs its own
identity. This chapter creates that identity: an API key auth method bound to
a role with narrow permissions. The principle is least privilege: the gateway
receives rights over Items and Targets under `/sra/*` and nothing else. User
authentication and user permissions are separate concerns handled in
chapter 7.

## Sign in to the CLI as an admin

Every command in this chapter runs with the Akeyless CLI acting as your admin
identity. If you have not configured a profile yet, create an API key for
yourself in the console, under your user menu and **Access Keys**, then:

```bash
akeyless configure --profile admin \
  --access-id p-xxxxxxxxxxxx \
  --access-key xxxxxxxxxx
```

**Expected output:** the CLI confirms the profile is saved. If it errors with
an access denied message, the key belongs to a user without admin
permissions; use an admin key.

## Create the gateway auth method

```bash
akeyless auth-method create api-key --name SraGatewayKey
```

**Expected output:**

```
Auth method SraGatewayKey successfully created
- Access ID: p-xxxxxxxxxxxxxx
- Access Key: <a long secret, shown once>
```

Save both in the information table from chapter 2 as the Gateway Access ID
and Gateway Access Key. The Access Key is shown once; if you lose it, run
`akeyless auth-method update api-key` with `--regenerate-key` or delete and
recreate the auth method.

## Create the gateway role

```bash
akeyless create-role --name SraGatewayRole
```

**Expected output:** `A new role named SraGatewayRole was successfully created`.

## Grant the role narrow permissions

Two rules, on the same path, with different rule types:

```bash
akeyless set-role-rule \
  --role-name SraGatewayRole \
  --path /sra/* \
  --capability read list create update

akeyless set-role-rule \
  --role-name SraGatewayRole \
  --path /sra/* \
  --rule-type target-rule \
  --capability read list
```

**What each rule grants:**

| Rule | Rule type | Grants on `/sra/*` |
|---|---|---|
| First | Items rule | Read, list, create, update secrets and keys |
| Second | Target rule | Read and list targets only, no create or update |

The split matters because the SSH Certificate Issuer, the DFC key behind it,
and the SSH targets all live under `/sra/`. The gateway must read and manage
those items and read the targets, and it has no reason to create targets
itself.

The target rule stops at `read` and `list` because the SSH certificate flow
in this runbook never writes targets. Dynamic secret producers do read and
write targets, so if you later add producers under `/sra/`, widen the target
rule with `create`, `update`, and `delete` at that point.

## Bind the role to the auth method

```bash
akeyless assoc-role-am \
  --role-name SraGatewayRole \
  --am-name SraGatewayKey
```

**Expected output:** `Association ass-xxxxxxxxxxxxxx was successfully created`.

## Verify the gateway identity works

Prove the credentials are valid before wiring them into the gateway:

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

If you see an authentication error, the Access Key is wrong or the auth
method was saved with different credentials; recreate the key as described
above.

## Understand ALLOWED_ACCESS_PERMISSIONS

The environment file in chapter 5 contains this line:

```json
ALLOWED_ACCESS_PERMISSIONS='[{"name":"Administrators","access_id":"p-zzzzzzzzzz","permissions":["admin"]}]'
```

It answers a different question than the role you just built. The role
controls what the gateway may read and write in the account.
`ALLOWED_ACCESS_PERMISSIONS` controls who may act as a gateway administrator,
which means who can sign in to the gateway console and configure the gateway.

| Field | Meaning |
|---|---|
| `name` | Display name for this entry |
| `access_id` | The Access ID of the auth method these users will authenticate against |
| `sub_claims` | Optional; restricts the entry to specific identities. Only meaningful for auth methods that carry an email claim, such as SAML or OIDC. Omit it for API keys |
| `permissions` | What they may do; `admin` grants full gateway configuration rights |

Set `access_id` to the Access ID of your own admin API key, the one you used
for `akeyless configure --profile admin` at the start of this chapter, not
the gateway key created above. An API key entry carries no `sub_claims`: the
key itself is the identity, so anyone holding it signs in. Chapter 7 revisits
this structure for granting narrower rights.

## What you have at this point

| Object | Value to carry forward |
|---|---|
| `SraGatewayKey` auth method | Gateway Access ID, Gateway Access Key, both go into `gateway.env` |
| `SraGatewayRole` | No action needed; already bound |
| Permissions JSON | One admin entry, your admin API key Access ID, no sub_claims |

## Next step

[Chapter 4: SSH Certificate Issuer](04-ssh-certificate-issuer.md) creates the
certificate authority that signs short-lived SSH certificates.
