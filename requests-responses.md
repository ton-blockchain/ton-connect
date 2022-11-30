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
  // arbitrary payload, e.g. nonce + expiration timestamp.
  payload: string;
}
```

Wallet responds with **ConnectEvent** message if the user approves the request. 

```tsx
type ConnectEvent = ConnectEventSuccess | ConnectEventError;

type ConnectEventSuccess = {
  event: "connect";
  payload: {
      items: ConnectItemReply[];
      device: DeviceInfo;   
  }
}
type ConnectEventError = {
  event: "connect_error",
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

type Feature = 'SendTransaction';

type ConnectItemReply = TonAddressItemReply | TonProofItemReply ...;

type TonAddressItemReply = {
  name: "ton_addr";
  address: string; // TON address raw (`0:<hex>`)
  network: NETWORK; // network global_id
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
   * workchain: 32-bit signed integer big endian;
   * hash: 256-bit unsigned integer big endian;
* `AppDomain` is Length ++ EncodedDomainName
  - `Length` is 32-bit value of utf-8 encoded app domain name length in bytes
  - `EncodedDomainName` id `Length`-byte  utf-8 encoded app domain name
* `Timestamp` 64-bit unix epoch time of the signing operation 
* `Payload` is a variable-length binary string.

Note: payload is variable-length untrusted data. To avoid using unnecessary length prefixes we simply put it last in the message.

The signature must be verified using the public key provided via `get_public_key` method on smart contract deployed at `Address`.


## Messages

- All messages from the app to the wallet are requests for an operation.
- Messages from the wallet to the application can be either responses to app requests or events triggered by user actions on the side of the wallet.

**Available operations:**

- sendTransaction
- singMessage (will be implemented later)

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
- method: name of the operation ('sendTransaction', 'singMessage', ...)
- params: array of the operation specific parameters
- id: identifier that allows to match requests and responses



**Wallet messages are responses or events.**

Response is an object formatted as a json-rpc 2.0 response. Response `id` must match request's id. 
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

Event is an object with property `event` that is equal to event's name, and `payload` that contains event additional data. 
```tsx
interface WalletEvent {
    event: WalletEventName;
    payload: <event-payload>; // specific payload for each event
}

type WalletEventName = 'connect' | 'connect_error' | 'disconnect';
```

### Methods

<ins>Sign and send transaction</ins>

App sends **SendTransactionRequest**:

```tsx
interface SendTransactionRequest {
	method: 'sendTransaction';
	params: [<transaction-payload>];
	id: number;
}
```

Where `<transaction-payload>` is JSON with following properties:

* `valid_until` (integer, optional): unix timestamp. after th moment transaction will be invalid.
* `messages` (array of messages): 1-4 outgoing messages from the wallet contract to other accounts. All messages are sent out in order, however **the wallet cannot guarantee that messages will be delivered and executed in same order**.

Message structure:
* `address` (string): message destination
* `amount` (decimal string): number of nanocoins to send.
* `payload` (string base64, optional): raw one-cell BoC encoded in Base64.
* `stateInit` (string base64, optional): raw once-cell BoC encoded in Base64.

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
  "messages": [
    {
      "address": "0:412410771DA82CBA306A55FA9E0D43C9D245E38133CB58F1457DFB8D5CD8892F",
      "amount": "20000000",
      "initState": "base64bocblahblahblah==" //deploy contract
    },{
      "address": "0:E69F10CC84877ABF539F83F879291E5CA169451BA7BCE91A37A5CED3AB8080D3",
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
   error: { code: 1; message: 'User declined the transaction' };
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


### Wallet events

<ins>Disconnect</ins>

The event fires when the user deletes the app in the wallet. The app must react to the event and delete the saved session. If the user disconnects the wallet on the app side, then the event does not fire, and the session information remains in the localstorage

```tsx
interface DisconnectEvent {
	type: "disconnect",
	payload: { }
}
```

<ins>Connect</ins>
```tsx
type ConnectEventSuccess = {
    event: "connect";
    payload: {
        items: ConnectItemReply[];
        device: DeviceInfo;
    }
}
type ConnectEventError = {
    event: "connect_error",
    payload: {
        code: number; // 1 = user cancelled; 0 = unknown;
        message: string;
    }
}
```
