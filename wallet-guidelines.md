# TON Wallet Guidelines

## Networks

1) **Network IDs**

    TON Connect protocol supports any TON network ID.
    The `network` parameter accepts any TON network global_id as a string (e.g., `-239`, `-3`, or any other network ID).


2) **Hide the testnet from ordinary users.**

    Testnet is used exclusively by developers. Ordinary users should not see the testnet. 

    This means that switching to testnet should not be readily available and users SHOULD NOT be prompted to switch wallet to testnet even if dapp is in testnet.

    Users switch to testnet, don't understand this action, can't switch back to mainnet.

    For these reasons, dapps do not need to switch networks in runtime, on the contrary, it is more preferable to have different instances of dapp on different domains dapp.com, testnet.dapp.com.
 
    For the same reason there is no `NetworkChanged` or `ChainChanged` event in the Ton Connect protocol.


3) **Network mismatch protection**

    It is necessary to prevent loss of funds when dapp tries to send a transaction to one network (e.g., testnet), and the wallet sends it to another network (e.g., mainnet).

    Dapps should explicitly indicate `network` field in `SendTransaction` and `SignData` requests.

    If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SEND/SIGN this transaction.

    The wallet SHOULD NOT offer to switch to another network in this case.

## Multi accounts

Multiple network accounts can be created for one key pair. Implement this functionality in your wallet - users will find it useful.

1) **In general, there is no current "active" account**

    At the moment, the TON Connect is not built on the paradigm that there is one selected account in the wallet, and when the user switches to another account, the `AccountChanged` event is sent to dapp.

    We think of a wallet as a physical wallet that can contain many "bank cards" (accounts). 

    In most cases the sender address is not important to dapp, in these cases the user can select the appropriate account at the time of approving the transaction and the transaction will be sent from selected account. 

    In some cases, it is important for dapp to send the transaction from a specific address, in which case it explicitly specifies the `from` field in `SendTransaction` request. If `from` parameter is set, the wallet should DO NOT ALLOW user to select the sender's address; If sending from the specified address is impossible, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.


2)  **Login flow**

    When dapp connects the wallet, the user selects in the wallet one of their accounts that they want to log into dapp.

    Regardless of what accounts the user uses next in the wallet, dapp works with the account he received on the connection.

    Just like if you logged into a web service with one of your email accounts - if you then change the email account in the email service, the web service continues to use the one he got when you logged in.

    For this reason, the protocol does not provide the `AccountChanged` event.
    
    To switch account user need to disconnect (Log out) and connect  (Login) again in dapp UI.

    We recommend wallets provide the ability to disconnect session with a specified dapp because the dapp may have an incomplete UI.

