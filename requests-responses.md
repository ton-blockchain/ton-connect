# Requests and Responses

App sends requests to the wallet. Wallet sends responses and events to the app.

```tsx
type DappMessage = InitialRequest | DappRequest;

type WalletMessage = InitialReply | WalletEvent;
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
  name: "init_ok";
  items: ConnectItemReply[];
  device: DeviceInfo;
}
type InitialReplyError = {
  name: "init_error";
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

- All messages from the dapp to the wallet are requests for an operation.
- Messages from the wallet to the application can be either responses to dapp requests or events triggered by user actions on the side of the wallet.

**Available operations:**

- sendTransaction
- singMessage (will be implemented later)

**Available events:**

- disconnect

### Structure

All requests have the following structure

```tsx
interface DappRequest {
	method: string;
	params: string[];
	id: string;
}
```

Wallet messages are

```tsx
type WalletEventName = 'disconnect';

interface WalletEvent {
	type: WalletEventName | 'response';
	<event-name>: <event-payload>
}
```

Exactly

```tsx
type WalletEvent = SendTransactionResponse | DisconnectEvent;
```

Where response is

```tsx
interface Response {
	type: 'response';
	response: {
		result: unknown;
		id: number;
	} |
	{
		error: { code: number; message: string; data?: unknown };
		id: number;
	}
}
```

### Operations

Sign and send transaction

App sends **SendTransactionRequest**:

```tsx
interface SendTransactionRequest {
	method: 'sendTransaction';
	params: [<boc>];
	id: number;
}
```

Wallet replies with **SendTransactionResponse**:

```tsx
type SendTransactionResponse = SendTransactionResponseSuccess | SendTransactionResponseError; 

interface SendTransactionResponseSuccess {
 type: 'response';
	response: {
		result: <boc>;
		id: number;
	}
}

interface SendTransactionResponseError {
	type: 'response';
	response: {
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

Disconnect

The event fires when the user deletes the dapp in the wallet. The dapp must react to the event and delete the saved session. If the user disconnects the wallet on the dapp side, then the event does not fire, and the session information remains in the localstorage

```tsx
interface DisconnectEvent {
	"type": "disconnect",
	"disconnect": {
	}
}
```
