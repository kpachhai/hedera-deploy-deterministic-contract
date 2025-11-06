# Hedera Deterministic Contract Deployment

This repository demonstrates how to deterministically deploy an EVM contract (e.g., Multicall3) on Hedera via a pre-signed Ethereum transaction, with the Hedera SDK handling:

- Hollow account creation/funding for the ECDSA alias if needed
- Submitting the pre-signed Ethereum transaction through HAPI
- Computing the deployed contract address deterministically from `from` and `nonce`

The deployment flow:

1. Parse the `SIGNED_TX` to derive the signer’s EVM address (`from`).
2. Ensure the hollow account for that alias exists and has at least 1 ℏ (creates and funds if needed).
3. Query the signer’s ethereumNonce BEFORE submitting, to compute the deterministic address for the CREATE.
4. Submit the pre-signed Ethereum transaction.
5. Print the deployed contract address, transaction ID, and explorer link.

## Prerequisites

- Node.js 18+ (LTS recommended)
- A Hedera account (operator) to pay fees and fund the alias
- An ECDSA private key for your operator account in `.env`
- A pre-signed Ethereum transaction hex (`SIGNED_TX`) that performs a contract creation (CREATE)

## Install

```bash
# Clone your repo, then:
npm install
```

This installs:

- @hashgraph/sdk
- ethers
- dotenv

## Configure environment

Create a `.env` file in the project root:

```bash
cp .env.example .env  # if you keep an example, otherwise just create .env
```

Minimum required values:

```bash
# Hedera operator paying fees and providing the top-up HBAR
OPERATOR_ID=0.0.xxxxx
OPERATOR_KEY=302e020100300506032b657004220420...   # ECDSA (preferred) or ED25519

# Optional network selection: mainnet | testnet | previewnet, default is testnet
HEDERA_NETWORK=testnet

# Allowance used by the Hedera relay to cover gas differences if needed
MAX_GAS_ALLOWANCE_HBAR=10

# The raw, pre-signed Ethereum transaction hex that deploys your contract (0x...)
SIGNED_TX=0xf9...
```

Notes:

- The script derives the signer (`from`) directly from `SIGNED_TX`.
- The script ensures the alias account exists and has at least 1 ℏ before submitting.
- If you don’t want to store `SIGNED_TX` in `.env`, you can paste it directly into `deploy.js`.

## Run the deployment

```bash
# Option A: via npm script
npm run deploy

# Option B: directly
node deploy.js
```

On success, you will see:

- Deterministic contract address
- Transaction ID
- Explorer link

## How it works (high level)

- The “hollow account” is auto-created the first time HBAR is sent to an ECDSA alias address on Hedera.
- The script funds the alias with at least 1 ℏ so the relay can charge gas from the transaction sender if gas terms are acceptable.
- If the `SIGNED_TX` gas price/fees don’t align with the network, the relay may cover the difference up to `MAX_GAS_ALLOWANCE_HBAR`.
- The contract address for a CREATE transaction is deterministic: `keccak256(rlp([from, nonce]))[12:]`, where `nonce` is the sender’s ethereumNonce BEFORE the create. The script fetches this and prints the computed address.

## Using this for other deterministic deployments

You can reuse the same `deploy.js` to deploy any contract deterministically:

1. Produce a pre-signed Ethereum “contract creation” transaction for your desired bytecode. The transaction must:

   - Have `to = null` (contract creation)
   - `value = 0` (typically)
   - `data = 0x<creationBytecode>` (your contract’s deploy/creation bytecode)
   - Be signed by the ECDSA key that corresponds to the alias you want to deploy from

2. Put that raw signed transaction hex into `SIGNED_TX` in your `.env` (or directly in `deploy.js`).

3. Run `npm run deploy` (or `node deploy.js`).

Because the address is `CREATE(from, nonce)`, each subsequent successful create increments `nonce` by 1, changing the next address deterministically. If you need to predict the next address:

- The script prints the current `ethereumNonce` before deployment.
- The deployed address corresponds to that `nonce`.

### Example: generating a signed tx with ethers v6

Below is an outline you can adapt to your offline signer flow. Make sure `chainId` matches your target Hedera network:

- mainnet: 295
- testnet: 296
- previewnet: 297

```ts
import { Wallet, parseUnits } from "ethers";

// ECDSA key for the desired alias (NOT necessarily your Hedera operator)
const wallet = new Wallet("0x<your-ecdsa-private-key>");

const chainId = 296; // testnet (295 mainnet, 296 testnet, 297 previewnet)

const unsigned = {
  type: 2, // EIP-1559
  chainId,
  to: null, // contract creation
  data: "0x<creationBytecode>", // your contract creation bytecode
  value: 0,
  nonce: await wallet.getNonce(), // or set explicitly if you track it
  gasLimit: 3_000_000, // example
  maxFeePerGas: parseUnits("100", "gwei"),
  maxPriorityFeePerGas: parseUnits("2", "gwei")
};

// Sign it (offline ideally)
const rawSigned = await wallet.signTransaction(unsigned);
// Put rawSigned into SIGNED_TX in your .env
console.log(rawSigned);
```

Important:

- The Hedera relay validates gas parameters. If they’re off, your `MAX_GAS_ALLOWANCE_HBAR` allows the relay to cover differences up to your limit.
- The script will fund the alias with at least 1 ℏ if needed; increase that if your use case requires more.

## Troubleshooting

- If you see errors related to account info queries, ensure you’re using `AccountInfoQuery` (the script is already set up to use it).
- Mirror node lag can delay `ethereumNonce` updates immediately after deployment; the script logs pre- and post- values when available.
- Make sure your operator key and network are correct. Using an Ed25519 operator is supported for paying fees; the EVM alias used by the signed tx must be ECDSA.

## License

MIT
