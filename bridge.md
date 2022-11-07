# Bridge API

Bridge is a transport mechanism to deliver messages from the app to the wallet and vice versa.

* **Bridge is maintained by the wallet provider**. App developers do not have to choose or build a bridge. Each wallet’s bridge is listed in the [wallets-list](https://github.com/ton-connect/wallets-list) config.
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

**lastEventId** – is eventId of last event wallet got over bridge. In this case wallet will fetch all the events wich happened after last connection.

Sending message from client A to client B. Bridge returns error if ttl is too high.

```tsx
request
    POST /message?client_id=<to_hex_str(A)>?to=<to_hex_str(B)>&ttl=300

    body: <base64_encode(Message)>
```

Bridge buffers messages up to TTL (in secs), but removes them as soon as the recipient receives the message.

If the TTL exceeds the hard limit of the bridge server, it should respond with HTTP 400. Bridges should support at least 300 seconds TTL.

When the bridge receives a message `base64_encoded_message` from client `A` addressed to client `B`, it generates a message `BridgeMessage`:

```js
{
  "from": <to_hex_str(A)>,
  "message": <base64_encoded_message>
}
```

and sends it to the client B via SSE connection 
```js
resB.write(BridgeMessage)
```


## Universal link

When the app initiates the connection it sends it directly to the wallet via the QR code or a universal link.

```
https://<wallet-universal-url>/ton-connect?
                                 v=2&
                                 id=<to_hex_str(A)>&
                                 r=<base64urlsafe(ConnectRequest)>
```

Parameter **v** specifies the protocol version. Unsupported versions are not accepted by the wallets.

Parameter **id** specifies app’s Client ID encoded in URL-safe Base64.

Parameter **r** specifies URL-safe Base64 [ConnectRequest](https://github.com/ton-connect/docs/blob/main/requests-responses.md).

The link may be embedded in a QR code or clicked directly.

The initial request is unencrypted because (1) there is no personal data being communicated yet, (2) app does not even know the identity of the wallet.


## JS bridge

Used by the embedded apps via the injected binding `window.tonconnect`.

JS bridge runs on the same device as the wallet and the app, so communication is not encrypted.

The app works directly with plaintext requests and responses, without session keys and encryption.

```tsx
interface TonConnectBridge {
    deviceInfo: DeviceInfo; // see Requests/Responses spec
    protocolVersion: number; // max supported Ton Connect version (e.g. 2)
    connect(protocolVersion: number, message: ConnectRequest): Promise<ConnectEvent>;
    autoConnect(): Promise<ConnectEvent>;
    send(message: AppRequest): Promise<WalletResponse>;
    listen(callback: (event: WalletEvent) => void): () => void;
}
```

Just like with the HTTP bridge, wallet side of the bridge does not receive the app requests except for **ConnectRequest** until the session is confirmed by the user. Technically, the messages arrive from the webview into the bridge controller, but they are silently ignored.

SDK around the implements **autoconnect()** and **connect()** as silent and non-silent attempts at establishing the connection.

#### connect()

Initiates connect request, this is analogous to QR/link when using the HTTP bridge.

If the app was previously approved for the current account — connects silently with ConnectEvent.

#### autoConnect()

Initiates connect request, this is analogous to QR/link when using the HTTP bridge.
If the app was previously approved for the current account — connects silently with ConnectEvent.
If the app was not previously approved ignores this request and returns ConnectEventError.


#### send()

Sends the message to the bridge, excluding the ConnectRequest (that goes into QR code when using HTTP bridge and into connect when using JS Bridge).
Directly returns promise with WalletResponse, do you don't need to wait for responses with `listen`;

#### listen()

Registers a listener for events from the wallet. 

Returns unsubscribe function.

Currently, only `disconnect` event is available. Later there will be a switch account event and other wallet events.
