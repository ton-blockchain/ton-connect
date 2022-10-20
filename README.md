# Ton Connect Documentation

## Developing a TON app? Use the SDK.

We currently provide JS SDK suitable for web apps.

* [Ton Connect JS](https://github.com/ton-connect/sdk)
* [Getting Started](#TBD) (TBD)
* [Demo app](#TBD) (TBD)

If you use another language, please take the JS SDK as a reference and implement your own wrapper. Let us know about your work — we'd be happy to list it here.

## Protocol specifications

If you implement an SDK, a wallet or want to learn more how Ton Connect works, please read below.

* [Bridge API](https://github.com/ton-connect/docs/blob/main/bridge.md) specifies how the data is transmitted between the app and the wallet.
* [Session protocol](session.md) ensures end-to-end encrypted communication over the bridge.
* [Requests protocol](https://github.com/ton-connect/docs/blob/main/requests-responses.md) defines requests and responses for the app and the wallet.

## Q&A

#### I am building an HTML/JS app, what should I read?

Simply use the JS SDK and do not worry about the underlying protocols.

#### I need an SDK in my favorite language

Please take the JS SDK as a reference and check out the protocol docs above.

#### How do you detect whether the app is embedded in the wallet? 

JS SDK does that for you; see `autoconnect()` method. If you build your own SDK you should detect `window.tonconnect` binding.

#### How do you detect if the wallet is a browser extension? 

Like with embedded apps (see above), JS SDK detects it for you via `window.tonconnect` binding.

#### How do I make my own bridge? 

You don’t need to, unless you are building a wallet.

If you build a wallet, you will need to provide a bridge. See our [reference implementation in Go](https://github.com/ton-connect/bridge).

Keep in mind that the wallet’s side of the bridge API is not mandated.

#### I make a wallet, how do I add it to the list of wallets? 

Submit a pull request for the [wallets-list](https://github.com/ton-connect/wallets-list) repository and fill our all the necessary metadata.

Apps may also add or remove wallets directly through the SDK.
