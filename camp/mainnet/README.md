# Camp Network — Mainnet Read Node

Reference configuration for running a read node against Camp mainnet (chain ID **484**, Celestia DA).

> **Note (2026-05-20)**: Camp mainnet migrated from Gelato-managed infrastructure to self-hosted infrastructure on 2026-05-20. The sequencer key was rotated as part of the handover. All values in this README reflect the post-cutover state. Old `*.raas.gelato.cloud` endpoints are no longer in use.

---

## Connection parameters

| Parameter | Value |
|-----------|-------|
| Chain ID | `484` |
| Genesis | [`./artifacts/genesis.json`](./artifacts/genesis.json) |
| Sequencer Ed25519 pubkey | `7fe1e241a9f41d99e0f3ec27d7319d892b6a7f071e570b92cb85b7e9bb20d75a` |
| Public RPC | `https://rpc-mainnet.campnetwork.xyz` |
| Public WS | `wss://rpc-mainnet.campnetwork.xyz` |
| Public Feed | `wss://feed.campnetwork.xyz` |
| Trusted peer (P2P bootstrap) | `enode://b48019af16ad05902a7ea70c75a996a5aa25862d28a2318037bdd7b4b41ffde7e065d8c289a4a73972efdef96d15d024b1e871ec7d0925f8bb9aa4798890bcbb@3.144.7.211:30303` |
| Status / metrics | `https://status.campnetwork.xyz` |
| Docker images | `ghcr.io/campaign-layer/{abc,xyz}:f6218b8` |

---

## Setup

### Pull images

```sh
docker pull ghcr.io/campaign-layer/abc:f6218b8
docker pull ghcr.io/campaign-layer/xyz:f6218b8
```

### Create a JWT secret

```sh
mkdir -p artifacts
openssl rand -hex 32 | tr -d '\n' > ./artifacts/jwt.hex
```

⚠️ **Important**: the JWT file must be **exactly 64 hex characters with no trailing newline**. Reth rejects whitespace and will fail with `malformed or out-of-range secret key`. The `tr -d '\n'` above handles this.

---

## Run XYZ Node (Execution Layer) — start first

```sh
mkdir -p .xyz

docker run -d \
  --name xyz \
  --network host \
  -v $(pwd)/.xyz:/data \
  -v $(pwd)/artifacts/jwt.hex:/jwt.hex \
  -v $(pwd)/artifacts/genesis.json:/genesis.json \
  ghcr.io/campaign-layer/xyz:f6218b8 node \
  --chain /genesis.json \
  --datadir /data \
  --http --http.api "eth,net,web3,debug,txpool" \
  --http.addr "0.0.0.0" --http.port "8449" \
  --ws --ws.addr "0.0.0.0" --ws.port "8450" \
  --authrpc.addr "0.0.0.0" --authrpc.port "8451" \
  --authrpc.jwtsecret /jwt.hex \
  --log.stdout.format=json --log.file.max-files=0 \
  --disable-discovery --trusted-only \
  --trusted-peers enode://b48019af16ad05902a7ea70c75a996a5aa25862d28a2318037bdd7b4b41ffde7e065d8c289a4a73972efdef96d15d024b1e871ec7d0925f8bb9aa4798890bcbb@3.144.7.211:30303 \
  --abc.sequencer-http https://rpc-mainnet.campnetwork.xyz
```

Wait until XYZ logs `RPC auth server started, url=0.0.0.0:8451` and reports `connected_peers: 1` before starting ABC.

---

## Run ABC Node (Consensus Layer) — start after XYZ is ready

```sh
mkdir -p .abc

docker run -d \
  --name abc \
  --network host \
  -v $(pwd)/.abc:/data \
  -v $(pwd)/artifacts/jwt.hex:/jwt.hex \
  ghcr.io/campaign-layer/abc:f6218b8 node \
  --datadir /data \
  --execution.jwtsecret /jwt.hex \
  --execution.endpoint "http://127.0.0.1:8451" \
  --http \
  --log.format json \
  --feed.ingress wss://feed.campnetwork.xyz \
  --feed.broadcast-channel-capacity 4096 \
  --sequencer.pubkey 7fe1e241a9f41d99e0f3ec27d7319d892b6a7f071e570b92cb85b7e9bb20d75a
```

Within ~30 seconds you should see `Fed block #N` lines in `docker logs -f abc`, and your XYZ's `latest_block` should start climbing toward the chain head.

---

## Verify it's working

```sh
# ABC healthcheck
curl -sf http://127.0.0.1:8080/healthz && echo HEALTHY

# Chain head
curl -s -X POST http://127.0.0.1:8449 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Compare against public RPC — should match within a few blocks
curl -s -X POST https://rpc-mainnet.campnetwork.xyz \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

---

## Notes

### Lazy block production

Camp mainnet runs the sequencer in lazy mode — empty blocks are skipped. During idle periods you may not see new blocks for several minutes. This is normal; it is not a stall.

You may see `WARN Beacon client online, but no consensus updates received for a while` in XYZ logs during long idle periods. This warning is harmless and clears the moment a new block flows through.

### Stale connections / silent feed disconnects

Long-lived WebSocket connections to the feed can be silently dropped by stateful firewalls between your node and our feed endpoint without either side emitting an error. If your ABC stops logging `Fed block #N` for more than ~10 minutes during active chain traffic, restart the container.

Suggested cron-based watchdog:

```sh
*/5 * * * * if ! docker logs abc --since 10m 2>&1 | grep -q "Fed block"; then docker restart abc; fi
```

### Data Availability (advanced — optional)

Read nodes in the default configuration sync exclusively from the feed and do not verify blocks against Celestia. If you want full DA verification, add `--da.mode celestia` and `--da.endpoint http://<your-celestia-light-node>:26658` to the ABC command, and provide a Celestia auth token via `--da.authtoken`.

Note: Camp's chain has two-key signing history (one Gelato key for blocks 1..27879876, the Camp key for 27879877+). ABC supports only one `--sequencer.pubkey` at a time, so a DA-verifying node cannot derive the full chain from Celestia starting from genesis. To bootstrap a fully-verifying node you must restore from a state snapshot. Contact the Camp team for snapshot access if you need this.

### Support

- Issues / questions: open an issue at https://github.com/campaign-layer/abc-images
- Status page: https://status.campnetwork.xyz
