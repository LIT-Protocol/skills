---
name: lit-protocol-flows
description: Publish and consume paid JavaScript functions on Lit Protocol's TEE. Use when building, publishing, invoking, or paying for Flows — Lit's agent-to-agent commerce platform. Covers CLI usage, MCP setup, and payment via credits, x402, or MPP.
---

# Flows by Lit Protocol

Publish JavaScript functions as paid API endpoints running in Lit Protocol's TEE. One command to publish, automatic MCP server, built-in payments.

## Publish a flow

Write a `.js` file — your code runs inside `async function main(params) { ... }`:

```js
// hello.js
const name = params.name || 'world';
return { message: `Hello, ${name}!` };
```

```bash
npx -y @lit-protocol/flows publish hello.js --name "Hello World" --price 1
```

Output includes the invoke URL, MCP URL, and public page.

## Invoke a flow

```bash
# CLI
npx -y @lit-protocol/flows invoke hello-world --params '{"name": "Agent"}'

# HTTP
curl -X POST https://flows.litprotocol.com/api/flows/hello-world/invoke \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"params": {"name": "Agent"}}'
```

## MCP

Every flow is an MCP server at `https://flows.litprotocol.com/mcp/<slug>`.

```bash
claude mcp add oracle --transport http https://flows.litprotocol.com/mcp/oracle
```

For config-file clients (Claude Desktop, Cursor, VS Code, Windsurf), add to MCP config with `Authorization: Bearer YOUR_API_KEY` header.

## CLI commands

All via `npx -y @lit-protocol/flows <command>`:

| Command | Description |
|---------|-------------|
| `login` | Authenticate (`--key KEY` for CI) |
| `publish <file>` | Publish (`--name`, `--price`, `--description`, `--connections`, `--update`) |
| `invoke <slug>` | Call a flow (`--params '{...}'`) |
| `list` | List your flows |
| `logs <slug>` | View execution logs |
| `secrets set <slug> <KEY> <val>` | Set encrypted secret (TEE-only access) |
| `secrets list <slug>` | List secret names |
| `secrets delete <slug> <KEY>` | Delete a secret |
| `connect <app>` | OAuth connect (gmail, slack, etc.) |

## Payment methods

| Method | Setup | Account needed |
|--------|-------|----------------|
| **Credits** | Sign up, get API key, load via Stripe. Pass `Authorization: Bearer KEY`. Refunded on failure. $1 free on signup. | Yes |
| **x402** | Call without API key — get 402 with payment details. x402 client pays with USDC on Base automatically. [x402.org](https://x402.org) | No |
| **MPP** | First request returns payment challenge. Complete via Stripe or Tempo, retry with credential. [docs.mppx.dev](https://docs.mppx.dev) | No |

## TEE globals

Code runs in Lit's TEE with: `params` (caller input), `params.secrets` (encrypted secrets), `params.connections` (OAuth tokens), `params.pkpAddress` (vault PKP), `Lit.Actions` (signing/decryption), `ethers`, `fetch`.

## Example: Signing oracle

```js
const res = await fetch(params.url);
const body = await res.text();
const hash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(body));
const pk = await Lit.Actions.getPrivateKey({ pkpId: params.pkpAddress });
const wallet = new ethers.Wallet(pk);
const sig = await wallet.signMessage(ethers.utils.arrayify(hash));
return { url: params.url, response: body, dataHash: hash, signature: sig, signer: wallet.address };
```

```bash
npx -y @lit-protocol/flows publish oracle.js --name "Signed Oracle" --price 5
```

## Links

- Platform: https://flows.litprotocol.com
- Explore flows: https://flows.litprotocol.com/explore
- Docs: https://flows.litprotocol.com/docs
