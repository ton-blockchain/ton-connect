# Wallet guide — Structured items

Structured items add an `items` field as an alternative to `messages` in `sendTransaction` and `signMessage`. Instead of dApps building raw BoCs for jetton and NFT transfers, they describe **what** the user wants to do, and the **wallet** constructs the BoC.

A request payload MUST contain either `messages` or `items`, never both. Three item types are defined: `ton`, `jetton`, `nft`.

## Payload format

See [`spec/rpc.md` § Structured `items`](../spec/rpc.md#structured-items) for the field definitions of `TonItem`, `JettonItem` and `NftItem`.

## Implementation

### Step 1 — declare the feature

Add `itemTypes` to the `SendTransaction` feature (and `SignMessage` if supported) listing only the kinds the wallet handles: `ton`, `jetton`, `nft`. If `itemTypes` is absent, the SDK treats the wallet as supporting only raw `messages`.

### Step 2 — parse the request

Read `messages` and `items` from the transaction payload. Each entry of `items` carries `type` as the discriminator.

### Step 3 — validate the payload

Reject with code `1` (`BAD_REQUEST`) when both `messages` and `items` are present, when neither is present or when an item's `type` is unknown. Reject with code `1` for asset-level failures (insufficient jetton balance, NFT not owned by the connected account) before signing.

### Step 4 — build BoCs per item type

The connected account is the sender for every item.

- `ton` — parse `address`, treat `amount` as nanograms, attach `payload` and `stateInit` if present, use the bounce flag from the friendly-format address.
- `jetton` — resolve the sender's jetton-wallet from `master` and the sender address; build the transfer body with opcode `0x0f8a7ea5` per [TEP-74](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md); encode `forwardPayload` as `Either Cell ^Cell`; choose attached TON from `attachAmount` or wallet estimation; bounce enabled.
- `nft` — build the transfer body with opcode `0x5fcc3d14` per [TEP-62](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md); encode `forwardPayload` with the same `Either Cell ^Cell` rule; choose attached TON from `attachAmount` or wallet estimation; bounce enabled.

### Step 5 — dispatch into the signing path

Route built messages back into the wallet's `sendTransaction` (or `signMessage`) signing flow as if they were raw `messages`. Show a per-item preview before signing — jetton symbol, amount, destination for jettons; NFT name, image, collection, new owner for NFTs.

## Error paths

| Condition                              | What to return                             |
|----------------------------------------|--------------------------------------------|
| Both `messages` and `items` present    | `1` Bad request                            |
| Unknown item `type`                    | `1` Bad request                            |
| Insufficient jetton balance            | `1` Bad request, with message; do not sign |
| NFT not owned by the connected account | `1` Bad request, with message; do not sign |
| User rejects                           | `300` User declined the request            |

## Backward compatibility

Existing dApps using `messages` continue to work unchanged. The `itemTypes` feature flag is optional — if absent, the SDK never sends `items` to the wallet.

## See also

- [`spec/rpc.md` § Structured `items`](../spec/rpc.md#structured-items) — canonical format
- [`spec/rpc.md` § `sendTransaction`](../spec/rpc.md#sendtransaction) — calling method
- [TEP-74 — Jettons standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md)
- [TEP-62 — NFT standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md)
