# Deep links and universal links

This document is normative for dApps, wallets, bridge operators, SDKs. The keywords MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

TON Connect uses deep-link transport for connect flows opened from QR codes, browsers, native dApps and in-app browsers.

## Link forms

### Universal link

Wallet-specific HTTPS link from `wallets-list` `universal_url`:

```text
https://<wallet-universal-url>?
                               v=2&
                               id=<to_hex_str(A)>&
                               r=<urlsafe(json.stringify(ConnectRequest))>&
                               ret=back&
                               trace_id=<uuid>&
                               e=<base64url(json.stringify(EmbeddedWireRequest))>
```

### Unified deep link `tc://`

In addition to the wallet-specific universal link, wallets MUST support:

```text
tc://?v=2&id=<to_hex_str(A)>&r=<urlsafe(json.stringify(ConnectRequest))>&ret=back&trace_id=<uuid>&e=<base64url(json.stringify(EmbeddedWireRequest))>
```

A single `tc://` link addresses any wallet that supports the unified scheme, so one QR code reaches them all.

### Custom-scheme deep link

Wallets MAY expose a wallet-specific custom scheme from `wallets-list` `deepLink` (for example `tonkeeper-tc://`, `mytonwallet-tc://`).

If `deepLink` is published, the wallet MUST accept the same TON Connect query parameters as universal links (`v`, `id`, `r`, optional `ret`, optional `trace_id`, optional `e`) and MUST apply the same validation and processing rules.

## Parameters

| Param      | Required | Meaning |
|------------|----------|---------|
| `v`        | yes      | Protocol version. Wallets MUST reject unsupported versions. |
| `id`       | yes      | dApp `client_id` encoded as hex (without `0x`). |
| `r`        | yes      | URL-safe JSON of [`ConnectRequest`](./connect.md). |
| `ret`      | no       | Return strategy after approval or rejection. |
| `trace_id` | no       | [UUID](https://www.rfc-editor.org/rfc/rfc9562) for analytics correlation. UUIDv7 is recommended. The wallet SHOULD reuse this value when it posts the matching connect-event reply to the bridge. See [`bridge.md` § `trace_id`](./bridge.md#trace_id--analytics-correlation). |
| `e`        | no       | Base64url JSON of [embedded request](#embedded-requests-e). |

The `id` and `ret` parameters SHOULD be supported even in empty deep links:

```text
https://<wallet-universal-url>?id=<to_hex_str(A)>&ret=back
```

## Return strategy (`ret`)

Allowed values:

- `back` (default): return to the dApp that initiated the jump.
- `none`: no return jump.
- custom URL: wallet opens that URL after user action.

dApps SHOULD NOT pass their own webpage URL as `ret` when running as a webpage.

For wallet-side handling of platform-specific return targets (Telegram in-app browser, iOS Safari, browser-specific deep links), see [`guides/wallet-guidelines.md` § Deep-link return targets](../guides/wallet-guidelines.md#deep-link-return-targets).

## Embedded requests (`e`)

The `e` parameter embeds `sendTransaction`, `signMessage` or `signData` into the connect link, so connection and action complete in one flow.

- Wallets that support this feature MUST advertise `{ name: 'EmbeddedRequest' }` in `DeviceInfo.features`.
- Wallets that do not support the feature MUST ignore `e`.
- Wallets MUST accept connect links both with and without `e`.
- `e` value format is `base64url(JSON.stringify(EmbeddedWireRequest))` without padding.

### Wire format

This section defines `EmbeddedWireRequest`. Wallet-side implementation guidance lives in [`guides/embedded-requests.md`](../guides/embedded-requests.md).

Field names are abbreviated to fit a URL. Each shape expands to the equivalent [`AppRequest`](./rpc.md#apprequest) payload before further processing.

#### Method discriminator

The top-level `m` field selects the request shape.

| `m`  | Method                                  | Wire shape            |
|------|-----------------------------------------|-----------------------|
| `st` | [`sendTransaction`](./rpc.md#sendtransaction) | `WireSendTransaction` |
| `sm` | [`signMessage`](./rpc.md#signmessage)         | `WireSignMessage`     |
| `sd` | [`signData`](./rpc.md#signdata)               | `WireSignData`        |

#### `WireSendTransaction` / `WireSignMessage`

Both methods share the same payload shape. Only `m` differs.

| Wire field | Full name     | Required | Type                  | Description |
|------------|---------------|----------|-----------------------|-------------|
| `m`        | —             | yes      | `"st"` \| `"sm"`      | Method discriminator. |
| `f`        | `from`        | no       | string                | Sender address (raw or friendly format). Defaults to the connected account. |
| `n`        | `network`     | no       | [`NETWORK_ID`](./connect.md#network_id) | Target TON network `global_id` as a stringified integer. |
| `vu`       | `valid_until` | no       | integer               | Unix seconds. The wallet MUST reject the request after this moment. |
| `ms`       | `messages`    | one of   | `WireMessage[]`       | Raw outgoing messages. Exactly one of `ms` or `i` MUST be present. |
| `i`        | `items`       | one of   | `WireItem[]`          | Structured items. Exactly one of `ms` or `i` MUST be present. |

#### `WireSignData`

The `t` field discriminates the payload variant.

| Wire field | Full name | Required | Type                            | Description |
|------------|-----------|----------|---------------------------------|-------------|
| `m`        | —         | yes      | `"sd"`                          | Method discriminator. |
| `f`        | `from`    | no       | string                          | Signer address. |
| `n`        | `network` | no       | [`NETWORK_ID`](./connect.md#network_id) | Target network. |
| `t`        | `type`    | yes      | `"text"` \| `"binary"` \| `"cell"` | Payload variant. |

Variant-specific fields:

| Variant (`t`) | Wire field | Full name | Required | Type   | Description |
|---------------|------------|-----------|----------|--------|-------------|
| `text`        | `tx`       | `text`    | yes      | string | UTF-8 text payload. |
| `binary`      | `b`        | `bytes`   | yes      | string | Base64-encoded bytes. |
| `cell`        | `s`        | `schema`  | yes      | string | TL-B schema describing the cell. |
| `cell`        | `c`        | `cell`    | yes      | string | Base64-encoded cell BoC. |

#### `WireMessage`

Raw outgoing message inside `ms`.

| Wire field | Full name        | Required | Type   | Description |
|------------|------------------|----------|--------|-------------|
| `a`        | `address`        | yes      | string | Destination address in friendly format. |
| `am`       | `amount`         | yes      | string | Nanograms as a decimal string. |
| `p`        | `payload`        | no       | string | Raw one-cell BoC, base64-encoded. |
| `si`       | `stateInit`      | no       | string | Raw one-cell BoC, base64-encoded. |
| `ec`       | `extra_currency` | no       | object | Map of extra-currency ID (integer) to amount (decimal string). |

#### `WireItem`

Structured item inside `i`. The `t` field discriminates the variant: `ton`, `jetton`, or `nft`.

##### `WireTonItem` (`t: "ton"`)

| Wire field | Full name        | Required | Type   | Description |
|------------|------------------|----------|--------|-------------|
| `t`        | `type`           | yes      | `"ton"` | Item discriminator. |
| `a`        | `address`        | yes      | string | Destination address in friendly format. |
| `am`       | `amount`         | yes      | string | Nanograms as a decimal string. |
| `p`        | `payload`        | no       | string | Raw one-cell BoC, base64-encoded. |
| `si`       | `stateInit`      | no       | string | Raw one-cell BoC, base64-encoded. |
| `ec`       | `extra_currency` | no       | object | Map of extra-currency ID to amount. |

##### `WireJettonItem` (`t: "jetton"`)

| Wire field | Full name             | Required | Type      | Description |
|------------|-----------------------|----------|-----------|-------------|
| `t`        | `type`                | yes      | `"jetton"` | Item discriminator. |
| `ma`       | `master`              | yes      | string    | Jetton master contract address. |
| `d`        | `destination`         | yes      | string    | Recipient address. |
| `am`       | `amount`              | yes      | string    | Jetton amount in elementary units. |
| `aa`       | `attachAmount`        | no       | string    | GRAM value (nanograms) to attach for transfer execution. |
| `rd`       | `responseDestination` | no       | string    | Address for the excess-GRAM refund. Defaults to the sender. |
| `cp`       | `customPayload`       | no       | string    | Raw one-cell BoC, base64-encoded. |
| `fa`       | `forwardAmount`       | no       | string    | Nanograms forwarded to the destination. |
| `fp`       | `forwardPayload`      | no       | string    | Raw one-cell BoC, base64-encoded. |
| `qi`       | `queryId`             | no       | string    | Query ID for the transfer body. |

##### `WireNftItem` (`t: "nft"`)

| Wire field | Full name             | Required | Type   | Description |
|------------|-----------------------|----------|--------|-------------|
| `t`        | `type`                | yes      | `"nft"` | Item discriminator. |
| `na`       | `nftAddress`          | yes      | string | NFT item contract address. |
| `no`       | `newOwner`            | yes      | string | New owner address. |
| `aa`       | `attachAmount`        | no       | string | GRAM value (nanograms) to attach. |
| `rd`       | `responseDestination` | no       | string | Address for the excess-GRAM refund. Defaults to the sender. |
| `cp`       | `customPayload`       | no       | string | Raw one-cell BoC, base64-encoded. |
| `fa`       | `forwardAmount`       | no       | string | Nanograms forwarded to the destination. |
| `fp`       | `forwardPayload`      | no       | string | Raw one-cell BoC, base64-encoded. |
| `qi`       | `queryId`             | no       | string | Query ID for the transfer body. |

The expanded payload is the same JSON shape that the wallet would receive over the standard bridge for [`sendTransaction`](./rpc.md#sendtransaction), [`signMessage`](./rpc.md#signmessage) or [`signData`](./rpc.md#signdata).

## See also

- [`bridge.md`](./bridge.md) — transport rules and full embedded wire format
- [`connect.md`](./connect.md) — `ConnectRequest` in `r`
- [`wallets-list.md`](./wallets-list.md) — wallet registry fields (`universal_url`, `deepLink`)
