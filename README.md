# Ton Connect Documentation

Ton Connect enables users to connect a wallet to an app, so they can perform actions in the app while using the wallet as an authentication device.

Ton Connect is designed to **communicate the user’s intent** between apps and wallets. It is not designed to send any other data unrelated to user’s actions: query the blockchain state, forward external events etc.

## SDK

We currently provide JS SDK suitable for web apps.

* [Ton Connect JS](#TODO)

If you use a language not supported here, please use the JS SDK as a reference and let us know — we'd be happy to list it here.

## Protocol specifications

* [Bridge API](https://github.com/ton-connect/docs/blob/main/bridge.md) specifies how the data is transmitted between the app and the wallet. We currently support HTTP and JS APIs. Each wallet manages their own bridge with a compatible app-facing APIs per this specification. Wallet-facing APIs are not mandated by this spec.
* [Session protocol](session.md) ensures end-to-end encrypted communication over the bridge.
* **Messages protocol** defines requests and responses for the app and the wallet.





[Requests and Responses](https://github.com/ton-connect/docs/blob/main/requests-responses.md)

[JS SDK](https://github.com/ton-connect/sdk)

### Q&A

#### I am building an HTML/JS app, what should I read

| I am building an HTML/JS app. | Use the JS SDK and skip all the other protocols. |
| --- | --- |
| I need an SDK in my favorite language. | Implement the messages protocol, encrypt them using the session protocol and then adopt HTTP bridge API to send requests to the wallet’s bridge. |
| How do you detect whether the app is embedded in the wallet? | JS SDK does that for you. See autoconnect() method. If you build your own SDK you should detect window.tonconnect binding.  |
| How do you detect if the wallet is a browser extension? | Like with embedded apps (see above), JS SDK detects it for you via window.tonconnect binding. |
| How do I make my own bridge? | You don’t need to, unless you are building a wallet. |
| I build a wallet, how do I make a compliant bridge? | See our reference implementation in Go. Keep in mind that the wallet’s side of the bridge API is not mandated and could be anything. |
| I make a wallet, how do I add it to the list of wallets? | Submit a pull request for the ‣ repository and fill our all the necessary metadata.
Apps may also add or remove wallets directly through the SDK. |
