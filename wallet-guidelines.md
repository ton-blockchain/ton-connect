# TON Wallet Guidelines

## Networks

1) **There aren't many networks.**

    At the moment, there are only two networks - mainnet and testnet. 

    In the foreseeable future, the emergence of new mainnet TON-like networks is not expected. Note that the current mainnet has a built-in mechanism for alternative networks - workchains.


2) **Hide the testnet from ordinary users.**

    Testnet is used exclusively by developers. Ordinary users should not see the testnet. 

    This means that switching to testnet should not be readily available and users SHOULD NOT be prompted to switch wallet to testnet even if dapp is in testnet.

    Users switch to testnet, don't understand this action, can't switch back to mainnet.

    For these reasons, dapps do not need to switch networks in runtime, on the contrary, it is more preferable to have different instances of dapp on different domains dapp.com, testnet.dapp.com.
 
    For the same reason there is no `NetworkChanged` or `ChainChanged` event in the Ton Connect protocol.


3) **Do not send anything if the dapp is in testnet and the wallet is in mainnet.**

    It is necessary to prevent loss of funds when dapp tries to send a transaction in testnet, and the wallet sends it in mainnet.

    Dapps should explicitly indicate `network` field in `SendTransaction` request.

    If the `network` parameter is set, but the wallet has a different network set, the wallet should show an alert and DO NOT ALLOW TO SEND this transaction.

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

## Security Verification

### Connection Verification Implementation

1) **Implement `/verify` endpoint integration**

    For HTTP Bridge connections, wallets MUST implement connection verification:

    - Extract origin and client_id from connection requests
    - Send POST request to `${bridgeUrl}/verify` with connection details
    - Process verification status and display appropriate UI to users

2) **Verification Status Handling**

    Wallets MUST handle all verification statuses appropriately:
    
    - `ok` → Show that message is coming from the shown source, or show nothing
    - `ok` + whitelisted domain → Show that message is coming from the shown source AND add source is verified
    - `danger` → Show strong warning, recommend declining connection
    - `warning` → Show caution message, let user decide  
    - `unknown` → Proceed without special indicators (default behavior)

3) **Rollout Phase Awareness**
    
    - **Phase 1 (first 6 months)**: Only `ok` and `unknown` statuses returned
    - **Phase 2 (after 6 months)**: Full status set including `danger` and `warning`

### Transaction Verification Implementation

1) **Request Source Metadata Processing**

    Wallets MUST implement transaction source verification:
    
    - Decrypt `request_source` field from BridgeMessage using session private key
    - Parse BridgeRequestSource JSON containing origin, IP, User-Agent, timestamp, client_id
    - Obtain current IP address from `connect_source` field in BridgeMessage
    - Compare metadata against current user information
    - Display warnings for any mismatches

2) **Mismatch Detection and Warnings**

    Display appropriate warnings for detected mismatches:
    
    - **Different origin**: Strong fraud warning - likely impersonation attack
    - **Different IP/User-Agent**: Network change warning - could be legitimate or suspicious
    - **Significant time gaps**: Message is not relevant anymore
    - **Missing metadata**: Should not occure after rollout. Messages without metadata should be rejected

### User Interface Guidelines

1) **Verification Status Messages**

    Wallets should be able to display following statuses:
    - **Verification Mark (ok + whitelisted)**: 
      "✅ Verified dApp — confirmed request from a trusted source"

    - **Ok**:
      "No message"
    
    - **Danger Warning**:
      "⚠️ This request could not be verified and may be fraudulent. Do not proceed unless you are certain of the source."
    
    - **Warning Message**:
      "⚠️ This request's details differ from your current connection. This could be due to a network change or other unusual event. Proceed with caution."

2) **Transaction Source Display**

    In transaction confirmation dialogs, wallets SHOULD display:
    
    - Request source information (origin, timestamp)
    - Any detected mismatches with stored connection data
    - Clear warnings for security concerns
    - Country of the request origin(from ip)
    - Information about request user agent in the human readable format(e.g. Chrome 113 from macOS)

3) **User Education**

    Wallets SHOULD educate users about:
    
    - Meaning of verification marks and warnings
    - How to identify legitimate vs suspicious connection patterns
    - Best practices for safe dApp interaction
    - When to decline suspicious requests

### Security Best Practices

1) **Metadata Storage**

    - Store connection metadata securely and encrypted
    - Implement proper session management and cleanup
    - Limit metadata retention to necessary duration

2) **Verification Logic**

    - Implement proper cryptographic verification of encrypted metadata
    - Handle edge cases and error conditions gracefully

3) **User Safety**

    - Default to more restrictive behavior when verification fails
    - Provide clear actionable guidance to users
    - Never suppress security warnings to improve user experience

