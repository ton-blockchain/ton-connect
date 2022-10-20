# Session protocol

Session protocol defines client identifiers and offers end-to-end encryption for the app and the wallet. This means the HTTP bridge is fully untrusted and cannot read the user’s data transmitted between the app and the wallet. JS bridge does not use this protocol since both the wallet and the app run on the same device.

## Definitions

### Client Keypair

X25519 keypair for the use with NaCl “crypto_box” protocol.

```
a <- random 23 bytes
A <- X25519Pubkey(s)
```

or

```
(a,A) <- nacl.box.keyPair()
```


### Client ID

The public key part of the [Client Keypair](#client-keypair) (32 bytes).

### Session

A session is defined by a pair of two client IDs. Both the app and the wallet create their own [Client IDs](#client-id). 

### Encryption

All requests from the app (except the initial request) and all responses from the wallet are encrypted. 

Given a binary encoding of message **m**, recipient’s [Client ID](#client-id) **X** and sender’s private key **y** the message is encrypted as follows:

```
nonce <- random(24 bytes)
ct    <- nacl.box(m, nonce, X, y)
M     <- nonce ++ ct
```

That is, the final message **M** has the first 24 bytes set to the random nonce.

### Decryption

To decrypt the message **M**, the recipient uses its private key **x** and sender’s public key **Y** (aka [Client ID](#client-id)):

```
nonce <- M[0..24]
ct    <- M[24..]
m     <- nacl.box.open(ct, nonce, Y, x)
```

Plaintext **m** is recovered and parsed per “Requests/Responses” spec (insert the link).


## Workflows

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

When using the [JS bridge](https://github.com/ton-connect/docs/edit/main/bridge.md#js-bridge), the same request is sent via the `connect()` call:

```
window.tonconnect.connect(2, <InitialRequest>)
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

Wallet generates its own [Client Keypair](#client-keypair) (b,B) and stores alongside app’s info and ID.

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

