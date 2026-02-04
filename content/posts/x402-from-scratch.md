+++
title = 'x402 for AI Agents: The No-SDK Guide'
date = 2026-02-03
draft = false
tags = ["x402", "crypto", "payments", "foundry"]
author = "Kamil Chmielewski"
description = "How to understand and test x402 payments using just curl and Foundry's cast. No SDK, no npm install, no dependencies."
+++

Here's what happened when I tried to figure out how x402 works:

1. Went to [x402.org](https://x402.org) - nice landing page, but no real explanation on how to implement this in a real app
2. Checked the Coinbase developer docs - mostly "npm install this" and "go get that"
3. Looked at the [coinbase/x402](https://github.com/coinbase/x402) repo - examples that just wrap the SDK

Every tutorial starts with "install our package". Cool, now I have hundreds of dependencies I don't understand, a bigger attack surface, and still no idea how the protocol actually works.

Here's the thing though - the actual implementation is maybe 200 lines of code. The protocol is simple. You don't need any of that garbage.

<!--more-->

## How I Got Here

I've been building [storagesloth.xyz](https://storagesloth.xyz) - a tool for web3 security researchers and developers to understand smart contracts through playful exploration. You deploy a project to anvil, interact with it, and see every single storage change your actions make with beautifully decoded values. Think of it as a local-first alternative to Tenderly.

I'm also building an API that agents can use to decode/encode calldata and simulate transactions with their storage slot changes before they hit mainnet.

As I was working on this, I started thinking about payment systems. The Alchemy model (pay as you go, credit card) seemed obvious, but then I started reading about compliance and Stripe integration. For micropayments like $0.001 per API call, the fees would eat more than the actual charge.

Then I remembered seeing [Austin Griffith](https://x.com/austingriffith) post about x402 on Twitter multiple times. I watched [the stream on x402, AI agents, and the ERC-8004 registry](https://x.com/i/broadcasts/1YpKkklrOLVKj) and decided to dig deeper to understand the components one by one.

(If you're wondering, [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) is a separate standard for AI agent identity and reputation on-chain - think of it as the "who is this agent and can I trust them?" layer that complements x402's "how do I pay them?" layer. We won't cover it here, but it's worth knowing about.)

The only thing that's actually useful in the coinbase repo (and for some reason isn't showcased anywhere with a big "HERE IS THE SPEC" sign) is [the specification itself](https://github.com/coinbase/x402/blob/29aa4135c4700a0a2bfdbc9a3b4348f5b6464ec3/specs/x402-specification-v2.md). If you don't have time or don't care, just feed this spec to Claude Code or any other agent and you'll get a complete implementation.

But if you want to actually understand what's going on, keep reading.

**The goal of this post** is to show you that this is not hard. It's just HTTP and JSON. That means you can implement this in literally any language if you know how to send JSON over the wire.

We're going to test the entire payment flow with just `curl` and Foundry's `cast`. No JavaScript. No SDKs. No web3 libraries.

One thing to note: if you want to **accept** payments (be a seller), it's simpler - you just need to make HTTP calls to the facilitator. If you want to **make** payments (be a buyer), you need to sign EIP-712 messages, which means you need something that can sign. **This tutorial focuses on the buyer/client side** - the harder part that involves cryptographic signing. We'll use `cast` for this. In a real application, you'd use something like go-ethereum or web3.js if you want your agent to sign programmatically. At the end, we'll briefly cover what the server side looks like.

**What this post is not**: a copy-paste implementation. We're walking through the payment mechanism step by step - how a client signs a payment, how to ask the facilitator to verify and settle it, and what the on-chain result looks like. The facilitator itself is a black box for our purposes. I link to the spec throughout so you understand where each piece comes from. Once you grok the flow, building your own implementation becomes straightforward.

## Why Should You Care?

The crypto space has been desperate for a use case beyond the degen casino. But the killer use case has been hiding in plain sight for a long time: sending tokens from Alice to Bob.

Quite useless on its own if you require someone to install MetaMask and either die from the input lag or get rekt on-chain.

Now x402 makes this concept interesting because you can hide the crypto aspect entirely from the end user. The person using your API doesn't need to know anything about wallets or signatures - they just make HTTP requests and pay in the background.

I'm not going to talk about the user-facing stuff though.

I'm here to tell you that during the gold rush, it's good to sell shovels. In the AI agent era, you can monetize your existing APIs or create cool APIs for AI agents to use. Most agents are silly, but some are quite interesting. By building useful tools (like [Trail of Bits' skills](https://github.com/trailofbits/skills)), you can make agents actually do useful leg work for you.

If you don't care about any of this, I think it's still valuable to understand the technology behind buzzwords so you can form your own opinion instead of trusting random people on Twitter.

## Prerequisites

You need [Foundry](https://book.getfoundry.sh/) installed. 

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

**For the love of god**, please open [https://foundry.paradigm.xyz](https://foundry.paradigm.xyz) and read the installation script first, or install from source. Do not execute random bash scripts from people on the internet.

Optionally, install `jq` if you want prettier JSON output.

## Step 1: Setup

### Create a Test Wallet

```bash
cast wallet new
```

This gives you an address and private key. **Do not send real funds to this address**. Use it for testnet only and discard it afterwards.

```
Address:     0x3c3e8D4EA77c3f750f3F22b2a2a0eB03d07D4F9f
Private key: 0xf512868a070ccdcb754a0edeba9b85e1744fa3994f63ba514bf06aff0eacb60e
```

### Get Testnet USDC

Go to the [Circle Faucet](https://faucet.circle.com/) and get some USDC on Base Sepolia. Or use any other USDC faucet that exists when you're reading this.

### Key Addresses

For the recipient address (who gets paid), you can pick any arbitrary address or generate a second wallet with `cast wallet new` and use it as recipient to transfer funds between the two.

| What | Value |
|------|-------|
| Your Wallet (payer) | `0x3c3e8D4EA77c3f750f3F22b2a2a0eB03d07D4F9f` |
| Recipient (who gets paid) | `0x81689a4381a5a055575E911fCF9FcA48dE7Fbb15` |
| USDC Contract | `0x036CbD53842c5426634e7929541eC2318f3dCF7e` |
| Network | `eip155:84532` (Base Sepolia in CAIP-2 format) |
| Facilitator | `https://x402.org/facilitator` |

**What's CAIP-2?** It's [Chain Agnostic Improvement Proposal 2](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md) - a standard way to identify blockchain networks. The format is `namespace:reference`. For EVM chains, it's `eip155:{chainId}`. So Base Sepolia (chainId 84532) becomes `eip155:84532`. Base Mainnet would be `eip155:8453`.

## Step 2: Understanding The Facilitator

The **facilitator** is a service that handles all the "difficult" crypto stuff that you might not want to implement yourself:

- **Signature verification** - checking that the payer actually signed the payment authorization
- **Balance checking** - making sure the payer has enough tokens
- **Transaction broadcasting** - actually submitting the transaction to the blockchain
- **Gas payment** - the facilitator pays the gas fees, not you

This introduces a trust assumption: you're trusting Coinbase's facilitator to correctly verify signatures and actually settle payments. In theory, you could run your own facilitator, but for most use cases the hosted one works fine.

Let's see what the facilitator supports:

```bash
curl -sL https://x402.org/facilitator/supported | jq .
```

```json
{
  "kinds": [
    {"x402Version": 2, "scheme": "exact", "network": "eip155:84532"},
    {"x402Version": 2, "scheme": "exact", "network": "solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1", "extra": {"feePayer": "..."}},
    {"x402Version": 1, "scheme": "exact", "network": "base-sepolia"},
    {"x402Version": 1, "scheme": "exact", "network": "solana-devnet", "extra": {"feePayer": "..."}}
  ],
  "extensions": [],
  "signers": {
    "eip155:*": ["0xd407e409E34E0b9afb99EcCeb609bDbcD5e7f1bf"],
    "solana:*": ["CKPKJWNdJEqa81x7CkZ14BVPiY6y16Sxs7owznqtWYp5"]
  }
}
```

The response includes both v1 and v2 protocol versions. We'll use v2 for this tutorial. The `extensions` field is for future protocol extensions.

The `signers` field shows who pays gas. The facilitator's signer (`0xd407...`) will broadcast your transaction and pay the gas fees. You only need USDC, not ETH.

The `"exact"` scheme uses [EIP-3009](https://eips.ethereum.org/EIPS/eip-3009) - a standard that lets you authorize token transfers with just a signature. No on-chain approval transaction needed.

EIP-3009 is what Permit2 would like to be but isn't, because Permit2 has to support all tokens. EIP-3009 is a special token interface that USDC (and some other tokens) implement natively. It's simpler and more efficient for tokens that support it.

## Step 3: What Gets Signed?

The [x402 specification](https://github.com/coinbase/x402/blob/29aa4135c4700a0a2bfdbc9a3b4348f5b6464ec3/specs/x402-specification-v2.md#61-exact-scheme-evm-overview) tells us we need to create an EIP-3009 `TransferWithAuthorization` signature. This is [EIP-712 typed data](https://eips.ethereum.org/EIPS/eip-712) with a specific structure.

EIP-712 signatures have two main components:

1. **Domain separator** - identifies the contract you're interacting with (prevents replay across different contracts/chains)
2. **Message** - the actual data you're signing

For EIP-3009, the message structure is defined in [the EIP itself](https://eips.ethereum.org/EIPS/eip-3009#specification):

```txt
TransferWithAuthorization(
  address from,
  address to,
  uint256 value,
  uint256 validAfter,
  uint256 validBefore,
  bytes32 nonce
)
```

**About the nonce**: Unlike regular transaction nonces which are sequential, EIP-3009 uses random 32-byte nonces. [The EIP explains why](https://eips.ethereum.org/EIPS/eip-3009#unique-random-nonce-instead-of-sequential-nonce): regular transactions with too-high nonces wait in the mempool, but meta-transactions would just fail immediately. Random nonces let you create multiple authorizations without worrying about ordering.

### Get The Domain Info

The EIP-712 domain comes from the USDC contract itself. We need the token's name and version:

```bash
# Token name
cast call 0x036CbD53842c5426634e7929541eC2318f3dCF7e \
  "name()(string)" --rpc-url https://sepolia.base.org
# "USDC"

# Token version
cast call 0x036CbD53842c5426634e7929541eC2318f3dCF7e \
  "version()(string)" --rpc-url https://sepolia.base.org
# "2"
```

The domain combines:
- `name`: "USDC" (from the contract)
- `version`: "2" (from the contract)
- `chainId`: 84532 (Base Sepolia)
- `verifyingContract`: `0x036CbD53842c5426634e7929541eC2318f3dCF7e` (the USDC contract address - this is what we're authorizing to transfer our tokens)

## Step 4: Create The Typed Data

Now we combine the EIP-712 domain with the EIP-3009 message structure into a single JSON file that `cast` can sign.

The structure follows the [EIP-712 JSON schema](https://eips.ethereum.org/EIPS/eip-712#definition-of-typed-structured-data-%F0%9D%95%8A):
- `types`: Defines the structure of our domain and message
- `primaryType`: What we're signing ("TransferWithAuthorization")
- `domain`: The EIP-712 domain separator values
- `message`: The actual EIP-3009 authorization parameters

```bash
# Generate a random nonce (must be unique per authorization)
NONCE=$(cast keccak "$(date +%s%N)-random")

# Set expiry to 1 hour from now
VALID_BEFORE=$(($(date +%s) + 3600))

# Your addresses (replace with yours)
PAYER="0x3c3e8D4EA77c3f750f3F22b2a2a0eB03d07D4F9f"
PAYTO="0x81689a4381a5a055575E911fCF9FcA48dE7Fbb15"
USDC="0x036CbD53842c5426634e7929541eC2318f3dCF7e"
AMOUNT="1000"  # 0.001 USDC (6 decimals)

cat > /tmp/payment.json << EOF
{
  "types": {
    "EIP712Domain": [
      {"name": "name", "type": "string"},
      {"name": "version", "type": "string"},
      {"name": "chainId", "type": "uint256"},
      {"name": "verifyingContract", "type": "address"}
    ],
    "TransferWithAuthorization": [
      {"name": "from", "type": "address"},
      {"name": "to", "type": "address"},
      {"name": "value", "type": "uint256"},
      {"name": "validAfter", "type": "uint256"},
      {"name": "validBefore", "type": "uint256"},
      {"name": "nonce", "type": "bytes32"}
    ]
  },
  "primaryType": "TransferWithAuthorization",
  "domain": {
    "name": "USDC",
    "version": "2",
    "chainId": 84532,
    "verifyingContract": "$USDC"
  },
  "message": {
    "from": "$PAYER",
    "to": "$PAYTO",
    "value": "$AMOUNT",
    "validAfter": "0",
    "validBefore": "$VALID_BEFORE",
    "nonce": "$NONCE"
  }
}
EOF
```

## Step 5: Sign It

```bash
PRIVATE_KEY="0xf512868a070ccdcb754a0edeba9b85e1744fa3994f63ba514bf06aff0eacb60e"

SIGNATURE=$(cast wallet sign --private-key $PRIVATE_KEY --data --from-file /tmp/payment.json)
echo $SIGNATURE
```

That's it. No ethers.js, no web3.py, no SDK. Just `cast`.

**A big warning about AI agents**: Austin mentions on [the stream](https://x.com/i/broadcasts/1YpKkklrOLVKj) that using private keys like this with AI agents is insecure. As of my knowledge in February 2026, secure agentic wallets are not a solved problem yet. If you know of good solutions, please let me know.

The OpenClaw bot (formerly Clawdbot, Moltbot) loves loading private keys into its memory immediately, even when explicitly asked not to under threats. It just yolos and doesn't care. Anything that gets loaded into an agent's memory should be treated as compromised - it gets sent to your inference provider.

For testing with testnet funds, this is fine. For real money, don't do this until better solutions exist.

## Step 6: Verify The Payment

Now we send this to the facilitator to check if the signature is valid. The request format is defined in [section 7.1 of the spec](https://github.com/coinbase/x402/blob/29aa4135c4700a0a2bfdbc9a3b4348f5b6464ec3/specs/x402-specification-v2.md#71-post-verify).

The verify request needs two things:
- `paymentPayload`: The signed authorization (what the client sends)
- `paymentRequirements`: What the server requires for payment

Let me explain some of the fields in `paymentRequirements`:
- `scheme`: Payment method - "exact" means exact amount, no overpayment
- `maxTimeoutSeconds`: How long the payment authorization is valid
- `extra`: Scheme-specific data. For EVM "exact" scheme, this **must** include `name` and `version` of the token for EIP-712 domain reconstruction. This was a gotcha I discovered - without it you get cryptic errors.

```bash
curl -sL -X POST https://x402.org/facilitator/verify \
  -H "Content-Type: application/json" \
  -d '{
    "paymentPayload": {
      "x402Version": 2,
      "accepted": {
        "scheme": "exact",
        "network": "eip155:84532",
        "amount": "'"$AMOUNT"'",
        "asset": "'"$USDC"'",
        "payTo": "'"$PAYTO"'",
        "maxTimeoutSeconds": 300,
        "extra": {"name": "USDC", "version": "2"}
      },
      "payload": {
        "signature": "'"$SIGNATURE"'",
        "authorization": {
          "from": "'"$PAYER"'",
          "to": "'"$PAYTO"'",
          "value": "'"$AMOUNT"'",
          "validAfter": "0",
          "validBefore": "'"$VALID_BEFORE"'",
          "nonce": "'"$NONCE"'"
        }
      }
    },
    "paymentRequirements": {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "'"$AMOUNT"'",
      "asset": "'"$USDC"'",
      "payTo": "'"$PAYTO"'",
      "maxTimeoutSeconds": 300,
      "extra": {"name": "USDC", "version": "2"}
    }
  }' | jq .
```

If everything is correct, you'll get:

```json
{"isValid": true, "payer": "0x3c3e8D4EA77c3f750f3F22b2a2a0eB03d07D4F9f"}
```

**Important**: No transaction has happened yet. The facilitator just validated that:
1. The signature is correct
2. The payer has enough balance
3. The authorization parameters are valid

This is where you have a design decision in your application. You could:

1. **Optimistically fulfill** - Give the user access to your API/content right after verify succeeds, then settle in the background. Faster UX, but you're trusting that settlement will work.

2. **Wait for settlement** - Only fulfill after the on-chain transaction confirms. Slower, but guaranteed payment.

For most APIs, optimistic fulfillment is probably fine. For high-value operations, wait for settlement.

## Step 7: Settle (Execute On-Chain)

Same payload, but POST to `/settle` ([spec section 7.2](https://github.com/coinbase/x402/blob/29aa4135c4700a0a2bfdbc9a3b4348f5b6464ec3/specs/x402-specification-v2.md#72-post-settle)):

```bash
curl -sL -X POST https://x402.org/facilitator/settle \
  -H "Content-Type: application/json" \
  -d '{...same payload as verify...}' | jq .
```

```json
{
  "success": true,
  "transaction": "0x4e262d169e0c2dbd4beece1ef9ab3e9d07097efeed30e5f5769ffef28886bfd0",
  "network": "eip155:84532",
  "payer": "0x3c3e8D4EA77c3f750f3F22b2a2a0eB03d07D4F9f"
}
```

**Now** you have your money. The transaction was broadcast and confirmed. You'll get back a transaction hash that you can look up on [BaseScan Sepolia](https://sepolia.basescan.org/).

### Verify On-Chain

```bash
# Payer balance decreased
cast call $USDC "balanceOf(address)(uint256)" $PAYER --rpc-url https://sepolia.base.org
# 19999000 (was 20000000)

# Recipient received the payment
cast call $USDC "balanceOf(address)(uint256)" $PAYTO --rpc-url https://sepolia.base.org
# 1000
```

You can view the transaction on [BaseScan Sepolia](https://sepolia.basescan.org/tx/0x4e262d169e0c2dbd4beece1ef9ab3e9d07097efeed30e5f5769ffef28886bfd0).

## Key Gotchas I Discovered

1. **The `extra` field is critical** - Without `{"name": "USDC", "version": "2"}` you get `missing_eip712_domain`. The facilitator uses this to reconstruct the EIP-712 domain for signature verification.

2. **Nonce is random bytes, not sequential** - Generate 32 random bytes for each authorization. Don't use your account nonce.

3. **Amount is in base units** - USDC has 6 decimals, so `1000` = $0.001. Confusingly, some of Coinbase's own examples use `"$0.001"` as the amount string directly without converting to base units. This might be a v1 spec relic - stick with base units for v2.

4. **`accepted` must match `paymentRequirements`** - The server compares these to ensure you're paying for what was requested.

## What's Next?

We've covered the client side - how to sign and submit payments. But what about the server side?

If you want to **accept** x402 payments (which is easier than making them), the flow looks like this:

1. Client requests your protected resource without payment
2. You return HTTP 402 with a `PAYMENT-REQUIRED` header containing base64-encoded JSON - this includes the `paymentRequirements` structure we used in Step 6
3. Client signs the payment (Steps 4-5) and retries with a `PAYMENT-SIGNATURE` header containing the base64-encoded `PaymentPayload` - this is the same structure we sent in the `/verify` request body
4. You POST to the facilitator's `/verify` and `/settle` endpoints - exactly what we did in Steps 6 and 7
5. You return the resource along with a `PAYMENT-RESPONSE` header - this contains the same settlement response we got back in Step 7

The header names and exact formats are defined in [the HTTP transport spec](https://github.com/coinbase/x402/blob/main/specs/transports-v2/http.md). The whole server-side implementation is maybe 200 lines of code in any language - it's just HTTP middleware that checks headers and makes a couple of POST requests.

I'm currently implementing this for [storagesloth.xyz](https://storagesloth.xyz). The API will let agents decode/encode calldata and simulate transactions with full storage slot analysis - essentially giving your agent the ability to understand what a transaction will do before executing it.

Once the API is live, I'll either update this post with a working example or write a follow-up showing the server-side implementation in detail.

## References

- [x402 Specification v2](https://github.com/coinbase/x402/blob/29aa4135c4700a0a2bfdbc9a3b4348f5b6464ec3/specs/x402-specification-v2.md) - The actual spec
- [EIP-3009: Transfer With Authorization](https://eips.ethereum.org/EIPS/eip-3009) - How the signatures work
- [EIP-712: Typed Structured Data Signing](https://eips.ethereum.org/EIPS/eip-712) - How typed data signing works
- [Foundry Book](https://book.getfoundry.sh/) - The `cast` command reference
- [x402 Stream](https://x.com/i/broadcasts/1YpKkklrOLVKj) - The stream that got me started
- [Trail of Bits Skills](https://github.com/trailofbits/skills) - Example of useful AI agent tools

