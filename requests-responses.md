# Requests and Responses

App sends requests to the wallet. Wallet sends responses and events to the app.

```tsx
type AppMessage = ConnectRequest | AppRequest;

type WalletMessage = WalletResponse | WalletEvent;
```

### App manifest
App needs to have its manifest to pass meta information to the wallet. Manifest is a JSON file named as `tonconnect-manifest.json` following format:

```json
{
  "url": "<app-url>",                        // required
  "name": "<app-name>",                      // required
  "iconUrl": "<app-icon-url>",               // required
  "termsOfUseUrl": "<terms-of-use-url>",     // optional
  "privacyPolicyUrl": "<privacy-policy-url>" // optional
}
```

Best practice is to place the manifest in the root of your app, e.g. `https://myapp.com/tonconnect-manifest.json`. It allows the wallet to handle your app better and improve the UX connected to your app.
Make sure that manifest is available to GET by its URL.

#### Fields description
- `url` -- app URL. Will be used as the dapp identifier. Will be used to open the dapp after click to its icon in the wallet. It is recommended to pass url without closing slash, e.g. 'https://mydapp.com' instead of 'https://mydapp.com/'.
- `name` -- app name. Might be simple, will not be used as identifier.
- `iconUrl` -- Url to the app icon. Must be PNG, ICO, ... format. SVG icons are not supported. Perfectly pass url to a 180x180px PNG icon.
- `termsOfUseUrl` -- (optional) url to the Terms Of Use document. Optional for usual apps, but required for the apps which is placed in the Tonkeeper recommended apps list.
- `privacyPolicyUrl` -- (optional) url to the Privacy Policy document. Optional for usual apps, but required for the apps which is placed in the Tonkeeper recommended apps list.

### Initiating connection

App’s request message is **InitialRequest**.

```tsx
type ConnectRequest = {
  manifestUrl: string;
  items: ConnectItem[], // data items to share with the app
}

// In the future we may add other personal items.
// Or, instead of the wallet address we may ask for per-service ID.
type ConnectItem = TonAddressItem | TonProofItem | ...;

type TonAddressItem = {
  name: "ton_addr";
}
type TonProofItem = {
  name: "ton_proof";
  payload: string; // arbitrary payload, e.g. nonce + expiration timestamp.
}
```

ConnectRequest description:
- `manifestUrl`: link to the app's tonconnect-manifest.json
- `items`: data items to share with the app.

Wallet responds with **ConnectEvent** message if the user approves the request. 

```tsx
type ConnectEvent = ConnectEventSuccess | ConnectEventError;

type ConnectEventSuccess = {
  event: "connect";
  id: number; // increasing event counter
  payload: {
      items: ConnectItemReply[];
      device: DeviceInfo;   
  }
}
type ConnectEventError = {
  event: "connect_error",
  id: number; // increasing event counter
  payload: {
      code: number;
      message: string;
  }
}

type DeviceInfo = {
  platform: "iphone" | "ipad" | "android" | "windows" | "mac" | "linux";
  appName:      string; // e.g. "Tonkeeper"  
  appVersion:  string; // e.g. "2.3.367"
  maxProtocolVersion: number;
  features: Feature[]; // list of supported features and methods in RPC
                                // Currently there is only one feature -- 'SendTransaction'; 
}

type Feature = { name: 'SendTransaction', maxMessages: number } | // `maxMessages` is maximum number of messages in one `SendTransaction` that the wallet supports
        { name: 'SignData' };

type ConnectItemReply = TonAddressItemReply | TonProofItemReply ...;

// Untrusted data returned by the wallet. 
// If you need a guarantee that the user owns this address and public key, you need to additionally request a ton_proof.
type TonAddressItemReply = {
  name: "ton_addr";
  address: string; // TON address raw (`0:<hex>`)
  network: NETWORK; // network global_id
  publicKey: string; // HEX string without 0x
  walletStateInit: string; // Base64 (not url safe) encoded stateinit cell for the wallet contract
}

type TonProofItemReply = TonProofItemReplySuccess | TonProofItemReplyError;

type TonProofItemReplySuccess = {
  name: "ton_proof";
  proof: {
    timestamp: string; // 64-bit unix epoch time of the signing operation (seconds)
    domain: {
      lengthBytes: number; // AppDomain Length
      value: string;  // app domain name (as url part, without encoding) 
    };
    signature: string; // base64-encoded signature
    payload: string; // payload from the request
  }
}

type TonProofItemReplyError = {
  name: "ton_addr";
  error: {
      code: ConnectItemErrorCode;
      message?: string;
  }
}

enum NETWORK {
  MAINNET = '-239',
  TESTNET = '-3'
}
```

**Connect event error codes:**

| code | description                  |
|------|------------------------------|
| 0    | Unknown error                |
| 1    | Bad request                  |
| 2    | App manifest not found       |
| 3    | App manifest content error   |
| 100  | Unknown app                  |
| 300  | User declined the connection |

**Connect item error codes:**

| code | description                  |
|------|------------------------------|
| 0    | Unknown error                |
| 400  | Method is not supported      |

If the wallet doesn't support the requested `ConnectItem` (e.g. "ton_proof"), it must send reply with the following ConnectItemReply corresponding to the requested item.
with following structure: 
```ts
type ConnectItemReplyError = {
  name: "<requested-connect-item-name>";
  error: {
      code: 400;
      message?: string;
  }
}
```

### Address proof signature (`ton_proof`)

If `TonProofItem` is requested, wallet proves ownership of the selected account’s key. The signed message is bound to:

- Unique prefix to separate messages from on-chain messages. (`ton-connect`)
- Wallet address.
- App domain
- Signing timestamp
- App’s custom payload (where server may put its nonce, cookie id, expiration time).

```
message = utf8_encode("ton-proof-item-v2/") ++ 
          Address ++
          AppDomain ++
          Timestamp ++  
          Payload 
signature = Ed25519Sign(privkey, sha256(0xffff ++ utf8_encode("ton-connect") ++ sha256(message)))
```

where:

* `Address` is the wallet address encoded as a sequence: 
   * `workchain`: 32-bit signed integer big endian;
   * `hash`: 256-bit unsigned integer big endian;
* `AppDomain` is Length ++ EncodedDomainName
  - `Length` is 32-bit value of utf-8 encoded app domain name length in bytes
  - `EncodedDomainName` id `Length`-byte  utf-8 encoded app domain name
* `Timestamp` 64-bit unix epoch time of the signing operation 
* `Payload` is a variable-length binary string.

Note: payload is variable-length untrusted data. To avoid using unnecessary length prefixes we simply put it last in the message.

The signature must be verified by public key:

1. First, try to obtain public key via `get_public_key` get-method on smart contract deployed at `Address`.

2. If the smart contract is not deployed yet, or the get-method is missing, you need:

   2.1. Parse `TonAddressItemReply.walletStateInit` and get public key from stateInit. You can compare the `walletStateInit.code` with the code of standard wallets contracts and parse the data according to the found wallet version.

   2.2. Check that `TonAddressItemReply.publicKey` equals to obtained public key
   
   2.3. Check that `TonAddressItemReply.walletStateInit.hash()` equals to `TonAddressItemReply.address`. `.hash()` means BoC hash.

## Messages

- All messages from the app to the wallet are requests for an operation.
- Messages from the wallet to the application can be either responses to app requests or events triggered by user actions on the side of the wallet.

**Available operations:**

- sendTransaction
- signData
- disconnect

**Available events:**

- connect
- connect_error
- disconnect

### Structure

**All app requests have the following structure (like json-rpc 2.0)**
```tsx
interface AppRequest {
	method: string;
	params: string[];
	id: string;
}
```
Where 
- `method`: name of the operation ('sendTransaction', 'signMessage', ...)
- `params`: array of the operation specific parameters
- `id`: increasing identifier that allows to match requests and responses


**Wallet messages are responses or events.**

Response is an object formatted as a json-rpc 2.0 response. Response `id` must match request's id. 

Wallet doesn't accept any request with an id that does not greater the last processed request id of that session.

```tsx
type WalletResponse = WalletResponseSuccess | WalletResponseError;

interface WalletResponseSuccess {
    result: string;
    id: string;
}

interface WalletResponseError {
    error: { code: number; message: string; data?: unknown };
    id: string;
}
```

Event is an object with property `event` that is equal to event's name, `id` that is increasing event counter (**not** related to `request.id` because there is no request for an event), and `payload` that contains event additional data. 
```tsx
interface WalletEvent {
    event: WalletEventName;
    id: number; // increasing event counter
    payload: <event-payload>; // specific payload for each event
}

type WalletEventName = 'connect' | 'connect_error' | 'disconnect';
```

Wallet must increase `id` while generating a new event. (Every next event must have `id` > previous event `id`)

DApp doesn't accept any event with an id that does not greater the last processed event id of that session. 

### Methods

#### Sign and send transaction

App sends **SendTransactionRequest**:

```tsx
interface SendTransactionRequest {
	method: 'sendTransaction';
	params: [<transaction-payload>];
	id: string;
}
```

Where `<transaction-payload>` is JSON with following properties:

* `valid_until` (integer, optional): unix timestamp. after th moment transaction will be invalid.
* `network` (NETWORK, optional): The network (mainnet or testnet) where DApp intends to send the transaction. If not set, the transaction is sent to the network currently set in the wallet, but this is not safe and DApp should always strive to set the network. If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.
* `from` (string in <wc>:<hex> format, optional) - The sender address from which DApp intends to send the transaction. If not set, wallet allows user to select the sender's address at the moment of transaction approval. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the sender's address; If sending from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.
* `messages` (array of messages): 1-4 outgoing messages from the wallet contract to other accounts. All messages are sent out in order, however **the wallet cannot guarantee that messages will be delivered and executed in same order**.

Message structure:
* `address` (string): message destination in user-friendly format
* `amount` (decimal string): number of nanocoins to send.
* `payload` (string base64, optional): raw one-cell BoC encoded in Base64.
* `stateInit` (string base64, optional): raw once-cell BoC encoded in Base64.
* `extra_currency` (object, optional): extra currency to send with the message. 


<details>
<summary>Common cases</summary>
1. No payload, no stateInit: simple transfer without a message.
2. payload is prefixed with 32 zero bits, no stateInit: simple transfer with a text message.
3. No payload or prefixed with 32 zero bits; stateInit is present: deployment of the contract.
</details>

<details>
<summary>Example</summary>

```json5
{
  "valid_until": 1658253458,
  "network": "-239", // enum NETWORK { MAINNET = '-239', TESTNET = '-3'}
  "from": "0:348bcf827469c5fc38541c77fdd91d4e347eac200f6f2d9fd62dc08885f0415f",
  "messages": [
    {
      "address": "EQBBJBB3HagsujBqVfqeDUPJ0kXjgTPLWPFFffuNXNiJL0aA",
      "amount": "20000000",
      "stateInit": "base64bocblahblahblah==", //deploy contract
      "extra_currency": {
        "239": "1000000000"
      }
    },{
      "address": "EQDmnxDMhId6v1Ofg_h5KR5coWlFG6e86Ro3pc7Tq4CA0-Jn",
      "amount": "60000000",
      "payload": "base64bocblahblahblah==" //transfer nft to new deployed account 0:412410771DA82CBA306A55FA9E0D43C9D245E38133CB58F1457DFB8D5CD8892F
    }
  ]
}
```
</details>


Wallet replies with **SendTransactionResponse**:

```tsx
type SendTransactionResponse = SendTransactionResponseSuccess | SendTransactionResponseError; 

interface SendTransactionResponseSuccess {
    result: <boc>;
    id: string;
	
}

interface SendTransactionResponseError {
   error: { code: number; message: string };
   id: string;
}
```

**Error codes:**

| code | description                   |
|------|-------------------------------|
| 0    | Unknown error                 |
| 1    | Bad request                   |
| 100  | Unknown app                   |
| 300  | User declined the transaction |
| 400  | Method not supported       |


#### Sign Data

App sends **SignDataRequest**:

```tsx
interface SignDataRequest {
	method: 'signData';
	params: [<sign-data-payload>];
	id: string;
}
```

Where `<sign-data-payload>` is JSON with one of the 3 types of payload:

- **Text**. JSON object with following properties:
  - **type** (string): 'text'
  - **text** (string): arbitrary UTF-8 text to sign. 

- **Binary**. JSON object with following properties:
  - **type** (string): 'binary'
  - **bytes** (string): base64 (not url safe) encoded arbitrary bytes array to sign.

- **Cell**. JSON object with following properties:
  - **type** (string): 'cell'
  - **schema** (string): TL-B schema of the cell payload as an UTF-8 string.  
  *If the schema contains several type definitions, the **last** declared type is treated as the root during serialization and deserialization.*
  - **cell** (string): base64 (not url safe) encoded BoC (single-root) with arbitrary cell to sign.

**Examples:**

```json5
{
  "method": "signData",
  "params": {
    "type": "text",
    "text": "Confirm new 2fa number:\n+1 234 567 8901"
  },
  "id": "1"
}
```

```json5
{
  "method": "signData",
  "params": {
    "type": "binary",
    "bytes": "1Z/SGh+3HFMKlVHSkN91DpcCzT4C5jzHT3sA/24C5A=="
  },
  "id": "2"
}
```

```json5
{
  "method": "signData",
  "params": {
    "type": "cell",
    "schema": "transfer#0f8a7ea5 query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress response_destination:MsgAddress custom_payload:(Maybe ^Cell) forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell) = InternalMsgBody;",
    "cell": "te6ccgEBAQEAVwAAqg+KfqVUbeTvKqB4h0AcnDgIAZucsOi6TLrfP6FcuPKEeTI6oB3fF/NBjyqtdov/KtutACCLqvfmyV9kH+Pyo5lcsrJzJDzjBJK6fd+ZnbFQe4+XggI="
  },
  "id": "3"
}
```

**Wallet behaviour:**

- If the payload is in the Text format, Wallet MUST show the content from the `text` field to the user with monospace font, without any formatting and with forced line breaks. User SHOULD scroll long message before signing.
- If the payload is in the Binary format, Wallet MUST display a warning message informing that the content being signed is unknown.
- If the payload is in the Cell format, Wallet CAN parse TL-B scheme from the `schema` field and verify payload using this scheme. Otherwise, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the `schema` contains unparseable data, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the data in the `cell` field cannot be parsed using the provided TL-B sheme, Wallet MUST display a warning message informing that the content being signed is unknown.
    - If the data is parsed successfully, Wallet MUST display the parsed data to the user in any readable format.

Wallet replies with **SignDataResponse**:

```tsx
type SignDataResponse = SignDataResponseSuccess | SignDataResponseError;

interface SignDataResponseSuccess {
    result: {
        signature: string; // base64 encoded signature
        address: string;   // wallet address in raw format
        timestamp: number; // UNIX timestamp in seconds (UTC) at the moment on creating the signature.
        domain: string;  // app domain name (as url part, without encoding) 
        payload: <sign-data-payload>; // payload received from the request in the `params` field
    };
    id: string;
}

interface SignDataResponseError {
    error: { code: number; message: string };
    id: string;
}
```

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 300  | User declined the request |
| 400  | Method not supported      |

**Signature**

If the payload is in the Text or Binary format, then signature computes as follows:

```
message = 0xffff ++
          utf8_encode("ton-connect/sign-data/") ++
          Address ++
          AppDomain ++
          Timestamp ++
          Payload
signature = Ed25519Sign(privkey, sha256(message))
```

where:

- `Address` is the wallet address encoded as a sequence:
  - `workchain`: 32-bit signed integer big endian;
  - `hash`: 256-bit unsigned integer big endian;
- `AppDomain` is Length ++ EncodedDomainName from dApp manifest without scheme (same as in `connect` event)
  - `Length` is 32-bit value of utf-8 encoded app domain name length in bytes;
  - `EncodedDomainName` is `Length`byte utf-8 encoded app domain name;
- `Timestamp` 64-bit unix epoch time of the signing operation
- `Payload` is Prefix ++ PayloadLength ++ PayloadData:
  - `Prefix` is `utf8_encode("txt")` for text data or `utf8_encode("bin")` for binary data;
  - `PayloadLength` is 32-bit value of PayloadData length in bytes;
  - `PayloadData` is `PayloadLength` byte payload data, which is `utf8_encode(text)` for text data or bytes array for binary data;

If the payload is in the Cell format, then signature computes as follows:

```tsx
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

```tsx
message#75569022 schema_hash:uint32 timestamp:uint64 userAddress:MsgAddress 
					{n:#} appDomain:^(SnakeData ~n) payload:^Cell = Message;
```

where:

- `schema` is the TL-B scheme of the cell payload in the utf-8 encoded string.
- `timestamp` is 64-bit unix epoch time of the signing operation.
- `userWalletAddress` is the user wallet address that signs the payload.
- `appDomain` from dApp manifest is DNS-like encoded domain name (e.g. `\0com\0stonfi`).
- `cell` is the arbitrary payload cell.

**Smart Contract Behaviour**

If receiver of the signed data is a smart contract, then the smart contract MUST perform following steps:

1. Smart contract MUST verify that the message signature belongs to the message.
2. Smart contract MUST verify that first 32 bytes equals to the prefix `0x75569022`.
3. Smart contract MUST verify `schema_hash` field is equal to the expected hash of the schema.
4. Smart contract MUST verify that the message has not expired according to its expiration rules by comparing timestamp from the `timestamp` field. Smart contract MUST have limits on the message expiration.
5. Smart contract MUST check that `userWalletAddress` is equal to the expected wallet address.
6. Smart contract MUST check that `appDomain` is equal to the application app domain. Smart contract CAN compare hash of the `appDomain` instead.

**Which format should App choose?**

If the data is human-readable text, use Text format.

If the data should be used in the TON Blockchain, use Cell format.

If the data needs to be human-readable—but is not textual—use the **Cell** format. In this format, include a `schema` field that contains the TL-B schema of the payload. However, note that the Wallet is not required to display the content of the Cell.

Otherwise, use Binary format.

#### Disconnect operation
When user disconnects the wallet in the dApp, dApp should inform the wallet to help the wallet save resources and delete unnecessary session.
Allows the wallet to update its interface to the disconnected state.

```tsx
interface DisconnectRequest {
	method: 'disconnect';
	params: [];
	id: string;
}
```

Wallet replies with **DisconnectResponse**:

```ts
type DisconnectResponse = DisconnectResponseSuccess | DisconnectResponseError; 

interface DisconnectResponseSuccess {
    id: string;
    result: { };
}

interface DisconnectResponseError {
   error: { code: number; message: string };
   id: string;
}
```

Wallet **shouldn't** emit a 'Disconnect' event if disconnect was initialized by the dApp. 

**Error codes:**

| code | description               |
|------|---------------------------|
| 0    | Unknown error             |
| 1    | Bad request               |
| 100  | Unknown app               |
| 400  | Method not supported      |


### Wallet events

<ins>Disconnect</ins>

The event fires when the user deletes the app in the wallet. The app must react to the event and delete the saved session. If the user disconnects the wallet on the app side, then the event does not fire, and the session information remains in the localstorage

```tsx
interface DisconnectEvent {
	type: "disconnect",
	id: number; // increasing event counter
	payload: { }
}
```

<ins>Connect</ins>
```tsx
type ConnectEventSuccess = {
    event: "connect";
    id: number; // increasing event counter
    payload: {
        items: ConnectItemReply[];
        device: DeviceInfo;
    }
}
type ConnectEventError = {
    event: "connect_error",
    id: number; // increasing event counter
    payload: {
        code: number;
        message: string;
    }
}
```
