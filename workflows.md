# Workflows

## Overview

### First-time connection via http bridge
1. App initiates SSE connection with bridge;
2. App passes connection info to the wallet via universal link or QR code;
3. Wallet connects to the bridge with given parameters, and save connection info locally;
4. Wallet sends account information to the app using bridge;
5. App receives message and save connection info locally;

### Reconnection with http bridge
1. App reads connection info from localstorage
2. App connects to the bridge
3. User opens the wallet, the wallet connects to the bridge using stored connection info

### First-time connection via js bridge
1. App checks existing of the `window.[walletJsBridgeKey].tonconnect`
2. App calls `window.[walletJsBridgeKey].tonconnect.connect()` and waits for a response
4. Wallet sends account information to the app;

###  Making ordinary requests and responses
1. App and wallet are in a connected state
2. App generates request and sends it to the bridge
3. Bridge forwards message to the wallet
4. Wallet generates response and sends it to the bridge
5. Bridge forwards message to the wallet


## Details

### App initiates connection

User clicks button “Connect Wallet” in the app.

App creates app’s Client Keypair (a, A):

```
(a,A) <- nacl.box.keyPair()
```

App generates the **InitialRequest**. See [requests spec](requests-responses.md).

App creates a [universal link](https://github.com/ton-connect/docs/blob/main/bridge.md#universal-link) to a target wallet:

```
https://<wallet-url>/ton-connect?v=2&id=<base64urlsafe(A)>&r=<base64urlsafe(InitialRequest)>
```

When using the [JS bridge](https://github.com/ton-connect/docs/blob/main/bridge.md#js-bridge), the same request is sent via the `connect()` call:

```
window.[walletJsBridgeKey].tonconnect.connect(2, <InitialRequest>)
```

Parameter **v** specifies the protocol version. Unsupported versions are not accepted by the wallets.

Parameter **id** specifies app’s Client ID encoded in URL-safe Base64.

Parameter **r** specifies URL-safe Base64 **InitialRequest**.

The link may be embedded in a QR code or clicked directly.

App connects to the bridge (link to bridge api) and listens for events using its Client ID **A**.

App is not yet in the connected state, and may restart the whole process at any moment.

### Wallet establishes connection

Wallet opens up a link or QR code, reads plaintext app’s **Client ID** (A from parameter “**id”**) and **InitialRequest** (from parameter **“r”**).

Wallet computes the **InitialResponse**.

Wallet generates its own [Client Keypair](https://github.com/ton-connect/docs/blob/main/session.md#client-keypair) (b,B) and stores alongside app’s info and ID.

Wallet encrypts the response and sends it to the **Bridge** using app’s Client ID A.

Wallet connects to the bridge (link to bridge api) and listens for events using its Client ID **B**.

### App receives the reply to the initial request

App receives the event from the bridge that contains the encrypted message from the Client ID **B.**

App decrypts the message and parses it as **InitialReply** (link to Requests/Responses spec).

If the reply is valid, App reads wallet info (address, proof of ownership etc.) and remembers the wallet’s ID **B**.

The session is considered established in the app when it has the wallet’s ID and can send requests directly to it through the bridge.

### Making ordinary requests and responses

When the user performs an action in the app, it may request confirmation from the wallet.

App generates request (see Requests/Responses spec).

App encrypts it to the wallet’s key B (see below).

App sends the encrypted message to B over the **Bridge**.

App shows “pending confirmation” UI to let user know to open the wallet.

Wallet receives the encrypted message through the Bridge.

Wallet decrypts the message and is now assured that it came from the app with ID **A.**

Wallet shows the confirmation dialog to the user, signs transaction and replies over the bridge with user’s decision: “Ok, sent” or “User cancelled”.

App receives the encrypted message, decrypts it and closes the “pending confirmation” UI.

