# RPC methods and events

This document is normative for dApps, wallets, bridge operators, SDKs. The keywords MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

This document specifies the four RPC methods (`sendTransaction`, `signMessage`, `signData`, `disconnect`) and the wallet-initiated events (`connect`, `connect_error`, `disconnect`). The connect handshake itself is described in [`connect.md`](./connect.md). The machine-readable envelope shapes are published as [`schemas/rpc.schema.json`](../schemas/rpc.schema.json).

## Envelope

```typescript
type AppMessage = ConnectRequest | AppRequest;
type WalletMessage = WalletResponse | WalletEvent;
```

### `AppRequest`

```typescript
interface AppRequest {
    method: string;     // 'sendTransaction', 'signMessage', 'signData', 'disconnect'
    params: string[];   // operation-specific parameters
    id: string;         // monotonically increasing per session
}
```

The wallet MUST accept the first `id` it sees in a session as the baseline. For every subsequent request, `id` MUST be strictly greater than the last `id` the wallet has processed in the same session; the wallet MUST reject any request that violates this.

### `WalletResponse`

```typescript
type WalletResponse = WalletResponseSuccess | WalletResponseError;

interface WalletResponseSuccess {
    result: string | object;
    id: string;
}

interface WalletResponseError {
    error: { code: number; message: string; data?: unknown };
    id: string;
}
```

`response.id` MUST equal the `request.id` it answers.

### `WalletEvent`

```typescript
interface WalletEvent {
    event: WalletEventName;
    id: number;          // monotonically increasing per session, separate from request IDs
    payload: unknown;    // event-specific
}

type WalletEventName = 'connect' | 'connect_error' | 'disconnect';
```

The wallet MUST increase `event.id` for every new event. The dApp MUST accept the first event `id` it sees in a session as the baseline; for every subsequent event the `id` MUST be strictly greater than the last `id` the dApp has processed in the same session, and the dApp MUST reject events that violate this.

## Central error catalogue

The following RPC error codes are used across TON Connect methods:

| Code | Name                   | Meaning |
|------|------------------------|---------|
| 0    | `UNKNOWN_ERROR`        | An unexpected wallet-side failure. |
| 1    | `BAD_REQUEST`          | The request payload is malformed or violates method constraints. |
| 100  | `UNKNOWN_APP`          | The session or app is unknown to the wallet. |
| 300  | `USER_DECLINED`        | The user explicitly declined the action. |
| 400  | `METHOD_NOT_SUPPORTED` | The method is not supported by this wallet. |

Each method's section defines which codes apply.

## `sendTransaction`

Asks the wallet to sign and broadcast one or more outgoing messages from the connected account.

### Request

```typescript
interface SendTransactionRequest {
    method: 'sendTransaction';
    params: [string]; // JSON string of the transaction payload
    id: string;
}
```

Transaction payload (JSON):

| Field         | Type                | Required | Description |
|---------------|---------------------|----------|-------------|
| `valid_until` | integer             | no       | Unix seconds. The wallet MUST reject the request after this moment. |
| `network`     | `NETWORK_ID`        | no       | Target [TON network global_id](./connect.md#network_id) where the dApp intends to send the transaction (`-239` mainnet, `-3` testnet, or any other valid global_id). The dApp SHOULD set this field explicitly. |
| `from`        | string              | no       | Sender address intended by the dApp (raw or friendly format). |
| `messages`    | `Message[]`         | one of   | Raw outgoing messages. Exactly one of `messages` or `items` MUST be present. |
| `items`       | `TransactionItem[]` | one of   | Structured items. Exactly one of `messages` or `items` MUST be present. |

A payload MUST contain `messages` or `items`, and MUST NOT contain both.

If `network` is omitted, the wallet sends on the currently active wallet network. This is allowed but unsafe because the dApp intent is not explicit. dApps SHOULD always provide `network`.

If `network` is provided and does not match the wallet's active network, the wallet MUST show an alert and MUST NOT allow sending this transaction.

If `from` is omitted, the wallet MAY allow the user to choose the sender account during confirmation.

If `from` is provided, the wallet MUST NOT allow the user to change the sender account. If sending from the specified address is impossible, the wallet MUST show an alert and MUST NOT allow sending this transaction.

#### Raw `Message`

| Field            | Type           | Required | Description |
|------------------|----------------|----------|-------------|
| `address`        | string         | yes      | Destination in friendly format per [TEP-2](https://github.com/ton-blockchain/TEPs/blob/master/text/0002-address.md). |
| `amount`         | decimal string | yes      | Nanograms. |
| `payload`        | base64 string  | no       | Raw one-cell BoC. |
| `stateInit`      | base64 string  | no       | Raw one-cell BoC. |
| `extra_currency` | object         | no       | Map of extra currency ID to decimal string amount. |

The `address` field MUST be in user-friendly format: base64url-encoded with the bounceable / non-bounceable flag. Wallets MUST reject addresses provided in raw format. The wallet MUST extract the bounce flag from the address and use it to determine the bounce behaviour of the message.

The dApp MUST use **bounceable** format (`bounce = true`) for smart-contract destinations. Smart contracts can throw during handling, and uninitialized contracts can reject messages without `stateInit`. The bounceable flag returns funds to the sender in these error cases.

The dApp SHOULD use **non-bounceable** format (`bounce = false`) for wallet contracts and for flows where errors are expected and value should still be delivered. This enables transfers to uninitialized wallets (with deployment on receive) and intentional value transfer to contracts that may throw as part of expected behaviour.

#### Structured `items`

```typescript
type TransactionItem = TonItem | JettonItem | NftItem;
```

Three item types are defined: `ton`, `jetton`, `nft`.

##### `TonItem`

| Field            | Type    | Required | Description |
|------------------|---------|----------|-------------|
| `type`           | `"ton"` | yes      | Item discriminator. |
| `address`        | string  | yes      | Destination address in friendly format. |
| `amount`         | string  | yes      | Nanograms as decimal string. |
| `payload`        | string  | no       | Raw one-cell BoC in base64. |
| `stateInit`      | string  | no       | Raw one-cell BoC in base64. |
| `extra_currency` | object  | no       | Extra currency map (`currency_id -> amount` as decimal string). |

##### `JettonItem`

| Field                 | Type       | Required | Description |
|-----------------------|------------|----------|-------------|
| `type`                | `"jetton"` | yes      | Item discriminator. |
| `master`              | string     | yes      | Jetton master contract address. |
| `destination`         | string     | yes      | Recipient address. |
| `amount`              | string     | yes      | Jetton amount in elementary units. |
| `attachAmount`        | string     | no       | GRAM value to attach for transfer execution. Wallet-calculated if omitted. |
| `queryId`             | string     | no       | Query ID for the transfer body. |
| `responseDestination` | string     | no       | Address for the excess-GRAM refund. Defaults to the sender. |
| `customPayload`       | string     | no       | Raw one-cell BoC in base64. |
| `forwardAmount`       | string     | no       | Nanograms forwarded to the destination. Defaults to `"1"`. |
| `forwardPayload`      | string     | no       | Raw one-cell BoC in base64. |

##### `NftItem`

| Field                 | Type    | Required | Description |
|-----------------------|---------|----------|-------------|
| `type`                | `"nft"` | yes      | Item discriminator. |
| `nftAddress`          | string  | yes      | NFT item contract address. |
| `newOwner`            | string  | yes      | New owner address. |
| `attachAmount`        | string  | no       | Grams value to attach for transfer execution. Wallet-calculated if omitted. |
| `queryId`             | string  | no       | Query ID for the transfer body. |
| `responseDestination` | string  | no       | Address for the excess-Gram refund. Defaults to the sender. |
| `customPayload`       | string  | no       | Raw one-cell BoC in base64. |
| `forwardAmount`       | string  | no       | Nanograms forwarded to the destination. Defaults to `"1"`. |
| `forwardPayload`      | string  | no       | Raw one-cell BoC in base64. |

### Response

```typescript
type SendTransactionResponse = SendTransactionResponseSuccess | SendTransactionResponseError;

interface SendTransactionResponseSuccess {
    result: string;  // base64 BoC of the broadcast external message
    id: string;
}

interface SendTransactionResponseError {
    error: { code: number; message: string };
    id: string;
}
```

### Examples

Request (`sendTransaction` with raw `messages`):

```json
{
  "method": "sendTransaction",
  "params": [
    "{\"valid_until\":1764424242,\"network\":\"-239\",\"from\":\"Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU\",\"messages\":[{\"address\":\"Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU\",\"amount\":\"100000000\"}]}"
  ],
  "id": "42"
}
```

Success response:

```json
{
  "result": "te6ccgEBAQEAAwAA...",
  "id": "42"
}
```

Error response (user declined):

```json
{
  "error": {
    "code": 300,
    "message": "User declined the transaction"
  },
  "id": "42"
}
```

### Errors

| Code | Name                   | Description                    |
|------|------------------------|--------------------------------|
| 0    | `UNKNOWN_ERROR`        | Unknown error.                 |
| 1    | `BAD_REQUEST`          | Bad request.                   |
| 100  | `UNKNOWN_APP`          | Unknown app.                   |
| 300  | `USER_DECLINED`        | User declined the transaction. |
| 400  | `METHOD_NOT_SUPPORTED` | Method not supported.          |

## `signMessage`

Asks the wallet to sign an internal message **without broadcasting** it. Returns a signed BoC the dApp can submit through a relayer.

The payload shape is identical to [`sendTransaction`](#sendtransaction):

- `valid_until` (optional)
- `network` (optional)
- `from` (optional)
- exactly one of `messages` or `items`

The wallet MUST apply the same payload validation rules as `sendTransaction`. The wallet MUST construct each outgoing message with **send mode 3** (`PAY_GAS_SEPARATELY + IGNORE_ERRORS`).

```typescript
interface SignMessageRequest {
    method: 'signMessage';
    params: [string]; // JSON string of the same shape as sendTransaction
    id: string;
}
```

Even though the wallet does not broadcast, the wallet MUST display a confirmation dialog warning the user that the signed message may transfer assets and could be submitted to the network later by the requesting application.

### Response

```typescript
type SignMessageResponse = SignMessageResponseSuccess | SignMessageResponseError;

interface SignMessageResponseSuccess {
    result: { internalBoc: string }; // base64 BoC of the signed internal message
    id: string;
}

interface SignMessageResponseError {
    error: { code: number; message: string };
    id: string;
}
```

### Examples

Request (`signMessage` with raw `messages`):

```json
{
  "method": "signMessage",
  "params": [
    "{\"valid_until\":1764424242,\"network\":\"-239\",\"from\":\"Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU\",\"messages\":[{\"address\":\"Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU\",\"amount\":\"100000000\"}]}"
  ],
  "id": "43"
}
```

Success response:

```json
{
  "result": {
    "internalBoc": "te6ccgEBAQEAAwAA..."
  },
  "id": "43"
}
```

Error response (user declined):

```json
{
  "error": {
    "code": 300,
    "message": "User declined the request"
  },
  "id": "43"
}
```

### Errors

| Code | Name                   | Description                |
|------|------------------------|----------------------------|
| 0    | `UNKNOWN_ERROR`        | Unknown error.             |
| 1    | `BAD_REQUEST`          | Bad request.               |
| 100  | `UNKNOWN_APP`          | Unknown app.               |
| 300  | `USER_DECLINED`        | User declined the request. |
| 400  | `METHOD_NOT_SUPPORTED` | Method not supported.      |

## `signData`

Asks the wallet to sign opaque data — text, binary bytes or cell.

### Request

```typescript
interface SignDataRequest {
    method: 'signData';
    params: [string]; // JSON string of one of three payload variants
    id: string;
}
```

Payload variants:

| `type`   | Required fields | Optional fields | Description |
|----------|-----------------|-----------------|-------------|
| `text`   | `type`, `text`  | `network`, `from` | Sign UTF-8 text payload. |
| `binary` | `type`, `bytes` | `network`, `from` | Sign binary payload (`bytes` is base64). |
| `cell`   | `type`, `schema`, `cell` | `network`, `from` | Sign cell payload (`cell` is base64 BoC, `schema` is TL-B). |

If `schema` defines several types, the **last** declared type is the root.

`network` is a [`NETWORK_ID`](./connect.md#network_id) — a stringified TON network global_id — specifying where the dApp intends signing to occur. If `network` is omitted, the wallet signs with the currently active wallet network. This is allowed but unsafe because dApp intent is not explicit. dApps SHOULD always set `network`.

If `network` is provided and does not match the wallet's active network, the wallet MUST show an alert and MUST NOT allow signing.

`from` specifies the signer address intended by the dApp (raw or friendly format). If `from` is omitted, the wallet MAY allow account selection during signing.

If `from` is provided, the wallet MUST NOT allow changing the signer address. If signing from the specified address is impossible, the wallet MUST show an alert and MUST NOT allow signing.

### Wallet display rules

For general wallet UX and confirmation requirements, see [`guides/wallet-guidelines.md`](../guides/wallet-guidelines.md).

- `text` — display the `text` field verbatim, monospace, with forced line breaks. The user SHOULD scroll long text before signing.
- `binary` — display a warning that the content being signed is unknown.
- `cell` — the wallet MAY parse the cell against `schema`. Otherwise, wallet MUST display a warning that the content is unknown. If `schema` is unparseable or `cell` does not match it, wallet MUST display the same warning.

### Response

```typescript
interface SignDataResponseSuccess {
    result: {
        signature: string;  // base64 signature
        address: string;    // raw wallet address
        timestamp: number;  // unix seconds at signing time (UTC)
        domain: string;     // app domain (URL part, not encoded)
        payload: object;    // the payload from the request
    };
    id: string;
}
```

### Examples

Request (`signData` with `text` payload):

```json
{
  "method": "signData",
  "params": [
    "{\"type\":\"text\",\"text\":\"Sign in to the dApp\",\"network\":\"-239\",\"from\":\"Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU\"}"
  ],
  "id": "44"
}
```

Success response:

```json
{
  "result": {
    "signature": "Q2hhbmdlTWU...",
    "address": "0:7f3a...9c21",
    "timestamp": 1764424200,
    "domain": "app.example",
    "payload": {
      "type": "text",
      "text": "Sign in to the dApp",
      "network": "-239",
      "from": "Ef8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAU"
    }
  },
  "id": "44"
}
```

Error response (user declined):

```json
{
  "error": {
    "code": 300,
    "message": "User declined the request"
  },
  "id": "44"
}
```

### Signature — `text` and `binary`

```text
message = 0xffff ++
          utf8_encode("ton-connect/sign-data/") ++
          Address ++
          AppDomain ++
          Timestamp ++
          Payload
signature = Ed25519Sign(privkey, sha256(message))
```

Where:

- `Address` is the wallet address encoded as:
  - `workchain`: 32-bit signed integer, big-endian;
  - `hash`: 256-bit unsigned integer, big-endian.
- `AppDomain` is `Length ++ EncodedDomainName` from the dApp manifest domain (without scheme), matching connect semantics:
  - `Length`: 32-bit length of UTF-8 encoded domain, in bytes;
  - `EncodedDomainName`: UTF-8 bytes of the domain name.
- `Timestamp` is a 64-bit Unix epoch time of the signing operation.
- `Payload` is `Prefix ++ PayloadLength ++ PayloadData`:
  - `Prefix`: `utf8_encode("txt")` for text payloads, `utf8_encode("bin")` for binary payloads;
  - `PayloadLength`: 32-bit length of `PayloadData`, in bytes;
  - `PayloadData`: UTF-8 text bytes for `text`, or raw bytes for `binary`.

### Signature — `cell`

```typescript
let payload = beginCell()
    .storeUint(0x75569022, 32)
    .storeUint(crc32(schema), 32)
    .storeUint(timestamp, 64)
    .storeAddress(userWalletAddress)
    .storeStringRefTail(appDomain)
    .storeRef(cell);

let signature = Ed25519Sign(payload.hash(), privkey);
```

TL-B:

```text
message#75569022 schema_hash:uint32 timestamp:uint64 userAddress:MsgAddress
                  {n:#} appDomain:^(SnakeData ~n) payload:^Cell = Message;
```

Where:

- `schema` is the TL-B scheme of the cell payload as a UTF-8 string.
- `timestamp` is a 64-bit Unix epoch time of the signing operation.
- `userWalletAddress` is the wallet address that signs the payload.
- `appDomain` MUST be encoded exactly in the DNS wire format defined by [TEP-81](https://github.com/ton-blockchain/TEPs/blob/master/text/0081-dns-standard.md), for example `com\0stonfi\0`.
- `cell` is an arbitrary payload cell.

### Smart contract verification (cell payload)

A receiver smart contract MUST:

1. Verify the message signature.
2. Verify the first 32 bits equal `0x75569022`.
3. Verify `schema_hash` equals the expected hash of the schema.
4. Verify the message has not expired against contract-specific limits.
5. Verify `userWalletAddress` equals the expected wallet address.
6. Verify `appDomain` equals the expected app domain (or its hash).

### Choosing a format

| Data type                                        | Use            |
|--------------------------------------------------|----------------|
| Human-readable text                              | `text`         |
| Will be used on-chain                            | `cell`         |
| Human-readable but not textual (e.g. structured) | `cell` with schema |
| Anything else                                    | `binary`       |

### Errors

| Code | Name                   | Description                |
|------|------------------------|----------------------------|
| 0    | `UNKNOWN_ERROR`        | Unknown error.             |
| 1    | `BAD_REQUEST`          | Bad request.               |
| 100  | `UNKNOWN_APP`          | Unknown app.               |
| 300  | `USER_DECLINED`        | User declined the request. |
| 400  | `METHOD_NOT_SUPPORTED` | Method not supported.      |

## `disconnect`

Tells the wallet that the dApp has ended the session — the wallet can free resources and update its UI.

```typescript
interface DisconnectRequest {
    method: 'disconnect';
    params: [];
    id: string;
}

type DisconnectResponse = DisconnectResponseSuccess | DisconnectResponseError;

interface DisconnectResponseSuccess {
    result: { };
    id: string;
}

interface DisconnectResponseError {
    error: { code: number; message: string };
    id: string;
}
```

The wallet SHOULD NOT emit a `disconnect` event when the dApp initiated the disconnect.

### Errors

| Code | Name                   | Description           |
|------|------------------------|-----------------------|
| 0    | `UNKNOWN_ERROR`        | Unknown error.        |
| 1    | `BAD_REQUEST`          | Bad request.          |
| 100  | `UNKNOWN_APP`          | Unknown app.          |
| 400  | `METHOD_NOT_SUPPORTED` | Method not supported. |

## Wallet events

### `connect` / `connect_error`

Emitted in response to a `ConnectRequest`. Shapes are defined in [`connect.md`](./connect.md). The optional `response` field on `ConnectEventSuccess` is present only when the connect URL carried an [embedded request](./deeplinks.md#embedded-requests-e).

### Disconnect event

Fires when the wallet ends the session — the user removed the dApp from the wallet's session list. The dApp MUST react by deleting any saved session state. The wire format is the same `disconnect` event:

```typescript
interface DisconnectEvent {
    event: "disconnect";
    id: number;
    payload: { };
}
```

## See also

- [`connect.md`](./connect.md) — `ConnectRequest`, `ConnectEvent`, `Feature`
- [`guides/structured-items.md`](../guides/structured-items.md) — wallet implementation guide for `items`
- [`guides/sign-message.md`](../guides/sign-message.md) — wallet implementation guide for `signMessage`
- [`guides/sign-data.md`](../guides/sign-data.md) — wallet implementation guide for `signData`
