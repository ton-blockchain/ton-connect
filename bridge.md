# Bridge API

Bridge is a transport mechanism to deliver messages from the app to the wallet and vice versa.

* **Bridge is maintained by the wallet provider**. App developers do not have to choose or build a bridge. Each wallet’s bridge is listed in the [wallets-list](https://github.com/ton-blockchain/wallets-list) config.
* **Messages are end-to-end encrypted.** Bridge does not see the contents or long-term identifiers of the app or wallet.
* **Communication is symmetrical.** Bridge does not distinguish between apps and wallets: both are simply clients.
* Bridge keeps separate queues of messages for each recipient’s **Client ID**.

Bridge comes in two flavors:

- [HTTP Bridge](#http-bridge): for the external apps and services.
- [JS Bridge](#js-bridge): for apps opened within the wallet; or when the wallet is a browser extension.

## HTTP Bridge

Client with ID **A** connects to the bridge to listen to incoming requests.

**Client ID is semi-private:** apps and wallets are not supposed to share their IDs with other entities to avoid having their messages removed unexpectedly.

**Client can subscribe on few Client IDs** - in this case it should enumerate IDs separated with commas. For example: `?client_id=<A1>,<A2>,<A3>`

```tsx
request
    GET /events?client_id=<to_hex_str(A)>

    Accept: text/event-stream
```

**Subscribing to the bridge second (any other) time**

```tsx
request
    GET /events?client_id=<to_hex_str(A)>&last_event_id=<lastEventId>

    Accept: text/event-stream
```

**lastEventId** – the eventId of the last SSE event wallet got over the bridge. In this case wallet will fetch all the events which happened after the last connection.

Sending message from client A to client B. Bridge returns error if ttl is too high.

```tsx
request
    POST /message?client_id=<to_hex_str(A)>?to=<to_hex_str(B)>&ttl=300&topic=<sendTransaction|signData>[&no_request_source=true]

    body: <base64_encoded_message>
```


The `topic` [optional] query parameter can be used by the bridge to deliver the push notification to the wallet. If the parameter is given, it must correspond to the RPC method called inside the encrypted `message`.

The `no_request_source` [optional] query parameter can be used to disable request source metadata forwarding and encryption. When set to `true`, the bridge will not include the `request_source` field in the BridgeMessage. This parameter should be set for messages from `wallet` to `dapp`, since the `dapp` side doesn't need this information.

Bridge buffers messages up to TTL (in secs), but removes them as soon as the recipient receives the message.

If the TTL exceeds the hard limit of the bridge server, it should respond with HTTP 400. Bridges should support at least 300 seconds TTL.

## Bridge Security Verification

For enhanced security, the bridge implements verification mechanisms to help wallets confirm the true source of connection and transaction requests.

### Request Source Metadata

When the bridge receives a message from client A to client B, it collects request source data:

```go
type BridgeRequestSource struct {
    Origin    string `json:"origin"`     // protocol + domain (e.g., "https://app.ton.org")
    IP        string `json:"ip"`         // client IP address
    Time      string `json:"time"`       // unixtime
    UserAgent string `json:"user_agent"` // HTTP User-Agent header
}
```

### Message Processing with Verification

When the bridge receives a message `base64_encoded_message` from client `A` addressed to client `B`, it:

1. Collects request source metadata into `BridgeRequestSource` struct
2. Serializes the struct to JSON
3. Encrypts it using the recipient wallet's Curve25519 public key with:
   ```
   naclbox.SealAnonymous(nil, data, receiverSessionPublicKey, rand.Reader)
   ```
4. Base64 encodes the encrypted bytes
5. Generates a message `BridgeMessage`:

```js
{
  "from": <to_hex_str(A)>,
  "message": <base64_encoded_message>,
  "request_source": <base64_encoded_encrypted_request_source>
}
```

and sends it to the client B via SSE connection:
```js
resB.write(BridgeMessage)
```

### Connect Verification Endpoint

Bridge provides a verification endpoint for connection requests:

```tsx
request
    POST /verify

    body: {
      "type": "connect",
      "client_id": "<client_id>",
      "origin": "<protocol+domain>"
    }
```

### IP Address Endpoint

Bridge provides an endpoint for wallets to obtain their current IP address for validation purposes:

```tsx
request
    POST /myip
```

```tsx
response
    {
      "ip": "<client_ip_address>"
    }
```

This endpoint returns the IP address from which the request originated, allowing wallets to compare their current IP with the IP address stored in request source metadata for security validation.

**Response statuses:**

- **Phase 1** (first 6 months): `ok`, `unknown`
- **Phase 2** (after 6 months): `ok`, `danger`, `warning`

```tsx
response
    {
      "status": "ok" | "danger" | "warning" | "unknown"
    }
```

**Status meanings:**
- `ok`: Request verified and matches expected source
- `danger`: Strong indication of fraudulent activity
- `warning`: Suspicious activity or mismatched details
- `unknown`: Cannot verify (default for new or untracked origins)

**Bridge Implementation:**
- On SSE connect store connection metadata for a short period(~5 minutes): origin, IP address, client ID, timestamp
- Compare verification requests against stored data
- Implement rate limiting and abuse detection

### Heartbeat

To keep the connection, bridge server should periodically send a "heartbeat" message to the SSE channel. Client should ignore such messages.
So, the bridge heartbeat message is a string with word `heartbeat`.


## Universal link

When the app initiates the connection it sends it directly to the wallet via the QR code or a universal link.

```
https://<wallet-universal-url>?
                               v=2&
                               id=<to_hex_str(A)>&
                               r=<urlsafe(json.stringify(ConnectRequest))>&
                               ret=back
```

Parameter **v** specifies the protocol version. Unsupported versions are not accepted by the wallets.

Parameter **id** specifies app’s Client ID encoded as hex (without '0x' prefix).

Parameter **r** specifies URL-safe json [ConnectRequest](requests-responses.md#initiating-connection).

Parameter **ret** (optional) specifies return strategy for the deeplink when user signs/declines the request.
- 'back' (default) means return to the app which initialized deeplink jump (e.g. browser, native app, ...),
- 'none' means no jumps after user action;
- a URL: wallet will open this URL after completing the user's action. Note, that you shouldn't pass your app's URL if it is a webpage. This option should be used for native apps to work around possible OS-specific issues with `'back'` option.

`ret` parameter should be supported for empty deeplinks -- it might be used to specify the wallet behavior after other actions confirmation (send transaction, sign raw, ...).
```
https://<wallet-universal-url>?ret=back
```


The link may be embedded in a QR code or clicked directly.

The initial request is unencrypted because (1) there is no personal data being communicated yet, (2) app does not even know the identity of the wallet.

### Unified deeplink `tc`
In addition to its own universal link, the wallet must support the unified deeplink.

This allows applications to create a single qr code, which can be used to connect to any wallet.

More specifically, the wallet must support `tc://` deeplink as well as its own <wallet-universal-url>.

Therefore, the following `connect request` must be processed by the wallet:

```
tc://?
       v=2&
       id=<to_hex_str(A)>&
       r=<urlsafe(json.stringify(ConnectRequest))>&
       ret=back
```


## JS bridge

Used by the embedded apps via the injected binding `window.<wallet-js-bridge-key>.tonconnect`.

`wallet-js-bridge-key` can be specified in the [wallets list](https://github.com/ton-blockchain/wallets-list)

JS bridge runs on the same device as the wallet and the app, so communication is not encrypted.

The app works directly with plaintext requests and responses, without session keys and encryption.

```tsx
interface TonConnectBridge {
    deviceInfo: DeviceInfo; // see Requests/Responses spec
    walletInfo?: WalletInfo;
    protocolVersion: number; // max supported Ton Connect version (e.g. 2)
    isWalletBrowser: boolean; // if the page is opened into wallet's browser
    connect(protocolVersion: number, message: ConnectRequest): Promise<ConnectEvent>;
    restoreConnection(): Promise<ConnectEvent>;
    send(message: AppRequest): Promise<WalletResponse>;
    listen(callback: (event: WalletEvent) => void): () => void;
}
```

Just like with the HTTP bridge, wallet side of the bridge does not receive the app requests except for [ConnectRequest](requests-responses.md#initiating-connection) until the session is confirmed by the user. Technically, the messages arrive from the webview into the bridge controller, but they are silently ignored.

SDK around the implements **autoconnect()** and **connect()** as silent and non-silent attempts at establishing the connection.

#### walletInfo (optional)
Represents wallet metadata. Might be defined to make an injectable wallet works with TonConnect even if the wallet is not listed in the [wallets-list.json](https://github.com/ton-blockchain/wallets-list).

Wallet metadata format:
```ts
interface WalletInfo {
    name: string;
    image: <png image url>;
    tondns?:  string;
    about_url: <about page url>;
}
```

Detailed properties description: https://github.com/ton-blockchain/wallets-list#entry-format.

If `TonConnectBridge.walletInfo` is defined and the wallet is listed in the [wallets-list.json](https://github.com/ton-blockchain/wallets-list), `TonConnectBridge.walletInfo` properties will override corresponding wallet properties from the wallets-list.json. 


#### connect()

Initiates connect request, this is analogous to QR/link when using the HTTP bridge.

If the app was previously approved for the current account — connects silently with ConnectEvent. Otherwise shows confirmation dialog to the user.

You shouldn't use the `connect` method without explicit user action (e.g. connect button click). If you want automatically try to restore previous connection, you should use the `restoreConnection` method.

#### restoreConnection()

Attempts to restore the previous connection. 

If the app was previously approved for the current account — connects silently with the new `ConnectEvent` with only a `ton_addr` data item.


Otherwise returns `ConnectEventError` with error code 100 (Unknown app).


#### send()

Sends a [message](requests-responses.md#messages) to the bridge, excluding the ConnectRequest (that goes into QR code when using HTTP bridge and into connect when using JS Bridge).
Directly returns promise with WalletResponse, do you don't need to wait for responses with `listen`;

#### listen()

Registers a listener for events from the wallet. 

Returns unsubscribe function.

Currently, only `disconnect` event is available. Later there will be a switch account event and other wallet events.
