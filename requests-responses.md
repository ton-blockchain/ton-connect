# Requests and Responses

App sends requests to the wallet. Wallet sends responses and events to the app.

```tsx
type AppSocketMessage = InitialRequest | AppRequest;

type WalletSocketMessage = InitialReply | WalletMesage;
```

### Initiating connection

App’s request message is **InitialRequest**.

```tsx
type InitialRequest = {
  name:  string; // app name
  url:   string; // app url
  icon:  string; // app icon url
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

Wallet responds with **InitialReply** message if the user approves the request. 

```tsx
type InitialReply = InitialReplyOk | InitialReplyError;

type InitialReplyOk = {
  type: "init_ok";
  items: ConnectItemReply[];
  device: DeviceInfo;
}
type InitialReplyError = {
  type: "init_error";
  code: number; // 1 = user cancelled; 0 = unknown;
  message: string;
}

type DeviceInfo = {
  platform: "iphone" | "ipad" | "android" | "windows" | "mac" | "linux";
  app:      string; // e.g. "Tonkeeper"  
  version:  string; // e.g. "2.3.367"
}

type ConnectItemReply = TonAddressItemReply | TonProofItemReply ...;

type TonAddressItemReply = {
  name: "ton_addr";
  address: string; // TON address raw (`0:<hex>`)
  network: string; // hex-encoded genesis hash
}

type TonProofItemReply = {
  name: "ton_proof";
  signature: string; // base64-encoded signature
}
```

### Address proof signature (`ton_proof`)

If `TonProofItem` is requested, wallet proves ownership of the selected account’s key. The signed message is bound to:

- Unique prefix to separate messages from on-chain messages. (`ton-connect`)
- Wallet address.
- Session: wallet’s client ID and app’s client ID.
- App’s custom payload (where server may put its nonce, cookie id, expiration time).

```
message = "TonProofItemV2/" ++ Address ++ "/" ++ 
          WalletClientID ++ AppClientID ++ Payload
signature = Ed25519Sign(privkey, sha256(0xffff ++ "ton-connect" ++ sha256(message)))
```

where:

* `Address` is the wallet address encoded as (TBD);
* `WalletClientID` is a 32-byte binary string;
* `AppClientID` is a 32-byte binary string;
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



**Wallet messages are operation responses or events.**
```tsx
interface WalletMessage {
	type: WalletEventName | 'response';
	payload: <json-rpc response or event payload>;
}

type WalletEventName = 'disconnect' // currently only disconnect event is available
```
Where 
- type: name of the event ('disconnect', ...) or 'response'
- payload: if type is name of some event then it is specific payload for the event, if type is 'response' it is json-rpc 2.0 response


Exactly 
```tsx
type WalletMessage = WalletResponse | WalletEvent;

interface WalletEvent {
    type: WalletEventName;
    payload: <event-payload>; // specific payload for each event
} 
```

Operation response is object with property `type` equals to `'response'` and property `payload` equals to json-rpc 2.0 response.
```tsx
interface WalletResponse {
    type: 'response';
    payload: OperationResult; // json-rpc 2.0 response
}

type OperationResult = OperationResultSuccess | OperationResultError;

interface OperationResultSuccess {
    result: unknown;
    id: number;
}

interface OperationResultError {
    error: { code: number; message: string; data?: unknown };
    id: number;
}
```

### Operations

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
    type: 'response';
    payload: {
		result: <boc>;
		id: number;
	}
}

interface SendTransactionResponseError {
	type: 'response';
    payload: {
		error: { code: 1; message: 'User declined the transaction' };
		id: number;
	}
}
```

**Error codes:**

| code | description |
| --- | --- |
| 0 | Unknown error |
| 1 | User declined the transaction |

### Wallet events

<ins>Disconnect</ins>

The event fires when the user deletes the app in the wallet. The app must react to the event and delete the saved session. If the user disconnects the wallet on the app side, then the event does not fire, and the session information remains in the localstorage

```tsx
interface DisconnectEvent {
	type: "disconnect",
	payload: { }
}
```
