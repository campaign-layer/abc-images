# ABC — Abundance Sovereign Stack

ABC Stack follows the same design and architecture as Ethereum, using the Engine API. Each ABC node requires the deployment of two components:

- **ABC (Consensus)** — Receives new blocks and verifies validity
- **XYZ (Execution)** — Executes EVM transactions

This repository contains the genesis files and reference configurations for chains running the ABC Stack.

| Network | Folder |
|---------|--------|
| Camp mainnet (chain ID 484) | [`camp/mainnet`](./camp/mainnet) |
| Camp testnet | [`camp/testnet`](./camp/testnet) |

---

## Prerequisites

- Install [Docker](https://docs.docker.com/get-docker/)

## Pull the images

Camp publishes the ABC and XYZ images on GitHub Container Registry under the `campaign-layer` org:

```bash
docker pull ghcr.io/campaign-layer/abc:f6218b8
docker pull ghcr.io/campaign-layer/xyz:f6218b8
```

`:f6218b8` is the current pinned tag — recommended for stability. `:latest` also points to the same image.

---

## Running with Docker (generic template)

For the exact post-cutover Camp mainnet configuration (sequencer pubkey, peer enode, feed URL, sequencer RPC), see [`camp/mainnet/README.md`](./camp/mainnet/README.md). The snippets below are a generic template — replace `${VARS}` with values from your chain's folder.

### Running ABC Node (Consensus / read node CL)

```sh
mkdir -p .tmp/abc

docker run -d \
  --name abc \
  --network host \
  -v $(pwd)/.tmp/abc:/data \
  -v $(pwd)/artifacts/jwt.hex:/jwt.hex \
  ghcr.io/campaign-layer/abc:f6218b8 node \
  --datadir /data \
  --execution.jwtsecret /jwt.hex \
  --execution.endpoint "http://127.0.0.1:8451" \
  --http \
  --log.format json \
  --feed.ingress ${FEED_URL} \
  --sequencer.pubkey ${SEQUENCER_PUBLIC_KEY}
```

### Running XYZ Node (Execution / read node EL)

```sh
mkdir -p .tmp/xyz

docker run -d \
  --name xyz \
  --network host \
  -v $(pwd)/.tmp/xyz:/data \
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
  --trusted-peers ${SEQUENCER_ENODE} \
  --abc.sequencer-http ${SEQUENCER_RPC}
```

All values marked `${...}` must be replaced from your network's mainnet/testnet README.
