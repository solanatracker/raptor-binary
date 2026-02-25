<div align="center">


# Raptor

**DEX Aggregator and Swap API for Solana**

[![Public Beta](https://img.shields.io/badge/Status-Public%20Beta-blueviolet?style=flat-square)](https://github.com/solanatracker/raptor-binary)
[![Solana](https://img.shields.io/badge/Solana-Mainnet-14F195?style=flat-square&logo=solana)](https://solana.com)
[![Discord](https://img.shields.io/badge/Discord-Join-5865F2?style=flat-square&logo=discord&logoColor=white)](https://discord.gg/Kj3CR65KpR)
![Raptor](https://i.imgur.com/mlKeoVW.jpeg)

[Getting Started](#getting-started) · [Documentation](https://docs.solanatracker.io/raptor/overview) · [Configuration](#configuration) · [Discord](https://discord.gg/Gfnwee4T6S)

</div>

---

Raptor is a high-performance DEX aggregator for Solana. It finds optimal swap routes across 20+ liquidity sources, supports multi-hop routing up to 4 hops, and includes real-time WebSocket streaming and transaction submission via Yellowstone Jet TPU.

```
Program ID (Mainnet): RaptorD5ojtsqDDtJeRsunPLg6GvLYNnwKJWxYE4m87
```

If you find Raptor useful, consider starring this repo.

---

## Features

- Fully self-hosted, no API key required
- Minimal RPC usage—pool state streams via Yellowstone gRPC
- Multi-hop routing (up to 4 hops) with sub-millisecond quote computation
- Real-time WebSocket streaming with slot-based quote updates
- Transaction submission via [Yellowstone Jet TPU](https://github.com/rpcpool/yellowstone-jet) with automatic retries
- Dynamic slippage based on volatility and route complexity
- Platform fee support (configurable, taken from input or output)
- DEX and pool filtering per request
- Arbitrage support

---

## Supported DEXs

**Raydium** — AMM, CLMM, CPMM, LaunchLab

**Meteora** — DLMM, Dynamic AMM, DAMM V2, Curve, DBC

**Orca** — Whirlpool (Legacy), Whirlpool V2

**Bonding Curves** — Pump.fun, Pumpswap, Heaven, MoonIt, Boopfun

**PropAMM** — Humidifi, Tessera, Solfi V1/V2

**Other** — FluxBeam, PancakeSwap V3

---

## Getting Started

Raptor is fully self-hosted. No API key required—just download the binary and run it on your own infrastructure.

**Requirements:**
- Solana RPC endpoint
- Yellowstone gRPC endpoint (required for pool indexing)

Raptor uses very few RPC calls during normal operation since pool state is streamed via Yellowstone.

```bash
# Download the latest release
curl -L https://github.com/solanatracker/raptor/releases/latest/download/raptor-linux-amd64 -o raptor
chmod +x raptor

# Set environment variables
export RPC_URL="https://your-rpc-endpoint.com"
export YELLOWSTONE_ENDPOINT="https://your-yellowstone-endpoint.com"
export YELLOWSTONE_TOKEN="your-token"  # if required by your provider

# Run
./raptor
```

Server starts at `http://0.0.0.0:8080` by default.

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/quote` | Get swap quote with routing |
| `POST` | `/swap` | Build swap transaction from quote |
| `POST` | `/swap-instructions` | Get swap instructions only |
| `POST` | `/quote-and-swap` | Quote + transaction in one request |
| `POST` | `/send-transaction` | Submit via [Yellowstone Jet TPU](https://github.com/rpcpool/yellowstone-jet) |
| `GET` | `/transaction/:signature` | Track transaction status |
| `GET` | `/health` | Health check |

### Get a Quote

```bash
curl "http://localhost:8080/quote?\
inputMint=So11111111111111111111111111111111111111112&\
outputMint=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v&\
amount=1000000000&\
slippageBps=50"
```

### Build and Send a Swap

```javascript
// 1. Get quote
const quote = await fetch('/quote?inputMint=...&outputMint=...&amount=...')
  .then(r => r.json());

// 2. Build transaction
const { swapTransaction } = await fetch('/swap', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    quote,
    userPublicKey: wallet.publicKey.toString(),
    wrapUnwrapSol: true,
    priorityFee: 'auto'
  })
}).then(r => r.json());

// 3. Sign and send
const tx = VersionedTransaction.deserialize(Buffer.from(swapTransaction, 'base64'));
tx.sign([wallet]);

const { signature } = await fetch('/send-transaction', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ 
    transaction: Buffer.from(tx.serialize()).toString('base64') 
  })
}).then(r => r.json());
```

---

## WebSocket Streaming

Connect to `/stream` for real-time quotes that update on pool state changes.

```javascript
const ws = new WebSocket('wss://raptor-beta.solanatracker.io/stream');

ws.send(JSON.stringify({
  type: 'subscribe',
  inputMint: 'So11111111111111111111111111111111111111112',
  outputMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
  amount: 1000000000,
  slippageBps: '50'
}));

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === 'quote') {
    console.log(`${msg.data.amountOut} (slot: ${msg.data.contextSlot})`);
  }
};
```

---

## Configuration

### CLI Flags

```
-h, --help                   Show help
-v, --version                Show version

--no-pool-indexer            Disable loading pools from Oracle
--include-dexes <DEXES>      Only include these DEXes (comma-separated)
--exclude-dexes <DEXES>      Exclude these DEXes (comma-separated)
--workers <N>                Worker threads (default: CPU cores)
--rpc-rate-limit <N>         Max RPC calls per second limit

--enable-arbitrage           Enable circular arbitrage routes
--enable-quote-and-swap      Enable /quote-and-swap endpoint
--enable-websocket           Enable WebSocket at /stream
--enable-yellowstone-jet     Enable Jet TPU for /send-transaction

--jet-identity <PATH>        Identity keypair path (optional)
--jet-identities <N>         Number of identities (default: 4)
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `RPC_URL` | Solana RPC endpoint | required |
| `YELLOWSTONE_ENDPOINT` | Yellowstone gRPC endpoint | required |
| `YELLOWSTONE_TOKEN` | Yellowstone auth token | — |
| `BIND_ADDR` | Server bind address | `0.0.0.0:8080` |
| `WORKER_THREADS` | Worker threads | CPU cores |
| `INCLUDE_DEXES` | DEXes to include | all |
| `EXCLUDE_DEXES` | DEXes to exclude | none |
| `ENABLE_WEBSOCKET` | Enable WebSocket | `false` |
| `ENABLE_YELLOWSTONE_JET` | Enable Jet TPU | `false` |

All CLI flags have corresponding environment variables. See the overview docs for the full mapping.

---

## Priority Fees

| Level | Use Case |
|-------|----------|
| `min` / `low` | Cost-saving |
| `auto` / `medium` | Recommended |
| `high` / `veryHigh` | Faster confirmation |
| `turbo` / `unsafeMax` | Maximum speed |

---

## Transaction Tracking

Transactions sent via `/send-transaction` can be tracked:

```javascript
const tx = await fetch(`/transaction/${signature}`).then(r => r.json());

// Status: pending | confirmed | failed | expired
console.log(tx.status, tx.latency_ms);

for (const event of tx.events ?? []) {
  if (event.name === 'SwapEvent') {
    console.log(event.parsed.amountIn, '→', event.parsed.amountOut);
  }
}
```

---

## Frequently Asked Questions
Is this an alternative to the Metis Binary? Yes this can be seen as an alternative to the Metis Binary. 
Raptor does not require a binary key, has no limits and can be resold. 

## Issues and Feedback

This is a closed-source project. The repository hosts binary releases only.

For bug reports, feature requests, or suggestions, please [open an issue](https://github.com/solanatracker/raptor-binary/issues).

Join the [Discord](https://discord.gg/Kj3CR65KpR) to chat with the team and community.

---

<div align="center">

**Public Beta — API and binary are now available **

Built by [Solana Tracker](https://www.solanatracker.io)

</div>
