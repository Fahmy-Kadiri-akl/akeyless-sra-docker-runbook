# 6. Start and Verify

This chapter starts the stack and verifies each layer: containers running,
gateway healthy, gateway authenticated, URLs reachable. Work through the
verifications in order; each one assumes the previous.

## Start

From the `compose/` directory:

```bash
docker compose --profile gateway --profile sra up -d
```

**What happens:** Docker pulls the images on first run, the gateway image is
the large one at roughly one gigabyte, then starts the containers in
dependency order: cache first, gateway after a health wait, bastions last.
First start takes several minutes on a fresh host; later starts take seconds.

The two profiles matter. The `gateway` and `sra` profiles both include the
gateway and cache; the `sra` profile adds both bastions. Starting with both
profiles gives the full deployment.

## Verify the containers

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

**Expected output:** four containers:

| Name | Status | Ports |
|---|---|---|
| `akeyless-gateway` | `Up ... healthy` | 8000, 8080, 8889 |
| `akeyless-cache` | `Up` | 127.0.0.1:6379 |
| `akeyless-sra-ssh` | `Up` | 2222, 9900 |
| `akeyless-sra-web` | `Up` | 8888 |

The bastions take up to a minute to show `Up` because they wait for the
gateway health check. If a bastion stays `Created` or exits, the gateway did
not become healthy; continue to the next verification to see why.

## Verify the gateway health endpoint

```bash
curl -f http://localhost:8080/health
```

**Expected output:** `OK` and HTTP 200. The `-f` flag makes curl fail on any
error status. If the connection is refused, the gateway is still starting;
wait sixty seconds and retry. If it keeps failing, check the logs:

```bash
docker logs akeyless-gateway --tail 50
```

The two common failure messages are an authentication error against
Akeyless, which means a wrong `GATEWAY_ACCESS_ID` or `GATEWAY_ACCESS_KEY`,
and a cache connection error, which means a `REDIS_PASS` mismatch between
`cache.env` and what Redis started with.

## Verify the gateway reached the account

In the Akeyless console, open the gateway list. Your gateway appears with
the `CLUSTER_NAME` you set. Alternatively:

```bash
docker logs akeyless-gateway 2>&1 | grep -i 'cluster'
```

**Expected output:** lines naming your cluster. A gateway that never appears
in the console despite healthy containers is usually the wrong `CLUSTER_NAME`
pairing; the pair of cluster name and Access ID is the gateway identity, and
both must match the account.

## Collect your URLs

Replace `sra.example.internal` with your Docker host address from chapter 2:

| URL | Purpose |
|---|---|
| `http://sra.example.internal:8000/console` | Local console for gateway configuration |
| `http://sra.example.internal:8000/sra/portal` | SRA portal for users |
| `http://sra.example.internal:8000/api/v1` | Gateway REST API |
| `http://sra.example.internal:8888` | Web bastion directly |
| `http://sra.example.internal:2222` | SSH bastion for `akeyless connect` |

Open the console URL and sign in as the admin you named in the
`ALLOWED_ACCESS_PERMISSIONS` JSON of chapter 3. The sign-in method there was
an API key, so use **API Key** sign-in with that Access ID and Access Key.

## What success looks like

Four containers `Up`, `curl -f` returning OK, the gateway visible in the
Akeyless console, and the local console reachable in a browser.

## What you have at this point

A running gateway and both bastions, connected to your account. Nothing can
reach a target through it yet: no user holds SRA permissions and the issuer
does not offer sessions. Those are the next two chapters, in that order.

## Next step

[Chapter 7: Access Control](07-access-control.md) creates the first
end-user role.
