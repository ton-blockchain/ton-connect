# TON Connect Specification

This repository provides details on the TON Connect protocol.

## Developing a DApp? Use the SDK

If you just want to integrate TON Connect into your app, just follow the [TON Documentation](https://docs.ton.org/develop/dapps/ton-connect/overview) guides. No need to dive deep into the protocols.

* [TON Connect Overview](https://docs.ton.org/develop/dapps/ton-connect/overview)
* [TON Connect for React Apps](https://docs.ton.org/develop/dapps/ton-connect/react)
* [TON Connect for HTML/JS Apps](https://docs.ton.org/develop/dapps/ton-connect/web)
* [List of supported SDKs](https://docs.ton.org/develop/dapps/ton-connect/developers)

If no SDK for your language, take the [JS SDK](https://github.com/ton-connect/sdk/tree/main/packages/sdk) as a reference and implement your own wrapper. Let us know about your work — we'd be happy to list it here.

## Protocol specifications

If you implement an SDK, a wallet or want to learn more about how TON Connect works, please read below.

* [Protocol workflow](workflows.md): an overview of all the protocols.
* [Bridge API](bridge.md) specifies how the data is transmitted between the app and the wallet.
* [Session protocol](session.md) ensures end-to-end encrypted communication over the bridge.
* [Requests protocol](requests-responses.md) defines requests and responses for the app and the wallet.
* [Wallet guidelines](wallet-guidelines.md) defines guidelines for wallet developers.

## Q&A

#### I am building an HTML/JS app. What should I read?

Simply use the [TON Documentation](https://docs.ton.org/develop/dapps/ton-connect/overview) manuals and do not worry about the underlying protocols.

#### I need an SDK in my favorite language

Please take the JS SDK as a reference and check out the protocol docs above.

#### How do you detect whether the app is embedded in the wallet? 

JS SDK does that for you; just get wallets list `connector.getWallets()` and check `embedded` property of the corresponding list item. If you build your own SDK you should check `window.[targetWalletJsBridgeKey].tonconnect.isWalletBrowser`.

#### How do you detect if the wallet is a browser extension? 

Like with embedded apps (see above), JS SDK detects it for you via `injected` property of the corresponding `connector.getWallets()` list item. If you build your own SDK you should check that `window.[targetWalletJsBridgeKey].tonconnect` exists.

#### How to implement backend authorization with TON Connect?

[See an example of dapp-backend](https://github.com/ton-connect/demo-dapp-backend)

#### How do I make my own bridge? 

You don’t need to unless you are building a wallet.

If you build a wallet, you will need to provide a bridge. See our [reference implementation in Go](https://github.com/ton-connect/bridge).

Keep in mind that the wallet’s side of the bridge API is not mandated.

For a quick start, you can use the common TON Connect bridge https://bridge.tonapi.io/bridge.

#### I make a wallet, how do I add it to the list of wallets? 

Submit a pull request for the [wallets-list](https://github.com/ton-blockchain/wallets-list) repository and fill out the wallet manifest.

Apps may also add wallets directly through the SDK.
